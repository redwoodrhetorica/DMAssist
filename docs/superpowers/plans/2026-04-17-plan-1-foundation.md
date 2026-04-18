# DM Assistant — Plan 1: Foundation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Scaffold the Electron + React app with a working event bus, all core data models, SQLite session storage, campaign/character file storage, and OS keychain settings.

**Architecture:** electron-vite project with a TypeScript main process hosting the event bus and storage layer, and a React renderer for the UI shell. All inter-process communication goes through typed IPC channels. Data lives in `~/Documents/DMAssist/` — SQLite for session events, JSON for campaigns/characters, OS keychain for secrets.

**Tech Stack:** Electron, electron-vite, TypeScript, React, shadcn/ui, Tailwind CSS, better-sqlite3, keytar, Vitest

---

## File Map

```
src/
  main/
    index.ts                  # Electron main entry, creates BrowserWindow
    eventBus.ts               # Typed EventEmitter — core event bus
    ipc/
      handlers.ts             # All ipcMain.handle() registrations
    storage/
      paths.ts                # Resolve ~/Documents/DMAssist paths
      campaignStore.ts        # Read/write campaign + character JSON
      sessionStore.ts         # SQLite session database (better-sqlite3)
      settingsStore.ts        # App-wide settings JSON + keytar secrets
    models/
      events.ts               # RawEvent, TranscribedEvent, ClassifiedEvent types
      campaign.ts             # Campaign, Character types
      session.ts              # SessionEvent, Session types
      settings.ts             # AppSettings type
  renderer/
    main.tsx                  # React entry point
    App.tsx                   # Root component, routing
    pages/
      Home.tsx                # Session control placeholder
      Characters.tsx          # Character registry placeholder
      History.tsx             # Session history placeholder
      Settings.tsx            # Settings placeholder
    lib/
      ipc.ts                  # Typed ipcRenderer wrappers
  preload/
    index.ts                  # contextBridge exposure of ipc channels

tests/
  main/
    eventBus.test.ts
    storage/
      campaignStore.test.ts
      sessionStore.test.ts
      settingsStore.test.ts
      paths.test.ts
```

---

### Task 1: Scaffold the electron-vite project

**Files:**
- Create: `package.json`, `electron.vite.config.ts`, `tsconfig.json`, `tsconfig.node.json`, `tsconfig.web.json`
- Create: `src/main/index.ts`, `src/renderer/main.tsx`, `src/renderer/App.tsx`, `src/preload/index.ts`

- [ ] **Step 1: Scaffold with electron-vite**

```bash
cd C:/Users/mcdon/Documents/Repo/Claude/DMAssist
npx create-electron-vite@latest . --template react-ts
```

Expected: project files created, `package.json` present with `electron`, `vite`, `react` dependencies.

- [ ] **Step 2: Install additional dependencies**

```bash
npm install better-sqlite3 keytar
npm install -D @types/better-sqlite3 vitest @vitest/coverage-v8
```

- [ ] **Step 3: Verify dev server launches**

```bash
npm run dev
```

Expected: Electron window opens showing default React page. No errors in terminal.

- [ ] **Step 4: Commit**

```bash
git add .
git commit -m "feat: scaffold electron-vite react-ts project"
```

---

### Task 1b: Configure Vitest for Electron

**Files:**
- Create: `vitest.config.ts`

This is required before any tests will run. Without it, `vi.mock('electron', ...)` and `vi.mock('keytar', ...)` fail with module-not-found errors.

- [ ] **Step 1: Create vitest.config.ts**

Create `vitest.config.ts`:

```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    environment: 'node',
    globals: true,
    exclude: ['**/node_modules/**', '**/dist/**'],
    // Prevent electron from actually launching during tests
    alias: {
      electron: new URL('./tests/__mocks__/electron.ts', import.meta.url).pathname
    }
  }
})
```

- [ ] **Step 2: Create Electron stub module**

Create `tests/__mocks__/electron.ts`:

```ts
import { vi } from 'vitest'

export const app = {
  getPath: vi.fn((name: string) => {
    if (name === 'documents') return '/mock/home'
    return `/mock/${name}`
  }),
  whenReady: vi.fn().mockResolvedValue(undefined),
  on: vi.fn(),
  quit: vi.fn()
}

export const ipcMain = {
  handle: vi.fn(),
  on: vi.fn(),
  removeAllListeners: vi.fn()
}

export const BrowserWindow = vi.fn().mockImplementation(() => ({
  loadURL: vi.fn(),
  loadFile: vi.fn(),
  webContents: { send: vi.fn() },
  on: vi.fn()
}))

export const clipboard = {
  writeText: vi.fn(),
  readText: vi.fn().mockReturnValue('')
}

export const contextBridge = { exposeInMainWorld: vi.fn() }
export const ipcRenderer = {
  invoke: vi.fn(),
  on: vi.fn(),
  removeAllListeners: vi.fn()
}
```

- [ ] **Step 3: Run existing tests to verify they now pass**

```bash
npx vitest run
```

Expected: Tests pass (or fail with "module not found" for modules not yet created — not with Electron import errors).

- [ ] **Step 4: Commit**

```bash
git add vitest.config.ts tests/__mocks__/electron.ts
git commit -m "test: add vitest config and electron stub for unit tests"
```

---

### Task 2: Install and configure shadcn/ui + Tailwind

**Files:**
- Modify: `src/renderer/main.tsx`, `src/renderer/App.tsx`
- Create: `src/renderer/index.css`, `components.json`, `tailwind.config.js`, `postcss.config.js`

- [ ] **Step 1: Install Tailwind**

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

- [ ] **Step 2: Configure Tailwind content paths**

Replace the contents of `tailwind.config.js`:

```js
/** @type {import('tailwindcss').Config} */
export default {
  content: ['./src/renderer/**/*.{ts,tsx}'],
  theme: { extend: {} },
  plugins: [],
}
```

- [ ] **Step 3: Add Tailwind directives to CSS**

Replace `src/renderer/index.css`:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

- [ ] **Step 4: Install shadcn/ui**

```bash
npx shadcn@latest init
```

When prompted:
- Style: Default
- Base color: Slate
- CSS variables: Yes

- [ ] **Step 5: Add a Button component to verify shadcn works**

```bash
npx shadcn@latest add button
```

- [ ] **Step 6: Update App.tsx to use Button**

```tsx
import { Button } from '@/components/ui/button'

export default function App() {
  return (
    <div className="flex h-screen items-center justify-center bg-background">
      <Button>DM Assistant</Button>
    </div>
  )
}
```

- [ ] **Step 7: Run dev and verify button renders**

```bash
npm run dev
```

Expected: Electron window shows a styled "DM Assistant" button.

- [ ] **Step 8: Commit**

```bash
git add .
git commit -m "feat: add tailwind and shadcn/ui"
```

---

### Task 3: Define core data models

**Files:**
- Create: `src/main/models/events.ts`
- Create: `src/main/models/campaign.ts`
- Create: `src/main/models/session.ts`
- Create: `src/main/models/settings.ts`

- [ ] **Step 1: Write event models**

Create `src/main/models/events.ts`:

```ts
export type InputSource = 'discord-voice' | 'discord-text' | 'local-mic'

export interface RawEvent {
  id: string
  timestamp: number
  source: InputSource
  speakerDiscordId?: string   // set for discord-voice, undefined for local-mic
  micLabel?: string           // optional label for local-mic inputs
  audioBuffer?: Buffer        // present for voice sources
  text?: string               // present for discord-text source
}

export interface TranscribedEvent extends RawEvent {
  text: string                // always present after transcription
}

export type EventType = 'dialogue' | 'action' | 'combat' | 'ooc' | 'meta' | 'unclassified'

export interface ClassifiedEvent extends TranscribedEvent {
  eventType: EventType
  characterId?: string        // references Character.id
  playerName?: string         // player name if OOC
  inGame: boolean
  confidence: number          // 0-1, flagged for review if < 0.7
  flagged: boolean
}
```

- [ ] **Step 2: Write campaign models**

Create `src/main/models/campaign.ts`:

```ts
export interface Character {
  id: string
  name: string
  playerName: string
  characterClass: string
  race: string
  description: string
  aliases: string[]
  archived: boolean
}

export interface Campaign {
  id: string
  name: string
  createdAt: number
  characters: Character[]
  metadata: Record<string, unknown>  // forward-allocated for Phase 2 lore pages
}
```

- [ ] **Step 3: Write session models**

Create `src/main/models/session.ts`:

```ts
import { EventType } from './events'

export interface SessionEvent {
  id: string
  sessionId: string
  timestamp: number
  speaker: string             // character name, player name, or 'unknown'
  rawText: string
  eventType: EventType
  characterId?: string
  inGame: boolean
  confidence: number
  flagged: boolean
}

export interface Session {
  id: string
  campaignId: string
  startedAt: number
  endedAt?: number
  title?: string
}
```

- [ ] **Step 4: Write settings model**

Create `src/main/models/settings.ts`:

```ts
export type TranscriptionQuality = 'good' | 'better' | 'best'
export type SummaryLength = 'short' | 'medium' | 'detailed'
export type CampaignTone = 'serious' | 'heroic' | 'humorous'

export interface OutputSettings {
  markdownEnabled: boolean
  markdownFolder: string
  discordEnabled: boolean
  discordChannelId?: string
  clipboardEnabled: boolean
}

export interface AppSettings {
  activeCampaignId?: string
  transcriptionQuality: TranscriptionQuality
  summaryTone: CampaignTone
  summaryLength: SummaryLength
  output: OutputSettings
  onboardingComplete: boolean
}
```

- [ ] **Step 5: Commit**

```bash
git add src/main/models/
git commit -m "feat: add core data models"
```

---

### Task 4: Storage paths helper

**Files:**
- Create: `src/main/storage/paths.ts`
- Create: `tests/main/storage/paths.test.ts`

- [ ] **Step 1: Write the failing test**

Create `tests/main/storage/paths.test.ts`:

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import path from 'path'

vi.mock('electron', () => ({
  app: { getPath: vi.fn(() => '/mock/home') }
}))

import { getBasePath, getCampaignPath, getSessionPath, getSettingsPath } from '../../../src/main/storage/paths'

describe('storage paths', () => {
  it('getBasePath returns DMAssist under documents', () => {
    expect(getBasePath()).toBe(path.join('/mock/home', 'DMAssist'))
  })

  it('getCampaignPath returns campaign json path', () => {
    expect(getCampaignPath('abc')).toBe(
      path.join('/mock/home', 'DMAssist', 'campaigns', 'abc', 'campaign.json')
    )
  })

  it('getSessionPath returns session db path', () => {
    expect(getSessionPath('abc', 'sess1')).toBe(
      path.join('/mock/home', 'DMAssist', 'campaigns', 'abc', 'sessions', 'sess1.db')
    )
  })

  it('getSettingsPath returns settings json path', () => {
    expect(getSettingsPath()).toBe(
      path.join('/mock/home', 'DMAssist', 'settings.json')
    )
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

```bash
npx vitest run tests/main/storage/paths.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement paths module**

Create `src/main/storage/paths.ts`:

```ts
import { app } from 'electron'
import path from 'path'

export function getBasePath(): string {
  return path.join(app.getPath('documents'), 'DMAssist')
}

export function getCampaignPath(campaignId: string): string {
  return path.join(getBasePath(), 'campaigns', campaignId, 'campaign.json')
}

export function getSessionPath(campaignId: string, sessionId: string): string {
  return path.join(getBasePath(), 'campaigns', campaignId, 'sessions', `${sessionId}.db`)
}

export function getSettingsPath(): string {
  return path.join(getBasePath(), 'settings.json')
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
npx vitest run tests/main/storage/paths.test.ts
```

Expected: PASS — 4 tests passing.

- [ ] **Step 5: Commit**

```bash
git add src/main/storage/paths.ts tests/main/storage/paths.test.ts
git commit -m "feat: add storage path helpers"
```

---

### Task 5: Campaign store (character registry file I/O)

**Files:**
- Create: `src/main/storage/campaignStore.ts`
- Create: `tests/main/storage/campaignStore.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `tests/main/storage/campaignStore.test.ts`:

```ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest'
import fs from 'fs'
import path from 'path'
import os from 'os'
import { CampaignStore } from '../../../src/main/storage/campaignStore'
import { Campaign, Character } from '../../../src/main/models/campaign'

let tmpDir: string
let store: CampaignStore

beforeEach(() => {
  tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), 'dmassist-'))
  store = new CampaignStore(tmpDir)
})

afterEach(() => {
  fs.rmSync(tmpDir, { recursive: true })
})

describe('CampaignStore', () => {
  it('saves and loads a campaign', () => {
    const campaign: Campaign = {
      id: 'c1', name: 'The Lost Mines', createdAt: 1000, characters: []
    }
    store.saveCampaign(campaign)
    expect(store.loadCampaign('c1')).toEqual(campaign)
  })

  it('returns null for missing campaign', () => {
    expect(store.loadCampaign('missing')).toBeNull()
  })

  it('lists all campaign ids', () => {
    store.saveCampaign({ id: 'c1', name: 'A', createdAt: 1000, characters: [] })
    store.saveCampaign({ id: 'c2', name: 'B', createdAt: 2000, characters: [] })
    expect(store.listCampaignIds().sort()).toEqual(['c1', 'c2'])
  })

  it('adds and retrieves a character', () => {
    const campaign: Campaign = { id: 'c1', name: 'Test', createdAt: 1000, characters: [] }
    store.saveCampaign(campaign)

    const character: Character = {
      id: 'ch1', name: 'Theron', playerName: 'Jake',
      characterClass: 'Paladin', race: 'Human',
      description: 'Lawful good', aliases: ['Theron', 'Sir Theron'],
      archived: false
    }
    store.addCharacter('c1', character)

    const loaded = store.loadCampaign('c1')
    expect(loaded?.characters).toHaveLength(1)
    expect(loaded?.characters[0].name).toBe('Theron')
  })

  it('archives a character without deleting', () => {
    const campaign: Campaign = {
      id: 'c1', name: 'Test', createdAt: 1000,
      characters: [{
        id: 'ch1', name: 'Theron', playerName: 'Jake',
        characterClass: 'Paladin', race: 'Human',
        description: '', aliases: [], archived: false
      }]
    }
    store.saveCampaign(campaign)
    store.archiveCharacter('c1', 'ch1')

    const loaded = store.loadCampaign('c1')
    expect(loaded?.characters[0].archived).toBe(true)
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npx vitest run tests/main/storage/campaignStore.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement CampaignStore**

Create `src/main/storage/campaignStore.ts`:

```ts
import fs from 'fs'
import path from 'path'
import { Campaign, Character } from '../models/campaign'

export class CampaignStore {
  constructor(private baseDir: string) {}

  private campaignPath(campaignId: string): string {
    return path.join(this.baseDir, 'campaigns', campaignId, 'campaign.json')
  }

  saveCampaign(campaign: Campaign): void {
    const filePath = this.campaignPath(campaign.id)
    fs.mkdirSync(path.dirname(filePath), { recursive: true })
    fs.writeFileSync(filePath, JSON.stringify(campaign, null, 2), 'utf-8')
  }

  loadCampaign(campaignId: string): Campaign | null {
    const filePath = this.campaignPath(campaignId)
    if (!fs.existsSync(filePath)) return null
    return JSON.parse(fs.readFileSync(filePath, 'utf-8')) as Campaign
  }

  listCampaignIds(): string[] {
    const dir = path.join(this.baseDir, 'campaigns')
    if (!fs.existsSync(dir)) return []
    return fs.readdirSync(dir).filter(entry =>
      fs.existsSync(path.join(dir, entry, 'campaign.json'))
    )
  }

  addCharacter(campaignId: string, character: Character): void {
    const campaign = this.loadCampaign(campaignId)
    if (!campaign) throw new Error(`Campaign ${campaignId} not found`)
    campaign.characters.push(character)
    this.saveCampaign(campaign)
  }

  updateCharacter(campaignId: string, character: Character): void {
    const campaign = this.loadCampaign(campaignId)
    if (!campaign) throw new Error(`Campaign ${campaignId} not found`)
    campaign.characters = campaign.characters.map(c => c.id === character.id ? character : c)
    this.saveCampaign(campaign)
  }

  archiveCharacter(campaignId: string, characterId: string): void {
    const campaign = this.loadCampaign(campaignId)
    if (!campaign) throw new Error(`Campaign ${campaignId} not found`)
    campaign.characters = campaign.characters.map(c =>
      c.id === characterId ? { ...c, archived: true } : c
    )
    this.saveCampaign(campaign)
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npx vitest run tests/main/storage/campaignStore.test.ts
```

Expected: PASS — 5 tests passing.

- [ ] **Step 5: Commit**

```bash
git add src/main/storage/campaignStore.ts tests/main/storage/campaignStore.test.ts
git commit -m "feat: add campaign store with character registry"
```

---

### Task 6: Session store (SQLite event log)

**Files:**
- Create: `src/main/storage/sessionStore.ts`
- Create: `tests/main/storage/sessionStore.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `tests/main/storage/sessionStore.test.ts`:

```ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest'
import fs from 'fs'
import path from 'path'
import os from 'os'
import { SessionStore } from '../../../src/main/storage/sessionStore'
import { SessionEvent } from '../../../src/main/models/session'

let tmpDir: string
let store: SessionStore

beforeEach(() => {
  tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), 'dmassist-'))
  store = new SessionStore(path.join(tmpDir, 'test.db'))
})

afterEach(() => {
  store.close()
  fs.rmSync(tmpDir, { recursive: true })
})

const makeEvent = (overrides: Partial<SessionEvent> = {}): SessionEvent => ({
  id: 'e1',
  sessionId: 's1',
  timestamp: 1000,
  speaker: 'Theron',
  rawText: 'I draw my sword.',
  eventType: 'action',
  inGame: true,
  confidence: 0.95,
  flagged: false,
  ...overrides
})

describe('SessionStore', () => {
  it('inserts and retrieves a session event', () => {
    store.insertEvent(makeEvent())
    const events = store.getEvents('s1')
    expect(events).toHaveLength(1)
    expect(events[0].rawText).toBe('I draw my sword.')
  })

  it('returns empty array for unknown session', () => {
    expect(store.getEvents('unknown')).toEqual([])
  })

  it('returns only flagged events', () => {
    store.insertEvent(makeEvent({ id: 'e1', flagged: false }))
    store.insertEvent(makeEvent({ id: 'e2', flagged: true, rawText: 'ambiguous' }))
    const flagged = store.getFlaggedEvents('s1')
    expect(flagged).toHaveLength(1)
    expect(flagged[0].rawText).toBe('ambiguous')
  })

  it('inserts and retrieves session metadata', () => {
    store.createSession({ id: 's1', campaignId: 'c1', startedAt: 1000 })
    const session = store.getSession('s1')
    expect(session?.campaignId).toBe('c1')
  })

  it('updates session end time', () => {
    store.createSession({ id: 's1', campaignId: 'c1', startedAt: 1000 })
    store.endSession('s1', 2000)
    const session = store.getSession('s1')
    expect(session?.endedAt).toBe(2000)
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npx vitest run tests/main/storage/sessionStore.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement SessionStore**

Create `src/main/storage/sessionStore.ts`:

```ts
import Database from 'better-sqlite3'
import { SessionEvent, Session } from '../models/session'

export class SessionStore {
  private db: Database.Database

  constructor(dbPath: string) {
    this.db = new Database(dbPath)
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS sessions (
        id TEXT PRIMARY KEY,
        campaignId TEXT NOT NULL,
        startedAt INTEGER NOT NULL,
        endedAt INTEGER,
        title TEXT
      );
      CREATE TABLE IF NOT EXISTS events (
        id TEXT PRIMARY KEY,
        sessionId TEXT NOT NULL,
        timestamp INTEGER NOT NULL,
        speaker TEXT NOT NULL,
        rawText TEXT NOT NULL,
        eventType TEXT NOT NULL,
        characterId TEXT,
        inGame INTEGER NOT NULL,
        confidence REAL NOT NULL,
        flagged INTEGER NOT NULL
      );
    `)
  }

  createSession(session: Session): void {
    this.db.prepare(`
      INSERT INTO sessions (id, campaignId, startedAt, endedAt, title)
      VALUES (@id, @campaignId, @startedAt, @endedAt, @title)
    `).run({ ...session, endedAt: session.endedAt ?? null, title: session.title ?? null })
  }

  getSession(sessionId: string): Session | null {
    const row = this.db.prepare('SELECT * FROM sessions WHERE id = ?').get(sessionId) as any
    if (!row) return null
    return { ...row, endedAt: row.endedAt ?? undefined, title: row.title ?? undefined }
  }

  endSession(sessionId: string, endedAt: number): void {
    this.db.prepare('UPDATE sessions SET endedAt = ? WHERE id = ?').run(endedAt, sessionId)
  }

  insertEvent(event: SessionEvent): void {
    this.db.prepare(`
      INSERT INTO events (id, sessionId, timestamp, speaker, rawText, eventType, characterId, inGame, confidence, flagged)
      VALUES (@id, @sessionId, @timestamp, @speaker, @rawText, @eventType, @characterId, @inGame, @confidence, @flagged)
    `).run({
      ...event,
      characterId: event.characterId ?? null,
      inGame: event.inGame ? 1 : 0,
      flagged: event.flagged ? 1 : 0
    })
  }

  getEvents(sessionId: string): SessionEvent[] {
    const rows = this.db.prepare('SELECT * FROM events WHERE sessionId = ? ORDER BY timestamp ASC').all(sessionId) as any[]
    return rows.map(r => ({ ...r, inGame: r.inGame === 1, flagged: r.flagged === 1, characterId: r.characterId ?? undefined }))
  }

  getFlaggedEvents(sessionId: string): SessionEvent[] {
    const rows = this.db.prepare('SELECT * FROM events WHERE sessionId = ? AND flagged = 1 ORDER BY timestamp ASC').all(sessionId) as any[]
    return rows.map(r => ({ ...r, inGame: r.inGame === 1, flagged: r.flagged === 1, characterId: r.characterId ?? undefined }))
  }

  close(): void {
    this.db.close()
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npx vitest run tests/main/storage/sessionStore.test.ts
```

Expected: PASS — 5 tests passing.

- [ ] **Step 5: Commit**

```bash
git add src/main/storage/sessionStore.ts tests/main/storage/sessionStore.test.ts
git commit -m "feat: add session store with SQLite event log"
```

---

### Task 7: Settings store (JSON + keytar secrets)

**Files:**
- Create: `src/main/storage/settingsStore.ts`
- Create: `tests/main/storage/settingsStore.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `tests/main/storage/settingsStore.test.ts`:

```ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import fs from 'fs'
import path from 'path'
import os from 'os'

vi.mock('keytar', () => ({
  default: {
    setPassword: vi.fn(),
    getPassword: vi.fn().mockResolvedValue('mock-api-key'),
    deletePassword: vi.fn()
  }
}))

import { SettingsStore } from '../../../src/main/storage/settingsStore'

let tmpDir: string
let store: SettingsStore

beforeEach(() => {
  tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), 'dmassist-'))
  store = new SettingsStore(path.join(tmpDir, 'settings.json'))
})

afterEach(() => {
  fs.rmSync(tmpDir, { recursive: true })
})

describe('SettingsStore', () => {
  it('returns default settings when no file exists', () => {
    const settings = store.load()
    expect(settings.onboardingComplete).toBe(false)
    expect(settings.transcriptionQuality).toBe('good')
    expect(settings.output.markdownEnabled).toBe(true)
  })

  it('saves and reloads settings', () => {
    const settings = store.load()
    settings.onboardingComplete = true
    store.save(settings)

    const reloaded = store.load()
    expect(reloaded.onboardingComplete).toBe(true)
  })

  it('saves and retrieves a secret via keytar', async () => {
    await store.setSecret('claude-api-key', 'sk-ant-test')
    const val = await store.getSecret('claude-api-key')
    expect(val).toBe('mock-api-key')
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npx vitest run tests/main/storage/settingsStore.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement SettingsStore**

Create `src/main/storage/settingsStore.ts`:

```ts
import fs from 'fs'
import path from 'path'
import keytar from 'keytar'
import { AppSettings } from '../models/settings'

const SERVICE_NAME = 'DMAssist'

const defaults: AppSettings = {
  transcriptionQuality: 'good',
  summaryTone: 'serious',
  summaryLength: 'medium',
  onboardingComplete: false,
  output: {
    markdownEnabled: true,
    markdownFolder: '',
    discordEnabled: false,
    clipboardEnabled: false
  }
}

export class SettingsStore {
  constructor(private filePath: string) {}

  load(): AppSettings {
    if (!fs.existsSync(this.filePath)) return { ...defaults, output: { ...defaults.output } }
    return JSON.parse(fs.readFileSync(this.filePath, 'utf-8')) as AppSettings
  }

  save(settings: AppSettings): void {
    fs.mkdirSync(path.dirname(this.filePath), { recursive: true })
    fs.writeFileSync(this.filePath, JSON.stringify(settings, null, 2), 'utf-8')
  }

  async setSecret(key: string, value: string): Promise<void> {
    await keytar.setPassword(SERVICE_NAME, key, value)
  }

  async getSecret(key: string): Promise<string | null> {
    return keytar.getPassword(SERVICE_NAME, key)
  }

  async deleteSecret(key: string): Promise<void> {
    await keytar.deletePassword(SERVICE_NAME, key)
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npx vitest run tests/main/storage/settingsStore.test.ts
```

Expected: PASS — 3 tests passing.

- [ ] **Step 5: Commit**

```bash
git add src/main/storage/settingsStore.ts tests/main/storage/settingsStore.test.ts
git commit -m "feat: add settings store with keytar secret management"
```

---

### Task 8: Event bus

**Files:**
- Create: `src/main/eventBus.ts`
- Create: `tests/main/eventBus.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `tests/main/eventBus.test.ts`:

```ts
import { describe, it, expect, vi } from 'vitest'
import { EventBus } from '../../src/main/eventBus'
import { RawEvent, ClassifiedEvent } from '../../src/main/models/events'

describe('EventBus', () => {
  it('emits and receives a raw event', () => {
    const bus = new EventBus()
    const handler = vi.fn()
    bus.on('raw', handler)

    const event: RawEvent = {
      id: 'e1', timestamp: 1000, source: 'local-mic', text: 'hello'
    }
    bus.emit('raw', event)
    expect(handler).toHaveBeenCalledWith(event)
  })

  it('emits and receives a classified event', () => {
    const bus = new EventBus()
    const handler = vi.fn()
    bus.on('classified', handler)

    const event: ClassifiedEvent = {
      id: 'e1', timestamp: 1000, source: 'local-mic', text: 'I attack',
      eventType: 'combat', inGame: true, confidence: 0.9, flagged: false
    }
    bus.emit('classified', event)
    expect(handler).toHaveBeenCalledWith(event)
  })

  it('removes a listener with off()', () => {
    const bus = new EventBus()
    const handler = vi.fn()
    bus.on('raw', handler)
    bus.off('raw', handler)

    bus.emit('raw', { id: 'e1', timestamp: 1000, source: 'local-mic' })
    expect(handler).not.toHaveBeenCalled()
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npx vitest run tests/main/eventBus.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement EventBus**

Create `src/main/eventBus.ts`:

```ts
import { EventEmitter } from 'events'
import { RawEvent, TranscribedEvent, ClassifiedEvent } from './models/events'

interface BusEvents {
  raw: RawEvent
  transcribed: TranscribedEvent
  classified: ClassifiedEvent
  error: Error
}

export class EventBus {
  private emitter = new EventEmitter()

  on<K extends keyof BusEvents>(event: K, listener: (data: BusEvents[K]) => void): void {
    this.emitter.on(event, listener)
  }

  off<K extends keyof BusEvents>(event: K, listener: (data: BusEvents[K]) => void): void {
    this.emitter.off(event, listener)
  }

  emit<K extends keyof BusEvents>(event: K, data: BusEvents[K]): void {
    this.emitter.emit(event, data)
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npx vitest run tests/main/eventBus.test.ts
```

Expected: PASS — 3 tests passing.

- [ ] **Step 5: Commit**

```bash
git add src/main/eventBus.ts tests/main/eventBus.test.ts
git commit -m "feat: add typed event bus"
```

---

### Task 9: IPC handlers and preload bridge

**Files:**
- Create: `src/main/ipc/handlers.ts`
- Modify: `src/preload/index.ts`
- Modify: `src/main/index.ts`
- Create: `src/renderer/lib/ipc.ts`

- [ ] **Step 1: Define IPC channels as constants**

Create `src/main/ipc/handlers.ts`:

```ts
import { ipcMain } from 'electron'
import { CampaignStore } from '../storage/campaignStore'
import { SettingsStore } from '../storage/settingsStore'
import { Campaign, Character } from '../models/campaign'
import { AppSettings } from '../models/settings'

export function registerHandlers(
  campaignStore: CampaignStore,
  settingsStore: SettingsStore
): void {
  ipcMain.handle('settings:load', () => settingsStore.load())
  ipcMain.handle('settings:save', (_e, settings: AppSettings) => settingsStore.save(settings))

  ipcMain.handle('campaign:list', () => campaignStore.listCampaignIds())
  ipcMain.handle('campaign:load', (_e, id: string) => campaignStore.loadCampaign(id))
  ipcMain.handle('campaign:save', (_e, campaign: Campaign) => campaignStore.saveCampaign(campaign))

  ipcMain.handle('character:add', (_e, campaignId: string, character: Character) =>
    campaignStore.addCharacter(campaignId, character))
  ipcMain.handle('character:update', (_e, campaignId: string, character: Character) =>
    campaignStore.updateCharacter(campaignId, character))
  ipcMain.handle('character:archive', (_e, campaignId: string, characterId: string) =>
    campaignStore.archiveCharacter(campaignId, characterId))
}
```

- [ ] **Step 2: Expose IPC in preload**

Replace `src/preload/index.ts`:

```ts
import { contextBridge, ipcRenderer } from 'electron'

const api = {
  settings: {
    load: () => ipcRenderer.invoke('settings:load'),
    save: (s: unknown) => ipcRenderer.invoke('settings:save', s)
  },
  campaign: {
    list: () => ipcRenderer.invoke('campaign:list'),
    load: (id: string) => ipcRenderer.invoke('campaign:load', id),
    save: (c: unknown) => ipcRenderer.invoke('campaign:save', c)
  },
  character: {
    add: (campaignId: string, character: unknown) =>
      ipcRenderer.invoke('character:add', campaignId, character),
    update: (campaignId: string, character: unknown) =>
      ipcRenderer.invoke('character:update', campaignId, character),
    archive: (campaignId: string, characterId: string) =>
      ipcRenderer.invoke('character:archive', campaignId, characterId)
  }
}

contextBridge.exposeInMainWorld('dmAssist', api)
```

- [ ] **Step 3: Create typed renderer IPC wrapper**

Create `src/renderer/lib/ipc.ts`:

```ts
import { Campaign, Character } from '../../main/models/campaign'
import { AppSettings } from '../../main/models/settings'

const api = (window as any).dmAssist

export const ipc = {
  settings: {
    load: (): Promise<AppSettings> => api.settings.load(),
    save: (s: AppSettings): Promise<void> => api.settings.save(s)
  },
  campaign: {
    list: (): Promise<string[]> => api.campaign.list(),
    load: (id: string): Promise<Campaign | null> => api.campaign.load(id),
    save: (c: Campaign): Promise<void> => api.campaign.save(c)
  },
  character: {
    add: (campaignId: string, character: Character): Promise<void> =>
      api.character.add(campaignId, character),
    update: (campaignId: string, character: Character): Promise<void> =>
      api.character.update(campaignId, character),
    archive: (campaignId: string, characterId: string): Promise<void> =>
      api.character.archive(campaignId, characterId)
  }
}
```

- [ ] **Step 4: Wire handlers into main process**

Modify `src/main/index.ts` to instantiate stores and register handlers. Add after existing imports:

```ts
import { app, BrowserWindow } from 'electron'
import path from 'path'
import { CampaignStore } from './storage/campaignStore'
import { SettingsStore } from './storage/settingsStore'
import { registerHandlers } from './ipc/handlers'
import { getBasePath, getSettingsPath } from './storage/paths'

app.whenReady().then(() => {
  const campaignStore = new CampaignStore(getBasePath())
  const settingsStore = new SettingsStore(getSettingsPath())
  registerHandlers(campaignStore, settingsStore)

  const win = new BrowserWindow({
    width: 1200,
    height: 800,
    webPreferences: {
      preload: path.join(__dirname, '../preload/index.js'),
      contextIsolation: true,
      nodeIntegration: false
    }
  })

  if (process.env.NODE_ENV === 'development') {
    win.loadURL('http://localhost:5173')
  } else {
    win.loadFile(path.join(__dirname, '../renderer/index.html'))
  }
})

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})
```

- [ ] **Step 5: Run dev and verify no console errors**

```bash
npm run dev
```

Expected: Electron window opens, no IPC errors in DevTools console.

- [ ] **Step 6: Commit**

```bash
git add src/main/ipc/ src/preload/index.ts src/main/index.ts src/renderer/lib/ipc.ts
git commit -m "feat: add IPC handlers and preload bridge"
```

---

### Task 10: React app shell with navigation

**Files:**
- Modify: `src/renderer/App.tsx`
- Create: `src/renderer/pages/Home.tsx`
- Create: `src/renderer/pages/Characters.tsx`
- Create: `src/renderer/pages/History.tsx`
- Create: `src/renderer/pages/Settings.tsx`
- Create: `src/renderer/components/Sidebar.tsx`

- [ ] **Step 1: Install React Router**

```bash
npm install react-router-dom
```

- [ ] **Step 2: Add shadcn components needed for shell**

```bash
npx shadcn@latest add separator tooltip
```

- [ ] **Step 3: Create page stubs**

Create `src/renderer/pages/Home.tsx`:
```tsx
export default function Home() {
  return <div className="p-6"><h1 className="text-2xl font-bold">Session</h1></div>
}
```

Create `src/renderer/pages/Characters.tsx`:
```tsx
export default function Characters() {
  return <div className="p-6"><h1 className="text-2xl font-bold">Characters</h1></div>
}
```

Create `src/renderer/pages/History.tsx`:
```tsx
export default function History() {
  return <div className="p-6"><h1 className="text-2xl font-bold">Session History</h1></div>
}
```

Create `src/renderer/pages/Settings.tsx`:
```tsx
export default function Settings() {
  return <div className="p-6"><h1 className="text-2xl font-bold">Settings</h1></div>
}
```

- [ ] **Step 4: Create sidebar navigation**

Create `src/renderer/components/Sidebar.tsx`:

```tsx
import { NavLink } from 'react-router-dom'
import { Sword, Users, BookOpen, Settings } from 'lucide-react'
import { cn } from '@/lib/utils'

const links = [
  { to: '/', label: 'Session', icon: Sword },
  { to: '/characters', label: 'Characters', icon: Users },
  { to: '/history', label: 'History', icon: BookOpen },
  { to: '/settings', label: 'Settings', icon: Settings }
]

export default function Sidebar() {
  return (
    <aside className="w-56 border-r bg-muted/30 flex flex-col gap-1 p-3">
      <div className="px-3 py-4 text-lg font-bold tracking-tight">DM Assistant</div>
      {links.map(({ to, label, icon: Icon }) => (
        <NavLink
          key={to}
          to={to}
          end={to === '/'}
          className={({ isActive }) =>
            cn('flex items-center gap-3 rounded-md px-3 py-2 text-sm font-medium transition-colors',
              isActive ? 'bg-primary text-primary-foreground' : 'hover:bg-muted')
          }
        >
          <Icon className="h-4 w-4" />
          {label}
        </NavLink>
      ))}
    </aside>
  )
}
```

- [ ] **Step 5: Wire up App.tsx with router**

Replace `src/renderer/App.tsx`:

```tsx
import { MemoryRouter, Routes, Route } from 'react-router-dom'
import Sidebar from './components/Sidebar'
import Home from './pages/Home'
import Characters from './pages/Characters'
import History from './pages/History'
import Settings from './pages/Settings'

export default function App() {
  return (
    <MemoryRouter>
      <div className="flex h-screen bg-background text-foreground">
        <Sidebar />
        <main className="flex-1 overflow-auto">
          <Routes>
            <Route path="/" element={<Home />} />
            <Route path="/characters" element={<Characters />} />
            <Route path="/history" element={<History />} />
            <Route path="/settings" element={<Settings />} />
          </Routes>
        </main>
      </div>
    </MemoryRouter>
  )
}
```

- [ ] **Step 6: Install lucide-react**

```bash
npm install lucide-react
```

- [ ] **Step 7: Run dev and verify navigation works**

```bash
npm run dev
```

Expected: Electron window shows sidebar with 4 nav items. Clicking each navigates to the correct page stub.

- [ ] **Step 8: Commit**

```bash
git add src/renderer/
git commit -m "feat: add React app shell with sidebar navigation"
```

---

### Task 11: Whisper model download wizard (CEO cherry-pick)

**Files:**
- Create: `src/main/whisper/modelManager.ts`
- Create: `src/main/ipc/modelHandlers.ts`
- Create: `tests/main/whisper/modelManager.test.ts`

Non-technical users must be guided through the one-time Whisper model download. This is a first-launch blocker: sessions cannot start until a model `.bin` file exists. Models are stored at `~/Documents/DMAssist/models/`.

Model size: `ggml-base.bin` ~150 MB, `ggml-small.bin` ~460 MB, `ggml-medium.bin` ~1.5 GB.

Quality setting → model mapping:
- `good` → `ggml-base.bin`
- `better` → `ggml-small.bin`
- `best` → `ggml-medium.bin`

**Note:** Hugging Face model URLs may not support `Accept-Ranges`. Implement resume as best-effort; fall back to full re-download if the server does not return `206 Partial Content`.

- [ ] **Step 1: Write the failing tests**

Create `tests/main/whisper/modelManager.test.ts`:

```ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import fs from 'fs'
import path from 'path'
import os from 'os'
import { ModelManager } from '../../../src/main/whisper/modelManager'

let tmpDir: string
let manager: ModelManager

beforeEach(() => {
  tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), 'dmassist-'))
  manager = new ModelManager(path.join(tmpDir, 'models'))
})

afterEach(() => {
  fs.rmSync(tmpDir, { recursive: true })
})

describe('ModelManager', () => {
  it('isModelPresent returns false when file does not exist', () => {
    expect(manager.isModelPresent('good')).toBe(false)
  })

  it('isModelPresent returns true when model file exists', () => {
    const modelsDir = path.join(tmpDir, 'models')
    fs.mkdirSync(modelsDir, { recursive: true })
    fs.writeFileSync(path.join(modelsDir, 'ggml-base.bin'), 'fake')
    expect(manager.isModelPresent('good')).toBe(true)
  })

  it('getModelPath returns correct path for quality', () => {
    expect(manager.getModelPath('good')).toContain('ggml-base.bin')
    expect(manager.getModelPath('better')).toContain('ggml-small.bin')
    expect(manager.getModelPath('best')).toContain('ggml-medium.bin')
  })

  it('getDiskRequiredMb returns correct values', () => {
    expect(manager.getDiskRequiredMb('good')).toBe(150)
    expect(manager.getDiskRequiredMb('better')).toBe(460)
    expect(manager.getDiskRequiredMb('best')).toBe(1500)
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npx vitest run tests/main/whisper/modelManager.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement ModelManager**

Create `src/main/whisper/modelManager.ts`:

```ts
import fs from 'fs'
import path from 'path'
import https from 'https'
import { TranscriptionQuality } from '../models/settings'

const MODEL_FILES: Record<TranscriptionQuality, string> = {
  good: 'ggml-base.bin',
  better: 'ggml-small.bin',
  best: 'ggml-medium.bin'
}

const MODEL_URLS: Record<TranscriptionQuality, string> = {
  good: 'https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.bin',
  better: 'https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-small.bin',
  best: 'https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-medium.bin'
}

const DISK_REQUIRED_MB: Record<TranscriptionQuality, number> = {
  good: 150,
  better: 460,
  best: 1500
}

export type DownloadProgress = { percent: number; bytesReceived: number; totalBytes: number }

export class ModelManager {
  constructor(private modelsDir: string) {}

  isModelPresent(quality: TranscriptionQuality): boolean {
    return fs.existsSync(this.getModelPath(quality))
  }

  getModelPath(quality: TranscriptionQuality): string {
    return path.join(this.modelsDir, MODEL_FILES[quality])
  }

  getDiskRequiredMb(quality: TranscriptionQuality): number {
    return DISK_REQUIRED_MB[quality]
  }

  async download(
    quality: TranscriptionQuality,
    onProgress: (progress: DownloadProgress) => void,
    signal?: AbortSignal
  ): Promise<void> {
    fs.mkdirSync(this.modelsDir, { recursive: true })

    const destPath = this.getModelPath(quality)
    const tmpPath = destPath + '.tmp'
    const url = MODEL_URLS[quality]

    // Check for partial download (resume best-effort)
    let startByte = 0
    if (fs.existsSync(tmpPath)) {
      startByte = fs.statSync(tmpPath).size
    }

    await new Promise<void>((resolve, reject) => {
      const headers: Record<string, string> = {}
      if (startByte > 0) headers['Range'] = `bytes=${startByte}-`

      const req = https.get(url, { headers }, (res) => {
        // If server doesn't support range requests, restart from 0
        if (res.statusCode === 416 || (startByte > 0 && res.statusCode === 200)) {
          fs.rmSync(tmpPath, { force: true })
          startByte = 0
        }

        if (res.statusCode !== 200 && res.statusCode !== 206) {
          reject(new Error(`Download failed: HTTP ${res.statusCode}`))
          return
        }

        const totalBytes = parseInt(res.headers['content-length'] ?? '0', 10) + startByte
        let bytesReceived = startByte
        const writeStream = fs.createWriteStream(tmpPath, { flags: startByte > 0 ? 'a' : 'w' })

        res.on('data', (chunk: Buffer) => {
          if (signal?.aborted) { req.destroy(); writeStream.destroy(); return }
          bytesReceived += chunk.length
          const percent = totalBytes > 0 ? Math.round((bytesReceived / totalBytes) * 100) : 0
          onProgress({ percent, bytesReceived, totalBytes })
        })

        res.pipe(writeStream)
        writeStream.on('finish', () => {
          fs.renameSync(tmpPath, destPath)
          resolve()
        })
        writeStream.on('error', reject)
      })

      req.on('error', reject)
      signal?.addEventListener('abort', () => req.destroy())
    })
  }

  cancelAndCleanPartial(quality: TranscriptionQuality): void {
    const tmpPath = this.getModelPath(quality) + '.tmp'
    fs.rmSync(tmpPath, { force: true })
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npx vitest run tests/main/whisper/modelManager.test.ts
```

Expected: PASS — 4 tests passing.

- [ ] **Step 5: Add model IPC handlers**

Create `src/main/ipc/modelHandlers.ts`:

```ts
import { ipcMain } from 'electron'
import { ModelManager, DownloadProgress } from '../whisper/modelManager'
import { TranscriptionQuality } from '../models/settings'

export function registerModelHandlers(
  modelManager: ModelManager,
  getWindow: () => Electron.BrowserWindow | null
): void {
  let activeAbortController: AbortController | null = null

  ipcMain.handle('model:isPresent', (_e, quality: TranscriptionQuality) =>
    modelManager.isModelPresent(quality))

  ipcMain.handle('model:getPath', (_e, quality: TranscriptionQuality) =>
    modelManager.getModelPath(quality))

  ipcMain.handle('model:download', async (_e, quality: TranscriptionQuality) => {
    activeAbortController = new AbortController()

    try {
      await modelManager.download(
        quality,
        (progress: DownloadProgress) => {
          getWindow()?.webContents.send('model:progress', progress)
        },
        activeAbortController.signal
      )
      getWindow()?.webContents.send('model:complete', quality)
    } catch (err) {
      if (!activeAbortController.signal.aborted) {
        getWindow()?.webContents.send('model:error', (err as Error).message)
      }
    } finally {
      activeAbortController = null
    }
  })

  ipcMain.handle('model:cancel', (_e, quality: TranscriptionQuality) => {
    activeAbortController?.abort()
    activeAbortController = null
  })
}
```

- [ ] **Step 6: Wire model handlers into main process**

In `src/main/index.ts`, import and register model handlers alongside existing handlers:

```ts
import { ModelManager } from './whisper/modelManager'
import { registerModelHandlers } from './ipc/modelHandlers'
import path from 'path'

// In app.whenReady():
const modelsDir = path.join(getBasePath(), 'models')
const modelManager = new ModelManager(modelsDir)
registerModelHandlers(modelManager, () => win)
```

- [ ] **Step 7: Add model channels to preload**

Add to preload `src/preload/index.ts` api object:

```ts
model: {
  isPresent: (quality: string) => ipcRenderer.invoke('model:isPresent', quality),
  getPath: (quality: string) => ipcRenderer.invoke('model:getPath', quality),
  download: (quality: string) => ipcRenderer.invoke('model:download', quality),
  cancel: (quality: string) => ipcRenderer.invoke('model:cancel', quality),
  onProgress: (cb: (p: unknown) => void) => {
    ipcRenderer.on('model:progress', (_e, p) => cb(p))
    return () => ipcRenderer.removeAllListeners('model:progress')
  },
  onComplete: (cb: (quality: string) => void) => {
    ipcRenderer.on('model:complete', (_e, q) => cb(q))
    return () => ipcRenderer.removeAllListeners('model:complete')
  },
  onError: (cb: (msg: string) => void) => {
    ipcRenderer.on('model:error', (_e, msg) => cb(msg))
    return () => ipcRenderer.removeAllListeners('model:error')
  }
}
```

- [ ] **Step 8: Add model IPC wrapper to renderer**

Add to `src/renderer/lib/ipc.ts`:

```ts
import { DownloadProgress } from '../../main/whisper/modelManager'
import { TranscriptionQuality } from '../../main/models/settings'

// Add to ipc object:
model: {
  isPresent: (quality: TranscriptionQuality): Promise<boolean> => api.model.isPresent(quality),
  getPath: (quality: TranscriptionQuality): Promise<string> => api.model.getPath(quality),
  download: (quality: TranscriptionQuality): Promise<void> => api.model.download(quality),
  cancel: (quality: TranscriptionQuality): Promise<void> => api.model.cancel(quality),
  onProgress: (cb: (p: DownloadProgress) => void): (() => void) => api.model.onProgress(cb),
  onComplete: (cb: (quality: TranscriptionQuality) => void): (() => void) => api.model.onComplete(cb),
  onError: (cb: (msg: string) => void): (() => void) => api.model.onError(cb)
}
```

- [ ] **Step 9: Create ModelSetup UI component**

Create `src/renderer/components/ModelSetup.tsx`:

```tsx
import { useState, useEffect } from 'react'
import { Button } from '@/components/ui/button'
import { Progress } from '@/components/ui/progress'
import { ipc } from '../lib/ipc'
import { TranscriptionQuality } from '../../main/models/settings'

interface Props {
  quality: TranscriptionQuality
  onReady: () => void
}

export default function ModelSetup({ quality, onReady }: Props) {
  const [progress, setProgress] = useState(0)
  const [downloading, setDownloading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    const cleanupProgress = ipc.model.onProgress(p => setProgress(p.percent))
    const cleanupComplete = ipc.model.onComplete(() => { setDownloading(false); onReady() })
    const cleanupError = ipc.model.onError(msg => { setDownloading(false); setError(msg) })
    return () => { cleanupProgress(); cleanupComplete(); cleanupError() }
  }, [onReady])

  const handleDownload = async () => {
    setError(null)
    setProgress(0)
    setDownloading(true)
    await ipc.model.download(quality)
  }

  const sizeMap: Record<TranscriptionQuality, string> = { good: '~150 MB', better: '~460 MB', best: '~1.5 GB' }

  return (
    <div className="flex flex-col gap-4 p-6 max-w-md">
      <h2 className="text-lg font-semibold">One-time Setup</h2>
      <p className="text-sm text-muted-foreground">
        DM Assistant uses a local AI model to transcribe speech. You need to download it once ({sizeMap[quality]}).
        No audio ever leaves your computer.
      </p>
      {error && (
        <p className="text-sm text-destructive">{error}</p>
      )}
      {downloading ? (
        <div className="flex flex-col gap-2">
          <Progress value={progress} className="h-2" />
          <p className="text-xs text-muted-foreground text-right">{progress}%</p>
          <Button variant="outline" onClick={() => ipc.model.cancel(quality)} size="sm" className="w-fit">
            Cancel
          </Button>
        </div>
      ) : (
        <Button onClick={handleDownload}>
          {error ? 'Retry Download' : 'Download Transcription Model'}
        </Button>
      )}
    </div>
  )
}
```

Add required shadcn component first: `npx shadcn@latest add progress`

- [ ] **Step 10: Gate session start behind model presence check**

In `src/main/ipc/handlers.ts`, in the `session:start` handler, add a pre-flight check before creating the pipeline:

```ts
// After loading settings, before creating the pipeline:
const modelPath = modelManager.getModelPath(settings.transcriptionQuality)
if (!modelManager.isModelPresent(settings.transcriptionQuality)) {
  throw new Error('Whisper model not downloaded. Open the app to complete setup.')
}
```

Pass `modelManager` into `registerHandlers()` (update signature).

- [ ] **Step 11: Commit**

```bash
git add src/main/whisper/ src/main/ipc/modelHandlers.ts src/renderer/components/ModelSetup.tsx tests/main/whisper/
git commit -m "feat: add Whisper model download wizard with resume and progress"
```

---

### Task 13: Run full test suite and push

- [ ] **Step 1: Run all tests**

```bash
npx vitest run
```

Expected: All tests pass. Note any failures and fix before continuing.

- [ ] **Step 2: Build to verify no TypeScript errors**

```bash
npm run build
```

Expected: Build completes with no errors.

- [ ] **Step 3: Push to remote**

```bash
git push origin plans
```

Expected: Branch pushed to GitHub.

## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | `/plan-ceo-review` | Scope & strategy | 1 | resolved | 6 proposals, 6 accepted, 4 deferred; 8 critical gaps identified |
| Eng Review | `/plan-eng-review` | Architecture & tests (required) | 1 | resolved | 3 bugs fixed; Task 1b (Vitest config) + Task 11 (Whisper wizard) added |
| Codex Review | `/codex review` | Independent 2nd opinion | 0 | — | — |
| Design Review | `/plan-design-review` | UI/UX gaps | 0 | — | — |

**FIXES APPLIED:**
- Bug: `require('path')` replaced with ES import in settingsStore
- Bug: `metadata: {}` added to Campaign model (CEO cherry-pick)
- Added: Task 1b — Vitest config + Electron stub (blocked all tests without this)
- Added: Task 11 — Whisper model download wizard with progress, resume, cancel, and ModelSetup UI

**VERDICT:** ENG REVIEW COMPLETE — plan ready for implementation.
