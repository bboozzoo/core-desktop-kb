# GNOME login, PAM and the host/runtime boundary

## Evidence and limits

Read-only inspection on 2026-11-19 of the Arch Linux host `galeon`
(GDM `51.1.1-1`, systemd `262.2-1`) and the Ubuntu Core VM
`ubuntu@vsock%59467` (`ubuntu-Standard-PC-Q35-ICH9-2009`, snapd
`2.76.3+g78dca3a.78dca3a`). The VM has `core26-desktop`,
`gnome-desktop-runtime` and `ubuntu-desktop-session` at revision `x1`,
and stock `core26` at revision `750`. Evidence includes installed PAM
policies, snap payloads, `systemctl show`, `loginctl`, `findmnt`, process
labels and boot journal entries. Source inspected read-only:

- `GNOME/gdm`, `main`, `eb66ded7b195f48a4853447a987147b6a861f9c3`:
  [`daemon/gdm-session.c`](https://github.com/GNOME/gdm/blob/eb66ded7b195f48a4853447a987147b6a861f9c3/daemon/gdm-session.c)
  and [`daemon/gdm-session-worker.c`](https://github.com/GNOME/gdm/blob/eb66ded7b195f48a4853447a987147b6a861f9c3/daemon/gdm-session-worker.c).
  This is upstream source, **not** a verified build match for either machine.
  The VT path was also checked in the upstream `50.0` tag,
  `7aa5c1a3d73b51b9ccf89c51d33bfa53cc57d52e`, in the same files.
- `GNOME/gnome-session`, `50.0`,
  `9d3b90ab0de1a3b479f8628446ac3e5b08cd71c1`:
  `gnome-session/{leader-main,leader-systemd,service-main}.c` and
  `tools/gnome-session-ctl.c`. The session snap builds this release with
  local patches; the tagged source alone is not the installed binary.
- `GNOME/mutter`, `50.0`,
  `e25c42e63234a9e3480567ab46ecf20fcefdf938`:
  `src/backends/meta-launcher.c` and
  `src/backends/native/meta-device-pool.c`. The installed Mutter build was
  not matched to this source.
- `canonical/core-base-desktop`, `desktop/26`,
  `fea653f29ef06a43d10460c788f0a8aa01687a39`: files under
  `variants/desktop/static/` and `hooks/` cited below.
- `canonical/ubuntu-desktop-session-snap`, `desktop/26`,
  `2697819d8c35678b5b1d9494c99b24f36d218f63`:
  `snap/snapcraft.yaml`, `scripts/run-session.sh`,
  `patches/gnome-session-environment-broker.patch` and `README.md`.
- `canonical/ubuntu-core-desktop`, `main`,
  `cc7069d64915dbd0f8d7cb8c97e2ba410745225a`: `build.sh`,
  `ubuntu-core-desktop-26-amd64-dangerous.json`.
- `canonical/ubuntu-core-desktop-snapd` (local `snapd-work-desktop`),
  `desktop/26`, `78dca3acd0dcc04b738a24b07e9dae7c7dc66c72`:
  `snap/validate.go`, `interfaces/mount/spec_test.go`, and
  `interfaces/builtin/{gdm,login_session_control,wayland}.go`.

No new graphical login, logout, refresh, stock-`core26` image, or
GDM-as-snap prototype was run. The installed `gnome-desktop-runtime` was
inspected on the VM; its separate source repository was not inspected.
For the observed boot and confinement path, see
[session-startup.md](session-startup.md); for the service graph, see
[session-services-model.md](session-services-model.md).

## Normal GDM → PAM → logind → desktop path

**Upstream source and host observation:** GDM brokers greeter verification
through a `gdm-session-worker` process. The worker calls `pam_start()` with
the selected service name, then (for password login) `pam_authenticate()`,
`pam_acct_mgmt()`, `pam_setcred()` and `pam_open_session()` as the exchange
advances. PAM conversation prompts and replies pass between worker and
greeter via GDM's session broker; the greeter does not itself call PAM.
After a successful open, the worker launches the selected session command
through `gdm-wayland-session` for Wayland. On cleanup it calls
`pam_close_session()` if the session reached the open state. These are
conditional steps, not a claim that every authentication reaches launch
(GDM source: `daemon/gdm-session{,-worker}.c`).

The **PAM service file** chooses authentication, account and session
modules; it is not the GNOME session definition. `pam_systemd.so` in the
session stack registers the login with the host's `systemd-logind` and
arranges its session scope, seat association and user runtime directory.
Logind starts or reuses the separate `user@UID.service` system unit, whose
`systemd --user` manages the user's services. User-manager processes are
not generally children of the GDM-launched session process; the manager
can outlive a graphical login if another login or linger keeps it running.
See [pam_systemd(8)](https://www.freedesktop.org/software/systemd/man/latest/pam_systemd.html),
[systemd-logind.service(8)](https://www.freedesktop.org/software/systemd/man/latest/systemd-logind.service.html)
and [user@.service(5)](https://www.freedesktop.org/software/systemd/man/latest/user@.service.html).

**Host check:** Arch GDM runs `/usr/bin/gdm` via `gdm.service`.
`/usr/lib/pam.d/gdm-password` includes `system-local-login`, reaching
`pam_systemd.so` through `/etc/pam.d/system-login`. The observed Wayland
login uses PAM service `gdm-password`, `session-4.scope` under
`user-1020.slice`, and a separate `user@1020.service` (`PAMName=systemd-user`).
Arch keeps the vendor GDM PAM service under `/usr/lib/pam.d`; Ubuntu Core's
installed service files are under `/etc/pam.d`. Do not generalize one
distribution's vendor-file path to the other.

## Observed Ubuntu Core VM

**Authentication and session registration:** A root GDM worker from
`/snap/gnome-desktop-runtime/current/usr/libexec/gdm-session-worker` runs
under the **host `gdm` AppArmor profile**, not a runtime snap-app profile.
The base's `/etc/pam.d/gdm-password` includes `common-auth`,
`common-account` and `common-session`; `common-auth` uses
`pam_extrausers.so` and `common-session` calls `pam_systemd.so`. These
common files, `pam_systemd.so`, logind and `user@.service` are also present
in the VM's installed stock `core26`; comparison found no differences in
the inspected `common-auth`, `common-account` and `common-session` files.
Stock `core26` did **not** contain the inspected GDM-specific PAM service
files, `gdm.service` or `/etc/apparmor.d/gdm`.

The boot journal records `pam_unix` opening the greeter's
`gdm-launch-environment` session, `pam_extrausers` authentication and
`pam_unix` opening the `ubuntu` user's `gdm-password` session. Logind
registered the greeter on `seat0` as class `greeter`, and the Wayland
`ubuntu` session as class `user` (`session-2.scope`). Its user manager runs
separately as `user@1000.service` (`PAMName=systemd-user`); SSH logins for
that UID share it. The journal also records `pam_systemd` failing to set
ambient capabilities and **ignoring** the error: the graphical session
nevertheless started. This is evidence for the observed boot, not a trace
of the entire authentication conversation.

### Session leader and the user service graph

**Observed:** GDM selects the runtime's Wayland `ubuntu.desktop` entry, whose
`Exec` calls `/usr/bin/gnome-session --session=ubuntu`. The base wrapper
enters `/snap/bin/ubuntu-desktop-session`; `run-session.sh` then execs the
session snap's own patched GNOME Session 50.0 leader. That leader prepares
its session environment and execs `gnome-session-init-worker` (the local
patch redirects the worker path into the session snap). The live init-worker
(PID 5110) is a child of `gdm-wayland-session`, **not** of user systemd.

**Source-derived startup, checked against the live target:** the worker
checks for an existing `graphical-session.target`, calls user systemd's
`ResetFailed`, then `StartUnit("gnome-session@ubuntu.target", "fail")`.
The VM's active target requires `org.gnome.Shell@ubuntu.service` and
`gnome-session-manager@ubuntu.service`. User systemd starts both through
snap entrypoints; the inspected Shell (PID 5653) and session-manager
service (PID 5597) are not children of the GDM-launched leader. The
session-manager service provides GNOME's D-Bus session-management process;
it is **not** the init-worker. Sources: [GNOME Session leader](https://github.com/GNOME/gnome-session/blob/9d3b90ab0de1a3b479f8628446ac3e5b08cd71c1/gnome-session/leader-systemd.c),
[session-manager entry point](https://github.com/GNOME/gnome-session/blob/9d3b90ab0de1a3b479f8628446ac3e5b08cd71c1/gnome-session/service-main.c),
`ubuntu-desktop-session-snap:snap/snapcraft.yaml:1511-1550`, its
`patches/gnome-session-environment-broker.patch:315-346`, and the VM's
`systemctl --user show gnome-session@ubuntu.target`.

**Source-derived lifetime, live FIFO verified:** after starting the target,
the init-worker remains in a main loop, holding
`$XDG_RUNTIME_DIR/gnome-session-leader-fifo` open. The user manager runs
`gnome-session-monitor.service` (`gnome-session-ctl --monitor`) and holds
the other FIFO end. Both live processes (PIDs 5110 and 5481) had FDs to
`/run/user/1000/snap.ubuntu-desktop-session/gnome-session-leader-fifo`.
On a termination signal, the leader requests
`gnome-session-shutdown.target` and notifies the monitor. The monitor
signals `STOPPING=1` to systemd; when it exits, the leader waits for
`graphical-session-pre.target` to become inactive before exiting. If the
leader disappears first, FIFO hangup also triggers monitor-side teardown.
GDM watches its session child and cleans up when it exits. Thus the
init-worker cannot simply start the target and exit without changing the
current login/lifecycle contract. This shutdown behavior was **not**
actively tested on the VM. Sources:
[leader-systemd.c](https://github.com/GNOME/gnome-session/blob/9d3b90ab0de1a3b479f8628446ac3e5b08cd71c1/gnome-session/leader-systemd.c)
and [gnome-session-ctl.c](https://github.com/GNOME/gnome-session/blob/9d3b90ab0de1a3b479f8628446ac3e5b08cd71c1/tools/gnome-session-ctl.c).

### VT and display-device handoff

**Source-derived sequence:** for a new `seat0` login, GDM's worker uses
`VT_OPENQRY` to allocate a VT, sets `XDG_VTNR` and `PAM_TTY` before
`pam_open_session()` registers the session with logind, then switches to
that VT with `VT_ACTIVATE`/`VT_WAITACTIVE` **before** launching the
session command. GDM's `NEW_VT` mode expects the session compositor to
use logind for display/input device access. Sources:
[GDM 50.0 session selection](https://github.com/GNOME/gdm/blob/7aa5c1a3d73b51b9ccf89c51d33bfa53cc57d52e/daemon/gdm-session.c)
and [worker](https://github.com/GNOME/gdm/blob/7aa5c1a3d73b51b9ccf89c51d33bfa53cc57d52e/daemon/gdm-session-worker.c).

**Observed result:** the VM's user session is active on `seat0`, `tty2`,
`VTNr=2`; `/sys/class/tty/tty0/active` also says `tty2`. The greeter and
user have **separate** Shell/Mutter processes and separate user managers.
**Inference from that separation:** there is no GDM-to-user-Shell greeter
buffer transfer; the seat/display-device access changes, and the user's
compositor produces its own frames. Logind's
[session controller API](https://www.freedesktop.org/software/systemd/man/latest/org.freedesktop.login1.html)
provides `TakeControl`, `TakeDevice` and device pause/resume on session
switch. Mutter 50.0's
[`meta-launcher.c`](https://github.com/GNOME/mutter/blob/e25c42e63234a9e3480567ab46ecf20fcefdf938/src/backends/meta-launcher.c)
selects the graphical logind session and requests control;
[`meta-device-pool.c`](https://github.com/GNOME/mutter/blob/e25c42e63234a9e3480567ab46ecf20fcefdf938/src/backends/native/meta-device-pool.c)
requests device FDs.
The Shell app declares `login-session-control` and a Wayland-server slot;
the connected snapd interfaces grant relevant logind D-Bus and tty/DRM
policy (`ubuntu-desktop-session-snap:snap/snapcraft.yaml:353-391`,
`snapd-work-desktop:interfaces/builtin/{login_session_control,wayland}.go`).

**Boundary:** Shell runs under `user@1000.service`, outside the graphical
`session-2.scope`. Its inspected environment has `XDG_VTNR=2` but **no**
`XDG_SESSION_ID`; logind reports the user's `Display=2`. Mutter's tagged
source falls back to `sd_uid_get_display()` when it cannot obtain a
session from its environment or PID. This explains a possible route for
Shell to find session 2, **not** a traced call sequence in the VM. Neither
an actual VT switch nor device-FD acquisition was syscall-traced.

**Where runtime-owned login metadata becomes visible:**

| Kind | Runtime payload | Host-facing integration actually observed |
| --- | --- | --- |
| Wayland login entry | `usr/share/wayland-sessions/ubuntu.desktop`, whose `Exec` is `/usr/bin/gnome-session --session=ubuntu` | `/usr/share/wayland-sessions/ubuntu.desktop` is a base-owned symlink into the runtime snap. GDM can find it through the host path. |
| GNOME session description | `usr/share/gnome-session/sessions/ubuntu.session` | GNOME consumes this after launch; it is distinct from the Wayland login entry and the PAM service. |
| User service graph | `usr/lib/systemd/user/`, including `gnome-session-manager@.service` | The base-supplied pre-snap login wrapper links `$XDG_RUNTIME_DIR/systemd/user.control` to that directory, then reloads the **host user manager**. Base-provided drop-ins still change several unit commands to snap app entrypoints. |
| GDM PAM policy | No `gdm-*` files in this installed runtime revision: its `etc/pam.d` is empty | Regular files in the read-only `core26-desktop` root at `/etc/pam.d/gdm-*`; not currently read out of runtime content. |

The runtime ownership of session definitions and upstream units is real,
but **ownership is not registration**. The intended direction of putting
GDM PAM policy in the runtime must be distinguished from this VM's
installed payload. The `ubuntu.desktop` symlink and the `user.control`
link prove that two types of metadata are already consumed from the
runtime; they do not prove that GDM PAM policy is. Sources:
`core-base-desktop:variants/desktop/static/{etc/pam.d/gdm-password,
usr/share/wayland-sessions/ubuntu.desktop,
usr/bin/core-desktop-session-wrapper.sh,
usr/lib/systemd/user/gnome-session-manager@.service,
usr/lib/systemd/user/gnome-session-manager@ubuntu.service.d/core-override.conf}`;
VM `/snap/gnome-desktop-runtime/current/` and effective user-unit paths.

## What moving off `core26-desktop` would entail

**Inference from the observed ownership boundaries, not a tested design:**
Keeping GDM *host-managed* while using stock `core26` still requires a
host-visible PAM service, `gdm.service`, GDM privilege policy, greeter
account/seat setup and a desktop-entry/launcher handoff. A content snap
being mounted at `/snap/...` does **not** make its PAM files appear at
`/etc/pam.d/gdm-*`, or make host systemd load arbitrary units from it.
The current base supplies these pieces: `variants/desktop/static/etc/pam.d/`,
`usr/lib/systemd/system/gdm.service`, `etc/apparmor.d/gdm`, the
`usr/bin/gnome-session` wrapper, and user-unit drop-ins. The image model
currently names `core26-desktop` as its boot base. The KDE image path
explicitly copies its display manager's PAM files into the root and factory
`/etc/pam.d` (`ubuntu-core-desktop:build.sh:319-357`); that is an image-time
integration example, **not** automatic host registration by a runtime snap.

A minimal base could hold symlinks into the runtime for more files, but
this still hard-codes a provider and paths into the boot base, trusts its
refreshes with host-read policy, and needs ordering and failure/rollback
behavior when the runtime is absent or changing. Baking copies into the
base instead loses independent policy updates. Neither is a clean generic
contract for several replaceable desktop runtimes.

A different **candidate**, if GDM became a privileged snap daemon, is to
present its PAM service files inside *its own mount namespace* through
`layout: bind-file` at `/etc/pam.d/gdm-*`, including the host/Core common
PAM policy. Snapd source permits `/etc` layout destinations and tests
bind-file at `/etc/foo.conf` (`snap/validate.go:1072-1128`,
`interfaces/mount/spec_test.go:166-185`). That would avoid globally
installing GDM's PAM files; it would **not** expose layouts to the host
`systemd --user` or solve the host-facing login entry. It also requires
review of PAM module availability (the inspected stock `core26` has no
`pam_gnome_keyring.so`), host account access, logind/seat privileges,
refresh behavior and the transition from GDM's child into the *different*
`ubuntu-desktop-session` snap. The existing snapd `gdm` interface permits
**clients** to call an existing GDM; `login-session-control` grants
specified logind D-Bus operations. Neither establishes that a confined
GDM worker can run the complete PAM/login flow. No prototype was tested.

**Conclusion:** PAM is not intrinsically incompatible with a GNOME runtime
snap. The unresolved architecture is **who registers runtime-owned login
metadata with host GDM/systemd, and under what trust and lifecycle rules**.
Removing the desktop base requires an explicit, maintained host-integration
mechanism (or a proved snap-daemon GDM plus a narrower host registration
mechanism), not merely moving files into the runtime squashfs. The first
validation should cover greeter/password login, PAM/logind registration,
keyring unlock, session-snap handoff, user-unit startup, logout/relogin,
and runtime refresh/rollback on a stock-`core26` image.
