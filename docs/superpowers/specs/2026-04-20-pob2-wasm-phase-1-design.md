# PoB2 WASM Phase 1 Design

Date: 2026-04-20

## Goal

Build a new web-oriented calculation engine layer for Path of Building 2 by reusing the existing Lua calculation system inside WASM.

Phase 1 optimizes for feature parity and browser compatibility, not minimal binary size. A few seconds of blocking startup on the web is acceptable. The output contract must provide exact numeric and string parity for values shown by the current PoB2 UI, while rich UI formatting and presentation logic can remain outside the binary.

## Decisions

### Chosen Architecture

- Phase 1 uses option A: Lua-in-WASM with a thin headless adapter.
- Phase 1 does not attempt a Rust rewrite of the calc engine.
- Phase 1 does not use Rust as the primary host unless a wrapper becomes necessary later for packaging or integration.
- A future option B remains explicitly planned: a Rust host around the engine contract after phase-1 parity is verified.

### Output Parity Level

- The engine must expose every numeric or string value the current PoB2 UI shows.
- The engine does not need to replicate all UI presentation formatting internally.
- Structured breakdown data should be exposed where available, but breakdown rendering remains a UI concern.

### Game Data Strategy

- Phase 1 request payloads use a `gameData` reference rather than a fully injected external data bundle.
- The request schema is intentionally future-compatible with later externally managed versioned data bundles.
- A future pass can move toward split downloads: WASM runtime plus separately cached versioned game-data bundles derived from a custom extraction pipeline.
- That future data-bundle work is explicitly out of scope for phase 1.

## Existing Codebase Findings

### Relevant Reused Entry Points

- `PathOfBuilding-PoE2/src/HeadlessWrapper.lua` already proves the app can run without the native UI host.
- `PathOfBuilding-PoE2/src/Modules/Build.lua` owns build XML loading and section reconstruction.
- `PathOfBuilding-PoE2/src/Modules/Calcs.lua` is the main calculation orchestration layer.
- `PathOfBuilding-PoE2/src/Modules/CalcSetup.lua`, `CalcPerform.lua`, `CalcOffence.lua`, and `CalcDefence.lua` contain the core calculation pipeline.
- `PathOfBuilding-PoE2/src/Classes/CalcsTab.lua` shows the current two-pass output model: `MAIN` output and `CALCS` output.

### Existing Output Surfaces

- `env.player.output`, `env.minion.output`, and `env.enemy.output` are the closest existing structured runtime result objects.
- `PathOfBuilding-PoE2/src/Modules/BuildDisplayStats.lua` defines sidebar and comparison-visible display stats.
- `PathOfBuilding-PoE2/src/Modules/CalcSections.lua` defines the Calcs tab sections and output references.
- `PathOfBuilding-PoE2/src/Modules/Calcs.lua` also builds several imperative text lists such as buffs, debuffs, combat, and curse summaries.

### Main Extraction Constraint

The hard part is not reimplementing the math. The hard part is that the calc pipeline expects a full mutable `build` object whose shape today is owned by UI-era tabs such as `configTab`, `itemsTab`, `skillsTab`, `treeTab`, and `calcsTab`. Phase 1 therefore focuses on introducing a headless adapter that satisfies that contract with the minimum required surface area.

## Phase 1 Architecture

### Runtime Boundary

The browser UI communicates only with a worker-facing calculation API.

Inside the worker:

1. A browser-safe Lua runtime compiled to WASM boots the reused PoB2 Lua engine.
2. A new headless adapter loads build XML and game-data references.
3. The adapter constructs the minimum `build` and `*Tab` state required by the existing calc pipeline.
4. The adapter executes the equivalent of the current headless import plus calc flow.
5. The adapter normalizes the results into a stable response contract for the new UI.

### Components

#### 1. Web UI Shell

Responsibilities:

- submit calculation requests
- own user-facing loading/spinner state
- render normalized outputs
- never depend on PoB-internal Lua object shapes

#### 2. Worker Host

Responsibilities:

- isolate engine crashes from the UI thread
- load the WASM runtime and engine assets
- serialize request and response messages
- report structured initialization and runtime errors

#### 3. Headless Adapter

Responsibilities:

- resolve `gameData` references
- ingest build XML
- apply optional config overrides
- create the minimum PoB-compatible runtime state needed by calculations
- invoke the calculation pipeline
- normalize outputs and warnings into a stable API shape

This is the key seam for both phase 1 and future option B.

#### 4. Reused PoB2 Lua Engine

Responsibilities:

- import build state using the existing PoB model
- compute all player, minion, and enemy outputs
- preserve current calculation semantics

Phase 1 should reuse this engine as directly as possible and avoid semantic rewrites.

## API Contract

### Request

```ts
type CalcRequest = {
  buildXml: string
  configOverrides?: Record<string, unknown>
  gameData: {
    kind: "version-ref" | "bundle-ref"
    id: string
    manifestHash?: string
  }
}
```

Request notes:

- `buildXml` is the canonical input because the current PoB loader already understands it.
- `configOverrides` allow a web UI to adjust runtime settings without mutating the base XML.
- `gameData.kind = "version-ref"` is the normal phase-1 path.
- `gameData.kind = "bundle-ref"` is reserved for future chunked and externally managed data bundles.

### Response

```ts
type Scalar = number | string | boolean | null

type CalcResponse = {
  metadata: {
    engineVersion: string
    gameDataId: string
    warnings: string[]
  }
  outputs: {
    player: Record<string, Scalar>
    minion?: Record<string, Scalar>
    enemy?: Record<string, Scalar>
  }
  derived: {
    displayStats: Array<{ key: string; label: string; value: Scalar }>
    minionDisplayStats?: Array<{ key: string; label: string; value: Scalar }>
    skillDps?: Array<{ name: string; dps: number; count?: number; skillPart?: string }>
    textLists?: {
      buffs?: string[]
      debuffs?: string[]
      combat?: string[]
      curses?: string[]
    }
  }
  breakdowns?: Record<string, unknown>
}
```

Response notes:

- `outputs.player` is backed primarily by `env.player.output`.
- `derived.displayStats` is the UI-parity export layer derived from `BuildDisplayStats.lua`.
- `textLists` capture imperative lists currently built in `Calcs.lua`.
- `breakdowns` remain structured and UI-agnostic.

## Output Parity Strategy

Phase 1 must make output coverage explicit and testable.

### Export Manifest

A checked-in export manifest should be generated from:

- `PathOfBuilding-PoE2/src/Modules/BuildDisplayStats.lua`
- `PathOfBuilding-PoE2/src/Modules/CalcSections.lua`
- imperative text-list builders in `PathOfBuilding-PoE2/src/Modules/Calcs.lua`

The manifest becomes the source of truth for:

- required exported keys
- expected display-stat coverage
- smoke-test completeness
- future Rust-host follow-up work

### Normalization Layer

The adapter should not expose raw PoB tables blindly. It should normalize outputs into:

- stable top-level output maps
- stable derived display rows
- stable text-list groups
- stable breakdown containers

This gives the new UI a consistent contract and prevents direct coupling to PoB-internal data layout changes.

## Data Strategy

### Phase 1

Phase 1 treats game data as a resolved internal asset set selected by `gameData.id` using the phase-1 `version-ref` path.

This avoids taking on a second extraction project while the engine is being moved into WASM.

### Future Direction

After phase-1 parity is verified, the preferred evolution is:

- keep the engine/runtime separately cacheable
- move game data into versioned external bundles
- fetch only the chosen version's data
- eventually support independently produced GGPK-derived data bundles

The API contract above is designed to allow that evolution without breaking callers.

## Verification And CI

### Fixture Source

Verified build XML fixtures already exist at:

- `/Users/andreashoffmann1/projects/pob-wasm2/crates/calc-parity/builds`

Current fixture files observed there:

- `0_4_acolyte_chayula_ci_aegis.xml`
- `0_4_blood_mage_spark.xml`
- `0_4_chronomancer_ignite.xml`
- `0_4_deadeye_bow_attack.xml`
- `0_4_infernalist_minions.xml`
- `0_4_invoker_hybrid.xml`
- `0_4_pathfinder_flask_stack.xml`
- `0_4_stormweaver_trigger_cascade.xml`
- `0_4_titan_melee_strike.xml`
- `0_4_warbringer_warcry.xml`

These fixtures are already verified and use the proper main skill. They should be copied into this project as the initial parity corpus.

### Verification Model

1. Run the existing PoB2 headless path as the reference implementation.
2. Normalize the reference results using the same export schema.
3. Run the WASM engine on the same fixtures.
4. Compare the normalized results.

Comparison rules:

- exact match for strings and booleans
- exact match for integer-like numeric fields
- tolerance-based match only for floating-point fields, with the tolerance defined centrally in the parity harness rather than per test
- zero missing manifest-required outputs

### CI Requirements

Phase 1 CI should at minimum perform:

- WASM build
- parity smoke test against a small representative fixture subset
- one browser-oriented worker load and invoke smoke test
- artifact size reporting

Artifact size reporting is required so size stays visible, but it is not the primary success gate unless the build becomes obviously unsuitable for web delivery.

## Error Handling

- malformed XML is a hard error
- invalid or missing game-data references are hard errors
- engine crashes must be isolated to the worker boundary
- unsupported or degraded behavior must surface as structured warnings
- manifest-required outputs must not be silently omitted

## Acceptance Criteria Mapping

### Project Is Scaffolded

The repository should contain:

- a WASM engine package
- a worker-facing runtime wrapper
- a parity harness
- copied fixture XMLs
- initial export-manifest generation

### CI Is Setup With Build And Smoke Test

CI must build the engine and run at least one parity smoke test from day 1.

### WASM Binary With In: Build + Config + Game-Data, Out: Calculation Values

The request and response contract above defines the exact phase-1 binary boundary.

### Binary Has Manageable Size For Web

For phase 1, manageable means acceptable with a few seconds of blocking startup. It does not mean aggressively minimized.

## Out Of Scope For Phase 1

- Rust reimplementation of the calc engine
- a fully externalized GGPK-derived data contract
- perfect UI formatting parity inside the binary
- aggressive binary-size optimization before parity is proven
- broad cleanup of unrelated PoB2 architecture issues

## Rust Host Follow-up

Future option B should be tracked explicitly after phase-1 validation.

### Intended Rust-Owned Responsibilities Later

- own the public WASM API boundary
- own packaging and asset resolution
- own runtime orchestration and worker integration
- potentially own parts of normalization and schema validation

### Current Lua-Coupled Assumptions That Block B Today

- calc modules expect the mutable PoB `build` graph and tab-owned state
- import and data loading are built around PoB-native loaders and tables
- output production is spread across several modules and some imperative UI-era helpers

### B Compatibility Rules Established In Phase 1

- the request and response contract must stay UI-agnostic
- the normalization layer must not leak Lua-internal table shapes
- the export manifest must remain independent of runtime implementation language
- parity fixtures and comparison harness must be reusable by a future Rust host

## Recommended Next Step

After user review of this spec, the next step is to write an implementation plan that breaks phase 1 into concrete execution steps: scaffolding, runtime integration, adapter extraction, export manifest generation, fixture import, and CI parity verification.
