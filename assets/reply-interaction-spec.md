# Reply · Interaction Spec

**Author**: Ziying (Design)
**For**: Piper (SwiftUI Implementation)
**Date**: 2026-06-25
**Companion files**: `reply_demo_v4.html`, `reply_demo_scene2_v2.html`

This document translates the design intent from the two HTML demos into engineering-readable specifications. The HTMLs show the **feel**; this document defines the **behavior**.

---

## 0. Core Design Philosophy

**One input box, two surfaces of AI presence**:
- The same input box handles user-to-human messaging AND user-to-AI communication
- Two distinct visual languages, never mixed:
  - **Halo** (rainbow glow around input box) = user-initiated AI conversation
  - **Pill** (inline AI suggestion inside input box) = AI-initiated ambient suggestion

**Three core principles**:
1. Reply must not introduce a second chat window — AI lives inside the host app's input box visually
2. AI's state (idle / listening / generating / done) must be glanceable in under 0.5 seconds
3. User intent precedes AI action — Reply never sends, only drafts

---

## 1. Activation Model

### Keyboard triggers (Option key)

| Trigger | When | Action |
|---|---|---|
| **Single Option** | Idle, empty input | Enter `ai-mode` (halo on, user types instruction to AI) |
| **Single Option** | In `ai-mode` | Exit to idle (halo off, instruction text preserved in buffer) |
| **Double Option** | Idle, empty input | Trigger `generate-fresh` (3 draft variants) |
| **Double Option** | Idle, has user text | Trigger `generate-polish` (3 polished variants of user's text) |
| **Double Option** | In `ai-mode` with instruction | Trigger `generate-from-instruction` (3 deep variants) |

### Double-click detection
- Threshold: 280ms between Option key presses
- Implementation: timer-based debounce (see `onOptionPress` in demo)

### Buffer model
Two independent text buffers preserve user state across mode toggles:
- `buffers.reply`: text shown in idle mode (user's draft to recipient)
- `buffers.ai`: text shown in ai-mode (user's instruction to AI)
- Toggling between idle ↔ ai-mode swaps which buffer is visible
- Buffers persist until explicit reset

---

## 2. State Machine (Demo 1 · User-Initiated)

### States

```
idle
  ├─ single Option ───────────────→ ai-mode
  ├─ double Option (empty) ───────→ generating-fresh
  └─ double Option (has text) ────→ generating-polish

ai-mode
  ├─ single Option ───────────────→ idle (buffers preserved)
  └─ double Option (has instr) ───→ generating-from-instruction

generating-fresh
  └─ auto (3.1s) ──────────────────→ complete-fresh

generating-polish
  └─ auto (2.4s) ──────────────────→ complete-polish

generating-from-instruction
  └─ auto (3.6s) ──────────────────→ complete-from-instruction

complete-* (any)
  ├─ ← / → ───────────────────────→ cycle draft variants (0-2)
  ├─ Enter ───────────────────────→ sent state
  ├─ Escape ──────────────────────→ idle (reset)
  ├─ single Option ───────────────→ idle OR ai-mode (state-dependent)
  └─ double Option ───────────────→ regenerate (same mode)
```

### State-specific visuals

| State | Halo | Box border | Input content | Caret | Hint row |
|---|---|---|---|---|---|
| `idle` | hidden | hairline gray | placeholder OR user's reply buffer | active | "⌥⌥ generate/polish · ⌥ talk to Reply" |
| `ai-mode` | visible, breathing | transparent | placeholder "Tell Reply…" OR ai buffer | active | "⌥⌥ ask Reply · ⌥ close" |
| `generating-*` | visible, breathing | transparent | shimmer text (3 phases) | inactive | empty |
| `complete-*` | hidden | subtle shadow | one of 3 variants | inactive | "← → swap · ⏎ send · ⌥⌥ regenerate" |

### Halo visual spec

- **Geometry**: positioned 10px outside the input box on all sides, 32px border-radius
- **Fill**: conic gradient with Apple Intelligence rainbow colors (see exact stops below)
- **Blur**: 22px filter blur
- **Animation**: NO rotation. Static gradient with 3-second opacity breathing (0.6 → 0.8 → 0.6)
- **Trigger**: opacity 0 → 0.7 over 0.6 seconds when entering ai-mode or generating-*

**Exact rainbow stops** (for SwiftUI gradient implementation):
```
0%   #FF3B7A  (pink)
15%  #FF8A3D  (orange)
30%  #FFC83D  (amber)
45%  #65D96A  (green)
60%  #3DB9FF  (blue)
75%  #8A5BFF  (purple)
90%  #FF3DD2  (magenta)
100% #FF3B7A  (pink, wrap)
```

Gradient angle: starts at 130° (so warm colors appear top-left, where eye lands first)

### Pagination spec (complete-fresh state)

- Position: absolute bottom-right inside input box, 6px from bottom, 14px from right
- 3 dots (5px each, 5px gap), active dot black + 1.25× scale
- "1 / 3" label next to dots, 10px font
- Visibility: fade in (0.3s) on entering complete-* state

### Shimmer text spec (generating states)

- Apply to `.input-text` during generating-* states
- Linear gradient sweeps across text: light gray → dark → light gray
- Animation: 1.6s linear infinite, gradient moves right-to-left
- Implementation: `background-clip: text` with animated `background-position`

---

## 3. State Machine (Demo 2 · AI-Initiated Pills)

### Pill lifecycle

```
context detected → wait 2.5s (let user read message)
  → pill appears (animated slide-in + opacity)
  → 6.0s at full opacity (calm presence)
  → 2.0s gentle opacity fade (0.8s 0% → 100% in CSS terms)
  → auto-dismiss → recall flower icon appears (soft dismiss)

OR user clicks ✓ → accept handler runs (scenario-specific)
OR user clicks ✕ → hard dismiss (no recall flower)
OR user hovers over pill → fade timer cancelled, opacity restored to 1.0
OR user clicks inside box (not on pill buttons) → soft dismiss (with recall)
```

### Four pill categories

| Category | Priority | Trigger | Pill text | Accept handler |
|---|---|---|---|---|
| **Intent perception** (意图感知) | P0 | Message contains time word | "Bei 在约下午 2 点 — 加日历?" | Insert calendar event, show "✓ 已加到日历…" for 2.4s |
| **Relationship perception** (关系感知) | P0 | Owed ≥ 2 days, no reply yet | "Bei 等你 3 天 — 起草一个?" | Enter generating-fresh flow (Demo 1) |
| **Attachment perception** (附件感知) | P1 | Message contains "急/asap/deadline" + attachment | "急 + PDF — 帮你读完起草?" | Read PDF → enter generating-from-instruction |
| **State perception** (状态感知) | P2 | User typing pause ≥ 5s | "需要 Reply 帮你接下去?" | Inline ghost text continuation, Tab/click ✓ to accept |

### Four discipline rules (must hold)

All four must pass for a pill to appear:
1. **Rate limit**: max 1 pill per conversation per 5 minutes
2. **Mute on rejection**: if user dismisses same pill category 3 times, mute that category for 7 days
3. **No-interrupt-while-typing**: do not show pill if user has typed in input box within last 3 seconds (exception: state-perception category, which IS triggered by typing pause)
4. **Trust threshold**: higher confidence threshold for unfamiliar contacts. Cold contacts need stronger signals than known coworkers.

### Pill visual spec

- **Container**: appears inside input box, replacing placeholder
- **Background tint** on input box: `linear-gradient(135deg, #FFF5F0 0%, #FBF1FC 100%)`, very subtle
- **Border**: input box border changes to `rgba(216, 90, 48, 0.25)` when pill is shown
- **Layout** (left to right):
  - Flower icon (18×18, rainbow, see spec below)
  - Pill text (14px, line-height 1.4)
  - Accept button (✓ icon, 28×28 button, hover bg)
  - Dismiss button (✕ icon, 28×28 button, hover bg)
- **Entry animation**: `opacity 0 → 1 + translateY 4px → 0` over 450ms ease
- **Exit animation**: `opacity → 0` over 600ms

### Flower icon spec

6-petal flower, each petal a different Apple Intelligence rainbow color:

```
Center: (16, 16) in 32×32 viewBox
Each petal: 6px radius circle at 85% opacity
Positions (top → clockwise):
  (16,  8)  #FF3B7A  pink
  (22.9, 12) #FF8A3D  orange
  (22.9, 20) #FFC83D  amber
  (16, 24) #65D96A  green
  (9.1, 20) #3DB9FF  blue
  (9.1, 12) #8A5BFF  purple
Center dot: 3.5px white at 70% opacity (gives the flower a "highlight")
```

Used at three sizes:
- 30px: in v3/v4 active state inside input box
- 18px: in pills
- 18px (faded to 40% opacity): recall icon after soft dismiss

### Soft vs. hard dismiss distinction

This is a core UX principle, not just a visual detail:

| Dismiss type | Trigger | Aftermath |
|---|---|---|
| **Hard dismiss** | User clicks ✕ button explicitly | Pill fades out completely. **No recall icon.** Signal: "I told you no." |
| **Soft dismiss** | Auto-dismiss after 8s OR user clicks elsewhere in box | Pill fades, but a small flower icon appears in box's bottom-right corner (40% opacity). Clicking flower brings pill back. Signal: "Maybe I missed it." |

Implementation: pass dismiss reason as parameter to dismiss handler, branch on type.

---

## 4. Attachment Reading Decision Tree

When the conversation context contains link/PDF/video/image attachments, AI's default behavior:

```
Has attachment in context?
  ├─ User triggers double-Option (quick reply, empty input)
  │     → Do NOT read attachment. Generate quick acknowledgment drafts.
  │     → Token-saving default.
  │
  ├─ User has typed draft that references the attachment
  │     → Read attachment (cheap acknowledgment of user's intent).
  │
  └─ User in ai-mode and instruction explicitly mentions the attachment
        → Read attachment fully, use in generation.
        → Explicit consent path.
```

**Special override**: Attachment-perception pill (Demo 2 P1) bypasses this — when "urgent + attachment" is detected, AI proactively reads even before user triggers.

---

## 5. Open Implementation Questions (please decide, Piper)

These are decisions I (designer) cannot make — they depend on SwiftUI/macOS constraints I don't know:

1. **Input box tracking strategy**:
   - OCR continuously scans for input box? (high cost, works everywhere)
   - AX (Accessibility) tree query? (cheap, but fails on WeChat / canvas-rendered apps)
   - Hybrid: AX-first, OCR fallback?
   - Affects which apps Reply can support.

2. **Halo rendering technique in SwiftUI**:
   - Metal shader for conic gradient + blur?
   - `CAGradientLayer` + `UIVisualEffectView` blur?
   - SVG filter (via WebView)?
   - Performance budget: must hit 60fps even with halo + breathing animation.

3. **Global Option-key hook**:
   - How to listen to Option key globally without modifier conflict?
   - `NSEvent.addGlobalMonitorForEvents`?
   - Need to ensure it doesn't block normal Option-key behavior (special character input).

4. **Pill trigger detection**:
   - Time word detection: regex? NER model? When does the detection run — on every new message, or on a scheduled scan?
   - Owed tracking: pull from existing Piper-built owed ledger.
   - Urgency detection: keyword list ("急/asap/紧急/deadline/截止")? Or LLM classification on incoming messages?
   - Typing-pause detection: 5-second timer on keystroke events in monitored input box.

5. **Draft generation data flow**:
   - Reuse existing `LLMChatProvider`?
   - Where does the "3 variants" come from — one API call returns array, or three sequential calls?
   - Caching strategy for re-generate (double-Option on complete state).

6. **PDF reading pipeline** (for attachment-perception pill):
   - Which library for PDF text extraction?
   - Token budget for "deep read" — fixed cap or based on PDF size?
   - Fallback if PDF is image-based / encrypted?

7. **Dark mode**:
   - Halo colors need adjustment in dark background? Or rainbow gradient works equally well?
   - Pill tint background needs dark variant?

---

## 6. Non-Goals (Explicitly Out of Scope This Iteration)

To prevent scope creep:

- **No auto-send.** Reply never commits — user always presses Enter to send.
- **No separate AI window.** All AI interactions happen in or around the host input box.
- **No conversation history UI.** Reply doesn't show a chat thread with AI; only state-machine view.
- **No multi-language UI switching this iteration.** Chinese-first; English follows later.
- **No mobile.** macOS-only.
- **No voice in this spec.** PTT voice is separate work (covered elsewhere).

---

## 7. Suggested Implementation Order

Ranked by ratio of (user value) ÷ (engineering effort):

| Order | Feature | Why first / Why later |
|---|---|---|
| 1 | Demo 1 idle ↔ ai-mode toggle + halo | Highest-leverage. Defines core interaction language. |
| 2 | Demo 1 generating-fresh + complete-fresh + ←/→ | Validates the "3 variants + swap" pattern. |
| 3 | Demo 1 generating-polish + generating-from-instruction | Reuses #2 infrastructure. |
| 4 | Demo 2 intent-perception (calendar pill) | Lightest pill, validates pill pattern. |
| 5 | Demo 2 relationship-perception (owed pill) | Connects to Piper's existing owed work. |
| 6 | Demo 2 attachment-perception (urgent PDF) | Validates attachment reading pipeline. |
| 7 | Demo 2 state-perception (typing pause) | Trickiest detection, lowest priority. |

---

## 8. How to Use This Document with AI Coding Assistants

This spec is structured to be paste-able into Claude Code / Codex with the two HTML demos. Suggested prompts for the AI:

**For initial scaffolding**:
> "Read the attached spec.md and the two HTML demos. Generate SwiftUI scaffolding for the Reply input box overlay. Implement the state machine in section 2 first. The HTML files show the visual target; reproduce the feel in SwiftUI."

**For specific features**:
> "Implement the halo visual per section 2 'Halo visual spec'. Match the rainbow stops and breathing animation. Use [Metal shader / CAGradientLayer / whatever you chose]."

**For decisions**:
> "Help me decide question 1 in section 5 of spec.md. Given Reply runs on macOS 14+ and needs to support WeChat (which has poor AX), recommend an approach."

---

## Appendix: Glossary

- **Halo**: the rainbow gradient glow around the input box that signals AI is active (user-initiated)
- **Pill**: the inline suggestion card inside the input box that signals AI is proposing (AI-initiated)
- **Soft dismiss**: pill goes away but flower icon stays for recall
- **Hard dismiss**: pill goes away completely, no recall
- **Recall flower**: small 18px flower icon at input box bottom-right after soft dismiss; clicking re-shows pill
- **Buffer**: persistent text storage that distinguishes user's reply text from user's AI instruction text
- **Shimmer**: the gradient text animation that signals "AI is thinking" during generating states
- **Owed**: outgoing message debt from existing Piper-built tracking ledger
- **Capture-to-paste**: Reply's core feature of reading host app context via OCR and pasting back drafts

---

*End of spec. Questions to: Ziying.*
