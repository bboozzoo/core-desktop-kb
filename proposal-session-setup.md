# Generic session setup through user systemd

## Status and basis

Under discussion; no helper, base change or dependency change has been
implemented or tested here. The proposal follows the 2026-11-20 source review
in [research-session-startup.md](research-session-startup.md#evidence-and-scope),
which records the earlier read-only VM observations separately. That document
owns the source revisions, current wrapper/broker behavior and ordering evidence.

## Proposed separation

**Proposal, not current behavior:** split the wrapper's desktop integration
from the confined login manager. A small, generic setup helper and user unit
could live in stock `core26`; provider-specific configuration and service
graphs would remain with the desktop. This would avoid granting
`login-manager` all of the wrapper's environment, unit-directory and socket
setup permissions. It is not a proposal to move the wrapper unchanged into
the base, or to run all desktop setup unconfined.

### Three phases, with different owners

| Phase | Proposed owner and work | Required timing |
| --- | --- | --- |
| Provider registration | Trusted snapd/host integration registers approved desktop units, activation metadata and provider configuration, with revision ownership. | Complete registration and any required manager reload before requesting the desktop startup transaction. |
| Per-login preparation | A base-owned user-systemd helper consumes validated login context and registered desktop configuration, then prepares shared environment and any known runtime mappings. | Finish before the desktop services that consume those values start. |
| Runtime publication | The compositor/audio provider and any narrowly authorized helper or broker publish actual display endpoints and credentials. | After the relevant producer is ready, before clients relying on those values are launched. |

The confined leader retains its own environment setup and session-lifetime
protocol. It submits the selected session and bounded login context, then
requests the registered desktop target; it does not have to install unit
files or copy compositor credentials itself. A preparation unit pulled into
that target's dependencies runs under the independent user manager, not in
the login manager's process tree or inherited sandbox.

The wrapper's work would be assigned as follows:

- **Static snap discovery:** reuse the existing user-service `environment.d`
  mechanism for `PATH` and `XDG_DATA_DIRS`; see the
  [source precedent](research-session-startup.md#existing-static-snap-environment).
- **Selected desktop policy:** keep desktop names, `GSETTINGS_BACKEND`,
  runtime/provider identities and endpoint mappings in provider configuration.
  GNOME/KDE names, `DISPLAY=:0` and the literal `pipewire` snap name should
  not become fixed behavior of a generic base helper.
- **Initial shared environment:** move validation, selected stale-state
  cleanup and publication to per-login preparation. Preserve the difference
  between [D-Bus empty values and systemd unsets](research-session-startup.md#initial-environment-the-base-wrapper).
- **User-unit exposure:** replace the whole-directory `user.control` link
  workaround with owned registration; do not give the login manager general
  unit-directory replacement authority.
- **Masked-unit and portal stops:** replace blanket stopping with explicit
  desktop-owned lifecycle handling where still needed. These are not generic
  setup operations to copy into the base helper.
- **Socket links and Xauthority:** use provider mappings for links with known
  destinations; publish or copy producer-created credentials after creation.
  A symlink can be prepared early but does not prove the socket is ready.
- **Final exec:** keep the selected leader inside login-manager confinement,
  rather than retaining the wrapper's direct cross-snap launch.

Source for the current operations: the
[base wrapper research](research-session-startup.md#initial-environment-the-base-wrapper). This
assignment is proposed ownership, not an existing snapd contract.

### Preparation gate and GNOME ordering

The [existing ordering research](research-session-startup.md#existing-pre-session-target-and-gnome-ordering)
records the confined migration unit and tagged GNOME dependencies. Proposed
per-login preparation would precede that chain:

```text
per-login preparation (proposed)
  → graphical-session-pre.target
  → gnome-session-pre.target
  → gnome-session-manager@….service
  → gnome-session-manager.target
  → org.gnome.Shell@….service
```

This is not the complete dependency graph. SDDM's selected desktop needs its
own integration; the display manager does not impose GNOME's user-unit graph.

For mandatory preparation, a candidate dependency on the applicable
pre-target is:

```ini
# Illustrative drop-in; no such helper is implemented here.
[Unit]
Requires=core-desktop-session-prepare.service
After=core-desktop-session-prepare.service
```

The helper would be a `Type=oneshot` user service. `Before=`/`After=` order
jobs but do not pull them in. `WantedBy=` creates a weak dependency when
enabled; it does not make preparation failure fatal. With `Requires=` and
`After=`, a failed preparation service prevents the dependent target from
starting. Missing mandatory inputs must fail preparation, not merely skip it
through a unit condition. The full desktop failure/cleanup behavior still
needs testing. See [systemd.unit(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html).

Register the dependency only for the intended desktop workflow; do not run
per-login preparation at every user manager's `default.target`. Consumers
must be ordered after the gate: it does not delay unrelated services or
retroactively update a portal or other service already running.

### Ordering and session-context constraints

**Registration comes first.** A service inside the desktop's startup
transaction cannot reliably supply the unit definitions needed to construct
that transaction. Prefer registration at provider activation. If selecting
units must happen per-login, complete a separate registration/reload phase
before requesting the desktop target. Do not rely on `daemon-reload` inside
a pre-target service to rebuild the transaction already being started.

**No implicit environment handoff.** A helper started by user systemd does
not inherit the newly authenticated GDM child's environment. Pass current
session identity, locale, seat/VT and other required values through a bounded,
validated handoff tied to the selected user/session. The transport and data
schema remain open. Publishing those values to the manager does not repair
the already-running confined leader's environment or namespace. Reuse the
existing broker where suitable, but its current authorized caller and fixed
paths are GNOME/session-snap specific, not a generic login-leader or KDE API.

**Do not wait for the compositor before starting it.** The wrapper backgrounds
`fixup_xauthority()` because it waits for a Mutter/KDE-produced file. Turning
that wait into mandatory pre-session preparation would make the compositor
wait for its own output. Prefer producer-side publication or a separately
supervised task with readiness, failure and logout handling. The same rule
applies to waiting for sockets produced by services ordered after preparation.

**Per-login, not once per UID lifetime.** SSH or linger can preserve the user
manager across logouts. Define when preparation reruns, how stale session
context is rejected, and how units and shared environment are reset. A unit
left active with `RemainAfterExit=yes` is not by itself a per-login trigger.
The existing one-graphical-session-per-UID assumptions remain; generic names
or template instances do not isolate a shared manager environment.

### What could live in the base

**Candidate base responsibility:** a fixed helper/unit for early session
preparation, validated inputs, controlled shared-environment changes and
bounded runtime mappings, with explicit failure and cleanup behavior. It
could execute as the user outside login-manager confinement, with authority
to perform its defined setup; the per-user work does not inherently need
root. Base ownership alone is not an input-validation or confinement policy.

**Provider responsibility:** desktop identity/settings, service dependencies,
runtime mappings, migrations and compositor-specific endpoint/credential
publication. Ordinary provider hooks can be separate user units launched
through confined snap entrypoints, as the
[migration-service evidence](research-session-startup.md#existing-pre-session-target-and-gnome-ordering) shows.

A base helper that accepts arbitrary commands or sources provider shell code
unconfined would instead grant arbitrary host execution as that user. That
is a separate trust decision, not the bounded configuration mechanism proposed
here. Trusted registration must also review what imported units execute;
read-only provider files do not make arbitrary `ExecStart` commands safe.

The intended result is a smaller
[login-manager contract](proposal-login-manager.md#scope-and-dependencies):
authentication, Unix credentials, login registration, a bounded setup/start
handoff and session-lifetime IPC. Unit installation, broad environment writes,
blanket recovery stops and credential-copy polling would not become general
powers of the login-manager app.

**Validation still required:** verify the installed unit graph and failure
propagation, preparation before consumers, fresh and reused user managers,
logout/relogin and provider switching, Xauthority/socket readiness and cleanup,
provider refresh/removal, and separate GNOME/KDE behavior. No generic helper,
base changes, systemd transaction or active login test was run for this proposal.

The current source also retains direct shared-environment grants outside the
broker. Remove inappropriate grants before claiming an exclusive filtered
update path; the [policy mismatch](research-session-startup.md#intended-separation-versus-the-inspected-policy)
is not resolved merely by introducing a preparation helper.
