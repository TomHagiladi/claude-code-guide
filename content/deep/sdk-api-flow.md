# API & Flow - ClaudeAgentOptions, messages, sessions

> המדריך המעמיק למה שעובר בתוך `query()` וכל האופציות.

**מקורות:** [overview](https://code.claude.com/docs/en/agent-sdk/overview) · [sessions](https://code.claude.com/docs/en/agent-sdk/sessions) · [streaming-output](https://code.claude.com/docs/en/agent-sdk/streaming-output) · [streaming-vs-single-mode](https://code.claude.com/docs/en/agent-sdk/streaming-vs-single-mode)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>כשאתה בונה תוכנה שמשתמשת ב-Claude, אתה צריך להגיד לו <strong>איך להתנהג</strong>: איזה מודל להשתמש (מהיר וזול, או חכם ויקר), אילו כלים מותר לו לקרוא, אילו הגבלות יש לו, איך להתייחס לשיחה.</p>

<p>הקובץ הזה מסביר את <strong>רשימת ההגדרות</strong> שיש ל-SDK - כל אופציה, מה היא עושה, ומתי לבחור מה. זה כמו <strong>לוח בקרה</strong> של האגן שאתה בונה.</p>

<p>בנוסף: איך לקבל תשובות <strong>בזרם חי</strong> (רואה את הטקסט מופיע בהדרגה, כמו ב-ChatGPT), איך להמשיך שיחה מפעם קודמת, ואיך לקבל תשובות <strong>מובנות</strong> (JSON שאפשר להסתמך עליו במקום טקסט חופשי).</p>
</div>

---

## שתי צורות של API

### 1. `query()` - one-shot

```python
async for message in query(prompt="...", options=options):
    ...
```

מתאים ל:
- משימה בודדת
- CI/CD
- scripting

### 2. Client/session-based (V2 preview, TS)

```typescript
import { ClaudeAgent } from "@anthropic-ai/claude-agent-sdk";

const agent = new ClaudeAgent(options);
const response1 = await agent.send("First message");
const response2 = await agent.send("Follow-up");
```

מתאים ל:
- multi-turn conversations
- stateful apps
- chat interfaces

ראה: [typescript-v2-preview](https://code.claude.com/docs/en/agent-sdk/typescript-v2-preview).

---

## ClaudeAgentOptions - פירוט מלא

### Tools

```python
allowed_tools: list[str]          # pre-approved (no prompt)
disallowed_tools: list[str]       # always denied
tools: list[str]                  # availability layer (built-ins)
```

### Model

```python
model: str                        # "sonnet", "opus", "haiku", or full ID
fast_mode: bool                   # fast mode (Opus)
effort: str                       # "low", "medium", "high", "max"
```

### System prompt

```python
system_prompt: str                # REPLACES default system prompt
append_system_prompt: str         # APPENDS to default
```

**הבדל חשוב:** `system_prompt` **מוחק** את system prompt של Claude Code. אתה יוצר agent חדש מאפס. `append_system_prompt` **מוסיף** להוראות הקיימות - שומר את היכולות של Claude Code.

ראה [modifying-system-prompts](https://code.claude.com/docs/en/agent-sdk/modifying-system-prompts).

### Permissions

```python
permission_mode: str              # "default", "acceptEdits", "plan", "auto", "dontAsk", "bypassPermissions"
can_use_tool: Callable            # custom permission callback
permissions: dict                 # rules
```

### MCP & extensions

```python
mcp_servers: dict                 # in-process + external
agents: dict[str, AgentDefinition]   # custom subagents
hooks: dict[str, list[HookMatcher]]  # callback hooks
plugins: list                        # plugin configs
```

### Session

```python
resume: str                       # session_id to resume
fork_session: bool                # fork instead of continue
continue_conversation: bool       # continue latest in cwd
```

### Filesystem

```python
cwd: str                          # working directory
add_dirs: list[str]               # additional directories
setting_sources: list[str]        # ["project", "user", "local"] - load .claude/ files
```

### Runtime

```python
max_turns: int                    # safeguard
env: dict                         # env vars for subprocesses
stderr_callback: Callable         # stderr from SDK
user: str                         # for analytics
```

---

## Message types - פירוט

### SystemMessage

- **`subtype: "init"`** - תחילת session. מכיל `session_id`, model, tools available.
- **`subtype: "compact"`** - compaction קרה.
- **`subtype: "shutdown"`** - session מסתיים.

```python
if isinstance(message, SystemMessage) and message.subtype == "init":
    session_id = message.data["session_id"]
```

### AssistantMessage

הודעה מ-Claude. `content` = list של blocks:

- `TextBlock` - טקסט רגיל
- `ToolUseBlock` - Claude קורא ל-tool. `name`, `input`, `id`
- `ThinkingBlock` - extended thinking (אם פעיל)

```python
if isinstance(message, AssistantMessage):
    for block in message.content:
        if isinstance(block, ToolUseBlock):
            print(f"Calling {block.name} with {block.input}")
        elif isinstance(block, TextBlock):
            print(block.text)
```

### UserMessage

הודעה שה-user שלח (גם ה-prompt הראשון). ב-multi-turn אתה רואה גם אותם.

### ToolResultBlock

תוצאה של tool call. מוצג בהקשר של UserMessage שמגיע אוטומטית אחרי AssistantMessage עם ToolUseBlock.

### ResultMessage

**הודעה סופית.** מגיעה פעם אחת ב-query, בסוף.

```python
if isinstance(message, ResultMessage):
    print(message.result)           # text סופי של Claude
    print(message.subtype)          # "success", "error_max_turns", "error_during_execution"
    print(message.usage)            # token counts
    print(message.total_cost_usd)   # עלות
    print(message.duration_ms)
    print(message.num_turns)
```

---

## Streaming modes

### Single prompt mode (default)

```python
async for message in query(prompt="fix the bug", options=options):
    ...
```

Prompt אחד, Claude עובד עד שגמר, ResultMessage מסיים. session נסגר.

### Streaming input mode (multi-turn)

```typescript
async function* prompts() {
  yield { type: "user", message: { role: "user", content: "first" } };
  // wait for Claude...
  yield { type: "user", message: { role: "user", content: "follow-up" } };
}

for await (const message of query({ prompt: prompts(), options })) {
  ...
}
```

אתה מספק async generator של user messages. Session נשאר פתוח, Claude עונה על כל אחד. יש לך שליטה על **מתי** להכניס message הבא.

Use case: chat interface, interactive review, HITL.

---

## Sessions - עמוק

### Capture session_id

```python
session_id = None
async for message in query(prompt="...", options=options):
    if isinstance(message, SystemMessage) and message.subtype == "init":
        session_id = message.data["session_id"]
```

### Resume

```python
options = ClaudeAgentOptions(resume=session_id)
# context הקודם נטען, ממשיך
```

### Fork

```python
options = ClaudeAgentOptions(resume=session_id, fork_session=True)
# session חדש עם היסטוריה, לא משפיע על המקורי
```

### Continue latest

```python
options = ClaudeAgentOptions(continue_conversation=True)
# session האחרון ב-cwd
```

### Storage

```
~/.claude/projects/<cwd-hash>/<session-id>.jsonl
```

Plaintext. ניתן לקרוא.

---

## `canUseTool` - custom permission callback

במקום rules סטטיים, פונקציה שמחליטה per-call:

```python
async def can_use_tool(tool_name: str, tool_input: dict, context) -> dict:
    if tool_name == "Bash" and "rm" in tool_input.get("command", ""):
        return {"behavior": "deny", "message": "rm blocked by policy"}
    if tool_name == "WebFetch":
        if not tool_input["url"].startswith("https://api.ourcompany.com"):
            return {"behavior": "deny", "message": "Only our API allowed"}
    return {"behavior": "allow", "updated_input": tool_input}

options = ClaudeAgentOptions(can_use_tool=can_use_tool)
```

Return values:
- `{"behavior": "allow", "updated_input": {...}}` - allow (עם שינוי אופציונלי)
- `{"behavior": "deny", "message": "reason"}` - חסום

**שונה מ-hooks:** canUseTool חוזר sync-style decision. Hooks יכולים גם להזריק context, לרוץ async ברקע, לפעול ב-events אחרים (PostToolUse, SessionStart, וכו').

---

## Claude Code filesystem features

**Default:** SDK לא טוען `.claude/` של הפרויקט. זה בכוונה - SDK agent שונה מ-Claude Code CLI.

**להפעיל:**
```python
options = ClaudeAgentOptions(
    setting_sources=["project"]
)
```

תומך: `"project"`, `"user"`, `"local"`.

טוען:
- `CLAUDE.md` / `.claude/CLAUDE.md`
- `.claude/rules/`
- `.claude/skills/`
- `.claude/commands/`
- `.claude/agents/`
- `.claude/settings.json` (permissions וכו')

---

## Plugins ב-SDK

```python
options = ClaudeAgentOptions(
    plugins=[
        {"type": "directory", "path": "./my-plugin"},
        {"type": "marketplace", "source": "org/marketplace", "plugin": "plugin-name"}
    ]
)
```

ראה [agent-sdk/plugins](https://code.claude.com/docs/en/agent-sdk/plugins).

---

## Structured outputs

Return validated JSON במקום טקסט חופשי:

```python
from pydantic import BaseModel

class BugReport(BaseModel):
    file: str
    line: int
    severity: str

options = ClaudeAgentOptions(
    output_schema=BugReport
)

async for message in query(prompt="Find bugs in auth.py", options=options):
    if isinstance(message, ResultMessage):
        bug: BugReport = message.structured_result
```

תומך: JSON Schema, Zod, Pydantic. ראה [structured-outputs](https://code.claude.com/docs/en/agent-sdk/structured-outputs).

---

## Todo tracking

SDK כולל TodoWrite tool built-in. Claude מנהל todos, אתה יכול לראות ב-stream:

```python
async for message in query(...):
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if isinstance(block, ToolUseBlock) and block.name == "TodoWrite":
                update_ui(block.input["todos"])
```

---

## File checkpointing ב-SDK

אותו מנגנון כמו CLI. לפני Edit, snapshot אוטומטי. ב-SDK יש API ל-restore:

```python
from claude_agent_sdk import restore_checkpoint

restore_checkpoint(session_id="...", checkpoint_id="...")
```

ראה [file-checkpointing](https://code.claude.com/docs/en/agent-sdk/file-checkpointing).

---

## המשך → [02_custom_tools.md](#/deep/sdk-tools) · [03_hooks_permissions.md](#/deep/sdk-hooks)
