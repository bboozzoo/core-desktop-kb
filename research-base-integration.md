# Desktop integration with the standard Core base: research

## Scope and question

This document records the 2026-11-20 source and VM investigation of what the
desktop base supplies, how host consumers see it, and which composition and
registration mechanisms already exist. It distinguishes source/VM observations
from inferences and limits; no replacement architecture was tested.

The design goal is stock `core26` without desktop-specific boot-base files.
[proposal-base-integration.md](proposal-base-integration.md) owns the possible
contracts, alternatives, operator constraints and proposed experiments.
Here, extending the booted host's filesystem view is distinct from modifying
the base squashfs or changing every app declaring `base: core26`.

## Evidence and limits

Source inspection was read-only. Source keys used below:

| Key | Repository | Branch | Commit |
| --- | --- | --- | --- |
| Base | `canonical/core-base-desktop` | `desktop/26` | `fea653f29ef06a43d10460c788f0a8aa01687a39` |
| Image | `canonical/ubuntu-core-desktop` | `main` | `cc7069d64915dbd0f8d7cb8c97e2ba410745225a` |
| Snapd | `canonical/ubuntu-core-desktop-snapd` | `desktop/26`, local/VM-source revision | `78dca3acd0dcc04b738a24b07e9dae7c7dc66c72` |
| Snapd-current | `canonical/ubuntu-core-desktop-snapd` | `desktop/26`, remote inventory tip | `e01549bbb9fc2b42eeadd0c6d25b01dc50f09a97` |
| Core-main | `canonical/core-base` | `main`, inventory reference | `26c0e62161574d1bd2d9d93de6c22b11286e46f8` |
| Snapd-main | `canonical/snapd` | `master`, inventory reference | `7651fe52f1df2f149f19aeb9771b03b61ff398f5` |

The remote snapd desktop tip was 15 commits newer than the local checkout
and VM build. The branch inventory below uses **Snapd-current**; earlier
mechanism findings use **Snapd** unless stated otherwise. Newer source
changes, notably accessibility-provider, are not observations of the
installed snapd's behavior. Remote objects were inspected in temporary
reference repositories without changing the source worktrees.

VM inspection was read-only on **2026-11-20**, over `ubuntu@vsock%59467`,
hostname `ubuntu-Standard-PC-Q35-ICH9-2009`. Versions observed:

- snapd: `2.76.3+g78dca3a.78dca3a`;
- systemd: `259.9-0ubuntu0.1`;
- kernel: `7.0.0-38-generic`, `pc-kernel` revision `3755`;
- `core26-desktop`: revision `x1`, version `20261108+desktop`;
- `gnome-desktop-runtime`: revision `x1`, version `0.1`;
- stock `core26`: revision `646`, version `20260925`, installed but not the
  boot base.

This is a new VM boot observation, not a correction of the dated
2026-11-19 inventory in `research-session-login.md`, which inspected stock `core26`
revision `750`. Findings must not silently mix those revisions.

Evidence consists of `findmnt`, mountinfo, `losetup`, `snap list/version`,
file listings and comparisons, `systemctl show`, and the boot journal.
The credential follow-up on the same date, host and snapd version also
inspected `ps` identities/labels, `/proc/PID/status`, `getent`, NSS and
GDM's host profile. Its source/observation distinction and GDM provenance
are recorded in [research-session-login.md](research-session-login.md#credential-transition-and-confinement).
The inventory follow-up used successful `unsquashfs -lln` listings of
`core26_646.snap`, `core26-desktop_x1.snap` and `snapd_x1.snap`, without
extraction. These complete metadata listings supersede earlier `find`
results that could not traverse protected directories. No authentication
secrets were exported. Common-file contents and dependency closure were
not exhaustively compared; metadata differences are not content diffs.
The destination inventory is amd64-specific; no arm64 payload was inspected.
Separate runtime/provider repositories were not inventoried.

No mount changes, service restarts, deliberate authentication/login/logout
tests, refreshes or rollbacks were performed. No stock-`core26` boot image
was tested. Noninteractive sudo was unavailable; the boot journal was
readable, but the installed kernel's protected initrd payload was not
extracted. The initrd source below is therefore a source explanation
consistent with the journal, not a byte-for-byte match to the installed
initrd.

Upstream [systemd-sysext][sysext], [systemd.exec][systemd-exec] and
[tmpfiles.d][tmpfiles] documentation supplies mechanism references. These
are moving `latest` manuals, not a build match to the VM's systemd. The
latter two were checked on 2026-11-20.

## Observed problem: payload ownership is not host registration

The desktop variant supplies GDM PAM service files, its systemd unit and
AppArmor profile, session-launch wrappers, user-unit overrides, and extra
writable-mount setup. Relevant Base paths include:

- `variants/desktop/static/etc/pam.d/gdm-*`;
- `variants/desktop/static/etc/apparmor.d/gdm`;
- `variants/desktop/static/usr/lib/systemd/system/gdm.service`;
- `variants/desktop/static/etc/systemd/system/gdm.service.d/50-gnome-content.conf`;
- `variants/desktop/static/usr/bin/{gnome-session,gnome-keyring-daemon}`;
- `variants/desktop/fstab.d/desktop.conf`.

The keyring wrapper is a particularly useful example: its runtime binary
has moved out of the base, but PAM's helper path, library lookup and
AppArmor handoff still require integration. Moving the executable alone
did not eliminate the base-side contract.

The Image repository's `build.sh:kde_sddm_integration()` copies `sddm`,
`sddm-autologin` and `sddm-greeter` PAM policies into the image root and its
factory tree. That is image-time integration, not registration when a
runtime snap is installed or connected.

**VM observation:** `/etc/pam.d/gdm-*` are regular files in the read-only
root. The installed runtime's `etc/pam.d` is empty. The root's inspected
`common-auth`, `common-account` and `common-session` are byte-identical to
those in installed stock `core26/646`. The runtime would first need an
appropriate export payload; this research does not assume it already has
one.

**Inference:** the current patch set demonstrates a missing generic host
integration mechanism, not proof that a separate desktop boot base is
intrinsically necessary. Hard-coding more provider names or copying each
new desktop's policies into the image does not establish that mechanism.

## How writable paths work on this boot

**Observed mounts:**

| Visible path | Backing storage |
| --- | --- |
| `/`, including `/etc/pam.d` | `/dev/loop0`, read-only squashfs |
| `/etc/systemd` | `/dev/vda5[/system-data/etc/systemd]`, writable ext4 bind mount |
| `/etc/writable` | `/dev/vda5[/system-data/etc/writable]`, writable ext4 bind mount |
| `/var/lib/snapd` | `/dev/vda5[/system-data/var/lib/snapd]`, writable ext4 bind mount |
| `/writable` | `/dev/vda5`, writable ext4 |

`losetup` identified `/dev/loop0` as the seeded desktop-base squashfs under
`/run/mnt/ubuntu-seed/systems/20261108/snaps/`. No OverlayFS mount was
reported in the inspected SSH process's mount namespace; other process
namespaces were not exhaustively inspected.

The journal records:

1. `sysroot-writable.mount` mounted `/sysroot/writable`.
2. `initrd-core-tmpfiles-setup.service` created initial writable directories.
3. `populate-writable.service` was skipped because
   `/sysroot/etc/system-image/writable-paths` did not exist.
4. Selected paths, including `/sysroot/etc/writable`, were mounted before
   switch-root.

**Source explanation:** Snapd's
`core-initrd/26.04/factory/usr/lib/systemd/system/initrd-core-tmpfiles-setup.service`
runs `systemd-tmpfiles --root=/sysroot --create core-writable.conf`.
Base's `hooks-build/100-factory-files.build` prepares factory copies;
`static/usr/lib/tmpfiles.d/core-writable.conf` declares the initial copies
and directories; `static/etc/fstab` declares bind mounts, with
`x-initrd.mount` for the selected `/etc` paths. The live `/etc/systemd`
mount unit was generated from `/etc/fstab`.

Neither the inspected stock writable-path configuration nor the desktop
variant adds `/etc/pam.d` as a writable mount. The root remains read-only;
only explicitly mounted locations gain a different backing filesystem.

### Initialization is not reconciliation

The plain tmpfiles `C` operation copies into an absent or empty destination;
it does not update an already populated directory from the next base
revision. See [tmpfiles.d][tmpfiles] and Base's
`hooks/032-move-ssh-config.chroot`, which documents an existing remodeling
problem: new SSH unit links were not copied because the writable
`/etc/systemd` directory already existed.

**Inference:** a one-time populated writable PAM directory could hide later
base-policy fixes. Existing initialization behavior does not establish an
update or reconciliation contract.

## Branch and installed-payload inventory

### Comparison boundaries and retained evidence

The [requested base comparison][base-comparison] is a three-dot comparison
from Core-main to Base. Its merge base is
`224ca346ae65367f63dd7a416bacba428ee1b623`. The Snapd-main to Snapd-current
comparison has merge base `58163ecc3eae8d0581d893934115eb354d85f85e`.
Counts below describe changes from those merge bases, not a two-dot
subtraction of the latest upstream tree. In particular, a patch can remain
in the comparison even after an equivalent upstream cherry-pick.

**Source observation:**

| Comparison | Added source paths | Modified | Deleted |
| --- | ---: | ---: | ---: |
| Core-main → Base | 335 | 18 | 0 |
| Snapd-main → Snapd-current | 39 | 74 | 1 test launcher |

The 335 base additions comprise **319 static variant entries** (193
regular files and 126 symlinks, including `.keep` scaffolding) and 16
other source files: build machinery, hooks, broker source, tests and docs.
Another **seven installed file destinations** are generated by the recipe.
A new source path is not necessarily a new host path.

The compact, per-file inventories are retained with this document:

| Inventory | Coverage |
| --- | --- |
| [Base payload destinations][base-payload] | All 319 static entries plus seven generated destinations, with source, installed path, type/link target, stock/desktop presence, nearest stock ancestor and proposed extension class. |
| [Other base source changes][base-other-source] | The remaining 16 added and 18 modified source paths, with installed effects. Together with the payload table, this covers every changed base source path. |
| [Snapd source changes][snapd-source] | All 114 changed paths, including the installed-effect classification for every added file. |
| [Desktop-only VM paths][vm-desktop-only] | All 385 additional regular-file/symlink paths and 119 directories, with source/image/version attribution. |

These are dated snapshots, not automatically maintained manifests.
`A`, `M`, and `D` are Git statuses; a blank payload source status/mode
identifies a recipe-generated entry. Image types are `-` for a regular
file, `l` for a symlink, `d` for a directory, or `absent`. The nearest-stock-
ancestor column records an existing lexical ancestor, not a tested graft
boundary. The retained `base-payload.tsv` column `extension_class` contains
proposed integration assessments, not observed capabilities or snapd APIs;
see the [candidate approaches](proposal-base-integration.md#location-by-location-integration-candidates).
Source/image origin classes are attributions, not raw observations. The dated
TSV artifacts are unchanged by this split.
Stock presence and fstab conclusions refer to **revision 646**, not a new
build of Core-main. Table counts and unique destinations were checked.

### Static destinations, generated files and image differences

Six static destinations already exist in stock `core26/646`:

| Destination | Comparison with the desktop image |
| --- | --- |
| `/etc/apparmor.d/unix-chkpwd` | Changed policy. |
| `/etc/nsswitch.conf` | Changed NSS configuration. |
| `/var/lib/extrausers/group` | Changed group seed. |
| `/var/lib/extrausers/gshadow` | Same inspected contents. |
| `/etc/fonts/conf.d/71-ubuntulegacy.conf` | Same symlink target. |
| `/usr/share/fontconfig/conf.avail/71-ubuntulegacy.conf` | Same inspected contents. |

Every static destination except
`/usr/share/wayland-sessions/plasma-desktop-session.desktop` is present in
the installed desktop snap. Image `build.sh` removes that entry for GNOME.
This verifies path presence, not a byte match of every source file.

Base `snapcraft.yaml` builds/stages these seven additional destinations:

```text
/usr/bin/xdg-fuse-broker
/usr/lib/security/pam_gdm.so
/usr/lib/x86_64-linux-gnu/security/pam_fprintd.so
/usr/lib/x86_64-linux-gnu/security/pam_gnome_keyring.so
/usr/lib/x86_64-linux-gnu/security/pam_systemd_loadkey.so
/usr/lib/x86_64-linux-gnu/libnss_systemd.so.2
/usr/share/doc/desktop-payload/provenance
```

All seven are present in the desktop image and absent in stock revision
646. `tools/apply-variant-payload.sh` applies static and generated payload,
substituting `@MULTIARCH@` with `x86_64-linux-gnu` for this architecture.
Other hooks alter existing files, preserve otherwise-trimmed files, or
create directories; those effects are not hidden by this seven-file list.

**VM observation and source attribution:** complete image enumeration found
385 desktop-only regular-file/symlink paths (250 regular files, 135
symlinks), plus 119 desktop-only directories:

| Attribution | Regular-file/symlink paths |
| --- | ---: |
| Direct destinations from static variant payload | 312 |
| Generated destinations above | 7 |
| Factory copies/seeds | 51 |
| Image serial-console integration | 5 |
| Upstream profile/locale/completion evolution | 6 |
| Certificate-package version differences, not attributed to the desktop branch | 4 |
| Total | 385 |

The bases have different build dates. Their payload difference is **not**
the desktop branch diff. The five console entries correspond to Image
`fix-console`; Image `build.sh` also enables the polkit helper socket in
the factory tree. The 51 factory additions include copies of existing
base policy and are not 51 independent host-integration requirements.
Profile/locale/completion additions correspond to upstream base changes
already in the branch's merge base. Certificate differences are recorded
as package-version drift, not claimed as desktop patches.

### Existing locations and consumer paths

**Observation:** stock fstab already exposes `/etc/systemd`, `/etc/dbus-1`,
`/etc/writable`, `/etc/polkit-1/{actions,rules.d}`, `/var/lib/extrausers`,
`/var/lib/console-conf`, `/var/lib/snapd` and `/var/log` as writable paths;
`/run` is runtime storage. Live mounts corroborate the relevant locations.
By contrast, writable `/usr/share/polkit-1` and
`/usr/share/wayland-sessions` mounts are **desktop-variant additions**, not
capabilities to assume on stock `core26`.

| Content/location | Observed basis and limits |
| --- | --- |
| Seven `/etc/pam.d/gdm-*` policies | The parent exists read-only; the desktop policies are absent from stock. |
| Four PAM modules | Multiarch `security` exists; `/usr/lib/security` does not. |
| 120 system/user unit, alias and drop-in entries | Destinations are under `/etc/systemd` and `/usr/lib/systemd`; stock has writable systemd configuration paths. |
| 19 `/usr/share/dbus-1/services` entries | The live session bus also searches writable `/var/lib/snapd/dbus-1/services`; not all current entries use snap registration. |
| `/usr/share/dbus-1/system.d/gdm.conf` | The system bus includes writable `/etc/dbus-1/system.d`; the locale1 deny removal is a separate source change. |
| Three AppArmor profiles | `unix-chkpwd` already exists in stock and is changed. |
| Eight polkit actions/rule entries | The live polkitd binary contains both `/etc` and `/usr/share` action paths; loading and precedence were not tested. Writability does not establish consumer lookup. |
| 96 fontconfig entries | `/etc/fonts` and `/usr/share/fontconfig` exist read-only; two entries duplicate stock content. |
| Four XDG autostart entries | `/etc/xdg` exists. |
| GDM configuration | `/etc/gdm3` is absent in stock; `/etc/writable` exists, but does not create that read-only-root path. |
| 18 command/launcher/shim entries, including the generated FUSE broker | `/usr/bin` and `/usr/libexec` exist; `desktop-coreutils` does not. The broker requires a host-visible mount namespace. |
| Runtime libraries and graphics directories | The multiarch root exists; some children and data roots are absent. |
| `/usr/share/wayland-sessions` | Absent in stock; its writable mount is desktop-only. Three source entries become two in the GNOME image. |
| `/usr/share/{gdm,gnome-session,gnome-shell,gnome-settings-daemon,dconf,desktop-provision,i18n,drirc.d,glvnd}` | Roots are absent in stock revision 646. `/usr/share/X11` is an existing ancestor for `xkb`. |
| Two sysusers snippets and system account records | Delivering snippets does not itself update read-only `/etc` account databases. |
| One tmpfiles snippet | Delivering a snippet after boot setup does not execute it. |
| `/var/lib/{AccountsService,gdm3,fprint,colord}`, `/var/cache/colord`, `/etc/{colord,firefox}` | Paths and extra mounts are absent in stock; these include mutable state. |
| Extrausers, console-conf, snapd skeleton, journal and runtime directories | These use existing writable areas. |
| `/usr/share/doc/desktop-payload/provenance` | Build provenance rather than a runtime consumer's required host path. |

The [candidate integration table](proposal-base-integration.md#location-by-location-integration-candidates)
uses these observations; its suggested mechanisms are not tested capabilities.

Ninety-three static entries have no immediate parent in the stock image.
Many are unit drop-in directories. This is a count of missing immediate
parents, not a count of independently required graft boundaries.

Sources: Base `variants/desktop/static/`, `snapcraft.yaml`,
`variants/desktop/{fstab.d,writable.conf.d}/`, and the VM image listings,
fstab, mount table and bus configuration. Existing Snapd registration code
is in `wrappers/dbus.go`, `dirs/dirs.go` and
`data/dbus/snapd.{session,system}-services.conf`. The FUSE constraint is in
Base `variants/desktop/static/usr/lib/systemd/system/xdg-fuse-broker.service`.
No provider search-path alternative was functionally validated.

### Changes an additive-file interface would miss

**Source findings, with selected VM corroboration:**

1. **Account databases:** Base `hooks/000-provide-uids-gids.chroot` adds
   users/groups to existing `/etc/{passwd,shadow,group,gshadow}`: gdm,
   colord, geoclue, pipewire, rtkit and additional groups. These are database
   updates, not new service-file registrations.
2. **NSS:** the variant replaces `/etc/nsswitch.conf`, adding systemd NSS
   and `[SUCCESS=merge]` group semantics alongside `libnss_systemd.so.2`.
   The configuration difference was observed on the VM.
3. **Existing AppArmor policy:** the variant replaces `unix-chkpwd`, adding
   extrausers passwd/group access and a disconnected socket path; this
   changes an existing profile, not only the set of available profile names.
4. **Existing D-Bus policy removal:**
   `hooks/027-remove-locale1-readonly.chroot` removes
   `/usr/share/dbus-1/system.d/org.freedesktop.locale1.read-only.conf`;
   the file is absent in the desktop image.
5. **OS identity:** `hooks/018-set-os-release.chroot` adds
   `VARIANT_ID=desktop` to existing `/usr/lib/os-release`. Snapd's existing
   `release.OnCoreDesktop` detection uses that value, and the desktop patch
   in `overlord/restart/restart.go` uses it to postpone reboot and notify
   sessions. Stock core26 lacks the marker.
6. **Provisioning and overrides:** extrausers group seeds, console-conf
   completion state and getty/console-conf masks change behavior. Even a
   new pathname can override an existing service.
7. **Mount/setup configuration:** existing `/etc/fstab` and
   `/usr/lib/tmpfiles.d/core-writable.conf` gain fragments; hooks create
   factory copies and `/var/log/journal`.
8. **Already-upstream fix:** the `pam_extrausers_chkpwd` root:shadow / 2755
   correction remains in the three-dot diff and differs between the two
   installed images, but Core-main already contains the equivalent fix
   (`6b8db3c`, upstream PR #485). It is **not** a remaining reason for
   desktop-specific base extension; stock revision 646 predates it.

**Inference:** additive-only can remain a useful rule for a particular
resource, but cannot cover this entire patch set unchanged. Generic Core
fixes and semantic changes are distinct from optional file registration;
[the proposal](proposal-base-integration.md#semantic-changes-and-state-ownership)
addresses their possible ownership.

### Snapd additions are mostly code and generated policy

At Snapd-current, the 39 added source files comprise:

- 13 interface implementations: accessibility-provider, accountsservice,
  bolt, fprintd, fuse-device, gdm, gdm-session-control, geoclue, kde-kauth,
  pam-unix-chkpwd, session-environment-broker, systemd-user-control and
  systemd-user-environment;
- `cmd/snap/cmd_desktop_launch.go`, compiled into the existing snap command;
- 23 tests/fixtures and two CI workflows.

None of those new Go source files is a new host `/etc` or `/usr` payload
file. Permission rules do not themselves create or expose the paths they
name. Relevant installed effects, including changes in existing sources:

| Destination/effect | Observation and implication |
| --- | --- |
| `/snap/snapd/<rev>/usr/bin/snap` | New desktop-launch subcommand in an existing binary, not a base graft. |
| `/snap/snapd/<rev>/usr/lib/libseccomp.so.2*` | Recipe stages libseccomp 2.6.0; observed in installed snapd. Architecture-specific recipe patterns also exist. This stays inside snapd, not the boot base's library tree. |
| `/var/lib/snapd/apparmor/profiles/*`, `/var/lib/snapd/seccomp/bpf/*` | Existing writable generated-policy stores, observed on the VM; interfaces extend policy through the existing backends. |
| `/var/lib/snapd/apparmor/home-dirs` | New Core-specific tunable output in the desktop patch, observed on the VM. It deliberately avoids writing into read-only `/etc/apparmor.d/tunables/home.d`. |
| `/var/lib/snapd/desktop/applications/*.desktop` | Existing store; branch changes generated `Exec`/`X-Snap-Exec` content and launch behavior. |
| `/etc/dbus-1/{system.d,session.d}` and `/var/lib/snapd/dbus-1/{services,system-services}` | Existing writable policy/activation stores; live bus configuration includes the service directories. |
| `$XDG_CONFIG_HOME/systemd/user/{default,sockets,timers}.target.wants` | Newer tip creates missing parent directories before enabling units; user state, not boot-base payload. This change postdates the VM's snapd build. |
| Paths such as `/run/xdg-fuse-broker.sock`, `/run/gdm3`, `/var/lib/AccountsService`, `/var/lib/fprint`, `/var/lib/boltd`, `/etc/geoclue` | New interface rules refer to these; provider layouts or host services must separately supply the paths. Access permission is not host registration. |

Sources: Snapd-current `build-aux/snap/snapcraft.yaml`,
`sandbox/apparmor/apparmor.go`, `interfaces/apparmor/{backend,template,spec}.go`,
`interfaces/builtin/*.go`, `usersession/agent/rest_api.go` and
`wrappers/desktop.go`; existing stores/activation mechanisms were also
inspected in Snapd `dirs/dirs.go` and `wrappers/dbus.go`.

## Accounts and session-launch authority

File visibility, account provisioning and permission to change process
credentials are distinct contracts. A synthesized `/etc/passwd` could in
principle be grafted, but would still require account-management semantics;
it would not make account creation or credential changes an ordinary
content-export operation.

### Registering service identities

**Source and VM observation:** Base `hooks/000-provide-uids-gids.chroot`
adds `gdm` (200), `colord` (202), `geoclue` (203), `pipewire` (204) and
`rtkit` (205), with matching primary group IDs, plus groups such as
`scanner`, `provd` and `gnome-initial-setup`. Their presence was checked
through `getent` on the 2026-11-20 VM. The initial-setup group does not imply
that a fixed user with that name is baked into the base.

Stock `core26/646` already uses `files extrausers` for passwd, group and
shadow lookup. The live `ubuntu` account is in
`/var/lib/extrausers/passwd`. Snapd's
`overlord/snapstate/check_snap.go:checkAndCreateSystemUsernames()` selects
extrausers on Core; `osutil/user.go:EnsureSnapUserGroup()` uses the
extrausers-aware group/user creation commands. Thus additional globally
resolvable accounts need not intrinsically be baked into squashfs.

### Supported snap users are not an arbitrary login-user contract

**Source-derived at Snapd, not a claim about every snapd version:**
`snap/system_usernames.go:SupportedSystemUsernames` contains a small
hard-coded allowlist, including `snap_daemon`, `_daemon_` and selected
snap-specific users. Installation validation rejects unknown names and
currently rejects `private` and `external` scopes; only `shared` is
supported in this inspected code. Simply declaring `gdm` is not an existing
account-registration API.

For declared system usernames, the AppArmor backend adds privilege-dropping
capabilities and the seccomp backend generates argument-filtered
UID/GID/chown rules for those identities and root. Without a declaration,
the baseline includes some unfiltered credential-changing syscalls and
relies on AppArmor capability mediation; connected interfaces can supply
additional permissions. Therefore this is **not a blanket prohibition on
snaps changing UID**, but neither is it a supported general-purpose login
launcher. Sources: Snapd `overlord/snapstate/check_snap.go` and
`interfaces/{apparmor,seccomp}/{backend,template}.go`.

Service identities such as `gdm` are also different from the set of
**authenticated interactive users**. Adding a service identity to an
allowlist would not by itself authorize login as arbitrary users or set
all of their supplementary groups.

### The current login transition is outside ordinary snap confinement

**Observed result and source-derived explanation:** the host-managed root
GDM worker performs the group/UID transition before entering the session
snap. GDM's source calls `setgid()` and `initgroups()` during accreditation,
then the forked session child calls `setuid()` before exec. PAM performs
its authentication/credential/session work within the caller's security
context; successful authentication is not a confinement exemption.

The 2026-11-20 snapshot showed a UID 0/GID 1000 worker, a UID/GID 1000
Wayland helper under the host `gdm` profile, and a UID/GID 1000 session
init-worker under the session snap's profile. The inspected `gdm3`, worker
and Wayland-helper processes had no seccomp filter; the host profile
explicitly grants setuid/setgid capabilities. This supports the current
boundary, not the feasibility of performing the same flow inside a strictly
confined GDM snap. See
[the source and process evidence](research-session-login.md#credential-transition-and-confinement);
no credential-changing syscall was traced.

## Available composition mechanisms

### Tmpfs and bind mounts: snapd's writable-mimic precedent

**Source observation:** Snapd's
`cmd/snap-update-ns/utils.go:planWritableMimic()`:

1. recursively bind-mounts the original directory aside;
2. mounts tmpfs over the original path;
3. bind-mounts original regular files and recursively bind-mounts original
   directories back into the tmpfs, recreating symlinks separately;
4. releases the temporary preservation mount.

`execWritableMimic()` has rollback handling, and
`interfaces/apparmor/spec.go:GenWritableMimicProfile()` generates policy for
these mount operations. Tests are in `cmd/snap-update-ns/utils_test.go`.
This is an existing no-OverlayFS composition pattern **inside snap mount
namespaces**, not an existing host PAM-extension facility.

### Existing versus missing graft points

An existing `/etc/pam.d` can be overmounted without writing to the base or
making `/etc` writable. A new individual file cannot be bind-mounted at
`/etc/pam.d/gdm-password` if its mountpoint is absent on read-only squashfs.
A missing directory also has no mountpoint on the read-only parent. The
mimic pattern can reconstruct an existing ancestor, but that affects a wider
filesystem boundary, including existing nested mounts, ownership and symlinks.
Thus being outside writable paths is not itself a blocker; destination
existence and the reconstruction boundary are separate constraints.
[The PAM-view proposal](proposal-base-integration.md#preparing-a-pam-view)
considers prepared directory entries, generic mountpoints and bounded
reconstruction; none was tested for host PAM here.

### Generated kernel trees: a host-boot precedent

**Source observation:** Snapd's
`kernel/kernel_drivers.go:EnsureKernelDriversTree()` constructs a
kernel-revision-specific tree with copied metadata and links to kernel and
component content. `cmd/snap-bootstrap/cmd_initramfs_mounts.go:createKernelMounts()`
generates mounts for the source snap/component revisions and bind mounts
for `/sysroot/usr/lib/{modules,firmware}`. Ordering is expressed in
`cmd/snap-bootstrap/initramfs_systemd_mount.go`; first-boot trees under
`_writable_defaults` are handled explicitly.

This is a precedent for revision-selected tree preparation and early-boot
mounting, not generic host-extension support already available through an
interface. Its possible reuse is discussed in the proposal.

Do not infer a general multi-resource transaction from this precedent.
The component-only update path swaps firmware and module directories
separately; the source explicitly acknowledges an interruption window
between them. Nor does its use of symlinks establish that identical path
resolution will satisfy a confined PAM consumer.

## GDM mount namespace on the inspected VM

The service-specific bind-mount mechanism is documented in
[systemd.exec][systemd-exec]. In the inspected VM, GDM currently has
`PrivateMounts=no`, no `BindPaths=`/`BindReadOnlyPaths=`, and
`AppArmorProfile=gdm`. The
[service-scoped PAM-view alternative](proposal-base-integration.md#3-a-login-provider-contract-with-a-service-scoped-filesystem-view)
was not in use or tested.

## Base-component support limits

In the inspected Snapd source,
`overlord/snapstate/snapstate.go:LinkNewBaseOrKernel()` and
`AddLinkNewBaseOrKernel()` explicitly note that bases do not currently use
components. `snap/types.go` defines standard and kernel-modules component
behavior, but no host-root extension type.

This does not prove that metadata validation rejects every conceivable
base/component combination. The narrower finding is that no established
base-component host-composition contract was identified. Component storage
and revision management alone would not decide graft points, provider
trust, boot selection or PAM lifecycle. Any future base-component composition
contract belongs to
[the design discussion](proposal-base-integration.md#4-base-components-as-packaging-not-yet-a-composition-contract).

## Composition mechanism limits

Standard systemd-sysext and systemd-confext use OverlayFS. Sysext extends
`/usr` and `/opt`, not `/etc`; confext extends `/etc`. Service-level
`ExtensionDirectories=` also uses OverlayFS. Plain `BindPaths=` and
`BindReadOnlyPaths=` are distinct mechanisms. Sources:
[systemd-sysext][sysext] and [systemd.exec][systemd-exec]. This is mechanism
research, not an independent reproduction of the operator's reported
snapd/AppArmor problems with OverlayFS.

## Related proposals and unverified work

[Base integration](proposal-base-integration.md) owns host registration,
composition, account provisioning and lifecycle alternatives.
[Login-manager](proposal-login-manager.md) owns the strict-login-provider
contract; [session setup](proposal-session-setup.md) owns generic preparation.
No stock-base boot, composed PAM view, full login flow, refresh or rollback
was tested here; see the evidence limits above and the
[proposed validation](proposal-base-integration.md#smallest-useful-validation).

[base-comparison]: https://github.com/canonical/core-base/compare/main...canonical:core-base-desktop:desktop/26
[base-payload]: evidence/reconcile-with-base-2026-11-20/base-payload.tsv
[base-other-source]: evidence/reconcile-with-base-2026-11-20/base-other-source.tsv
[snapd-source]: evidence/reconcile-with-base-2026-11-20/snapd-source.tsv
[vm-desktop-only]: evidence/reconcile-with-base-2026-11-20/vm-desktop-only.tsv
[sysext]: https://www.freedesktop.org/software/systemd/man/latest/systemd-sysext.html
[systemd-exec]: https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html
[tmpfiles]: https://www.freedesktop.org/software/systemd/man/latest/tmpfiles.d.html
