# PRD: Reproducible macOS Configuration Repository

**Status:** Proposed
**Version:** 1.0
**Target platform:** macOS, primarily Apple Silicon
**Primary use case:** One developer maintaining one or more personal Macs
**Primary technologies:** Lix/Nix, nix-darwin, Home Manager, nix-homebrew, Homebrew
**Source of truth:** Git repository

---

# 1. Summary

Create a Git repository that declaratively describes the desired configuration of a macOS workstation.

The repository should make it possible to:

1. Start with a new or factory-reset Mac.
2. Perform a very small number of unavoidable manual bootstrap steps.
3. Clone this repository.
4. Apply the configuration.
5. Arrive at substantially the same operating-system configuration, command-line environment, applications, shell configuration, and developer tooling as an existing configured Mac.
6. Make future workstation changes by editing the repository, testing them, applying them, and committing them.
7. Detect configuration drift rather than accumulating undocumented machine state.

The repository should use:

* **nix-darwin** for macOS/system configuration.
* **Home Manager** for the user's environment and dotfiles.
* **Nixpkgs** for CLI software and other packages that work naturally through Nix.
* **nix-homebrew** to provision Homebrew itself.
* **nix-darwin's Homebrew integration** for declarative Homebrew formulae, casks, and Mac App Store applications.
* **Homebrew casks primarily for native macOS GUI applications.**
* Existing project-specific tools such as `uv` and `mise` for project runtime management.

Home Manager should be integrated into nix-darwin so that there is a **single system activation operation**, rather than independent `darwin-rebuild` and `home-manager switch` workflows. Home Manager explicitly supports this model.

The intended mental model is:

> Git describes the machine.
> `check` verifies the description.
> `apply` makes the Mac conform to it.

---

# 2. Goals

## 2.1 Reproducibility

A fresh Mac configured from a particular Git revision should substantially reproduce the configuration represented by that revision.

For Nix-managed software, dependency revisions must be pinned by `flake.lock`.

The project should distinguish between:

* **strongly reproducible state**, such as Nix packages and configuration;
* **declaratively managed but version-fluid state**, such as many Homebrew casks;
* **manual state**, such as Apple ID authentication and macOS privacy permissions.

The README must explicitly explain this distinction rather than claiming byte-for-byte reproduction of the entire Mac.

## 2.2 Incremental adoption

The repository must be useful before every aspect of the user's Mac is represented.

It should be natural to configure one thing at a time:

1. notice something that should be changed;
2. identify the appropriate configuration layer;
3. add it to Git;
4. build/check;
5. apply;
6. verify;
7. commit.

There must be no requirement to inventory and encode the entire existing Mac before the repository becomes useful.

## 2.3 One obvious workflow

Normal usage should center around a handful of commands:

```bash
./bin/check
./bin/apply
./bin/update
./bin/doctor
```

The user should almost never need to remember detailed Nix invocation syntax.

## 2.4 Safe changes

A bad configuration should normally be discovered before system activation.

The workflow should distinguish:

```text
evaluate/check
      ↓
build
      ↓
apply
```

An `apply` should not silently update every dependency merely because it was run.

## 2.5 Minimal imperative state

Installing software manually should be treated as temporary exploration rather than the normal workflow.

For example:

```bash
brew install something
```

may be useful while experimenting, but the permanent operation is:

1. add it to the repository;
2. apply the configuration;
3. commit the change.

## 2.6 Understandability

This is a personal workstation repository, not a generalized configuration-management framework.

Prefer explicit configuration over abstractions.

A future reader should be able to open:

```text
modules/darwin/homebrew.nix
```

and immediately understand which GUI applications are expected to exist.

## 2.7 Multiple-machine readiness

Version 1 only needs to configure the primary Mac well.

However, the layout should support adding another Mac later without restructuring the repository.

For example:

```text
hosts/
  macbook-pro.nix
  mac-studio.nix
```

with most configuration shared between them.

---

# 3. Non-goals

The project is explicitly **not** intended to:

* manage a corporate fleet of Macs;
* replace MDM;
* automate Apple ID authentication;
* automate Touch ID enrollment;
* bypass macOS privacy/security prompts;
* put credentials or API keys into the Nix store;
* replace `uv`, `mise`, language package managers, or project lockfiles;
* make every application's runtime state declarative;
* recreate caches, browser history, application databases, or ephemeral data;
* declaratively reproduce every file under `$HOME`;
* pin exact versions of every Homebrew GUI application;
* use Ansible;
* use Nix development shells as a mandatory replacement for existing project workflows.

---

# 4. Architectural principles

## 4.1 One flake

The repository must contain a single root:

```text
flake.nix
flake.lock
```

All Nix inputs should be managed through this flake.

For the initial implementation, prefer a current stable Nixpkgs/nix-darwin/Home Manager release rather than tracking arbitrary `master` revisions.

At the time of writing, nix-darwin's documentation explicitly supports release 26.05 as well as unstable, and recommends flakes for new configurations.

Conceptually:

```nix
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixpkgs-26.05-darwin";

  nix-darwin.url =
    "github:nix-darwin/nix-darwin/nix-darwin-26.05";

  nix-darwin.inputs.nixpkgs.follows = "nixpkgs";

  home-manager.url =
    "github:nix-community/home-manager/release-26.05";

  home-manager.inputs.nixpkgs.follows = "nixpkgs";

  nix-homebrew.url =
    "github:zhaofengli/nix-homebrew";
};
```

The exact selected branches should be validated when implementing the repository.

`flake.lock` must be committed.

Running a normal `apply` must **not** mutate `flake.lock`.

Updating dependencies must be a separate explicit action.

---

# 5. Ownership model

Every piece of workstation configuration should have exactly one preferred owner.

Use this order when deciding where something belongs.

## 5.1 nix-darwin: operating system

Use nix-darwin for things that conceptually describe the Mac rather than a particular application.

Examples:

* Finder preferences
* Dock preferences
* keyboard repeat behavior
* trackpad preferences
* global macOS preferences
* hostname
* timezone
* launchd services
* system shells
* system-level security configuration
* Touch ID support for `sudo`
* Homebrew integration

nix-darwin exposes many macOS defaults as typed configuration options rather than requiring raw `defaults write` commands. Its current option set also includes support for Touch ID/Apple Watch authentication for `sudo`.

Preference order:

```text
typed nix-darwin option
        ↓
CustomUserPreferences / equivalent
        ↓
small idempotent activation script
        ↓
manual step
```

Activation shell scripts are a last resort.

---

# 6. Home Manager: user environment

Use Home Manager for configuration belonging to the Unix user.

Examples:

* Git
* zsh
* shell aliases
* shell environment variables
* tmux
* SSH client configuration where appropriate
* command-line program configuration
* XDG configuration files
* dotfiles
* CLI packages used by the user

Home Manager is specifically designed to reproducibly manage user programs, configuration files, environment variables, and arbitrary files.

Home Manager must be installed **as a nix-darwin module**, not operated independently.

Use:

```nix
home-manager.useGlobalPkgs = true;
home-manager.useUserPackages = true;
```

unless implementation reveals a concrete reason not to.

This gives the system and Home Manager the same Nixpkgs package set and allows a single `darwin-rebuild` operation to activate both.

---

# 7. Nix packages: CLI software

Prefer Nixpkgs for command-line software when the package works normally on macOS.

Examples might include:

```text
bat
curl
fd
fzf
gh
git
htop
jq
ripgrep
shellcheck
tree
wget
yq
```

The exact initial package list should be conservative.

Do not install packages merely because they might someday be useful.

The desired configuration should represent actual workstation intent.

---

# 8. Homebrew: native applications and exceptions

Use Homebrew primarily for:

* native macOS GUI applications;
* vendor-distributed software better supported through casks;
* software whose Nix packaging causes substantial macOS integration problems;
* Mac App Store management where useful.

Examples:

```text
1Password
Chrome
Docker Desktop
Ghostty
Raycast
Slack
Zed
Zoom
```

These examples are illustrative rather than a mandated initial application list.

nix-homebrew should provision Homebrew itself.

nix-homebrew's role must remain narrow: it manages the Homebrew installation and optionally taps; nix-darwin's `homebrew.*` configuration manages packages and casks. This separation is explicitly how nix-homebrew is designed to operate.

---

# 9. Homebrew drift policy

Version 1 should use:

```nix
homebrew.onActivation = {
  autoUpdate = false;
  upgrade = false;
  cleanup = "check";
};
```

Rationale:

### `autoUpdate = false`

A normal system activation should not unexpectedly fetch new Homebrew metadata and change package behavior.

### `upgrade = false`

`./bin/apply` means:

> apply my configuration

not:

> upgrade software on my computer.

nix-darwin intentionally defaults these behaviors to false so repeated activation can remain idempotent.

### `cleanup = "check"`

This is particularly useful for this repository.

If someone manually performs:

```bash
brew install foo
```

and forgets to declare it, the next activation should flag the drift.

In `"check"` mode, nix-darwin reports undeclared Homebrew packages and aborts rather than deleting them.

This is preferable initially to:

```nix
cleanup = "uninstall";
```

because automatic deletion is more surprising.

Once the configuration is mature, switching to `"uninstall"` may be considered.

Do **not** default to `"zap"` because cask zap operations can remove associated application data.

---

# 10. Project runtime boundary

This repository manages the workstation.

It must not attempt to own every runtime required by individual software projects.

The expected boundary is:

```text
Mac / global developer environment
    nix-darwin + Home Manager

Python repository
    uv / pyproject.toml / uv.lock

Node repository
    mise / package.json / lockfile

Other project runtimes
    project's native tooling
```

For example, it is reasonable for the workstation configuration to install:

```text
uv
mise
```

while individual projects determine:

```text
Python 3.13.8
Node 24.x
```

This prevents the workstation configuration from becoming coupled to individual repositories.

---

# 11. Proposed repository structure

The initial repository should have approximately this shape:

```text
mac-config/
├── README.md
├── AGENTS.md
├── flake.nix
├── flake.lock
├── .gitignore
│
├── hosts/
│   └── <primary-host>.nix
│
├── modules/
│   └── darwin/
│       ├── default.nix
│       ├── nix.nix
│       ├── macos.nix
│       ├── homebrew.nix
│       ├── packages.nix
│       └── security.nix
│
├── home/
│   ├── default.nix
│   ├── packages.nix
│   ├── shell.nix
│   ├── git.nix
│   └── programs/
│       ├── tmux.nix
│       └── ...
│
├── dotfiles/
│   └── ...
│
├── bin/
│   ├── bootstrap
│   ├── check
│   ├── build
│   ├── apply
│   ├── update
│   └── doctor
│
└── docs/
    ├── manual-steps.md
    ├── architecture.md
    ├── package-policy.md
    └── recovery.md
```

Do not create empty directories or placeholder modules merely to exactly match this structure.

The implemented structure should grow when there is content to justify it.

---

# 12. `flake.nix`

`flake.nix` is the composition root.

It is responsible for:

* declaring external inputs;
* pinning nix-darwin, Home Manager, Nixpkgs, and nix-homebrew through `flake.lock`;
* constructing each supported `darwinConfiguration`;
* passing shared arguments such as username and hostname;
* loading common Darwin modules;
* loading the host-specific module;
* loading Home Manager;
* exposing a formatter if useful.

It should contain very little actual workstation configuration.

Bad:

```nix
flake.nix = {
  # 300 lines of packages, Dock settings, Git settings, etc.
}
```

Good:

```text
flake.nix
    ↓
composition

modules/
    ↓
shared configuration

hosts/
    ↓
machine-specific configuration

home/
    ↓
user configuration
```

---

# 13. Host configuration

Every Mac should have an explicit host entry.

Example:

```text
hosts/macbook-pro.nix
```

A host configuration may contain:

* hostname;
* computer display name;
* architecture;
* host-specific packages;
* host-specific Homebrew applications;
* hardware-specific preferences;
* configuration specific to a work versus personal machine.

It should **not** duplicate all shared configuration.

Conceptually:

```nix
{
  networking.hostName = "macbook-pro";

  # machine-specific differences here
}
```

A second Mac should eventually be addable by creating:

```text
hosts/mac-studio.nix
```

and another `darwinConfigurations` entry.

---

# 14. User configuration

Version 1 may assume one primary user.

The username should be defined in one obvious location and passed into modules rather than scattered through the repository.

Avoid repeated literals such as:

```nix
"/Users/matt"
users.users.matt
home-manager.users.matt
```

throughout unrelated files.

Prefer a value such as:

```nix
username = "...";
```

passed through the flake/module configuration.

The architecture should allow another user later but does not need a generic multi-user framework today.

---

# 15. macOS configuration

Create a dedicated:

```text
modules/darwin/macos.nix
```

for macOS preferences.

Organize it by macOS subsystem.

For example:

```nix
system.defaults = {
  dock = {
    # ...
  };

  finder = {
    # ...
  };

  NSGlobalDomain = {
    # ...
  };

  trackpad = {
    # ...
  };
};
```

This module should be intentionally readable.

Prefer:

```nix
dock.autohide = true;
```

over opaque shell commands.

The initial repository should not make dozens of subjective macOS changes based on someone else's popular dotfiles repository.

The first configuration should represent only settings explicitly selected by the owner.

Future settings should be added iteratively.

---

# 16. Shell configuration

Home Manager should own the shell environment.

The project should assume zsh unless explicitly changed later.

`home/shell.nix` should cover things such as:

* enabling zsh;
* PATH additions that genuinely belong globally;
* aliases;
* shell initialization;
* history behavior;
* environment variables;
* integration with `mise`, `fzf`, etc.

Avoid maintaining a giant opaque `.zshrc` if Home Manager has structured support for the desired behavior.

Raw files are fine when they are clearer.

The principle is:

> Home Manager modules where useful; plain configuration files where simpler.

---

# 17. Git configuration

`home/git.nix` should declaratively manage stable Git preferences.

Examples:

* default branch;
* pull/rebase behavior;
* aliases;
* diff preferences;
* Git LFS if desired;
* user name/email if appropriate.

Machine-specific or corporate Git settings should be separated rather than buried in a global monolithic config.

Credentials must never be stored in Nix expressions.

---

# 18. Dotfiles policy

Use this preference order:

1. Home Manager program module.
2. Home Manager-generated configuration.
3. `xdg.configFile`.
4. `home.file`.
5. manual configuration.

Example:

```nix
xdg.configFile."some-app/config.toml".source =
  ../dotfiles/some-app/config.toml;
```

Do not symlink configuration into the Nix store when the application itself expects to mutate that same file.

For applications with mixed static/runtime configuration, declaratively manage only the stable part.

---

# 19. Secrets

**No secret may enter the Nix store.**

This includes:

* API keys;
* AWS secrets;
* access tokens;
* private SSH keys;
* private certificates;
* passwords;
* authentication cookies.

Do not write:

```nix
environment.variables.OPENAI_API_KEY = "...";
```

or equivalent.

The Nix store is not a secrets vault.

Authentication should initially remain in systems designed for authentication:

* macOS Keychain;
* SSH agent;
* 1Password or another password manager;
* AWS credential mechanisms;
* application-specific login state.

If declarative secret management becomes desirable later, introduce a dedicated encrypted secrets system as a separate project decision.

It is not required for v1.

---

# 20. Lix/Nix installation

Use the Lix installer for the bootstrap.

nix-darwin currently recommends the Lix installer in part because it provides an automated uninstall path, and nix-darwin supports both upstream Nix and Lix.

The official Lix installer currently documents:

```bash
curl --proto '=https' \
  --tlsv1.2 \
  -sSf \
  -L https://install.lix.systems/lix \
  | sh -s -- install
```

and provides `/nix/lix-installer uninstall` for removal.

If the repository intends to continue running Lix after nix-darwin activation, configure:

```nix
nix.package = pkgs.lix;
```

because nix-darwin otherwise manages the Nix installation and defaults to upstream Nix.

---

# 21. Bootstrap workflow

`bin/bootstrap` exists for a **new Mac**.

It is not the normal daily command.

The README should describe the complete process.

## Phase A: unavoidable macOS setup

The human performs macOS Setup Assistant and creates the primary administrator account.

Potential manual tasks include:

* Apple ID login;
* FileVault decision;
* network access;
* Touch ID setup.

These are outside the repository.

## Phase B: developer bootstrap

Install Apple's Command Line Tools if required:

```bash
xcode-select --install
```

Install Lix.

Clone the repository.

Example:

```bash
mkdir -p ~/dev
cd ~/dev
git clone <repo-url> mac-config
cd mac-config
```

## Phase C: first activation

`bin/bootstrap <host>` should:

1. confirm macOS;
2. confirm architecture;
3. verify Nix/Lix exists;
4. verify the requested host exists in the flake;
5. show the selected host;
6. perform a build;
7. invoke the initial nix-darwin activation;
8. stop with an informative error if any prerequisite is unavailable.

Because `darwin-rebuild` does not exist before the first nix-darwin installation, bootstrap may invoke it through Nix itself.

nix-darwin documents this exact bootstrap model: use `nix run ...#darwin-rebuild -- switch` on the first installation, then use the installed `darwin-rebuild` command subsequently.

The script should hide those details from normal users.

---

# 22. `bin/check`

This is the cheapest normal validation operation.

It should:

* validate the flake;
* evaluate configuration;
* run formatting checks where practical;
* detect obvious shell-script problems;
* return a nonzero exit code on failure.

The command must not modify system state.

Example user flow:

```bash
./bin/check
```

Expected output should be concise.

---

# 23. `bin/build`

`bin/build` should build the selected machine configuration without activating it.

Conceptually:

```bash
darwin-rebuild build --flake ".#${HOST}"
```

Its purpose is:

> Prove that this configuration can produce a system generation.

The host may default to the current `LocalHostName`, with an explicit argument or environment-variable override supported for bootstrap and testing.

---

# 24. `bin/apply`

This is the primary command.

User behavior:

```bash
./bin/apply
```

Internally it should roughly perform:

```text
resolve host
    ↓
check
    ↓
build
    ↓
sudo darwin-rebuild switch --flake .#host
```

It should not:

* update `flake.lock`;
* implicitly upgrade the entire Homebrew installation;
* run arbitrary application installers outside declared configuration;
* mutate Git.

Running it twice against an unchanged repository should result in no material configuration changes.

---

# 25. `bin/update`

Updating dependencies must be explicit.

The normal workflow is:

```bash
./bin/update
git diff
./bin/check
./bin/apply
git commit
```

`bin/update` should update the Nix flake lock.

Home Manager documents `nix flake update` as the mechanism for updating pinned flake inputs.

The implementation should print which inputs changed.

Homebrew upgrades should be treated separately because Homebrew packages and casks do not have the same reproducibility semantics as Nix packages.

Possible commands may therefore eventually be:

```bash
./bin/update
./bin/update-homebrew
```

rather than conflating them.

Version 1 may omit `update-homebrew` if application self-updates are sufficient.

---

# 26. `bin/doctor`

`doctor` should answer:

> Is this Mac broadly consistent with the assumptions of this repository?

It should perform non-destructive checks such as:

* macOS detected;
* supported CPU architecture;
* Nix/Lix command available;
* Nix daemon responsive;
* flakes available;
* current hostname;
* hostname represented by the flake;
* Homebrew available;
* expected Homebrew prefix;
* Git worktree status;
* `darwin-rebuild` available after bootstrap;
* potentially detect obvious undeclared Homebrew drift.

Output should look roughly like:

```text
✓ macOS
✓ Apple Silicon
✓ Lix
✓ nix-daemon
✓ host: macbook-pro
✓ flake configuration found
✓ Homebrew
✓ darwin-rebuild

Configuration appears healthy.
```

Failure messages should include the appropriate corrective command.

---

# 27. Manual-state documentation

Create:

```text
docs/manual-steps.md
```

This file is important.

It should capture things that cannot or should not be automated.

Examples:

```text
[ ] Sign into iCloud
[ ] Enable FileVault
[ ] Configure Touch ID
[ ] Sign into 1Password
[ ] Grant terminal Full Disk Access if desired
[ ] Grant application accessibility permissions where required
[ ] Sign into Chrome
[ ] Sign into Slack
[ ] Authenticate GitHub CLI
[ ] Authenticate AWS
[ ] Configure application licenses
```

The exact checklist should grow based on actual experience rebuilding the machine.

Do not fake declarative automation for things that are fundamentally interactive.

---

# 28. State-version policy

Both nix-darwin and Home Manager have compatibility state versions.

These are **not package versions** and should not be casually bumped during dependency updates.

The initial implementation should:

* use the state version recommended for a new installation by the selected stable releases;
* commit it explicitly;
* document it;
* include a comment saying not to change it casually.

Routine:

```bash
nix flake update
```

must not imply changing either state version.

---

# 29. Application-install decision tree

Add the following to `docs/package-policy.md`.

When installing software, ask:

### Is this project-specific?

Yes → use the project's package/runtime manager.

Examples:

```text
uv
npm/pnpm
mise
Cargo
```

Otherwise continue.

### Is it a CLI program well supported in Nixpkgs?

Yes → install through Nix/Home Manager.

### Does Home Manager have a useful module for the program?

Yes → use the Home Manager module.

### Is it a native Mac GUI application?

Usually → Homebrew cask.

### Is it primarily distributed through the Mac App Store?

Use the Mac App Store integration if reliable.

### Is it impossible or inappropriate to automate?

Document it in `manual-steps.md`.

There must not be two package managers owning the same program.

---

# 30. Configuration-change workflow

This is the central daily usage model.

Suppose the user decides:

> I want `jq`.

Do not permanently do:

```bash
brew install jq
```

Instead:

1. edit:

```text
home/packages.nix
```

2. add:

```nix
pkgs.jq
```

3. run:

```bash
./bin/check
```

4. inspect:

```bash
git diff
```

5. apply:

```bash
./bin/apply
```

6. verify:

```bash
jq --version
```

7. commit:

```bash
git add .
git commit -m "Install jq"
```

The Git history becomes the machine's configuration history.

---

# 31. Exploring unknown macOS settings

A different workflow is appropriate for poorly documented macOS preferences.

For example:

> I want Finder to behave differently.

It is acceptable to:

1. change the preference manually in System Settings/Finder;
2. inspect the relevant preference value;
3. determine the nix-darwin representation;
4. revert or leave the manual state temporarily;
5. encode the desired state in the repository;
6. apply;
7. verify;
8. commit.

The repository should optimize for **eventual declarative ownership**, not prohibit interactive experimentation.

---

# 32. Updating dependencies

Dependency updates should be deliberate Git events.

Recommended cadence:

```bash
git checkout -b chore/update-system
./bin/update
git diff flake.lock
./bin/check
./bin/build
./bin/apply
```

Then exercise the machine normally.

If successful:

```bash
git add flake.lock
git commit -m "Update Nix dependencies"
```

Home Manager's documentation confirms that flake inputs remain pinned until the lock file is explicitly updated.

This is a major reason for using flakes.

---

# 33. Rollback and recovery

Create:

```text
docs/recovery.md
```

Document at least:

## Bad uncommitted config

Revert the source:

```bash
git restore .
```

and re-apply.

## Bad Git commit

Return to a known-good revision:

```bash
git checkout <known-good-revision>
./bin/apply
```

## Bad activated generation

Document how to inspect and activate previous nix-darwin generations.

The exact commands should be validated against the selected nix-darwin release when implementing this document.

## Complete removal

Document nix-darwin's uninstaller and Lix's uninstaller separately.

nix-darwin currently provides:

```bash
sudo darwin-uninstaller
```

or its flake-based uninstaller.

Lix's installer provides:

```bash
/nix/lix-installer uninstall
```

Recovery documentation should emphasize restoring a previous configuration before reaching for complete uninstallation.

---

# 34. Git policy

The repository should be private unless all contents are intentionally public-safe.

Even in a private repository:

**never commit secrets.**

Recommended commit granularity:

```text
Install Ghostty
Configure Git defaults
Enable Finder path bar
Set keyboard repeat rate
Add tmux configuration
Update Nix dependencies
```

Avoid giant commits such as:

```text
Configure Mac
```

Small commits make configuration changes:

* understandable;
* revertible;
* bisectable;
* transferable between machines.

---

# 35. Branching

Direct commits to `main` are acceptable for tiny changes.

For riskier configuration:

```bash
git switch -c configure-touch-id-sudo
```

Then:

```bash
./bin/check
./bin/build
./bin/apply
```

Only merge after verification.

The repository should not require elaborate PR ceremony for personal use.

---

# 36. Formatting and linting

The repository should provide a standard Nix formatter through the flake.

Shell scripts should:

* use Bash;
* use `set -euo pipefail`;
* quote variables;
* pass ShellCheck;
* determine repository root robustly;
* produce concise errors;
* not assume execution from the repo root unless explicitly documented.

Prefer readable Bash over a large Python bootstrap program.

---

# 37. CI

CI is useful but not required to bootstrap version 1.

A later GitHub Actions workflow should run on a macOS runner and at minimum:

* check Nix formatting;
* run `nix flake check`;
* evaluate/build supported Darwin configuration as practical;
* ShellCheck scripts.

Do not block initial usefulness on building an elaborate CI pipeline.

Local build-before-apply is more important.

---

# 38. `AGENTS.md`

The repository must include `AGENTS.md` because coding agents are likely to modify it.

Recommended contents:

```markdown
# AGENTS.md

This repository declaratively configures personal macOS machines.

## Rules

1. Prefer nix-darwin for macOS/system configuration.
2. Prefer Home Manager for user configuration.
3. Prefer Nixpkgs for CLI software.
4. Prefer Homebrew casks for native GUI applications.
5. Do not install project-specific language versions globally unless explicitly requested.
6. Never put credentials or secrets into Nix expressions or files that enter the Nix store.
7. Prefer typed nix-darwin/Home Manager options over activation shell scripts.
8. Do not change system.stateVersion or home.stateVersion as part of routine upgrades.
9. Do not modify flake.lock except during an intentional dependency update.
10. Do not use both Nix and Homebrew to manage the same package.
11. Keep host-specific configuration under hosts/.
12. Keep common configuration out of host files.
13. Changes must pass ./bin/check before being considered complete.
14. System-affecting changes should also pass ./bin/build.
15. Do not make unrelated workstation preference changes.
16. Update README or docs when a change affects bootstrap or normal usage.

## Design principle

The Git repository is the desired state of the machine.

Prefer explicit, boring configuration over abstractions intended for hypothetical future machines.
```

---

# 39. Required README

The initial `README.md` should approximately contain the following.

---

## README.md

# mac-config

Declarative configuration for my Macs.

This repository manages the reproducible portion of my macOS workstation configuration using:

* nix-darwin
* Home Manager
* Nix/Lix
* nix-homebrew
* Homebrew

The repository is the source of truth for the machine's intended configuration.

## Mental model

Do not configure the Mac imperatively unless experimenting.

Normal change:

```text
edit configuration
      ↓
./bin/check
      ↓
./bin/apply
      ↓
verify
      ↓
git commit
```

If a setting or application should survive replacing this Mac, it probably belongs here.

## What owns what?

| Kind of configuration         | Owner                          |
| ----------------------------- | ------------------------------ |
| macOS settings                | nix-darwin                     |
| system services               | nix-darwin                     |
| shell/user environment        | Home Manager                   |
| dotfiles                      | Home Manager                   |
| CLI packages                  | Nixpkgs                        |
| GUI Mac applications          | Homebrew casks                 |
| Mac App Store applications    | Homebrew/mas                   |
| project Python                | uv                             |
| project Node/runtime versions | mise/project tooling           |
| secrets                       | Keychain/password manager/etc. |
| interactive account state     | manual                         |

## Repository structure

```text
hosts/              machine-specific configuration
modules/darwin/     shared macOS configuration
home/               user configuration
dotfiles/           raw configuration files
bin/                workstation management commands
docs/               architecture and recovery documentation
flake.nix           dependency/composition root
flake.lock          pinned Nix dependencies
```

## New Mac

### 1. Complete macOS setup

Create the primary administrator user and connect the Mac to the internet.

### 2. Install Command Line Tools

```bash
xcode-select --install
```

### 3. Install Lix

```bash
curl --proto '=https' \
  --tlsv1.2 \
  -sSf \
  -L https://install.lix.systems/lix \
  | sh -s -- install
```

Open a fresh terminal after installation if necessary.

### 4. Clone this repository

```bash
mkdir -p ~/dev
cd ~/dev
git clone <REPOSITORY_URL> mac-config
cd mac-config
```

### 5. Bootstrap the host

```bash
./bin/bootstrap <host>
```

For example:

```bash
./bin/bootstrap macbook-pro
```

### 6. Complete manual setup

Follow:

```text
docs/manual-steps.md
```

### 7. Verify

```bash
./bin/doctor
```

The machine is now managed by this repository.

## Everyday use

### Change configuration

Edit the appropriate `.nix` file.

Then:

```bash
./bin/check
./bin/apply
```

Verify the change and commit it:

```bash
git add .
git commit -m "Describe the workstation change"
```

### Add a CLI application

Prefer Nix.

Edit:

```text
home/packages.nix
```

then:

```bash
./bin/apply
```

### Add a GUI Mac application

Edit:

```text
modules/darwin/homebrew.nix
```

and add its cask.

Then:

```bash
./bin/apply
```

Do not normally run `brew install --cask ...` directly.

### Change a macOS preference

Edit:

```text
modules/darwin/macos.nix
```

then:

```bash
./bin/apply
```

Some macOS preferences require restarting an application, logging out, or rebooting before becoming visible.

### Check without applying

```bash
./bin/check
```

For a full system build:

```bash
./bin/build
```

### Update Nix dependencies

```bash
./bin/update
git diff flake.lock
./bin/check
./bin/build
./bin/apply
```

If everything works:

```bash
git add flake.lock
git commit -m "Update Nix dependencies"
```

Dependency updates should be intentional. `./bin/apply` does not update `flake.lock`.

## Installing something temporarily

It is okay to experiment.

For example:

```bash
brew install foo
```

But permanent workstation software must subsequently be added to this repository.

Homebrew drift checking may cause `./bin/apply` to fail until an imperatively installed package is either declared or removed.

This is intentional.

## Secrets

Never put secrets in this repository or directly into Nix configuration.

Do not store:

* passwords;
* API tokens;
* SSH private keys;
* cloud credentials;
* authentication cookies.

Use Keychain, a password manager, SSH agent, or another dedicated credential system.

## Manual configuration

Not everything on macOS can or should be automated.

See:

```text
docs/manual-steps.md
```

for the small remaining interactive setup checklist.

## Troubleshooting

Run:

```bash
./bin/doctor
```

Then see:

```text
docs/recovery.md
```

## Philosophy

The goal is not to declaratively model every byte on the computer.

The goal is that after replacing the Mac I can say:

```text
install Lix
clone repo
bootstrap
```

and recover the workstation configuration I actually care about.

---

# 40. Initial implementation scope

The first implementation should deliberately be small.

It should establish the architecture before attempting to reproduce every current preference.

## Phase 1

Implement:

* flake;
* locked stable inputs;
* nix-darwin;
* Home Manager integration;
* nix-homebrew;
* one host;
* one user;
* basic shell;
* Git;
* a handful of existing CLI tools;
* a handful of known GUI applications;
* `check`;
* `build`;
* `apply`;
* `bootstrap`;
* `doctor`;
* README;
* AGENTS.md;
* manual-steps documentation;
* recovery documentation.

Do not yet spend significant effort on obscure macOS defaults.

## Phase 2

While using the new Mac, encode configuration as needs arise:

```text
I changed this Finder setting
→ codify it.

I installed Ghostty
→ codify it.

I changed my Git configuration
→ codify it.

I configured tmux
→ codify it.
```

This should be the dominant way the repository grows.

## Phase 3

After the configuration is mature:

* consider stricter Homebrew cleanup;
* add CI;
* add a second machine if needed;
* add encrypted secret management only if a real use case appears;
* improve recovery/testing;
* extract reusable modules only when duplication actually exists.

---

# 41. Acceptance criteria

Version 1 is complete when all of the following are true.

### Bootstrap

A newly initialized Mac can be configured from documented steps without needing knowledge that exists only in the author's head.

### Single command

After initial bootstrap:

```bash
./bin/apply
```

is the standard operation for applying both system and user configuration.

### Idempotence

Running:

```bash
./bin/apply
./bin/apply
```

against unchanged configuration does not intentionally upgrade dependencies or produce new desired state.

### Pinning

`flake.lock` exists and is committed.

### Separation

A normal `apply` does not update `flake.lock`.

### User configuration

Home Manager is activated through nix-darwin rather than requiring a second independent command.

### Applications

Native GUI applications can be declared as Homebrew casks.

### Drift

Undeclared Homebrew packages are detected.

### CLI packages

Normal command-line utilities can be declaratively installed through Nix.

### macOS

At least several macOS settings are declaratively represented through nix-darwin.

### Documentation

README explains:

* bootstrap;
* everyday usage;
* adding software;
* applying changes;
* updating dependencies;
* secrets;
* manual steps;
* recovery.

### Safety

No credentials are present in the Git repository or Nix store.

### Agent usability

`AGENTS.md` gives coding agents enough context to modify the repository without introducing a second configuration-management paradigm.

### Extensibility

Adding a second Mac requires a new host configuration rather than restructuring the entire repository.

---

# 42. Design decisions to preserve

These choices are intentional and should not be casually changed during implementation.

**Nix instead of Ansible**

The desired abstraction is machine state, not a sequence of provisioning operations.

**Home Manager integrated with nix-darwin**

There should be one activation lifecycle.

**Nix for CLI; Homebrew primarily for GUI**

This maximizes useful Nix reproducibility without fighting normal macOS application distribution.

**Stable release + `flake.lock`**

The workstation should not follow moving upstream branches merely because an `apply` was run.

**Explicit update operation**

Applying configuration and updating dependencies are different actions.

**Homebrew `cleanup = "check"`**

Detect drift before automatically destroying it.

**Minimal bootstrap script**

The bootstrap mechanism should not itself become a second configuration-management system.

**Manual checklist for inherently manual state**

A five-line honest checklist is better than 200 lines of brittle automation around Apple security mechanisms.

**No project runtime takeover**

`uv`, `mise`, and project lockfiles continue to own project environments.

**No secrets in Nix**

This is a hard security boundary.

**Grow iteratively**

The repository should become an accurate description of the workstation through normal use, not through an enormous one-time reverse-engineering exercise.

---

# 43. Intended steady-state experience

Several months from now, the repository should make interactions like these mundane.

> Install a new CLI.

```bash
$EDITOR home/packages.nix
./bin/apply
git commit -am "Install ..."
```

> Install a Mac application.

```bash
$EDITOR modules/darwin/homebrew.nix
./bin/apply
git commit -am "Install ..."
```

> Change Finder behavior.

```bash
$EDITOR modules/darwin/macos.nix
./bin/apply
git commit -am "Configure Finder ..."
```

> Upgrade the underlying system configuration ecosystem.

```bash
./bin/update
./bin/check
./bin/build
./bin/apply
git add flake.lock
git commit -m "Update Nix dependencies"
```

> Buy another Mac.

```text
Install Lix
Clone repository
Add/select host
Run bootstrap
Complete manual checklist
```

That experience—not the quantity of Nix code—is the actual product this repository is intended to create.
