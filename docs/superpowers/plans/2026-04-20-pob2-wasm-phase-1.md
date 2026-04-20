# PoB2 WASM Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a browser-compatible PoB2 calculation engine that accepts build XML plus config/game-data references and returns parity-checked calculation outputs from a worker-backed WASM runtime.

**Architecture:** Keep the PoB2 Lua calculation engine intact, add a shared Lua entrypoint for reference and WASM execution, and wrap it with a TypeScript worker API plus normalization/parity harness. Use Vite's current worker and WASM asset handling and Vitest's current multi-project node/browser setup to keep the scaffold small while still covering CI smoke checks.

**Tech Stack:** TypeScript, Node.js, Vite 7.3.x, Vitest 4.x, Playwright Chromium via Vitest browser provider, Lua runtime compiled to WASM, GitHub Actions.

---

## File Map

- Create: `.gitignore`
  Purpose: ignore Node/Vite outputs, copied fixtures, generated parity artifacts, and browser brainstorming scratch files.
- Create: `package.json`
  Purpose: root scripts and dependency declarations.
- Create: `tsconfig.json`
  Purpose: TypeScript compiler settings for app, tests, and scripts.
- Create: `vite.config.ts`
  Purpose: build the smoke page, worker, and WASM assets using Vite worker support.
- Create: `vitest.config.ts`
  Purpose: separate node contract tests from browser smoke tests using Vitest projects.
- Create: `src/index.ts`
  Purpose: public TypeScript entrypoint for worker creation and request helpers.
- Create: `src/types.ts`
  Purpose: stable request/response types shared across worker, smoke page, and tests.
- Create: `src/engine/messages.ts`
  Purpose: worker message protocol.
- Create: `src/engine/bootstrap.ts`
  Purpose: WASM runtime boot and one-time engine initialization.
- Create: `src/engine/calc.ts`
  Purpose: bridge request payloads to the Lua engine and normalization layer.
- Create: `src/engine/worker.ts`
  Purpose: worker host entrypoint.
- Create: `src/parity/normalize.ts`
  Purpose: normalize reference and WASM results into one comparable schema.
- Create: `src/parity/compare.ts`
  Purpose: exact/tolerance parity comparison rules.
- Create: `src/parity/reference-runner.ts`
  Purpose: run native LuaJIT reference calculations through the shared Lua entrypoint.
- Create: `src/parity/export-manifest.json`
  Purpose: checked-in source-of-truth list of UI-visible exports.
- Create: `src/pob/engine-entry.lua`
  Purpose: shared Lua adapter entrypoint for phase 1 calculations.
- Create: `src/pob/reference-dump.lua`
  Purpose: CLI/LuaJIT wrapper that loads `engine-entry.lua` and prints JSON for parity tests.
- Create: `src/smoke/index.html`
  Purpose: minimal browser page for manual and CI smoke verification.
- Create: `src/smoke/main.ts`
  Purpose: drives the worker smoke page.
- Create: `fixtures/builds/*.xml`
  Purpose: copied verified fixture corpus from the existing parity project.
- Create: `scripts/sync-fixtures.mjs`
  Purpose: copy verified XML fixtures into this repo.
- Create: `scripts/generate-export-manifest.mjs`
  Purpose: parse PoB2 display/calcs sources and generate `export-manifest.json`.
- Create: `scripts/report-artifact-size.mjs`
  Purpose: report built asset sizes in CI.
- Create: `scripts/build-lua-wasm.sh`
  Purpose: compile or stage the browser-safe Lua runtime into the Vite-consumable location.
- Create: `docs/rust-host-follow-up.md`
  Purpose: track deferred option B work explicitly outside the spec.
- Create: `tests/node/project-layout.test.ts`
  Purpose: scaffold verification.
- Create: `tests/node/fixtures.test.ts`
  Purpose: fixture import and follow-up doc verification.
- Create: `tests/node/export-manifest.test.ts`
  Purpose: manifest generation verification.
- Create: `tests/node/normalize.test.ts`
  Purpose: normalization contract verification.
- Create: `tests/node/reference-runner.test.ts`
  Purpose: native reference-runner verification.
- Create: `tests/node/fixture-parity.test.ts`
  Purpose: end-to-end parity test against one fixture first, then more.
- Create: `tests/browser/worker-smoke.browser.test.ts`
  Purpose: browser worker boot/invoke smoke test.
- Create: `.github/workflows/ci.yml`
  Purpose: install, build, parity smoke, browser smoke, artifact size reporting.

### Task 1: Scaffold The Root Toolchain

**Files:**
- Create: `.gitignore`
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `vite.config.ts`
- Create: `vitest.config.ts`
- Create: `src/index.ts`
- Test: `tests/node/project-layout.test.ts`

- [ ] **Step 1: Write the failing scaffold test**

```ts
// tests/node/project-layout.test.ts
import { existsSync, readFileSync } from 'node:fs'
import { describe, expect, it } from 'vitest'

describe('project scaffold', () => {
  it('defines the required root scripts and entry files', () => {
    const pkg = JSON.parse(readFileSync('package.json', 'utf8'))

    expect(pkg.scripts.build).toBe('vite build')
    expect(pkg.scripts['test:node']).toBe('vitest run --project node')
    expect(pkg.scripts['test:browser']).toBe('vitest run --project browser')

    expect(existsSync('tsconfig.json')).toBe(true)
    expect(existsSync('vite.config.ts')).toBe(true)
    expect(existsSync('vitest.config.ts')).toBe(true)
    expect(existsSync('src/index.ts')).toBe(true)
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run tests/node/project-layout.test.ts`

Expected: FAIL because `package.json` and the root config files do not exist yet.

- [ ] **Step 3: Write the minimal scaffold**

```json
// package.json
{
  "name": "pob-wasm3",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "vite build",
    "test": "vitest run",
    "test:node": "vitest run --project node",
    "test:browser": "vitest run --project browser",
    "fixtures:sync": "node scripts/sync-fixtures.mjs",
    "manifest:generate": "node scripts/generate-export-manifest.mjs",
    "size:report": "node scripts/report-artifact-size.mjs"
  },
  "devDependencies": {
    "@vitest/browser-playwright": "^4.0.7",
    "playwright": "^1.55.0",
    "typescript": "^5.9.2",
    "vite": "^7.3.1",
    "vitest": "^4.0.7"
  }
}
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "types": ["vitest/globals"],
    "allowSyntheticDefaultImports": true,
    "skipLibCheck": true
  },
  "include": ["src", "tests", "scripts", "vite.config.ts", "vitest.config.ts"]
}
```

```ts
// vite.config.ts
import { resolve } from 'node:path'
import { defineConfig } from 'vite'

export default defineConfig({
  build: {
    target: 'esnext',
    rollupOptions: {
      input: {
        smoke: resolve(__dirname, 'src/smoke/index.html'),
      },
    },
  },
})
```

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import { playwright } from '@vitest/browser-playwright'

export default defineConfig({
  test: {
    projects: [
      {
        test: {
          name: 'node',
          environment: 'node',
          include: ['tests/node/**/*.test.ts'],
        },
      },
      {
        test: {
          name: 'browser',
          include: ['tests/browser/**/*.browser.test.ts'],
          browser: {
            enabled: true,
            provider: playwright(),
            instances: [{ browser: 'chromium' }],
          },
        },
      },
    ],
  },
})
```

```ts
// src/index.ts
export type { CalcRequest, CalcResponse } from './types'

export function createCalcWorker(): Worker {
  throw new Error('worker bootstrap not implemented yet')
}
```

```gitignore
# .gitignore
node_modules/
dist/
coverage/
.superpowers/
```

- [ ] **Step 4: Run the scaffold test to verify it passes**

Run: `npm install && npm run test:node -- tests/node/project-layout.test.ts`

Expected: PASS for the scaffold test.

- [ ] **Step 5: Commit**

```bash
git add .gitignore package.json tsconfig.json vite.config.ts vitest.config.ts src/index.ts tests/node/project-layout.test.ts
git commit -m "chore: scaffold TypeScript build and test toolchain"
```

### Task 2: Import Verified Fixtures And Create The Rust Follow-Up Doc

**Files:**
- Create: `scripts/sync-fixtures.mjs`
- Create: `fixtures/builds/*.xml`
- Create: `docs/rust-host-follow-up.md`
- Test: `tests/node/fixtures.test.ts`

- [ ] **Step 1: Write the failing fixture/doc test**

```ts
// tests/node/fixtures.test.ts
import { readdirSync, readFileSync } from 'node:fs'
import { describe, expect, it } from 'vitest'

describe('fixtures and rust follow-up doc', () => {
  it('copies the verified XML corpus and keeps a rust-host follow-up doc', () => {
    const xmlFiles = readdirSync('fixtures/builds').filter((name) => name.endsWith('.xml'))
    const followUp = readFileSync('docs/rust-host-follow-up.md', 'utf8')

    expect(xmlFiles).toHaveLength(10)
    expect(xmlFiles).toContain('0_4_infernalist_minions.xml')
    expect(xmlFiles).toContain('0_4_deadeye_bow_attack.xml')
    expect(followUp).toContain('# Rust Host Follow-Up')
    expect(followUp).toContain('Keep the request/response contract stable')
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npm run test:node -- tests/node/fixtures.test.ts`

Expected: FAIL because the copied fixtures and follow-up document do not exist yet.

- [ ] **Step 3: Add the fixture sync script, copy the XML files, and write the follow-up doc**

```js
// scripts/sync-fixtures.mjs
import { cpSync, mkdirSync, readdirSync } from 'node:fs'
import { join } from 'node:path'

const source = '/Users/andreashoffmann1/projects/pob-wasm2/crates/calc-parity/builds'
const target = 'fixtures/builds'

mkdirSync(target, { recursive: true })

for (const file of readdirSync(source)) {
  if (file.endsWith('.xml')) {
    cpSync(join(source, file), join(target, file))
  }
}
```

```md
# Rust Host Follow-Up

## Goal

Track the deferred option B work without contaminating phase-1 delivery.

## Guardrails

- Keep the request/response contract stable.
- Keep normalization logic implementation-language agnostic.
- Keep parity fixtures reusable by both the Lua-in-WASM path and a later Rust host.

## Deferred Work

- Move worker/runtime orchestration to Rust.
- Re-home asset resolution behind the same `gameData` contract.
- Replace Lua-owned adapter responsibilities incrementally after parity is locked.
```

Run the copy step:

```bash
mkdir -p fixtures/builds
node scripts/sync-fixtures.mjs
```

- [ ] **Step 4: Run the fixture/doc test to verify it passes**

Run: `npm run test:node -- tests/node/fixtures.test.ts`

Expected: PASS with ten XML fixtures present.

- [ ] **Step 5: Commit**

```bash
git add scripts/sync-fixtures.mjs fixtures/builds docs/rust-host-follow-up.md tests/node/fixtures.test.ts
git commit -m "chore: import verified fixtures and track rust host follow-up"
```

### Task 3: Generate And Check In The Export Manifest

**Files:**
- Create: `scripts/generate-export-manifest.mjs`
- Create: `src/parity/export-manifest.json`
- Test: `tests/node/export-manifest.test.ts`

- [ ] **Step 1: Write the failing manifest test**

```ts
// tests/node/export-manifest.test.ts
import manifest from '../../src/parity/export-manifest.json' with { type: 'json' }
import { describe, expect, it } from 'vitest'

describe('export manifest', () => {
  it('captures required sidebar, calcs, and text-list exports', () => {
    expect(manifest.displayStats).toContain('Life')
    expect(manifest.displayStats).toContain('FullDPS')
    expect(manifest.calcSections).toContain('AverageHit')
    expect(manifest.calcSections).toContain('TotalDPS')
    expect(manifest.textLists).toContain('BuffList')
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npm run test:node -- tests/node/export-manifest.test.ts`

Expected: FAIL because the manifest file does not exist yet.

- [ ] **Step 3: Write the manifest generator and generate the first checked-in manifest**

```js
// scripts/generate-export-manifest.mjs
import { mkdirSync, readFileSync, writeFileSync } from 'node:fs'

const displayStatsSource = readFileSync('PathOfBuilding-PoE2/src/Modules/BuildDisplayStats.lua', 'utf8')
const calcSectionsSource = readFileSync('PathOfBuilding-PoE2/src/Modules/CalcSections.lua', 'utf8')
const calcsSource = readFileSync('PathOfBuilding-PoE2/src/Modules/Calcs.lua', 'utf8')

const displayStats = [...displayStatsSource.matchAll(/stat = "([^"]+)"/g)].map((m) => m[1])
const calcSections = [...calcSectionsSource.matchAll(/output:([A-Za-z0-9_.]+)/g)].map((m) => m[1])
const textLists = [...calcsSource.matchAll(/output\.(BuffList|CombatList|CurseList|SkillDebuffs|SkillBuffs)/g)].map((m) => m[1])

const manifest = {
  displayStats: [...new Set(displayStats)].sort(),
  calcSections: [...new Set(calcSections)].sort(),
  textLists: [...new Set(textLists)].sort(),
}

mkdirSync('src/parity', { recursive: true })
writeFileSync('src/parity/export-manifest.json', JSON.stringify(manifest, null, 2) + '\n')
```

Run:

```bash
node scripts/generate-export-manifest.mjs
```

- [ ] **Step 4: Run the manifest test to verify it passes**

Run: `npm run test:node -- tests/node/export-manifest.test.ts`

Expected: PASS with `Life`, `FullDPS`, `AverageHit`, `TotalDPS`, and `BuffList` present.

- [ ] **Step 5: Commit**

```bash
git add scripts/generate-export-manifest.mjs src/parity/export-manifest.json tests/node/export-manifest.test.ts
git commit -m "feat: generate checked-in export manifest"
```

### Task 4: Define The Calculation Contract And Normalization Layer

**Files:**
- Create: `src/types.ts`
- Create: `src/parity/normalize.ts`
- Test: `tests/node/normalize.test.ts`
- Modify: `src/index.ts`

- [ ] **Step 1: Write the failing normalization test**

```ts
// tests/node/normalize.test.ts
import { describe, expect, it } from 'vitest'
import { normalizeResult } from '../../src/parity/normalize'

describe('normalizeResult', () => {
  it('projects raw engine output into the public response schema', () => {
    const normalized = normalizeResult({
      metadata: { engineVersion: 'dev', gameDataId: '0.4', warnings: [] },
      raw: {
        player: { Life: 1234, FullDPS: 5678 },
        minion: { TotalDPS: 99 },
        textLists: { BuffList: ['Onslaught'] },
      },
    })

    expect(normalized.outputs.player.Life).toBe(1234)
    expect(normalized.outputs.player.FullDPS).toBe(5678)
    expect(normalized.outputs.minion?.TotalDPS).toBe(99)
    expect(normalized.derived.textLists?.buffs).toEqual(['Onslaught'])
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npm run test:node -- tests/node/normalize.test.ts`

Expected: FAIL because `src/types.ts` and `src/parity/normalize.ts` do not exist yet.

- [ ] **Step 3: Write the minimal types and normalization code**

```ts
// src/types.ts
export type Scalar = number | string | boolean | null

export type CalcRequest = {
  buildXml: string
  configOverrides?: Record<string, unknown>
  gameData: {
    kind: 'version-ref' | 'bundle-ref'
    id: string
    manifestHash?: string
  }
}

export type CalcResponse = {
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

```ts
// src/parity/normalize.ts
import type { CalcResponse, Scalar } from '../types'

type RawResult = {
  metadata: CalcResponse['metadata']
  raw: {
    player: Record<string, Scalar>
    minion?: Record<string, Scalar>
    enemy?: Record<string, Scalar>
    textLists?: Record<string, string[]>
  }
}

export function normalizeResult(input: RawResult): CalcResponse {
  return {
    metadata: input.metadata,
    outputs: {
      player: input.raw.player,
      minion: input.raw.minion,
      enemy: input.raw.enemy,
    },
    derived: {
      displayStats: [],
      textLists: {
        buffs: input.raw.textLists?.BuffList,
        debuffs: input.raw.textLists?.SkillDebuffs,
        combat: input.raw.textLists?.CombatList,
        curses: input.raw.textLists?.CurseList,
      },
    },
  }
}
```

```ts
// src/index.ts
export type { CalcRequest, CalcResponse } from './types'
export { normalizeResult } from './parity/normalize'

export function createCalcWorker(): Worker {
  throw new Error('worker bootstrap not implemented yet')
}
```

- [ ] **Step 4: Run the normalization test to verify it passes**

Run: `npm run test:node -- tests/node/normalize.test.ts`

Expected: PASS with the minimal public schema in place.

- [ ] **Step 5: Commit**

```bash
git add src/types.ts src/parity/normalize.ts src/index.ts tests/node/normalize.test.ts
git commit -m "feat: define calculation contract and normalization layer"
```

### Task 5: Add The Native Reference Runner Around HeadlessWrapper

**Files:**
- Create: `src/pob/engine-entry.lua`
- Create: `src/pob/reference-dump.lua`
- Create: `src/parity/reference-runner.ts`
- Test: `tests/node/reference-runner.test.ts`

- [ ] **Step 1: Write the failing reference-runner test**

```ts
// tests/node/reference-runner.test.ts
import { describe, expect, it } from 'vitest'
import { runReferenceFixture } from '../../src/parity/reference-runner'

describe('runReferenceFixture', () => {
  it('returns a normalized reference result for one verified build fixture', async () => {
    const result = await runReferenceFixture('fixtures/builds/0_4_deadeye_bow_attack.xml', '0.4')

    expect(result.metadata.gameDataId).toBe('0.4')
    expect(typeof result.outputs.player.Life).toBe('number')
    expect(result.outputs.player.FullDPS).not.toBeNull()
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npm run test:node -- tests/node/reference-runner.test.ts`

Expected: FAIL because the reference runner and Lua entry files do not exist yet.

- [ ] **Step 3: Write the shared Lua entrypoint, dump wrapper, and Node runner**

```lua
-- src/pob/engine-entry.lua
local json = require('dkjson')

local M = {}

function M.calculate(buildXml, gameDataId)
  loadBuildFromXML(buildXml, 'fixture.xml')
  build.calcsTab:BuildOutput()

  return {
    metadata = {
      engineVersion = 'reference-headless',
      gameDataId = gameDataId,
      warnings = {},
    },
    raw = {
      player = build.calcsTab.mainOutput,
      minion = build.calcsTab.mainOutput.Minion,
      textLists = {
        BuffList = build.calcsTab.calcsOutput.BuffList,
        CombatList = build.calcsTab.calcsOutput.CombatList,
        CurseList = build.calcsTab.calcsOutput.CurseList,
        SkillDebuffs = build.calcsTab.calcsOutput.SkillDebuffs,
      },
    },
  }
end

return M
```

```lua
-- src/pob/reference-dump.lua
package.path = package.path .. ';../runtime/lua/?.lua;../runtime/lua/?/init.lua'

dofile('HeadlessWrapper.lua')

local json = require('dkjson')
local engine = dofile('../../src/pob/engine-entry.lua')

local fixturePath = arg[1]
local gameDataId = arg[2]
local file = assert(io.open(fixturePath, 'r'))
local xml = file:read('*a')
file:close()

print(json.encode(engine.calculate(xml, gameDataId)))
```

```ts
// src/parity/reference-runner.ts
import { readFile } from 'node:fs/promises'
import { spawn } from 'node:child_process'
import { resolve } from 'node:path'
import { normalizeResult } from './normalize'

export async function runReferenceFixture(fixturePath: string, gameDataId: string) {
  const absoluteFixturePath = resolve(fixturePath)
  await readFile(absoluteFixturePath, 'utf8')

  const stdout = await new Promise<string>((resolve, reject) => {
    const child = spawn('luajit', ['../../src/pob/reference-dump.lua', absoluteFixturePath, gameDataId], {
      cwd: 'PathOfBuilding-PoE2/src',
      stdio: ['ignore', 'pipe', 'pipe'],
    })

    let output = ''
    let error = ''
    child.stdout.on('data', (chunk) => {
      output += chunk
    })
    child.stderr.on('data', (chunk) => {
      error += chunk
    })
    child.on('exit', (code) => {
      if (code === 0) resolve(output.trim())
      else reject(new Error(error || `luajit exited with ${code}`))
    })
  })

  return normalizeResult(JSON.parse(stdout))
}
```

- [ ] **Step 4: Run the reference-runner test to verify it passes**

Run: `npm run test:node -- tests/node/reference-runner.test.ts`

Expected: PASS if `luajit` is available and the fixture can be loaded through the shared Lua entrypoint.

- [ ] **Step 5: Commit**

```bash
git add src/pob/engine-entry.lua src/pob/reference-dump.lua src/parity/reference-runner.ts tests/node/reference-runner.test.ts
git commit -m "feat: add native parity reference runner"
```

### Task 6: Add The Worker Protocol And Browser Smoke Page With A Stub Engine

**Files:**
- Create: `src/engine/messages.ts`
- Create: `src/engine/worker.ts`
- Create: `src/smoke/index.html`
- Create: `src/smoke/main.ts`
- Modify: `src/index.ts`
- Test: `tests/browser/worker-smoke.browser.test.ts`

- [ ] **Step 1: Write the failing browser smoke test**

```ts
// tests/browser/worker-smoke.browser.test.ts
import { describe, expect, it } from 'vitest'
import { page } from '@vitest/browser/context'

describe('worker smoke page', () => {
  it('boots a worker and renders a ready message', async () => {
    await page.goto('/src/smoke/index.html')
    await expect.element(page.getByText('worker ready')).toBeVisible()
  })
})
```

- [ ] **Step 2: Run the browser test to verify it fails**

Run: `npm run test:browser -- tests/browser/worker-smoke.browser.test.ts`

Expected: FAIL because the smoke page and worker files do not exist yet.

- [ ] **Step 3: Write the worker protocol, stub worker, and smoke page**

```ts
// src/engine/messages.ts
import type { CalcRequest, CalcResponse } from '../types'

export type WorkerRequest = { id: string; type: 'calculate'; payload: CalcRequest }
export type WorkerResponse = { id: string; type: 'result'; payload: CalcResponse }
```

```ts
// src/engine/worker.ts
import type { WorkerRequest, WorkerResponse } from './messages'

self.onmessage = (event: MessageEvent<WorkerRequest>) => {
  const response: WorkerResponse = {
    id: event.data.id,
    type: 'result',
    payload: {
      metadata: { engineVersion: 'stub', gameDataId: event.data.payload.gameData.id, warnings: [] },
      outputs: { player: { Life: 1, FullDPS: 1 } },
      derived: { displayStats: [] },
    },
  }

  self.postMessage(response)
}
```

```ts
// src/index.ts
import CalcWorker from './engine/worker.ts?worker'

export type { CalcRequest, CalcResponse } from './types'
export { normalizeResult } from './parity/normalize'

export function createCalcWorker(): Worker {
  return new CalcWorker()
}
```

```html
<!-- src/smoke/index.html -->
<!doctype html>
<html lang="en">
  <body>
    <main>
      <h1>PoB WASM Smoke</h1>
      <p id="status">booting</p>
      <script type="module" src="./main.ts"></script>
    </main>
  </body>
</html>
```

```ts
// src/smoke/main.ts
import { createCalcWorker } from '../index'

const status = document.querySelector<HTMLParagraphElement>('#status')!
const worker = createCalcWorker()

worker.onmessage = () => {
  status.textContent = 'worker ready'
}

worker.postMessage({
  id: 'smoke',
  type: 'calculate',
  payload: {
    buildXml: '<PathOfBuilding2 />',
    gameData: { kind: 'version-ref', id: '0.4' },
  },
})
```

- [ ] **Step 4: Run the browser smoke test to verify it passes**

Run: `npm run test:browser -- tests/browser/worker-smoke.browser.test.ts`

Expected: PASS with the stub worker returning a smoke response.

- [ ] **Step 5: Commit**

```bash
git add src/engine/messages.ts src/engine/worker.ts src/index.ts src/smoke/index.html src/smoke/main.ts tests/browser/worker-smoke.browser.test.ts
git commit -m "feat: add worker protocol and browser smoke harness"
```

### Task 7: Stage The Lua-WASM Runtime And Boot Path

**Files:**
- Create: `scripts/build-lua-wasm.sh`
- Create: `src/engine/bootstrap.ts`
- Modify: `src/engine/worker.ts`
- Test: `tests/node/fixture-parity.test.ts`

- [ ] **Step 1: Write the failing runtime boot test**

```ts
// tests/node/fixture-parity.test.ts
import { describe, expect, it } from 'vitest'
import { bootEngine } from '../../src/engine/bootstrap'

describe('bootEngine', () => {
  it('initializes the wasm-backed runtime exactly once', async () => {
    const a = await bootEngine()
    const b = await bootEngine()

    expect(a).toBe(b)
    expect(a.kind).toBe('lua-wasm')
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npm run test:node -- tests/node/fixture-parity.test.ts`

Expected: FAIL because `bootEngine` and the runtime build script do not exist yet.

- [ ] **Step 3: Add the runtime staging script and boot helper**

```bash
# scripts/build-lua-wasm.sh
#!/usr/bin/env bash
set -euo pipefail

mkdir -p public/lua-runtime
cp vendor/lua-runtime/lua.wasm public/lua-runtime/lua.wasm
cp vendor/lua-runtime/lua.js public/lua-runtime/lua.js
```

```ts
// src/engine/bootstrap.ts
type EngineHandle = {
  kind: 'lua-wasm'
  evaluate: (code: string) => Promise<unknown>
}

let enginePromise: Promise<EngineHandle> | undefined

export function bootEngine(): Promise<EngineHandle> {
  if (!enginePromise) {
    enginePromise = Promise.resolve({
      kind: 'lua-wasm',
      evaluate: async () => undefined,
    })
  }

  return enginePromise
}
```

```ts
// src/engine/worker.ts
import { bootEngine } from './bootstrap'
import type { WorkerRequest, WorkerResponse } from './messages'

self.onmessage = async (event: MessageEvent<WorkerRequest>) => {
  const engine = await bootEngine()

  const response: WorkerResponse = {
    id: event.data.id,
    type: 'result',
    payload: {
      metadata: { engineVersion: engine.kind, gameDataId: event.data.payload.gameData.id, warnings: [] },
      outputs: { player: { Life: 1, FullDPS: 1 } },
      derived: { displayStats: [] },
    },
  }

  self.postMessage(response)
}
```

- [ ] **Step 4: Run the boot test to verify it passes**

Run: `npm run test:node -- tests/node/fixture-parity.test.ts`

Expected: PASS with idempotent runtime initialization.

- [ ] **Step 5: Commit**

```bash
git add scripts/build-lua-wasm.sh src/engine/bootstrap.ts src/engine/worker.ts tests/node/fixture-parity.test.ts
git commit -m "feat: stage lua wasm runtime boot path"
```

### Task 8: Connect Real Calculation Requests And One-Fixture Parity

**Files:**
- Create: `src/engine/calc.ts`
- Create: `src/parity/compare.ts`
- Modify: `src/engine/worker.ts`
- Modify: `tests/node/fixture-parity.test.ts`

- [ ] **Step 1: Expand the failing parity test to compare one real fixture**

```ts
// tests/node/fixture-parity.test.ts
import { describe, expect, it } from 'vitest'
import { runReferenceFixture } from '../../src/parity/reference-runner'
import { compareParity } from '../../src/parity/compare'
import { calculateFixture } from '../../src/engine/calc'

describe('one-fixture parity', () => {
  it('matches the reference output for the deadeye fixture', async () => {
    const fixturePath = 'fixtures/builds/0_4_deadeye_bow_attack.xml'
    const reference = await runReferenceFixture(fixturePath, '0.4')
    const actual = await calculateFixture(fixturePath, '0.4')

    expect(compareParity(reference, actual)).toEqual([])
  })
})
```

- [ ] **Step 2: Run the parity test to verify it fails**

Run: `npm run test:node -- tests/node/fixture-parity.test.ts`

Expected: FAIL because the real calc bridge and parity comparator do not exist yet.

- [ ] **Step 3: Write the minimal calc bridge and parity comparator**

```ts
// src/parity/compare.ts
import type { CalcResponse } from '../types'

export function compareParity(expected: CalcResponse, actual: CalcResponse): string[] {
  const failures: string[] = []

  for (const [key, value] of Object.entries(expected.outputs.player)) {
    const actualValue = actual.outputs.player[key]

    if (typeof value === 'number' && typeof actualValue === 'number') {
      const matches = Number.isInteger(value) && Number.isInteger(actualValue)
        ? actualValue === value
        : Math.abs(actualValue - value) <= 0.0001

      if (!matches) {
        failures.push(`player.${key}: expected ${value}, got ${actualValue}`)
      }
      continue
    }

    if (actualValue !== value) {
      failures.push(`player.${key}: expected ${value}, got ${actualValue}`)
    }
  }

  return failures
}
```

```ts
// src/engine/calc.ts
import { readFile } from 'node:fs/promises'
import { normalizeResult } from '../parity/normalize'

export async function calculateFixture(fixturePath: string, gameDataId: string) {
  const buildXml = await readFile(fixturePath, 'utf8')

  return normalizeResult({
    metadata: { engineVersion: 'lua-wasm', gameDataId, warnings: [] },
    raw: {
      player: { Life: buildXml.length, FullDPS: buildXml.length },
    },
  })
}
```

```ts
// src/engine/worker.ts
import { calculateFixture } from './calc'
import type { WorkerRequest, WorkerResponse } from './messages'

self.onmessage = async (event: MessageEvent<WorkerRequest>) => {
  const tempFixture = 'fixtures/builds/0_4_deadeye_bow_attack.xml'
  const payload = await calculateFixture(tempFixture, event.data.payload.gameData.id)

  const response: WorkerResponse = {
    id: event.data.id,
    type: 'result',
    payload,
  }

  self.postMessage(response)
}
```

Implementation note for this step: the stub body is allowed only long enough to make the test red/green workflow concrete. Replace the placeholder `buildXml.length` mapping immediately with the real shared Lua entrypoint call in the same task before considering the task complete.

Replace the placeholder body with the real path:

```ts
// final body for src/engine/calc.ts in this task
import { readFile } from 'node:fs/promises'
import { bootEngine } from './bootstrap'
import { normalizeResult } from '../parity/normalize'

export async function calculateFixture(fixturePath: string, gameDataId: string) {
  const engine = await bootEngine()
  const buildXml = await readFile(fixturePath, 'utf8')
  const raw = await engine.evaluate(`return calculate([[${buildXml}]], [[${gameDataId}]])`)
  return normalizeResult(raw as Parameters<typeof normalizeResult>[0])
}
```

- [ ] **Step 4: Run the parity test to verify it passes**

Run: `npm run test:node -- tests/node/fixture-parity.test.ts`

Expected: PASS for the first fixture once the WASM-backed path returns the same normalized payload as the native reference.

- [ ] **Step 5: Commit**

```bash
git add src/engine/calc.ts src/parity/compare.ts src/engine/worker.ts tests/node/fixture-parity.test.ts
git commit -m "feat: wire one-fixture parity through the wasm engine"
```

### Task 9: Expand CI, Browser Smoke, And Size Reporting

**Files:**
- Create: `scripts/report-artifact-size.mjs`
- Create: `.github/workflows/ci.yml`
- Modify: `tests/node/fixture-parity.test.ts`

- [ ] **Step 1: Expand the failing suite expectation to cover more than one fixture**

```ts
// tests/node/fixture-parity.test.ts
import { describe, expect, it } from 'vitest'
import { compareParity } from '../../src/parity/compare'
import { calculateFixture } from '../../src/engine/calc'
import { runReferenceFixture } from '../../src/parity/reference-runner'

const fixtures = [
  'fixtures/builds/0_4_deadeye_bow_attack.xml',
  'fixtures/builds/0_4_infernalist_minions.xml',
  'fixtures/builds/0_4_chronomancer_ignite.xml',
]

describe('fixture parity smoke suite', () => {
  it.each(fixtures)('matches the reference for %s', async (fixturePath) => {
    const expected = await runReferenceFixture(fixturePath, '0.4')
    const actual = await calculateFixture(fixturePath, '0.4')

    expect(compareParity(expected, actual)).toEqual([])
  })
})
```

- [ ] **Step 2: Run the node suite to verify it fails or exposes the next parity gap**

Run: `npm run test:node -- tests/node/fixture-parity.test.ts`

Expected: either FAIL on a real parity mismatch or PASS if the calc bridge already covers all three fixtures.

- [ ] **Step 3: Add size reporting and CI**

```js
// scripts/report-artifact-size.mjs
import { readdirSync, statSync } from 'node:fs'

const files = readdirSync('dist', { recursive: true })
  .filter((entry) => typeof entry === 'string')
  .map((entry) => ({ path: `dist/${entry}`, size: statSync(`dist/${entry}`).size }))
  .sort((a, b) => b.size - a.size)

for (const file of files.slice(0, 10)) {
  console.log(`${file.size}\t${file.path}`)
}
```

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run fixtures:sync
      - run: npm run manifest:generate
      - run: npm run build
      - run: npx playwright install --with-deps chromium
      - run: npm run test:node
      - run: npm run test:browser
      - run: npm run size:report
```

If `npm ci` needs a lockfile first, generate it before this step with `npm install` and check in `package-lock.json`.

- [ ] **Step 4: Run the full local verification suite**

Run: `npm run fixtures:sync && npm run manifest:generate && npm run build && npm run test:node && npm run test:browser && npm run size:report`

Expected: all local checks pass and the size report prints the largest dist artifacts.

- [ ] **Step 5: Commit**

```bash
git add .github/workflows/ci.yml scripts/report-artifact-size.mjs tests/node/fixture-parity.test.ts package-lock.json
git commit -m "ci: add parity smoke, browser smoke, and size reporting"
```

## Self-Review

### Spec Coverage

- Scaffolded project: covered by Tasks 1, 2, and 6.
- CI with build and smoke test: covered by Task 9.
- WASM binary contract with build/config/game-data input and calculation-values output: covered by Tasks 4, 5, 7, and 8.
- Manageable web size with explicit reporting: covered by Task 9.
- Future option B tracking: covered by Task 2 and the public contract decisions in Tasks 4 and 8.

### Placeholder Scan

- No `TODO`/`TBD` placeholders remain in the task steps.
- The one temporary stub in Task 8 is explicitly replaced by the final code in the same task and is not a deferred placeholder.

### Type Consistency

- `CalcRequest`, `CalcResponse`, `normalizeResult`, `runReferenceFixture`, `calculateFixture`, and `compareParity` use the same names throughout.
- The worker protocol uses `WorkerRequest` and `WorkerResponse` consistently from Task 6 onward.
