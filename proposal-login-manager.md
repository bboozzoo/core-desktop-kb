# Login-manager: a strict login provider with a confined session leader

## Status and evidence

**Status: under discussion, not selected.** Design notes from 2026-11-20.
`login-manager` is a **working name for a proposed superprivileged interface**,
not an existing snapd API. No metadata
schema, implementation or store declaration is specified here. The proposal
assumes stock `core26` as the boot base and strictly confined login-manager
apps, not classic confinement or a host daemon merely executing snap content.

This document owns the proposed contract, including the confined-leader model
previously discussed alongside the login research. Current behavior and
source-policy gaps are recorded in
[research-session-login.md](research-session-login.md). Its VM observations were read-only on
2026-11-19/20, host `ubuntu-Standard-PC-Q35-ICH9-2009` at
`ubuntu@vsock%59467`, snapd `2.76.3+g78dca3a.78dca3a`. This proposal involved
**no new VM inspection or active login test**. No strict GDM/SDDM prototype,
stock-base deployment, logout, refresh or rollback was tested.

Source references below use these revisions; upstream release sources were
not matched to the installed GDM/GNOME binaries or a KDE image:

| Reference | Repository, branch/tag, commit | Relevant files |
| --- | --- | --- |
| GDM | `GNOME/gdm`, tag `50.0`, `7aa5c1a3d73b51b9ccf89c51d33bfa53cc57d52e` | `daemon/gdm-session{,-worker}.c` |
| GNOME Session | `GNOME/gnome-session`, tag `50.0`, `9d3b90ab0de1a3b479f8628446ac3e5b08cd71c1` | `gnome-session/{leader-main,leader-systemd,service-main}.c`, `tools/gnome-session-ctl.c` |
| Base | `canonical/core-base-desktop`, `desktop/26`, `fea653f29ef06a43d10460c788f0a8aa01687a39` | `variants/desktop/static/etc/apparmor.d/gdm`, `etc/pam.d/gdm-*`, `usr/bin/core-desktop-session-wrapper.sh`, `etc/systemd/system/gdm.service.d/50-gnome-content.conf`; paths after the first share the `variants/desktop/static/` prefix |
| Image | `canonical/ubuntu-core-desktop`, `main`, `cc7069d64915dbd0f8d7cb8c97e2ba410745225a` | `build.sh:kde_sddm_integration()` |
| Session snap | `canonical/ubuntu-desktop-session-snap`, `desktop/26`, `2697819d8c35678b5b1d9494c99b24f36d218f63` | `scripts/run-session.sh`, `snap/snapcraft.yaml`, `patches/gnome-session-environment-broker.patch` |
| Snapd | `canonical/ubuntu-core-desktop-snapd`, `desktop/26`, `78dca3acd0dcc04b738a24b07e9dae7c7dc66c72` | `interfaces/builtin/{gdm,login_session_control,systemd_user_control,desktop_launch,wayland}.go`, `interfaces/{apparmor,seccomp}/{backend,template}.go`, `snap/{system_usernames,validate}.go`, `snap/snapenv/snapenv.go`, `overlord/snapstate/check_snap.go`, `osutil/user.go` |
| SDDM | `sddm/sddm`, tag `v0.21.0`, `63780fcd79f1dbf81a30eef48c28c699ab15aded` | `src/daemon/Greeter.cpp`, `src/helper/{HelperApp,UserSession}.cpp`, `services/sddm.service.in` |
| External policy | `roddhjav/apparmor.d`, `main`, `fcf47c7111e3b4bd4c2c083ce603850415352dd2` | `apparmor.d/groups/kde/{sddm,sddm-greeter,sddm-xsession}` |

## Proposal: login-manager

### Purpose and trust

Authorize a selected login provider to authenticate users, register their
logins, establish their Unix credentials, and coordinate desktop startup
and shutdown **while its own session leaders remain confined**. The host
logind and user managers remain independent services; desktop units launch
their respective snaps through normal snap entrypoints.

**Proposed interface shape:** a plug on the privileged login-manager app(s),
connected to a system-provided slot on Core. Installation/connection needs
explicit privileged authorization; it should not be automatically available
to arbitrary publishers or every app in the provider snap. Exact model/store
approval policy and metadata are undecided. Classic-host support is not part
of the initial proposal.

This is security-sensitive OS integration, not an ordinary desktop-client
permission. The provider handles authentication secrets and may establish
arbitrary authenticated users' credentials. It is trusted to bind the target
identity and selected session to a successful PAM exchange, or to an explicitly
authorized autologin policy. AppArmor/seccomp cannot enforce "setuid only
after PAM succeeded". Confinement limits available operations; it does not
make a compromised authentication provider trustworthy.

The greeter UI should not receive the privileged plug merely because it
belongs to the same snap. Dropping UID does not create a separate AppArmor
profile: any stronger greeter/worker separation needs explicit policy and
IPC authorization. The retained session leader may inherit the privileged
app's sandbox in the initial design; a narrower descendant profile is an
option, not a prerequisite or a fresh seccomp sandbox.

### Execution model

The proposed GNOME sequence follows the
[recorded GDM/GNOME division of work](research-session-login.md#session-leader-and-the-user-service-graph):

1. The root, confined GDM worker runs the PAM conversation/account checks,
   establishes the target GID and supplementary groups, and opens the PAM
   session while still root.
2. `pam_systemd` registers the login with logind. Logind arranges the session
   scope/runtime directory and starts or reuses `user@UID.service` through
   the system manager.
3. GDM forks; its child changes UID and launches the Wayland/session helper.
   GNOME's leader/init-worker executes **within login-manager confinement**.
4. That leader asks the independent user manager to start the registered
   GNOME target. The user manager launches desktop units through snap app
   entrypoints, establishing each app's own sandbox.
5. The leader stays alive for GNOME's monitor/shutdown protocol. GDM watches
   its session child and closes the PAM session during cleanup.

```text
system systemd
├─ login-manager snap daemon                         root, confined
│  ├─ greeter path                                   dedicated user, confined
│  └─ authentication worker                          root, confined
│     ├─ PAM authentication/account/credential work
│     ├─ pam_open_session() → pam_systemd → logind
│     └─ fork; child changes UID and executes session helper
│        └─ GNOME session leader / init-worker        login user, confined
│           ├─ request user-systemd target startup
│           └─ coordinate session shutdown and child lifetime
└─ user@UID.service: systemd --user                   login user, unconfined
   └─ registered desktop units → snap entrypoints     each app's confinement
```

The diagram is schematic: greeter services can also be started through
the greeter's independent user manager. It does not prescribe that every
UI process be a direct child of the daemon or share its privileged profile.

Sources: GDM and GNOME Session files listed above; observed process/service
separation and FIFO evidence in [research-session-login.md](research-session-login.md).
`gnome-session-init-worker` belongs to GNOME Session, not GDM.

**Architectural inference:** neither changing UID nor registering a logind
session inherently requires an AppArmor transition to `unconfined`. The
leader retains its AppArmor/seccomp restrictions and mount namespace. IPC
does not transfer those restrictions or the caller's mount namespace to
independently launched user services.
The leader therefore need not execute another snap's launcher or shed an
inherited seccomp filter. Executable content can be packaged or supplied by
a runtime, but must be available and executable inside the provider namespace.

**Not a clean-slate guarantee:** an existing SSH login, graphical login or
linger can keep the user manager alive. Its environment and services may
already exist. Logind does not unconfine a GDM child by registering it or
placing it in a session scope. Independent service creation is the relevant
boundary, not a sandbox-reset operation.

### Scope and dependencies

| Concern | Proposed responsibility or boundary |
| --- | --- |
| PAM and Unix identity changes | Core authority of `login-manager`: required policy for authentication, account lookup, credential changes and session open/close. |
| Login registration, seat/VT handling and lifecycle IPC | Core authority, with permissions limited to the supported login workflows and explicit peers where possible. |
| Confined desktop bootstrap | Permit the selected bootstrap and its runtime resources; keep its lifetime tied to the login. No required unconfined exec or direct nested snap launch. |
| Greeter account provisioning | Required integration dependency. Use a trusted Core-compatible registrar, not general writes to account databases from the daemon. Whether owned by this interface's backend or a companion contract is open. |
| PAM filesystem view | Required packaging/mount dependency. Prefer provider-scoped policy composition; no implicit host-wide `/etc/pam.d` replacement. |
| Login-manager service selection | Required snapd/system integration: generated daemon unit, startup ordering, display-manager selection and lifecycle. Not arbitrary provider-supplied root units. |
| Desktop entries and user-unit registration | Required companion integration with explicit ownership, completed before the desktop startup transaction. Making files visible to GDM does not register them with the host user manager. |
| Early session setup and runtime publication | Proposed companion user services: a generic base-owned preparation helper, provider-owned configuration/hooks and later producer updates. Give the login manager a bounded context/start handoff, not all of the current wrapper's setup powers. |
| Greeter/compositor graphics, audio and device access | Reuse applicable interfaces; do not make `login-manager` a blanket substitute for graphics/device policy. |
| General desktop application launching | Separate concern; this is not a launch API for untrusted clients or a reason to grant unrestricted `snap-confine` execution. |
| Generic host extension, account creation UI and arbitrary authentication backends | Not implicitly authorized. Provisioning/first-boot, optional authentication methods and host-resource changes require their own scope decisions. |

The wider stock-base registration/composition problem is described in
[research-base-integration.md](research-base-integration.md). Its possible
solutions and the [no-OverlayFS constraint](proposal-base-integration.md#constraint-no-overlayfs)
are part of the base-integration proposal; that constraint still applies here. An interface may authorize operations across multiple snapd
backends; an AppArmor snippet alone does not provision accounts, publish
user units, construct a PAM view or select the active display manager.

## Required permissions and integration

The following are proposed requirements, not a ready-to-copy allowlist.

### Accounts, credentials and PAM

- Resolve the dedicated greeter identity and arbitrary login identities
  consistently through the intended host/Core NSS view. Keep account names,
  numeric IDs, groups and state ownership stable across refresh/rollback.
- Permit the root worker's required UID/GID operations and supplementary-group
  setup in **both AppArmor and seccomp**. Merely clearing groups is insufficient
  for GDM's `initgroups()` or SDDM's `setgroups()` path. Account for the
  existing declared-user/root seccomp generation when supplying broader
  login-worker permissions.
- Make provider PAM service files, selected boot-base common policy,
  ABI-compatible modules, libraries and helpers resolve coherently in the
  snap namespace.
  The app's base/filesystem view must not silently authenticate against a
  different account database or obsolete common policy.
- Authorize the selected modules' account/secret access, authentication state,
  limits/audit/keyring operations and helper execution as needed. Do not copy
  all capabilities from the host GDM profile without validating their use.
- Preserve authentication failure, account rejection and session-open failure
  behavior; no desktop launch after a failed required step. Define password,
  greeter and explicitly enabled autologin paths separately.

**Research basis:** see [accounts and credential policy](research-session-login.md#accounts-and-credential-policy)
for the existing support, source references and limits. Those findings do not
establish this proposed contract.

### Logind, seat access and provider IPC

- Permit PAM's actual session-registration and release transport for the
  supported systemd build, whether D-Bus or its local socket protocol.
  Do not assume that allowing public login1 D-Bus methods covers every path.
- Supply the required seat/session metadata, VT access and ioctl/capability
  permissions. GDM's inspected new-VT path allocates a VT before PAM session
  registration and activates it before launching the session command.
- Cover provider bus-name ownership, system-bus policy, greeter/worker private
  sockets, peer authentication, signals, child status and orderly cleanup.
  Allowing a private socket does not validate its application protocol.
- Keep compositor session-control/device acquisition distinct from the root
  worker's login duties; validate the independently launched compositor can
  locate and control the intended logind session.

**Research basis:** see [logind and GDM IPC](research-session-login.md#logind-and-gdm-ipc)
for the existing support, source references and limits. A snap provider needs
compatible provider/client policy for its actual snap label and bus registration.
Those findings do not establish this proposed contract.

### User-manager startup and authorization

The confined leader needs access to the target user's bus/manager, target
startup and state queries/subscriptions, and the desktop's shutdown protocol.
GNOME's inspected leader uses `GetUnit`, `StartUnit`, `ResetFailed` and unit
properties/signals; the required set must be checked against the packaged
version. Environment publication needs explicit ownership and limits, not
necessarily direct shared-environment writes from the leader; the proposed
setup handoff below would keep those operations in companion services.

**Trust boundary:** allowing a D-Bus method does not constrain its arguments.
`StartTransientUnit` can create command-bearing services, not just tracking
scopes. `StartUnit` does not itself enforce an approved target name; writable
unit files and unsafe environment updates can likewise cause independent
unconfined execution. Direct user-manager socket access must be considered
alongside bus-mediated access, not treated as an equivalent method filter.

The proposal trusts the login provider with session control; it does **not**
claim that keeping its AppArmor label prevents every indirect execution path.
Before implementation, decide whether broad authenticated-user session control
is an accepted superprivilege, or whether an approved-session-only contract
requires additional authorization/registration enforcement. Do not advertise
the latter on the strength of AppArmor D-Bus rules alone. A new broker is not
assumed necessary unless that narrower contract is selected.

**Research basis:** see [user-manager control](research-session-login.md#user-manager-control)
for the existing support, source references and limits. Those findings do not
establish this proposed contract.

### Runtime environment and session lifetime

A root-started snap later changing UID does not obtain a new user-specific
snap environment or namespace. Snapd constructs `HOME`, `SNAP_USER_*` and
`XDG_RUNTIME_DIR` at launch (`snap/snapenv/snapenv.go`). GDM's normal environment
handling must therefore be checked, not assumed to repair those values or
remove all snap-specific state before publication to user systemd.

Define runtime paths, ownership and environment for both the greeter and each
login user. The confined leader and independently launched monitor must agree
on the GNOME FIFO path: today's observed path is under
`/run/user/1000/snap.ubuntu-desktop-session/`, which cannot simply be presumed
to become the new login-manager path on both sides. Cover socket/FD passing,
EOF/hangup, signals, failed startup and PAM cleanup without retaining stale
credentials or root-only descriptors in the child.

The current `core-desktop-session-wrapper.sh` prepares manager/activation
environments, installs the runtime user-unit link, reloads the manager and
sets up socket links before entering the session snap. Sources: Base wrapper,
Session snap launcher/patch, GNOME Session leader/monitor.

**Proposed separation:** use provider registration before the startup
transaction, per-login preparation through a generic base-owned user service,
and runtime publication after the relevant compositor/audio producer is
ready. The detailed [session-setup proposal](proposal-session-setup.md#proposed-separation)
covers `graphical-session-pre.target`, the existing migration-unit precedent,
GNOME ordering and the Xauthority wait that cannot block compositor startup.

Under that proposal, `login-manager` retains authentication, credential and
logind operations, a bounded session-context/start handoff and lifecycle IPC.
Unit installation, broad manager-environment mutation, blanket recovery stops
and Xauthority polling stay outside its general app permissions. The leader
still needs its own local environment/runtime setup. A user-systemd helper
does not inherit or rewrite that already-running process's environment.

Base tooling would interpret validated data, not execute arbitrary provider
scripts unconfined. Desktop-specific hooks can run as confined snap apps.
This separation is not implemented; the handoff, provider registration,
per-login reset and failure/cleanup contract still need validation.

### Service, cgroup and refresh lifecycle

Prefer normal snap daemon registration over exporting a raw `gdm.service`
that executes outside snap confinement. Define display-manager selection,
conflicts with the existing manager/getty, seat availability, and startup
after account, PAM, runtime and security-policy preparation. Reuse existing
registration mechanisms where possible; no final unit/alias API is proposed.

Validate what logind session-scope placement does to snap process tracking
and device-cgroup enforcement. Retaining AppArmor/seccomp is not proof that
all snap restrictions remain intact after cgroup changes. Whether the current
snapd mechanisms already suffice is **unverified**.

Stopping, disconnecting, refreshing or removing the provider must account for
long-lived session workers, the separate desktop units and shared user
managers. Define admission of new logins, draining/restart or reboot policy,
revision retention and rollback. Preserve console/SSH recovery and avoid
terminating unrelated sessions when cleaning up this graphical login. These
are lifecycle requirements, not claims about existing snap refresh behavior.

## Accommodating GDM and SDDM

### GDM: compatible process model, different integration

**Research basis:** the [recorded credential transition](research-session-login.md#credential-transition-and-confinement)
retains the host `gdm` label while changing UID, but includes later `Ux`
transitions. It is not evidence of an uninterrupted strict-snap handoff.

**Compatibility with GDM:** the existing sequence already registers the
login through PAM **before** launching the user's session command. The
GNOME init-worker later starts the desktop target through **user systemd**;
it does not normally create the logind session. It belongs to GNOME Session,
not GDM. Its persistent leader/monitor protocol lets GDM continue watching
its session child and perform cleanup when that child exits. No fundamental
change to that lifecycle follows from keeping the child confined. This is
source-based compatibility reasoning, not a claim that unmodified GDM and
GNOME Session already work in the proposed packaging.

**Proposed adaptation:** retain GDM's PAM/open/fork/watch/cleanup lifecycle,
but execute the GNOME bootstrap within the login-manager sandbox rather
than using the unconfined wrapper to enter the desktop-session snap. Package
or mount the matching executable/resources; adapt the current launcher and
its snap-specific path/environment assumptions. Continue launching actual
desktop services independently through registered user units.

Audit every helper and the greeter path, not only the ordinary user login.
The current greeter leader also enters a snap through a launcher, and the
initial-setup path deliberately runs unconfined. Initial setup, reauthentication,
optional PAM methods and keyring startup need explicit support decisions;
no unconfined fallback should silently stand in for an unsupported path.
No claim is made that this requires no GDM/packaging patches. The existing
architecture fits; compatibility of unchanged binaries is untested.

### SDDM: same authority categories, desktop-specific bootstrap

**Research basis:** the [SDDM comparison](research-session-login.md#service-identities-and-the-sddm-comparison)
records its greeter, root authentication helper and supplementary-group/UID
transition. These fit the same authority categories, not a verified strict
SDDM implementation.

**Proposed accommodation:** do not encode GNOME executable names, target names
or FIFO protocols as the interface's universal model. SDDM's selected session
script and Plasma bootstrap need their own executable/environment, bus,
state/log and lifecycle review. The Image repository currently copies SDDM
PAM files, creates accounts and generates a host service at image-build time;
that is integration to replace, not existing strict-snap support. No complete
SDDM/Plasma service graph was validated for this proposal.

The [SDDM policy comparison](research-session-login.md#service-identities-and-the-sddm-comparison)
records the absence of a local profile and the external profile's `PUx`
fallbacks and dependency on its own abstractions. It is not a ready snapd policy.

## Validation before claiming support

Proposed tests require separate approval for active work:

1. **Stock-base preparation:** account/NSS consistency, PAM common-policy and
   module resolution, runtime content, service ordering and registered desktop
   units without desktop-specific boot-base files or OverlayFS.
2. **Authentication and credentials:** greeter startup, rejected/accepted
   password login, account/session-open failures and enabled autologin. Record
   real/effective/saved UIDs/GIDs, supplementary groups and capabilities through
   the transition; verify secrets and privileged FDs are not leaked.
3. **Continuous confinement:** record labels, seccomp state and namespaces
   across every worker/helper exec, not only the final leader. Verify there is
   no unconfined exec/fallback and no required nested `snap run` in this path.
   Confirm desktop services start independently with their own snap policies.
4. **Session identity and access:** logind scope/seat/VT metadata, device access
   and cgroup enforcement/tracking. Check both new and pre-existing user managers
   (SSH/linger), concurrent logins and isolation of runtime/IPC state by user.
5. **Authorization boundary:** exercise the chosen user-manager trust contract,
   including transient services, unit/environment mutation, direct sockets,
   greeter-to-worker requests and malformed or unauthorized session selections.
6. **Lifecycle and failure:** startup failure, leader/monitor disappearance,
   logout/relogin, failed cleanup, provider stop/crash, disconnect/removal,
   refresh/rollback and missing runtime. Confirm unrelated logins and recovery
   remain usable, and record any explicitly accepted session interruption.
7. **SDDM separately:** repeat the relevant gates with SDDM and its selected
   desktop. A GNOME result does not verify Plasma, X11, remote login or optional
   authentication mechanisms.

**Viability assessment:** the GDM/systemd-GNOME process model supports this
separation in principle. A superprivileged strict snap can retain its session
leader while independently launched desktop services enter their own sandboxes.
The remaining work is policy, packaging, registration and lifecycle integration;
none of those requirements is discharged by preserving an AppArmor label alone.

## Relationship to the host-managed alternative

The earlier base-integration recommendation retained a host-side login worker
as a lower-change starting point. It is an alternative, not a requirement
imposed by GDM, and not superseded by a decision to implement this proposal.
The [base-integration discussion](proposal-base-integration.md#login-provider-packaging-alternatives)
compares those choices and owns PAM composition and host-resource registration.
Both remain untested on stock `core26`; no architecture has been selected.
