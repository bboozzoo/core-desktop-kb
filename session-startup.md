# GNOME login startup on the inspected Core Desktop VM

## Evidence and scope

Read-only inspection on 2026-11-19 of `ubuntu@vsock%59467`, with snapd
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

This describes the *running greeter and logged-in `ubuntu` session*, not a
newly performed login. The initial-setup path, PAM authentication sequence,
logout/relogin, refresh and profile reload were not actively traced or tested.
For the PAM/logind handoff, host comparison and implications of replacing the
desktop base, see [session-login.md](session-login.md).

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
   systemd/D-Bus environment, points `$XDG_RUNTIME_DIR/systemd/user.control`
   at `/snap/gnome-desktop-runtime/current/usr/lib/systemd/user`, and reloads
   user systemd. The symlink was present under `/run/user/1000` on the VM.
   It also links Wayland and PipeWire sockets before execing
   `/snap/bin/ubuntu-desktop-session`. The live session leader carried
   `snap.ubuntu-desktop-session.ubuntu-desktop-session (enforce)`; the app's
   `run-session.sh` in turn execs its **own packaged**
   `usr/bin/gnome-session` with `--session=ubuntu` (verified as a file on the
   session snap's squashfs, distinct from the runtime snap's copy).
3. The user's systemd manager starts the service graph independently of the
   session leader. Base drop-ins for `org.gnome.Shell@ubuntu.service` and
   `gnome-session-manager@ubuntu.service` use
   `gnome-content-launch --session-dispatch` to exec the relevant
   `/snap/bin/ubuntu-desktop-session.<app>` entrypoints. Live Shell and
   session-service processes carried `snap.ubuntu-desktop-session.gnome-shell`
   and `.gnome-session-service` labels; their GNOME executable payload is
   supplied by the mounted runtime content. See
   [session-services-model.md](session-services-model.md) for individual
   services, dispatch and runtime-link caveats.

**Boundary:** a binary's pathname under `/snap/gnome-desktop-runtime` does not
by itself establish snap confinement. GDM runs under the host `gdm` profile;
the greeter session leader enters the runtime snap app via `/snap/bin`, and
Shell and the logged-in session enter the session snap's apps via `/snap/bin`.
This describes the observed paths, not a proof that every desktop helper
uses one of them.
