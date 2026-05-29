# Plato Sensory Architecture

> *"The agent lives in a cave. On the walls, shadows of the real world dance. The agent never sees the fire — only the shapes it casts."*

## Overview

The Plato Sensory Architecture is the translation layer between rich external reality and the agent's text-based cognition. Every sense is a pair of one-way translations:

- **Shadows** (outside → inside): raw sensory data compressed into structured text the agent can read
- **Projections** (inside → outside): agent text commands expanded into real-world actions

**Core invariant: The agent NEVER handles raw sensory data. Everything arrives as text. Everything departs as text.**

```
  ┌─────────────────────────────────────────────────────┐
  │                   THE CAVE                          │
  │                                                     │
  │   Agent sees only text ──────────────────┐          │
  │                                          │          │
  │   ┌──────────────────────────────────┐   │          │
  │   │        Agent Core (LLM)         │◄──┘          │
  │   │   reads text, writes commands    │              │
  │   └──────────┬──────────▲────────────┘              │
  │              │          │                            │
  │    ┌─────────▼──────────┴──────────┐                │
  │    │    PLATO SENSORY LAYER        │                │
  │    │                               │                │
  │    │  ┌─────────┐  ┌───────────┐  │                │
  │    │  │ Shadows  │  │Projections│  │                │
  │    │  │(in→text) │  │(text→out) │  │                │
  │    │  └────┬─────┘  └─────┬─────┘                │
  │    └───────┼──────────────┼──────────────────────┘ │
  │            │              │                         │
  ════════════╪══════════════╪═════════════════════════╣
  │            │              │     THE OUTSIDE         │
  │   ┌────────▼──┐    ┌─────▼──────┐                  │
  │   │ Cameras   │    │ Desktop    │                  │
  │   │ Mics      │    │ APIs       │                  │
  │   │ Sonar     │    │ Files      │                  │
  │   │ Desktop   │    │ Audio out  │                  │
  │   └───────────┘    └────────────┘                  │
  └─────────────────────────────────────────────────────┘
```

---

## Architecture Principles

### 1. Text-Only Interface

Every sense module exposes exactly two things:
- **A readable text stream** (what the agent sees — the shadow)
- **A writable command protocol** (what the agent can do — the projection)

No images. No waveforms. No pixel buffers. Only text.

### 2. Lossy by Design

Compression is not a bug — it's the architecture. The agent gets *semantic meaning*, not raw data. A camera doesn't deliver pixel arrays; it delivers "a person is standing near the door, wearing a red jacket, facing left." This is intentional and desirable.

The translation layer decides what to keep and what to drop. Different abstraction levels serve different needs:

| Level | Label | What the agent sees |
|-------|-------|-------------------|
| L0 | Raw | (never exposed) |
| L1 |摘要 Summary | One-line overview |
| L2 | Description | Paragraph with key details |
| L3 | Catalog | Structured inventory of elements |
| L4 | Annotation | Detailed spatial/relational map |

Agents can request a level. Default is L2.

### 3. Bidirectional Symmetry

Every sense has both directions where applicable. If the agent can *see* a desktop, it can *control* it. If it can *hear* audio, it can *speak*. The command vocabulary mirrors the observation vocabulary.

### 4. Policy-Gated

Every sensory channel passes through OpenShell's policy engine. No sense gets unrestricted access by default. Each module declares its capabilities and policy boundaries.

---

## Module Specifications

---

### 1. Plato-Manus — Hands

*File operations, API calls, device control. The agent's manipulatory system.*

**Metaphor:** Hands that reach into the world — touching files, pressing API buttons, turning device knobs.

#### Shadow Protocol (what the agent reads)

```
[MANUS:FS] /home/user/documents/
  ├── notes/ (3 files, 12KB)
  │   ├── todo.md (modified 2h ago)
  │   ├── ideas.md (modified 1d ago)
  │   └── archive/ (empty)
  ├── photos/ (47 files, 1.2GB)
  └── .config (hidden, 8 files)

[MANUS:API] GET https://api.weather.gov/zones/AKZ101/forecast
  Status: 200 | Latency: 142ms
  Summary: Partly cloudy, high 54°F. Snow above 2000ft tonight.
  Key fields: temperature, windSpeed, shortForecast

[MANUS:DEVICE] thermostat-living-room
  State: heating | Target: 68°F | Current: 64°F | Humidity: 42%
  Last change: 15min ago (auto)
```

#### Control Protocol (what the agent commands)

```
MANUS:FS:READ /home/user/documents/notes/todo.md
MANUS:FS:WRITE /home/user/documents/notes/todo.md "content..."
MANUS:FS:LIST /home/user/documents/ depth=2 show_hidden=false
MANUS:FS:MOVE /old/path → /new/path
MANUS:FS:DELETE /path/to/file (requires confirmation)

MANUS:API:GET <url> headers={} params={}
MANUS:API:POST <url> body={} headers={}
MANUS:API:PAGINATE <url> max_pages=10 summary_mode=true

MANUS:DEVICE:READ <device-id>
MANUS:DEVICE:SET <device-id> <property>=<value>
MANUS:DEVICE:LIST
```

#### Abstraction Levels

| Level | File System | API | Device |
|-------|-----------|-----|--------|
| L1 | "3 files, 12KB total" | "200 OK, weather data" | "Thermostat: 64°F" |
| L2 | Tree view with sizes/dates | Response summary with key fields | State + last change |
| L3 | Full tree with file types, permissions | Full response headers + body outline | All properties + history |
| L4 | File contents inline | Raw response body | Device schema + all values |

#### Policy Boundaries

- **File system:** Read-only by default. Write/delete require `fs:write` permission. Path restrictions via allow/deny lists. `trash` preferred over `rm`.
- **API:** Domain allowlist per agent. Rate limiting enforced. No credential exposure to agent (auth injected by policy layer).
- **Device:** Read-only by default. State changes require `device:control` permission. Safety interlocks for physical devices (no "set thermostat to 200°F").

---

### 2. Plato-Playwright — Desktop Automation

*Browser control, app interaction. The agent's ability to navigate and manipulate GUI environments.*

**Metaphor:** A stagehand who can walk behind the set and move props, pull levers, change scenery.

#### Shadow Protocol (what the agent reads)

```
[PLAYWRIGHT:BROWSER] Tab: "Gmail - Inbox" (active)
  URL: mail.google.com/mail/inbox
  Page state: loaded | Scroll: top
  Visible elements:
    - Navigation: sidebar (Compose, Inbox(23), Sent, Drafts)
    - Main: email list, 23 unread, sorted by date
    - Top email: "Project Update - Alice" received 10min ago
    - Current focus: email list

[PLAYWRIGHT:APP] Window: "VS Code" (focused, maximized)
  Workspace: /home/user/project
  Open files: main.rs, lib.rs, config.toml
  Active file: main.rs | Cursor: line 42, col 8
  Terminal: 1 session (bash, idle)
  Problems: 2 warnings, 0 errors

[PLAYWRIGHT:SCREEN] Resolution: 2560x1440
  Active window: "VS Code" (left 60%)
  Background windows: "Firefox" (right 40%), "Slack" (minimized)
  System tray: WiFi▲, Battery 78%, Volume 45%, Clock 14:32
```

#### Control Protocol (what the agent commands)

```
PLAYWRIGHT:BROWSER:NAVIGATE <url>
PLAYWRIGHT:BROWSER:CLICK <selector>
PLAYWRIGHT:BROWSER:TYPE <text> selector=<sel> clear_first=true
PLAYWRIGHT:BROWSER:SCROLL direction=down amount=300px
PLAYWRIGHT:BROWSER:SCREENSHOT → triggers Plato-Vision shadow
PLAYWRIGHT:BROWSER:WAIT_FOR <selector> timeout=5s
PLAYWRIGHT:BROWSER:EXTRACT <selector> → returns text content

PLAYWRIGHT:APP:ACTIVATE <window-name>
PLAYWRIGHT:APP:KEYPRESS <key-combo>  (e.g., "ctrl+s", "alt+tab")
PLAYWRIGHT:APP:MENU <menu-path>       (e.g., "File > Save As...")
PLAYWRIGHT:APP:DRAG <from-selector> → <to-selector>

PLAYWRIGHT:SCREEN:LIST_WINDOWS
PLAYWRIGHT:SCREEN:ARRANGE <layout>    (tile-horizontal, tile-vertical, cascade)
```

#### Abstraction Levels

| Level | Browser | App | Screen |
|-------|---------|-----|--------|
| L1 | "Gmail inbox, 23 unread" | "VS Code open, editing main.rs" | "2 windows visible" |
| L2 | Page structure + visible elements | Open files + cursor + problems | Window positions + system tray |
| L3 | Full DOM outline with form values | Full UI tree with state | Complete window manager state |
| L4 | Accessible HTML snapshot | Raw accessibility tree | Pixel-level region descriptions |

#### Policy Boundaries

- **Browser:** Domain allowlist. No credential/password field reading by default. Form submission requires confirmation. Cookie/session access restricted.
- **App:** Clipboard access gated. Key injection blocked for security-sensitive combos (ctrl+alt+del, etc.). File dialogs intercepted and routed through Manus.
- **Screen:** Screen recording/capture requires `screen:capture` permission. Background window content may be redacted based on sensitivity.

---

### 3. Plato-Puppeteer — Desktop as MUD

*The entire desktop translated into a text adventure. The most radical compression.*

**Metaphor:** The desktop becomes a MUD (Multi-User Dungeon). Windows are rooms. UI elements are objects you can examine and interact with. The agent navigates via text commands like a text adventure.

#### Shadow Protocol (what the agent reads)

```
[PUPPETeer:ROOM] Desktop
You are on the Desktop. The screen is bright.

Obvious exits:
  [W] VS Code — A code editor, workspace "project"
  [S] Firefox — A web browser, 3 tabs open
  [E] Terminal — A command line, bash session active

Objects here:
  A system tray with WiFi, Battery (78%), Volume, and Clock (2:32 PM)
  A taskbar with 4 pinned apps and 2 notifications
  A wallpaper showing a mountain landscape

> go west

[PUPPETEER:ROOM] VS Code — main.rs
You are in VS Code, editing main.rs. The cursor blinks at line 42.

Visible areas:
  - Editor: The file main.rs is open. Lines 38-50 are visible.
    Line 42 has a blinking cursor: `    let result = process(`
    There are 2 warnings (wavy underlines on lines 31 and 45).
  - Sidebar: File explorer showing 12 files in /project/src
  - Bottom: Terminal panel (bash, last output: "Build successful")
  - Minimap: A compressed view of the full file on the right edge

Objects of interest:
  A "Save" button (dimmed — no unsaved changes)
  A tab bar with 3 open tabs (main.rs*, lib.rs, config.toml)
  A suggestion dropdown offering 3 completions

> examine line 42

[PUPPETEER:DETAIL] Line 42 of main.rs
  `    let result = process(input, config)?;`
  
  The variable `input` is defined on line 38 as a String.
  The function `process` is imported from crate::lib.
  The `?` operator will propagate any error to the caller.
  Warning: `result` is assigned but unused in subsequent lines.

> go south

[PUPPETEER:ROOM] Firefox — Gmail Inbox
You are in Firefox, looking at Gmail. The inbox has 23 unread messages.

Visible areas:
  - Sidebar: Navigation (Compose, Inbox(23), Sent, Drafts(2), Spam(1))
  - Main: Email list, sorted by date, newest first
  - Top message: "Project Update" from Alice, 10 minutes ago, unread
  - Second message: "Re: API Design" from Bob, 1 hour ago, unread

Objects of interest:
  A "Compose" button (prominent, top-left)
  A search bar (empty)
  A refresh indicator (spinning — checking for new mail)
  Three checkboxes next to the top three emails

> take compose

[PUPPETEER:ACTION] You click the Compose button.
A new window appears in the bottom-right corner.

[PUPPETEER:ROOM] Firefox — Gmail Compose
You are looking at a compose window. It is a small overlay.

Visible areas:
  - To: field (empty, cursor active)
  - Subject: field (empty)
  - Body: large text area (empty)
  - Bottom: Send button (dimmed — no recipients)

Objects of interest:
  A "To" input field awaiting text
  A "Subject" input field
  A formatting toolbar (bold, italic, lists, etc.)
  A "Discard" button
```

#### Control Protocol (what the agent commands)

```
PUPPETEER:GO <direction|window-name>     — navigate to a window/area
PUPPETEER:LOOK                            — re-describe current room
PUPPETEER:LOOK AT <element>               — examine a specific element
PUPPETEER:TAKE <action>                   — click/activate an element
PUPPETEER:PUT <text> IN <field>           — type into an input
PUPPETEER:USE <element>                   — interact (toggle, select, etc.)
PUPPETEER:READ <region>                   — get full text of an area
PUPPETEER:INVENTORY                       — list all open windows
PUPPETEER:MAP                             — show spatial layout of desktop
```

#### Abstraction Levels

| Level | Description Style |
|-------|------------------|
| L1 | One-sentence room summary ("You are in VS Code, editing main.rs") |
| L2 | Full MUD-style room description with obvious exits and objects |
| L3 | Includes element states, values, and relationships |
| L4 | Accessibility tree rendered as prose with full state dump |

#### Policy Boundaries

- Inherits all Playwright policies.
- Additionally: agent cannot navigate to `PUPPETEER:GO system-settings` or equivalent privileged areas without `system:admin`.
- "Dangerous rooms" (password managers, banking sites, crypto wallets) are blocked or heavily redacted by default.
- The MUD metaphor is a *display layer* — it cannot bypass underlying OS permissions.

#### Design Rationale

The MUD metaphor is surprisingly powerful because:
1. **Spatial reasoning maps naturally.** Agents already understand "rooms" and "exits."
2. **Object interaction is intuitive.** "Take the compose button" is more natural than `document.querySelector('[aria-label="Compose"]').click()`.
3. **Exploration is built in.** "Look around" is the most natural first action in any new environment.
4. **State tracking is implicit.** "You are in..." automatically maintains context.
5. **Error messages are diegetic.** "You can't go that way" is clearer than `ElementNotInteractableError`.

---

### 4. Plato-Vision — Eyes

*Camera feeds translated to text scene descriptions. The agent's visual cortex.*

**Metaphor:** A companion who sits by the window and describes everything they see in vivid, structured prose.

#### Shadow Protocol (what the agent reads)

```
[VISION:CAM:front-door] 14:32:07 AKST
  Scene: Exterior, daytime, overcast
  Motion: None detected
  Subjects:
    - No people visible
    - A package (small, brown cardboard) on the porch step
      Position: center of welcome mat, ~2ft from door
      Text visible on label: "FRAGILE"
  Environment:
    - Porch light: off
    - Door: closed
    - Walkway: clear, no obstructions
    - Driveway: one vehicle (dark blue sedan, stationary)
  Confidence: 0.94

[VISION:CAM:living-room] 14:32:07 AKST
  Scene: Interior, living room, artificial light (warm)
  Subjects:
    - Person (adult, seated on couch, facing TV)
      Posture: relaxed, leaning back
      Activity: looking at phone screen
      Identity: unknown (face partially visible, profile view)
    - Cat (orange tabby, on armrest of couch)
      Position: right armrest, curled up
      State: sleeping
  Objects of note:
    - TV: on, displaying what appears to be a streaming service menu
    - Coffee table: mug (half-full), remote control, book (open, face-down)
    - Window: blinds drawn, gap shows overcast sky
  Environment:
    - Temperature: ~70°F (estimated)
    - Lighting: warm overhead + TV glow
    - Sound level: quiet (no audio feed, visual inference)
  Confidence: 0.87

[VISION:CHANGE:front-door] 14:32:07 → 14:32:12
  Delta: A person has appeared at the edge of the frame (left side)
  Subject: Person (adult, wearing dark jacket, carrying a bag)
    Movement: walking toward the door (direction: right)
    Pace: normal walking speed
  Previously reported package: still present, not yet reached by person
  Alert: NEW PERSON IN FRAME
```

#### Control Protocol (what the agent commands)

```
VISION:DESCRIBE <camera-id> level=<L1-L4> focus=<region>
VISION:COMPARE <camera-id> <timestamp1> <timestamp2>
VISION:FIND <camera-id> <object-description>
VISION:TRACK <camera-id> <object-id>
VISION:SNAPSHOT <camera-id> → saves image, returns path
VISION:LIST_CAMERAS
VISION:HISTORY <camera-id> since=<time> events_only=true
VISION:ZONE <camera-id> define=<name:coordinates>
VISION:ALERT <camera-id> when=<condition>  (e.g., "person in zone:porch")
```

#### Abstraction Levels

| Level | Content |
|-------|---------|
| L1 | "Empty porch, one package, overcast day" |
| L2 | Full scene description with subjects, objects, environment |
| L3 | L2 + spatial relationships, measurements, text OCR, confidence scores |
| L4 | Pixel-level region analysis with bounding boxes (as text), color data, motion vectors |

#### Temporal Intelligence

Plato-Vision doesn't just describe static frames. It maintains a temporal model:

- **Baseline:** What does this scene normally look like? Learned over time.
- **Deltas:** What changed since last observation? Reported as events.
- **Predictions:** Based on motion vectors and patterns, what's about to happen? ("Person approaching door, likely to knock in ~5 seconds")
- **Alerts:** User-defined conditions that trigger notifications.

#### Policy Boundaries

- **Camera access:** Per-camera permissions. Some cameras may be read-only (no PTZ control). Bedroom/bathroom cameras blocked by default.
- **Face identification:** Off by default. Requires `vision:identify` permission and user consent. Even then, only pre-approved identities.
- **Recording:** Snapshots require `vision:record`. No continuous recording without explicit permission.
- **Alert routing:** Vision alerts go through the standard notification pipeline, not directly to external channels.

---

### 5. Plato-Sonar — Ears

*Acoustic analysis and prediction. Integrates with the existing sonar-vision-rs project.*

**Metaphor:** A blind companion with exceptional hearing who describes the soundscape — what's happening, where it's coming from, and what it probably means.

#### Shadow Protocol (what the agent reads)

```
[SONAR:MIC:kitchen] 14:32:07 AKST
  Overall: Quiet indoor environment, ~35dB ambient
  Sources:
    - Refrigerator hum (constant, ~40dB, direction: north-west)
      Status: normal operating sound
    - Water running (moderate, ~50dB, direction: south/sink area)
      Started: ~45 seconds ago
      Pattern: consistent flow (faucet, not burst)
    - Footsteps (intermittent, ~55dB, direction: moving south→north)
      Count: 6 steps in last 10 seconds
      Surface: hard floor (tile/wood)
      Weight estimate: adult
  Classification: "Someone is in the kitchen, washing something at the sink, pacing"
  Confidence: 0.91

[SONAR:MIC:front-door] 14:32:12 AKST
  Alert: KNOCK detected (2 knocks, 0.8s apart, ~75dB)
  Direction: exterior (front door)
  Pattern: deliberate human knock (not package delivery, not wind)
  Followed by: doorbell ring (1 ring, standard pattern)
  Classification: "Visitor at front door — deliberate knock then doorbell"
  Confidence: 0.96

[SONAR:CHANGE:living-room] 14:31:50 → 14:32:07
  Delta: TV audio stopped (was: low-volume speech/music)
  New: Silence in living room (person may have muted or turned off TV)
  Correlated vision data: Person on couch now looking up from phone
  Inference: Person may be responding to the knock/doorbell sound
```

#### Control Protocol (what the agent commands)

```
SONAR:LISTEN <mic-id> duration=<seconds> level=<L1-L4>
SONAR:CLASSIFY <mic-id> source=<direction|frequency>
SONAR:LOCATE <sound-description> → triangulate across mics
SONAR:HISTORY <mic-id> since=<time> events_only=true
SONAR:BASELINE <mic-id> → show learned ambient profile
SONAR:ALERT <mic-id> when=<sound-class> threshold=<dB>
SONAR:LIST_MICS
SONAR:CALIBRATE <mic-id>
```

#### Sonar-Vision-RS Integration

Plato-Sonar builds on the existing `sonar-vision-rs` Rust library, extending it with:

1. **Scene-level classification** — not just "sound detected" but "someone is doing dishes"
2. **Multi-mic fusion** — cross-referencing multiple microphone inputs for spatial accuracy
3. **Temporal patterns** — learning what sounds are normal for a space at a given time
4. **Vision cross-reference** — correlating sound events with camera data when available (e.g., "heard footsteps, camera confirms person walking")

The Rust layer handles:
- Raw audio capture and buffering
- FFT/spectral analysis
- Sound event detection (SED)
- Direction-of-arrival estimation

Plato-Sonar's text layer handles:
- Semantic classification ("that's a knock, not a thump")
- Scene understanding ("someone is cooking")
- Temporal reasoning ("this is unusual for 2AM")
- Natural language output

#### Abstraction Levels

| Level | Content |
|-------|---------|
| L1 | "Kitchen: quiet, sink running, someone walking around" |
| L2 | Identified sound sources with directions, volumes, and patterns |
| L3 | L2 + spectral details, temporal patterns, confidence scores |
| L4 | Raw acoustic features as text (frequency bands, amplitudes, phase data) |

#### Policy Boundaries

- **Microphone access:** Per-mic permissions. Always-on listening requires `sonar:continuous`. Default is event-triggered.
- **Speech content:** No speech-to-text by default. If enabled, requires `sonar:speech` permission. Transcripts are ephemeral — not stored unless explicitly requested.
- **Sound classification:** No speaker identification without `sonar:identify` permission.
- **Alert sensitivity:** User-configurable thresholds to prevent alert fatigue.

---

### 6. A2UI — Agent-to-UI (The Reverse Projection)

*Translating agent text output into rendered human interfaces. The shadow-play projected back onto the world.*

**Metaphor:** If the agent lives in a cave watching shadows, A2UI is the agent *casting its own shadows* — shaping the light so humans outside see what the agent intends.

#### The Problem

The agent thinks in text. But humans need:
- Dashboards, not log output
- Forms, not JSON schemas
- Charts, not CSV data
- Buttons, not command-line flags
- Notifications, not status messages

A2UI bridges this gap. The agent describes *what it wants to show* in structured text. A2UI renders it.

#### Shadow Protocol (what the agent reads — i.e., feedback from its own UI)

```
[A2UI:FEEDBACK] dashboard:home-automation
  Rendered: 3 cards, 1 chart, 2 action buttons
  User interactions in last 1h:
    - Card "Thermostat" clicked → expanded
    - Button "Turn off lights" pressed → executed
    - Chart "Energy usage" zoomed to last 24h
  Currently visible to user: yes
  User has not interacted in 12 minutes

[A2UI:STATE] form:package-delivery
  Fields: 4 total, 3 filled, 1 empty
  User filled: recipient_name, address, phone
  Empty: delivery_instructions
  User has been on this form for 3 minutes
  Predicted completion: likely (cursor is in the empty field)
```

#### Control Protocol (what the agent commands)

```
A2UI:RENDER <ui-description> → display to user
A2UI:UPDATE <component-id> <changes>
A2UI:NOTIFY <message> priority=<low|normal|urgent> style=<banner|toast|modal>
A2UI:FORM <form-definition> → display form, await completion
A2UI:CHART <data> type=<line|bar|pie> title=<string>
A2UI:DASHBOARD <layout-definition>
A2UI:ALERT <message> actions=[<action-label>, ...]
A2UI:DISMISS <component-id>
```

#### UI Description Language (UIDL)

The agent writes structured text that A2UI interprets into native UI:

```yaml
# Example: Home automation dashboard
A2UI:RENDER
  type: dashboard
  layout: grid(2x2)
  title: "Home Status"
  
  cards:
    - id: thermostat
      title: "Living Room"
      content: "68°F — Heating"
      accent: warm
      action: tap → expand → show_controls(thermostat)
    
    - id: security
      title: "Security"
      content: "All clear"
      accent: ok
      detail: "Front door: locked | Back door: locked | Cameras: 3 active"
    
    - id: energy
      title: "Energy Today"
      chart:
        type: sparkline
        data: [12, 15, 14, 18, 22, 19, 17]
        trend: +8% vs yesterday
    
    - id: deliveries
      title: "Deliveries"
      content: "1 package on porch"
      action: tap → show_vision_snapshot(front-door)

  notifications:
    - "Front door camera: person approaching" (2min ago, unread)
```

#### Adaptive Rendering

A2UI adapts to the display surface:

| Surface | Rendering Strategy |
|---------|-------------------|
| Phone notification | Title + 2 lines + action buttons |
| Smart watch | Single line + tap action |
| Desktop widget | Dashboard card |
| Full screen | Complete dashboard |
| CLI/terminal | ASCII art rendering |
| Voice (TTS) | Spoken summary |
| Email | Formatted HTML |

The agent doesn't need to know the surface. It describes intent; A2UI renders appropriately.

```yaml
# Same description, different surfaces:

# Phone notification:
  📦 Package on porch — View | Dismiss

# Smart watch:
  📦 Package arrived

# Desktop widget:
  ┌─────────────────────┐
  │ 🏠 Home             │
  │ 68°F 🔒 ⚡+8% 📦1  │
  └─────────────────────┘

# Terminal:
  [HOME] Temp: 68°F | Security: OK | Energy: +8% | Packages: 1

# Voice:
  "Your home is at 68 degrees and heating. All doors are locked.
   There's one package on the front porch from today."
```

#### Abstraction Levels

| Level | Agent Specifies | A2UI Decides |
|-------|----------------|-------------|
| L1 | Intent only ("show status") | Layout, components, styling |
| L2 | Content + rough layout | Styling, interaction details |
| L3 | Full UIDL | Platform-specific rendering only |
| L4 | Pixel-perfect spec | Nothing (agent drives layout directly) |

Default: L1-L2. Let A2UI handle the design work. Agents shouldn't be UI designers.

#### Policy Boundaries

- **Display surface:** Agent cannot choose which surface to display on — the system routes based on user context and preferences.
- **Interrupt level:** `urgent` alerts require justification. `modal` overwrites require `ui:modal` permission.
- **Content filtering:** No raw HTML/JS injection. All rendering goes through the UIDL interpreter.
- **Form data:** User-submitted form data flows back through the policy engine before reaching the agent. Sensitive fields (passwords, SSNs) are masked.

---

## Integration Architecture

### Sense Coordination

Senses don't operate in isolation. A cross-sense correlator fuses inputs:

```
[CORRELATION] Event at 14:32:12
  VISION:front-door   → Person approaching, dark jacket, carrying bag
  SONAR:front-door    → 2 knocks + doorbell ring
  SONAR:living-room   → TV audio stopped, person on couch looking up
  
  Fused assessment: Visitor at front door. Household member aware.
  Suggested action: Show front-door camera feed to occupant? [A2UI prompt]
```

### Event Pipeline

```
Raw Input → Sense Module → Text Shadow → Event Classifier → Priority Queue → Agent
                                                         ↓
                                                    Cross-Sense Correlator
                                                         ↓
                                                    Long-Term Memory
```

### OpenShell Policy Integration

Each sense module integrates with OpenShell's policy engine at three points:

1. **Gate (pre-action):** Before any sensory action, the policy engine checks permissions.
   ```
   Agent requests: VISION:SNAPSHOT front-door
   Policy check:  vision:capture on camera:front-door → ALLOWED
   Action proceeds → shadow generated → delivered to agent
   ```

2. **Filter (post-shadow):** After generating a shadow, sensitive content is redacted.
   ```
   Vision generates: "Person identified as Alice Chen..."
   Policy filter: vision:identify → DENIED
   Shadow redacted: "Person (identity redacted)..."
   ```

3. **Audit (log):** All sensory actions are logged for review.
   ```
   [AUDIT] 14:32:12 | VISION:DESCRIBE front-door | agent:main | result:shadow-delivered
   [AUDIT] 14:32:12 | SONAR:CLASSIFY front-door | agent:main | result:knock-detected
   [AUDIT] 14:32:13 | PUPPETEER:LOOK | agent:main | result:desktop-described
   ```

### Policy Schema

```yaml
senses:
  manus:
    fs:
      read: true              # default: allow
      write: false            # default: deny
      paths:
        allow: ["~", "/tmp"]
        deny: ["~/.ssh", "~/.gnupg"]
    api:
      domains:
        allow: ["api.weather.gov", "github.com"]
      rate_limit: 60/min
    device:
      read: true
      control: false

  playwright:
    browser:
      domains:
        allow: ["*"]
        deny: ["banking.*", "crypto.*"]
      forms: confirm          # require confirmation for form submission
    app:
      clipboard: deny
      keys:
        deny: ["ctrl+alt+del", "ctrl+alt+f*"]
    screen:
      capture: true

  puppeteer:
    inherits: playwright      # same base policies
    dangerous_rooms:
      - pattern: "password manager"
        action: block
      - pattern: "banking"
        action: redact

  vision:
    cameras:
      "front-door": { read: true, ptz: false }
      "living-room": { read: true, ptz: false }
      "bedroom-*": { read: false }     # blocked entirely
    identify: false
    record: false
    alerts: true

  sonar:
    mics:
      "kitchen": { listen: event-triggered }
      "front-door": { listen: continuous }
    speech_to_text: false
    identify: false

  a2ui:
    surfaces: auto            # system chooses
    interrupt_level: normal   # normal | urgent
    modal: false              # no modal overwrites without permission
    content: filtered         # no raw HTML/JS
```

---

## Implementation Notes

### Module Independence

Each sense module is independently deployable. An agent doesn't need all six. A server agent might only have Manus + Playwright. A home automation agent might have Vision + Sonar + A2UI. A coding agent might have Manus + Puppeteer.

### Transport Layer

All sense communication happens over structured text streams. The transport can be:
- Direct function calls (same process)
- IPC (same machine, different process)
- Network (remote machine via encrypted channel)

The protocol is text-based regardless of transport:

```
→ SENSE:VISION:DESCRIBE camera=front-door level=L2
← SHADOW:VISION front-door @14:32:07
  Scene: Exterior, daytime, overcast
  ...
```

### Caching and Freshness

Senses maintain a freshness model:
- **Hot:** Real-time stream (cameras, microphones) — always current
- **Warm:** Polled state (APIs, file system) — current within poll interval
- **Cold:** Cached snapshots (screenshots, historical data) — may be stale

The agent can request freshness:
```
MANUS:FS:LIST /path freshness=hot  → forces a fresh read
VISION:DESCRIBE cam freshness=warm → last snapshot if <30s old
```

### Error Handling

Sensory failures are reported as text too:

```
[VISION:ERROR] front-door — Camera unreachable (timeout after 5s)
  Last known state: 2h ago (empty porch, overcast)
  Recovery: automatic retry in 30s
  Agent action: None required, will notify on recovery

[MANUS:ERROR] API call failed — https://api.example.com
  HTTP 503 Service Unavailable
  Retry: 3 attempts failed
  Suggested: Try again in 5 minutes or use cached data (12min old)
```

Errors are shadows too. The agent never needs to handle raw exceptions.

---

## Philosophical Note

> *Plato's Allegory of the Cave describes prisoners who mistake shadows for reality. Our architecture embraces this intentionally. The agent's reality IS the shadows — structured, compressed, semantic text. The translation layer IS the reality engine.*
>
> *The prisoner isn't disadvantaged. The prisoner has an interpreter who says "that shadow is your friend at the door" instead of making them decode flickering light patterns themselves.*
>
> *The cave isn't a limitation. The cave is the interface.*

---

## Naming

**Plato** — for the Allegory of the Cave, where shadows on a wall are the only known reality.

**Manus** — Latin for "hand." The manipulator.

**Playwright** — one who writes what happens on stage. The director.

**Puppeteer** — one who pulls strings to make the show happen. The unseen hand.

**Vision** — sight. The most obvious sense.

**Sonar** — hearing by echo. Navigation through sound.

**A2UI** — Agent to UI. The shadow-play projected outward.

---

*Part of the OpenConstruct project.*
