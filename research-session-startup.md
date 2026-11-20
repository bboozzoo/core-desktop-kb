# GNOME login startup on the inspected Core Desktop VM

## Evidence and scope

Read-only inspection on 2026-11-19 of `ubuntu@vsock%59467`
(`ubuntu-Standard-PC-Q35-ICH9-2009`), with snapd
`2.76.3+g78dca3a.78dca3a`: `snap list`, `snap connections`, installed
`meta/snap.yaml`, `systemctl cat/show`, `findmnt`, mountinfo, `ps` with
AppArmor labels, `aa-status`, `loginctl`, and installed scripts/units. The VM's
`core26-desktop`, `gnome-desktop-runtime`, and `ubuntu-desktop-session` snaps
were all revision `x1`. Inspected source (read-only):

- `canonical/core-base-desktop`, `desktop/26`,
  `fea653f29ef06a43d10460c788f0a8aa01687a39`, under
  `variants/desktop/static/` (paths cited below); the GDM profile, drop-in,
  host `gnome-session`, content launcher and session wrapper match the VM's
  installed files by SHA-256.
- `canonical/ubuntu-desktop-session-snap`, `desktop/26`,
  `2697819d8c35678b5b1d9494c99b24f36d218f63`, `snap/snapcraft.yaml`;
  the installed app commands and content target were checked against
  `/snap/ubuntu-desktop-session/current/meta/snap.yaml`.
- No separate source checkout for `gnome-desktop-runtime` was inspected;
  its app and slot definitions below come from the VM's installed
  `/snap/gnome-desktop-runtime/current/meta/snap.yaml`.

A **2026-11-20 read-only source follow-up** examined environment setup and
user-service integration:

- At the same base revision: the session wrapper above,
  `variants/desktop/static/usr/lib/systemd/user/user-session-migration.service`,
  its `core-override.conf` drop-in and `graphical-session-pre.target.wants`
  link, and `hooks/050-snap-symlink.chroot`.
- At the same session-snap revision: `environment-broker/{broker,main}.go`,
  `patches/gnome-session-environment-broker.patch`, `snap/snapcraft.yaml`
  and `README.md`.
- `canonical/ubuntu-core-desktop-snapd`, `desktop/26`,
  `78dca3acd0dcc04b738a24b07e9dae7c7dc66c72`:
  `interfaces/builtin/{session_environment_broker,systemd_user_environment,
  systemd_user_control}.go` and `data/systemd-env/990-snapd.conf.in`.
- `GNOME/gnome-session`, tag `50.0`,
  `9d3b90ab0de1a3b479f8628446ac3e5b08cd71c1`:
  `data/gnome-session-pre.target` and `data/gnome-session-manager@.service.in`.
- `GNOME/gnome-shell`, tag `50.0`,
  `aecaec6ad79e5f77e19d590d746d0f4c06825420`:
  `data/org.gnome.Shell@.service.in`.

That follow-up did not inspect the VM again, trace broker calls, match the
running broker binary to source, or recheck live generated AppArmor policy.
The tagged GNOME units were not matched to the installed runtime's complete
unit graph. The environment and ordering findings below distinguish this
source evidence from the earlier VM observations; the proposed setup helper
and dependency changes were not implemented or tested.

The VM evidence describes the *running greeter and logged-in `ubuntu`
session*, not a newly performed login. The initial-setup path, PAM
authentication sequence, logout/relogin, refresh and profile reload were not
actively traced or tested.
For the PAM/logind handoff, host comparison and implications of replacing the
desktop base, see [research-session-login.md](research-session-login.md).

## Snap roles

| Snap | Observed responsibility |
| --- | --- |
| `core26-desktop` (base) | Mounted read-only as `/`. Supplies `/usr/lib/systemd/system/gdm.service`, `/usr/bin/gnome-session`, `/usr/bin/core-desktop-session-wrapper.sh`, `/usr/libexec/gnome-content-launch`, `/etc/apparmor.d/gdm`, and desktop-specific systemd overrides. It supplies integration and GDM policy, not the GDM executable used here. |
| `gnome-desktop-runtime` (app/content provider, `base: core26`) | Supplies the GNOME binaries and data, including `usr/sbin/gdm3`, `usr/libexec/gdm-{session-worker,wayland-session}`, `usr/bin/gnome-session`, Shell, and user unit files. Exposes the whole tree as a `gnome-desktop-runtime` content slot; *also* declares its own strictly confined `ubuntu-desktop-session` app, running `usr/bin/gnome-session` for the greeter. GDM itself is executed as host-managed content, not this snap app. |
| `ubuntu-desktop-session` (app/content consumer, `base: core26`) | Its connected content plug mounts the runtime at `$SNAP/gnome` **inside its snap mount namespace** (confirmed in the live Shell's mountinfo). Defines the user session, Shell and many service apps and their interfaces. Its own squashfs includes launch scripts, `usr/bin/gnome-session`, and native helpers such as `gnome-session-service`, `gnome-session-init-worker` and `session-environment-broker`. Most other GNOME app commands use the runtime's payload through the content mount; it is thin, not empty. |

The runtime-to-session content connection was present (`snap connections
ubuntu-desktop-session`, marked `gadget`). The host's ordinary view of
`/snap/ubuntu-desktop-session/current/gnome` does not itself show the mounted
payload: that mount was confirmed in the confined Shell's mount namespace.

## Boot to greeter

1. The root filesystem is the `core26-desktop_x1.snap` squashfs. The base's
   `/usr/lib/systemd/system/apparmor.service` reloads the system profiles;
   `aa-status` showed `gdm` loaded and enforced. This confirms the live label,
   not a syscall-level trace of precisely when it was loaded.
2. The effective `gdm.service` combines the base's
   `usr/lib/systemd/system/gdm.service` with
   `etc/systemd/system/gdm.service.d/50-gnome-content.conf`. The latter sets
   `AppArmorProfile=gdm` and replaces `ExecStartPre` and `ExecStart` with
   `/usr/libexec/gnome-content-launch --content` pointing at the runtime's
   `usr/share/gdm/generate-config` and `usr/sbin/gdm3`, respectively.
   `--content` adjusts environment and execs the specified binary; it does
   **not** enter a snap-app sandbox. Live `gdm3`, `gdm-session-worker` and
   `gdm-wayland-session` all had `gdm (enforce)` labels. The base's
   `etc/apparmor.d/gdm` has `ix` rules for those runtime-snap executables.
3. For the greeter, `gnome-content-launch` rewrites the
   `gdm-wayland-session` argument to
   `--session-dispatch gnome-login-session --session=gnome-login` when the
   session is `gnome-greeter` (source: `usr/libexec/gnome-content-launch`).
   The running greeter helper's argv showed this dispatch. It execs
   `/snap/bin/gnome-desktop-runtime.ubuntu-desktop-session`: a *real snap
   app* (despite its name), with the runtime's `usr/bin/gnome-session` as its
   command. Its live session process was labeled
   `snap.gnome-desktop-runtime.ubuntu-desktop-session (enforce)` (by the time
   inspected, the process command showed `gnome-session-init-worker`).
4. The greeter's user systemd instance separately starts session services.
   Base drop-ins for `gnome-session-manager@gnome-login.service` and
   `org.gnome.Shell@gdm.service` select
   `/snap/bin/ubuntu-desktop-session.gnome-session-service` and
   `/snap/bin/ubuntu-desktop-session.gnome-shell --mode=gdm --no-x11`.
   Live greeter processes carried those `snap.ubuntu-desktop-session.*`
   labels. Thus greeter GDM, greeter session leader and greeter Shell have
   **three distinct confinement identities**.

**Important path distinction:** the live `/etc/apparmor.d/gdm` comes from the
read-only base squashfs, but the live
`/etc/systemd/system/gdm.service.d/50-gnome-content.conf` is under a writable
ext4 bind mount at `/etc/systemd`. Its bytes match both the base snap's copy
and the checked-out base source; it is not currently served directly from
the root squashfs. Do not infer immutability for the effective systemd drop-in.

## Greeter to the logged-in `ubuntu` session

1. A separate live GDM worker and Wayland helper handled the real user's
   session; the helper's argv showed `/usr/bin/gnome-session --session=ubuntu`.
   The base's `usr/bin/gnome-session` wrapper selects
   `core-desktop-session-wrapper.sh ubuntu-desktop-session ubuntu:GNOME` for
   ordinary accounts (and its `.real` launcher for GDM/initial-setup accounts
   or when `SNAP_NAME` is already set). The GDM profile permits the wrapper
   to run unconfined (`Ux`); the wrapper itself precedes the snap sandbox.
2. The base's `usr/bin/core-desktop-session-wrapper.sh` prepares the user
   systemd/D-Bus environment (the initial seeding described
   [below](#initial-environment-the-base-wrapper)), points
   `$XDG_RUNTIME_DIR/systemd/user.control` at
   `/snap/gnome-desktop-runtime/current/usr/lib/systemd/user`, and reloads
   user systemd. The symlink was present under `/run/user/1000` on the VM.
   It also links Wayland and PipeWire sockets before execing
   `/snap/bin/ubuntu-desktop-session`. The live session leader carried
   `snap.ubuntu-desktop-session.ubuntu-desktop-session (enforce)`; the app's
   `run-session.sh` in turn execs its **own packaged**
   `usr/bin/gnome-session` with `--session=ubuntu` (verified as a file on the
   session snap's squashfs, distinct from the runtime snap's copy).
3. The patched `gnome-session` execs `gnome-session-init-worker`, which asks
   the user's systemd manager to start `gnome-session@ubuntu.target` and
   remains alive to coordinate shutdown. That target requires Shell and the
   session-manager service; user systemd starts them, not the leader's
   process tree. See [research-session-login.md](research-session-login.md) for the observed
   leader/monitor FIFO and the source-derived lifecycle. Base drop-ins for
   `org.gnome.Shell@ubuntu.service` and
   `gnome-session-manager@ubuntu.service` use
   `gnome-content-launch --session-dispatch` to exec the relevant
   `/snap/bin/ubuntu-desktop-session.<app>` entrypoints. Live Shell and
   session-service processes carried `snap.ubuntu-desktop-session.gnome-shell`
   and `.gnome-session-service` labels; their GNOME executable payload is
   supplied by the mounted runtime content. See
   [research-session-services-model.md](research-session-services-model.md) for individual
   services, dispatch and runtime-link caveats.

**Boundary:** a binary's pathname under `/snap/gnome-desktop-runtime` does not
by itself establish snap confinement. GDM runs under the host `gdm` profile;
the greeter session leader enters the runtime snap app via `/snap/bin`, and
Shell and the logged-in session enter the session snap's apps via `/snap/bin`.
This describes the observed paths, not a proof that every desktop helper
uses one of them.

## Session environment: initial setup and later updates

**Source finding:** environment handling is split between the base's
unconfined login wrapper and a filtered broker in the **session snap**, not
the base. The wrapper seeds shared launch environments before the confined
session starts; the broker handles subsequent updates from the patched
GNOME session manager, including compositor-originated display changes.

There are three different environments to keep separate:

- A process's own environment, inherited by its children.
- The user systemd manager's environment for subsequently started services.
- The session bus's activation environment for subsequently activated programs.

Changing the compositor's environment does not automatically update the
other two: those services are not its children. Publishing to the shared
launch environments does not change already-running processes either.

### Initial environment: the base wrapper

**Source:** `core-base-desktop:variants/desktop/static/usr/bin/core-desktop-session-wrapper.sh`.
Before its final `exec /snap/bin/ubuntu-desktop-session`, the wrapper:

1. Sources snap's `apps-bin-path.sh`, sets the selected
   `XDG_CURRENT_DESKTOP` and `GSETTINGS_BACKEND=keyfile`, and computes the
   host runtime directory as `/run/user/UID`.
2. Reads the user-manager environment with `systemctl --user show-environment`.
   It collects `SNAP`/`SNAP_*` names and stale `LC_*` names from that environment
   and its own, along with selected display/session variables to clear.
3. Builds an initial environment with `DISPLAY=:0`,
   `WAYLAND_DISPLAY=wayland-0`, `XDG_RUNTIME_DIR=/run/user/UID`,
   `XAUTHORITY=/run/user/UID/.Xauthority`, and selected identity, locale,
   desktop, path and input-method values. It sets `PULSE_SERVER` to the
   PipeWire snap's per-user PulseAudio socket.
4. Clears the selected stale values, updates both D-Bus activation and user
   systemd, then explicitly unsets snap-variable names in user systemd.
   D-Bus has no unset operation, so removals there use empty values. The
   wrapper deliberately excludes `WAYLAND_SOCKET` from its stale-variable
   list: its comment records that an empty value breaks libwayland clients.

These are implementation choices, including fixed display/provider paths,
not generic properties of GDM or guaranteed fresh-session values. An SSH
login or linger may keep the same user manager alive across graphical logins;
initial setup cannot assume empty shared state. See
[the shared-UID lifecycle limits](research-session-services-model.md).

The wrapper also registers runtime user units, creates socket links and
starts a background Xauthority-copy loop. The broker does none of those
jobs. In particular, publishing the `.Xauthority` path does not create the
file or wait for it to become usable.

### Later updates: the session environment broker

**Source-defined service:** `ubuntu-desktop-session:snap/snapcraft.yaml`
declares `session-environment-broker` as a D-Bus-activated user daemon with
the `systemd-user-environment` plug. It executes
`usr/libexec/ubuntu-desktop-session/session-environment-broker` inside the
session snap's confinement. It is a user service, not a root daemon or an
unconfined helper exported by `core26-desktop`.

The patched update path is:

```text
GNOME Shell / Mutter Setenv request
  → gnome-session-service
  → org.ubuntu.SessionEnvironmentBroker.UpdateEnvironment(set, unset)
      → org.freedesktop.DBus.UpdateActivationEnvironment(...)
      → org.freedesktop.systemd1.Manager.UnsetAndSetEnvironment(...)
```

**Systemd already exposes the environment API.** Its `SetEnvironment`,
`UnsetEnvironment` and `UnsetAndSetEnvironment` methods are not replaced.
The broker is an application-level intermediary which checks requests and
then calls the existing API. It also updates the D-Bus activation environment.
This is explicit client routing, not transparent interception of systemd calls.

In `patches/gnome-session-environment-broker.patch`, the `gsm-util.c` changes
replace GNOME's direct per-variable updates with broker calls. They also make
bulk shared-environment exports no-ops, while retaining the local child
environment handling. The patch explicitly relies on the host wrapper's
initial seeding. This describes the patched GNOME Session path, not a claim
that every session process has been patched to use the broker.

**Filtering in `environment-broker/broker.go` and `main.go`:**

- Authenticate the sender through D-Bus `GetConnectionCredentials`: its UID
  must match the broker's and its AppArmor profile must be
  `snap.ubuntu-desktop-session.gnome-session-service`.
- Accept selected display/session, locale and cursor variables, plus `PATH`
  and `XDG_DATA_DIRS`; validate names, text, value formats and path lists.
  Other valid but unlisted variable names are ignored, not passed through.
- Treat supplied `SNAP`/`SNAP_*` names as removals and clear snap-variable
  names inherited by the broker at startup. This is not an enumeration of
  every stale name currently in the user manager; the wrapper does that
  initial inspection.
- Force `XDG_RUNTIME_DIR=/run/user/UID` and
  `XAUTHORITY=/run/user/UID/.Xauthority`, rather than forwarding the caller's
  snap-private paths. Empty values of allowed, non-forced variables request
  removal.
- Update D-Bus first, using empty values for removals, then explicitly set
  and unset variables in systemd. The code comment explains that D-Bus
  updates also seed user systemd on this image, so systemd must be updated
  last to preserve actual unsets there. The two calls are not transactional.

**Purpose:** allow selected desktop state to reach newly launched programs
without copying a snap's entire private environment into shared launch state.
Allowing a systemd D-Bus method in AppArmor does not restrict variable names
or values in its arguments; the broker supplies that validation. A confined
broker can perform the updates through its interface permissions without
itself executing unconfined.

### Intended separation versus the inspected policy

The session-snap `README.md` says shared-environment writes have been removed
from `systemd-user-control` and isolated in `systemd-user-environment`.
**That statement does not match the inspected snapd source revision.**
`interfaces/builtin/systemd_user_control.go` still emits
`UpdateActivationEnvironment`/`UnsetAndSetEnvironment` for the
`ubuntu-desktop-session` leader app, and other environment-write rules for
selected Plasma apps. These are distinct from the broker caller,
`gnome-session-service`.

The broker and `session_environment_broker.go` restrict who may call its
filtered API, but that does not prove that all other direct mutation paths
are denied. Treat the README as design intent on this point, not evidence
of the live policy. The follow-up did not verify generated profiles or
attempt a bypass. The evidence does not establish the broker as the exclusive
confined-client update path; closing other paths is a proposal, not a finding.

## Existing static snap environment

The non-desktop base hook `hooks/050-snap-symlink.chroot` links snapd's
`990-snapd.conf`; its `environment.d` template adds snap command and desktop-data
paths to `PATH` and `XDG_DATA_DIRS`. This is source evidence at the revisions
above, not a new check of stock `core26`'s installed environment.

## Existing pre-session target and GNOME ordering

**Source precedent:** the base's `user-session-migration.service` is a
`Type=oneshot` user unit with `Before=graphical-session-pre.target` and
`WantedBy=graphical-session-pre.target`. The corresponding wants link is
present in the source tree. Its drop-in excludes the `gdm` group and replaces
`ExecStart` with `/snap/bin/ubuntu-desktop-session.user-session-migration`.
This already combines a platform ordering point with a confined desktop task;
it does not establish the proposed generic setup helper.

**Tagged GNOME source:** `gnome-session-pre.target` requires and orders itself
after `graphical-session-pre.target`. The session-manager service orders itself
after `gnome-session-pre.target` and before `gnome-session-manager.target`;
Shell orders itself after that manager target. The simplified ordering is:

```text
graphical-session-pre.target
  → gnome-session-pre.target
  → gnome-session-manager@….service
  → gnome-session-manager.target
  → org.gnome.Shell@….service
```

This is an ordering chain, not the complete dependency graph. Sources:
[GNOME pre-target](https://github.com/GNOME/gnome-session/blob/9d3b90ab0de1a3b479f8628446ac3e5b08cd71c1/data/gnome-session-pre.target),
[session-manager service](https://github.com/GNOME/gnome-session/blob/9d3b90ab0de1a3b479f8628446ac3e5b08cd71c1/data/gnome-session-manager%40.service.in)
and [Shell service](https://github.com/GNOME/gnome-shell/blob/aecaec6ad79e5f77e19d590d746d0f4c06825420/data/org.gnome.Shell%40.service.in)
at the revisions above. GDM or SDDM alone does not impose this user-unit graph;
no equivalent SDDM desktop integration was validated in this review.

## Related proposal

[Generic session setup](proposal-session-setup.md) proposes splitting provider
registration, per-login preparation and later runtime publication. It builds
on the wrapper, broker and ordering research above; the helper and its
integration have not been implemented or tested.
