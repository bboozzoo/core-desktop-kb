# Core Desktop knowledge base

Evidence-based research and proposals on Ubuntu Core Desktop internals:
confinement, session startup, launch brokering and base-snap integration.
Research claims carry evidence; proposals state their status and assumptions.

## Local notes

If `AGENTS-LOCAL.md` exists in this directory, read it and follow it. It holds
machine-specific paths and context that do not belong in this repository.
`AGENTS-LOCAL.md` must never be committed.

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

## Document naming and ownership

- `research-<topic>.md` — observations, source analysis, explicitly labelled
  inferences, inspection limits and open research questions. Distinguish
  working-demo observations from source-derived explanations.
- `proposal-<topic>.md` — proposed changes, alternatives, trade-offs,
  dependencies, open decisions and validation criteria. Record whether the
  proposal is under discussion, selected, superseded or rejected; selection
  is not evidence of implementation.
- Use lowercase, hyphen-separated topics. Keep documents at the repository
  root; research and proposals need not have a one-to-one pairing.
- Give each finding and proposal one primary home. Proposals link to supporting
  research, and research links to related proposals. Prefer links over repeated
  detailed accounts; short, clearly attributed evidence summaries are fine.
- Keep recommendations and proposed remedies in proposal documents, not inside
  research findings. Preserve research limitations and unanswered checks in
  research; put a proposed change's acceptance tests with its proposal.
- A proposal is not evidence of current behavior. Preserve alternatives and
  their status when consolidating; do not silently turn a preference into a
  decision or discard an untested option.
- Keep dated artifacts under `evidence/` with their provenance. Distinguish
  observed fields from source attributions and proposed assessments; being in
  `evidence/` does not make every field an observation. Do not silently rewrite
  a dated snapshot to reflect a newer design.
- When splitting or renaming, preserve information, source revisions, VM dates
  and versions, inspection limits and open questions. Update document links,
  heading anchors and the README index, and check the migrated content.

[README.md](README.md) is the document index, grouped into research and proposals.
`README.md`, `AGENTS.md` and the uncommitted `AGENTS-LOCAL.md` are navigation or
working instructions, not topic documents.

## Design goals

- Drop the `core26-desktop` base snap and use stock `core26` as the boot base,
  like other Ubuntu Core systems, without desktop-specific files in that base.
