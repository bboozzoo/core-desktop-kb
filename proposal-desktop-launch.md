# Desktop launch: broker-only client and policy changes

## Status and basis

Under discussion; not implemented or validated here. The proposal was developed
with the 2026-11-18/19 review recorded in
[research-desktop-launch.md](research-desktop-launch.md#scope-and-evidence).
That document owns the source revisions, VM findings and inspection limits.
No new VM inspection, helper build or active test accompanied this split.

## Proposed behavior

### Broker-only launching

**Recommendation:** make confined desktop launches consistently use the broker,
then remove the direct-launch permissions from `desktop-launch`. Do not grant
callers unrestricted user-systemd `StartTransientUnit` access to make launching
work.

### Explicit client routing

**Required direction:** use an explicit brokered client path or reliable
confinement detection rather than treating `SNAP` as proof of confinement. Keep
authorization independent of this client-side routing decision.

### Remove direct-launch grants

**Required direction:** fix broker routing and remove the
[direct-launch caller-side grants](research-desktop-launch.md#what-57b436e758-was-trying-to-address).
Do not expand the interface into a general cross-snap execution facility as a
workaround for launch failures.

### Keep target creation on the broker side

**Required direction:** keep target creation on the broker side. Any independent
need for desktop components to manage tracking scopes requires a separately
justified, constrained mechanism.

### Parallel-instance desktop IDs

**Fix:** accept snapd's instance separator in the ID validator and exercise a
parallel-instance launch through the confined helper. See
[the source finding and VM limit](research-desktop-launch.md#5-separate-compatibility-issue-parallel-instance-desktop-ids).

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

**Status of this alternative:** retained for comparison, not selected. It
assumes keeping `core26-desktop` and therefore does not meet the current goal
of using stock `core26`. The snapd-shipped alternative below avoids that
particular dependency. Current entry generation, model bases and variant
support are recorded in [the packaging research](research-desktop-launch.md#current-helper-packaging).

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

## Packaging questions still open

The earlier discussion also asked whether a dedicated helper in the snapd
snap would be too large, and how changes to the generated desktop-entry
format would remain compatible across snapd updates. These concerns were not
measured or resolved. A dedicated binary versus `snap routine desktop-launch`,
entry rewriting and model-base-aware selection remain design choices, not
verified improvements.
