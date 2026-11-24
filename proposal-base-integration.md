# Base integration without a desktop-specific boot base

## Status and question

**Status: under discussion.** Design discussion recorded 2026-11-20,
extended on 2026-11-24 with session-file publication and brokered-launch
integration.
**No architecture has been selected or prototyped.** The current goal is to replace
`core26-desktop` with the same `core26` boot base used by other Ubuntu Core
systems, without putting GNOME-, KDE-, or future desktop-specific files in
that base.

The missing contract is not simply how to distribute desktop binaries. It
is **how an authorized provider registers files with host consumers that
expect conventional filesystem paths**, with coherent boot, refresh and
rollback behavior. PAM is the concrete example: without changing PAM's
lookup behavior, GDM's worker needs its service policy at
`/etc/pam.d/gdm-*`, alongside the base's common policy.

Here, “extending the base” means extending the **booted host's filesystem
view**. It does not mean modifying the base squashfs, or automatically
changing every application snap that declares `base: core26`.

See [research-session-login.md](research-session-login.md) for the PAM/logind boundary,
[research-session-startup.md](research-session-startup.md) for the observed boot-to-session
path, and [research-session-services-model.md](research-session-services-model.md) for the
separate host user-manager service graph. The proposed login workflow and
connection diagrams are in [proposal-login-high-level.md](proposal-login-high-level.md);
[proposal-login-manager.md](proposal-login-manager.md) owns the detailed
execution alternatives, authority and lifetime contract.

## Evidence and scope

The source revisions, VM versions, read-only inspection details and inventory
boundaries are recorded in
[research-base-integration.md](research-base-integration.md#evidence-and-limits).
Source keys such as Base, Image, Snapd and Snapd-current below refer to that
evidence table; the remote inventory tip is not the VM's snapd build.
Login-provider alternatives also use the source and VM evidence in
[research-session-login.md](research-session-login.md#evidence-and-limits).
No new VM inspection or active experiment accompanied this split or the
2026-11-24 extension. The latter uses the read-only source follow-up and
SSH-access limitation recorded in the
[login-manager proposal](proposal-login-manager.md#status-and-evidence).

## Constraint: no OverlayFS

**Operator-supplied constraint:** OverlayFS is excluded because of reported
snapd confinement/AppArmor problems. Those reports were not independently
reproduced or tied to a specific bug in this investigation.

This supersedes the initial suggestion to prototype a read-only OverlayFS
merge of `/etc/pam.d`. Standard systemd-sysext and systemd-confext also use
OverlayFS, so neither is a workaround under this constraint:

- sysext extends `/usr` and `/opt`, not `/etc`;
- confext extends `/etc`, but still uses OverlayFS;
- service-level `ExtensionDirectories=` likewise uses OverlayFS.

Sources: [systemd-sysext][sysext] and [systemd.exec][systemd-exec]. Plain
`BindPaths=`/`BindReadOnlyPaths=` are distinct mechanisms and remain
candidates. None of these mechanisms is, by itself, a snap authorization
or lifecycle contract.

## Desktop service ownership

The [service-startup research](research-session-services-model.md) records
base-owned overrides, trusted imported units and desktop-name-specific snapd
policy.

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

A new desktop should not need to impersonate `ubuntu-desktop-session`,
`plasma-desktop-session` or `plasma-kwin-wayland` to obtain appropriate
session-management capability. The current app-name selection is a source
finding, not a generic registration contract.

Early preparation and runtime publication are described separately in
[proposal-session-setup.md](proposal-session-setup.md); application launching
is covered by [proposal-desktop-launch.md](proposal-desktop-launch.md).

## Location-by-location integration candidates

The [research inventory](research-base-integration.md#existing-locations-and-consumer-paths)
owns observed path existence and consumer lookup. The following mechanisms
are **candidates**, not functional test results.
None uses OverlayFS. All 326 payload destinations are resolved individually
in the [payload inventory][base-payload].

| Content/location | Assessment |
| --- | --- |
| Seven `/etc/pam.d/gdm-*` policies | Existing read-only directory: compose a complete base+provider view, host-wide or login-service-scoped. Preserve base services and coordinate modules/helpers. |
| Four PAM modules | Multiarch `security` exists, but `/usr/lib/security` does not. Prefer an existing standard module location or provider view where lookup permits; otherwise explicitly handle the missing mountpoint. ABI, dependency and AppArmor checks remain. |
| 120 system/user unit, alias and drop-in entries under `/etc/systemd` and `/usr/lib/systemd` | Prefer existing writable `/etc/systemd/{system,user}` or supported runtime unit paths over making `/usr/lib/systemd` writable. Specify name ownership, masks, precedence, activation and both manager reloads; reconcile the existing `user.control` registration. |
| 19 `/usr/share/dbus-1/services` entries | Existing `/var/lib/snapd/dbus-1/services` is writable and explicitly searched by the live session bus. Reuse snap activation registration where applicable; not every current entry is already expressed that way. Check duplicate-name precedence and systemd mapping. |
| `/usr/share/dbus-1/system.d/gdm.conf` | Existing writable `/etc/dbus-1/system.d` is included by the system bus. Register policy there with controlled loading; the locale1 deny removal is a separate issue. |
| Three `/etc/apparmor.d` profiles | Trusted loading from another path could avoid grafting the directory. Includes, ordering, ownership and unload/replacement matter; `unix-chkpwd` already exists and is changed. |
| Eight polkit actions/rule entries | Investigate existing writable `/etc/polkit-1/{actions,rules.d}` first. The live polkitd binary contains both `/etc` and `/usr/share` action paths; actual loading and precedence were not tested. Writability alone is not evidence that a consumer searches a path. |
| 96 fontconfig entries | `/etc/fonts` and `/usr/share/fontconfig` exist read-only. Compose selected directories or configure provider-local search paths; preserve existing policy and resolve config symlinks. Two entries duplicate existing stock content. |
| Four XDG autostart entries | `/etc/xdg` exists. Composition or configured XDG paths may work, but independently launched consumers must receive the right environment. |
| `/etc/gdm3` configuration and `/etc/writable/gdm3/custom.conf` | `/etc/gdm3` is absent in stock. Existing `/etc/writable` can hold data, but cannot create the expected read-only-root path. Use a configured path, service view, generic mountpoint or bounded parent reconstruction. |
| 18 commands/launchers/shim entries, including generated FUSE broker | Prefer provider paths in generated units/activation files; `/usr/bin` and `/usr/libexec` exist, while `desktop-coreutils` does not. Fixed PAM/helper paths may still require composition. The FUSE broker explicitly requires a host-visible mount namespace. |
| Runtime shared libraries and graphics directories | Existing multiarch root can be composed; some children and data roots are absent. Prefer provider/service views or configurable search paths where consumers permit them. Do not globally merge arbitrary competing library sets. |
| `/usr/share/wayland-sessions` | Absent in inspected stock; its writable mount is desktop-only. For a confined login manager, the proposed provider interface mounts individual session files in its snap namespace, with missing-directory/target preparation. A configured search path or host/service-scoped composition remains an alternative for other packaging. There are three source entries but only two in the GNOME image. |
| `/usr/share/{gdm,gnome-session,gnome-shell,gnome-settings-daemon,dconf,desktop-provision,i18n,drirc.d,glvnd}` | These directory roots are absent in stock revision 646. Provider-local data/search paths may avoid host grafts; preserving every current absolute path requires missing-parent handling. `/usr/share/X11` is an existing ancestor for `xkb`. |
| Two sysusers snippets and system account records | Delivering `/usr/lib/sysusers.d` files does not create accounts in read-only `/etc` databases. Need Core/extrausers-compatible provisioning and UID/GID ownership across refresh/removal. |
| One tmpfiles snippet | `/run/tmpfiles.d` or an explicit configuration invocation can avoid a `/usr/lib/tmpfiles.d` graft. Run before consumers; merely delivering a snippet after boot setup does nothing. |
| `/var/lib/{AccountsService,gdm3,fprint,colord}`, `/var/cache/colord`, `/etc/{colord,firefox}` | These paths and extra mounts are absent in stock. Prefer provider-owned storage with private layouts where possible, or explicitly register host state directories/mounts. Mutable state is not disposable content. |
| Extrausers, console-conf, snapd skeleton, journal and runtime directories | Existing writable areas need semantic owners and initialization/migration, not broad base grafts. |
| `/usr/share/doc/desktop-payload/provenance` | Build provenance, not a consumer requiring a host path; could remain provider metadata. |

Ninety-three static entries lack an immediate parent in stock, but that does
not imply 93 broad grafts: unit drop-ins could use the existing writable
systemd registry. Provider search-path alternatives require consumer-specific
validation; inventory counts do not establish a working implementation.

## Semantic changes and state ownership

The [source findings](research-base-integration.md#changes-an-additive-file-interface-would-miss)
include changes that adding files alone cannot replace. Proposed treatment:

- **Accounts:** use Core/extrausers-aware registration, not file copies into
  account databases; the registrar requirements are below.
- **NSS:** make a generic NSS decision or define explicit configuration
  ownership, not only module export.
- **Existing AppArmor policy:** make a generic `unix-chkpwd` fix or deliberately
  replace the profile; adding a new profile name does not fix the existing one.
- **D-Bus deny policy:** decide the base policy or validate an override for
  locale1; another added file does not by itself reproduce the current removal.
- **OS identity:** prefer separating desktop policy selection from base identity
  rather than exporting a replacement `/usr/lib/os-release` by default.
- **Provisioning and overrides:** own and migrate state instead of unconditional
  copying, including extrausers group seeds, console-conf state and masks.
- **Mount/setup configuration:** a registrar should own the resulting setup,
  not try appending to read-only fstab or tmpfiles configuration at runtime.
- **Already-upstream fixes:** keep the `pam_extrausers_chkpwd` correction out of
  the remaining desktop-extension requirements; stock revision 646 predates it.

Separate generic Core fixes from optional desktop registration. Remaining
semantic changes need explicit contracts even when they use writable paths.

### Service-account registrar

**Candidate, not implemented:** use a trusted account registrar backed by
Core's existing account facilities. It must own name/UID/GID allocation,
collision checks, supplementary groups, filesystem ownership, creation
before consumers start, and persistence across refresh/rollback/removal.
Every relevant host and snap consumer must resolve the intended identity.
Do not overwrite a mutable account database with a runtime's file copy, or
remove identities still owning state or used by another service.

### Authenticated session-launch authority

**Design requirement:** authorize session launch independently of account
registration and PAM-file registration. Bind the target identity and
approved session entrypoint to a successful authentication/session exchange,
not an arbitrary caller-supplied UID. Changing UID does not itself change an
AppArmor label or remove an inherited seccomp filter; a subsequent exec
into another snap must have a validated confinement handoff as well.

**Earlier provisional direction (2026-11-20):** retaining a trusted host-side
login service/worker for PAM, credential changes and launch is a smaller
change than making GDM
an ordinary strictly confined daemon. It can still receive runtime-owned
files through a service-specific mount namespace. That does not remove its
privilege or the need to register accounts, and has not been prototyped.

**Status clarification:** that host-worker preference was an earlier,
lower-change starting point, not a selected architecture. The later
[strict login-manager proposal](proposal-login-manager.md#execution-model) has
two alternatives: retain the leader in login-manager confinement, or retain
a lifetime proxy while a host service launches the provider app in its own
sandbox. Both use independent user-systemd launches for desktop services.
The host-worker alternative and both strict-snap models remain untested;
none has been selected.

## Session-provider publication and launch integration

**Proposal added 2026-11-24, not implemented.** This section owns the
filesystem/registration and packaging requirements. The
[login-manager contract](proposal-login-manager.md#interface-roles) owns
authority and lifecycle; the [overview](proposal-login-high-level.md)
shows the connections and workflow.

### Session-file publication

Use a new interface, provisionally `desktop-session`, with a slot on each
session provider and a plug on the login-manager snap. Its mount backend
would expose individual approved files read-only at
`/usr/share/wayland-sessions/<provider-session>.desktop` in the consumer
snap's mount namespace. This needs a purpose-built mount/registration
contract, not an implicit directory merge by the content interface. It does
not publish files on the host.

Prepare the missing directory and per-file mount targets without OverlayFS
or desktop-specific files in stock `core26`. Bounded namespace construction
is required where parents are absent; existing writable-mimic machinery is
a precedent, not a verified implementation of this interface. Mounts belong
to the consumer snap, not an individual plug; app policy controls access.

Define filename ownership and collision rejection, validate source paths,
file types and symlinks, and bind the visible entry to a registered provider
app and revision. In the brokered model, its `Exec=` invokes the generic
client with a registered session identifier; any executable-availability
checks must resolve inside the consumer namespace. Whether the provider
supplies that entry directly or snapd generates it is still open.

Connection, refresh and disconnection must keep the entry, app registration
and selected revisions consistent. Decide when existing namespaces reload
and how active logins retain their inputs. Removing admission for new logins
must not silently break an existing session's launcher or cleanup path.

### Independent registration and helper packaging

Session-file visibility, permission to launch a session, and host user-unit
registration are separate operations:

- The publication contract associates each entry with a provider app; the
  launch service resolves that registration rather than executing arbitrary
  `Exec=` text as a host command.
- User units/activation metadata must be registered where the independent
  host user manager and bus consume them, before desktop startup. Mounting
  those files into GDM's namespace is not registration.
- For the brokered model, supply a generic launch client at a stable path
  visible inside the login-manager namespace, plus an independent
  system-D-Bus-activated launch service. Register its unit, bus activation
  and access policy through host integration before admitting logins.

A generic client/service could be platform tooling delivered by stock
`core26` or snapd; the packaging owner and client/server compatibility
contract are undecided. A base-owned client still runs confined when GDM
executes it. The server must start outside GDM's inherited sandbox; shipping
another helper for GDM to fork would not provide that boundary. Keep GNOME
names, provider paths and desktop-specific scripts out of generic base
tooling. The existing unconfined session wrapper is not this broker.

The earlier in-sandbox-leader model does not require the launch service.
Both models need provider registration and environment preparation; the
[session-setup proposal](proposal-session-setup.md#proposed-separation)
describes those phases using the earlier model's final-exec choice.
Brokered-launch ordering and failure handling require separate validation.

## Preparing a PAM view

The [writable-mimic](research-base-integration.md#tmpfs-and-bind-mounts-snapds-writable-mimic-precedent)
and [kernel-tree](research-base-integration.md#generated-kernel-trees-a-host-boot-precedent)
mechanisms are precedents, not an existing host PAM-extension API. Their
possible reuse separates snapd preparing a selected tree from boot-time
publication.

**Design implication:** making `/etc/pam.d` persistently writable and
seeding it once could hide future base-policy fixes. Reusing the bind-mount
pattern is attractive; blindly reusing one-time factory-copy semantics for
PAM is not. A generated directory needs explicit reconstruction or
reconciliation against the selected base and provider revisions.

**Candidate:** assemble a complete PAM directory away from its live
location, with base entries and approved provider additions. It could use
copied policy files, or a tmpfs directory containing per-file bind mounts:

```text
prepared PAM directory
  common-auth       <- selected boot base
  common-account    <- selected boot base
  common-session    <- selected boot base
  sshd, login, ...  <- remaining base entries
  gdm-password      <- selected runtime revision
  gdm-...           <- remaining approved provider entries
```

Publish the complete view at `/etc/pam.d`, read-only to consumers. If the
prepared tree contains submounts, publication must preserve them, for
example with a recursive bind. Mounts are reconstructed at boot; a saved
directory alone does not persist a tree of bind mounts across reboot.

Preparing the tree separately avoids exposing an empty live PAM directory
while reconstructing it. It does **not** make a subsequent live switch
transactional across every file open in an authentication flow.

A missing destination can require a generic mountpoint or bounded ancestor
reconstruction; see [the path-existence limits](research-base-integration.md#existing-versus-missing-graft-points).
A request for one file must not implicitly authorize reconstruction of `/etc`,
`/usr` or `/`. Preserve nested mounts, ownership and symlinks.

## Login-provider packaging alternatives

**Design implications of the recorded ownership boundaries, not a tested design:**
Keeping GDM *host-managed* while using stock `core26` still requires PAM
policy visible to its worker, a host-visible `gdm.service`, GDM privilege
policy, greeter account/seat setup and a desktop-entry/launcher handoff.
Host management does not require the worker to share the host mount
namespace; the service-scoped alternative is option 3 below. A content snap
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
`systemd --user` or provide the separate session-file publication contract.
It also requires review of PAM module availability (the inspected stock `core26` has no
`pam_gnome_keyring.so`), host account access, logind/seat privileges,
UID/GID and supplementary-group changes under AppArmor/seccomp, refresh
behavior and session bootstrap. Retaining today's direct transition from
GDM's child into the *different* `ubuntu-desktop-session` snap would need
separate validation. The strict-login proposal instead retains the leader
inside GDM's sandbox or delegates launch to an independent host service.
The existing snapd `gdm` interface permits **clients** to call an existing
GDM; `login-session-control` grants specified logind D-Bus operations.
Neither establishes that a confined GDM worker can run the complete
PAM/login flow. No prototype was tested.

**Further candidate (2026-11-20):** host-managed GDM could instead receive a
prepared PAM directory through systemd's service-specific
`BindReadOnlyPaths=`, without becoming a snap daemon. This would give GDM
and its ordinary descendants a composed policy view, not automatically
change the host user manager or other independent consumers. The read-only VM
follow-up and mechanism precedents are in
[research-base-integration.md](research-base-integration.md); the no-OverlayFS
constraint and proposed provider interfaces are in this document. This
alternative is untested.

**Conclusion:** PAM is not intrinsically incompatible with a GNOME runtime
snap. The unresolved architecture covers **account registration, login
policy registration, and authority to launch the authenticated user's
session**, with explicit trust and lifecycle rules for each. Removing the
desktop base requires maintained host integration, not merely moving files
into the runtime squashfs. The earlier recommendation to keep the worker
host-managed remains a lower-change alternative, not a requirement imposed
by GDM. With an explicitly superprivileged `login-manager` contract, a strict
snap retaining a confined leader or a confined lifetime proxy is
architecturally plausible. Neither strict-snap model requires an unconfined
exec in GDM's process tree or a direct nested snap launch. The brokered model
adds an independent host launch service, not a sandbox-resetting GDM child.
The host-managed alternative and both strict-snap models need validation.

The strict-snap proposal should first validate account/group visibility,
greeter/password login, UID/GID/supplementary-group transitions, PAM/logind
registration, keyring handling, uninterrupted leader confinement and
independent desktop-snap startup. Logout/relogin, reused user managers,
scope/device-policy interactions and runtime refresh/rollback on stock
`core26` are also required. See the
[interface validation checklist](proposal-login-manager.md#validation-before-claiming-support).

## Discussed design options

All interface names below are illustrative proposals, not existing snapd
APIs. A provider publishing host policy is privileged even if its own apps
are strictly confined. A normal content connection or snap layout does not
publish files into the host namespace.

### 1. A generic, superprivileged runtime-extension interface

**User proposal:** an interface declares runtime content to be grafted onto
the booted base. Connection authorizes snapd to generate a tree in writable
storage, subject to rules such as prohibiting replacement of existing
files. `snap-bootstrap` or another early service exposes that tree before
switch-root. Kernel-component integration is the precedent.

**Advantages:** desktop-neutral; supports new window managers and desktop
environments without baking their names and files into `core26`; one
registration and lifecycle mechanism can cover several integration paths.

**Trade-off:** this grants OS-extension authority. Additive-only is a useful
collision rule, not a security boundary: new units, drop-ins, polkit rules
or PAM policies can introduce privileged behavior without overwriting an
existing file. The contract needs explicit authorization, bounded graft
points, ownership rules, and validation of source paths, file types and
symlinks. Model-selected platform providers are a possible trust policy,
not a decided requirement.

An interface would authorize and describe the operation; a new host/boot
integration backend would still be needed. It is not enough to add mount
rules to an ordinary snap consumer namespace.

### 2. Specialized resource interfaces with a shared backend

**User proposal:** interfaces such as `pam-support` extend particular host
resources, rather than permitting arbitrary destination paths.

A hypothetical `pam-service-provider` could declare service names and
policy files. Snapd could preserve base-owned `common-*` and existing login
services, reject duplicate ownership, and relate provider removal to its
consumers. Supplying policy does not by itself solve PAM module, library,
helper, account-access or confinement requirements.

**Advantages:** a smaller authorization contract, resource-specific
validation and failure behavior, and no need for one interface per desktop.
GNOME, SDDM and another login manager could share the same PAM contract.

**Trade-off:** resources have different consumers and activation rules. The
inventory above distinguishes existing registration paths, actual
composition needs, and semantic state changes; interfaces should follow
those boundaries rather than become one-off exports for every directory.
Narrow scope does not make authentication policy low-risk. Where composition
is needed, it can use the same backend as option 1.

### 3. A login-provider contract with a service-scoped filesystem view

**Alternative discussed:** compose the PAM/configuration view for the login
provider's service instead of extending the whole host. A host-managed GDM
could receive the prepared directory at `/etc/pam.d` through systemd's
`BindReadOnlyPaths=`. This does **not** require converting GDM into a snap
daemon or patching PAM. A GDM snap using layouts is the separate candidate
discussed under
[login-provider packaging](#login-provider-packaging-alternatives) and the
[login-manager proposal](proposal-login-manager.md).

The [GDM mount-namespace observation](research-base-integration.md#gdm-mount-namespace-on-the-inspected-vm)
records that the inspected service did not use this mechanism. The alternative
was not tested; systemd's [service-specific bind-mount documentation][systemd-exec]
does not establish a complete login contract.

**Advantages:** GDM and its ordinary descendants can receive provider policy
without changing SSH's PAM view. Activation can be associated with the
login provider rather than all host authentication consumers.

**Limits:** unrelated processes, host systemd, polkit and independently
started user services do not automatically share this namespace. Each
consumer must be identified, and the transition into another snap's
namespace must be checked. Modules and helpers must resolve consistently
with the policy. Account registration and authority to change UID/GID and
supplementary groups remain separate requirements; neither a mount namespace
nor a successful PAM exchange removes confinement restrictions. This is a
candidate for login-specific resources, not a complete solution for every
desktop integration file.

### 4. Base components as packaging, not yet a composition contract

The earlier moonshot was components attached to the base. The
[source finding](research-base-integration.md#base-component-support-limits)
is that no established base-component host-composition contract was identified,
not that every conceivable metadata combination was proved invalid.

Component storage and revision management alone would not decide graft points,
provider trust, boot selection or PAM lifecycle. This remains a larger snapd
design option, not a shortcut around options 1 or 2.

### Provisional recommendation, not a decision

The inventory and credential follow-up refine the recommendation into four
contracts:

1. **Registration into existing writable consumers:** systemd units,
   D-Bus activation/policy, generated confinement policy, and potentially
   polkit. Avoid new `/usr` grafts where existing registration suffices.
2. **Selected filesystem composition:** PAM and genuinely fixed helper,
   module or data paths. Combine option 2's resource authorization with a
   shared backend from option 1; evaluate option 3 where the consumer set
   is limited to the login provider.
3. **Semantic operations and state ownership:** accounts/NSS, existing host
   profiles, base deny policy, provisioning state, OS identity and mutable
   storage. A new-file-only interface cannot implement these unchanged.
4. **Authenticated session-launch authority:** establish the target user's
   UID/GID and supplementary groups, then enter the intended session
   confinement. Registering an account or delivering PAM policy does not
   grant this authority. Retaining the host-side login worker was the earlier
   lower-change preference, not a selection. The strict login-manager models
   retain either the leader or its lifetime proxy inside GDM confinement;
   compare them in the [login-manager proposal](proposal-login-manager.md#execution-model).

Start composition validation with PAM, not arbitrary filesystem grafting.
Keep generic Core fixes out of the optional desktop payload. Reserve a
generic OS-extension privilege for an explicit decision to trust selected
providers with broad host integration.

Copying small policy files into a newly generated tree is not excluded;
per-file bind mounts are an alternative. The important distinction from
factory copying is that the generated view has explicit inputs, ownership
and lifecycle, rather than becoming an unmanaged persistent snapshot.

## Lifecycle and ordering questions shared by the options

These are proposed requirements, not properties of an implemented design:

- **Select a generation:** bind policy to the boot-base revision, provider
  revisions and authorized registrations. Do not follow `current` links
  opportunistically. A base refresh may make a previously valid addition
  collide with a new base file; validate before activation.
- **Stage separately from activation:** connection/refresh can authorize and
  prepare a generation without immediately replacing live authentication
  policy. Boot-only activation is a plausible first version. Pending
  activation, disconnection and removal semantics must be visible to users.
- **Retain inputs for rollback:** mounted or selected provider revisions
  must not be garbage-collected merely because ordinary snap apps no longer
  use them. A raw-executed host daemon is not automatically equivalent to a
  tracked snap-app consumer.
- **Preserve authority and integrity:** only the trusted compositor should
  control generated host policy; ordinary provider apps must not mutate it.
  Decide how generated artifacts in writable storage are validated or
  reconstructed. Collision-free content is not necessarily safe content.
- **Preserve a coherent policy/module/helper set:** a directory switch alone
  does not prove consistency for existing workers or namespaces. Define
  consumer restart, session draining, or reboot behavior before live updates.
- **Preserve identities and session authority:** reconcile account/group
  allocation and filesystem ownership before startup; retain identities
  needed by existing sessions and rollback generations. Account availability
  and permission to launch as that account must be validated independently.
- **Handle first boot and recovery:** pre-switch-root composition cannot
  assume normal post-boot interface connection has already happened. Source
  snaps and authorization must be available at the chosen stage. Missing or
  invalid desktop integration should not unnecessarily prevent console/SSH
  recovery; exact fail-closed and fallback behavior remains to be designed.
- **Order policy loading as well as file visibility:** AppArmor profiles,
  units, accounts and other metadata have consumers with different startup
  times. Making a file visible does not necessarily load or activate it.

Pre-switch-root activation is one choice, not a universal requirement.
The requirement is **before the relevant consumers start**. A main-system
service before GDM may suffice for PAM and avoid initrd desktop-provider
handling. Resources needed by early unit discovery, generators or account
setup may require earlier composition or explicit later setup/reloads.
No exact boot dependency graph for desktop exports has been established.

## Smallest useful validation

Proposed next experiment, requiring separate approval for active testing:

1. On a disposable stock-`core26` image, prepare a PAM view from the actual
   boot base and a test provider's policy, without OverlayFS. First inspect
   it in an isolated mount namespace, not over the live host PAM directory.
2. Check preservation of all base entries, collision rejection, permissions,
   source revision selection and AppArmor behavior at the consumer paths.
   Verify service-account and group registration, consistent account lookup,
   UID/GID allocation and ownership without rewriting the base databases.
3. Test the chosen host-wide or GDM-scoped publication mechanism, including
   common-policy inclusion, module/helper resolution, logind registration,
   keyring unlock, UID/GID/supplementary-group transitions and the confinement
   handoff into the session snap.
4. Test absent providers, failed preparation, base-policy changes, logout and
   relogin, refresh/disconnect/removal, reboot and rollback. Verify that
   unrelated login/recovery paths remain available as intended.

For the proposed session-file interface, also verify per-file visibility on
stock `core26`, absent-target preparation, collisions, entry/app revision
agreement and namespace updates. Broker packaging adds activation-policy,
client/server compatibility and retained-revision checks; launch and lifetime
tests belong to the
[login-manager validation](proposal-login-manager.md#validation-before-claiming-support).

Success at file visibility is only the first gate. No claim is made here
that a bind-mounted PAM view already works through the complete confined
login path, or that removing `core26-desktop` is solved by PAM alone.

[base-comparison]: https://github.com/canonical/core-base/compare/main...canonical:core-base-desktop:desktop/26
[base-payload]: evidence/reconcile-with-base-2026-11-20/base-payload.tsv
[base-other-source]: evidence/reconcile-with-base-2026-11-20/base-other-source.tsv
[snapd-source]: evidence/reconcile-with-base-2026-11-20/snapd-source.tsv
[vm-desktop-only]: evidence/reconcile-with-base-2026-11-20/vm-desktop-only.tsv
[sysext]: https://www.freedesktop.org/software/systemd/man/latest/systemd-sysext.html
[systemd-exec]: https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html
[tmpfiles]: https://www.freedesktop.org/software/systemd/man/latest/tmpfiles.d.html
