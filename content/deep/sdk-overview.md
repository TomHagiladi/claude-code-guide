# Agent SDK - סקירה

> ה-SDK מאפשר לך להריץ את אותו engine של Claude Code כספרייה בקוד שלך. Python ו-TypeScript. עם agent loop מובנה, tools, context management - הכל.

**מקור:** [agent-sdk/overview](https://code.claude.com/docs/en/agent-sdk/overview)

**⚠️ שם:** עד לא מזמן קראו לו "Claude Code SDK". עכשיו **Claude Agent SDK**. [Migration guide](https://code.claude.com/docs/en/agent-sdk/migration-guide) אם יש לך קוד ישן.

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>ל-Claude Code יש שתי "פנים". הראשונה - אתה פותח טרמינל, מקליד <code>claude</code>, ומדבר איתו. זה מה שרוב האנשים מכירים.</p>

<p>ה-<strong>SDK</strong> זה הפנים השנייה: אותו Claude, אבל כחלק מ<strong>תוכנה שאתה כותב</strong>. אתה בונה בוט שקורא מיילים ועונה, או שירות שמקבל קוד ומתקן באגים - ו-Claude עובד <em>בתוך</em> התוכנה שלך.</p>

<p>היתרון הגדול: <strong>אתה לא צריך לכתוב את הלוגיקה של הסוכן</strong>. Claude כבר יודע לקרוא קבצים, להריץ פקודות, לחפש ברשת - אתה רק אומר לו "תתקן את הבאג ב-auth.py" והוא עושה את כל העבודה אוטונומית.</p>

<p>ה-SDK זמין ב-Python וב-TypeScript. הקבצים הבאים מסבירים איך לקנפג, איך לחבר יכולות משלך, ואיך להעלות ל-production בצורה בטוחה.</p>
</div>

---

## מתי SDK, מתי CLI

| שימוש | כלי |
|-------|-----|
| פיתוח אינטראקטיבי | CLI |
| One-off task | CLI |
| CI/CD pipeline | SDK |
| Custom application | SDK |
| Production automation | SDK |
| שילוב בתוך app קיים | SDK |

אותן יכולות, ממשק אחר. הרבה צוותים: CLI ליום-יום, SDK ל-production.

---

## SDK vs Client SDK של Anthropic

Client SDK (`anthropic`):
```python
response = client.messages.create(...)
while response.stop_reason == "tool_use":
    result = your_tool_executor(response.tool_use)   # ← אתה
    response = client.messages.create(tool_result=result, ...)
```
**אתה מממש את ה-tool loop.**

Agent SDK (`claude_agent_sdk`):
```python
async for message in query(prompt="Fix the bug in auth.py"):
    print(message)   # Claude כבר מטפל בכל tool calls
```
**Claude מטפל autonomously.** קורא קבצים, מריץ shell, עורך - בלי שכתבת tool executor.

---

## התקנה

### TypeScript
```bash
npm install @anthropic-ai/claude-agent-sdk
```

### Python
```bash
pip install claude-agent-sdk
```

---

## Authentication

```bash
export ANTHROPIC_API_KEY=your-key
```

Third-party providers:

| Provider | Env var | Setup |
|----------|---------|-------|
| Amazon Bedrock | `CLAUDE_CODE_USE_BEDROCK=1` | AWS creds |
| Google Vertex AI | `CLAUDE_CODE_USE_VERTEX=1` | GCP creds |
| Microsoft Azure | `CLAUDE_CODE_USE_FOUNDRY=1` | Azure creds |

---

## Hello World

<!-- Python -->
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="What files are in this directory?",
        options=ClaudeAgentOptions(allowed_tools=["Bash", "Glob"])
    ):
        if hasattr(message, "result"):
            print(message.result)

asyncio.run(main())
```

<!-- TypeScript -->
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "What files are in this directory?",
  options: { allowedTools: ["Bash", "Glob"] }
})) {
  if ("result" in message) console.log(message.result);
}
```

`query()` מחזיר **async iterator של messages** - stream של הודעות שמגיעות כ-Claude עובד.

---

## מה כלול

כל מה שעושה את Claude Code חזק - זמין ב-SDK:

| יכולת | ראה |
|-------|-----|
| Built-in tools (Read, Write, Edit, Bash, Grep, Glob, WebSearch, WebFetch, AskUserQuestion, Monitor) | built-in |
| Custom tools (in-process MCP) | [02_custom_tools.md](#/deep/sdk-tools) |
| Hooks (callback-based) | [03_hooks_permissions.md](#/deep/sdk-hooks) |
| Subagents (inline AgentDefinition) | overview + subagents doc |
| MCP servers (external) | MCP via options |
| Permissions (modes + rules + canUseTool) | [03_hooks_permissions.md](#/deep/sdk-hooks) |
| Sessions (resume/fork) | built into options |
| Skills, Commands, CLAUDE.md | via `settingSources: ['project']` |
| Plugins | `plugins` option |
| Structured outputs | JSON Schema / Zod / Pydantic |

---

## ארכיטקטורה בקצרה

```
Your code
    ↓
query(prompt, options)
    ↓
SDK process (Node.js subprocess spawned by Python SDK / native in TS)
    ↓
Agent loop: gather → action → verify
    ↓
Tools (built-in + MCP servers + your in-process tools)
    ↓
Messages streamed back to your code
```

**SDK ב-Python הוא wrapper מעל Node.js** - דורש Node.js מותקן. SDK ב-TypeScript native.

---

## Message types

Async iterator מחזיר הודעות מכמה סוגים:

| Type | משמעות |
|------|---------|
| `SystemMessage` (`init`) | תחילת session; מכיל session_id |
| `AssistantMessage` | הודעה מ-Claude (text + tool_use blocks) |
| `UserMessage` | user input (prompt הראשון, או ב-multi-turn) |
| `ToolUseBlock` (תת-תוכן של Assistant) | call ל-tool: name + input |
| `ToolResultBlock` | תוצאה של tool |
| `ResultMessage` | ההודעה הסופית - `result` text, `subtype` (success/error), usage stats |

### דוגמה: parsing ההודעות

```python
from claude_agent_sdk import AssistantMessage, ResultMessage, ToolUseBlock

async for message in query(prompt="...", options=options):
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if isinstance(block, ToolUseBlock):
                print(f"Tool: {block.name}({block.input})")
    elif isinstance(message, ResultMessage):
        if message.subtype == "success":
            print("Result:", message.result)
        print("Usage:", message.usage)
```

---

## ClaudeAgentOptions - גולמי (פירוט מלא בהמשך)

```python
ClaudeAgentOptions(
    # Tools
    allowed_tools=["Read", "Edit"],           # pre-approve
    disallowed_tools=["Bash"],                # deny
    tools=["Read", "Grep"],                   # which built-ins exist (availability)

    # Model & behavior
    model="claude-sonnet-4-6",
    system_prompt="You are a security auditor",
    append_system_prompt="Always respond in English",
    permission_mode="acceptEdits",

    # Extensions
    mcp_servers={"weather": weather_server},
    agents={"reviewer": AgentDefinition(...)},
    hooks={"PreToolUse": [...]},

    # Session
    resume="session-id",
    fork_session=True,
    continue_conversation=True,

    # Claude Code filesystem features
    setting_sources=["project"],              # טען .claude/ מהפרויקט

    # Advanced
    max_turns=20,
    cwd="/path/to/project",
    add_dirs=["../shared"],
    env={"MY_VAR": "value"},
    plugins=[...]
)
```

TypeScript - שמות camelCase (`allowedTools`, `disallowedTools`, `settingSources`).

---

## Streaming

Default = streaming. ההודעות מגיעות כ-Claude עובד, לא בסוף.

```python
async for message in query(...):
    # תגובה ב-real-time
    handle(message)
```

למי שרוצה לקבל הכל בסוף: אסוף ל-list.

[04_production.md](#/deep/sdk-production) מכסה streaming מתקדם ו-"streaming input mode" למולטי-turn.

---

## Multi-turn sessions

### Resume

```python
# turn 1 - capture session_id
async for message in query(prompt="Read auth.py", options=ClaudeAgentOptions(
    allowed_tools=["Read", "Glob"]
)):
    if isinstance(message, SystemMessage) and message.subtype == "init":
        session_id = message.data["session_id"]

# turn 2 - resume with full context
async for message in query(
    prompt="Find all places that call it",
    options=ClaudeAgentOptions(resume=session_id)
):
    ...
```

### Fork

```python
options = ClaudeAgentOptions(resume=session_id, fork_session=True)
# חדש session ID, אותה היסטוריה
```

---

## תיעוד רשמי ו-reference

| כלי | Reference |
|------|-----------|
| Python API | [python](https://code.claude.com/docs/en/agent-sdk/python) |
| TypeScript API | [typescript](https://code.claude.com/docs/en/agent-sdk/typescript) |
| TS V2 preview (simplified) | [typescript-v2-preview](https://code.claude.com/docs/en/agent-sdk/typescript-v2-preview) |
| Quickstart | [quickstart](https://code.claude.com/docs/en/agent-sdk/quickstart) |
| Demos | [github.com/anthropics/claude-agent-sdk-demos](https://github.com/anthropics/claude-agent-sdk-demos) |
| Changelog TS | [CHANGELOG.md](https://github.com/anthropics/claude-agent-sdk-typescript/blob/main/CHANGELOG.md) |
| Changelog Python | [CHANGELOG.md](https://github.com/anthropics/claude-agent-sdk-python/blob/main/CHANGELOG.md) |

---

## Branding (כשבונים מוצר)

**מותר:**
- "Claude Agent" (מומלץ ל-dropdown menus)
- "Claude" (כשכבר ב-menu בשם "Agents")
- "{YourName} Powered by Claude"

**אסור:**
- "Claude Code" / "Claude Code Agent"
- Visual elements שמחקים Claude Code

---

## המשך → [01_api_and_flow.md](#/deep/sdk-api-flow) - ClaudeAgentOptions מלא, streaming, sessions
