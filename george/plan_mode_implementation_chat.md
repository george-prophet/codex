# Plan-mode implementation for Codex
_Exported on 2/23/2026 at 04:30:29 EST from Cursor (2.5.20)_

---

**User**

I want to understand how plan-mode is implemented for codex. Create a detailed markdown explanation, with diagrams. What are the tools available in plan mode, what are the specific prompt changes for it, how is it diferent from normal agent mode, etc. 

---

**Cursor**

Let me explore the codebase to understand how plan mode is implemented.

Now let me read the actual plan mode template and the default mode template to get the exact prompt content.

Now let me look at the TUI plan implementation prompt and the streaming/event handling for plans.

Now I have a comprehensive understanding. Let me also quickly check how the footer indicator works for plan mode.

Now I have all the information needed. Here is the detailed explanation:

---

# How Plan Mode Is Implemented in Codex

## Overview

Plan mode is one of two **collaboration modes** in Codex (the other being **Default mode**). It transforms Codex from a code-executing agent into a conversational planner that explores the codebase, asks clarifying questions, and produces a detailed implementation plan -- all without making any mutations to the repository.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER / TUI                                  │
│                                                                     │
│  /plan command ─┐    Shift+Tab ─┐    Footer: "Plan mode" (magenta)  │
│                 │               │                                    │
│                 ▼               ▼                                    │
│         ┌─────────────────────────────┐                             │
│         │    CollaborationModeMask    │                              │
│         │  mode: Plan                │                              │
│         │  reasoning_effort: Medium  │                              │
│         │  developer_instructions:   │                              │
│         │    (plan.md content)       │                              │
│         └────────────┬───────────────┘                              │
└──────────────────────┼──────────────────────────────────────────────┘
                       │ apply_mask()
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       CORE (codex.rs)                               │
│                                                                     │
│  ┌──────────────────────────────────────────┐                      │
│  │         Session Configuration            │                      │
│  │  collaboration_mode: CollaborationMode   │                      │
│  │    .mode = ModeKind::Plan                │                      │
│  │    .settings.developer_instructions =    │                      │
│  │      "<collaboration_mode>...plan.md..." │                      │
│  └──────────────┬───────────────────────────┘                      │
│                 │                                                    │
│    ┌────────────▼────────────┐    ┌───────────────────────┐        │
│    │   Prompt Construction   │    │  Tool Handler Layer   │        │
│    │                         │    │                       │        │
│    │ base system prompt      │    │ request_user_input:   │        │
│    │ + <collaboration_mode>  │    │   ✅ allowed          │        │
│    │     plan.md content     │    │                       │        │
│    │   </collaboration_mode> │    │ update_plan:          │        │
│    │                         │    │   ❌ blocked (error)  │        │
│    └────────────┬────────────┘    │                       │        │
│                 │                  │ shell, apply_patch:   │        │
│                 │                  │   available but model │        │
│                 │                  │   instructed not to   │        │
│                 │                  │   use for mutations   │        │
│                 ▼                  └───────────────────────┘        │
│    ┌────────────────────────────────────────┐                      │
│    │         Streaming Response             │                      │
│    │                                        │                      │
│    │  PlanModeStreamState                   │                      │
│    │  ├─ PlanParsers (per-item)             │                      │
│    │  ├─ pending_agent_message_items        │                      │
│    │  └─ ProposedPlanItemState              │                      │
│    │       ├─ parse <proposed_plan> tags    │                      │
│    │       ├─ emit PlanDelta events         │                      │
│    │       └─ emit TurnItem::Plan           │                      │
│    └────────────────────────────────────────┘                      │
└─────────────────────────────────────────────────────────────────────┘
                       │
                       ▼ events
┌─────────────────────────────────────────────────────────────────────┐
│                     TUI (chatwidget.rs)                              │
│                                                                     │
│  on PlanDelta ──► render plan content in chat                       │
│  on TurnComplete + saw_plan_item ──►                                │
│      ┌─────────────────────────────────────┐                       │
│      │  "Implement this plan?" popup       │                       │
│      │  ┌─ Yes, implement this plan ──►    │                       │
│      │  │    switch to Default mode,       │                       │
│      │  │    send "Implement the plan."    │                       │
│      │  └─ No, stay in Plan mode           │                       │
│      └─────────────────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────────┘
```

## 1. Type System: `ModeKind` and `CollaborationMode`

The foundation is the `ModeKind` enum defined in `codex-rs/protocol/src/config_types.rs`:

```169:214:codex-rs/protocol/src/config_types.rs
pub enum ModeKind {
    Plan,
    #[default]
    Default,
    // ... PairProgramming and Execute are hidden/internal
}
// ...
pub const TUI_VISIBLE_COLLABORATION_MODES: [ModeKind; 2] = [ModeKind::Default, ModeKind::Plan];
// ...
impl ModeKind {
    pub const fn allows_request_user_input(self) -> bool {
        matches!(self, Self::Plan)
    }
}
```

Each `ModeKind` is wrapped in a `CollaborationMode` that carries settings (model, reasoning effort, developer instructions). Switching modes is done by applying a `CollaborationModeMask` -- a partial overlay that updates only the fields it specifies.

## 2. Plan Mode Preset

The plan preset is defined in `codex-rs/core/src/models_manager/collaboration_mode_presets.rs`:

```16:24:codex-rs/core/src/models_manager/collaboration_mode_presets.rs
fn plan_preset() -> CollaborationModeMask {
    CollaborationModeMask {
        name: ModeKind::Plan.display_name().to_string(),
        mode: Some(ModeKind::Plan),
        model: None,
        reasoning_effort: Some(Some(ReasoningEffort::Medium)),
        developer_instructions: Some(Some(COLLABORATION_MODE_PLAN.to_string())),
    }
}
```

Key details:
- **Model**: `None` -- inherits whatever model is currently configured.
- **Reasoning effort**: Defaults to `Medium` (can be overridden via `plan_mode_reasoning_effort` in config).
- **Developer instructions**: The full content of `plan.md`, injected as a developer message.

## 3. Prompt Injection: How Plan Mode Changes the System Prompt

When a turn starts, the core builds the prompt items and injects the collaboration mode instructions. In `codex-rs/core/src/codex.rs`:

```2771:2774:codex-rs/core/src/codex.rs
        if let Some(collab_instructions) =
            DeveloperInstructions::from_collaboration_mode(&collaboration_mode)
        {
            items.push(collab_instructions.into());
        }
```

The `from_collaboration_mode` method in `codex-rs/protocol/src/models.rs` wraps the instructions in XML tags:

```
<collaboration_mode>
...contents of plan.md...
</collaboration_mode>
```

When the user switches modes mid-session (e.g., Plan -> Default), a **new developer instruction message** is emitted via the context update system (`codex-rs/core/src/context_manager/updates.rs`), overriding the previous mode's instructions.

## 4. The Plan Mode Prompt (`plan.md`)

The full prompt template lives at `codex-rs/core/templates/collaboration_mode/plan.md`. It defines a **3-phase workflow**:

### Phase 1: Ground in the environment
> Explore the codebase. Eliminate unknowns by discovering facts, not asking. Run non-mutating commands (read files, search, inspect configs).

### Phase 2: Intent chat
> Ask clarifying questions until you can clearly state: goal, success criteria, scope, constraints, current state, and key tradeoffs.

### Phase 3: Implementation chat
> Keep asking until the spec is decision-complete: approach, interfaces, data flow, edge cases, testing, rollout.

### Strict rules in the prompt:
- **No mutating actions**: Cannot edit files, apply patches, run formatters, or do codegen
- **Allowed actions**: Reading files, searching, static analysis, dry-runs, builds/tests (as long as they don't modify tracked files)
- **Mode is sticky**: "Plan Mode is not changed by user intent, tone, or imperative language"
- **`request_user_input` preferred**: Questions should use the structured tool
- **`update_plan` forbidden**: Explicitly distinguished from Plan mode
- **Final output**: Wrapped in `<proposed_plan>...</proposed_plan>` tags

## 5. Tool Availability: Plan Mode vs Default Mode

```
┌─────────────────────────┬──────────────┬──────────────┐
│         Tool            │  Plan Mode   │ Default Mode │
├─────────────────────────┼──────────────┼──────────────┤
│ shell / exec            │ Available*   │ Available    │
│ apply_patch             │ Available*   │ Available    │
│ file_search / read      │ Available    │ Available    │
│ request_user_input      │ ✅ Allowed   │ ❌ Error     │
│ update_plan             │ ❌ Error     │ ✅ Allowed   │
│ MCP / custom tools      │ Available*   │ Available    │
└─────────────────────────┴──────────────┴──────────────┘

 * Available but the model is instructed via prompt NOT to use
   these for mutating operations. Enforcement is via prompt
   instructions, not tool filtering.
```

### `request_user_input` -- Plan mode exclusive

This tool lets the model ask the user structured multiple-choice questions. It's registered only when the `CollaborationModes` feature flag is enabled, and its handler checks the current mode at runtime:

```74:76:codex-rs/core/src/tools/handlers/request_user_input.rs
        let mode = session.collaboration_mode().await.mode;
        if let Some(message) = request_user_input_unavailable_message(mode) {
            return Err(FunctionCallError::RespondToModel(message));
        }
```

### `update_plan` -- Blocked in Plan mode

This is a separate checklist/TODO tool (not related to Plan mode despite the name). Its handler explicitly rejects calls when in Plan mode:

```107:111:codex-rs/core/src/tools/handlers/plan.rs
    if turn_context.collaboration_mode.mode == ModeKind::Plan {
        return Err(FunctionCallError::RespondToModel(
            "update_plan is a TODO/checklist tool and is not allowed in Plan mode".to_string(),
        ));
    }
```

### Shell and apply_patch -- Prompt-enforced restriction

These tools remain in the tool set for both modes. The Plan mode prompt instructs the model not to use them for mutations. This is a deliberate design choice: the model can still run read-only shell commands (e.g., `grep`, `find`, `cargo check --message-format=short`) to gather information during planning.

## 6. Streaming: `<proposed_plan>` Parsing

When the model streams a response in Plan mode, the core uses a specialized parser pipeline:

```
Model output stream
       │
       ▼
 ProposedPlanParser
 (line-by-line tag detection)
       │
       ├─── Normal text ──► AgentMessage deltas (shown in chat)
       │
       ├─── <proposed_plan> ──► ProposedPlanStart
       │         │
       │    plan content ──► PlanDelta events (streamed to TUI)
       │         │
       └─── </proposed_plan> ──► ProposedPlanEnd
                                       │
                                       ▼
                              TurnItem::Plan(PlanItem)
                              (persisted in turn history)
```

The `PlanModeStreamState` struct in `codex.rs` manages this:

```5373:5396:codex-rs/core/src/codex.rs
struct PlanModeStreamState {
    plan_parsers: PlanParsers,
    pending_agent_message_items: HashMap<String, TurnItem>,
    started_agent_message_items: HashSet<String>,
    leading_whitespace_by_item: HashMap<String, String>,
    plan_item_state: ProposedPlanItemState,
}
```

Key behaviors:
- **Plan text is stripped** from the agent message content (via `strip_proposed_plan_blocks`) so it doesn't appear as regular chat text.
- **Plan content is emitted separately** as `PlanDelta` events for special UI rendering.
- **Agent messages that are plan-only** (only `<proposed_plan>` content, no surrounding text) are suppressed to avoid empty chat bubbles.

## 7. TUI: Entering and Exiting Plan Mode

### Entry points

1. **`/plan` slash command**: Types `/plan` in the input box.
2. **Shift+Tab cycling**: Cycles between Default and Plan modes via the footer indicator.
3. **Config**: `mode: plan` in config.toml sets Plan as the initial mode.

### Footer indicator

When Plan mode is active, the TUI footer displays **"Plan mode"** in magenta:

```95:96:codex-rs/tui/src/bottom_pane/footer.rs
            CollaborationModeIndicator::Plan => format!("Plan mode{suffix}"),
// ...
            CollaborationModeIndicator::Plan => Span::from(label).magenta(),
```

### "Implement this plan?" popup

After a turn completes in Plan mode with a `<proposed_plan>` block, the TUI shows a popup:

```
┌─ Implement this plan? ──────────────────────────┐
│                                                  │
│  > Yes, implement this plan                      │
│    Switch to Default and start coding.           │
│                                                  │
│    No, stay in Plan mode                         │
│    Continue planning with the model.             │
│                                                  │
└──────────────────────────────────────────────────┘
```

- **"Yes"**: Switches to Default mode and sends `"Implement the plan."` as a user message.
- **"No"**: Stays in Plan mode for further refinement.

This is implemented in `chatwidget.rs`:

```1422:1447:codex-rs/tui/src/chatwidget.rs
    fn maybe_prompt_plan_implementation(&mut self) {
        if !self.collaboration_modes_enabled() { return; }
        if !self.queued_user_messages.is_empty() { return; }
        if self.active_mode_kind() != ModeKind::Plan { return; }
        if !self.saw_plan_item_this_turn { return; }
        // ...
        self.open_plan_implementation_prompt();
    }
```

## 8. Plan Mode vs Default Mode: Summary of Differences

| Aspect | Plan Mode | Default Mode |
|---|---|---|
| **Purpose** | Conversational planning | Code execution |
| **Mutating actions** | Forbidden (prompt-enforced) | Allowed |
| **`request_user_input`** | Available | Returns error |
| **`update_plan`** | Returns error | Available |
| **`<proposed_plan>` blocks** | Emitted and specially rendered | Not used |
| **Default reasoning effort** | Medium | Model default |
| **System prompt** | `plan.md` (3-phase workflow) | `default.md` (minimal, execute-first) |
| **Post-turn behavior** | "Implement?" popup if plan produced | Nothing special |
| **Footer indicator** | "Plan mode" in magenta | No indicator (Default is the baseline) |
| **Mode stickiness** | Explicit exit required | Explicit exit required |

## 9. Configuration

Plan mode can be configured in `config.toml`:

- **`mode: plan`** -- Start the TUI in Plan mode.
- **`plan_mode_reasoning_effort`** -- Override the default `Medium` reasoning effort for Plan mode. Accepts `low`, `medium`, `high`, or `none`.

## 10. Feature Flag

Plan mode (and collaboration modes generally) are gated behind the `CollaborationModes` feature flag:

```rust
Feature::CollaborationModes  // key: "collaboration_modes"
```

When this flag is disabled:
- The `/plan` command shows "Collaboration modes are disabled."
- The `request_user_input` tool is not registered.
- Shift+Tab cycling is unavailable.
- The mode indicator doesn't appear in the footer.

## Data Flow Summary

```
User types /plan
       │
       ▼
TUI applies plan_preset CollaborationModeMask
       │
       ▼
Session.collaboration_mode updated to ModeKind::Plan
       │
       ▼
Next turn: plan.md injected as <collaboration_mode> developer instructions
       │
       ▼
Model receives: base prompt + plan.md + tools (including request_user_input)
       │
       ▼
Model explores codebase (read-only), asks questions via request_user_input
       │
       ▼
Model produces <proposed_plan>...</proposed_plan> block
       │
       ▼
ProposedPlanParser strips plan from agent message, emits PlanDelta events
       │
       ▼
TUI renders plan specially, shows "Implement this plan?" popup
       │
       ▼
User selects "Yes" → switches to Default mode, sends "Implement the plan."
       │
       ▼
Model now in Default mode with full mutation capabilities, executes the plan
```

---

**User**

Create a markdown file with the details you found. Use mermaid or ascii diagrams - your choice

---

**Cursor**

Created `docs/plan-mode-implementation.md` with the full writeup. It includes:

- **Mermaid diagrams** for the architecture, 3-phase workflow, tool availability, streaming parser pipeline, and end-to-end sequence diagram
- **Type system** breakdown (`ModeKind`, `CollaborationMode`, `CollaborationModeMask`)
- **Prompt injection** mechanics — how `plan.md` gets wrapped in `<collaboration_mode>` tags and injected as developer instructions
- **Tool availability table** with enforcement details (runtime handler checks vs prompt-only)
- **Streaming parser** — how `<proposed_plan>` blocks are parsed, stripped from agent messages, and emitted as separate `PlanDelta` events
- **TUI interactions** — `/plan`, Shift+Tab, footer indicator, "Implement this plan?" popup with its trigger conditions
- **Plan vs Default comparison** table
- **Configuration** and feature flag details
- **Source file index** for every relevant module

---

**User**

If in plan mode, and I request an update to the plan, how is the plan updated without the plan-update tool? Is there a tool exposed to edit the plan by submitting diffs/patches to the plan? Or does the llm regenerate the plan from scratch with the requested modifications?

---

**Cursor**

The LLM regenerates the entire plan from scratch. There is no diff/patch tool for plan editing.

The relevant line from the plan mode prompt (`codex-rs/core/templates/collaboration_mode/plan.md`) makes this explicit:

```118:122:codex-rs/core/templates/collaboration_mode/plan.md
Do not ask "should I proceed?" in the final output. The user can easily switch out of Plan mode and request implementation if you have included a `<proposed_plan>` block in your response. Alternatively, they can decide to stay in Plan mode and continue refining the plan.

Only produce at most one `<proposed_plan>` block per turn, and only produce it when you are presenting a complete spec.

If the user stays in Plan mode and asks for revisions after a prior `<proposed_plan>`, any new `<proposed_plan>` must be a complete replacement.
```

So the flow is:

1. Model emits a `<proposed_plan>` with the full plan.
2. User stays in Plan mode and says e.g. "Actually, use Redis instead of Memcached."
3. Model produces a **new** `<proposed_plan>` block that is a **complete replacement** — not a delta or patch against the previous one.

There is no tool, protocol message, or data structure for incremental plan edits. The `ProposedPlanItemState` in the streaming layer only tracks a single plan item per turn (with a simple `started` / `completed` lifecycle), and each turn's plan item is independent — there's no merging with a prior turn's plan. The `ProposedPlanParser` just looks for `<proposed_plan>` open/close tags in the current response stream; it has no awareness of any previous plan.

This is a straightforward design choice: the conversation history already contains the prior plan (as a `TurnItem::Plan`), so the model has full context to regenerate a revised version. Incremental patching would add complexity and introduce risks of drift between the model's understanding and the actual plan state.

---

**User**

When iterating on a plan, how does the agent know what the previous plan was in order to generate the new plan?

---

**Cursor**

Good question — let me trace exactly how the previous plan makes it back into the model's context.

Now let me check what happens to the agent message that *contains* the `<proposed_plan>` — does the raw text with the tags stay in the conversation history sent to the API, or is it stripped?

Now let me check what happens with the raw response items — the actual API conversation history that gets sent back to the model on the next turn.

This is the key mechanism. Let me also check how the input items are built for a new turn to confirm that the raw response (including `<proposed_plan>` tags) stays in the conversation.

Now let me look at what `clone_history().for_prompt()` returns — does it include the raw response items with `<proposed_plan>` tags intact?

Now I have the full picture. The key insight is in `codex-rs/core/src/codex.rs` at the sampling request construction:

```rust
let sampling_request_input: Vec<ResponseItem> = {
    sess.clone_history().await.for_prompt(&turn_context.model_info.input_modalities)
};
```

The conversation history contains the **raw `ResponseItem`s** from the API — these are the actual model outputs including assistant messages with `<proposed_plan>` tags fully intact. The stripping only happens for the *TUI display layer* (`TurnItem`), not for the conversation history sent back to the model.

Here's the answer:

---

The model knows the previous plan because the **raw API conversation history** is preserved and sent back on every subsequent turn — and it contains the assistant message with the `<proposed_plan>...</proposed_plan>` tags intact.

There are two parallel data paths, and they handle the plan text differently:

```
Model response: "Here's my analysis.\n<proposed_plan>\n- Step 1\n- Step 2\n</proposed_plan>"
                          │
                          ├──────────────────────────────────┐
                          │                                  │
                    Conversation History               TUI Display Layer
                    (ResponseItem)                     (TurnItem)
                          │                                  │
                    Raw text preserved               strip_proposed_plan_blocks()
                    with <proposed_plan>              removes plan from AgentMessage
                    tags intact                       ──► emits as separate PlanItem
                          │                                  │
                    Sent back to model               User sees plan rendered
                    on next turn via                  specially in chat UI
                    clone_history().for_prompt()
```

**Conversation history** (`ResponseItem` in `ContextManager`): The raw assistant message — including `<proposed_plan>` tags and their content — is stored as-is. When the next turn starts, `clone_history().for_prompt()` returns these raw items directly. The model receives the full previous conversation including its own `<proposed_plan>` output verbatim.

**TUI display layer** (`TurnItem`): The `strip_proposed_plan_blocks()` function removes plan content from the `AgentMessage` shown in chat, and a separate `TurnItem::Plan(PlanItem)` is emitted for dedicated UI rendering. This is purely cosmetic — it doesn't affect what the model sees.

The relevant code in `codex-rs/core/src/stream_events_utils.rs` shows the stripping only applies to the `TurnItem` path:

```170:184:codex-rs/core/src/stream_events_utils.rs
        | ResponseItem::WebSearchCall { .. } => {
            let mut turn_item = parse_turn_item(item)?;
            if plan_mode && let TurnItem::AgentMessage(agent_message) = &mut turn_item {
                let combined = agent_message
                    .content
                    .iter()
                    .map(|entry| match entry {
                        codex_protocol::items::AgentMessageContent::Text { text } => text.as_str(),
                    })
                    .collect::<String>();
                let stripped = strip_proposed_plan_blocks(&combined);
                agent_message.content =
                    vec![codex_protocol::items::AgentMessageContent::Text { text: stripped }];
            }
            Some(turn_item)
        }
```

While the prompt input is built from the unmodified history:

```4709:4713:codex-rs/core/src/codex.rs
        let sampling_request_input: Vec<ResponseItem> = {
            sess.clone_history()
                .await
                .for_prompt(&turn_context.model_info.input_modalities)
        };
```

So when iterating on a plan, the conversation the model sees looks something like:

```
[developer] <collaboration_mode>...plan.md...</collaboration_mode>
[user]      "Refactor the auth module to use JWT"
[assistant] "I've explored the codebase. Here's my plan:\n<proposed_plan>\n## Auth Refactor\n- Step 1: ...\n</proposed_plan>"
[user]      "Use Redis for session storage instead of Postgres"
[assistant] ← model generates a complete new <proposed_plan> here, seeing the full prior one in context
```

The model simply reads its own previous `<proposed_plan>` output from the conversation history and generates a complete replacement incorporating the requested changes. No diffing, no patching — just full regeneration with full context.

---

**User**

Do the phases in @codex-rs/core/templates/collaboration_mode/plan.md only apply in plan-mode? Are they also used in the execution mode prompt?

---

**Cursor**

Let me check the default mode prompt and see if there's any phase-related content there.

Yes, the 3-phase workflow (Ground in environment → Intent chat → Implementation chat) is **exclusive to Plan mode**. It does not appear in the Default mode prompt at all.

The Default mode prompt (`default.md`) is minimal — just 12 lines. It says:

1. You're in Default mode, previous mode instructions are no longer active.
2. `request_user_input` is unavailable.
3. Strongly prefer executing the user's request rather than stopping to ask questions.

That's the opposite philosophy: Plan mode is "ask a lot, explore, don't execute"; Default mode is "just go do it."

There is also a hidden `execute.md` template that mentions a single "phase" in passing, but that's for the internal `Execute` mode (which is `#[doc(hidden)]`, not exposed in the TUI, and can't be selected by users). It's unrelated to the 3-phase structure in `plan.md`.

---

**User**

When is @codex-rs/core/templates/collaboration_mode/default.md used vs @codex-rs/core/templates/collaboration_mode/execute.md ?

---

**Cursor**

`execute.md` is **not currently used anywhere in the codebase**. It exists as a file on disk but nothing `include_str!`s it or references it.

The only two collaboration mode templates that are wired up are:
- **`plan.md`** → used by the `Plan` preset (Plan mode)
- **`default.md`** → used by the `Default` preset (Default mode)

This matches the type definitions — `ModeKind::Execute` is marked `#[doc(hidden)]`, `#[serde(skip_serializing, skip_deserializing)]`, and `#[ts(skip)]`, meaning it can't be selected through config, the TUI, or the API. It's not included in `TUI_VISIBLE_COLLABORATION_MODES` either.

So `execute.md` appears to be either a leftover from earlier development or a future mode that hasn't been connected yet. The only active modes today are **Plan** (uses `plan.md`) and **Default** (uses `default.md`).

---

**User**

What about @codex-rs/core/templates/model_instructions/gpt-5.2-codex_instructions_template.md . How does this relate to @codex-rs/core/templates/collaboration_mode/plan.md and @codex-rs/core/templates/collaboration_mode/default.md ?

---

**Cursor**

Now let me see how these all get assembled into the final prompt sent to the model.

Now I have the full picture. Here's how they relate:

---

The `gpt-5.2-codex_instructions_template.md` file is the **base system prompt** (also called "base instructions"). The collaboration mode prompts (`plan.md`, `default.md`) are **layered on top** of it as separate developer messages. They are not alternatives — they're additive.

Here's the layering:

```
┌─────────────────────────────────────────────────────────┐
│  instructions field (system prompt / base instructions) │
│                                                         │
│  gpt-5.2-codex_instructions_template.md                 │
│  "You are Codex, a coding agent..."                     │
│  + formatting rules, editing constraints, plan tool     │
│    usage, frontend guidelines, etc.                     │
└─────────────────────────────────────────────────────────┘
                        +
┌─────────────────────────────────────────────────────────┐
│  developer message (injected into input items)          │
│                                                         │
│  <collaboration_mode>                                   │
│    plan.md   OR   default.md                            │
│  </collaboration_mode>                                  │
└─────────────────────────────────────────────────────────┘
                        +
┌─────────────────────────────────────────────────────────┐
│  conversation history (user messages, assistant         │
│  messages, tool calls/outputs)                          │
└─────────────────────────────────────────────────────────┘
```

The key code in `client.rs` shows this:

```519:521:codex-rs/core/src/client.rs
        let instructions = &prompt.base_instructions.text;
        let input = prompt.get_formatted_input();
        let tools = create_tools_json_for_responses_api(&prompt.tools)?;
```

The `instructions` field of the API request is set to the base instructions (from the template). The collaboration mode prompt is a separate `DeveloperInstructions` item pushed into the `input` array alongside the conversation history. From `codex.rs`:

```2771:2774:codex-rs/core/src/codex.rs
        if let Some(collab_instructions) =
            DeveloperInstructions::from_collaboration_mode(&collaboration_mode)
        {
            items.push(collab_instructions.into());
        }
```

So for **both** Plan mode and Default mode, the model always receives:

1. **`instructions`** (system-level): The `gpt-5.2-codex_instructions_template.md` content — formatting rules, editing constraints, `rg` preference, plan tool guidance, git safety rules, frontend guidelines, etc. This is **always present regardless of mode**.

2. **`input` items**: A developer message containing the `<collaboration_mode>` block — either `plan.md` or `default.md` — followed by the conversation history.

The template's `{{ personality }}` placeholder gets substituted with a personality string (friendly, pragmatic, or empty) based on the user's config. That's unrelated to collaboration mode.

In short: the model instructions template is the "who you are and how you work" foundation that's **shared across all modes**. The collaboration mode prompts are "what mode you're in right now" instructions layered on top. The 3-phase workflow from `plan.md` only appears when Plan mode is active.

---

**User**

Add all this info (from the questions I asked) to the markdown file

---

**Cursor**

Updated the markdown file with three new sections:

1. **How Prompt Layers Compose** -- explains that the base instructions template (`gpt-5.2-codex_instructions_template.md`) is the system-level `instructions` field, while collaboration mode prompts are layered on top as developer messages in `input`. Includes a mermaid diagram showing the layering. Also clarifies that `execute.md` exists on disk but is not wired up.

2. **Plan Iteration: How the Model Knows the Previous Plan** -- explains the two parallel data paths (conversation history keeps raw `<proposed_plan>` tags intact for the model; TUI display layer strips them for rendering). Includes a mermaid diagram, the relevant code snippets, and an example of what the model's conversation looks like during iteration.

3. **Key Source Files** table expanded with the new files discussed: `execute.md`, `gpt-5.2-codex_instructions_template.md`, `model_info.rs`, `stream_events_utils.rs`, `history.rs`, and `client.rs`.

---

**User**

Add absolute links (using github.com) to all files mentioned in the markdown (including line numbers when applicable). I'm going to copy this markdown file out of this repo, so the links need to be absolute links

---

**Cursor**

Good, so the base URL is `https://github.com/openai/codex`. Let me get the current commit SHA for stable links, then update the file.

Now I need to look up the exact line numbers for all the referenced code locations so links are accurate. Let me gather what I need.

Now I have all the line numbers. Let me construct the updated file with absolute GitHub links. I'll use the `main` branch since this is for sharing.

Done. Every file reference in the markdown now has an absolute GitHub link to `https://github.com/openai/codex/blob/main/...` with line number anchors where applicable (e.g. `#L174-L194` for the `ModeKind` enum definition). This includes:

- All inline code references like `` `codex-rs/protocol/src/config_types.rs` ``
- All "Handler in..." and "Defined in..." references
- The comparison table entries for `plan.md` and `default.md`
- The Key Source Files table (every row is now a clickable link with line numbers)
- The prose references to `execute.md`, `gpt-5.2-codex_instructions_template.md`, `stream_events_utils.rs`, etc.

The links use `blob/main/` so they track the main branch. If you'd prefer permalink links pinned to the current commit (`blob/335a4e1c.../`), let me know and I can switch them.

---

**User**

How is the plan exposed to execution mode? How does the update_plan tool work? Is it a way to check off to-do list items? How is an item in the plan referred to when trying to check it off as done? 

---

**Cursor**

Now let me check how the plan gets from Plan mode to Default mode — specifically what happens when the user says "Yes, implement this plan":

Now let me check the base instructions template to see what it says about the plan tool, since that's where Default mode gets its plan tool guidance:

Now I have the complete picture. Here are the answers:

---

These are actually two completely separate systems that unfortunately share the word "plan." Let me clarify both and how the handoff between Plan mode and Default mode works.

## How the plan gets to Default (execution) mode

There is **no structured handoff** mechanism. When the user clicks "Yes, implement this plan" in the TUI popup, the system:

1. Switches the collaboration mode to Default (applies the `default_mode_mask`)
2. Sends the literal string `"Implement the plan."` as a user message

That's it. The model sees the plan because the **raw conversation history** — including the assistant message with `<proposed_plan>` tags — is still in context. The model in Default mode reads its own earlier `<proposed_plan>` output and implements it. There's no special data structure passed between modes; it's all just conversation history.

## How `update_plan` works

`update_plan` is a **completely separate system** from Plan mode. The code comments are explicit about this:

```98:100:codex-rs/core/src/tools/handlers/plan.rs
/// This function doesn't do anything useful. However, it gives the model a structured way to record its plan that clients can read and render.
/// So it's the _inputs_ to this function that are useful to clients, not the outputs and neither are actually useful for the model other
/// than forcing it to come up and document a plan (TBD how that affects performance).
```

`update_plan` is a **checklist/TODO tool** for Default mode. It does not read or write any persistent state. The handler simply:

1. Parses the JSON arguments
2. Emits a `PlanUpdate` event to the TUI
3. Returns the string `"Plan updated"` to the model

The tool schema (defined in [`plan.rs`](https://github.com/openai/codex/blob/main/codex-rs/core/src/tools/handlers/plan.rs#L22-L62)) accepts:

```json
{
  "explanation": "optional string",
  "plan": [
    { "step": "description of the step", "status": "pending" },
    { "step": "another step", "status": "in_progress" },
    { "step": "done step", "status": "completed" }
  ]
}
```

The `status` field is one of three values, defined in [`plan_tool.rs`](https://github.com/openai/codex/blob/main/codex-rs/protocol/src/plan_tool.rs#L9-L13):

```rust
pub enum StepStatus {
    Pending,
    InProgress,
    Completed,
}
```

## How items are "checked off"

There is **no item ID or reference system**. Steps are identified purely by their text content. Every call to `update_plan` provides the **entire plan from scratch** — there's no incremental "check off item X" operation.

For example, the model might first call:

```json
{
  "plan": [
    { "step": "Add Redis client dependency", "status": "in_progress" },
    { "step": "Create session store abstraction", "status": "pending" },
    { "step": "Write migration script", "status": "pending" }
  ]
}
```

Then after completing the first step, it calls `update_plan` again with the **full list**:

```json
{
  "explanation": "Redis client added to Cargo.toml",
  "plan": [
    { "step": "Add Redis client dependency", "status": "completed" },
    { "step": "Create session store abstraction", "status": "in_progress" },
    { "step": "Write migration script", "status": "pending" }
  ]
}
```

The TUI renders this as a checkbox list (from [`history_cell.rs`](https://github.com/openai/codex/blob/main/codex-rs/tui/src/history_cell.rs#L2063-L2115)):

```
• Updated Plan
  └ ✔ Add Redis client dependency          (struck through, dim)
    □ Create session store abstraction      (cyan, bold)  
    □ Write migration script                (dim)
```

## Key distinction

| | Plan mode (`<proposed_plan>`) | `update_plan` tool |
|---|---|---|
| **When** | Plan mode only | Default mode only |
| **Purpose** | Produce a detailed implementation spec | Track progress on steps during execution |
| **Persistence** | Embedded in conversation history as raw text | No persistence — fire-and-forget event |
| **Structure** | Free-form Markdown in `<proposed_plan>` tags | Structured JSON with step/status pairs |
| **State management** | Full replacement each time | Full replacement each time |
| **Item identity** | N/A (prose) | Step text string (no IDs) |
| **Server-side effect** | None (parser strips for display) | None (emits `PlanUpdate` event for UI) |
| **Model return value** | N/A (inline in message) | Always `"Plan updated"` |

The base instructions template ([`gpt-5.2-codex_instructions_template.md`](https://github.com/openai/codex/blob/main/codex-rs/core/templates/model_instructions/gpt-5.2-codex_instructions_template.md#L57-L62)) gives the model guidance on when to use the `update_plan` tool:

> - Skip using the planning tool for straightforward tasks (roughly the easiest 25%).
> - Do not make single-step plans.
> - When you made a plan, update it after having performed one of the sub-tasks that you shared on the plan.

---

**User**

Are generated plans a flat list, or are they nested lists? If flat, how are complex plans handled?

---

**Cursor**

The answer depends on which plan system you mean — they're different.

**`update_plan` tool (Default mode)**: Strictly a **flat list**. The schema is a single-level array of `{ step, status }` objects with no nesting, no sub-steps, no parent references:

```22:46:codex-rs/core/src/tools/handlers/plan.rs
pub static PLAN_TOOL: LazyLock<ToolSpec> = LazyLock::new(|| {
    let mut plan_item_props = BTreeMap::new();
    plan_item_props.insert("step".to_string(), JsonSchema::String { description: None });
    plan_item_props.insert(
        "status".to_string(),
        JsonSchema::String {
            description: Some("One of: pending, in_progress, completed".to_string()),
        },
    );

    let plan_items_schema = JsonSchema::Array {
        description: Some("The list of steps".to_string()),
        items: Box::new(JsonSchema::Object {
            properties: plan_item_props,
            required: Some(vec!["step".to_string(), "status".to_string()]),
            additional_properties: Some(false.into()),
        }),
    };
    // ...
```

There's no `children`, `parent`, or `indent_level` field. `additional_properties: false` means the model can't sneak extra fields in. For complex work, the model handles this by using descriptive step text — e.g. `"Auth: add Redis client dependency"` — or by breaking work into sequential coarse-grained milestones rather than deeply nested sub-tasks. The base instructions reinforce this by saying "do not make single-step plans" and "update it after having performed one of the sub-tasks," steering the model toward a running checklist rather than an exhaustive hierarchy.

**`<proposed_plan>` (Plan mode)**: This is **free-form Markdown**, so it can have arbitrary structure — nested lists, headers, tables, code blocks, whatever the model wants. From the plan mode prompt:

```96:102:codex-rs/core/templates/collaboration_mode/plan.md
When you present the official plan, wrap it in a `<proposed_plan>` block so the client can render it specially:

1) The opening tag must be on its own line.
2) Start the plan content on the next line (no text on the same line as the tag).
3) The closing tag must be on its own line.
4) Use Markdown inside the block.
```

The prompt also specifies what content to include but not how to structure it:

```110:117:codex-rs/core/templates/collaboration_mode/plan.md
plan content should be human and agent digestible. The final plan must be plan-only and include:

* A clear title
* A brief summary section
* Important changes or additions to public APIs/interfaces/types
* Test cases and scenarios
* Explicit assumptions and defaults chosen where needed
```

So in practice: Plan mode produces rich, nested, detailed specs. The `update_plan` checklist in Default mode is intentionally flat — it's a progress tracker, not a spec. Complex plans are handled by the Plan mode's free-form Markdown, and `update_plan` just tracks coarse milestones during execution.

---

**User**

Add answers to my most recent questions to the markdown file. Also compress/reorganize the markdown file to include/highlight the important bits (based on my line of questioning in this whole chat)

---

**Cursor**

Now I'll rewrite the file — reorganized around the key insights from the conversation, with the new content about `update_plan`, plan handoff, and plan structure added.

Done. Here's what changed in the reorganization:

**New content added:**
- Callout box at the top contrasting the two "plan" systems side-by-side (the most common source of confusion)
- **"The `update_plan` Tool"** section covering: how it works (fire-and-forget, no persistence), the JSON schema, how items are "checked off" (full replacement by text, no IDs), TUI rendering with checkbox styles, and why it's intentionally flat
- **"Handoff to Default mode"** section explaining there's no structured handoff — just a mode switch and `"Implement the plan."` string
- **Plan output structure** section noting free-form Markdown vs flat JSON

**Reorganized structure (from most to least important based on your questions):**
1. Two-system comparison table (front and center)
2. End-to-end flow (now includes the handoff and `update_plan` in execution)
3. How prompt layers compose (base instructions vs collaboration mode)
4. Plan mode details (prompt, tools, preset, TUI)
5. Plan iteration and handoff (dual data paths, full replacement, no structured handoff)
6. `update_plan` tool deep dive (schema, checking off, rendering, flat list rationale)
7. Streaming parser (moved down — implementation detail)
8. Type system (moved down — reference material)
9. Config, source files (reference)

**Removed/compressed:**
- Merged redundant Mermaid diagrams (architecture diagram removed, its info folded into the sequence diagram)
- Collapsed verbose code snippets that were just illustrating the same point
- Source files table reorganized by category instead of flat list

---

**User**

Add an analysis of the base prompt 

---

**Cursor**

Now let me check how the `{{ personality }}` placeholder gets resolved and what the different personality values are.

Now let me check what `prompt.md` is (the `BASE_INSTRUCTIONS` for non-gpt-5.2 models) and whether it differs.

Now let me add the base prompt analysis to the markdown file.

Added a **Base Prompt Analysis** section covering:

1. **Identity and personality** — the `{{ personality }}` placeholder mechanism with the three substitution values (default/empty, friendly, pragmatic) and how `get_model_instructions()` resolves it
2. **Output formatting rules** — flat lists only, file reference format, no emojis, GitHub-flavored Markdown
3. **Work presentation rules** — user can't see command output, solution-first for complex changes, suggest next steps
4. **General tool usage** — `rg` preference, `apply_patch` guidance, and the significant git safety section (never revert user changes, no hard resets, non-interactive git only)
5. **Plan tool guidance** — the three rules for `update_plan` in Default mode (skip for easy tasks, no single-step plans, update after each sub-task)
6. **Special request handling** — utility commands, code review mindset
7. **Frontend design guidelines** — the anti-"AI slop" rules
8. **Two base prompt variants** — the 81-line template for `gpt-5.2-codex` vs the 276-line `prompt.md` fallback for other models, and how selection works

