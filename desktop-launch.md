# Desktop launch: confinement model and review findings

## Summary

Ubuntu Core Desktop needs a way for confined desktop components to launch other
applications without making those applications inherit the desktop component's
sandbox. A session D-Bus broker is a sound way to do this.

The proposed desktop launcher provides that brokered path. A later commit also
allows callers to execute `snap run` and enter `snap-confine` directly. These are
two different designs: the latter bypasses the broker's restricted desktop-entry
API and grants substantially more authority.

**Recommendation:** make confined desktop launches consistently use the broker,
then remove the direct-launch permissions from `desktop-launch`. Do not grant
callers unrestricted user-systemd `StartTransientUnit` access to make launching
work.

## Scope and evidence

Reviewed commits:

- [`fee88808965af4b014b3b0514e46f21db78b44dd`](https://github.com/canonical/snapd/commit/fee88808965af4b014b3b0514e46f21db78b44dd):
  introduces `snap routine desktop-launch`, `OpenDesktopEntry2`, rewritten desktop
  entries, and availability of `desktop-launch` on Core.
- `57b436e7582d461e11266a72bb7c7a652c2586f4`:
  `interfaces: confine gnome-session AppArmorProfile= units`; adds the direct-launch
  permissions discussed below, alongside other desktop-session policy changes.
- `326973edc70320dca02fd00f8f865498a22c070a`:
  `Fix desktop launcher runtime environment`; clears `SNAP` on broker-created
  target processes and corrects their runtime directory.

The fetched `desktop/26` branch was at `78dca3acd0` when inspected. Additional
read-only source inspection on 2026-11-19 used `core-base-desktop` `desktop/26`
`fea653f29ef06a43d10460c788f0a8aa01687a39` and `ubuntu-core-desktop` `main`
`cc7069d64915dbd0f8d7cb8c97e2ba410745225a` for the alternative below; no
helper was built or run.

The VM accessed through `ssh ubuntu@vsock%59467` reported snapd
`2.76.3+g78dca3a.78dca3a`. VM observations below were made on 2026-11-18.

Inspection was read-only: source, process labels, selected environment variables,
unit configuration, generated AppArmor profiles, desktop entries, and D-Bus
metadata. No application launch, breakout attempt, policy change, service restart,
or daemon-down test was performed. Generated policy files were inspected; their
individual permissions were not verified through active allow/deny probes.

## Intended authorization and execution model

```text
confined desktop component
  -> launch client, retaining the caller's confinement
  -> session bus checks the caller's AppArmor permission
  -> userd validates the desktop entry, action, arguments and environment
  -> unconfined user systemd starts the target launch command
  -> snap-confine establishes the target application's sandbox
```

The broker is outside the requesting application's confinement, but normally runs
as the same user, not as root. The target should enter its own sandbox rather than
inherit GNOME Shell's or Nautilus's restrictions.

### Authorization is not the right to execute the client binary

The interface authorizes both execution of the client and the D-Bus operation:

```apparmor
dbus (send)
    bus=session
    path=/io/snapcraft/PrivilegedDesktopLauncher
    interface=io.snapcraft.PrivilegedDesktopLauncher
    member={OpenDesktopEntry,OpenDesktopEntry2}
    peer=(label=unconfined),
```

An AppArmor-aware bus daemon checks messages against the caller's authenticated
security label and loaded policy before delivering them to `userd`. Thus the lack
of a connection query inside `userd` is not, by itself, an authorization hole.

This establishes permission under the **effective AppArmor policy**, not literally
that one particular plug is connected. Other sufficiently privileged interfaces
can grant equivalent access. Unconfined callers are also outside the restrictive
snap policy. The guarantee depends on functioning D-Bus mediation.

The original `/usr/bin/snap ixr` rule retains the caller's AppArmor confinement.
It does not, by itself, authorize every snap operation. Socket access, daemon API
authorization, filesystem access and profile transitions remain separate checks.
A hostile snap can also bundle a client or construct D-Bus requests directly.
Hiding or restricting the official client is not an adequate security boundary.

### No additional live snapd query is needed for this permission check

Loaded AppArmor policy survives snapd being unavailable. Connecting or
disconnecting an interface updates that policy. Querying snapd for each launch
would duplicate this authorization layer and introduce another availability
dependency.

This does not prove that every existing launch path works with snapd down. For
example, security-profile regeneration after a system-key mismatch can require
the daemon. That is distinct from adding an unconditional authorization round
trip for every desktop launch.

## What 57b436e758 was trying to address

The commit message describes desktop components started by the unconfined
`systemd --user`. Systemd `AppArmorProfile=` overrides were used to confine those
components. This exposed denials during session startup and application launching.

Its additions to `interfaces/builtin/desktop_launch.go` allow:

- executing `snap-confine` with an AppArmor profile transition;
- invoking `snap-seccomp` and reading security-system-key inputs;
- reading other snaps' sequence files;
- creating/accessing snap runtime directories;
- calling user-systemd `StartTransientUnit` and receiving `JobRemoved`;
- additional image-loader and intra-session communication accesses.

The first five items support this direct path:

```text
confined desktop component -> snap run target -> snap-confine -> target
```

They are not required in the caller's profile merely to send a launch request to
an unconfined broker. The commit describes successful functional testing, but
adds no tests demonstrating why these grants are necessary or preserving the
restricted launch boundary.

## Findings

### 1. The helper mistakes an environment variable for confinement state

**Location:** `cmd/snap/cmd_desktop_launch.go:107-145`.

The helper chooses its execution path using:

```go
if os.Getenv("SNAP") != "" {
    // Call userd over D-Bus.
    return nil
}
// Expand the desktop entry and execute snap run locally.
```

A systemd unit with `AppArmorProfile=` can be confined without having passed
through snap's environment setup. If `SNAP` is absent, the helper selects direct
execution despite the process having a confined security label.

This is a plausible explanation for the original denials and the permissions
added by `57b436e758`. **It is not a verified account of the historical failure:**
neither the original failing process environment nor a reproducer was available.

The reverse mismatch also matters. An unconfined process can inherit `SNAP` from
a confined desktop's exported environment. Commit `326973edc7` clears `SNAP` on
the broker-created target specifically to prevent it from re-entering the broker.

**Required direction:** use an explicit brokered client path or reliable
confinement detection rather than treating `SNAP` as proof of confinement. Keep
authorization independent of this client-side routing decision.

### 2. Direct snap-confine access bypasses the desktop-entry API

**Location:** `interfaces/builtin/desktop_launch.go:83-106`.

```apparmor
/usr/lib/snapd/snap-confine px,
/snap/snapd/*/usr/lib/snapd/snap-confine px,
```

`px` permits transition into the executable's AppArmor profile. It is not itself
an unconfined-execution rule, but it exposes a privileged sandbox-setup mechanism
to the confined caller.

The permission is not restricted to launching the command recorded in an
installed desktop entry. If the added grants make direct `snap run target.app`
work as intended, the caller can also request `snap run --shell target.app`.
`cmd/snap/cmd_run.go:1662` passes `--command=shell`, and `cmd/snap-exec/main.go`
selects `/bin/bash` instead of the app command. This is a route toward executing
caller-controlled code under another application's AppArmor permissions, rather
than merely asking that application to start.

No successful cross-snap shell execution was demonstrated. Inherited seccomp
filters and other restrictions can still prevent a particular nested launch.
That is also a correctness problem: an AppArmor transition does not remove an
inherited seccomp filter, so direct execution is not generally equivalent to a
fresh launch by the broker. Although snap-confine has logic to recover the host
mount namespace, that does not reset all inherited confinement.

**Required direction:** fix broker routing and remove these caller-side grants.
Do not expand the interface into a general cross-snap execution facility as a
workaround for launch failures.

### 3. StartTransientUnit is not restricted to tracking scopes

**Location:** `interfaces/builtin/desktop_launch.go:108-116`.

```apparmor
dbus (send)
    bus=session
    path=/org/freedesktop/systemd1
    interface=org.freedesktop.systemd1.Manager
    member=StartTransientUnit
    peer=(label=unconfined),
```

The comment justifies creating a tracking **scope**, but the same method creates
**services** with caller-selected executables. AppArmor's D-Bus rule does not
constrain those method arguments.

With normal user-systemd behavior, this exposes an alternative route to executing
commands through the unconfined user manager. A service started by that manager
does not inherit the requesting snap process's sandbox. The VM's user manager
was confirmed to be unconfined; an actual breakout was not attempted.

This bypass remains even if the desktop helper is narrowed or `userd` gains more
checks. It is granted through the generic `desktop-launch` snippet, not solely
to GNOME Shell. The same rules were observed in Nautilus and snap-store profiles.

**Required direction:** keep target creation on the broker side. Any independent
need for desktop components to manage tracking scopes requires a separately
justified, constrained mechanism.

### 4. Tests do not establish the intended boundary

The existing spread test exercises successful D-Bus launches and the generated
main desktop command. Its disconnected-interface checks concern snapd HTTP
metadata endpoints, not denial of the launch methods themselves. The action test
calls D-Bus directly rather than executing a generated desktop-action command.

`57b436e758` adds no tests for its additional permissions. Successful application
startup alone does not distinguish a correct brokered launch from one that works
because confinement has been broadened.

**Required checks:** see the validation plan below.

### 5. Separate compatibility issue: parallel-instance desktop IDs

Snapd generates desktop filenames such as `basic-desktop+bar_echo.desktop` for
parallel installations. The confined helper sends that basename to `userd`, but
its `isValidDesktopFileID` regexp does not accept the `+` instance separator.

This prevents the brokered launch before execution begins. Existing parallel
installation tests check the generated file contents, not this launch path.

**Fix:** accept snapd's instance separator in the ID validator and exercise a
parallel-instance launch through the confined helper. This was identified from
source; the VM did not provide a parallel-installed desktop app for testing.

## Current VM: evidence and limits of the historical explanation

The VM no longer uses the AppArmorProfile-only startup described by `57b436e758`
for GNOME Shell and its session manager:

- Their systemd service properties have an empty `AppArmorProfile=` setting.
- `ExecStart` invokes `/usr/libexec/gnome-content-launch --session-dispatch ...`.
- The dispatch script executes the corresponding
  `/snap/bin/ubuntu-desktop-session.<app>` entrypoint.
- GNOME Shell has `snap.ubuntu-desktop-session.gnome-shell (enforce)`, active
  seccomp filtering, and `SNAP=/snap/ubuntu-desktop-session/x1`.
- The session manager and Nautilus also have enforcing snap profiles and `SNAP`
  set.

Other enforcing profiles were observed for settings daemons, the desktop portal
and GNOME/GTK portal backends, PipeWire and WirePlumber. The session bus reports
AppArmor support, and its `GetConnectionCredentials` response identifies GNOME
Shell with its actual security label.

The snap-store desktop entry visible inside Shell's mount namespace uses:

```desktop
Exec=/usr/bin/snap routine desktop-launch --desktop /var/lib/snapd/desktop/applications/snap-store_snap-store.desktop -- %U
X-Snap-Exec=snap-store %U
```

These are two different commands. `Exec` is the standard desktop-entry command:
the desktop invokes it, and on the brokered path `userd` reads it and passes it
to `systemd-run --user`. `X-Snap-Exec` records the snap app invocation; `userd`
does not execute it. The broker clears `SNAP` for the transient service, so the
relaunched `snap routine desktop-launch` takes its unconfined branch, expands
`X-Snap-Exec`, and executes `snap run` instead of calling the broker again
(snapd `desktop/26` at `78dca3acd0`, `wrappers/desktop.go:127-159`,
`usersession/userd/privileged_desktop_launcher.go:87-161`,
`cmd/snap/cmd_desktop_launch.go:107-149`). Desktop actions have the same pair
of commands.

An `Exec` line is not a shell command: `FOO=bar app` is not an environment
assignment there ([Desktop Entry Specification](https://specifications.freedesktop.org/desktop-entry/latest/exec-variables.html)).
Although a general desktop entry can invoke `env FOO=bar app`, snapd's entry
rewriter accepts only an `Exec` starting with the snap app wrapper; it rejects
that prefix and may replace the line with the app's default command
(`wrappers/desktop.go:127-176`). The broker's D-Bus environment argument also
allows only `DESKTOP_STARTUP_ID` and `XDG_ACTIVATION_TOKEN`
(`usersession/userd/privileged_desktop_launcher.go:186-200`). Editing the
installed, generated `Exec` to add `env` before the helper could affect the
transient service, but snapd can regenerate that file; this was inferred from
source, not tested.

The session bus lists `io.snapcraft.Launcher` as activatable. Therefore, a normal
launch inheriting today's Shell environment should select the brokered branch.
No actual click was traced, and application-specific environment changes were
not excluded. There is no demonstrated current need for the direct-launch grants
on that path.

## Helper design recommendation

A small, unprivileged dedicated desktop-launch helper is a reasonable separation:

- It sends the constrained request directly to the session D-Bus service.
- It retains the caller's confinement and needs no snap-confine transition.
- The broker validates the request and performs target execution outside the
  caller's sandbox.

This simplifies the executable and policy surface, but it is not a replacement
for authorization at the IPC boundary. A symlink or wrapper into the general
`snap` client does not establish a meaningful binary restriction by itself.

Putting this in `snapctl` is less attractive: its existing model forwards requests
to the system snapd daemon, adding a round trip and requiring a subsequent handoff
to the correct user session.

### Alternative: ship the client in `core26-desktop`

**Observed:** Snapd `desktop/26` at `78dca3acd0` writes
`/usr/bin/snap routine desktop-launch` and `X-Snap-Exec` into generated entries
(`wrappers/desktop.go:127-132`). `userd` expands their `Exec` line and runs it
through user systemd (`usersession/userd/privileged_desktop_launcher.go:87-161`).
The GNOME and KDE image models both specify `core26-desktop` as their base
(`ubuntu-core-desktop` `main` at `cc7069d64915dbd0f8d7cb8c97e2ba410745225a`,
`ubuntu-core-desktop-26-amd64-dangerous.json` and
`ubuntu-core-desktop-kde-26-amd64-dangerous.json`). The base variant supports
shipping additional files (`core-base-desktop` `desktop/26` at
`fea653f29ef06a43d10460c788f0a8aa01687a39`, `variants/README.md`).

**Proposed, not tested:** put a broker-only client (for example,
`confined-snap-launch`) in that base and let `desktop-launch` grant its execution
with the caller's profile (`ix`) and the existing D-Bus request. The executable
is not the authorization boundary: the session bus must still mediate the
caller's label, and `userd` must validate requests. Snapd would select its path
when generating entries **for a device whose model base is `core26-desktop`**,
not based on the application's own base or a distro-name check. The wrapper
currently receives only the application snap (`wrappers/desktop.go:127-132`), so
this choice needs model-aware input from its caller, and existing entries need
regeneration after a model-base change. The path must also be executable inside
callers using other bases, and must not be emitted before the helper is present.

Changing `Exec` alone would cause a loop: the broker would run the entry and
call the client again. Its target launch must instead use validated snap-app
metadata or a trusted, snapd-generated `X-Snap-Exec`, never an arbitrary caller
command. This separates the confined client from the `snap` executable, **not**
from snapd's entry generation, interface policy or `userd` broker. Shipping the
same dedicated client with snapd instead would avoid a model-base-dependent
entry path, at the cost of retaining snapd package coupling. Neither packaging
choice, cross-base execution nor model-change regeneration has been tested.

## Focused validation plan

Use a minimal test snap without unrelated powerful interfaces; GNOME Shell and
snap-store already have other substantial grants and are poor isolation tests.

1. Verify direct D-Bus launch requests are denied without `desktop-launch`,
   allowed with it, and denied after disconnecting it.
2. Verify the client retains the caller's security label and a launched target
   receives its own sandbox rather than the caller's.
3. Exercise a confined caller with `SNAP` unset. It must not fall through to
   privileged local launch setup.
4. Exercise a broker-created process with a polluted session environment to
   ensure it does not recursively call the broker.
5. Verify callers cannot bypass the broker through direct snap-confine execution
   or arbitrary `StartTransientUnit` service creation.
6. Run a generated desktop-action command and a parallel-instance desktop entry.
7. Test a normal, already-installed application launch while snapd is unavailable,
   separately from refresh inhibition or security-profile regeneration cases.
8. If using the base-shipped client, check entry generation on desktop and
   non-desktop models, helper execution from snaps using other bases, and entry
   regeneration after a model-base change.

These system-level tests require explicit approval before changing the VM or
starting/stopping services. None has been performed as part of this review.
