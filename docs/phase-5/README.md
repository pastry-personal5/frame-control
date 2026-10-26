# Phase 5: H3 text-to-audio-video and graph-based metadata

**Status:** planned

## Goal

Add H3 text-to-audio-video generation to version-2 jobs, then produce and consume graph-based metadata that records trustworthy provenance for generated artifacts.

## Scope

- Run independent H3 text-to-audio-video (`t2va`) generations, with verified audiovisual output and conditional visual handoffs, in Milestone 02.
- Export deterministic, bounded provenance graphs from settled execution history in Milestone 08.
- Validate and trace a saved graph offline, with gaps in old, imported, or pruned history represented explicitly rather than inferred.

## Non-goals

- Replacing SQLite, manifests, or queue snapshots with a graph database.
- Adding a version-1 T2VA mode, rewriting H3 prompts, or requiring real-host inference as a phase gate.

## Milestones

| # | Milestone | Status |
|---|-----------|--------|
| 01 | [Cleanup and canonical H3 terms](milestone-01-cleanup.md) | planned |
| 02 | [H3 text-to-audio-video support](milestone-02-h3-t2va.md) | planned |
| 08 | [Graph-based metadata](milestone-08-graph-metadata.md) | planned |

Milestone 08 was the first milestone added to this phase and kept its number when it moved from Phase 4. Milestone 01 first makes the established identities and H3 pipeline terms consistent; Milestone 02 then adds T2VA, which Milestone 08 consumes as provenance.

## Depends on

- [Phase 4](../phase-4/README.md) Milestones 02, 03, 05, 06, and 09 provide the backend, H3 media, reference, color, handoff, and per-run contracts extended by Milestone 02 and projected by Milestone 08.
- Phase 4 Milestones 10, 11, 14, and 15 provide the timing, forecast, external-knob, validation, and tuning system that Milestone 02 extends to T2VA.

## Changelog

Decisions and notable changes are recorded in [phase-5-changelog.md](phase-5-changelog.md).

## Exit criteria

- Version-2 jobs run independent H3 T2VA generations through every front end, preserve native audiovisual output, hand a verified final frame to a following visual-input run when needed, and forecast the pipeline as defined in Milestone 02.
- The authenticated API and CLI produce identical, deterministic, bounded version 1 YAML from a consistent history snapshot.
- Saved graphs are strictly validated and can be traced without the server, database, media files, or network.
- Missing, ambiguous, imported, or pruned evidence is explicit; projection never guesses provenance from a path, filename, command, or schedule position.
- T2VA behavior and examples, and the graph format, privacy boundary, compatibility policy, and retention behavior, are documented as defined in Milestones 02 and 08.
