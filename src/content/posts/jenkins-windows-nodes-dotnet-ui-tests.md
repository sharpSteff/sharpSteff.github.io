---
title: "Provisioning Jenkins Windows Nodes for .NET Builds and UI Testing with Ansible"
author: "sharpSteff"
pubDatetime: 2026-06-24T08:00:00Z
slug: jenkins-windows-nodes-dotnet-ui-tests
featured: false
draft: false
tags:
  - jenkins
  - ansible
  - dotnet
  - windows
  - ci-cd
  - ui-testing
  - flaui
description: >
  How I provision Jenkins Windows execution nodes for .NET builds and — the tricky
  part — automated UI testing. Why GUI automation refuses to work from a Windows
  service, and how an auto-logon plus a logon-triggered scheduled task gives you a
  real interactive desktop that Ansible can stamp out repeatably.
---

Setting up a Windows build node for a .NET project is mostly an exercise in installing
the right tools: the SDK, Git, a few global tools, an MSI toolchain. Tedious, but
straightforward. Where it gets genuinely interesting is the moment you add **automated
UI testing** to the same node — because the way Jenkins normally runs an agent on Windows
is fundamentally incompatible with how GUI automation works.

This post walks through how I provision these nodes with Ansible, and digs into the one
problem that took the most thought: getting UI tests to actually see a desktop.

## Table of contents

## The two jobs of a build node

The nodes I'm describing have to do two quite different things:

1. **Headless work** — `dotnet build`, `dotnet test`, `dotnet publish`, NuGet restores,
   MSI installer creation, archiving artifacts, generating license files. None of this
   needs a screen.
2. **Windows UI tests** — an NUnit3 test project that drives a real WPF application with
   [FlaUI](https://github.com/FlaUI/FlaUI): finding controls, moving the mouse, sending
   keystrokes, taking screenshots, asserting on what's actually on screen.

The first category is the easy 90%. The second is where the design of the node really
matters.

## The UI tests: FlaUI

For the UI tests I use [FlaUI](https://github.com/FlaUI/FlaUI), a .NET wrapper around the
native Windows UI Automation APIs (UIA2/UIA3). It's a pure NuGet dependency — there's no
separate runner or browser-driver to install on the node — so it restores with the rest of
the project and runs straight through `dotnet test`. That keeps the toolchain side of the
node refreshingly boring: the .NET SDK is already there, and FlaUI comes along for the ride
with the test project.

A FlaUI test looks roughly like this — launch the app, find elements by automation ID, and
drive them like a user would:

```csharp
using var app = FlaUI.Core.Application.Launch("MyApp.exe");
using var automation = new UIA3Automation();

var window = app.GetMainWindow(automation);
window.FindFirstDescendant(cf => cf.ByAutomationId("UsernameBox"))
      .AsTextBox().Enter("test-user");
window.FindFirstDescendant(cf => cf.ByAutomationId("LoginButton"))
      .AsButton().Invoke();

Assert.That(window.FindFirstDescendant(cf => cf.ByAutomationId("StatusLabel"))
                  .AsLabel().Text, Is.EqualTo("Signed in"));
```

The important thing for this post is *how* FlaUI works under the hood. It talks to the
application through the Windows UI Automation provider — it inspects the live UI tree,
moves the real cursor, and synthesizes real input. All of that is mediated by a desktop and
a window station. Which is exactly why the node has to be set up the way the next section
describes: FlaUI driving a UI from inside Session 0 has nothing to drive.

## Why the usual Jenkins agent setup breaks UI tests

The default, sensible way to run a Jenkins agent on Windows is as a **Windows service**.
It starts on boot, restarts on failure, runs without anyone logged in. For headless builds
this is exactly what you want.

But Windows services run in **Session 0**, an isolated session with *no interactive
desktop*. This isolation was introduced back in Windows Vista for good security reasons,
and it has a hard consequence for UI automation: there is no desktop to automate.

In practice, that means:

- windows can't be found, because there's no window station with a visible desktop;
- mouse clicks and keystrokes go nowhere;
- screenshots come back **solid black**.

You can spend a long time debugging "flaky" UI tests before realizing the tests were never
flaky — the agent simply had no screen to drive. No amount of retry logic fixes a black
screenshot.

So the requirement becomes: **the agent must run inside an interactive, unlocked desktop
session that exists at all times**, surviving reboots, with nobody physically sitting at
the machine.

## The approach: auto-logon + a logon-triggered scheduled task

The combination that solves this cleanly:

1. **Auto-logon** a dedicated account on boot, so a real interactive session *always*
   exists after a reboot.
2. Start the Jenkins agent from a **scheduled task triggered on logon**, running in that
   interactive session — not as a service.
3. **Keep the session alive**: no lock screen, no screensaver, no display/sleep
   power-downs, and a fixed screen resolution.

Let me take those in turn.

### A dedicated, least-privilege agent account

The agent runs as its own local account rather than as an administrator. A standard user
is perfectly capable of running the agent and driving the desktop for UI tests, so that's
the default — it only gets `Modify` rights on the agent's working directory and nothing
more. Elevation is opt-in, for the rare case where the application under test itself needs
to run elevated.

```yaml
- name: Create dedicated Jenkins agent user
  win_user:
    name: "{{ jenkins_agent_user }}"
    password: "{{ jenkins_agent_password }}"
    password_never_expires: true
    groups: "{{ ['Administrators'] if jenkins_agent_admin else ['Users'] }}"
    groups_action: replace
    state: present
  no_log: true

- name: Grant the agent user modify rights on the Jenkins directory
  win_acl:
    path: "{{ jenkins_agent_dir }}"
    user: "{{ jenkins_agent_user }}"
    rights: Modify,Synchronize
    type: allow
    state: present
```

### Auto-logon so a session always exists

After a reboot there's nobody to log in — so the machine logs itself in. This is the
classic `Winlogon` registry approach:

```yaml
- name: Configure auto-logon for the agent account
  win_regedit:
    path: HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
    name: "{{ item.name }}"
    data: "{{ item.data }}"
    type: string
  loop:
    - { name: AutoAdminLogon,   data: "1" }
    - { name: DefaultUserName,  data: "{{ jenkins_agent_user }}" }
    - { name: DefaultPassword,  data: "{{ jenkins_agent_password }}" }
    - { name: DefaultDomainName, data: "{{ jenkins_agent_domain }}" }
  no_log: true
```

There's an honest trade-off here worth calling out: `AutoAdminLogon` stores the password
**in plain text** in the registry. For a locked-down, single-purpose CI node on an
internal network this is often an acceptable risk, but if it isn't acceptable for you, the
hardened alternative is Sysinternals **Autologon.exe**, which stashes the password as an
LSA secret instead. The playbook lets you turn the registry method off (`configure_autologon: false`)
and configure Autologon separately.

> One more detail: Windows can be configured to auto-logon only a fixed number of times via
> `AutoLogonCount`. Removing that value keeps auto-logon permanent, which is what you want
> for a node that may reboot for Windows Updates and must come back unattended.

### Launch the agent on logon, in the interactive session

Now that a session reliably exists, the agent starts from a **logon-triggered scheduled
task** rather than a service. The trigger fires for the agent user, and the task runs with
an `interactive_token` so it lands in the real desktop session:

```yaml
- name: Create Jenkins agent logon task (runs in interactive session)
  win_scheduled_task:
    name: JenkinsAgent
    actions:
      - path: powershell.exe
        arguments: >-
          -NoProfile -ExecutionPolicy Bypass
          -File "{{ jenkins_agent_dir }}\jenkins-agent.ps1"
    triggers:
      - type: logon
        user_id: "{{ jenkins_agent_user }}"
    username: "{{ jenkins_agent_user }}"
    logon_type: interactive_token
    run_level: "{{ 'highest' if jenkins_agent_admin else 'limited' }}"
    state: present
    enabled: true
```

The launch script itself downloads `agent.jar` from the controller if it's missing and
connects over WebSocket (which is friendly to firewalls and proxies, since it rides over
the same HTTP(S) port as the controller rather than a separate inbound TCP port):

```powershell
$AgentJar = Join-Path $AgentDir "agent.jar"
if (-not (Test-Path $AgentJar)) {
    (New-Object System.Net.WebClient).DownloadFile("$JenkinsUrl/jnlpJars/agent.jar", $AgentJar)
}

$JavaArgs = @(
    "-jar", $AgentJar,
    "-url", $JenkinsUrl,
    "-name", $AgentName,
    "-workDir", $WorkDir,
    "-webSocket",
    "-secret", $Secret
)
& java @JavaArgs
```

### Keep the desktop usable for automation

A logged-in session isn't enough on its own — Windows will happily lock it, blank it with
a screensaver, or power down the display, and any of those will sink a UI test. So the node
disables all of it:

```yaml
- name: Disable the lock screen
  win_regedit:
    path: HKLM:\SOFTWARE\Policies\Microsoft\Windows\Personalization
    name: NoLockScreen
    data: 1
    type: dword

- name: Disable display, sleep and hibernate power-downs (AC)
  win_command: "{{ item }}"
  loop:
    - powercfg /change monitor-timeout-ac 0
    - powercfg /change standby-timeout-ac 0
    - powercfg /change disk-timeout-ac 0
    - powercfg /change hibernate-timeout-ac 0
```

There's also the resolution problem. A headless or RDP-disconnected machine frequently
comes up with a tiny or unusable virtual resolution, which wrecks any test that clicks at
fixed coordinates. The agent launch script forces a known resolution (e.g. 1920×1080) via
a small `ChangeDisplaySettings` P/Invoke before the agent starts, so every run begins from
the same, predictable screen geometry.

## The headless 80%: installing the toolchain

With the hard part out of the way, the rest of the provisioning is a list of tool installs,
mostly via Chocolatey. A representative slice:

```yaml
- name: Install latest Git for Windows
  win_chocolatey:
    name: git
    state: latest

- name: Install .NET SDK
  win_chocolatey:
    name: "dotnet-{{ item }}-sdk"
    state: present
  loop: "{{ dotnet_sdk_versions }}"
```

A few non-obvious lessons baked into this part:

- **Git's long-path limit is separate from Windows'.** Setting `LongPathsEnabled` in the
  registry isn't enough — Git for Windows enforces its own `MAX_PATH` and ignores that flag.
  Deeply nested test-data paths fail with *"Filename too long"* until you also set
  `git config --system core.longpaths true`.
- **Install .NET global tools to a shared `--tool-path`, not `--global`.** Provisioning
  runs over SSH as one account, but Jenkins jobs run as the *agent* account. A `--global`
  install lands in the provisioning user's profile and is invisible to the agent. A
  machine-wide tool path on the system `PATH` is visible to both.
- **The same shared-location problem bites MSI tooling.** WiX caches its extensions per
  user by default; redirecting that cache to a machine-wide folder (via an environment
  variable both accounts resolve) is what makes the extensions visible to the build account.
- **Your SSH session's `PATH` is stale.** Right after installing tools, the bare commands
  aren't on the current session's `PATH`. Either call executables by full path or refresh
  `PATH` from the registry before verifying — otherwise your verification step reports
  perfectly-installed tools as missing.

The playbook finishes with a verification pass over every tool and only then reboots — which
triggers the auto-logon, which fires the logon task, which connects the agent. If any tool
fails verification, the reboot is skipped so you don't kick a half-provisioned node.

## Idempotency: don't disrupt a node that's testing

One subtle but important behavior: before redeploying the launch script or rebooting, the
playbook checks whether the agent is **already connected and running a job**. If it is, it
leaves the launch script and the reboot untouched. Re-running the playbook against a fleet
shouldn't yank a node out from under a test that's mid-run. This makes the playbook safe to
run repeatedly — the whole point of using Ansible in the first place.

```yaml
- name: Check whether the Jenkins agent is already connected
  win_shell: |
    $state = (Get-ScheduledTask -TaskName 'JenkinsAgent' -ErrorAction SilentlyContinue).State
    $javaIds = (Get-Process java -ErrorAction SilentlyContinue).Id
    # ...established connection to the controller port?
    if ($state -eq 'Running' -and $conn) { 'CONNECTED' } else { 'NOT-CONNECTED' }
  register: jenkins_agent_precheck
  changed_when: false
```

## Keeping secrets out of the repo

Nothing sensitive lives in source control. Every credential — the SSH password for
provisioning, the agent account password, and the **per-node** inbound-agent secret — is
read from environment variables (or an Ansible Vault file) at run time. In CI, those are
masked pipeline secrets; locally, you export them (or source a git-ignored `.env`) before
running. Each Jenkins node has its own inbound-agent secret, so the convention is one
variable per host, suffixed with the node name.

The Ansible tasks that touch these values are marked `no_log: true` so the secrets never
land in the job log either.

## Wrapping up

The headline lesson, if there's one to take away: **UI automation needs a real, interactive,
unlocked desktop**, and the default Windows-service agent can't give it one. Auto-logon plus
a logon-triggered scheduled task is the combination that does — and once you've encoded all
of it (plus the toolchain, the PATH quirks, the shared install locations, and the
idempotency guards) into an Ansible role, standing up a new node goes from a day of
clicking around to a single playbook run.

The black-screenshot rabbit hole is one I'd happily save someone else from falling into.
