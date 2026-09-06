# Hooks & Permissions ב-SDK

> ב-CLI hooks הם shell commands. ב-SDK הם **callback functions** - JavaScript/Python functions שרצות בתהליך שלך. שינוי מנטלי חשוב.

**מקורות:** [agent-sdk/hooks](https://code.claude.com/docs/en/agent-sdk/hooks) · [agent-sdk/permissions](https://code.claude.com/docs/en/agent-sdk/permissions) · [agent-sdk/user-input](https://code.claude.com/docs/en/agent-sdk/user-input)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>ב-CLI הרגיל, hooks הם <strong>סקריפטים</strong> שרצים באירועים. ב-SDK זה <strong>פונקציות שאתה כותב בקוד שלך</strong> - קוד שירוץ לפני שClaude עושה משהו, או אחרי.</p>

<p>למשל: <strong>רישום פעולות ל-log</strong> לפני כל שינוי קובץ. <strong>חסימת עריכות</strong> בקבצי production. <strong>בדיקת עלות</strong> ועצירה אחרי תקציב שנגמר. <strong>שינוי פרמטרים</strong> לפני הרצה (Claude רצה לקרוא ל-API - אתה מחליף את המפתח בוורסיה הנכונה).</p>

<p>הרשאות ב-SDK יותר דינמיות מאשר ב-CLI: אפשר להגיד <strong>"בדוק בכל קריאה"</strong> עם פונקציה שמחליטה per-call - למשל "מותר לגשת רק ל-URL של החברה שלנו".</p>

<p>זה הלב של <strong>הגנה ב-production</strong>: כל פעולה חשובה עוברת דרך הקוד שלך לפני שClaude מבצע.</p>
</div>

---

## Hooks ב-SDK - callbacks

```python
from claude_agent_sdk import query, ClaudeAgentOptions, HookMatcher
from datetime import datetime

async def log_file_change(input_data, tool_use_id, context):
    file_path = input_data.get("tool_input", {}).get("file_path", "unknown")
    with open("./audit.log", "a") as f:
        f.write(f"{datetime.now()}: {file_path}\n")
    return {}

options = ClaudeAgentOptions(
    permission_mode="acceptEdits",
    hooks={
        "PostToolUse": [
            HookMatcher(matcher="Edit|Write", hooks=[log_file_change])
        ]
    }
)
```

TypeScript:
```typescript
const logFileChange: HookCallback = async (input) => {
  const filePath = (input as any).tool_input?.file_path ?? "unknown";
  await appendFile("./audit.log", `${new Date().toISOString()}: ${filePath}\n`);
  return {};
};

for await (const message of query({
  prompt: "Refactor utils.py",
  options: {
    permissionMode: "acceptEdits",
    hooks: {
      PostToolUse: [{ matcher: "Edit|Write", hooks: [logFileChange] }]
    }
  }
})) { ... }
```

---

## Events זמינים

רוב events מה-CLI זמינים גם ב-SDK: `PreToolUse`, `PostToolUse`, `Stop`, `SessionStart`, `SessionEnd`, `UserPromptSubmit`, `SubagentStart`, `SubagentStop`, `PreCompact`, `PostCompact`, וכו'.

[hooks reference של CLI](#/deep/hooks) - עובד לרוב המוחלט גם כאן. השינוי: מ-shell process → callback function.

---

## Return value - structured

```python
return {
    "hookSpecificOutput": {
        "permissionDecision": "allow",   # PreToolUse
        "additionalContext": "info to Claude",
        "updatedInput": {...}             # modify tool call
    },
    "continue": True,                     # False → stop session
    "stopReason": "reason",
    "suppressOutput": False,
    "systemMessage": "warning"
}
```

**החזר dict ריק `{}`** = ה-hook רץ, לא שינה כלום.

---

## Hook matcher - שמות tools

```python
HookMatcher(matcher="Edit|Write", hooks=[callback])
HookMatcher(matcher="mcp__.*", hooks=[...])          # regex
HookMatcher(matcher="*", hooks=[...])                 # הכל
```

אותו syntax כמו CLI.

---

## Permissions ב-SDK - שלוש שכבות

### 1. Pre-approved lists

```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Grep", "mcp__weather__*"],
    disallowed_tools=["Bash"]
)
```

### 2. Permission mode

```python
permission_mode="default"           # default - prompt על כל חדש
permission_mode="acceptEdits"       # accept edits + common fs
permission_mode="plan"              # read-only
permission_mode="auto"              # classifier-based
permission_mode="bypassPermissions" # skip (זהירות!)
```

### 3. `canUseTool` callback - dynamic

```python
async def can_use_tool(tool_name, tool_input, context):
    # context - session_id, previous messages, etc.

    if tool_name == "Bash":
        cmd = tool_input.get("command", "")
        if any(danger in cmd for danger in ["rm -rf", "dd if=", "mkfs"]):
            return {"behavior": "deny", "message": "Dangerous command blocked"}

    if tool_name == "WebFetch":
        url = tool_input.get("url", "")
        if not url.startswith(("https://api.ourcompany.com/", "https://docs.")):
            return {"behavior": "deny", "message": "Domain not allowed"}

    if tool_name == "Edit":
        file = tool_input.get("file_path", "")
        if "/prod/" in file:
            return {"behavior": "deny", "message": "prod files protected"}

    return {"behavior": "allow", "updated_input": tool_input}

options = ClaudeAgentOptions(can_use_tool=can_use_tool)
```

**הערה:** `canUseTool` לא תומך כשב-bypassPermissions mode.

---

## Order of evaluation

```
can_use_tool callback
      ↓
permissions.deny rules (matched → block)
      ↓
Hook PreToolUse (exit 2 → block)
      ↓
permissions.allow rules (matched → allow)
      ↓
permission_mode (default → prompt, acceptEdits → auto, etc.)
      ↓
User prompt (if needed)
      ↓
Execute tool
```

**Deny תמיד גובר.** גם canUseTool שמחזיר allow - permissions.deny שאחרי יכול לחסום.

---

## User input - AskUserQuestion

Built-in tool של SDK. Claude שואל שאלה עם multiple choice:

```python
async for message in query(prompt="Plan migration", options=ClaudeAgentOptions(
    allowed_tools=["AskUserQuestion", "Read", "Edit"]
)):
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if isinstance(block, ToolUseBlock) and block.name == "AskUserQuestion":
                # Claude asking you something
                question = block.input["question"]
                options = block.input["options"]   # list of choices

                # handle in UI, send back via ToolResultBlock
                user_choice = await show_prompt(question, options)
```

אחרי ה-user answer, SDK ממשיך את הsession. אתה אחראי להציג UI ולהחזיר תשובה.

---

## Permission prompts - interactive

במקום polling messages, יש callback ייעודי:

```python
async def on_permission_request(tool_name, tool_input, suggestions):
    # הצג UI, קבל תשובה
    approved = await show_permission_dialog(tool_name, tool_input)
    return {
        "behavior": "allow" if approved else "deny",
        "updated_input": tool_input,
        "updated_permissions": [
            {
                "type": "addRules",
                "behavior": "allow",
                "destination": "session"    # or localSettings, projectSettings
            }
        ] if approved else []
    }

options = ClaudeAgentOptions(on_permission_request=on_permission_request)
```

`updated_permissions` מאפשר "yes, don't ask again" - הוספת rule לsession/project/user.

---

## Patterns נפוצים

### 1. Audit log

```python
async def audit(input_data, tool_use_id, context):
    with open("audit.jsonl", "a") as f:
        f.write(json.dumps({
            "time": datetime.now().isoformat(),
            "session": context.get("session_id"),
            "tool": input_data.get("tool_name"),
            "input": input_data.get("tool_input"),
        }) + "\n")
    return {}

hooks={"PreToolUse": [HookMatcher(matcher="*", hooks=[audit])]}
```

### 2. Block production writes

```python
async def block_prod(input_data, tool_use_id, context):
    file = input_data.get("tool_input", {}).get("file_path", "")
    if "/production/" in file or ".prod." in file:
        return {
            "hookSpecificOutput": {
                "permissionDecision": "deny",
                "permissionDecisionReason": "Production files protected"
            }
        }
    return {}

hooks={"PreToolUse": [HookMatcher(matcher="Edit|Write", hooks=[block_prod])]}
```

### 3. Auto-inject context on SessionStart

```python
async def inject_context(input_data, tool_use_id, context):
    return {
        "hookSpecificOutput": {
            "additionalContext": f"Current user: {get_user()}\nTimezone: {get_tz()}"
        }
    }

hooks={"SessionStart": [HookMatcher(hooks=[inject_context])]}
```

### 4. Rate limit external APIs

```python
from collections import defaultdict
call_counts = defaultdict(int)

async def rate_limit(input_data, tool_use_id, context):
    tool = input_data.get("tool_name")
    if tool == "WebFetch":
        call_counts["WebFetch"] += 1
        if call_counts["WebFetch"] > 50:
            return {
                "hookSpecificOutput": {
                    "permissionDecision": "deny",
                    "permissionDecisionReason": "Rate limit: 50 web fetches per session"
                }
            }
    return {}
```

### 5. Cost cap

```python
total_cost = [0.0]

async def cost_cap(input_data, tool_use_id, context):
    usage = context.get("usage", {})
    total_cost[0] += usage.get("cost_usd", 0)
    if total_cost[0] > 5.0:
        return {"continue": False, "stopReason": "$5 cost cap reached"}
    return {}

hooks={"PostToolUse": [HookMatcher(matcher="*", hooks=[cost_cap])]}
```

---

## המשך → [04_production.md](#/deep/sdk-production) - hosting, secure deployment, observability, cost tracking
