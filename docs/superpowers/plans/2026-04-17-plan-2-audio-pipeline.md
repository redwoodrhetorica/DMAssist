# DM Assistant — Plan 2: Audio Pipeline

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire local microphone capture → Whisper transcription → Claude classification → live event log written to SQLite, with the live transcript updating in the renderer UI in real time.

**Architecture:** Three processing stages (capture, transcription, classification) connected through the EventBus from Plan 1. Each stage is a class with a clear interface. The main process runs all three; the renderer receives classified events via IPC push (ipcMain.webContents.send) and displays them live. Whisper runs locally via whisper-node. Classification calls the Claude API with the character registry and recent event history as context.

**Tech Stack:** naudiodon (PortAudio), whisper-node, @anthropic-ai/sdk, better-sqlite3, Electron IPC

**Prerequisite:** Plan 1 complete. EventBus, SessionStore, CampaignStore, SettingsStore, and IPC bridge all in place.

---

## File Map

```
src/
  main/
    plugins/
      inputPlugin.ts          # InputPlugin interface
      localMicPlugin.ts       # naudiodon mic capture, VAD, emits RawEvent
      # Note: DiscordVoicePlugin and DiscordTextPlugin are implemented in Plan 4 (Advanced/Discord)
    pipeline/
      transcriptionService.ts # Whisper wrapper, RawEvent → TranscribedEvent
      classificationEngine.ts # Claude API, TranscribedEvent → ClassifiedEvent
      sessionPipeline.ts      # Wires plugins + pipeline + store for a session
    ipc/
      handlers.ts             # Extended with session:start, session:stop, session:events
  renderer/
    pages/
      Home.tsx                # Updated: Start/Stop session, live transcript display
    components/
      TranscriptEntry.tsx     # Single classified event row
      LiveTranscript.tsx      # Scrolling list of TranscriptEntry

tests/
  main/
    plugins/
      localMicPlugin.test.ts
    pipeline/
      transcriptionService.test.ts
      classificationEngine.test.ts
      sessionPipeline.test.ts
```

---

### Task 1: InputPlugin interface

**Files:**
- Create: `src/main/plugins/inputPlugin.ts`

- [ ] **Step 1: Define the interface**

Create `src/main/plugins/inputPlugin.ts`:

```ts
import { EventBus } from '../eventBus'

export interface InputPlugin {
  readonly name: string
  start(bus: EventBus): Promise<void>
  stop(): Promise<void>
}
```

- [ ] **Step 2: Commit**

```bash
git add src/main/plugins/inputPlugin.ts
git commit -m "feat: add InputPlugin interface"
```

---

### Task 2: LocalMicPlugin

**Files:**
- Create: `src/main/plugins/localMicPlugin.ts`
- Create: `tests/main/plugins/localMicPlugin.test.ts`

- [ ] **Step 1: Install naudiodon**

```bash
npm install naudiodon
npm install -D @types/naudiodon
```

- [ ] **Step 2: Write the failing tests**

Create `tests/main/plugins/localMicPlugin.test.ts`:

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { EventBus } from '../../../src/main/eventBus'

vi.mock('naudiodon', () => ({
  default: {
    AudioIO: vi.fn().mockImplementation(() => ({
      on: vi.fn(),
      start: vi.fn(),
      quit: vi.fn()
    })),
    SampleFormat16Bit: 16
  }
}))

import { LocalMicPlugin } from '../../../src/main/plugins/localMicPlugin'

describe('LocalMicPlugin', () => {
  let bus: EventBus
  let plugin: LocalMicPlugin

  beforeEach(() => {
    bus = new EventBus()
    plugin = new LocalMicPlugin({ label: 'test-mic', deviceId: 0 })
  })

  it('starts without throwing', async () => {
    await expect(plugin.start(bus)).resolves.not.toThrow()
  })

  it('stops without throwing', async () => {
    await plugin.start(bus)
    await expect(plugin.stop()).resolves.not.toThrow()
  })

  it('has name "local-mic"', () => {
    expect(plugin.name).toBe('local-mic')
  })
})
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
npx vitest run tests/main/plugins/localMicPlugin.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 4: Implement LocalMicPlugin**

Create `src/main/plugins/localMicPlugin.ts`:

```ts
import { v4 as uuid } from 'uuid'
import { EventBus } from '../eventBus'
import { InputPlugin } from './inputPlugin'
import { RawEvent } from '../models/events'

export interface LocalMicConfig {
  label: string
  deviceId?: number
  silenceThresholdMs?: number
}

export class LocalMicPlugin implements InputPlugin {
  readonly name = 'local-mic'
  private audioStream: any = null
  private bus: EventBus | null = null
  private buffer: Buffer[] = []
  private silenceTimer: NodeJS.Timeout | null = null
  private readonly silenceThresholdMs: number

  constructor(private config: LocalMicConfig) {
    this.silenceThresholdMs = config.silenceThresholdMs ?? 1200
  }

  async start(bus: EventBus): Promise<void> {
    this.bus = bus
    let naudiodon: any
    try {
      naudiodon = (await import('naudiodon')).default
    } catch (err) {
      bus.emit('error', new Error('Microphone not found. Check your system settings.'))
      throw err
    }

    let audioIO: any
    try {
      audioIO = new naudiodon.AudioIO({
      inOptions: {
        channelCount: 1,
        sampleFormat: naudiodon.SampleFormat16Bit,
        sampleRate: 16000,
        deviceId: this.config.deviceId ?? -1,
        closeOnError: false
      }
      })
    } catch (err) {
      bus.emit('error', new Error('Microphone not found. Check your system settings.'))
      throw err
    }

    this.audioStream = audioIO

    this.audioStream.on('data', (chunk: Buffer) => {
      this.buffer.push(chunk)
      this.resetSilenceTimer()
    })

    this.audioStream.on('error', (err: Error) => {
      bus.emit('error', new Error('Microphone not found. Check your system settings.'))
    })

    this.audioStream.start()
  }

  async stop(): Promise<void> {
    if (this.silenceTimer) clearTimeout(this.silenceTimer)
    this.flushBuffer()
    this.audioStream?.quit()
    this.audioStream = null
  }

  private resetSilenceTimer(): void {
    if (this.silenceTimer) clearTimeout(this.silenceTimer)
    this.silenceTimer = setTimeout(() => this.flushBuffer(), this.silenceThresholdMs)
  }

  private flushBuffer(): void {
    if (this.buffer.length === 0 || !this.bus) return
    const audio = Buffer.concat(this.buffer)
    this.buffer = []

    const event: RawEvent = {
      id: uuid(),
      timestamp: Date.now(),
      source: 'local-mic',
      micLabel: this.config.label,
      audioBuffer: audio
    }
    this.bus.emit('raw', event)
  }
}
```

- [ ] **Step 5: Install uuid**

```bash
npm install uuid
npm install -D @types/uuid
```

- [ ] **Step 6: Run tests to verify they pass**

```bash
npx vitest run tests/main/plugins/localMicPlugin.test.ts
```

Expected: PASS — 3 tests passing.

- [ ] **Step 7: Commit**

```bash
git add src/main/plugins/localMicPlugin.ts tests/main/plugins/localMicPlugin.test.ts
git commit -m "feat: add local microphone input plugin"
```

---

### Task 3: TranscriptionService

**Files:**
- Create: `src/main/pipeline/transcriptionService.ts`
- Create: `tests/main/pipeline/transcriptionService.test.ts`

- [ ] **Step 1: Install whisper-node**

```bash
npm install whisper-node
```

- [ ] **Step 2: Write the failing tests**

Create `tests/main/pipeline/transcriptionService.test.ts`:

```ts
import { describe, it, expect, vi } from 'vitest'

vi.mock('whisper-node', () => ({
  whisper: vi.fn().mockResolvedValue([{ speech: 'I draw my sword.' }])
}))

import { TranscriptionService } from '../../../src/main/pipeline/transcriptionService'
import { RawEvent } from '../../../src/main/models/events'

const makeAudioEvent = (text?: string): RawEvent => ({
  id: 'e1',
  timestamp: 1000,
  source: 'local-mic',
  audioBuffer: Buffer.from('fake-audio'),
  text
})

describe('TranscriptionService', () => {
  it('transcribes audio buffer to text', async () => {
    const service = new TranscriptionService({ modelPath: '/mock/model.bin' })
    const result = await service.transcribe(makeAudioEvent())
    expect(result.text).toBe('I draw my sword.')
  })

  it('passes through events that already have text', async () => {
    const service = new TranscriptionService({ modelPath: '/mock/model.bin' })
    const event = makeAudioEvent('already transcribed')
    const result = await service.transcribe(event)
    expect(result.text).toBe('already transcribed')
  })

  it('preserves all original event fields', async () => {
    const service = new TranscriptionService({ modelPath: '/mock/model.bin' })
    const event = makeAudioEvent()
    const result = await service.transcribe(event)
    expect(result.id).toBe('e1')
    expect(result.source).toBe('local-mic')
    expect(result.timestamp).toBe(1000)
  })
})
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
npx vitest run tests/main/pipeline/transcriptionService.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 4: Implement TranscriptionService**

Create `src/main/pipeline/transcriptionService.ts`:

```ts
import { RawEvent, TranscribedEvent } from '../models/events'
import os from 'os'
import path from 'path'
import fs from 'fs'

export interface TranscriptionConfig {
  modelPath: string
}

export class TranscriptionService {
  constructor(private config: TranscriptionConfig) {}

  async transcribe(event: RawEvent): Promise<TranscribedEvent> {
    if (event.text) {
      return { ...event, text: event.text }
    }

    if (!event.audioBuffer) {
      return { ...event, text: '' }
    }

    const { whisper } = await import('whisper-node')
    const tmpPath = path.join(os.tmpdir(), `dmassist-${event.id}.wav`)

    try {
      fs.writeFileSync(tmpPath, event.audioBuffer)
      const result = await whisper(tmpPath, { modelName: this.config.modelPath })
      const text = result.map((r: { speech: string }) => r.speech).join(' ').trim()
      return { ...event, text }
    } finally {
      if (fs.existsSync(tmpPath)) fs.unlinkSync(tmpPath)
    }
  }
}
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
npx vitest run tests/main/pipeline/transcriptionService.test.ts
```

Expected: PASS — 3 tests passing.

- [ ] **Step 6: Commit**

```bash
git add src/main/pipeline/transcriptionService.ts tests/main/pipeline/transcriptionService.test.ts
git commit -m "feat: add whisper transcription service"
```

---

### Task 4: ClassificationEngine

**Files:**
- Create: `src/main/pipeline/classificationEngine.ts`
- Create: `tests/main/pipeline/classificationEngine.test.ts`

- [ ] **Step 1: Install Anthropic SDK**

```bash
npm install @anthropic-ai/sdk
```

- [ ] **Step 2: Write the failing tests**

Create `tests/main/pipeline/classificationEngine.test.ts`:

```ts
import { describe, it, expect, vi } from 'vitest'

vi.mock('@anthropic-ai/sdk', () => ({
  default: vi.fn().mockImplementation(() => ({
    messages: {
      create: vi.fn().mockResolvedValue({
        content: [{
          type: 'text',
          text: JSON.stringify({
            eventType: 'action',
            characterId: 'ch1',
            playerName: null,
            inGame: true,
            confidence: 0.93
          })
        }]
      })
    }
  }))
}))

import { ClassificationEngine } from '../../../src/main/pipeline/classificationEngine'
import { TranscribedEvent } from '../../../src/main/models/events'
import { Character } from '../../../src/main/models/campaign'

const characters: Character[] = [{
  id: 'ch1', name: 'Theron', playerName: 'Jake',
  characterClass: 'Paladin', race: 'Human',
  description: 'Lawful good', aliases: ['Theron'], archived: false
}]

const makeTranscribed = (): TranscribedEvent => ({
  id: 'e1', timestamp: 1000, source: 'local-mic', text: 'I draw my sword.'
})

describe('ClassificationEngine', () => {
  it('classifies an event with correct type', async () => {
    const engine = new ClassificationEngine({ apiKey: 'test', characters })
    const result = await engine.classify(makeTranscribed(), [])
    expect(result.eventType).toBe('action')
    expect(result.inGame).toBe(true)
  })

  it('sets flagged=true when confidence is below 0.7', async () => {
    const engine = new ClassificationEngine({ apiKey: 'test', characters })
    vi.mocked(require('@anthropic-ai/sdk').default).mockImplementationOnce(() => ({
      messages: {
        create: vi.fn().mockResolvedValue({
          content: [{ type: 'text', text: JSON.stringify({
            eventType: 'dialogue', characterId: null,
            playerName: null, inGame: true, confidence: 0.5
          })}]
        })
      }
    }))
    const result = await engine.classify(makeTranscribed(), [])
    expect(result.flagged).toBe(true)
  })

  it('preserves original event fields', async () => {
    const engine = new ClassificationEngine({ apiKey: 'test', characters })
    const result = await engine.classify(makeTranscribed(), [])
    expect(result.id).toBe('e1')
    expect(result.text).toBe('I draw my sword.')
  })
})
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
npx vitest run tests/main/pipeline/classificationEngine.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 4: Implement ClassificationEngine**

Create `src/main/pipeline/classificationEngine.ts`:

```ts
import Anthropic from '@anthropic-ai/sdk'
import { TranscribedEvent, ClassifiedEvent } from '../models/events'
import { Character } from '../models/campaign'
import { SessionEvent } from '../models/session'

const CONFIDENCE_THRESHOLD = 0.7
const MODEL = 'claude-sonnet-4-6'

export interface ClassificationConfig {
  apiKey: string
  characters: Character[]
}

interface ClassificationResult {
  eventType: ClassifiedEvent['eventType']
  characterId: string | null
  playerName: string | null
  inGame: boolean
  confidence: number
}

export class ClassificationEngine {
  private client: Anthropic

  constructor(private config: ClassificationConfig) {
    this.client = new Anthropic({ apiKey: config.apiKey })
  }

  updateCharacters(characters: Character[]): void {
    this.config.characters = characters
  }

  async classify(
    event: TranscribedEvent,
    recentHistory: SessionEvent[]
  ): Promise<ClassifiedEvent> {
    const characterList = this.config.characters
      .filter(c => !c.archived)
      .map(c => `- ${c.name} (played by ${c.playerName}, ${c.characterClass} ${c.race}): ${c.description}. Aliases: ${c.aliases.join(', ')}`)
      .join('\n')

    const historyText = recentHistory.slice(-10)
      .map(e => `[${e.eventType}] ${e.speaker}: ${e.rawText}`)
      .join('\n')

    const prompt = `You are classifying D&D session events. Determine if this utterance is in-game or out-of-character, identify the event type, and attribute it to a character if possible.

CHARACTERS IN THIS CAMPAIGN:
${characterList || 'No characters registered yet.'}

RECENT SESSION HISTORY:
${historyText || 'No prior events.'}

CURRENT UTTERANCE:
"${event.text}"
${event.speakerDiscordId ? `Speaker Discord ID: ${event.speakerDiscordId}` : ''}
${event.micLabel ? `Mic label: ${event.micLabel}` : ''}
${event.text.startsWith('//ooc') ? 'NOTE: Speaker used //ooc prefix indicating out-of-character.' : ''}

Respond with ONLY a JSON object:
{
  "eventType": "dialogue" | "action" | "combat" | "ooc" | "meta",
  "characterId": "<character id from list above, or null>",
  "playerName": "<player name if OOC, or null>",
  "inGame": true | false,
  "confidence": <0.0 to 1.0>
}`

    try {
      const response = await this.client.messages.create({
        model: MODEL,
        max_tokens: 200,
        messages: [{ role: 'user', content: prompt }]
      })

      const text = response.content[0].type === 'text' ? response.content[0].text : '{}'
      const parsed: ClassificationResult = JSON.parse(text)

      return {
        ...event,
        eventType: parsed.eventType,
        characterId: parsed.characterId ?? undefined,
        playerName: parsed.playerName ?? undefined,
        inGame: parsed.inGame,
        confidence: parsed.confidence,
        flagged: parsed.confidence < CONFIDENCE_THRESHOLD
      }
    } catch {
      return {
        ...event,
        eventType: 'unclassified',
        inGame: false,
        confidence: 0,
        flagged: true
      }
    }
  }
}
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
npx vitest run tests/main/pipeline/classificationEngine.test.ts
```

Expected: PASS — 3 tests passing.

- [ ] **Step 6: Commit**

```bash
git add src/main/pipeline/classificationEngine.ts tests/main/pipeline/classificationEngine.test.ts
git commit -m "feat: add Claude classification engine"
```

---

### Task 5: SessionPipeline — wire it all together

**Files:**
- Create: `src/main/pipeline/sessionPipeline.ts`
- Create: `tests/main/pipeline/sessionPipeline.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `tests/main/pipeline/sessionPipeline.test.ts`:

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { EventBus } from '../../../src/main/eventBus'
import { RawEvent, ClassifiedEvent } from '../../../src/main/models/events'

const mockTranscribe = vi.fn()
const mockClassify = vi.fn()
const mockInsertEvent = vi.fn()
const mockGetEvents = vi.fn().mockReturnValue([])
const mockPlugin = { name: 'local-mic', start: vi.fn(), stop: vi.fn() }

vi.mock('../../../src/main/pipeline/transcriptionService', () => ({
  TranscriptionService: vi.fn().mockImplementation(() => ({ transcribe: mockTranscribe }))
}))
vi.mock('../../../src/main/pipeline/classificationEngine', () => ({
  ClassificationEngine: vi.fn().mockImplementation(() => ({ classify: mockClassify, updateCharacters: vi.fn() }))
}))
vi.mock('../../../src/main/storage/sessionStore', () => ({
  SessionStore: vi.fn().mockImplementation(() => ({
    createSession: vi.fn(),
    insertEvent: mockInsertEvent,
    getEvents: mockGetEvents,
    endSession: vi.fn(),
    close: vi.fn()
  }))
}))

import { SessionPipeline } from '../../../src/main/pipeline/sessionPipeline'

describe('SessionPipeline', () => {
  let bus: EventBus
  let pipeline: SessionPipeline

  beforeEach(() => {
    vi.clearAllMocks()
    bus = new EventBus()
    pipeline = new SessionPipeline({
      bus,
      dbPath: ':memory:',
      modelPath: '/mock/model.bin',
      apiKey: 'test-key',
      campaignId: 'c1',
      characters: []
    })
  })

  it('processes a raw event through the full pipeline', async () => {
    const transcribed = { id: 'e1', timestamp: 1000, source: 'local-mic' as const, text: 'I attack' }
    const classified: ClassifiedEvent = {
      ...transcribed, eventType: 'combat', inGame: true, confidence: 0.9, flagged: false
    }
    mockTranscribe.mockResolvedValue(transcribed)
    mockClassify.mockResolvedValue(classified)

    await pipeline.start('s1', [mockPlugin as any])

    const rawEvent: RawEvent = { id: 'e1', timestamp: 1000, source: 'local-mic', audioBuffer: Buffer.from('audio') }
    bus.emit('raw', rawEvent)

    await new Promise(r => setTimeout(r, 50))

    expect(mockTranscribe).toHaveBeenCalledWith(rawEvent)
    expect(mockClassify).toHaveBeenCalledWith(transcribed, [])
    expect(mockInsertEvent).toHaveBeenCalled()
  })

  it('stops all plugins on stop()', async () => {
    await pipeline.start('s1', [mockPlugin as any])
    await pipeline.stop()
    expect(mockPlugin.stop).toHaveBeenCalled()
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npx vitest run tests/main/pipeline/sessionPipeline.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement SessionPipeline**

Create `src/main/pipeline/sessionPipeline.ts`:

```ts
import { v4 as uuid } from 'uuid'
import { EventBus } from '../eventBus'
import { InputPlugin } from '../plugins/inputPlugin'
import { TranscriptionService } from './transcriptionService'
import { ClassificationEngine } from './classificationEngine'
import { SessionStore } from '../storage/sessionStore'
import { Character } from '../models/campaign'
import { RawEvent, ClassifiedEvent } from '../models/events'
import { SessionEvent } from '../models/session'

export interface SessionPipelineConfig {
  bus: EventBus
  dbPath: string
  modelPath: string
  apiKey: string
  campaignId: string
  characters: Character[]
}

export class SessionPipeline {
  private transcription: TranscriptionService
  private classification: ClassificationEngine
  private store: SessionStore
  private plugins: InputPlugin[] = []
  private sessionId: string | null = null
  private onRaw: ((event: RawEvent) => void) | null = null
  onClassified: ((event: ClassifiedEvent) => void) | null = null

  constructor(private config: SessionPipelineConfig) {
    this.transcription = new TranscriptionService({ modelPath: config.modelPath })
    this.classification = new ClassificationEngine({ apiKey: config.apiKey, characters: config.characters })
    this.store = new SessionStore(config.dbPath)
  }

  async start(sessionId: string, plugins: InputPlugin[]): Promise<void> {
    this.sessionId = sessionId
    this.plugins = plugins

    this.store.createSession({ id: sessionId, campaignId: this.config.campaignId, startedAt: Date.now() })

    this.onRaw = async (event: RawEvent) => {
      try {
        const transcribed = await this.transcription.transcribe(event)
        const history = this.store.getEvents(sessionId)
        const classified = await this.classification.classify(transcribed, history)

        const sessionEvent: SessionEvent = {
          id: uuid(),
          sessionId,
          timestamp: classified.timestamp,
          speaker: classified.characterId
            ? (this.config.characters.find(c => c.id === classified.characterId)?.name ?? 'Unknown')
            : (classified.playerName ?? 'Unknown'),
          rawText: classified.text,
          eventType: classified.eventType,
          characterId: classified.characterId,
          inGame: classified.inGame,
          confidence: classified.confidence,
          flagged: classified.flagged
        }

        this.store.insertEvent(sessionEvent)
        this.onClassified?.(classified)
      } catch (err) {
        this.config.bus.emit('error', err as Error)
      }
    }

    this.config.bus.on('raw', this.onRaw)

    for (const plugin of plugins) {
      await plugin.start(this.config.bus)
    }
  }

  async stop(): Promise<void> {
    for (const plugin of this.plugins) {
      await plugin.stop()
    }
    if (this.onRaw) {
      this.config.bus.off('raw', this.onRaw)
      this.onRaw = null
    }
    if (this.sessionId) {
      this.store.endSession(this.sessionId, Date.now())
    }
  }

  close(): void {
    this.store.close()
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npx vitest run tests/main/pipeline/sessionPipeline.test.ts
```

Expected: PASS — 2 tests passing.

- [ ] **Step 5: Commit**

```bash
git add src/main/pipeline/sessionPipeline.ts tests/main/pipeline/sessionPipeline.test.ts
git commit -m "feat: add session pipeline wiring capture → transcription → classification → store"
```

---

### Task 6: Extend IPC for session control

**Files:**
- Modify: `src/main/ipc/handlers.ts`
- Modify: `src/preload/index.ts`
- Modify: `src/renderer/lib/ipc.ts`

- [ ] **Step 1: Add session IPC handlers to handlers.ts**

Add the following to `src/main/ipc/handlers.ts` (extend the `registerHandlers` function signature and body):

```ts
import { ipcMain, BrowserWindow } from 'electron'
import { CampaignStore } from '../storage/campaignStore'
import { SettingsStore } from '../storage/settingsStore'
import { SessionPipeline } from '../pipeline/sessionPipeline'
import { LocalMicPlugin } from '../plugins/localMicPlugin'
import { ModelManager } from '../whisper/modelManager'
import { Campaign, Character } from '../models/campaign'
import { AppSettings } from '../models/settings'
import { getSessionPath } from '../storage/paths'
import { v4 as uuid } from 'uuid'
import path from 'path'

let activePipeline: SessionPipeline | null = null

export function registerHandlers(
  campaignStore: CampaignStore,
  settingsStore: SettingsStore,
  modelManager: ModelManager,
  getWindow: () => BrowserWindow | null
): void {
  // --- existing handlers (settings, campaign, character) remain unchanged ---
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

  // --- session handlers ---
  ipcMain.handle('session:start', async (_e, campaignId: string) => {
    if (activePipeline) await activePipeline.stop()

    const settings = settingsStore.load()
    const apiKey = await settingsStore.getSecret('claude-api-key')
    if (!apiKey) throw new Error('Claude API key not configured. Add it in Settings.')

    // Fail fast before starting mic — model must be present
    if (!modelManager.isModelPresent(settings.transcriptionQuality)) {
      throw new Error('Whisper model not downloaded. Complete setup before starting a session.')
    }
    const modelPath = modelManager.getModelPath(settings.transcriptionQuality)

    const campaign = campaignStore.loadCampaign(campaignId)
    if (!campaign) throw new Error(`Campaign ${campaignId} not found`)

    const sessionId = uuid()
    const dbPath = getSessionPath(campaignId, sessionId)

    const { EventBus } = await import('../eventBus')
    const bus = new EventBus()

    // Wire error events to renderer so silent crashes surface in the UI
    bus.on('error', (err: Error) => {
      getWindow()?.webContents.send('session:error', err.message)
    })

    activePipeline = new SessionPipeline({
      bus,
      dbPath,
      modelPath,
      apiKey,
      campaignId,
      characters: campaign.characters
    })

    activePipeline.onClassified = (event) => {
      getWindow()?.webContents.send('session:event', event)
    }

    const micPlugin = new LocalMicPlugin({ label: 'default' })
    await activePipeline.start(sessionId, [micPlugin])

    return sessionId
  })

  ipcMain.handle('session:stop', async () => {
    if (!activePipeline) return
    await activePipeline.stop()
    activePipeline.close()
    activePipeline = null
  })
}
```

- [ ] **Step 2: Add session channels to preload**

Add to the `api` object in `src/preload/index.ts`:

```ts
session: {
  start: (campaignId: string) => ipcRenderer.invoke('session:start', campaignId),
  stop: () => ipcRenderer.invoke('session:stop'),
  onEvent: (cb: (event: unknown) => void) => {
    ipcRenderer.on('session:event', (_e, event) => cb(event))
    return () => ipcRenderer.removeAllListeners('session:event')
  },
  onError: (cb: (message: string) => void) => {
    ipcRenderer.on('session:error', (_e, msg) => cb(msg))
    return () => ipcRenderer.removeAllListeners('session:error')
  }
}
```

- [ ] **Step 3: Add session methods to renderer IPC wrapper**

Add to `src/renderer/lib/ipc.ts`:

```ts
import { ClassifiedEvent } from '../../main/models/events'

// Add to ipc object:
session: {
  start: (campaignId: string): Promise<string> => api.session.start(campaignId),
  stop: (): Promise<void> => api.session.stop(),
  onEvent: (cb: (event: ClassifiedEvent) => void): (() => void) =>
    api.session.onEvent(cb),
  onError: (cb: (message: string) => void): (() => void) =>
    api.session.onError(cb)
}
```

- [ ] **Step 4: Commit**

```bash
git add src/main/ipc/handlers.ts src/preload/index.ts src/renderer/lib/ipc.ts
git commit -m "feat: add session start/stop IPC and classified event push"
```

---

### Task 7: Live transcript UI

**Files:**
- Create: `src/renderer/components/TranscriptEntry.tsx`
- Create: `src/renderer/components/LiveTranscript.tsx`
- Modify: `src/renderer/pages/Home.tsx`

- [ ] **Step 1: Add shadcn components**

```bash
npx shadcn@latest add badge scroll-area
```

- [ ] **Step 2: Create TranscriptEntry**

Create `src/renderer/components/TranscriptEntry.tsx`:

```tsx
import { Badge } from '@/components/ui/badge'
import { ClassifiedEvent } from '../../main/models/events'
import { cn } from '@/lib/utils'

interface Props {
  event: ClassifiedEvent
}

const typeColors: Record<string, string> = {
  dialogue: 'bg-blue-500/10 text-blue-400 border-blue-500/20',
  action: 'bg-green-500/10 text-green-400 border-green-500/20',
  combat: 'bg-red-500/10 text-red-400 border-red-500/20',
  ooc: 'bg-muted text-muted-foreground border-muted',
  meta: 'bg-yellow-500/10 text-yellow-400 border-yellow-500/20',
  unclassified: 'bg-muted text-muted-foreground border-muted'
}

export default function TranscriptEntry({ event }: Props) {
  const time = new Date(event.timestamp).toLocaleTimeString()
  const speaker = event.characterId ? event.text : (event.playerName ?? 'Unknown')

  return (
    <div className={cn('flex gap-3 py-2 px-3 rounded-md text-sm', event.flagged && 'opacity-60 border border-dashed border-muted')}>
      <span className="text-muted-foreground w-16 shrink-0">{time}</span>
      <Badge variant="outline" className={cn('shrink-0 text-xs', typeColors[event.eventType])}>
        {event.eventType}
      </Badge>
      <span className="font-medium shrink-0">{speaker}</span>
      <span className="text-muted-foreground">{event.text}</span>
    </div>
  )
}
```

- [ ] **Step 3: Create LiveTranscript**

Create `src/renderer/components/LiveTranscript.tsx`:

```tsx
import { useEffect, useRef } from 'react'
import { ScrollArea } from '@/components/ui/scroll-area'
import TranscriptEntry from './TranscriptEntry'
import { ClassifiedEvent } from '../../main/models/events'

interface Props {
  events: ClassifiedEvent[]
}

export default function LiveTranscript({ events }: Props) {
  const bottomRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: 'smooth' })
  }, [events.length])

  if (events.length === 0) {
    return (
      <div className="flex h-full items-center justify-center text-muted-foreground text-sm">
        Session events will appear here
      </div>
    )
  }

  return (
    <ScrollArea className="h-full">
      <div className="flex flex-col gap-1 p-3">
        {events.map(event => (
          <TranscriptEntry key={event.id} event={event} />
        ))}
        <div ref={bottomRef} />
      </div>
    </ScrollArea>
  )
}
```

- [ ] **Step 4: Update Home page with session control**

Replace `src/renderer/pages/Home.tsx`:

```tsx
import { useState, useEffect, useCallback } from 'react'
import { Button } from '@/components/ui/button'
import { Mic, MicOff } from 'lucide-react'
import LiveTranscript from '../components/LiveTranscript'
import { ipc } from '../lib/ipc'
import { ClassifiedEvent } from '../../main/models/events'

export default function Home() {
  const [sessionActive, setSessionActive] = useState(false)
  const [events, setEvents] = useState<ClassifiedEvent[]>([])
  const [sessionId, setSessionId] = useState<string | null>(null)

  useEffect(() => {
    const cleanup = ipc.session.onEvent((event) => {
      setEvents(prev => [...prev, event])
    })
    return cleanup
  }, [])

  const handleStart = useCallback(async () => {
    try {
      const settings = await ipc.settings.load()
      if (!settings.activeCampaignId) {
        alert('Please select a campaign in Settings first.')
        return
      }
      const id = await ipc.session.start(settings.activeCampaignId)
      setSessionId(id)
      setEvents([])
      setSessionActive(true)
    } catch (err) {
      alert(`Failed to start session: ${(err as Error).message}`)
    }
  }, [])

  const handleStop = useCallback(async () => {
    await ipc.session.stop()
    setSessionActive(false)
  }, [])

  return (
    <div className="flex flex-col h-full p-6 gap-4">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">Session</h1>
        <Button
          size="lg"
          variant={sessionActive ? 'destructive' : 'default'}
          onClick={sessionActive ? handleStop : handleStart}
          className="gap-2"
        >
          {sessionActive ? <><MicOff className="h-4 w-4" /> Stop Session</> : <><Mic className="h-4 w-4" /> Start Session</>}
        </Button>
      </div>
      <div className="flex-1 rounded-lg border bg-muted/20 overflow-hidden">
        <LiveTranscript events={events} />
      </div>
    </div>
  )
}
```

- [ ] **Step 5: Run dev and verify live transcript updates**

```bash
npm run dev
```

Expected: Home page shows Start Session button. Clicking it starts the mic and classified events appear in the scrolling transcript.

- [ ] **Step 6: Commit**

```bash
git add src/renderer/components/ src/renderer/pages/Home.tsx
git commit -m "feat: add live transcript UI with start/stop session control"
```

---

### Task 8: Two-stage live transcript + error display (CEO cherry-picks)

**Files:**
- Modify: `src/renderer/components/TranscriptEntry.tsx` — add PENDING state
- Modify: `src/renderer/components/LiveTranscript.tsx` — manage pending entries
- Modify: `src/renderer/pages/Home.tsx` — wire error channel

Two-stage state machine:
```
audio captured → Whisper transcribes → show as PENDING (gray, spinner icon)
     ↓
Claude classifies → update in-place to CLASSIFIED (color-coded by type)
     ↓ (if classify fails OR >5s elapsed after Whisper completes)
show as UNCLASSIFIED (yellow, flagged) — never disappears
```

PENDING entries are NOT persisted to SQLite. If the app crashes between Whisper completing and Claude responding, those entries are lost. This is acceptable for v1.

SLA: 3s measured from when Whisper completes. If >5s elapsed, mark UNCLASSIFIED.

- [ ] **Step 1: Add a `session:transcribed` IPC push from SessionPipeline**

In `src/main/pipeline/sessionPipeline.ts`, add a callback for transcribed events (before classification):

```ts
// Add to SessionPipelineConfig:
onTranscribed?: (event: TranscribedEvent) => void
onClassified?: (event: ClassifiedEvent) => void

// In the raw event handler, after transcription succeeds, before classify():
this.config.onTranscribed?.(transcribed)
```

In `src/main/ipc/handlers.ts`, wire the new callback:

```ts
activePipeline.config.onTranscribed = (event) => {
  getWindow()?.webContents.send('session:transcribed', event)
}
activePipeline.onClassified = (event) => {
  getWindow()?.webContents.send('session:event', event)
}
```

Add `session:transcribed` to preload:

```ts
onTranscribed: (cb: (event: unknown) => void) => {
  ipcRenderer.on('session:transcribed', (_e, event) => cb(event))
  return () => ipcRenderer.removeAllListeners('session:transcribed')
}
```

Add to renderer IPC wrapper:

```ts
import { TranscribedEvent } from '../../main/models/events'

onTranscribed: (cb: (event: TranscribedEvent) => void): (() => void) =>
  api.session.onTranscribed(cb)
```

- [ ] **Step 2: Add pending entry type to TranscriptEntry**

Update `src/renderer/components/TranscriptEntry.tsx`:

```tsx
import { Badge } from '@/components/ui/badge'
import { Loader2 } from 'lucide-react'
import { ClassifiedEvent, TranscribedEvent } from '../../main/models/events'
import { cn } from '@/lib/utils'

export type PendingEntry = { id: string; timestamp: number; text: string; status: 'pending' | 'unclassified' }
export type TranscriptItem = ClassifiedEvent | PendingEntry

function isPending(item: TranscriptItem): item is PendingEntry {
  return 'status' in item
}

interface Props { item: TranscriptItem }

const typeColors: Record<string, string> = {
  dialogue: 'bg-blue-500/10 text-blue-400 border-blue-500/20',
  action: 'bg-green-500/10 text-green-400 border-green-500/20',
  combat: 'bg-red-500/10 text-red-400 border-red-500/20',
  ooc: 'bg-muted text-muted-foreground border-muted',
  meta: 'bg-yellow-500/10 text-yellow-400 border-yellow-500/20',
  unclassified: 'bg-yellow-500/10 text-yellow-400 border-yellow-500/20'
}

export default function TranscriptEntry({ item }: Props) {
  const time = new Date(item.timestamp).toLocaleTimeString()

  if (isPending(item)) {
    return (
      <div className={cn(
        'flex gap-3 py-2 px-3 rounded-md text-sm',
        item.status === 'unclassified' ? 'border border-yellow-500/30' : 'opacity-50'
      )}>
        <span className="text-muted-foreground w-16 shrink-0">{time}</span>
        {item.status === 'pending'
          ? <Loader2 className="h-4 w-4 animate-spin shrink-0 mt-0.5 text-muted-foreground" />
          : <Badge variant="outline" className={typeColors.unclassified}>unclassified</Badge>
        }
        <span className="text-muted-foreground">{item.text}</span>
      </div>
    )
  }

  const speaker = item.characterId ? item.text : (item.playerName ?? 'Unknown')
  return (
    <div className={cn('flex gap-3 py-2 px-3 rounded-md text-sm', item.flagged && 'opacity-60 border border-dashed border-muted')}>
      <span className="text-muted-foreground w-16 shrink-0">{time}</span>
      <Badge variant="outline" className={cn('shrink-0 text-xs', typeColors[item.eventType])}>
        {item.eventType}
      </Badge>
      <span className="font-medium shrink-0">{speaker}</span>
      <span className="text-muted-foreground">{item.text}</span>
    </div>
  )
}
```

- [ ] **Step 3: Update LiveTranscript to manage pending→classified transitions**

Replace `src/renderer/components/LiveTranscript.tsx`:

```tsx
import { useEffect, useRef } from 'react'
import { ScrollArea } from '@/components/ui/scroll-area'
import TranscriptEntry, { TranscriptItem } from './TranscriptEntry'

interface Props { items: TranscriptItem[] }

export default function LiveTranscript({ items }: Props) {
  const bottomRef = useRef<HTMLDivElement>(null)
  useEffect(() => { bottomRef.current?.scrollIntoView({ behavior: 'smooth' }) }, [items.length])

  if (items.length === 0) {
    return (
      <div className="flex h-full items-center justify-center text-muted-foreground text-sm">
        Session events will appear here
      </div>
    )
  }

  return (
    <ScrollArea className="h-full">
      <div className="flex flex-col gap-1 p-3">
        {items.map(item => <TranscriptEntry key={item.id} item={item} />)}
        <div ref={bottomRef} />
      </div>
    </ScrollArea>
  )
}
```

- [ ] **Step 4: Update Home page to manage two-stage state and error toasts**

Replace `src/renderer/pages/Home.tsx`:

```tsx
import { useState, useEffect, useCallback, useRef } from 'react'
import { Button } from '@/components/ui/button'
import { Mic, MicOff, AlertCircle } from 'lucide-react'
import LiveTranscript from '../components/LiveTranscript'
import { TranscriptItem, PendingEntry } from '../components/TranscriptEntry'
import { ipc } from '../lib/ipc'
import { ClassifiedEvent } from '../../main/models/events'

const CLASSIFY_TIMEOUT_MS = 5000

export default function Home() {
  const [sessionActive, setSessionActive] = useState(false)
  const [items, setItems] = useState<TranscriptItem[]>([])
  const [sessionId, setSessionId] = useState<string | null>(null)
  const [campaignId, setCampaignId] = useState<string | null>(null)
  const [errorMessage, setErrorMessage] = useState<string | null>(null)
  const timeoutsRef = useRef<Map<string, NodeJS.Timeout>>(new Map())

  useEffect(() => {
    ipc.settings.load().then(s => setCampaignId(s.activeCampaignId ?? null))

    const cleanupEvent = ipc.session.onEvent((classified: ClassifiedEvent) => {
      // Clear timeout and replace pending entry with classified
      const timer = timeoutsRef.current.get(classified.id)
      if (timer) { clearTimeout(timer); timeoutsRef.current.delete(classified.id) }
      setItems(prev => prev.map(item => item.id === classified.id ? classified : item))
    })

    const cleanupTranscribed = ipc.session.onTranscribed((transcribed) => {
      const pending: PendingEntry = { id: transcribed.id, timestamp: transcribed.timestamp, text: transcribed.text, status: 'pending' }
      setItems(prev => [...prev, pending])
      // Timeout: if no classified event arrives within 5s, mark as unclassified
      const timer = setTimeout(() => {
        timeoutsRef.current.delete(transcribed.id)
        setItems(prev => prev.map(item =>
          item.id === transcribed.id && 'status' in item ? { ...item, status: 'unclassified' as const } : item
        ))
      }, CLASSIFY_TIMEOUT_MS)
      timeoutsRef.current.set(transcribed.id, timer)
    })

    const cleanupError = ipc.session.onError((msg: string) => {
      setErrorMessage(msg)
      setTimeout(() => setErrorMessage(null), 8000)
    })

    return () => { cleanupEvent(); cleanupTranscribed(); cleanupError() }
  }, [])

  const handleStart = useCallback(async () => {
    try {
      const settings = await ipc.settings.load()
      if (!settings.activeCampaignId) { setErrorMessage('Please select a campaign in Settings first.'); return }
      const id = await ipc.session.start(settings.activeCampaignId)
      setSessionId(id)
      setItems([])
      setSessionActive(true)
    } catch (err) {
      setErrorMessage((err as Error).message)
    }
  }, [])

  const handleStop = useCallback(async () => {
    timeoutsRef.current.forEach(t => clearTimeout(t))
    timeoutsRef.current.clear()
    await ipc.session.stop()
    setSessionActive(false)
  }, [])

  return (
    <div className="flex flex-col h-full p-6 gap-4">
      {errorMessage && (
        <div className="flex items-center gap-2 rounded-md border border-destructive/50 bg-destructive/10 px-4 py-2 text-sm text-destructive">
          <AlertCircle className="h-4 w-4 shrink-0" />
          {errorMessage}
        </div>
      )}
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">Session</h1>
        <Button
          size="lg"
          variant={sessionActive ? 'destructive' : 'default'}
          onClick={sessionActive ? handleStop : handleStart}
          className="gap-2"
        >
          {sessionActive ? <><MicOff className="h-4 w-4" /> Stop Session</> : <><Mic className="h-4 w-4" /> Start Session</>}
        </Button>
      </div>
      <div className="flex-1 rounded-lg border bg-muted/20 overflow-hidden">
        <LiveTranscript items={items} />
      </div>
    </div>
  )
}
```

- [ ] **Step 5: Commit**

```bash
git add src/renderer/components/TranscriptEntry.tsx src/renderer/components/LiveTranscript.tsx src/renderer/pages/Home.tsx src/main/pipeline/sessionPipeline.ts src/main/ipc/handlers.ts src/preload/index.ts src/renderer/lib/ipc.ts
git commit -m "feat: two-stage transcript (pending→classified) with error display"
```

---

### Task 9: Run full test suite and push

- [ ] **Step 1: Run all tests**

```bash
npx vitest run
```

Expected: All tests pass.

- [ ] **Step 2: Build to verify no TypeScript errors**

```bash
npm run build
```

Expected: Clean build.

- [ ] **Step 3: Push to remote**

```bash
git push origin plans
```

---

## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | `/plan-ceo-review` | Scope & strategy | 1 | resolved | Gaps applied; cherry-picks incorporated |
| Eng Review | `/plan-eng-review` | Architecture & tests (required) | 1 | resolved | 3 bugs fixed; Task 8 (two-stage transcript + error display) added |
| Codex Review | `/codex review` | Independent 2nd opinion | 0 | — | — |
| Design Review | `/plan-design-review` | UI/UX gaps | 0 | — | — |

**FIXES APPLIED:**
- Bug: `modelPath: settings.transcriptionQuality` replaced with `modelManager.getModelPath(quality)` — was passing quality enum as a file path
- Bug: `naudiodon.AudioIO` constructor wrapped in try/catch with friendly "Microphone not found" error
- Bug: `EventBus` imported statically in session:start (was dynamic `await import()`)
- Added: `bus.on('error', ...)` wired to `webContents.send('session:error', ...)` (CEO Gap 1)
- Added: `session:error` IPC channel in preload and renderer
- Added: Model presence pre-flight check in `session:start` before starting mic
- Added: Task 8 — two-stage transcript state machine (pending→classified→unclassified) with error toast UI

**VERDICT:** ENG REVIEW COMPLETE — plan ready for implementation.
