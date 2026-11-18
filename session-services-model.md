# Session service startup: how GNOME components actually get confined

## Summary

The inspected GNOME session components (Shell, the session manager and
settings daemons) are started by the unconfined `systemd --user` through
snap application entrypoints, directly or via a dispatcher:

```text
user systemd -> /snap/bin/ubuntu-desktop-session.<app>
             -> snap run -> snap-confine -> confined component
```

This is a reasonable confinement mechanism. It establishes the normal snap
sandbox rather than applying only an AppArmor label. These startup paths
need neither `desktop-launch` nor a separate `AppArmorProfile=` override.
The current Shell/session-manager configuration differs from the
AppArmorProfile-only approach described by `57b436e758` (see
`desktop-launch.md`); the historical failing setup was not reproduced.

The integration is nevertheless desktop-specific: the base supplies unit
overrides, activation files, runtime links and security policy. The wrapper
also assumes one selected graphical desktop per UID. These are compatibility
and lifecycle constraints, not evidence that the observed runtime directory
is insecure or that every confined service can escape its sandbox.

Sources: `canonical/core-base-desktop` at `fea653f`, the local
`ubuntu-desktop-session-snap` checkout, snapd at `78dca3acd0`, and the VM
used in `desktop-launch.md` (`ssh ubuntu@vsock%59467`, snapd
`2.76.3+g78dca3a.78dca3a`, inspected 2026-11-18). Paths under `variants/`
and `hooks/` below are relative to the `core-base-desktop` repository.
VM inspection was read-only.

## The two unconfined layers involved

1. **Login handoff.** For the real `ubuntu` account, `/usr/bin/gnome-session`
   (`variants/desktop/static/usr/bin/gnome-session`) execs
   `core-desktop-session-wrapper.sh ubuntu-desktop-session ubuntu:GNOME`. This
   wrapper runs unconfined, prepares environment variables, replaces
   `$XDG_RUNTIME_DIR/systemd/user.control` with a symlink into
   `/snap/gnome-desktop-runtime/current/usr/lib/systemd/user` (this is how the
   upstream GNOME unit files become visible to the user's systemd instance),
   runs `daemon-reload`, then `exec "/snap/bin/ubuntu-desktop-session"`. That
   last exec is the first `snap run` -> `snap-confine` transition: the
   confined `gnome-session` process itself starts here.

2. **Unit activation.** The inspected Shell and session-manager services
   are started independently by user systemd, not forked directly from the
   confined session leader. GNOME uses unit/target activation, not the
   `systemd-run` command or the desktop-launch broker, for this startup.
   In the inspected snapd source, `systemd-user-control` grants selected
   session-leader apps `GetUnit`, `StartUnit`, `ResetFailed` and unit property
   access. It is not merely an observation interface; some app-specific
   snippets also grant shared-environment writes.

## Per-instance ExecStart overrides

Static drop-ins shipped by `core-base-desktop`
(`variants/desktop/static/usr/lib/systemd/user/<unit>.service.d/core-override.conf`)
rewrite the generic upstream `ExecStart=` for the units that are instantiated
per-login-mode (`@ubuntu`, `@gdm`, `@gnome-login`, ...) or that simply need to
point at the snap. Two patterns were observed:

| Unit instance | Override | Confinement path |
|---|---|---|
| `org.gnome.Shell@ubuntu.service` | `ExecStart=/usr/libexec/gnome-content-launch --session-dispatch gnome-shell --mode=%i` | dispatched |
| `org.gnome.Shell@gdm.service` | `ExecStart=/snap/bin/ubuntu-desktop-session.gnome-shell --mode=%i --no-x11` | direct |
| `gnome-session-manager@ubuntu.service` | `ExecStart=/usr/libexec/gnome-content-launch --session-dispatch gnome-session-service --session=%i` | dispatched |
| `gnome-session-manager@gnome-login.service` | `ExecStart=/snap/bin/ubuntu-desktop-session.gnome-session-service --session=%i` | direct |
| `org.gnome.SettingsDaemon.Sharing.service` (+ `ConditionGroup=!gdm`) | `ExecStart=/snap/bin/ubuntu-desktop-session.gsd-sharing` | direct |

Both the "direct" and "dispatched" forms terminate at
`/snap/bin/ubuntu-desktop-session.<app>`, i.e. ordinary `snap run` ->
`snap-confine`. Confirmed live on the VM: `gnome-shell`'s process carries
`snap.ubuntu-desktop-session.gnome-shell (enforce)` and
`SNAP=/snap/ubuntu-desktop-session/x1`. The live Shell and session-manager
units have empty `AppArmorProfile=` properties; the listed drop-ins do not
set it. This does not extend to all desktop services: GDM has a separate
host AppArmor profile, described below.

## The dispatcher: `/usr/libexec/gnome-content-launch`

`variants/desktop/static/usr/libexec/gnome-content-launch` has three modes:

1. **`--session-dispatch SESSION_APP`** maps `gnome-session-service` and
   `gnome-shell` to their `/snap/bin/ubuntu-desktop-session.<app>` entrypoints.
   `gnome-login-session` maps to
   `/snap/bin/gnome-desktop-runtime.ubuntu-desktop-session`. If the invoking
   process belongs to the `gdm` group, only `gnome-login-session` is accepted;
   otherwise the three recognized names are accepted. The direct greeter
   overrides bypass this dispatcher, so they do not encounter its group
   check. That check neither changes UID nor establishes a sandbox.
2. **`--content CONTENT_EXECUTABLE`** accepts the supplied executable path
   directly, adjusts library/data environment variables, and execs it.
   Despite the option name, the script does not enforce that the path is
   beneath `/snap/gnome-desktop-runtime/current`.
3. **Plain alias mode** derives a content executable from the invoked `/usr/`
   path. For example, `/usr/libexec/gnome-session-ctl` is a symlink to this
   launcher and resolves to the content snap's corresponding binary. This
   mode restricts callers to named GDM/initial-setup accounts, except for
   `/usr/libexec/gdm-*`, `/usr/bin/dconf` and `/usr/libexec/gnome-session-ctl`.

The latter two modes **do not establish a snap sandbox**, but they do not
inherently remove confinement either: existing policy and exec transitions
still apply. In particular,
`variants/desktop/static/etc/systemd/system/gdm.service.d/50-gnome-content.conf`
sets `AppArmorProfile=gdm` while invoking GDM through `--content`. This is
host AppArmor confinement, not the runtime snap's app confinement.

The `/usr/bin/gnome-shell -> gnome-content-launch` symlink alone therefore
does **not** route Shell through `snap run`. The `@ubuntu` unit's explicit
`--session-dispatch` override is what selects the snap entrypoint. Plain
alias invocation would be rejected for an ordinary `ubuntu` account.

The session-manager unit's `ExecStopPost=` invokes
`/usr/libexec/gnome-session-ctl --shutdown` through plain alias mode. Unlike
its main process, this helper is launched separately by unconfined user
systemd without the snap bootstrap. That is consistent with the historical
commit's description of an unconfined `--signal-init` helper. It is an
intentional trusted-helper path, not proof that every lifecycle helper must
remain unconfined or that its full authority has been reviewed.

## Runtime paths: who creates them?

The creator identified in source for the following links is the base's
`/usr/bin/core-desktop-session-wrapper.sh`, before its final snap exec.
These links are not automatically created by logind, systemd or `snap run`.
Source searches across the base, session-snap and image repositories, and
selected VM `/usr` and `/etc` paths, found this wrapper as their creator;
no syscall-level creation trace was recorded.

### `systemd/user.control`: standard search path, custom use

`$XDG_RUNTIME_DIR/systemd/user.control` is a standard systemd user-unit
search location for runtime configuration created through its D-Bus API.
It need not exist on an ordinary installation. The nonstandard part here
is replacing it with a symlink to an entire content snap's unit directory.
See [systemd.unit: unit load paths](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html#Unit%20File%20Load%20Path).

The VM's `systemd-analyze --user unit-paths` reports, in priority order:

```text
/home/ubuntu/.config/systemd/user.control
/run/user/1000/systemd/user.control
/run/user/1000/systemd/transient
/run/user/1000/systemd/generator.early
...
/usr/lib/systemd/user
```

The wrapper selects GNOME's
`/snap/gnome-desktop-runtime/current/usr/lib/systemd/user` or KDE's
`/snap/plasma-core26-desktop/current/usr/lib/systemd/user`, then runs
(lines 174-177):

```sh
rm -rf "$XDG_RUNTIME_DIR/systemd/user.control"
mkdir -p "$XDG_RUNTIME_DIR/systemd"
ln -sf "$user_unit_source" "$XDG_RUNTIME_DIR/systemd/user.control"
systemctl --user daemon-reload
```

The script explicitly calls this a temporary workaround. The consequences
are more specific than merely making units visible:

- The imported main unit files take precedence over same-named files in
  `/usr/lib/systemd/user`. The inspected base-provided drop-ins still apply;
  a higher-priority main fragment does not discard all lower-path drop-ins.
- An existing symlink is removed without deleting its target. If the path
  were a real directory, the recursive removal would delete its contents,
  including any unrelated runtime overrides. No such loss was observed.
- The path is now backed by read-only snap content rather than an ordinary
  writable runtime-configuration directory. Compatibility with systemd
  operations that write there was not tested.

### Wayland, audio and Xauthority

The same wrapper creates these links (lines 188-191):

| Host-visible path under `/run/user/1000` | Observed link target |
|---|---|
| `wayland-0` | `snap.ubuntu-desktop-session/wayland-0` |
| `pipewire-0` | `snap.pipewire/pipewire-0` |
| `pulse` | `snap.pipewire/pulse` |

The real Wayland socket is in Shell's snap-private runtime directory; the
link exposes its conventional host path. It does not itself grant a client
permission to connect: AppArmor and other access checks still apply.

`stat` on the VM showed this sequence on 2026-11-18 (local time, +0100):

| Time | Object |
|---|---|
| 14:34:46.884 | `systemd/user.control` symlink |
| 14:34:47.053 | `wayland-0` symlink |
| 14:34:47.054-47.057 | Audio symlinks |
| 14:34:48.900 | Actual `snap.ubuntu-desktop-session/wayland-0` socket |
| 14:34:49.064 | `.Xauthority` regular file |

This is consistent with the wrapper creating a dangling Wayland link before
Mutter creates the socket. There is no wait for that socket in the wrapper.
Separately, `fixup_xauthority` polls for the desktop's auth file and copies it
to `$XDG_RUNTIME_DIR/.Xauthority`; this is a copy, not a symlink. Timestamps
support the source-derived explanation, but do not independently identify
the creating process.

## Per-user runtime versus individual graphical sessions

A shared user manager and `/run/user/<uid>` are normal. The concern is
**same-UID graphical-session coexistence and lifecycle**, not the appearance
or permissions of the directory. On the VM, `/run/user/1000` is owned by
`ubuntu`, mode `0700`.

The observed logind sessions were:

| Session | Purpose | UID |
|---|---|---|
| `2` | GDM Wayland login on `seat0` | 1000 |
| `12` | SSH login | 1000 |
| `3` | Systemd user manager, class `manager` | 1000 |

They share `user@1000.service` (PID 20120). The SSH shell reports the same
`XDG_RUNTIME_DIR=/run/user/1000` and
`DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus` as the desktop uses.
SSH does not run the desktop wrapper, so opening SSH alone does not replace
these links or reconfigure the desktop.

The user manager has one selected activation environment:

```text
DISPLAY=:0
WAYLAND_DISPLAY=wayland-0
XAUTHORITY=/run/user/1000/.Xauthority
XDG_CURRENT_DESKTOP=ubuntu:GNOME
```

Likewise, `org.gnome.Shell@ubuntu.service` is one unit: its instance identifies
the GNOME mode, not a unique logind session. The `snap.<name>` runtime
directory is per user and snap, not per graphical login.

If a second graphical login for the same UID were allowed, its wrapper
could replace the shared unit source, activation environment, Wayland link
and auth file, and stop the shared `xdg-desktop-portal` unit. New activations
or connections could then use the other desktop's settings. Existing
process environments and established connections are not retroactively
rewritten; `daemon-reload` also does not restart everything.

This is a reasonable single-graphical-session-per-user assumption if
intentional. It is not unique to snaps, and some desktop/login stacks
already disallow or do not support concurrent graphical sessions for one
UID. Different UIDs have separate runtime directories and user managers.
No concurrent-graphical-session failure was demonstrated.

Sequential login also needs lifecycle handling. Although the VM reports
`Linger=no`, an SSH login can keep the user manager alive after graphical
logout. A new graphical login cannot assume a fresh runtime directory,
activation environment or service set. The wrapper clears some stale
environment values, but complete logout/relogin and desktop-switch behavior
was not tested.

## Maintainability and desktop-version coupling

The base encodes more than generic snap startup:

| Integration detail | Source example |
|---|---|
| Component names, arguments and session instances | `variants/desktop/static/usr/lib/systemd/user/*.service.d/` |
| Desktop D-Bus activation | `variants/desktop/static/usr/share/dbus-1/services/` |
| Provider identity and filesystem layout | `gnome-content-launch` and the wrapper's GNOME/KDE branches |
| Runtime library ABI | `hooks/995-gnome-content-lib-symlinks.chroot`, including explicit SONAMEs |
| Security identities | `variants/desktop/static/etc/apparmor.d/gdm`, naming particular snap app labels |

A fixed base fixes this integration contract, not necessarily an exact GNOME
or KDE release. Compatible runtime updates may need no base change. New or
renamed units, helper commands, libraries or activation paths may require
coordinated base/session/runtime changes. Missing overrides could cause
startup failures or execution without the intended snap bootstrap, depending
on the path and policy; neither outcome was tested with a new release.
Moving payload into a content snap therefore does not remove the base's
desktop compatibility obligations.

Snapd also contains desktop-name coupling:
`interfaces/builtin/systemd_user_control.go:300-302` selects extra permissions
using app names `ubuntu-desktop-session`, `plasma-desktop-session` and
`plasma-kwin-wayland`. A new desktop should not need to impersonate one of
those names to obtain an appropriate session-management capability.

## Security boundary and architectural direction

**Starting a declared snap app from a trusted manager is sound; trusting
arbitrary imported unit files is a separate decision.** Unit files instruct
the host manager what to execute. Importing the runtime's whole directory
makes its publisher a supplier of trusted host-session configuration, not
merely libraries consumed inside another snap's sandbox. Read-only content
and signatures establish integrity/provenance, not confinement of every
host execution of that content. GDM additionally loads runtime-provided
code under its host policy rather than a snap app profile.

Checking `ExecStart=` alone is insufficient: startup/shutdown helpers,
D-Bus activation, autostart and dependency/failure paths also need deliberate
confinement decisions. Effective service-control permissions matter after
startup too: the inspected `StartUnit` rule does not constrain its unit-name
argument, and selected app-specific rules allow shared-environment writes.
The broad `desktop-launch` permissions remain a separate concern.

Per-app profiles can express least privilege, but apps within one snap share
substantial resources and should not automatically be treated as independent
snap-level isolation boundaries. A compositor is inherently trusted with its
desktop's pixels/input; X11 adds further client-isolation limits. Enforcing
labels alone do not establish the intended end-to-end threat model.

For a managed first-party image, the current design can be supported through
explicit trust and tested version combinations. For independently updatable
desktops, prefer:

- Generic login/session mechanisms in the platform; desktop-specific service
  graphs and activation metadata owned with the desktop.
- Existing snap-managed user daemons where they fit. Shell, session-manager
  and Sharing are currently declared as apps, while some other components
  use `daemon-scope: user`; refresh and lifecycle behavior must account for
  this hybrid arrangement.
- A constrained registration/control mechanism for any missing graphical
  session semantics, not unrestricted installation of host unit files.
  Moving the same unvalidated files into another snap is not sufficient.
- Brokered desktop-entry launches for applications; service supervision
  remains a distinct responsibility.

A simple compositor/window manager may only need one snap session entrypoint
and a few helpers, not GNOME's service graph. Its ordinary child processes
retain its confinement unless an authorized transition occurs. Arbitrary
commands in a WM configuration must not silently become unrestricted host
execution; that is a different capability from desktop-entry launching.

## Relationship to `desktop-launch.md`

These inspected service-startup paths do not justify giving confined callers
snap-confine transitions or unrestricted `StartTransientUnit`: user systemd
already starts the snap bootstrap from outside their sandboxes. This supports
fixing broker routing and removing the direct-launch grants discussed in
`desktop-launch.md`, rather than expanding them for more session components.
It does not prove that every application-launch path or remaining interface
grant is unnecessary; those need the focused tests listed there. In
particular, `systemd-user-control` is not merely an observation interface and
needs a separate boundary review.

## Limits and remaining checks

- Source inspection plus live unit properties, selected process labels and
  environment, `loginctl`, unit search paths, links and timestamps were used.
  Upstream unit files were read through the mounted content snap. Matching
  checksums of two paths confirm disk content, not systemd's cached in-memory
  definition; `systemctl show` was used for effective command properties.
- Shell/session-manager/Sharing were representative paths, not an exhaustive
  review of all overrides and helpers. Greeter overrides were read, but the
  full greeter and initial-setup execution chains were not traced.
- No desktop application or service was deliberately launched, stopped or
  reconfigured. Link creation was not traced live, and searches do not prove
  that no other creator exists outside the inspected locations.
- No upgrade/rollback, same-UID concurrent graphical login, desktop-switch,
  logout-with-SSH-surviving or runtime-property-write test was performed.
- Confinement bypasses were not actively probed. Claims about authority and
  compatibility risks are source-based assessments, not demonstrated exploits
  or observed multi-session failures.
