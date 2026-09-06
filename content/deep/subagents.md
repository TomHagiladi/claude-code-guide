# Subagents - חלונות קונטקסט נפרדים

> Subagent = instance של Claude עם context window נפרד, system prompt משלו, ורשימת tools מוגבלת. הוא עושה עבודה, חוזר עם summary בלבד. המחיר שלך: ה-summary. הרווח: עבודה כבדה שלא מציפה את ה-context הראשי.

**מקור:** [sub-agents](https://code.claude.com/docs/en/sub-agents)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>תחשוב על מנהל פרויקט שעובד על משימה מורכבת. במקום לעשות הכל לבד, הוא <strong>שולח עוזר לחקור משהו</strong> ולחזור עם סיכום. העוזר עושה את כל העבודה הכבדה - קורא 20 קבצים, מסכם - ומחזיר רק את התשובה הקצרה.</p>

<p><strong>Subagents בדיוק זה</strong>: Claude יכול "לזמן" עוזר עם משימה ספציפית. העוזר עובד ב<strong>חלון נפרד</strong> - כל הקריאות שלו לא מבלבלות את השיחה הראשית. בסוף הוא מחזיר רק תקציר.</p>

<p>שימושים: מחקר עמוק בקוד שלא רוצים לזכור הכל ממנו, code reviewer מומחה, debugger ייעודי, או עבודה במקביל ("חקור את A, B, ו-C במקביל").</p>

<p>אפשר להגדיר עוזרים מותאמים - למשל "סוקר אבטחה" שמכיר רק את הקוד, או "מומחה DB" שמותר לו להריץ רק שאילתות קריאה.</p>
</div>

---

## מתי להשתמש

- **Research heavy** - "איך מטופל timeout בקוד הזה?" → קריאה של 10 קבצים שלא יחזרו אליהם.
- **Specialized role** - code-reviewer, security-auditor, data-scientist.
- **Constrained task** - "רק read, שום write" - subagent עם `tools: Read Grep Glob`.
- **Cost routing** - משימה פשוטה → Haiku subagent במקום Opus של session ראשי.
- **Parallel work** - 3 subagents במקביל לחקור 3 קצוות של משימה.

**אל תשתמש אם:** המשימה קצרה, מצריכה context מהשיחה הראשית, או שהתוצאה צריכה להשפיע ישירות (edit files במשך zero-context).

---

## Built-in subagents

Claude Code מגיע עם שלושה:

### Explore

- **כלים:** read-only (Read, Grep, Glob, WebFetch, וכו')
- **מתי:** Claude מאציל אליו כשהוא צריך לחפש/להבין codebase בלי לערוך.
- **Thoroughness levels:** `quick` / `medium` / `very thorough` - Claude מציין כשקורא.
- **תוצאה:** summary של מה שנמצא.

### Plan

- **כלים:** read-only, אבל בהקשר plan mode.
- **מתי:** בתוך plan mode, כש-Claude צריך לחקור לפני יצירת plan.
- **למה זה נפרד:** Plan mode שהוא גם ב-main + גם ב-Explore היה יוצר nesting אינסופי.

### general-purpose

- **כלים:** כל מה שיש לsession הראשי.
- **מתי:** משימות שדורשות exploration + modification + reasoning.
- **דוגמה:** "תבנה endpoint חדש לפי הדפוס שב-X" - כולל קריאה, כתיבה, ריצה.

---

## יצירת subagent מותאם

### איפה

| Scope | Path |
|-------|------|
| Managed | `.claude/agents/` ב-managed settings |
| User | `~/.claude/agents/<name>.md` |
| Project | `.claude/agents/<name>.md` |
| Plugin | `<plugin>/agents/<name>.md` |

קדימות: Managed > Project > User > Plugin.

### מבנה קובץ

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Grep, Glob, Bash
model: sonnet
---

You are a code reviewer. Your job is...

Rules you must follow...
```

**רק `name` ו-`description` חובה.** ה-markdown body = system prompt של ה-subagent.

---

## Frontmatter fields - מלא

| Field | משמעות |
|-------|---------|
| `name` | lowercase + hyphens. ייחודי. |
| `description` | מתי Claude יאציל אליו - חשוב שיהיה ספציפי |
| `tools` | **allowlist** - רק אלה זמינים |
| `disallowedTools` | **denylist** - מוריד מ-inherited או מ-tools |
| `model` | `sonnet` / `opus` / `haiku` / full ID / `inherit` |
| `permissionMode` | `default` / `acceptEdits` / `auto` / `dontAsk` / `bypassPermissions` / `plan` |
| `maxTurns` | max turns לפני עצירה |
| `skills` | skills ל-preload - התוכן **מוזרק**, לא רק זמין |
| `mcpServers` | MCP scoped ל-subagent |
| `hooks` | hooks ב-lifetime של ה-subagent |
| `memory` | `user` / `project` / `local` - persistent |
| `background` | `true` → תמיד background task |
| `effort` | `low` / `medium` / `high` / `max` |
| `isolation` | `worktree` → worktree זמני (copy isolated) |
| `color` | display color - `red`, `blue`, `green`, וכו' |
| `initialPrompt` | auto-submitted כ-first user turn (עם `--agent`) |

### Resolution order למודל

1. `CLAUDE_CODE_SUBAGENT_MODEL` env var
2. Per-invocation `model` parameter (Claude בוחר)
3. Frontmatter `model`
4. Main conversation's model
5. Default

---

## Tools - אישור ושלילה

### Allowlist (`tools`)

```yaml
tools: Read, Grep, Glob, Bash
```

Subagent יכול רק את אלה. לא יכול Edit, Write, MCP tools, Agent.

### Denylist (`disallowedTools`)

```yaml
disallowedTools: Write, Edit
```

יורש הכל, חוץ מהאלה. שומר Bash, MCP tools, כל השאר.

### שניהם

`disallowedTools` מוחל קודם, `tools` resolved אחר כך. tool ב-שניהם → removed.

### Agent tool syntax

```yaml
tools: Agent(worker, researcher), Read, Bash
```

רק `worker` ו-`researcher` יכולים להיות spawned. עד v2.1.172 זה היה רלוונטי **רק ל-main thread subagent** (`claude --agent`) - subagents רגילים לא יכלו spawn אחרים (מניעת nesting). מ-יוני 2026 גם הם יכולים - ראה Recursive subagents למטה.

`tools: Agent` (בלי סוגריים) = כל ה-subagents מותרים.

`Agent` מושמט מ-tools = אי אפשר spawn בכלל.

---

## Recursive subagents (יוני 2026)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>עד עכשיו רק השיחה הראשית יכלה "לזמן עוזרים". העוזרים עצמם ישבו בקצה השרשרת - הם עשו את העבודה לבד והחזירו סיכום, ולא יכלו לזמן עוזרים משלהם.</p>

<p>מ-יוני 2026 זה השתנה: <strong>עוזר יכול עכשיו לזמן עוזרים משלו</strong>. תחשוב על מנהל שממנה ראש-צוות, וראש-הצוות ממנה לעצמו בנאי וסוקר. כל שכבה מקבלת חלון נקי משלה, ורק הסיכום עולה כלפי מעלה.</p>

<p>זה מאפשר ל-coordinator אחד לנהל שרשרת builder→reviewer משלו, במקום שכל העוזרים יישבו "שטוחים" באותה רמה תחת השיחה הראשית.</p>
</div>

### מה השתנה

לפני **v2.1.172**, subagent רגיל לא יכל spawn subagent אחר - ה-nesting נחסם כדי למנוע לולאות אינסופיות. ב-**v2.1.172** (8-12 ביוני 2026) זה נפתח: subagents יכולים עכשיו ליצור subagents משלהם - **recursive, עד 5 רמות עומק** ב-background chains.

```
main session
└─ coordinator (subagent)
   ├─ builder   (subagent של coordinator)
   └─ reviewer  (subagent של coordinator)
```

במקום שכל agent יהיה flattened ל-top level תחת ה-session הראשי, ה-coordinator מאציל ל-chain פרטי משלו. כל רמה = context window נקי, ורק ה-summary עולה לרמה שמעליה.

כדי לאפשר את זה, ה-subagent צריך את `Agent` ב-`tools` שלו (אותו syntax מ-"Agent tool syntax" למעלה) - בלי זה הוא עדיין לא יכול spawn.

### Depth tracking

**v2.1.181** תיקן את ספירת העומק כך שתעבוד נכון גם כש-subagents עוברים resume או fork. לפני התיקון, subagent שחודש (resume) או פוצל (fork) היה עלול לאבד את ספירת ה-5-רמות - ולחרוג ממנה בלי לדעת.

### מתי זה עוזר

- **Coordinator pattern** - agent מתאם שמחלק עבודה ל-sub-chain משלו (builder, reviewer, tester) בלי להציף את ה-main thread.
- **שרשראות עמוקות** - research → plan → implement, כשכל שלב בעצמו צריך לפצל עבודה לכמה עוזרים.

---

## MCP scoped

```yaml
mcpServers:
  - playwright:                       # inline
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
  - github                            # reference existing
```

- Inline → רק ל-subagent הזה. מתחבר בהתחלה, מתנתק בסוף.
- Reference string → משתמש בחיבור הקיים של session.

**Pro tip:** MCP server עם הרבה tools שרלוונטיים רק ל-subagent ספציפי - define inline. שומר את ה-context הראשי נקי מ-tool definitions.

---

## Permission modes

| Mode | התנהגות |
|------|----------|
| `default` | שאל לפני edits/shell |
| `acceptEdits` | אוטו-edits ו-fs commands (mkdir, mv), שאר שואל |
| `auto` | classifier ברקע, auto-decides |
| `dontAsk` | דחה הכל (allowed tools עדיין עובדים) |
| `bypassPermissions` | דלג על כל approval |
| `plan` | read-only mode |

**Parent mode דורס:**
- Parent `bypassPermissions` → תמיד גובר.
- Parent `auto` → subagent יורש auto, frontmatter `permissionMode` מתעלם.

---

## Preload skills

```yaml
skills:
  - api-conventions
  - error-handling
```

**התוכן המלא** של כל skill מוזרק ל-system prompt של subagent. לא רק זמין - מוזרק.

Subagents **לא יורשים skills מ-parent** - חייב לרשום.

### ההבדל מ-`context: fork` ב-skill

| גישה | system prompt | task | נוסף |
|------|---------------|------|------|
| Skill עם `context: fork` | של ה-agent type | skill content | CLAUDE.md |
| Subagent עם `skills:` | subagent body | delegation message | preloaded skills + CLAUDE.md |

---

## Persistent memory

```yaml
memory: project
```

| Scope | Location |
|-------|----------|
| `user` | `~/.claude/agent-memory/<name>/` |
| `project` | `.claude/agent-memory/<name>/` |
| `local` | `.claude/agent-memory-local/<name>/` |

כש-memory פעיל:
- System prompt כולל הוראות לקרוא/לכתוב ב-directory.
- 200 שורות ראשונות של `MEMORY.md` מוזרקות.
- Read, Write, Edit מופעלים אוטומטית.

**Use case:** code-reviewer שבונה ידע על codebase לאורך זמן. בקש: "Check your memory before reviewing. Update after."

---

## Conditional rules עם hooks

Subagent שרק מאפשר read-only queries:

```yaml
---
name: db-reader
tools: Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-readonly.sh"
---
```

`validate-readonly.sh`:
```bash
#!/bin/bash
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
if echo "$CMD" | grep -iE '\b(INSERT|UPDATE|DELETE|DROP)\b' > /dev/null; then
  echo "Blocked: read-only only" >&2
  exit 2
fi
exit 0
```

---

## Invocation

### אוטומטי

Claude קורא את ה-description, מחליט להאציל. "Research how sessions work" → מאציל ל-Explore.

### מפורש

```
> Use the code-reviewer subagent to check my auth changes
```

### `claude --agent <name>`

**Session הראשי עצמו** לובש את ה-subagent system prompt, tools, permissions, model. שימושי להרצת CI/CD או לתפקידים ייעודיים.

### JSON inline

```bash
claude --agents '{"code-reviewer": {"description": "...", "prompt": "..."}}'
```

---

## Disable specific

```json
{
  "permissions": {
    "deny": ["Agent(Explore)", "Agent(my-custom)"]
  }
}
```

או CLI:
```bash
claude --disallowedTools "Agent(Explore)"
```

---

## Patterns

### Foreground vs background

```yaml
background: true
```

Default: foreground - Claude מחכה לתוצאה לפני המשך. Background: ממשיך בינתיים, מתקבל notification כשגמר.

שימוש: subagent לחיפוש ארוך ברקע בזמן שאתה עובד על משהו אחר.

### Background permission prompts (יוני 2026)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>כש-subagent רץ ברקע ונתקל בפעולה שדורשת אישור (edit, shell, או משהו שה-guard חוסם), פעם הוא היה פשוט "נתקע בשקט" - אף אחד לא ראה שהוא מבקש רשות, אז הבקשה נדחתה אוטומטית והריצה עצרה בלי שום הסבר.</p>

<p>מ-25 ביוני 2026 זה תוקן: ה-subagent ברקע <strong>מקפיץ את בקשת האישור בשיחה הראשית שלך</strong>, עם ציון איזה agent מבקש. אתה מאשר או דוחה, והריצה ממשיכה.</p>
</div>

ב-**v2.1.193** (25 ביוני 2026), background subagents מציגים את ה-permission dialogs שלהם **ב-main session** במקום auto-deny שקט.

- ה-dialog מציין **איזה agent מבקש** - כדי שתדע מי תקוע ועל מה.
- `Esc` דוחה רק את ה-tool הספציפי הזה, לא את כל הריצה.
- לפני התיקון: פעולה ש-guard חסם הייתה הופכת ל-**stall שקט** של הריצה ברקע - בלי שתדע למה כלום לא מתקדם.

### Worktree isolation

```yaml
isolation: worktree
```

Subagent מקבל worktree זמני עם העתק של ה-repo. שינויים שלו לא משפיעים על cwd שלך עד שאתה מאשר merge.

**אוטו-cleanup** אם ה-subagent לא עשה שינויים. אחרת - path + branch מוחזרים.

### Parallel research

```
> Research X, Y, and Z in parallel using three subagents
```

Claude spawns 3 במקביל, משלב summaries. חיסכון זמן מרשים למשימות מפוצלות.

### Chain

Subagent 1 חוקר → Subagent 2 מתכנן → Subagent 3 מממש. כל אחד context נקי.

---

## דוגמאות

### Code reviewer

```yaml
---
name: code-reviewer
description: Expert code review. Use immediately after significant code changes.
tools: Read, Grep, Glob, Bash
model: inherit
---

Review the recent changes for:
- Code quality
- Security issues
- Performance
- Test coverage

Start by running `git diff` to see changes.
```

### Debugger

```yaml
---
name: debugger
description: Expert debugging. Use when tests fail or errors appear.
tools: Read, Edit, Bash, Grep, Glob
---

Workflow:
1. Reproduce the issue
2. Add logging if needed
3. Identify root cause
4. Fix and verify
```

### Read-only DB

```yaml
---
name: db-inspector
tools: Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./validate-readonly.sh"
---

Query the database to answer questions. Never write.
```

---

## `/agents` command

פותח TUI לניהול subagents:
- רואה את כולם
- יוצר חדש
- עורך קיים
- בדיקה (test invocation)

במקום לערוך ידנית markdown.

---

## Subagents vs Agent Teams

| | Subagents | Agent Teams |
|---|-----------|-------------|
| Context | נפרד לכל אחד | sessions שונות לגמרי |
| Coordination | main thread | teammate messaging |
| Duration | עד שה-task הזה מסתיים | persistent עד shutdown |
| Shared state | רק summary חוזר | shared tasks + channels |
| שימוש | delegation בתוך session | orchestration של agents עובדים יחד |

ל-[agent-teams](https://code.claude.com/docs/en/agent-teams) - קריאה לבד.

---

## המשך → [mcp.md](#/deep/mcp)
