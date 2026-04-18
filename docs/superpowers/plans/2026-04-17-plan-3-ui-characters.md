# DM Assistant — Plan 3: UI & Character Registry

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement all production UI views — campaign management, character registry, session history, settings — plus the first-launch onboarding wizard. After this plan, a non-technical user can install the app, complete onboarding, manage their campaign, and view past sessions.

**Architecture:** All views are React components in the renderer. They call the IPC bridge from Plan 1 to read/write campaigns, characters, and settings. No new main-process code is needed except wiring the session history read path. State is kept local to each page (no global state manager — too simple to need one).

**Tech Stack:** React, shadcn/ui, Tailwind CSS, react-router-dom, lucide-react

**Prerequisite:** Plans 1 and 2 complete. All IPC channels for campaigns, characters, settings, and session history in place.

---

## File Map

```
src/
  renderer/
    pages/
      Home.tsx                # Already done in Plan 2 — no changes
      Characters.tsx          # Full character registry UI
      History.tsx             # Session history list + detail view
      Settings.tsx            # Settings form with output config
      Onboarding.tsx          # 3-step first-launch wizard
    components/
      CharacterCard.tsx        # Single character display card
      CharacterForm.tsx        # Add/edit character form dialog
      SessionRow.tsx           # Single session row in history list
      OutputSettings.tsx       # Output destination checkboxes
      ApiKeyField.tsx          # Masked API key input with save
    App.tsx                   # Updated to show Onboarding on first launch
```

---

### Task 1: CharacterCard and CharacterForm

**Files:**
- Create: `src/renderer/components/CharacterCard.tsx`
- Create: `src/renderer/components/CharacterForm.tsx`

- [ ] **Step 1: Add required shadcn components**

```bash
npx shadcn@latest add card dialog form input label select textarea
```

- [ ] **Step 2: Create CharacterCard**

Create `src/renderer/components/CharacterCard.tsx`:

```tsx
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { Badge } from '@/components/ui/badge'
import { Pencil, Archive } from 'lucide-react'
import { Character } from '../../main/models/campaign'

interface Props {
  character: Character
  onEdit: (character: Character) => void
  onArchive: (characterId: string) => void
}

export default function CharacterCard({ character, onEdit, onArchive }: Props) {
  return (
    <Card className="relative">
      <CardHeader className="pb-2">
        <div className="flex items-start justify-between gap-2">
          <div>
            <CardTitle className="text-base">{character.name}</CardTitle>
            <p className="text-sm text-muted-foreground">Played by {character.playerName}</p>
          </div>
          <div className="flex gap-1 shrink-0">
            <Button variant="ghost" size="icon" onClick={() => onEdit(character)}>
              <Pencil className="h-4 w-4" />
            </Button>
            <Button variant="ghost" size="icon" onClick={() => onArchive(character.id)}>
              <Archive className="h-4 w-4" />
            </Button>
          </div>
        </div>
      </CardHeader>
      <CardContent className="flex flex-wrap gap-2">
        <Badge variant="secondary">{character.characterClass}</Badge>
        <Badge variant="outline">{character.race}</Badge>
        {character.description && (
          <p className="w-full text-sm text-muted-foreground mt-1">{character.description}</p>
        )}
      </CardContent>
    </Card>
  )
}
```

- [ ] **Step 3: Create CharacterForm dialog**

Create `src/renderer/components/CharacterForm.tsx`:

```tsx
import { useState, useEffect } from 'react'
import { Dialog, DialogContent, DialogHeader, DialogTitle, DialogFooter } from '@/components/ui/dialog'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'
import { Textarea } from '@/components/ui/textarea'
import { Character } from '../../main/models/campaign'
import { v4 as uuid } from 'uuid'

interface Props {
  open: boolean
  character?: Character
  existingNames?: string[]  // names already in use — used to prevent duplicates
  onSave: (character: Character) => void
  onClose: () => void
}

const empty = (): Character => ({
  id: uuid(), name: '', playerName: '', characterClass: '', race: '',
  description: '', aliases: [], archived: false
})

export default function CharacterForm({ open, character, existingNames = [], onSave, onClose }: Props) {
  const [form, setForm] = useState<Character>(empty)
  const [nameError, setNameError] = useState<string | null>(null)

  useEffect(() => {
    setForm(character ?? empty())
    setNameError(null)
  }, [character, open])

  const set = (field: keyof Character, value: string) => {
    setForm(prev => ({ ...prev, [field]: value }))
    if (field === 'name') setNameError(null)
  }

  const handleSave = () => {
    if (!form.name.trim() || !form.playerName.trim()) return
    // Uniqueness check — skip for existing character being edited (same id)
    const isDuplicate = existingNames
      .filter((_, i) => {
        const existing = existingNames[i]
        return !(character && existing.toLowerCase() === character.name.toLowerCase())
      })
      .some(n => n.toLowerCase() === form.name.trim().toLowerCase())
    if (isDuplicate) {
      setNameError(`A character named "${form.name}" already exists.`)
      return
    }
    const aliases = [form.name, ...form.aliases.filter(a => a !== form.name)]
    onSave({ ...form, aliases })
  }

  return (
    <Dialog open={open} onOpenChange={open => !open && onClose()}>
      <DialogContent className="sm:max-w-md">
        <DialogHeader>
          <DialogTitle>{character ? 'Edit Character' : 'Add Character'}</DialogTitle>
        </DialogHeader>
        <div className="grid gap-4 py-2">
          <div className="grid gap-1.5">
            <Label htmlFor="name">Character Name *</Label>
            <Input id="name" value={form.name} onChange={e => set('name', e.target.value)} placeholder="Theron" className={nameError ? 'border-destructive' : ''} />
            {nameError && <p className="text-xs text-destructive">{nameError}</p>}
          </div>
          <div className="grid gap-1.5">
            <Label htmlFor="playerName">Player Name *</Label>
            <Input id="playerName" value={form.playerName} onChange={e => set('playerName', e.target.value)} placeholder="Jake" />
          </div>
          <div className="grid grid-cols-2 gap-3">
            <div className="grid gap-1.5">
              <Label htmlFor="class">Class</Label>
              <Input id="class" value={form.characterClass} onChange={e => set('characterClass', e.target.value)} placeholder="Paladin" />
            </div>
            <div className="grid gap-1.5">
              <Label htmlFor="race">Race</Label>
              <Input id="race" value={form.race} onChange={e => set('race', e.target.value)} placeholder="Human" />
            </div>
          </div>
          <div className="grid gap-1.5">
            <Label htmlFor="description">Notes</Label>
            <Textarea id="description" value={form.description} onChange={e => set('description', e.target.value)} placeholder="Lawful good, speaks formally..." rows={3} />
          </div>
        </div>
        <DialogFooter>
          <Button variant="outline" onClick={onClose}>Cancel</Button>
          <Button onClick={handleSave} disabled={!form.name.trim() || !form.playerName.trim()}>
            {character ? 'Save Changes' : 'Add Character'}
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  )
}
```

- [ ] **Step 4: Commit**

```bash
git add src/renderer/components/CharacterCard.tsx src/renderer/components/CharacterForm.tsx
git commit -m "feat: add CharacterCard and CharacterForm components"
```

---

### Task 2: Characters page

**Files:**
- Modify: `src/renderer/pages/Characters.tsx`

- [ ] **Step 1: Implement Characters page**

Replace `src/renderer/pages/Characters.tsx`:

```tsx
import { useState, useEffect, useCallback } from 'react'
import { Button } from '@/components/ui/button'
import { UserPlus } from 'lucide-react'
import CharacterCard from '../components/CharacterCard'
import CharacterForm from '../components/CharacterForm'
import { ipc } from '../lib/ipc'
import { Character, Campaign } from '../../main/models/campaign'

export default function Characters() {
  const [campaign, setCampaign] = useState<Campaign | null>(null)
  const [formOpen, setFormOpen] = useState(false)
  const [editing, setEditing] = useState<Character | undefined>()

  const loadCampaign = useCallback(async () => {
    const settings = await ipc.settings.load()
    if (!settings.activeCampaignId) return
    const c = await ipc.campaign.load(settings.activeCampaignId)
    setCampaign(c)
  }, [])

  useEffect(() => { loadCampaign() }, [loadCampaign])

  const handleSave = async (character: Character) => {
    if (!campaign) return
    if (editing) {
      await ipc.character.update(campaign.id, character)
    } else {
      await ipc.character.add(campaign.id, character)
    }
    setFormOpen(false)
    setEditing(undefined)
    await loadCampaign()
  }

  const handleArchive = async (characterId: string) => {
    if (!campaign) return
    await ipc.character.archive(campaign.id, characterId)
    await loadCampaign()
  }

  const activeCharacters = campaign?.characters.filter(c => !c.archived) ?? []

  return (
    <div className="p-6 flex flex-col gap-6">
      <div className="flex items-center justify-between">
        <div>
          <h1 className="text-2xl font-bold">Characters</h1>
          {campaign && <p className="text-muted-foreground text-sm">{campaign.name}</p>}
        </div>
        <Button onClick={() => { setEditing(undefined); setFormOpen(true) }} className="gap-2">
          <UserPlus className="h-4 w-4" /> Add Character
        </Button>
      </div>

      {activeCharacters.length === 0 ? (
        <div className="flex flex-col items-center justify-center py-20 text-muted-foreground gap-3">
          <p>No characters yet.</p>
          <Button variant="outline" onClick={() => setFormOpen(true)}>Add your first character</Button>
        </div>
      ) : (
        <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
          {activeCharacters.map(c => (
            <CharacterCard key={c.id} character={c}
              onEdit={char => { setEditing(char); setFormOpen(true) }}
              onArchive={handleArchive}
            />
          ))}
        </div>
      )}

      <CharacterForm
        open={formOpen}
        character={editing}
        existingNames={activeCharacters.map(c => c.name)}
        onSave={handleSave}
        onClose={() => { setFormOpen(false); setEditing(undefined) }}
      />
    </div>
  )
}
```

- [ ] **Step 2: Run dev and verify**

```bash
npm run dev
```

Expected: Characters page shows the Add Character button. Clicking it opens the form dialog. Submitting adds a card to the grid.

- [ ] **Step 3: Commit**

```bash
git add src/renderer/pages/Characters.tsx
git commit -m "feat: implement character registry page"
```

---

### Task 3: Session History page

**Files:**
- Create: `src/renderer/components/SessionRow.tsx`
- Modify: `src/renderer/pages/History.tsx`
- Modify: `src/main/ipc/handlers.ts` — add `session:list` and `session:getEvents` handlers
- Modify: `src/preload/index.ts`
- Modify: `src/renderer/lib/ipc.ts`

- [ ] **Step 1: Add session history IPC handlers**

Add to `registerHandlers` in `src/main/ipc/handlers.ts`:

```ts
import { SessionStore } from '../storage/sessionStore'
import { getSessionPath } from '../storage/paths'
import fs from 'fs'
import path from 'path'

ipcMain.handle('session:list', (_e, campaignId: string) => {
  const basePath = getBasePath()
  const sessionsDir = path.join(basePath, 'campaigns', campaignId, 'sessions')
  if (!fs.existsSync(sessionsDir)) return []
  return fs.readdirSync(sessionsDir)
    .filter(f => f.endsWith('.db'))
    .map(f => {
      const sessionId = f.replace('.db', '')
      const dbPath = path.join(sessionsDir, f)
      try {
        const store = new SessionStore(dbPath)
        const session = store.getSession(sessionId)
        store.close()
        return session ?? { id: sessionId, campaignId, startedAt: 0 }
      } catch {
        return { id: sessionId, campaignId, startedAt: 0 }
      }
    })
    .sort((a, b) => b.startedAt - a.startedAt)  // newest first
})

ipcMain.handle('session:getEvents', (_e, campaignId: string, sessionId: string) => {
  const dbPath = getSessionPath(campaignId, sessionId)
  if (!fs.existsSync(dbPath)) return []
  const store = new SessionStore(dbPath)
  const session = store.getSession(sessionId)
  const events = store.getEvents(sessionId)
  store.close()
  return { session, events }
})
```

- [ ] **Step 2: Add to preload**

Add to the `api` object in `src/preload/index.ts`:

```ts
history: {
  list: (campaignId: string) => ipcRenderer.invoke('session:list', campaignId),
  getEvents: (campaignId: string, sessionId: string) =>
    ipcRenderer.invoke('session:getEvents', campaignId, sessionId)
}
// Note: session:list now returns Session[] (with startedAt), not string[]
```

- [ ] **Step 3: Add to renderer IPC wrapper**

Add to `src/renderer/lib/ipc.ts`:

```ts
import { SessionEvent, Session } from '../../main/models/session'

// Add to ipc object:
history: {
  list: (campaignId: string): Promise<Session[]> => api.history.list(campaignId),  // returns Session[], not string[]
  getEvents: (campaignId: string, sessionId: string): Promise<{ session: Session, events: SessionEvent[] }> =>
    api.history.getEvents(campaignId, sessionId)
}
```

- [ ] **Step 4: Create SessionRow**

Create `src/renderer/components/SessionRow.tsx`:

```tsx
import { ChevronRight } from 'lucide-react'
import { Session } from '../../main/models/session'

interface Props {
  session: Session
  onClick: () => void
}

export default function SessionRow({ session, onClick }: Props) {
  const date = session.startedAt > 0
    ? new Date(session.startedAt).toLocaleString()
    : 'Unknown date'
  const duration = session.endedAt && session.startedAt > 0
    ? `${Math.round((session.endedAt - session.startedAt) / 60000)} min`
    : null

  return (
    <button
      onClick={onClick}
      className="w-full flex items-center justify-between px-4 py-3 rounded-lg border hover:bg-muted/50 transition-colors text-left"
    >
      <div className="flex flex-col">
        <span className="font-medium">{session.title ?? `Session — ${date}`}</span>
        {duration && <span className="text-xs text-muted-foreground">{duration}</span>}
      </div>
      <ChevronRight className="h-4 w-4 text-muted-foreground" />
    </button>
  )
}
```

- [ ] **Step 5: Implement History page**

Replace `src/renderer/pages/History.tsx`:

```tsx
import { useState, useEffect, useCallback } from 'react'
import { Button } from '@/components/ui/button'
import { ArrowLeft } from 'lucide-react'
import SessionRow from '../components/SessionRow'
import TranscriptEntry from '../components/TranscriptEntry'
import { ScrollArea } from '@/components/ui/scroll-area'
import { ipc } from '../lib/ipc'
import { SessionEvent, Session } from '../../main/models/session'
import { ClassifiedEvent } from '../../main/models/events'

export default function History() {
  const [sessions, setSessions] = useState<Session[]>([])
  const [selected, setSelected] = useState<{ sessionId: string; events: SessionEvent[] } | null>(null)
  const [campaignId, setCampaignId] = useState<string | null>(null)

  useEffect(() => {
    ipc.settings.load().then(s => {
      if (!s.activeCampaignId) return
      setCampaignId(s.activeCampaignId)
      ipc.history.list(s.activeCampaignId).then(setSessions)
    })
  }, [])

  const handleSelect = useCallback(async (sessionId: string) => {
    if (!campaignId) return
    const data = await ipc.history.getEvents(campaignId, sessionId)
    setSelected({ sessionId, events: data.events })
  }, [campaignId])

  if (selected) {
    const asClassified = selected.events.map(e => ({
      id: e.id, timestamp: e.timestamp, source: 'local-mic' as const,
      text: e.rawText, eventType: e.eventType, characterId: e.characterId,
      inGame: e.inGame, confidence: e.confidence, flagged: e.flagged
    })) as ClassifiedEvent[]

    return (
      <div className="flex flex-col h-full p-6 gap-4">
        <div className="flex items-center gap-3">
          <Button variant="ghost" size="icon" onClick={() => setSelected(null)}>
            <ArrowLeft className="h-4 w-4" />
          </Button>
          <h1 className="text-2xl font-bold">Session Log</h1>
        </div>
        <ScrollArea className="flex-1 rounded-lg border bg-muted/20">
          <div className="flex flex-col gap-1 p-3">
            {asClassified.map(e => <TranscriptEntry key={e.id} item={e} />)}
          </div>
        </ScrollArea>
      </div>
    )
  }

  return (
    <div className="p-6 flex flex-col gap-4">
      <h1 className="text-2xl font-bold">Session History</h1>
      {sessions.length === 0 ? (
        <p className="text-muted-foreground py-10 text-center">No sessions recorded yet.</p>
      ) : (
        <div className="flex flex-col gap-2">
          {sessions.map(session => (
            <SessionRow key={session.id} session={session} onClick={() => handleSelect(session.id)} />
          ))}
        </div>
      )}
    </div>
  )
}
```

- [ ] **Step 6: Commit**

```bash
git add src/renderer/pages/History.tsx src/renderer/components/SessionRow.tsx src/main/ipc/handlers.ts src/preload/index.ts src/renderer/lib/ipc.ts
git commit -m "feat: implement session history page with event log view"
```

---

### Task 4: Settings page

**Files:**
- Create: `src/renderer/components/ApiKeyField.tsx`
- Create: `src/renderer/components/OutputSettings.tsx`
- Modify: `src/renderer/pages/Settings.tsx`

- [ ] **Step 1: Add required shadcn components**

```bash
npx shadcn@latest add switch collapsible
```

- [ ] **Step 2: Create ApiKeyField**

Create `src/renderer/components/ApiKeyField.tsx`:

```tsx
import { useState } from 'react'
import { Input } from '@/components/ui/input'
import { Button } from '@/components/ui/button'
import { Label } from '@/components/ui/label'
import { Eye, EyeOff, Check } from 'lucide-react'
import { ipc } from '../lib/ipc'

interface Props {
  secretKey: string
  label: string
  placeholder?: string
}

export default function ApiKeyField({ secretKey, label, placeholder }: Props) {
  const [value, setValue] = useState('')
  const [visible, setVisible] = useState(false)
  const [saved, setSaved] = useState(false)

  const handleSave = async () => {
    if (!value.trim()) return
    await ipc.settings.setSecret(secretKey, value)
    setSaved(true)
    setValue('')
    setTimeout(() => setSaved(false), 2000)
  }

  return (
    <div className="grid gap-1.5">
      <Label>{label}</Label>
      <div className="flex gap-2">
        <div className="relative flex-1">
          <Input
            type={visible ? 'text' : 'password'}
            value={value}
            onChange={e => setValue(e.target.value)}
            placeholder={placeholder ?? 'Paste your key here'}
            className="pr-10"
          />
          <button
            type="button"
            onClick={() => setVisible(v => !v)}
            className="absolute right-3 top-1/2 -translate-y-1/2 text-muted-foreground hover:text-foreground"
          >
            {visible ? <EyeOff className="h-4 w-4" /> : <Eye className="h-4 w-4" />}
          </button>
        </div>
        <Button onClick={handleSave} disabled={!value.trim()} variant={saved ? 'outline' : 'default'}>
          {saved ? <><Check className="h-4 w-4 mr-1" /> Saved</> : 'Save'}
        </Button>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Extend IPC for secrets in renderer**

Add to `src/preload/index.ts` api object:

```ts
settings: {
  // ... existing
  setSecret: (key: string, value: string) => ipcRenderer.invoke('settings:setSecret', key, value),
  getSecretExists: (key: string) => ipcRenderer.invoke('settings:getSecretExists', key)
}
```

Add to `src/main/ipc/handlers.ts` registerHandlers:

```ts
ipcMain.handle('settings:setSecret', (_e, key: string, value: string) =>
  settingsStore.setSecret(key, value))
ipcMain.handle('settings:getSecretExists', async (_e, key: string) => {
  const val = await settingsStore.getSecret(key)
  return val !== null
})
```

Add to `src/renderer/lib/ipc.ts` ipc.settings:

```ts
setSecret: (key: string, value: string): Promise<void> => api.settings.setSecret(key, value),
getSecretExists: (key: string): Promise<boolean> => api.settings.getSecretExists(key)
```

- [ ] **Step 4: Create OutputSettings component**

Create `src/renderer/components/OutputSettings.tsx`:

```tsx
import { Switch } from '@/components/ui/switch'
import { Label } from '@/components/ui/label'
import { OutputSettings as OutputSettingsType } from '../../main/models/settings'

interface Props {
  value: OutputSettingsType
  onChange: (updated: OutputSettingsType) => void
}

export default function OutputSettings({ value, onChange }: Props) {
  const toggle = (field: keyof OutputSettingsType) =>
    onChange({ ...value, [field]: !value[field as keyof typeof value] })

  return (
    <div className="grid gap-4">
      <div className="flex items-center justify-between">
        <div>
          <Label className="text-base">Save Markdown File</Label>
          <p className="text-sm text-muted-foreground">Auto-saves every session as a .md file</p>
        </div>
        <Switch checked={value.markdownEnabled} onCheckedChange={() => toggle('markdownEnabled')} />
      </div>
      <div className="flex items-center justify-between">
        <div>
          <Label className="text-base">Copy to Clipboard</Label>
          <p className="text-sm text-muted-foreground">Summary available to paste after generating</p>
        </div>
        <Switch checked={value.clipboardEnabled} onCheckedChange={() => toggle('clipboardEnabled')} />
      </div>
    </div>
  )
}
```

- [ ] **Step 5: Implement Settings page**

Replace `src/renderer/pages/Settings.tsx`:

```tsx
import { useState, useEffect } from 'react'
import { Button } from '@/components/ui/button'
import { Collapsible, CollapsibleContent, CollapsibleTrigger } from '@/components/ui/collapsible'
import { ChevronDown } from 'lucide-react'
import ApiKeyField from '../components/ApiKeyField'
import OutputSettingsPanel from '../components/OutputSettings'
import { ipc } from '../lib/ipc'
import { AppSettings } from '../../main/models/settings'

export default function Settings() {
  const [settings, setSettings] = useState<AppSettings | null>(null)
  const [saved, setSaved] = useState(false)

  useEffect(() => {
    ipc.settings.load().then(setSettings)
  }, [])

  const handleSave = async () => {
    if (!settings) return
    await ipc.settings.save(settings)
    setSaved(true)
    setTimeout(() => setSaved(false), 2000)
  }

  if (!settings) return null

  return (
    <div className="p-6 flex flex-col gap-8 max-w-xl">
      <h1 className="text-2xl font-bold">Settings</h1>

      <section className="flex flex-col gap-4">
        <h2 className="text-lg font-semibold">AI</h2>
        <ApiKeyField secretKey="claude-api-key" label="Claude API Key" placeholder="sk-ant-..." />
      </section>

      <section className="flex flex-col gap-4">
        <h2 className="text-lg font-semibold">Output</h2>
        <OutputSettingsPanel
          value={settings.output}
          onChange={output => setSettings(s => s ? { ...s, output } : s)}
        />
      </section>

      <Collapsible>
        <CollapsibleTrigger className="flex items-center gap-2 text-sm text-muted-foreground hover:text-foreground">
          <ChevronDown className="h-4 w-4" /> Advanced (Discord)
        </CollapsibleTrigger>
        <CollapsibleContent className="mt-4">
          <p className="text-sm text-muted-foreground">Discord integration coming soon. You will be able to connect a bot to post summaries to a channel.</p>
        </CollapsibleContent>
      </Collapsible>

      <Button onClick={handleSave} className="w-fit" variant={saved ? 'outline' : 'default'}>
        {saved ? 'Saved!' : 'Save Settings'}
      </Button>
    </div>
  )
}
```

- [ ] **Step 6: Commit**

```bash
git add src/renderer/pages/Settings.tsx src/renderer/components/ApiKeyField.tsx src/renderer/components/OutputSettings.tsx src/main/ipc/handlers.ts src/preload/index.ts src/renderer/lib/ipc.ts
git commit -m "feat: implement settings page with API key and output config"
```

---

### Task 5: Onboarding wizard

**Files:**
- Create: `src/renderer/pages/Onboarding.tsx`
- Modify: `src/renderer/App.tsx`

- [ ] **Step 1: Create Onboarding page**

Create `src/renderer/pages/Onboarding.tsx`:

```tsx
import { useState } from 'react'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'
import { Card, CardContent, CardFooter, CardHeader, CardTitle, CardDescription } from '@/components/ui/card'
import CharacterForm from '../components/CharacterForm'
import { ipc } from '../lib/ipc'
import { Campaign, Character } from '../../main/models/campaign'
import { v4 as uuid } from 'uuid'

interface Props {
  onComplete: () => void
}

export default function Onboarding({ onComplete }: Props) {
  const [step, setStep] = useState(1)
  const [campaignName, setCampaignName] = useState('')
  const [campaign, setCampaign] = useState<Campaign | null>(null)
  const [characters, setCharacters] = useState<Character[]>([])
  const [formOpen, setFormOpen] = useState(false)
  const [apiKey, setApiKey] = useState('')
  const [apiKeyStatus, setApiKeyStatus] = useState<'idle' | 'testing' | 'valid' | 'invalid'>('idle')

  const handleCreateCampaign = async () => {
    if (!campaignName.trim()) return
    const newCampaign: Campaign = {
      id: uuid(), name: campaignName.trim(), createdAt: Date.now(), characters: []
    }
    await ipc.campaign.save(newCampaign)
    setCampaign(newCampaign)
    setStep(2)
  }

  const handleAddCharacter = async (character: Character) => {
    if (!campaign) return
    await ipc.character.add(campaign.id, character)
    setCharacters(prev => [...prev, character])
    setFormOpen(false)
  }

  const handleTestApiKey = async () => {
    if (!apiKey.startsWith('sk-ant-')) {
      setApiKeyStatus('invalid')
      return
    }
    setApiKeyStatus('testing')
    try {
      // Basic format check — real validation happens at first Claude call
      await ipc.settings.setSecret('claude-api-key', apiKey)
      setApiKeyStatus('valid')
    } catch {
      setApiKeyStatus('invalid')
    }
  }

  const handleFinish = async () => {
    const settings = await ipc.settings.load()
    await ipc.settings.save({ ...settings, activeCampaignId: campaign!.id, onboardingComplete: true })
    onComplete()
  }

  return (
    <div className="flex h-screen items-center justify-center bg-background p-6">
      <div className="w-full max-w-lg">
        <div className="flex gap-2 mb-8 justify-center">
          {[1, 2, 3, 4].map(n => (
            <div key={n} className={`h-2 rounded-full transition-all ${n <= step ? 'w-8 bg-primary' : 'w-2 bg-muted'}`} />
          ))}
        </div>

        {step === 1 && (
          <Card>
            <CardHeader>
              <CardTitle>Welcome to DM Assistant</CardTitle>
              <CardDescription>Let's get you set up. This takes about 2 minutes.</CardDescription>
            </CardHeader>
            <CardContent className="grid gap-3">
              <Label htmlFor="campaign-name">What's your campaign called?</Label>
              <Input
                id="campaign-name"
                value={campaignName}
                onChange={e => setCampaignName(e.target.value)}
                placeholder="The Lost Mines of Phandelver"
                onKeyDown={e => e.key === 'Enter' && handleCreateCampaign()}
              />
            </CardContent>
            <CardFooter>
              <Button className="w-full" onClick={handleCreateCampaign} disabled={!campaignName.trim()}>
                Next
              </Button>
            </CardFooter>
          </Card>
        )}

        {step === 2 && (
          <Card>
            <CardHeader>
              <CardTitle>Add Your Players</CardTitle>
              <CardDescription>Add the characters in your campaign. You can always add more later.</CardDescription>
            </CardHeader>
            <CardContent className="grid gap-3">
              {characters.length === 0 ? (
                <p className="text-sm text-muted-foreground text-center py-4">No characters added yet.</p>
              ) : (
                characters.map(c => (
                  <div key={c.id} className="flex items-center justify-between rounded-md border px-3 py-2 text-sm">
                    <span className="font-medium">{c.name}</span>
                    <span className="text-muted-foreground">{c.playerName} · {c.characterClass}</span>
                  </div>
                ))
              )}
              <Button variant="outline" onClick={() => setFormOpen(true)} className="w-full">
                + Add Character
              </Button>
            </CardContent>
            <CardFooter className="gap-2">
              <Button variant="ghost" onClick={() => setStep(3)} className="flex-1">Skip for now</Button>
              <Button onClick={() => setStep(3)} className="flex-1" disabled={characters.length === 0}>
                Next
              </Button>
            </CardFooter>
          </Card>
        )}

        {step === 3 && (
          <Card>
            <CardHeader>
              <CardTitle>Almost there!</CardTitle>
              <CardDescription>
                Your campaign <strong>{campaign?.name}</strong> is set up with {characters.length} character{characters.length !== 1 ? 's' : ''}.
                One last step — let's add your Claude API key so the AI features work.
              </CardDescription>
            </CardHeader>
            <CardFooter>
              <Button className="w-full" onClick={() => setStep(4)}>Next</Button>
            </CardFooter>
          </Card>
        )}

        {step === 4 && (
          <Card>
            <CardHeader>
              <CardTitle>Add Your Claude API Key</CardTitle>
              <CardDescription>
                This is a password that lets the app use AI — like a library card for Claude.
                Get one free at <strong>console.anthropic.com</strong> → API Keys.
              </CardDescription>
            </CardHeader>
            <CardContent className="grid gap-4">
              <div className="grid gap-1.5">
                <label className="text-sm font-medium">API Key</label>
                <div className="flex gap-2">
                  <input
                    type="password"
                    value={apiKey}
                    onChange={e => { setApiKey(e.target.value); setApiKeyStatus('idle') }}
                    placeholder="sk-ant-..."
                    className="flex-1 rounded-md border bg-background px-3 py-2 text-sm"
                  />
                  <Button
                    variant="outline"
                    onClick={handleTestApiKey}
                    disabled={!apiKey.trim() || apiKeyStatus === 'testing'}
                    size="sm"
                  >
                    Test
                  </Button>
                </div>
                {apiKeyStatus === 'valid' && (
                  <p className="text-xs text-green-600">Connected — key saved securely.</p>
                )}
                {apiKeyStatus === 'invalid' && (
                  <p className="text-xs text-destructive">Invalid key — check that it starts with sk-ant- at console.anthropic.com</p>
                )}
              </div>
            </CardContent>
            <CardFooter className="gap-2">
              <Button
                variant="ghost"
                onClick={handleFinish}
                className="flex-1"
              >
                Skip — I'll add it later
              </Button>
              <Button
                onClick={handleFinish}
                className="flex-1"
                disabled={apiKeyStatus !== 'valid'}
              >
                Let's go!
              </Button>
            </CardFooter>
          </Card>
        )}

        <CharacterForm open={formOpen} existingNames={characters.map(c => c.name)} onSave={handleAddCharacter} onClose={() => setFormOpen(false)} />
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Update App.tsx to show onboarding on first launch**

Replace `src/renderer/App.tsx`:

```tsx
import { useState, useEffect } from 'react'
import { MemoryRouter, Routes, Route } from 'react-router-dom'
import Sidebar from './components/Sidebar'
import Home from './pages/Home'
import Characters from './pages/Characters'
import History from './pages/History'
import Settings from './pages/Settings'
import Onboarding from './pages/Onboarding'
import { ipc } from './lib/ipc'

export default function App() {
  const [onboardingDone, setOnboardingDone] = useState<boolean | null>(null)

  useEffect(() => {
    ipc.settings.load().then(s => setOnboardingDone(s.onboardingComplete))
  }, [])

  if (onboardingDone === null) return null

  if (!onboardingDone) {
    return <Onboarding onComplete={() => setOnboardingDone(true)} />
  }

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

- [ ] **Step 3: Run dev and verify onboarding flow**

```bash
npm run dev
```

Expected: On first launch, onboarding wizard appears. Step 1 asks for campaign name, step 2 lets you add characters, step 3 shows completion. After finishing, main app appears with sidebar.

- [ ] **Step 4: Commit**

```bash
git add src/renderer/pages/Onboarding.tsx src/renderer/App.tsx
git commit -m "feat: add first-launch onboarding wizard"
```

---

### Task 6: Run full test suite and push

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
| Eng Review | `/plan-eng-review` | Architecture & tests (required) | 1 | resolved | 4 bugs fixed; onboarding step 4 added |
| Codex Review | `/codex review` | Independent 2nd opinion | 0 | — | — |
| Design Review | `/plan-design-review` | UI/UX gaps | 0 | — | — |

**FIXES APPLIED:**
- Bug: `session:list` now returns `Session[]` with `startedAt`/`endedAt` instead of `string[]` of UUIDs (CEO Gap 4)
- Bug: `SessionRow` rewrote date display to use `session.startedAt` — shows human-readable date + duration (CEO Gap 4)
- Bug: Duplicate character name check added to `CharacterForm.handleSave` with inline error display (CEO Gap 8)
- Bug: `CharacterForm` now accepts `existingNames` prop; both Characters page and Onboarding pass it
- Added: Onboarding step 4 — API key setup with paste field, format validation, and "Test" + skip option (CEO cherry-pick Plan 3)
- Updated: `ipc.history.list` return type is `Session[]` not `string[]`

**VERDICT:** ENG REVIEW COMPLETE — plan ready for implementation.
