# Reconciling desktop runtimes with the standard Core base

## Status and question

Research and design discussion, recorded 2026-11-20. **No architecture has
been selected or prototyped.** The current goal is to replace
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

See [session-login.md](session-login.md) for the PAM/logind boundary,
[session-startup.md](session-startup.md) for the observed boot-to-session
path, and [session-services-model.md](session-services-model.md) for the
separate host user-manager service graph.

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
2026-11-19 inventory in `session-login.md`, which inspected stock `core26`
revision `750`. Findings must not silently mix those revisions.

Evidence consists of `findmnt`, mountinfo, `losetup`, `snap list/version`,
file listings and comparisons, `systemctl show`, and the boot journal.
The inventory follow-up used successful `unsquashfs -lln` listings of
`core26_646.snap`, `core26-desktop_x1.snap` and `snapd_x1.snap`, without
extraction. These complete metadata listings supersede earlier `find`
results that could not traverse protected directories. No credential
contents were exported. Common-file contents and dependency closure were
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

**Design implication:** making `/etc/pam.d` persistently writable and
seeding it once could hide future base-policy fixes. Reusing the bind-mount
pattern is attractive; blindly reusing one-time factory-copy semantics for
PAM is not. A generated directory needs explicit reconstruction or
reconciliation against the selected base and provider revisions.

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
boundary. Extension classes are assessments explained below, not snapd APIs.
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

### Location-by-location extensibility assessment

**Observation:** stock fstab already exposes `/etc/systemd`, `/etc/dbus-1`,
`/etc/writable`, `/etc/polkit-1/{actions,rules.d}`, `/var/lib/extrausers`,
`/var/lib/console-conf`, `/var/lib/snapd` and `/var/log` as writable paths;
`/run` is runtime storage. Live mounts corroborate the relevant locations.
By contrast, writable `/usr/share/polkit-1` and
`/usr/share/wayland-sessions` mounts are **desktop-variant additions**, not
capabilities to assume on stock `core26`.

The following mechanisms are **candidates**, not functional test results.
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
| `/usr/share/wayland-sessions` | Absent in stock; its writable mount is desktop-only. Prefer a configured display-manager search path if available; otherwise provide a mountpoint or bounded ancestor composition. There are three source entries but only two in the GNOME image. |
| `/usr/share/{gdm,gnome-session,gnome-shell,gnome-settings-daemon,dconf,desktop-provision,i18n,drirc.d,glvnd}` | These directory roots are absent in stock revision 646. Provider-local data/search paths may avoid host grafts; preserving every current absolute path requires missing-parent handling. `/usr/share/X11` is an existing ancestor for `xkb`. |
| Two sysusers snippets and system account records | Delivering `/usr/lib/sysusers.d` files does not create accounts in read-only `/etc` databases. Need Core/extrausers-compatible provisioning and UID/GID ownership across refresh/removal. |
| One tmpfiles snippet | `/run/tmpfiles.d` or an explicit configuration invocation can avoid a `/usr/lib/tmpfiles.d` graft. Run before consumers; merely delivering a snippet after boot setup does nothing. |
| `/var/lib/{AccountsService,gdm3,fprint,colord}`, `/var/cache/colord`, `/etc/{colord,firefox}` | These paths and extra mounts are absent in stock. Prefer provider-owned storage with private layouts where possible, or explicitly register host state directories/mounts. Mutable state is not disposable content. |
| Extrausers, console-conf, snapd skeleton, journal and runtime directories | Existing writable areas need semantic owners and initialization/migration, not broad base grafts. |
| `/usr/share/doc/desktop-payload/provenance` | Build provenance, not a consumer requiring a host path; could remain provider metadata. |

Ninety-three static entries have no immediate parent in the stock image.
This does not imply 93 broad grafts: many are unit drop-in directories that
could instead be created in the existing writable systemd registry.

Sources: Base `variants/desktop/static/`, `snapcraft.yaml`,
`variants/desktop/{fstab.d,writable.conf.d}/`, and the VM image listings,
fstab, mount table and bus configuration. Existing Snapd registration code
is in `wrappers/dbus.go`, `dirs/dirs.go` and
`data/dbus/snapd.{session,system}-services.conf`. The FUSE constraint is in
Base `variants/desktop/static/usr/lib/systemd/system/xdg-fuse-broker.service`.
Provider search-path alternatives require consumer-specific validation.

### Changes an additive-file interface would miss

**Source findings, with selected VM corroboration:**

1. **Account databases:** Base `hooks/000-provide-uids-gids.chroot` adds
   users/groups to existing `/etc/{passwd,shadow,group,gshadow}`: gdm,
   colord, geoclue, pipewire, rtkit and additional groups. These are database
   updates, not new service-file registrations.
2. **NSS:** the variant replaces `/etc/nsswitch.conf`, adding systemd NSS
   and `[SUCCESS=merge]` group semantics alongside `libnss_systemd.so.2`.
   The configuration difference was observed on the VM. Need a generic NSS
   decision or explicit configuration ownership, not only module export.
3. **Existing AppArmor policy:** the variant replaces `unix-chkpwd`, adding
   extrausers passwd/group access and a disconnected socket path. Need a
   generic profile fix or deliberate replacement; a new profile name alone
   does not solve it.
4. **Existing D-Bus policy removal:**
   `hooks/027-remove-locale1-readonly.chroot` removes
   `/usr/share/dbus-1/system.d/org.freedesktop.locale1.read-only.conf`;
   the file is absent in the desktop image. Need an explicit base-policy
   decision or validated override, not simply another added policy file.
5. **OS identity:** `hooks/018-set-os-release.chroot` adds
   `VARIANT_ID=desktop` to existing `/usr/lib/os-release`. Snapd's existing
   `release.OnCoreDesktop` detection uses that value, and the desktop patch
   in `overlord/restart/restart.go` uses it to postpone reboot and notify
   sessions. Stock core26 lacks the marker. Prefer separating desktop
   policy selection from base identity rather than exporting a replacement
   OS identity file by default.
6. **Provisioning and overrides:** extrausers group seeds, console-conf
   completion state and getty/console-conf masks change behavior. Even a
   new pathname can override an existing service. State needs ownership and
   migration rather than an unconditional content copy.
7. **Mount/setup configuration:** existing `/etc/fstab` and
   `/usr/lib/tmpfiles.d/core-writable.conf` gain fragments; hooks create
   factory copies and `/var/log/journal`. A registrar should own the
   resulting setup, not try appending to read-only files at runtime.
8. **Already-upstream fix:** the `pam_extrausers_chkpwd` root:shadow / 2755
   correction remains in the three-dot diff and differs between the two
   installed images, but Core-main already contains the equivalent fix
   (`6b8db3c`, upstream PR #485). It is **not** a remaining reason for
   desktop-specific base extension; stock revision 646 predates it.

**Inference:** additive-only can remain a useful rule for a particular
resource, but cannot cover this entire patch set unchanged. Generic Core
fixes should be separated from optional desktop registration, and remaining
semantic changes need explicit contracts.

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

### Existing versus missing graft points

An existing `/etc/pam.d` can be overmounted without writing to the base or
making `/etc` writable. A new individual file cannot be bind-mounted at
`/etc/pam.d/gdm-password` if its mountpoint is absent on read-only squashfs.
The prepared directory supplies those new directory entries.

A missing destination directory is harder: either provide a generic
mountpoint in the base or reconstruct an existing ancestor with the mimic
pattern. Arbitrarily expanding the reconstruction boundary to `/etc`,
`/usr`, or `/` should not be an implicit consequence of requesting one
file. Existing nested mounts, ownership and symlinks must be preserved.
Thus being outside writable paths is not itself a blocker; destination
existence and the permitted reconstruction boundary are the key questions.

### Generated kernel trees: a host-boot precedent

**Source observation:** Snapd's
`kernel/kernel_drivers.go:EnsureKernelDriversTree()` constructs a
kernel-revision-specific tree with copied metadata and links to kernel and
component content. `cmd/snap-bootstrap/cmd_initramfs_mounts.go:createKernelMounts()`
generates mounts for the source snap/component revisions and bind mounts
for `/sysroot/usr/lib/{modules,firmware}`. Ordering is expressed in
`cmd/snap-bootstrap/initramfs_systemd_mount.go`; first-boot trees under
`_writable_defaults` are handled explicitly.

This supports the proposed separation of **snapd preparing a selected tree**
and **early boot making it visible**. It is not generic host-extension
support already available through an interface.

Do not infer a general multi-resource transaction from this precedent.
The component-only update path swaps firmware and module directories
separately; the source explicitly acknowledges an interruption window
between them. Nor does its use of symlinks establish that identical path
resolution will satisfy a confined PAM consumer.

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
daemon or patching PAM. A GDM snap using layouts is a separate candidate
already discussed in `session-login.md`.

The service-specific bind-mount mechanism is documented in
[systemd.exec][systemd-exec]. In the inspected VM, GDM currently has
`PrivateMounts=no`, no `BindPaths=`/`BindReadOnlyPaths=`, and
`AppArmorProfile=gdm`; this alternative was not in use or tested.

**Advantages:** GDM and its ordinary descendants can receive provider policy
without changing SSH's PAM view. Activation can be associated with the
login provider rather than all host authentication consumers.

**Limits:** unrelated processes, host systemd, polkit and independently
started user services do not automatically share this namespace. Each
consumer must be identified, and the transition into another snap's
namespace must be checked. Modules and helpers must resolve consistently
with the policy. This is a candidate for login-specific resources, not a
complete solution for every desktop integration file.

### 4. Base components as packaging, not yet a composition contract

The earlier moonshot was components attached to the base. In the inspected
Snapd source, `overlord/snapstate/snapstate.go:LinkNewBaseOrKernel()` and
`AddLinkNewBaseOrKernel()` explicitly note that bases do not currently use
components. `snap/types.go` defines standard and kernel-modules component
behavior, but no host-root extension type.

This does not prove that metadata validation rejects every conceivable
base/component combination. The narrower finding is that no established
base-component host-composition contract was identified. Component storage
and revision management alone would not decide graft points, provider
trust, boot selection or PAM lifecycle. This remains a larger snapd design
option, not a shortcut around options 1 or 2.

### Provisional recommendation, not a decision

The inventory refines the initial recommendation into three contracts:

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
3. Test the chosen host-wide or GDM-scoped publication mechanism, including
   common-policy inclusion, module/helper resolution, logind registration,
   keyring unlock and the session-snap handoff.
4. Test absent providers, failed preparation, base-policy changes, logout and
   relogin, refresh/disconnect/removal, reboot and rollback. Verify that
   unrelated login/recovery paths remain available as intended.

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
