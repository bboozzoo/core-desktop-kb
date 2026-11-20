# Core Desktop knowledge base

Evidence-based notes and design proposals about Ubuntu Core Desktop:
login, session startup, confinement, application launching, and integration
with the standard `core26` boot base instead of `core26-desktop`.

This repository contains analysis, not the implementation of the desktop or
snapd. Start with [session-startup.md](session-startup.md) for the inspected
boot-to-desktop path, or [reconcile-with-base.md](reconcile-with-base.md) for
the work involved in removing the desktop-specific base.

## Documents

| Document | What it covers |
| --- | --- |
| [session-startup.md](session-startup.md) | Snap roles, boot to greeter, launch of the user session, the wrapper's initial environment setup and the environment broker. Also contains the proposed split into provider registration, early user-systemd preparation and runtime publication, including generic base-owned setup tooling. |
| [session-login.md](session-login.md) | GDM, PAM, logind, UID/GID changes, session-leader lifetime and VT/device handoff. Compares SDDM and describes the proposed strictly confined login manager with a confined session leader. |
| [session-services-model.md](session-services-model.md) | How user systemd starts desktop components through snap entrypoints; unit overrides, the content dispatcher, runtime paths, shared per-user state and desktop-version coupling. |
| [desktop-launch.md](desktop-launch.md) | Launching applications from confined desktop components. Reviews the desktop-entry broker, direct `snap-confine` access, `StartTransientUnit` permissions and environment-based confinement detection. |
| [reconcile-with-base.md](reconcile-with-base.md) | What the desktop base supplies, source/payload inventories, writable paths, account provisioning, PAM and host-resource registration. Compares composition options without OverlayFS and records boot/refresh/rollback questions. |
| [snapd-interfaces.md](snapd-interfaces.md) | The proposed superprivileged `login-manager` interface: scope, trust, required permissions, companion integration, GDM/SDDM requirements and validation still needed. The name and API are not implemented contracts. |
| [topics.md](topics.md) | Short discussion prompts and open questions. Use the detailed documents for findings and proposal status. |

## Topic shortcuts

- **Authentication and changing Unix user:**
  [credential transition and confinement](session-login.md#credential-transition-and-confinement).
- **Dedicated greeter accounts and SDDM:**
  [service identities and comparison](session-login.md#service-identities-and-the-sddm-comparison).
- **Environment setup and why the broker exists:**
  [initial setup and later updates](session-startup.md#session-environment-initial-setup-and-later-updates).
- **Replacing the wrapper with systemd integration:**
  [generic session-setup proposal](session-startup.md#proposal-generic-session-setup-through-user-systemd),
  including `graphical-session-pre.target` ordering and what could live in the base.
- **What belongs in `login-manager`, and what stays outside it:**
  [scope and dependencies](snapd-interfaces.md#scope-and-dependencies).
- **User units, Wayland/audio sockets and Xauthority paths:**
  [runtime paths and their creators](session-services-model.md#runtime-paths-who-creates-them).
- **SSH, linger and multiple logins sharing one user manager:**
  [per-user runtime versus graphical sessions](session-services-model.md#per-user-runtime-versus-individual-graphical-sessions).
- **Application-launch authority and confinement risks:**
  [desktop-launch findings](desktop-launch.md#findings).
- **Moving desktop integration off the boot base:**
  [location-by-location assessment](reconcile-with-base.md#location-by-location-extensibility-assessment)
  and [composition options](reconcile-with-base.md#discussed-design-options).

## Evidence and how to read the notes

Each document records its source repositories, branches/commits, inspection
dates and, where relevant, VM/snap versions. Observed behavior, source-based
explanations and proposed changes are distinguished. A source inspection or
an enforcing AppArmor label is not proof that a complete login, confinement
boundary or refresh path has been tested.

References to the "current" VM or implementation describe the cited snapshot,
not necessarily the latest upstream state. Read the evidence and limits
sections before applying a finding elsewhere. Proposal sections describe
work to evaluate, not features already available in stock Core or snapd.

The retained [base-reconciliation inventory tables](evidence/reconcile-with-base-2026-11-20/)
cover base payload destinations, other base-source changes, snapd-source
changes and desktop-only VM paths. Their comparison boundaries and limitations
are documented in
[reconcile-with-base.md](reconcile-with-base.md#comparison-boundaries-and-retained-evidence).

[AGENTS.md](AGENTS.md) lists the source repositories and repository working
rules. Keep new findings in the relevant existing document, cite the evidence,
and update or mark superseded claims rather than leaving conflicting accounts.
