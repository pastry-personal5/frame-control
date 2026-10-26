# Phase 5 changelog

Owner decisions, design decisions, and notable changes for [Phase 5](README.md). Newest first.

## 2026-10-20

- **Design decision** [M01]: The canonical-name migration is typed and boundary-specific. It atomically migrates version-2 state fields and queue duplicates, while explicitly versioned version-1 queue snapshots and immutable timing backups remain readable historical evidence. Timing migration first resolves its journal under the existing lock and atomic-file protocol; restoring an old backup canonicalizes and validates it before activation. This preserves queued work and recovery without turning legacy spellings into accepted new job syntax.
- **Owner decision** [M01]: Use `frame-control` for repository and directory naming; use Framecontrol in product prose; retain `framecontrol` for the package, `fc` command, HTTP API, gRPC namespace, caller header, and MCP server. The control API and MCP server already use the protocol identity and receive no alias or breaking rename.
- **Design decision** [M01]: Canonical H3 job pipelines are `i2va` and `ref2va`, with no aliases for `i2av` or `r2av`. Reserve `t2va` for Milestone 02 rather than accepting it early. A forward migration preserves queued work, history, and timing/calibration facts by renaming structured pipeline fields; legacy `h3av` execution modes become `mixed` because they were an umbrella for detailed H3 runs.
- **Owner decision** [M01]: Upgrade checked-in legacy H3 version-1 job definitions to valid version-2 definitions, rename obsolete H3 example paths, and preserve completed plans, releases, research, archives, LICENSE attribution, and prior changelog entries as historical records.

- **Owner decision**: Changes to the directory layout require owner approval.

- **Owner decision**: Keep `architecture.md` at the directory and runtime-boundary level. Its source-layout tree is the responsibility guide; the detailed `Modules` inventory was removed because it duplicated implementation detail and was costly to keep current.

- **Design decision** [M02]: Add H3 `t2va` only to version-2 jobs. A T2VA run passes its positive prompt unchanged, takes no input or references wherever it appears, uses measure-only color handling, and produces a handoff only when the immediately following `i2av`, `i2v`, or `i2i` consumes it.
- **Design decision** [M02]: Integrate T2VA with every execution and history surface and with the timing system from its first release. A cold-start forecast borrows the measured H3 baseline conservatively; completed T2VA runs learn in their own pipeline group. Automated validation is the completion gate, with no required or optional real-host run in the milestone.
- **Owner decision** [M02]: Use Milestone 02 for the second milestone added to Phase 5, leave the carried-over Milestone 08 number unchanged, broaden the phase to H3 text-to-audio-video and graph-based metadata, and make Milestone 08 consume T2VA provenance.
- **Design decision** [M08]: Freeze version 1 as a closed, bounded YAML schema. Unknown fields require a new format version; IDs use only fixed vocabulary and recorded numeric identities, digests have one `sha256:` spelling, and export and offline reading share the same byte, depth, node, and edge limits.
- **Design decision** [M08]: Treat imported resume IDs as external evidence until a two-pass import or explicit repair uniquely maps the child and ancestor in one import batch. Migration preserves the external reference, invalidates only unverified imported local links, and never scans manifests or guesses from a coincidentally equal local execution number.
- **Change** [M08]: Review the first Phase 5 milestone against the completed Phase 4 implementation. Fix its Phase 4 dependency links and specify consistent-read projection, durable artifact transactions, unresolved reason codes, safe atomic CLI output, and deterministic offline trace behavior.
- **Owner decision** [M08]: Move the planned graph-based metadata work from [Phase 4 Milestone 20](../phase-4/phase-4-changelog.md) into Phase 5 as Milestone 08. Its status, scope, and plan are unchanged.
