# RustyPod Behavioral Specifications

![Functions documented](badges/documented.svg) ![Implementation ready](badges/ready.svg) ![Drafts](badges/drafts.svg) ![Source coverage](badges/coverage.svg)

Detailed behavioral contracts for an independent Rust implementation of the iPod Classic firmware environment.

## Repository boundary

This repository contains only reviewed specifications and aggregate progress metadata. Original/decompiled source, raw analysis drafts, source-location maps, agent sessions, and review evidence stay in restricted source-side storage. No previous Rust implementation or Git history is imported.

The source-facing team cannot access the implementation workspace. Future implementation agents receive only reviewed specifications and a fresh repository: https://github.com/gabrielchantayan/rustypod-clean . No implementation workers have been launched.

## Documentation stages

A source-facing author drafts each contract. A separate fresh reviewer checks fidelity, missing behavior, uncertainty, and source leakage. An automated publication gate rejects source-like content and incomplete structure before committing approved documents. Drafts and rejected reviews never enter this repository's history.

Contracts are classified as **documented_only** or **implementation_ready**. Documented does not mean behavior has been executed or validated against hardware. Inferred expectations are explicitly distinguished from static evidence and unresolved questions.

## Progress

Badges and [progress.json](progress.json) update when reviewed contracts are published. Source coverage counts inventoried C input units; this is not a verified count of distinct firmware functions. No draft is counted as a documented function. Runtime validation is tracked separately.

The first batch is a bounded 25-input documentation/review pilot. Broader generation follows inspection of pilot quality, not just a high word count. There is no implementation stage running yet.

## Contract format

Every contract uses a stable opaque function ID and covers purpose, interface and data layout, behavior, side effects, dependencies, acceptance examples, uncertainties, and evidence. Observable behavior is described independently; copied source, statement-by-statement pseudocode, and old implementation code are excluded.

## Feedback across the boundary

Implementation-side questions must refer only to the spec and observable requirements, without implementation code. Source-side replies take the form of spec amendments, never implementation patches. Source authors do not review the new implementation.

## Models and execution

Fresh tool-free source-side author and reviewer sessions use OpenAI GPT-6.1 Sol, with medium and high reasoning respectively. They run in a filesystem-allowlisted sandbox with no shared legacy sessions, tools, MCP servers, extensions, or implementation mounts. Authentication is isolated from the agent's previous working state.
