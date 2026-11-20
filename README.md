# Core Desktop knowledge base

Evidence-based research and design proposals about Ubuntu Core Desktop:
login, session startup, confinement, application launching, and integration
with the standard `core26` boot base instead of `core26-desktop`.

This repository contains analysis, not the implementation of the desktop or
snapd. Start with [session-startup research](research-session-startup.md) for
the inspected boot-to-desktop path, or
[base-integration research](research-base-integration.md) for what the desktop
base supplies. Proposed replacements are in the separate proposal group.

## Research and findings

Research documents own observations, source explanations, labelled inferences,
inspection limits and open research questions.

| Document | What it covers |
| --- | --- |
| [research-session-startup.md](research-session-startup.md) | Snap roles, boot to greeter, user-session launch, initial wrapper environment, the environment broker, policy mismatch and existing pre-session ordering. |
| [research-session-login.md](research-session-login.md) | GDM, PAM, logind, UID/GID changes, session-leader lifetime, VT/device handoff, SDDM comparison and existing snapd policy gaps. |
| [research-session-services-model.md](research-session-services-model.md) | User-systemd startup through snap entrypoints, unit overrides, the content dispatcher, runtime paths, shared per-user state, trust and desktop-version coupling. |
| [research-desktop-launch.md](research-desktop-launch.md) | Desktop-entry broker, direct `snap-confine` access, `StartTransientUnit` permissions, environment-based routing, review findings and VM evidence. |
| [research-base-integration.md](research-base-integration.md) | Source/payload inventories, writable paths, account facilities, PAM and host-resource registration, current composition mechanisms and their limits. |

## Proposals

These are under discussion, not selected or implemented contracts. Each links
to the research supporting it; alternatives and validation still needed remain
explicit. A proposal may depend on another proposal without either being an
available feature.

| Document | Proposed change |
| --- | --- |
| [proposal-desktop-launch.md](proposal-desktop-launch.md) | Broker-only launching, a dedicated client, permission changes, helper packaging alternatives and validation. |
| [proposal-session-setup.md](proposal-session-setup.md) | Provider registration, early per-login user-systemd preparation and later runtime publication, including candidate generic base tooling. |
| [proposal-login-manager.md](proposal-login-manager.md) | A superprivileged `login-manager` interface with a confined session leader, required integration, GDM/SDDM adaptations and validation. The name and API are not implemented contracts. |
| [proposal-base-integration.md](proposal-base-integration.md) | Host registration and composition without OverlayFS, interface alternatives, account/state ownership, login-provider packaging and boot/refresh/rollback requirements. |

## Topic shortcuts

- **Authentication and changing Unix user:**
  [credential transition and confinement](research-session-login.md#credential-transition-and-confinement).
- **Dedicated greeter accounts and SDDM:**
  [service identities and comparison](research-session-login.md#service-identities-and-the-sddm-comparison).
- **Environment setup and why the broker exists:**
  [initial setup and later updates](research-session-startup.md#session-environment-initial-setup-and-later-updates).
- **Replacing the wrapper with systemd integration:**
  [generic session-setup proposal](proposal-session-setup.md),
  including the preparation gate and what could live in the base.
- **What belongs in `login-manager`, and what stays outside it:**
  [scope and dependencies](proposal-login-manager.md#scope-and-dependencies).
- **User units, Wayland/audio sockets and Xauthority paths:**
  [runtime paths and their creators](research-session-services-model.md#runtime-paths-who-creates-them).
- **SSH, linger and multiple logins sharing one user manager:**
  [per-user runtime versus graphical sessions](research-session-services-model.md#per-user-runtime-versus-individual-graphical-sessions).
- **Application-launch authority and confinement risks:**
  [desktop-launch findings](research-desktop-launch.md#findings).
- **Moving desktop integration off the boot base:**
  [observed locations](research-base-integration.md#existing-locations-and-consumer-paths),
  [candidate integration approaches](proposal-base-integration.md#location-by-location-integration-candidates)
  and [composition options](proposal-base-integration.md#discussed-design-options).
- **Earlier discussion prompts:**
  the former `topics.md` questions now live in
  [desktop-launch research](research-desktop-launch.md#related-proposal-and-open-questions),
  [helper packaging questions](proposal-desktop-launch.md#packaging-questions-still-open)
  and the base-integration proposal above.

## Evidence and how to read the notes

Research records source repositories, branches/commits, inspection dates and,
where relevant, VM/snap versions. Observation and source-based inference are
distinguished. Proposals link to that evidence and record their status,
assumptions and validation limits; short evidence summaries are not a second
source of truth.

A source inspection or an enforcing AppArmor label is not proof that a complete
login, confinement boundary or refresh path has been tested. References to the
"current" VM or implementation describe the cited snapshot, not necessarily the
latest upstream state. Read the evidence and limits before applying a finding
elsewhere.

The retained [base-reconciliation inventory tables](evidence/reconcile-with-base-2026-11-20/)
cover base payload destinations, other base-source changes, snapd-source changes
and desktop-only VM paths. They remain dated artifacts, unchanged by the document
split. Their boundaries and limitations are in
[the inventory research](research-base-integration.md#comparison-boundaries-and-retained-evidence).
In particular, `base-payload.tsv`'s `extension_class` is a proposed integration
assessment, not an observed capability or an existing snapd API; origin
classifications are source attributions rather than raw observations.

[AGENTS.md](AGENTS.md) defines the naming and working rules. Update the relevant
existing research or proposal, preserve provenance, and update or mark
superseded claims rather than leaving conflicting accounts.
