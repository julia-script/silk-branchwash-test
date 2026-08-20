# language-server-synchronization Specification

## Purpose

Defines how the Silk language server keeps project analysis consistent with editor buffers and
filesystem changes while bounding redundant work and preventing stale protocol results.
## Requirements
### Requirement: Open documents form project-scoped overlays

The language server SHALL group synchronized documents by discovered Silk project, SHALL resolve an
open document's bytes before the corresponding on-disk file, and SHALL isolate overlays belonging
to different projects. A document outside a project SHALL use a stable isolated workspace identity
that cannot collide with another unrelated document.

#### Scenario: Unsaved imported module

- **WHEN** an open module imports another open module whose editor contents differ from disk
- **THEN** analysis uses the imported module's synchronized editor contents

#### Scenario: Same module spelling in different projects

- **WHEN** two open projects each contain a module with the same canonical name
- **THEN** each project's analysis resolves only its own module and overlay

#### Scenario: Standalone virtual document

- **WHEN** a synchronized document cannot be assigned a canonical filesystem module
- **THEN** the server gives it a stable isolated identity rather than sharing a fallback identity with another document

### Requirement: Analysis scheduling is bounded and latest-wins

The language server SHALL coalesce bursts of changes at the trailing edge per project, SHALL run no
more than one analysis job at a time for a project, and SHALL eventually analyze the newest
synchronized revision without first completing every superseded revision. Superseded work MUST NOT
replace a newer completed document snapshot or publish protocol results after a newer revision has
been accepted. Successful completion, typed failure, defect, and interruption SHALL all finalize the
active worker state, settle revision waiters that can no longer succeed, and allow the newest pending
revision to run. Only a complete atomic commit MAY become the reuse basis for later work.

#### Scenario: Rapid edits during analysis

- **WHEN** several revisions arrive while project analysis is queued or running
- **THEN** the server supersedes obsolete queued work, preempts or disregards obsolete active work, analyzes the newest pending revision next, and does not start one concurrent analysis per revision

#### Scenario: Older analysis finishes last

- **WHEN** analysis for an older revision finishes after the server has accepted a newer revision
- **THEN** the older result neither replaces the current session nor publishes diagnostics

#### Scenario: Valid revision follows an analysis defect

- **WHEN** analysis of revision N terminates with an internal defect and valid revision N+1 is accepted
- **THEN** the scheduler finalizes revision N, analyzes and publishes revision N+1, and remains able to process subsequent revisions

#### Scenario: Failed work is not reused

- **WHEN** one revision fails or is interrupted after producing partial frontend work
- **THEN** the next analysis starts from the last complete atomic commit rather than the partial failed revision

### Requirement: Protocol queries use a coherent document snapshot

The language server SHALL answer a semantic request from a document and analysis snapshot produced
for the same synchronized revision. If that revision has not completed analysis, the request SHALL
wait only for its own project scheduler to commit the applicable snapshot or SHALL return no
semantic result when that revision is superseded, fails, or is interrupted. Every request waiter
MUST settle when its exact revision can no longer commit, and a request MUST NOT interpret a current
editor position against an older or newer document snapshot.

#### Scenario: Definition requested during refresh

- **WHEN** a definition request arrives after a document change but before analysis of that change completes
- **THEN** the server answers from analysis of that exact captured revision or returns no location if it is superseded, without consulting an older or newer document snapshot

#### Scenario: Atomic snapshot replacement

- **WHEN** a new project analysis completes
- **THEN** all document text, line indexes, module identities, and semantic facts from that result become queryable together

#### Scenario: Request waits on a defective revision

- **WHEN** a semantic request is waiting for a revision whose analysis defects
- **THEN** the request settles without a semantic result rather than remaining pending indefinitely

#### Scenario: Unrelated project is analyzing

- **WHEN** a semantic request targets one project while another project has queued or active document updates
- **THEN** the request synchronizes only with its target project and is not delayed by the unrelated work

### Requirement: Diagnostics describe only the latest accepted revision

The language server SHALL publish diagnostics only from the newest accepted analysis for each open
document, SHALL include the analyzed document version when the protocol permits it, and SHALL clear
diagnostics when the document closes. A dependency change SHALL refresh diagnostics for affected
open documents even when their own text did not change. If the current revision cannot produce an
analysis result, the server SHALL retire diagnostics published for an older document version and
SHALL allow a later valid revision to publish normally.

#### Scenario: Superseded diagnostics

- **WHEN** an edit fixes an error while analysis of the erroneous revision is still running
- **THEN** diagnostics from the erroneous revision are not published after diagnostics for the fixed revision

#### Scenario: Closing a document

- **WHEN** an open document with published diagnostics closes
- **THEN** the server clears its diagnostics and removes its synchronized overlay

#### Scenario: Open dependency changes

- **WHEN** an open imported module changes in a way that invalidates an importing document
- **THEN** the server republishes diagnostics for the affected importing document without requiring an edit to it

#### Scenario: Current analysis defects

- **WHEN** a current document revision cannot produce a snapshot because analysis defects
- **THEN** the server clears diagnostics from the older published version and does not present them as findings for the current text

#### Scenario: Diagnostics recover on the next edit

- **WHEN** a valid revision follows one whose analysis failed
- **THEN** the valid revision publishes its complete current diagnostics without requiring a language-server restart

### Requirement: Protocol handling remains responsive during analysis

Frontend computation for one project SHALL NOT prevent the server from accepting newer document
notifications, cancellation, shutdown, or requests for unrelated projects. The amount of obsolete
work executed after a newer revision arrives SHALL be bounded independently of the number of edits
already superseded.

#### Scenario: Edit arrives during expensive analysis

- **WHEN** frontend analysis is computing and a newer document revision arrives
- **THEN** the server accepts the revision without waiting for every obsolete analysis revision to complete

#### Scenario: Sustained typing does not build a revision backlog

- **WHEN** edits continue faster than frontend analysis can complete
- **THEN** queued work remains bounded to the newest required project state rather than growing once per edit

#### Scenario: Another project receives a request

- **WHEN** one project is performing expensive frontend analysis and a request targets another idle project
- **THEN** the server can answer the idle project's request without waiting for the expensive analysis

### Requirement: Language-server shutdown is bounded

Shutdown SHALL interrupt or retire every project worker, settle all pending project waiters, and
complete without depending on a worker that has already failed or stopped making progress. No
worker-state marker or pending analysis revision MAY keep shutdown open indefinitely.

#### Scenario: Shutdown after analysis defect

- **WHEN** shutdown begins after a project analysis worker terminated with a defect
- **THEN** the server settles that project's waiters and completes shutdown without waiting for a nonexistent worker

#### Scenario: Shutdown during active analysis

- **WHEN** shutdown begins while one or more projects are analyzing
- **THEN** active analysis is interrupted or retired and server lifecycle resources are released exactly once

### Requirement: Filesystem changes invalidate affected project analysis

The language server SHALL react to relevant changes to closed `.silk` files and `silk.toml`
manifests within an open document's project. It SHALL preserve open-buffer precedence and SHALL
reanalyze only projects that can be affected by the changed path.

#### Scenario: Closed dependency changes on disk

- **WHEN** a closed imported module changes on disk while an importing document remains open
- **THEN** the server refreshes the importing document against the new on-disk contents

#### Scenario: Open file also changes on disk

- **WHEN** a file with unsaved synchronized contents also changes on disk
- **THEN** analysis continues to use the synchronized contents until the document closes or the editor updates them

#### Scenario: Project manifest changes

- **WHEN** a relevant `silk.toml` change alters project or source-root discovery
- **THEN** the server rediscovers affected open documents before publishing subsequent analysis results

### Requirement: Editor analysis executes frontend phases only

The language server SHALL produce each ordinary synchronized-document result from an immutable
frontend analysis snapshot. Diagnostics, hover, completion, navigation, symbols, inlay hints, and
formatting SHALL NOT require instance discovery, target selection, layout planning, MIR lowering,
evaluation, or code generation. This computation choice SHALL NOT weaken version matching, atomic
project revision commit, recovery, or compiler ownership of semantic facts.

#### Scenario: Serve editor features from frontend analysis

- **WHEN** an accepted document revision completes analysis and receives hover, definition, completion, symbol, inlay-hint, formatting, and diagnostic requests
- **THEN** every request uses the committed frontend snapshot for that exact document version and no runtime realization phase executes

#### Scenario: Preserve coherent replacement

- **WHEN** a newer frontend snapshot atomically replaces the committed project revision
- **THEN** protocol requests observe its document text and semantic facts together under the existing latest-wins scheduling rules

### Requirement: One accepted project revision uses one shared analysis

For each accepted synchronized project revision, the language server SHALL invoke one project
frontend analysis over the complete captured open-document set and SHALL derive every analyzed
document result from that one immutable project revision. The amount of compiler frontend work
within a revision SHALL scale with the union module closure rather than with open roots multiplied
by their individual closures.

#### Scenario: Open documents share dependencies

- **WHEN** multiple synchronized documents in one project reach overlapping dependency closures
- **THEN** the accepted revision analyzes each shared module once and commits document results backed by the same project analysis

#### Scenario: Supersede shared project work

- **WHEN** a newer synchronized revision arrives while shared project analysis is queued or running
- **THEN** the older result cannot publish or replace the newer project revision under the existing latest-wins rules

### Requirement: Shared analysis preserves atomic document results

The language server SHALL commit the complete analyzed-document map only after the shared project
analysis and every root view are complete. A protocol request SHALL continue to observe document
bytes, line indexes, module identity, URI mappings, and semantic facts from one exact accepted
revision.

#### Scenario: Commit several root views

- **WHEN** shared analysis completes for several synchronized documents
- **THEN** all analyzed-document results become queryable together and no request can observe a partially replaced project map

### Requirement: Revision reuse begins from the last atomic commit

The project scheduler SHALL supply the last complete committed analyzed-document map to a new
project analysis callback. The first analysis SHALL receive an empty prior map. A superseded,
failed, interrupted, or still-running analysis MUST NOT become the reuse basis for later work.

#### Scenario: Analyze after one accepted edit

- **WHEN** revision N commits and revision N+1 begins
- **THEN** the N+1 analysis callback receives revision N's complete committed map as its prior value

#### Scenario: Supersede an active revision

- **WHEN** revision N+1 becomes stale while N+2 is queued
- **THEN** N+2 receives the last committed map from revision N rather than any result produced for N+1

### Requirement: Shared workspace results retain their project analysis

Every analyzed-document view produced by one workspace analysis SHALL reference the same completed
project analysis value. Workspace revision analysis SHALL derive its syntax reuse only from the
prior committed value for that workspace.

#### Scenario: Commit several revised root views

- **WHEN** a multi-root workspace revision completes using a prior committed project
- **THEN** every document result references the same new project analysis and its own current root view
