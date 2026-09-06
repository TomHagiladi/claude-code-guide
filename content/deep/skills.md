# Skills - playbooks שנטענים on demand

> Skill = תיקייה עם `SKILL.md` שמרחיבה את יכולות Claude. נטען רק כש-Claude מחליט להשתמש בו (או שאתה מקליד `/name`). בניגוד ל-CLAUDE.md שתמיד בקונטקסט - skill יכול להיות מאות KB של reference material בעלות אפס עד שצריך.

**מקור:** [skills](https://code.claude.com/docs/en/skills) · [agentskills.io](https://agentskills.io) (standard פתוח)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>תחשוב על <strong>ספר מתכונים במטבח</strong>. יש לך מתכונים לאירועים מיוחדים: "איך לאפות עוגה", "איך להכין רוטב", "איך לעשות deploy לאתר". אתה לא קורא את כל הספר מההתחלה כל בוקר - אתה פותח אותו רק כשאתה צריך מתכון ספציפי.</p>

<p><strong>Skills זה אותו רעיון</strong>: חבילות של הוראות שClaude "פותח" רק כשצריך. כתבת פעם אחת "איך לעשות commit נכון בפרויקט הזה" - ועכשיו בכל פעם שאתה כותב <code>/commit</code>, Claude עוקב אחרי ההוראות שלך בדיוק.</p>

<p>היתרון הגדול: זה לא תופס מקום בזיכרון של Claude עד שאתה משתמש בו. אתה יכול לבנות עשרות skills ולא להעמיס עליו.</p>

<p>מתי ליצור skill? כשאתה מדביק את אותן הוראות שוב ושוב בצ'אט. במקום להעתיק - צור skill והפעל עם <code>/שם-ה-skill</code>.</p>
</div>

---

## מתי ליצור skill

השאלות שמסננות:
- אתה מדביק את אותו playbook/checklist לצ'אט שוב ושוב?
- section ב-CLAUDE.md הפך לprocedure רב-שלבי (ולא עובדה)?
- יש ידע מתמחה שנדרש רק לחלק מהעבודה?

אם כן → skill.

---

## המבנה

```
my-skill/
├── SKILL.md              # חובה - entrypoint
├── reference.md          # detailed docs (optional)
├── examples/
│   └── sample.md         # דוגמאות format
└── scripts/
    └── helper.py         # סקריפט ש-Claude יכול להריץ
```

רק `SKILL.md` חובה. שאר הקבצים נטענים on demand - Claude קורא אותם כשהוא צריך (וכותב ב-SKILL.md "for details see reference.md").

---

## Custom commands = Skills

```
.claude/commands/deploy.md     # קובץ legacy
.claude/skills/deploy/SKILL.md  # skill
```

שניהם יוצרים `/deploy`. הקיימים עובדים. Skills הוסיפו: directory לקבצי תמיכה, frontmatter עשיר יותר, invocation אוטומטי על ידי Claude.

אם יש conflict בשם - skill גובר על command.

---

## SKILL.md - frontmatter מלא

```yaml
---
name: my-skill
description: Short description. Claude reads this to decide when to auto-invoke
when_to_use: Additional trigger context
argument-hint: "[issue-number]"
disable-model-invocation: false
user-invocable: true
allowed-tools: Read Grep Bash(git *)
model: sonnet
effort: high
context: fork
agent: Explore
paths:
  - "src/api/**/*.ts"
hooks:
  Stop:
    - hooks:
        - type: command
          command: ./cleanup.sh
shell: bash
---

Skill instructions here...
```

### כל השדות

| Field | Required | הסבר |
|-------|----------|------|
| `name` | לא | אות, מספר, hyphen. ברירת מחדל: שם תיקייה. max 64. |
| `description` | מומלץ | מה ומתי. Claude קורא להחלטה אוטומטית. |
| `when_to_use` | לא | trigger phrases נוספים. מצטבר ל-description. |
| `argument-hint` | לא | hint באוטוקומפליט: `"[filename] [format]"` |
| `disable-model-invocation` | לא | `true` = רק אתה מפעיל. ברירת מחדל `false`. |
| `user-invocable` | לא | `false` = רק Claude. ברירת מחדל `true`. |
| `allowed-tools` | לא | tools מאושרים אוטו כשהskill פעיל |
| `model` | לא | מודל להרצה: `sonnet`, `opus`, `haiku` |
| `effort` | לא | `low`, `medium`, `high`, `max`, `xhigh` (דגמי Opus בלבד) |
| `context` | לא | `fork` = רץ ב-subagent נפרד |
| `agent` | לא | איזה subagent type (עם `context: fork`) |
| `hooks` | לא | hooks ב-lifetime של ה-skill |
| `paths` | לא | glob - auto-load רק כש-Claude עובד על קבצים תואמים |
| `shell` | לא | `bash` (ברירת מחדל) או `powershell` (Windows) |

**1,536 chars** - Hard cap על `description` + `when_to_use` ב-skill listing. front-load את הcase העיקרי.

---

## היררכית מיקום

| Scope | Path | Applies to |
|-------|------|------------|
| Enterprise | managed settings dir | כל המשתמשים בארגון |
| Personal | `~/.claude/skills/<name>/SKILL.md` | כל הפרויקטים |
| Project | `.claude/skills/<name>/SKILL.md` | פרויקט זה בלבד |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | כש-plugin מופעל |

**קדימות:** enterprise > personal > project. Plugin skills ב-namespace נפרד (`plugin-name:skill-name`).

### Live change detection

שינויי skill ב-`~/.claude/skills/`, `.claude/skills/`, או `--add-dir`-directory → אפקטיביים מיד, בלי restart.

**יוצאי דופן:** יצירת top-level `skills/` directory חדש שלא היה בתחילת session → דורש restart.

### Skills מ-`--add-dir`

`--add-dir` נותן גישה לקבצים, לא טעינת קונפיג. **Skills הם יוצא דופן** - `.claude/skills/` ב-`--add-dir` כן נטען. שאר ההגדרות (agents, commands, output styles) - לא.

---

## שני סוגי skill content

### Reference content

Skill שמוסיף ידע:

```yaml
---
name: api-conventions
description: API patterns for this codebase
---

When writing API endpoints:
- Use RESTful naming
- Return consistent error formats
- Include request validation
```

רץ inline, Claude משתמש במידע לצד שיחה.

### Task content

Skill שמבצע פעולה:

```yaml
---
name: deploy
description: Deploy to production
disable-model-invocation: true
---

Deploy:
1. Run tests
2. Build
3. Push to deployment target
```

שלבים ברורים. בדרך כלל `disable-model-invocation: true` - אתה לא רוצה ש-Claude יחליט לדפלוי.

---

## שליטה במי מפעיל

| Frontmatter | You | Claude | נטען ל-context כ... |
|-------------|-----|--------|---------------------|
| (default) | ✅ | ✅ | description תמיד, content ב-invocation |
| `disable-model-invocation: true` | ✅ | ❌ | **לא בקונטקסט** עד ש-you מפעיל |
| `user-invocable: false` | ❌ | ✅ | description תמיד |

**Pro tip:** Skills עם side-effects (commit, deploy, send-message) → תמיד `disable-model-invocation: true`. גם לחסכון בקונטקסט וגם למניעת "Claude חשב שהקוד מוכן ו-deploy".

---

## Content lifecycle

כש-skill מופעל - ה-content רץ ונכנס ל-conversation כ-user message אחד. **נשאר לכל שאר ה-session.** Claude Code **לא קורא את הקובץ שוב** בturns הבאים.

**Implication:** כתוב standing instructions, לא one-time steps. "Always validate X" טוב; "Now do Y then Z" בעייתי - Claude יראה את זה גם בturns מאוחרים.

### שרידות ב-auto-compaction

- עד 5,000 tokens של ההפעלה האחרונה של כל skill - נשמרים אחרי compaction.
- budget כולל: 25,000 tokens לכל ה-skills שהופעלו.
- מ-recent ל-oldest - skills ישנים יכולים ליפול.

### לא משפיע אחרי הפעלה ראשונה?

התוכן כנראה עדיין שם, Claude בוחר כלים/גישות אחרים. פתרונות:
- חיזוק ה-description.
- הוראות פנימיות יותר קטגוריות.
- Hooks לאכיפה דטרמיניסטית.
- הפעלה חוזרת אחרי compaction.

---

## `allowed-tools` - pre-approve

```yaml
allowed-tools: Bash(git add *) Bash(git commit *) Bash(git status)
```

**לא מגביל אילו tools זמינים** - כל ה-tools עדיין קריאים. רק **מוותר על approval prompts** לאלה ברשימה.

**חסימה של tools ב-skill?** → deny rules ב-`permissions.deny`, לא כאן.

---

## `$ARGUMENTS` ומשתנים

```yaml
---
name: fix-issue
---
Fix GitHub issue $ARGUMENTS following standards.
```

`/fix-issue 123` → `$ARGUMENTS = "123"`.

### אינדקס ספציפי

```yaml
Migrate $ARGUMENTS[0] from $1 to $2
```

`/migrate SearchBar React Vue` → `$ARGUMENTS[0]=SearchBar`, `$1=React`, `$2=Vue`.

**Shell quoting:** `/skill "hello world" second` → `$0="hello world"`, `$1="second"`.

### Substitutions זמינים

| Var | תוכן |
|-----|------|
| `$ARGUMENTS` | כל ה-args |
| `$ARGUMENTS[N]` / `$N` | arg ספציפי |
| `${CLAUDE_SESSION_ID}` | session ID |
| `${CLAUDE_SKILL_DIR}` | ה-directory של SKILL.md |

`${CLAUDE_SKILL_DIR}` שימושי במיוחד ל-scripts:

```yaml
!`python ${CLAUDE_SKILL_DIR}/scripts/validate.py`
```

---

## Shell injection - `!` syntax

**רץ לפני ש-Claude רואה את ה-skill.** ה-output מחליף את ה-placeholder.

```yaml
---
name: pr-summary
allowed-tools: Bash(gh *)
---

## PR context
- Diff: !`gh pr diff`
- Comments: !`gh pr view --comments`
- Files: !`gh pr diff --name-only`

## Task
Summarize this pull request...
```

### Block שורות מרובות

```markdown
## Environment
```!
node --version
npm --version
git status --short
```
```

**חשוב:** זה preprocessing. Claude לא מריץ את הפקודות - הן רצות קודם, והוא מקבל את התוצאה בלבד.

### השבתה

`"disableSkillShellExecution": true` ב-settings → הפקודות מוחלפות ב-`[shell command execution disabled by policy]`. שימושי ב-managed settings.

---

## `context: fork` - skill ב-subagent

```yaml
---
name: deep-research
context: fork
agent: Explore
---

Research $ARGUMENTS:
1. Find relevant files
2. Analyze
3. Summarize with file refs
```

- יוצר context window נפרד.
- ה-skill body נהפך ל-task של ה-subagent.
- ה-`agent` field בוחר system prompt (Explore, Plan, general-purpose, או custom).
- חוזר רק summary.

**Gotcha:** `context: fork` ל-skill שהוא reference-only ("use these conventions") → ה-subagent יקבל guidelines בלי task. חזור ללא output משמעותי. השתמש ב-`fork` רק ל-skills עם task ברור.

---

## Extended thinking

הוסף את המילה `ultrathink` בכל מקום ב-skill content → Claude משתמש ב-extended thinking mode. שימושי ל-skills שדורשים reasoning עמוק.

---

## Restrict Claude's skill access

### Disable all skills
```
permissions:
  deny:
    - Skill
```

### Allow/deny specific

```
Skill(commit)           # אישור ספציפי
Skill(review-pr *)      # prefix
Skill(deploy *)         # deny rule
```

---

## Skill descriptions - budget

Claude Code מקצה `SLASH_COMMAND_TOOL_CHAR_BUDGET` ל-skill descriptions - 1% מחלון הקונטקסט, fallback 8,000 chars.

אם יש לך הרבה skills - descriptions מתקצרים. Claude עלול לא למצוא את ה-keyword הנכון.

**הגדלה:**
```bash
SLASH_COMMAND_TOOL_CHAR_BUDGET=15000 claude
```

או תקצר description במקור - 1,536 chars cap per entry.

---

## Bundled skills

Claude Code כולל skills מובנים: `/simplify`, `/batch`, `/debug`, `/loop`, `/claude-api`. רשימה: [commands reference](https://code.claude.com/docs/en/commands).

בניגוד ל-built-in commands שמריצים לוגיקה fixed - bundled skills הם prompt-based. Claude מקבל playbook ומוציא לפועל.

---

## Patterns ברמת מומחה

### 1. Skill + Script לפלט ויזואלי

Skill מזמין script Python → מייצר HTML אינטראקטיבי → `webbrowser.open`. דוגמאות: dependency graphs, test coverage reports, DB schemas.

ה-SKILL.md קצר, ה-script עושה את העבודה:

```yaml
---
name: visualize-codebase
allowed-tools: Bash(python *)
---

Run: python ${CLAUDE_SKILL_DIR}/scripts/visualize.py .
```

### 2. Skill + MCP למשימות deploy

```yaml
---
name: deploy
disable-model-invocation: true
mcpServers:
  - kubernetes
allowed-tools: mcp__kubernetes__*
---

Deploy $1 to $2...
```

MCP servers ב-skill frontmatter? **לא תקף בפרויקט skills** - רק ב-subagents. עבור deploy → skill שמפעיל subagent עם MCP.

### 3. User skills ל-personal workflows

`~/.claude/skills/commit/SKILL.md`:
```yaml
---
name: commit
disable-model-invocation: true
allowed-tools: Bash(git add *) Bash(git commit *) Bash(git status *) Bash(git diff *)
---

Review changes and create commit with message focused on WHY.
Include "Co-Authored-By: Claude <noreply@anthropic.com>"
```

### 4. Path-scoped skill לפרויקט ספציפי

```yaml
---
name: api-style
paths:
  - "src/api/**"
---

API conventions:
- RESTful naming
- Consistent error envelope
```

אוטו-נטען רק כש-Claude נוגע ב-`src/api/**`.

---

## Troubleshooting

**לא מופעל אוטומטית?**
- description עם keywords טבעיים (מה המשתמש יגיד)
- `/skills` - האם רשום?
- Rephrase הבקשה
- הפעל ישירות `/name`

**מופעל יותר מדי?**
- description יותר ספציפי
- `disable-model-invocation: true`

**description נחתך?**
- הגדל `SLASH_COMMAND_TOOL_CHAR_BUDGET`
- קצר במקור - front-load

---

## Skills vs Rules vs CLAUDE.md - מתי מה

| תרחיש | כלי מתאים |
|--------|-----------|
| "תמיד השתמש ב-2-space indent" | CLAUDE.md |
| "API conventions לכל קובץ ב-src/api" | `.claude/rules/` עם `paths:` |
| "Checklist ל-review PR (5 שלבים)" | Skill |
| "Playbook ל-deploy production" | Skill (disable-model-invocation) |
| "Legacy system context" | Skill (user-invocable: false) |
| "Research heavy codebase" | Skill (context: fork) |

---

## המשך → [subagents.md](#/deep/subagents)
