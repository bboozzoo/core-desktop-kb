# Core Desktop knowledge base

Evidence-based notes on Ubuntu Core Desktop internals: confinement, session
startup, launch brokering, base-snap integration. Every claim carries the
evidence it came from.

## Rules

- Update the existing documents whenever new evidence is discovered. Never
  leave a stale claim next to a newer observation: edit it in place, or mark it
  superseded. Prefer extending a document over adding a near-duplicate one.
- Cite evidence: repository + branch + commit SHA, file paths, and for VM
  findings the host, snapd version and date. Say whether inspection was
  read-only or active, and what was not verified.
- Keep observation separate from inference.

## Sources

Repositories are checked out under `/home/maciek/work/canonical/core-desktop/`,
on the `desktop/26` branch unless noted.

| Local directory | Remote | What it is |
| --- | --- | --- |
| `ubuntu-core-desktop` | `canonical/ubuntu-core-desktop` | image and model build (GNOME, KDE) |
| `core-base-desktop` | `canonical/core-base-desktop` | core26 base snap: units, hooks, `variants/`, generators |
| `ubuntu-desktop-session-snap` | `canonical/ubuntu-desktop-session-snap` | strictly confined desktop session snap |
| `snapd-work-desktop` | `canonical/ubuntu-core-desktop-snapd` | snapd desktop worktree (git worktree of `../snapd`) |
| `snapd` | `canonical/snapd` (`master`) | upstream snapd, reference only |

## Scope

- Analysis and review notes only. Upstream repositories are read-only
  references; nothing here is their source of truth.
- Do not commit image artifacts, VM disks or other large binaries. Keep
  credentials, keys and tokens out of the repository.

## Documents

- `desktop-launch.md` — desktop launcher confinement model, broker versus
  direct launch, review findings.
- `session-services-model.md` — how GNOME session components actually get
  started and confined.
