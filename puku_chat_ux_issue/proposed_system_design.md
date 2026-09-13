# Puku Chat — Next-Generation Chat UX: System Design Proposal

> **Scope**: Investigation + architecture proposal for richer, structured, interactive AI responses in Puku Chat (visual charts, content cards, comparison cards, step-by-step timelines, structured forms, follow-up action buttons).
> **Audience**: Puku AI product, design, and engineering team.
> **Status**: Design proposal. **No implementation in this pass.** Awaiting team review.

---

## 1. Executive summary

Puku Chat is a single-page Next.js web app that today produces **flat Markdown strings** as AI responses. The frontend renders them through `react-markdown` + `remark-gfm`, with one custom renderer for code blocks and one for thought/reasoning. There is no notion of "blocks," "parts," or a component registry. There is no charting library. There is no form library. There is no client-side schema validator.

The attached screenshots demonstrate a richer experience: inline illustrative charts, comparison cards with badges, numbered timeline cards, structured input forms (radio + text + textarea + CTA), and follow-up action buttons.

**The recommendation** is a **hybrid Markdown + typed parts** design:

1. The assistant message body becomes an **ordered array of typed parts** — `markdown`, `chart`, `card`, `timeline`, `form`, `actions` — instead of one Markdown string.
2. The existing `MarkdownContent` path keeps working untouched for legacy text/code, so existing conversations and existing API behavior stay compatible.
3. The streaming parser learns to recognize a small, well-defined `data: {"type":"...", ...}` envelope (in addition to the existing `delta.content` string), and fans it into a per-message parts buffer.
4. A new `parts` renderer (`MessageParts.js`) dispatches each part to a dedicated component. New components (`RichChart`, `Card`, `Timeline`, `Form`, `Actions`) sit alongside the legacy `MarkdownContent` rather than replacing it.
5. Forms and actions submit back through the existing `/v1/chat/conversations/:id/messages` endpoint with a structured payload (no new auth, no new protocol), so the server contract evolves incrementally.

This keeps the existing Markdown experience intact while adding structured content where it adds the most value. It avoids a rewrite. It introduces only one small new dependency family (a lightweight charting library) and zero new dependencies for forms (we render the existing Tailwind primitives).

The roadmap below delivers a usable structured-response experience in **four short phases** (foundation → core rich responses → interactive forms → refinement), each independently shippable.

---

## 2. Screenshot analysis — what Puku should learn

| File | Observed pattern | Product take-away |
|---|---|---|
| `screenshot-1.png` | A structured **form** ("Let's design your winning model"): numbered questions, radio buttons for choices, single-line text input, multi-line text area, a primary CTA ("Design my optimized neural network"). | The AI should be able to collect multiple required pieces of information in a single, organized form rather than asking open-ended follow-ups. |
| `screenshot-2.png` | An inline **chart card** ("What an efficient model looks like"): illustrative accuracy–parameter tradeoff. Curve, labeled axes, on-chart annotations ("Compact winner", "Diminishing returns"), and an explicit disclaimer ("The curve is conceptual, not measured data"). | Charts are valid when they aid comprehension. Real vs illustrative must be clearly labelled. Annotations live on the chart, not in a separate paragraph. |
| `screenshot-3.png` | **Dataset type cards** with small left thumbnails (image / tabular / text) — each with a section title, body, and "Strong candidates:" highlight line. | Pairing short prose with a small visual is more scannable than prose alone. Sections must have predictable structure (heading → explanation → "strong candidates"). |
| `screenshot-4.png` | A **numbered timeline** ("Final competition plan", 6 steps): each step has a circular number badge, bold title, and supporting paragraph; a vertical line ties them together. | Long sequential plans should render as vertical timelines. Number badges make order obvious. The vertical connector is a small but high-value detail. |
| `screenshot-5.png` | A **comparison card** ("Compact MLP search space") with 5 model rows. Each row has name + a short formula + a colored badge ("Smallest baseline", "Baseline", "Recommended", "Comparison", "Upper baseline"). Footer note: "These are starting candidates, not guaranteed winners." | Multi-option responses should use a card with structured rows. Badges must be semantically meaningful (recommended vs comparison vs baseline) and consistent. |
| `screenshot-6.png` | A second **numbered timeline** (7 steps) "What I would actually do to win" — same component as screenshot 4, reused across two places. | Reusability matters: timeline + card + form components should be invoked from many places in a single response, not be one-offs. |
| `screenshot-7.png` | Follow-up actions: bulleted explanation, a focused question, then 3 buttons (one primary black, two outlined). | Buttons should describe what will happen next ("Use nominal dataset — build the notebook"), not just "Continue". Primary action gets the most prominent style. |

**Cross-cutting takeaways:**

- **Cards, timelines, and forms are reusable building blocks** — the same shape appears multiple times in the same conversation.
- **Visual hierarchy is consistent**: card chrome (border + background) is the same in screenshots 2, 4, 5; numbered badges share one style; action buttons share one style.
- **Badges carry semantic weight** (recommended, baseline, comparison) — they are not decoration.
- **Each visual element earns its place**: charts are illustrative (labelled), buttons have explicit outcomes, forms have placeholders, timelines have a connecting line.

Puku's current system can already render some of this (cards via Markdown blockquotes, timelines via ordered lists, badges via Markdown bold), but the screenshots demonstrate that **a purpose-built component is meaningfully better** than Markdown for these patterns. The proposal below adds those purpose-built components incrementally.

---

## 3. Current architecture investigation — verified from the code

### 3.1 Stack (verified from `package.json`, `next.config.mjs`, `wrangler.jsonc`)

- **Next.js 16.3.4** (App Router), **React 19.2.8**, **Tailwind v4** (CSS-first config in `app/globals.css`).
- **react-markdown 10** + **remark-gfm 4** for Markdown; **react-syntax-highlighter 16** (Prism, `vsc-dark-plus`) for code.
- **lucide-react 1.40.0** for icons.
- Deployed to **Cloudflare Workers** via `@opennextjs/cloudflare` (`wrangler.jsonc`, `compatibility_date: "2026-09-11"`).
- **No** charting, form, validation, animation, or UI primitives library beyond Tailwind + lucide.

### 3.2 Repo map relevant to chat

| Area | Key files |
|---|---|
| Chat routes | `app/new/page.js`, `app/new/layout.js`, `app/share/[shareId]/page.js` |
| Chat shell | `components/chat/ChatWorkspace.js`, `components/chat/MessageList.js`, `components/chat/MessageItem.js`, `components/chat/ChatInput.js` |
| Markdown rendering | `components/chat/MarkdownContent.js`, `components/chat/CodeBlock.js` |
| Side-channel content | `components/chat/ThoughtProcess.js`, `components/chat/ArtifactCard.js`, `components/chat/ArtifactsPanel.js`, `components/chat/ToolEventCard.js` (orphaned — see §3.5) |
| State | `hooks/useChatSession.js` (572 lines, single owner of all chat state), `contexts/AuthContext.js`, `hooks/useConversationActions.js` |
| Streaming | `services/chat/messages.js` (`sendMessage`, `parseStreamDelta`, `cancelTurn`) |
| REST surface | `services/chat/conversations.js`, `services/chat/artifacts.js`, `services/chat/connectors.js`, `services/chat/projects.js`, `services/chat/shares.js`, `services/chat/skills.js`, `services/chat/usage.js` |
| Helpers | `utils/messageContent.js#splitMessageContent`, `utils/codeBlocks.js`, `utils/artifactSync.js`, `utils/artifactKind.js` |
| Tests | `tests/playwright/` (5 specs, fixtures in `fixtures/api-mocks.ts` with the **SSOT for the SSE wire format** in `buildSseBody()`) |
| Design tokens | `app/globals.css` (Puku dark palette: `#100D1D` bg, `#F3F1EB` text, `#A4ABFF` accent, `#201C59` primary button) |

### 3.3 Current message shape

A user message is `{ id, role: "user", content: "string" }`.
An assistant message committed by `commitAssistantMessage` in `useChatSession.js` is:

```js
{
  id, role: "assistant",
  content: streamContentRef.current,           // string (markdown)
  reasoning: streamReasoningRef.current,       // string (markdown, optional)
  thinking: streamReasoningRef.current,        // alias
  toolEvents: streamToolsRef.current,          // array (never rendered)
  artifacts: streamArtifactsRef.current,       // array of artifact descriptors
}
```

There is **no `parts` array, no `blocks`, no typed content**, no component registry. Everything inside `content` is opaque Markdown.

### 3.4 Streaming transport

`POST https://chat.api.dev.puku.sh/v1/chat/conversations/:id/messages` returns a response whose **body is an SSE-like stream** of `data: <json>` lines ending with `data: [DONE]`, plus the `x-puku-turn-id` response header. The body is read via a manual `fetch` `ReadableStream` reader in `services/chat/messages.js#sendMessage` (not `EventSource`). Each chunk is fed to `parseStreamDelta(chunk)` and the deltas accumulate into refs in `useChatSession.js`:

```js
streamContentRef   += delta.content
streamReasoningRef += delta.reasoning_text
streamToolsRef     = [...streamToolsRef, delta.tool_execution]
streamArtifactsRef = [...streamArtifactsRef, delta.artifact]
```

Ref state is flushed to React state **once per animation frame** via `flushStreamUi()` to avoid render storms. On `[DONE]`, `commitAssistantMessage` builds the final message and `syncConversationState` refetches the conversation so the server-truth message (with full history) lands in `messages`.

### 3.5 Notable absences that the design must address

1. **No component registry.** Only `code` and `pre` are overridden in `MarkdownContent`. No `<chart>`, `<button>`, `<card>`, or `<form>` exists in the rendering surface.
2. **`ToolEventCard.js` is orphaned.** It exists (a 16-line indicator) but `MessageList` never renders it — `streamingTools` is collected in `useChatSession` and discarded. The team scaffolded tool/structured rendering and stopped before finishing. This is useful prior art (style + intent) for the new design.
3. **Five parallel streaming refs** (`streamContentRef`, `streamReasoningRef`, `streamToolsRef`, `streamArtifactsRef`, plus per-render `setStreaming*` state) will need to collapse into a single `streamPartsRef` if we adopt parts.
4. **No backend changes visible here.** The upstream at `chat.api.dev.puku.sh` is opaque from this repo (only the SSE wire format is implied). Any new envelope type must be added on the server too; the proposal below assumes the team controls the upstream and treats the format change as a server task out of scope for this design pass.
5. **Like/Dislike buttons are unwired** in `MessageItem` — the buttons render but `onClick` is omitted. Useful prior art for action wiring.
6. **The share view (`app/share/[shareId]/page.js`) also uses `MarkdownContent`** — any new renderer must work for shared read-only views too.
7. **Cloudflare Workers target** rules out server-side libraries that depend on Node built-ins; any new chart library must be browser-side bundle only.

---

## 4. Current limitations (only the ones actually blocking the goal)

| Capability needed | Today | Gap |
|---|---|---|
| Render charts | Not supported | No charting library; no data shape; no renderer. |
| Render labeled cards | Possible only via blockquote Markdown | No badges, no consistent chrome, no semantic row grouping. |
| Render numbered timelines | Possible only via Markdown ordered lists | No numbered badge style, no vertical connecting line, no card framing. |
| Render forms (radio, text, textarea, submit) | Not supported | No form component, no submit-back protocol, no validation. |
| Render action buttons (chat continuation) | Not supported | No button-as-message primitive. |
| Structured response envelope from server | Not supported | Server emits Markdown strings; no typed parts. |
| Frontend dispatcher | Not supported | Only `code`/`pre` are dispatched in `MarkdownContent`. |
| Streaming of partial structured content | Not supported | Streaming parser expects `content` string; would need to handle partial JSON parts. |
| Schema validation of AI-generated content | Not supported | No `zod`/`valibot`; today the frontend trusts whatever string the server emits. |

What Puku **already has** that this proposal reuses:

- A working Markdown renderer with strong theme tokens (`app/globals.css`).
- A robust streaming pipeline (RAF-batched, abortable).
- A working `parts` precedent in `utils/messageContent.js#splitMessageContent` (which already accepts arrays of `{type: "thinking"|"reasoning"}` parts). The "dispatcher pattern" already exists for reasoning — we extend it.
- Existing `ToolEventCard.js` shows the team already planned for typed inline content.
- A clean auth boundary: all chat traffic flows through one `Bearer` call (`pukuFetch`), so adding structured content does not require new auth.

---

## 5. Proposed system design

### 5.1 Architecture overview

```
┌────────────────────────────────────────────────────────────┐
│  Upstream LLM API  (chat.api.dev.puku.sh)                   │
│  Emits SSE: data: {type:"delta"|"part_start"|"part_delta"|"part_end", ...} │
└──────────┬─────────────────────────────────────────────────┘
           │ fetch + ReadableStream reader
┌──────────▼─────────────────────────────────────────────────┐
│  services/chat/messages.js                                  │
│  • sendMessage, parseStreamDelta  (UNCHANGED wire format)   │
│  • NEW: parseStreamPart(delta)  → typed-part delta          │
└──────────┬─────────────────────────────────────────────────┘
           │ onChunk
┌──────────▼─────────────────────────────────────────────────┐
│  hooks/useChatSession.js                                    │
│  • REPLACE 5 parallel refs with streamPartsRef: Part[]     │
│  • RAF-batched flush (existing pattern preserved)           │
│  • commitAssistantMessage stores parts[], not content str  │
└──────────┬─────────────────────────────────────────────────┘
           │ messages[].parts
┌──────────▼─────────────────────────────────────────────────┐
│  components/chat/MessageList → MessageItem → MessageParts │
│  • MessageParts.js: dispatcher that maps part.type → cpt │
│  • Existing MarkdownContent kept for "markdown" parts     │
│  • New components: RichChart, CardView, TimelineView,     │
│    FormView, ActionsView, CalloutView, ImageView           │
└──────────┬─────────────────────────────────────────────────┘
           │ user interacts
┌──────────▼─────────────────────────────────────────────────┐
│  Form submit / Action click                                  │
│  • validate payload (client-side schema)                    │
│  • call existing sendMessage(...) with structured content   │
│  • server merges into conversation                          │
└────────────────────────────────────────────────────────────┘
```

Mermaid (system-level):

```mermaid
flowchart TB
  A[User types / picks option / submits form] --> B[useChatSession.handleSend]
  B --> C[sendMessage POST /messages]
  C --> D[SSE stream from chat.api.dev.puku.sh]
  D --> E[messages.js reader loop]
  E --> F[parseStreamDelta / parseStreamPart]
  F --> G[streamPartsRef accumulator]
  G --> H[RAF flush → setStreamingParts]
  H --> I[MessageList renders streaming tail]
  G --> J[onComplete → commitAssistantMessage]
  J --> K[messages[i].parts persisted]
  K --> L[MessageItem renders via MessageParts]
  L --> M{part.type}
  M -->|markdown| N1[MarkdownContent]
  M -->|chart| N2[RichChart]
  M -->|card| N3[CardView]
  M -->|timeline| N4[TimelineView]
  M -->|form| N5[FormView]
  M -->|actions| N6[ActionsView]
  M -->|image| N7[ImageView]
  M -->|callout| N8[CalloutView]
  M -->|unknown| N9[FallbackMarkdown]
```

### 5.2 Response model — `parts[]`

Replace the single `content: string` with an ordered array. Each part is a small typed object. The discriminator is a string `type` so JSON is compact and the parser is cheap.

```ts
type Part =
  | { type: "markdown"; text: string }
  | { type: "chart"; id: string; title?: string; kind: "line"|"bar"|"scatter";
      data: { x: number|string; y: number }[]; xLabel?: string; yLabel?: string;
      annotations?: { x?: number|string; y?: number; label: string; tone?: "info"|"warn"|"good" }[];
      illustrative?: boolean }
  | { type: "card"; title?: string; items: CardItem[]; footer?: string }
  | { type: "timeline"; title?: string; steps: { title: string; body: string }[] }
  | { type: "form"; id: string; title: string; description?: string;
      fields: FormField[]; submit: { label: string; intent: string } }
  | { type: "actions"; prompt?: string; options: { label: string; intent: string; primary?: boolean }[] }
  | { type: "callout"; tone: "info"|"success"|"warn"|"danger"; text: string }
  | { type: "image"; src: string; alt: string; caption?: string };
```

`markdown` parts preserve today's behavior bit-for-bit. All other types are additive. `commitAssistantMessage` builds the message as:

```js
{
  id, role: "assistant",
  parts: streamPartsRef.current,           // Part[] (replaces content)
  reasoning: streamReasoningRef.current,   // unchanged
  toolEvents: streamToolsRef.current,      // unchanged
  artifacts: streamArtifactsRef.current,   // unchanged
  // DEPRECATED but kept for back-compat: content = markdown parts joined by \n\n
  content: streamPartsRef.current.filter(p=>p.type==="markdown").map(p=>p.text).join("\n\n"),
}
```

Keeping `content` populated for legacy readers (the share page, copy-conversation, sync conversation GET) makes the change additive — nothing in the existing pipeline breaks.

### 5.3 Wire format — minimal, additive, backward-compatible

Today, the SSE stream is `data: <JSON>` lines. `parseStreamDelta` reads `choices[0].delta.content`, `delta.reasoning_text`, `delta.tool_execution`, `delta.artifact`. We add **one new top-level field** without changing existing ones:

```
data: {"type":"delta","choices":[{"delta":{"content":"...","reasoning_text":"..."}}]}

data: {"type":"part_start","part":{"type":"chart","id":"c1","title":"..."}}
data: {"type":"part_delta","id":"c1","patch":{"data":[...]}}      // partial chart data
data: {"type":"part_end","id":"c1"}

data: {"type":"part","part":{"type":"form","id":"f1","title":"...","fields":[...],
                              "submit":{"label":"...","intent":"..."}}}

data: [DONE]
```

- `delta` is the legacy shape — preserved.
- `part_start` / `part_delta` / `part_end` allow long parts (charts, large forms) to stream progressively without blocking text deltas.
- `part` is the "shortcut" for fully-formed atomic parts (forms, actions, small cards) — useful when the server doesn't need to stream partial JSON.

The client **never** trusts the wire format directly: every `part` is validated by a Zod-style schema in `utils/partSchema.js` before being stored or rendered. Malformed parts are dropped and rendered as a small error card ("This part could not be rendered") instead of crashing the message.

### 5.4 Frontend rendering — `MessageParts.js` dispatcher

A new file at `components/chat/MessageParts.js`:

```jsx
function MessageParts({ parts, deferHighlight, onPartSubmit }) {
  return (
    <div className="space-y-3">
      {parts.map((p, i) => {
        switch (p.type) {
          case "markdown": return <MarkdownContent key={i} content={p.text} deferHighlight={deferHighlight} />;
          case "chart":    return <RichChart     key={i} part={p} />;
          case "card":     return <CardView      key={i} part={p} />;
          case "timeline": return <TimelineView  key={i} part={p} />;
          case "form":     return <FormView      key={i} part={p} onSubmit={onPartSubmit} />;
          case "actions":  return <ActionsView   key={i} part={p} onPick={onPartSubmit} />;
          case "callout":  return <CalloutView   key={i} part={p} />;
          case "image":    return <ImageView     key={i} part={p} />;
          default:         return <FallbackMarkdown key={i} text={`[unsupported part: ${p.type}]`} />;
        }
      })}
    </div>
  );
}
```

`MessageItem` and `MessageList` replace their `MarkdownContent` and `streamingArtifacts.map(ArtifactCard)` blocks with `<MessageParts parts={message.parts ?? messageToParts(message)} … />`. A small back-compat shim (`messageToParts`) converts a legacy `{content}` message into a single `{type:"markdown", text: content}` part so old conversations keep working.

### 5.5 Interaction design

For each part that supports interaction:

| Part | Interaction | Payload back to AI |
|---|---|---|
| `form` submit | validate required fields → call `useChatSession.handleSendMessage({ structuredSubmit: { partId, intent, values } })` | `{ structuredSubmit: { partId, intent, values: { fieldKey: value } } }` |
| `actions` pick | same `handleSendMessage` path with `structuredAction: { partId, intent }` | `{ structuredAction: { partId, intent } }` |
| `chart` annotation tap (optional, post-MVP) | same as action pick | same |

`useChatSession.handleSendMessage` learns two new optional fields and forwards them inside the existing POST body — no new endpoint, no new auth.

The new components themselves never call fetch — they always go through `useChatSession`. That keeps a single chokepoint for cancel, retry, and history sync.

### 5.6 Streaming and error handling

- **Markdown parts** stream as today: each `delta.content` accumulates into a trailing `{type:"markdown"}` part. When a new `part_start` arrives, the trailing markdown part is **closed** and a new part begins; further deltas open a fresh trailing markdown part.
- **Streaming chart parts**: `part_start` reserves the card slot with a skeleton; `part_delta` updates `data`, `annotations`, etc.; `part_end` finalizes. A RAF batch coalesces updates.
- **Incomplete parts** (stream aborted or cut off): render the part in a "still loading" state with a subtle shimmer. If the user stops the stream (`handleCancel`), parts with `part_end` received are kept as final; parts without `part_end` are flagged as `incomplete: true` and rendered with a soft "Response stopped mid-part" footer.
- **Malformed parts** (schema validation fails): dropped silently and rendered as the fallback markdown `[unsupported part]`. Never break the chat.
- **Unsupported part types** (server adds a type we don't yet know): same fallback. Renderer is forward-compatible by default.

### 5.7 Security and trust

Because the AI generates UI, the design must defend against:

- **Prompt injection in `chart.data`** — large numbers, non-finite values, NaN, etc. → chart renderer must clamp to a sane domain and skip invalid points.
- **Forms that try to exfiltrate** — fields are constrained to `text | textarea | radio | number`; URLs in labels are sanitized; submission always goes through our existing `pukuFetch` so the access token is attached.
- **Action buttons** — intents are typed strings from a server-allow-listed vocabulary (`"build_notebook" | "compare_datasets" | "refine_plan" | …`). Unknown intents are rejected client-side with a friendly error and logged. (Vocabulary lives in `constants/partIntents.js`, mirroring `constants/ui.js`.)
- **Arbitrary HTML / Markdown XSS** — already mitigated by `react-markdown`'s default; new part renderers never receive raw HTML, only typed fields. Chart annotations are escaped by React by default.
- **Unsafe URLs** — image sources and any link-like field pass through a small `isSafeUrl()` check (http/https/mailto/data:image only).
- **Validation** — every part is parsed by a hand-written validator (`utils/partSchema.js`) that mirrors what `zod` would do, but adds **no dependency**. We avoid `zod` to keep the bundle slim and to keep `react-markdown` / `react-syntax-highlighter` as the only large deps.
- **No arbitrary code execution** — we never eval server-provided strings as code. The `tool_execution` event from the existing stream continues to be parsed but **not** rendered as inline JS.

---

## 6. Proposed response model — small example

A response producing the screenshot-7 pattern might look like:

```json
{
  "id": "msg_8f3a",
  "role": "assistant",
  "parts": [
    {
      "type": "markdown",
      "text": "But I would not call this the final winner yet. We need to run the experiment."
    },
    {
      "type": "markdown",
      "text": "### Next step: let's run the real optimization\n\nI can help you build a complete notebook that automatically:\n\n- Loads both ARFF files\n- Runs the architecture sweep\n- Counts trainable parameters\n- …"
    },
    {
      "type": "markdown",
      "text": "**One question before we finalize the experiment:** Is your instructor requiring you to use the nominal dataset, the numeric dataset, or can you choose either one?"
    },
    {
      "type": "actions",
      "prompt": "Pick the dataset to run next:",
      "options": [
        { "label": "Use nominal dataset — build the notebook",  "intent": "build_notebook",        "primary": true },
        { "label": "Use numeric dataset — build the notebook",  "intent": "build_notebook_numeric", "primary": false },
        { "label": "I can choose either — compare them",          "intent": "compare_datasets",      "primary": false }
      ]
    }
  ]
}
```

A screenshot-2-style chart part:

```json
{
  "type": "chart",
  "id": "c1",
  "title": "What an efficient model looks like",
  "kind": "line",
  "illustrative": true,
  "xLabel": "Trainable parameters",
  "yLabel": "Validation accuracy",
  "data": [
    {"x": 1.0, "y": 0.74}, {"x": 2.0, "y": 0.83}, {"x": 3.0, "y": 0.88},
    {"x": 4.0, "y": 0.90}, {"x": 5.0, "y": 0.905}, {"x": 6.0, "y": 0.91}
  ],
  "annotations": [
    {"x": 2.0, "y": 0.83, "label": "Compact winner", "tone": "good"},
    {"x": 4.0, "y": 0.90, "label": "Diminishing returns", "tone": "info"}
  ]
}
```

A screenshot-5-style card part:

```json
{
  "type": "card",
  "title": "Compact MLP search space",
  "items": [
    {"name": "Model A — Tiny",          "detail": "Input → 4 → Output",             "badge": {"label": "Smallest baseline", "tone": "good"}},
    {"name": "Model B — Small",         "detail": "Input → 8 → Output",             "badge": {"label": "Baseline",          "tone": "muted"}},
    {"name": "Model C — Compact deep",  "detail": "Input → 16 → 8 → Output",        "badge": {"label": "Recommended",       "tone": "accent"}},
    {"name": "Model D — Medium",        "detail": "Input → 32 → 16 → Output",       "badge": {"label": "Comparison",        "tone": "muted"}},
    {"name": "Model E — Wider",         "detail": "Input → 64 → 32 → Output",       "badge": {"label": "Upper baseline",    "tone": "warn"}}
  ],
  "footer": "These are starting candidates, not guaranteed winners. We will prune the search based on validation results."
}
```

A screenshot-1-style form part:

```json
{
  "type": "form",
  "id": "f_design",
  "title": "Let's design your winning model",
  "description": "Fill in what you know, then upload the dataset and assignment PDF or notebook.",
  "fields": [
    {"key": "dataset_type",   "label": "What type of dataset is it?",
     "kind": "radio", "options": ["Tabular (CSV / Excel)", "Images", "Text / NLP", "Time series", "Other / unsure"]},
    {"key": "framework",      "label": "What framework are you using?",
     "kind": "radio", "options": ["TensorFlow / Keras", "PyTorch", "Not decided"]},
    {"key": "metric",         "label": "What is the evaluation metric?",
     "kind": "text", "placeholder": "e.g. Accuracy, F1, parameter limit…"},
    {"key": "constraints",    "label": "Assignment rules or parameter budget",
     "kind": "textarea", "placeholder": "Paste any rules here, or upload the assignment PDF."}
  ],
  "submit": {"label": "Design my optimized neural network", "intent": "design_model"}
}
```

These examples use only what the existing rendering system can already produce in spirit (cards, badges, radios, text inputs, buttons). They are not new paradigms; they are **purpose-built renderings of patterns that exist in Markdown but deserve better**.

---

## 7. Frontend rendering design — file layout

```
components/chat/
  MessageParts.js              ← NEW dispatcher
  parts/
    RichChart.js               ← NEW: thin wrapper around chart library
    CardView.js                ← NEW
    TimelineView.js            ← NEW
    FormView.js                ← NEW
    ActionsView.js             ← NEW
    CalloutView.js             ← NEW
    ImageView.js               ← NEW
    FallbackMarkdown.js        ← NEW
  parts/registry.js            ← NEW: switch table used by MessageParts (also exported for tests)
  MarkdownContent.js           ← unchanged (still used for `markdown` parts and user bubbles)
  CodeBlock.js                 ← unchanged

utils/
  partSchema.js                ← NEW: hand-written validator for each part type
  parts.js                     ← NEW: messageToParts(message) back-compat shim

hooks/
  usePartSubmit.js             ← NEW: small wrapper that handles validation + calling handleSendMessage

constants/
  partIntents.js               ← NEW: allow-listed intents for actions / form submit
  parts.js                     ← NEW: default labels, badge tone palettes
```

**Why this layout**: rendering lives next to existing chat components; validation lives in `utils/` so it can be imported by the share viewer too; intent vocabulary lives in `constants/` like the existing `ui.js`. No new top-level directory.

---

## 8. Interaction design — how a form round-trip works

1. The assistant message contains a `form` part.
2. `FormView` reads `part.fields`, renders radio groups, text inputs, and a textarea with Tailwind primitives (no new dependency). Submit button shows `part.submit.label`.
3. User clicks submit. `FormView` runs a per-field validator (required, max-length, type) before calling `usePartSubmit`.
4. `usePartSubmit` calls `useChatSession.handleSendMessage({ structuredSubmit: { partId: part.id, intent: part.submit.intent, values: { fieldKey: value } } })`.
5. `handleSendMessage` forwards this as a new field on the existing POST body:

   ```json
   {
     "action": "send",
     "content": "",
     "structuredSubmit": { "partId": "f_design", "intent": "design_model",
                           "values": { "dataset_type": "Tabular", "framework": "PyTorch",
                                       "metric": "Accuracy", "constraints": "..." } },
     "attachments": [], "webEnabled": false, "deepResearch": false, ...
   }
   ```

6. Server processes it, the new assistant message may include another `form`, a `markdown` reply, or follow-up `actions` — same envelope, same renderer.
7. The original `form` part remains in the conversation history as already-submitted (visually greyed or with a check mark) so the user can see what they sent.

`actions` follow the same path but with `structuredAction: { partId, intent }` and no `values`.

---

## 9. Streaming and error handling — detailed behavior

| Situation | Behavior |
|---|---|
| Stream opens, no `part_start` yet, `delta.content` arrives | accumulate into a trailing `markdown` part (existing behavior). |
| `part_start` arrives | close current trailing markdown part; open new part with a stable `id`. |
| `part_delta` arrives | merge patch into the in-progress part (immutable update). RAF-batched setState. |
| `part_end` arrives | mark part as `complete: true`; if it was trailing, open a new empty trailing markdown part so subsequent text deltas have somewhere to go. |
| Stream aborts mid-part | in-progress parts flagged `incomplete: true`; render skeleton + "Response stopped" footer. |
| Part fails schema validation | part is dropped; render a tiny inline "could not render" notice. Message continues. |
| Unknown `part.type` | render `[unsupported part: xyz]` fallback; never crash. |
| Server sends malformed JSON line | already ignored in `parseStreamDelta`; new part parser follows same swallow-and-log pattern. |
| `parseStreamPart` throws | caught, logged via `lib/logger.js` (existing dev-only shim), stream continues. |

---

## 10. Security considerations — practical safeguards

| Risk | Mitigation |
|---|---|
| XSS via Markdown | Already handled by `react-markdown`; new parts never inject HTML strings, only typed props. |
| XSS via chart annotation `label` | React text node escaping (default). |
| Image URL injection (`javascript:`, `data:` other than image) | `isSafeUrl()` filter in `partSchema.js`. |
| Form submits containing malicious field values | Field values are sent as JSON; never interpolated into HTML; max length per field (default 4000 chars). |
| Untrusted "intent" strings from server | Allow-listed in `constants/partIntents.js`; unknown intents surface a "Sorry, that action is not available" toast and are dropped. |
| Chart data DoS (10⁹ points) | `partSchema.js` rejects parts whose `data` array exceeds N (configurable, default 1000). |
| Memory blow-up from streaming chart patches | RAF batching caps UI updates at one per frame; old part objects are GC'd. |
| Token exfiltration via form values | All requests go through `pukuFetch` with Bearer token; no `fetch()` in components. |
| Cross-tab pollution | `useChatSession` keeps state per tab; structured submits never reach localStorage. |

These guards are intentionally small and local. The proposal does **not** introduce a permissions system, sandboxed iframes, or content-security policy changes in this design pass — that is for the team to decide in a follow-up if they want richer interactive UI.

---

## 11. Alternatives considered

### A. Improve the existing Markdown-only renderer

**What it would do**: Better CSS for blockquotes-as-cards, ordered lists with custom markers, more spacing, more emoji-as-badges.

**Pros**: No server changes. No new code paths. Pure CSS work.

**Cons**:
- Cannot express *badges* (semantic Recommended / Comparison / Baseline).
- Cannot express *forms* with live validation.
- Cannot express *interactive buttons* (Markdown buttons are links; they can't continue a chat).
- Charts impossible.
- Streaming of partial structured content is impossible.

**Verdict**: Insufficient. This is a fine *adjunct* but not a path to the goal.

### B. Markdown + custom GFM directives (e.g. `:chart[...]{...}`, `::: form ... :::`)

**What it would do**: Define custom block/inline syntaxes in `remark-gfm`; the renderer dispatches on directive name.

**Pros**: Stays in Markdown. No envelope change. Familiar to engineers.

**Cons**:
- Streaming partial structured content is awkward (you'd need a custom remark plugin to know when a directive is "complete enough" to render).
- Forms, validation, and submit-back are still hard (you'd still need a JS-side form state).
- No schema validation: typos and malformed directives become silent render failures.
- Long-term, custom remark plugins become a maintainability burden.

**Verdict**: Reasonable as a *minimum* viable path if the team wants to avoid envelope changes, but it bakes in the limitation that "anything more than text needs a custom parser per directive." It would also require significant work in the existing `MarkdownContent.js` (which currently only overrides `code`/`pre`).

### C. Pure component-based response format (drop Markdown entirely)

**What it would do**: Replace `content: string` with `parts: Part[]`. Server emits only typed parts, never Markdown.

**Pros**: Cleanest model. Trivial streaming. Easy validation.

**Cons**:
- Massive migration: every existing conversation's history needs a `parts[]` representation, or a fallback.
- The team has to teach the model to express everything as parts (paragraphs become `{type:"markdown", text:"…"}`). The current model emits Markdown beautifully — replacing it loses that strength.
- Risk that the model produces verbose JSON for things that read better as prose.

**Verdict**: Overkill. Better to keep Markdown for prose and add parts for structure.

### D. Hybrid Markdown + parts (this proposal)

**What it does**: `parts: Part[]` where `markdown` is one of the types. Markdown keeps doing what it does well; new types cover structure.

**Pros**:
- Backward compatible (`content` field kept for legacy code paths).
- Streaming of prose is unchanged.
- New types are additive — adding a `table` or `checklist` part later is one new file + one validator.
- Validation lives in one place (`utils/partSchema.js`).
- No new heavy dependencies (chart library aside).

**Cons**:
- The team has to design a server envelope change (out of scope here, but unavoidable for any path that includes forms/actions).
- Two ways to render text (Markdown vs parts) requires discipline: don't let both grow into separate worlds.

**Verdict**: **Recommended.** It matches the actual shape of Puku today, supports the screenshots' patterns, and stays incremental.

### E. Adopt Vercel AI SDK (`@ai-sdk/react`) or a similar chat framework

**What it would do**: Replace `useChatSession` with the framework's hook; let it manage parts.

**Pros**: Less code to write. Framework handles streaming, abort, persistence patterns.

**Cons**:
- Major refactor (the entire 572-line `useChatSession` becomes a wrapper).
- Framework's wire format is OpenAI-style; Puku's SSE delta shape is bespoke (no `choices[0].delta.content`, only `delta.content`).
- Existing tests (`tests/playwright/`) would need a complete rewrite of fixtures.
- Adds a sizable dependency.
- Risk of losing the bespoke parts of Puku's UX (artifacts, thought process, copy conversation).

**Verdict**: Rejected for this pass. The proposal below stays within Puku's existing patterns. If the team later decides to standardize, the parts envelope defined here is forward-compatible with the AI SDK's UI message shape.

---

## 12. Recommended approach

**Adopt the hybrid Markdown + parts model (Option D).** Specifically:

1. **Message shape** becomes `{ parts: Part[], reasoning?, toolEvents?, artifacts?, content? }` (keep `content` populated as a Markdown-only projection for back-compat).
2. **Streaming** learns three new event kinds: `part_start`, `part_delta`, `part_end`; legacy `delta` events keep working.
3. **Renderer** becomes `MessageParts` dispatcher with six new part components plus the existing `MarkdownContent` for `markdown` parts.
4. **Forms and actions** submit through `handleSendMessage` with a `structuredSubmit`/`structuredAction` payload, no new endpoint.
5. **Validation** lives in `utils/partSchema.js` (hand-written, no new deps).
6. **Charts** introduce **one** new dependency: a small browser-side chart library (recommendation: **`lightweight-charts`** or **`uplot`** — both small, both have dark themes, both render SVG/Canvas so they look at home on Cloudflare Workers). Defer the choice to phase 1.
7. **No rewrite**. `useChatSession` grows a new `streamPartsRef` and a new `parts` state; old `content` / `streamingContent` / `streamingReasoning` paths continue to function in parallel.
8. **Share view** is automatically compatible because `parts` carries the data and `MessageParts` is added to `app/share/[shareId]/page.js` in one PR.

This recommendation is justified because:
- The investigation shows the upstream is already streaming typed events (`tool_execution`, `artifact`) — extending that pattern to `part_start/delta/end` is the smallest server change.
- The frontend has exactly one render boundary (`MarkdownContent`) and exactly one streaming ingestion (`useChatSession#handleSendMessage`) — both are clean insertion points.
- The team already shipped prior art (`ToolEventCard.js`) that demonstrates the intent; finishing it is cheaper than starting over.
- It satisfies every screenshot pattern: chart, card, timeline, form, actions, callouts, images.

---

## 13. Implementation roadmap — four small phases

Each phase is independently shippable behind a flag. We recommend gating all of them behind a single `NEXT_PUBLIC_CHAT_PARTS_ENABLED` flag for the first release, then per-part flags after that.

### Phase 1 — Foundation (target: 1 PR, ~3–5 days)

- Define `parts.js` types and `partSchema.js` validators (no UI yet).
- Add `MessageParts.js` with the dispatcher and a `FallbackMarkdown` for every part type.
- Back-compat shim: `messageToParts(message)` → `[ {type:"markdown", text: message.content} ]`.
- Update `MessageItem` and `MessageList` to render `MessageParts` when `message.parts` exists, else fall back to the current `MarkdownContent` path.
- Wire `useChatSession` to *also* populate `streamPartsRef` from existing `delta.content` deltas (one trailing `markdown` part), so legacy SSE streams render identically.
- Update `commitAssistantMessage` to compute both `parts` and `content`.
- Update Playwright fixtures (`buildSseBody`) to verify legacy deltas still produce correct rendering.
- Result: no visual change, but the codebase is ready for parts. All existing tests pass.

### Phase 2 — Core rich responses (target: 2–3 PRs, ~1–2 weeks)

- **Cards**: `CardView.js` with header, items (name + detail + badge), and footer.
- **Timelines**: `TimelineView.js` with circular number badges and vertical connector.
- **Callouts**: `CalloutView.js` (info / success / warn / danger tones).
- **Images**: `ImageView.js` with caption, lazy-loaded.
- Server emits `part_start` / `part_end` for these. Add SSE test fixtures for each type.
- Polish: badges (small, consistent with the existing palette in `app/globals.css`).
- Result: Puku can show the screenshot-3, screenshot-4, screenshot-5, and screenshot-6 patterns. The user experience is meaningfully better on day one.

### Phase 3 — Interactive forms & actions (target: 1–2 PRs, ~1 week)

- **Actions**: `ActionsView.js` rendering primary + secondary buttons.
- **Forms**: `FormView.js` with `radio | text | textarea` fields, validation, and submit.
- `usePartSubmit` hook.
- `handleSendMessage` learns `structuredSubmit` and `structuredAction` body fields.
- Server contract: documented in `services/chat/messages.js` JSDoc (the actual server change is out of scope for this design pass).
- Add Playwright fixtures for the form submit round-trip (mock a server that consumes `structuredSubmit` and returns the next message).
- Result: screenshot-1 and screenshot-7 patterns work. Forms validate; buttons describe outcomes.

### Phase 4 — Charts + refinement (target: 1 PR + 1 polish PR, ~1–2 weeks)

- Pick a chart library (lightweight-charts or uPlot) based on a small benchmark in `tests/bench/charts/`.
- `RichChart.js` with axis labels, annotations, and an "illustrative" badge when `illustrative: true`.
- Accessibility pass: focus rings on every interactive element, ARIA labels on form fields and buttons, `prefers-reduced-motion` honored on the chart's animations.
- Streaming polish: parts render progressively; trailing text re-opens correctly.
- Mobile pass: every part reflows on small screens (≤375 px).
- Result: full feature parity with the screenshots. Mobile and accessibility done.

Total: **~4–6 weeks** of focused work across **~5 PRs**. Each phase independently delivers value.

---

## 14. Testing strategy

### 14.1 Functional

- **Unit tests** (`vitest` if the team wants to add it, otherwise Playwright component mode):
  - `partSchema.js` — every part type, valid + invalid fixtures.
  - `MessageParts.js` — dispatcher returns correct component for each `type`.
  - `messageToParts.js` — back-compat shim produces a single markdown part.
  - Each new component renders without crashing on a minimal valid part.

- **Streaming tests** (extend `tests/playwright/`):
  - `buildSseBody` in `fixtures/api-mocks.ts` extended with helpers: `partStart()`, `partDelta()`, `partEnd()`, `part()`.
  - New spec: `07-stream-parts.spec.ts` — emits a `part_start` then `delta.content`, asserts both render correctly.
  - New spec: `08-form-submit.spec.ts` — renders a form, fills it, submits, asserts a `structuredSubmit` POST is made with correct payload.
  - New spec: `09-action-pick.spec.ts` — clicks an action button, asserts `structuredAction` POST.
  - Update `03-send-message.spec.ts` to assert that legacy deltas still produce the same DOM as before.

### 14.2 UX

- Manual review against the seven screenshots, side-by-side.
- Mobile pass at 375 × 812 (iPhone-class) and 320 × 568 (smallest supported).
- A11y pass with axe (or `eslint-plugin-jsx-a11y` rules already present in `eslint.config.mjs`).
- Read-only share view (`app/share/[shareId]/page.js`) renders the same components and looks consistent.

### 14.3 Performance

- Render 50 messages with mixed parts (timeline + cards + forms) and measure TTI / FPS.
- Stream a 5000-point chart and assert no jank (no frame > 100 ms).
- Bundle size: assert `npm run build` output for the chat route increases by < 100 KB gzipped.

### 14.4 Compatibility

- Open an existing conversation created before parts existed → renders identically (no `parts` field falls back to `MarkdownContent`).
- Open a shared link → renders identically for both new and old messages.
- Switch between `/new` and `/chats` and `/share/:id` → all three routes handle parts consistently.
- Replay a recording from before the rollout with parts enabled → no errors.

---

## 15. Open questions and assumptions (for the Puku AI team)

These are points the proposal **cannot decide without input from the team** and that should be confirmed before implementation begins.

1. **Server contract change.** The new envelope (`part_start` / `part_delta` / `part_end` / `part`) requires a coordinated server change at `chat.api.dev.puku.sh`. Is the team prepared to ship that as part of phase 1, or do we need to start with a Markdown-only escape hatch (Option B) for the first iteration?
2. **Model guidance.** The model needs system-prompt guidance to emit parts. Who owns that prompt — Puku infra team? — and how is it versioned?
3. **Chart library choice.** Recommend lightweight-charts or uPlot; confirm no preference / no compliance constraints that block either.
4. **Intent allow-list.** Should `constants/partIntents.js` be a static list maintained by the team, or pulled from the server? Static is simpler and safer for phase 1.
5. **Form upload fields.** Screenshot-1 mentions "upload the assignment PDF." Should forms support file uploads as a field kind (which would require combining with `uploadAttachment`)? Out of scope for this proposal; recommend deferring to a phase 5.
6. **Persistence model.** When a user submits a form, should the original form part be visually marked "submitted" (with a checkmark) so they can see what they sent? Recommend yes (phase 3).
7. **Accessibility bar.** Should the proposal meet WCAG 2.1 AA from day one, or ship AA in phase 4 only?
8. **Theming.** Puku is dark-first today. Are light themes on the roadmap? The CSS uses `--color-dashboard-*` tokens so a light theme is mostly a token flip; confirm before phase 4.
9. **Telemetry.** The proposal does not add telemetry. Should we instrument which parts render, how often forms are submitted vs abandoned, and which action buttons get clicked? Recommend deferring to a follow-up; the existing `lib/logger.js` is sufficient for dev-only observability.
10. **Like/Dislike wiring.** The unwired buttons in `MessageItem` are not in scope here. Should they be wired in phase 3 alongside the new actions (they share the same affordance)?

---

## 16. Reference paths for the implementation team

When work begins, these are the files to read first:

| File | Why |
|---|---|
| `services/chat/messages.js` | The SSE reader loop (`sendMessage`) and the delta normalizer (`parseStreamDelta`); the new `parseStreamPart` slots in next to `parseStreamDelta`. |
| `hooks/useChatSession.js` | Owns all chat state. `commitAssistantMessage` (around line 288) and the `onChunk` callback in `handleSendMessage` (around line 388) are the two change sites. |
| `components/chat/MarkdownContent.js` | Existing Markdown renderer; reused for `markdown` parts and for the user bubble. |
| `components/chat/MessageItem.js` and `MessageList.js` | Where `MessageParts` plugs in. |
| `utils/messageContent.js#splitMessageContent` | The existing "dispatcher" precedent for reasoning vs content; the parts model extends this pattern. |
| `app/globals.css` | The Puku design tokens; new components should consume them, not introduce new colors. |
| `tests/playwright/fixtures/api-mocks.ts#buildSseBody` | The wire-format SSOT; extend with `partStart/partDelta/partEnd/part` helpers. |
| `app/share/[shareId]/page.js` | Read-only mirror; confirm parts render the same here. |
| `wrangler.jsonc` / `next.config.mjs` | Confirm any new chart library is browser-bundle only (no Node built-ins). |

---

## 17. Conclusion

Puku already has the bones for richer responses: a single render boundary (`MarkdownContent`), a single streaming ingestion (`useChatSession#handleSendMessage`), an orphaned tool-event component (`ToolEventCard`), and prior art for typed inline content (`splitMessageContent`). The proposal above finishes that scaffold without disrupting anything that already works.

The smallest useful first version is **phase 1 + phase 2**: cards, timelines, callouts, and images. That alone delivers most of the screenshot-4/5/6 patterns. Phases 3 and 4 add the genuinely interactive pieces (forms, action buttons) and the high-value chart visualization.

**Do not start implementing until the Puku AI team has reviewed this proposal.** The most important confirmation is **#1** — whether the server can ship the new envelope. Everything else can be adjusted.
