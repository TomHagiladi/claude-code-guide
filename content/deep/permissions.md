# Permissions - בקרת גישה

> מי יכול לעשות מה, מתי, ואיך לאכוף. שכבת ההגנה העיקרית של Claude Code מול pagina bugs, prompt injection, ו-destructive actions.

**מקורות:** [permissions](https://code.claude.com/docs/en/permissions) · [permission-modes](https://code.claude.com/docs/en/permission-modes) · [sandboxing](https://code.claude.com/docs/en/sandboxing)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>תחשוב על <strong>הרשאות באפליקציות בטלפון</strong>. כשאפליקציה רוצה גישה למצלמה - הטלפון שואל אותך. אתה יכול לאשר תמיד, לדחות תמיד, או להחליט בכל פעם.</p>

<p><strong>עם Claude זה בדיוק אותו דבר.</strong> הוא רוצה להריץ פקודה, לערוך קובץ, או להתחבר לרשת - אתה מחליט מה מותר לו לעשות בלי לשאול, ומה חייב אישור בכל פעם.</p>

<p>יש <strong>שישה מצבי עבודה</strong>: מ"שאל על הכל" (הכי בטוח) ועד "עשה הכל בלי לשאול" (הכי מהיר, הכי מסוכן - רק במכונות מבודדות). אתה יכול גם לכתוב כללים מדויקים: "מותר להריץ <code>npm test</code>, אסור למחוק קבצים, שאל לפני git push".</p>

<p>עקרון הזהב: <strong>דחייה תמיד חזקה יותר מאישור</strong>. אם מנהל המערכת שלך חסם משהו - אתה לא יכול לעקוף. זה מה שהופך את המערכת לבטוחה לשימוש בארגונים.</p>
</div>

---

## המודל

Claude Code משתמש במערכת שלוש-רמות:

| סוג tool | דוגמה | Approval? | "Yes, don't ask again" |
|----------|-------|-----------|------------------------|
| Read-only | Read, Grep | ❌ | N/A |
| Bash commands | shell | ✅ | קבוע לפי project + command |
| File modification | Edit, Write | ✅ | עד סוף session |

**Rules evaluated:** `deny → ask → allow`. ה-match הראשון מנצח. deny תמיד גובר.

---

## Permission Modes - דרך הפעולה

```json
{
  "permissions": {
    "defaultMode": "default"
  }
}
```

| Mode | מה קורה |
|------|----------|
| `default` | שואל לפני tool שלא מוכר |
| `acceptEdits` | אוטו לעריכות וfs-commands (mkdir, mv, cp) ב-cwd/additionalDirectories |
| `plan` | Plan mode - רק read, מייצר plan |
| `auto` | Auto-approve עם classifier ברקע שמזהה risks |
| `dontAsk` | דחה אוטומטית tools שלא allowed מראש |
| `bypassPermissions` | דלג על הכל (חריגים: `.git`, `.claude`, `.vscode`, `.idea`, `.husky` - עדיין שואל) |

**Shift+Tab** - מחליף בין modes בזמן session.

### `bypassPermissions` - זהירות!

דלג על הכל, חוץ מהתיקיות המוגנות למעלה. Writes ל-`.claude/commands`, `.claude/agents`, `.claude/skills` **כן עוברים בלי שאלה** (Claude יוצר אותם רוטינית).

**שימוש:** רק ב-containers / VMs מבודדים.

**חסימה ארגונית:**
```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable",
    "disableAutoMode": "disable"
  }
}
```

ב-managed settings - לא ניתן לדרוס.

---

## Rule syntax - `Tool(specifier)`

### כללי - כל שימוש ב-tool

```
Bash          # כל הפקודות
WebFetch      # כל בקשה
Read          # כל קריאה
Edit          # כל עריכה
```

`Bash(*)` === `Bash`.

### Specifier

```
Bash(npm run build)       # בדיוק הזה
Read(./.env)              # קובץ ספציפי
WebFetch(domain:example.com)   # דומיין ספציפי
```

### Wildcards ב-Bash

`*` matches any sequence - כולל רווחים.

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(git commit *)",
      "Bash(git * main)",
      "Bash(* --version)",
      "Bash(* --help *)"
    ],
    "deny": [
      "Bash(git push *)"
    ]
  }
}
```

**Space matters:**
- `Bash(ls *)` → match `ls -la`, **לא** `lsof` (word boundary)
- `Bash(ls*)` → match **שניהם**

**`:*` שווה־ערך ל-`space *`:**
- `Bash(ls:*)` === `Bash(ls *)` (בסוף בלבד)
- Permission dialog משתמש ב-`:*` בלחיצה על "Yes, don't ask".

### Compound commands

Claude מפצל לפי `&&`, `||`, `;`, `|`, `|&`, `&`, newlines. **כל subcommand נבדק בנפרד.**

`Bash(safe-cmd *)` לא נותן ל-`safe-cmd && other-cmd` - `other-cmd` צריך rule משלו.

כשמאשרים compound עם "Yes, don't ask" → עד 5 rules נשמרים, אחד ל-subcommand.

### Process wrappers - מוסרים אוטו

Claude מסיר את אלה לפני matching: `timeout`, `time`, `nice`, `nohup`, `stdbuf`, `xargs` (bare).

`Bash(npm test *)` matches גם `timeout 30 npm test`.

**לא מוסר:** `direnv exec`, `devbox run`, `mise exec`, `npx`, `docker exec` - אלה "tool runners", צריך rule ספציפי:

```
Bash(devbox run npm test)    # ספציפי
```

**`Bash(devbox run *)` יאפשר גם `devbox run rm -rf .`** - זהירות!

---

## Read ו-Edit - gitignore patterns

ארבעה סוגי paths:

| Pattern | משמעות | דוגמה |
|---------|---------|-------|
| `//path` | Absolute מה-root | `Read(//Users/alice/secrets/**)` |
| `~/path` | מ-home | `Read(~/Documents/*.pdf)` |
| `/path` | **יחסית ל-project root** ⚠️ | `Edit(/src/**/*.ts)` |
| `path` or `./path` | יחסית ל-cwd | `Read(*.env)` |

**⚠️ Gotcha:** `/Users/alice/file` **לא** absolute. יחסית ל-project root. לצורך absolute: `//Users/alice/file`.

**Windows:** נרמול ל-POSIX. `C:\Users\alice` → `/c/Users/alice`. Match .env בכל drive: `//**/.env`.

### Glob patterns

- `*` - קבצים ב-directory בודד
- `**` - רקורסיבי

```
Edit(/docs/**)     # edits ב-<project>/docs/ (לא /docs/ absolute)
Read(~/.zshrc)     # קובץ ב-home
Read(src/**)       # read מ-<cwd>/src/
```

### Warning חשוב

**Read/Edit deny rules לא חוסמים Bash!**

`Read(./.env)` ב-deny - חוסם את ה-Read tool, **אבל לא חוסם `cat .env` ב-Bash**.

ל-OS-level enforcement → sandbox (ראה בהמשך).

---

## MCP rules

```
mcp__puppeteer                        # כל tool משרת puppeteer
mcp__puppeteer__*                     # אותו דבר עם wildcard
mcp__puppeteer__puppeteer_navigate    # tool ספציפי
```

---

## Agent rules - subagents

```
Agent(Explore)              # Explore subagent
Agent(Plan)                 # Plan subagent
Agent(my-custom)            # custom
```

```json
{
  "permissions": {
    "deny": ["Agent(Explore)"]
  }
}
```

או CLI: `--disallowedTools "Agent(Explore)"`.

---

## Parameter matching - `Tool(param:value)`

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>עד עכשיו כלל הרשאה הסתכל רק על <strong>שם</strong> ה-tool - "מותר/אסור להפעיל sub-agent", "מותר/אסור Bash". עכשיו אפשר לכתוב כלל שמסתכל גם על <strong>הפרמטרים</strong> שה-tool מבקש - למשל "אם Claude מנסה להפעיל sub-agent על המודל היקר Opus, תשאל אותי קודם".</p>

<p>זה נותן שליטה הרבה יותר עדינה: לא רק "כן/לא לכל ה-sub-agents", אלא "כן ל-sub-agents - אבל שאל אותי לפני הגרסה היקרה".</p>
</div>

מ-**v2.1.178 (15 ביוני 2026)**: כללי `deny` ו-`ask` יכולים להתאים לפי **input parameters** של ה-tool, בעזרת התחביר `Tool(param:value)`.

```json
{
  "permissions": {
    "ask": ["Agent(model:opus)"]
  }
}
```

- `Agent(model:opus)` - match כש-sub-agent מבוקש על ה-Opus tier (המודל היקר).
- `Agent(isolation:*)` - wildcard: match על כל ערך של `isolation`.

כך אפשר policy **ברמת הפרמטר** - למשל `ask` rule לפני sub-agent יקר על Opus, בלי לחסום sub-agents לגמרי.

---

## Extend with hooks

`PreToolUse` hooks רצים לפני permission prompt. Hook output יכול:
- `allow` - skip prompt
- `deny` - block
- `ask` - force prompt
- `defer` - let normal flow continue

**Important:**
- Deny rules **עוקפים** hook decisions. Hook החזיר allow אבל יש deny → denied.
- Ask rules עוקפים hook decisions.
- Hook שחוזר exit 2 → חוסם לפני שה-rules בכלל נבדקים.

Use case: allow all bash except specific dangerous patterns:

```json
{
  "permissions": {
    "allow": ["Bash"]
  },
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{"type": "command", "command": "scripts/check-dangerous.sh"}]
    }]
  }
}
```

---

## `additionalDirectories` - גישה לקבצים

```json
{
  "additionalDirectories": ["../shared-config", "/absolute/path"]
}
```

או CLI: `--add-dir <path>` או ב-session: `/add-dir`.

**מה נטען מ-`--add-dir`:**

| Config | נטען? |
|--------|-------|
| Skills (`.claude/skills/`) | ✅ עם live reload |
| `enabledPlugins`, `extraKnownMarketplaces` | ✅ |
| CLAUDE.md, `.claude/rules/`, CLAUDE.local.md | רק עם `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` |
| Subagents, commands, output styles, hooks | ❌ |

לשיתוף שאר הקונפיג: user-level (`~/.claude/`), plugins, או להריץ מה-directory המרכזי.

---

## Sandboxing - OS-level enforcement

Permissions = Claude-level. Sandbox = OS-level.

- **Permissions** - שולט ב-tools שClaude משתמש. על כל ה-tools (Bash, Read, Edit, WebFetch, MCP).
- **Sandbox** - OS מגביל את ה-Bash tool's filesystem ו-network. רק ל-Bash וילדים שלו.

Defense-in-depth: שניהם ביחד.

- Deny rules מונעים מ-Claude לנסות.
- Sandbox מונע אם prompt injection עקף את Claude.
- **Filesystem restrictions בsandbox משתמש ב-Read/Edit deny rules** (לא config נפרד).
- **Network restrictions** = WebFetch permission rules + sandbox's `allowedDomains`.

כש-`autoAllowBashIfSandboxed: true` (default): Bash בתוך sandbox רץ בלי prompt גם אם `ask: Bash(*)`. ה-sandbox boundary מחליף את ה-prompt.

---

## Managed settings - ארגוני

**שדות managed-only** (לא עובדים ב-user/project):

| Setting | תפקיד |
|---------|-------|
| `allowManagedPermissionRulesOnly` | רק managed rules, user/project מתעלמים |
| `allowManagedMcpServersOnly` | רק allowed MCP servers מ-managed |
| `allowManagedHooksOnly` | רק managed hooks + SDK hooks |
| `allowedChannelPlugins` | allowlist ל-channel plugins |
| `channelsEnabled` | הפעלת channels (team/enterprise) |
| `forceRemoteSettingsRefresh` | block startup אם fetch נכשל |
| `pluginTrustMessage` | הודעה מותאמת לאישור plugins |
| `blockedMarketplaces` | blocklist למקורות marketplace |
| `strictKnownMarketplaces` | controls אילו marketplaces מותר |
| `sandbox.filesystem.allowManagedReadPathsOnly` | רק managed allowRead paths |
| `sandbox.network.allowManagedDomainsOnly` | רק managed allowed domains |

`disableBypassPermissionsMode` עובד מכל scope - משתמש יכול גם לנעול את עצמו.

---

## Settings precedence

1. **Managed settings** - לא ניתן לדרוס (גם לא עם CLI args)
2. **CLI arguments** - לsession זה בלבד
3. **Local project** (`.claude/settings.local.json`)
4. **Shared project** (`.claude/settings.json`)
5. **User** (`~/.claude/settings.json`)

**Deny בכל רמה** = blocked, גם אם רמה אחרת allow.

User allow + project deny → denied (project מנצח).

---

## Auto mode classifier

Auto mode משתמש ב-classifier כדי להחליט אם action בטוח. Default: סומך רק על cwd + git remotes של ה-repo. Push ל-org company → **חסום** כ-potential exfiltration.

### `autoMode.classifyAllShell` - סווג כל פקודת shell

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>ה-classifier הוא ה"בודק" שמחליט אם פקודה בטוחה. כברירת מחדל הוא לא טורח לבדוק כל פקודה - הוא בודק רק פקודות שנראות חשודות, כדי לחסוך זמן. ההגדרה הזאת אומרת לו: <strong>תבדוק את כולן, בלי יוצא מן הכלל</strong>. קצת יותר איטי, אבל אף פקודה לא חומקת מהבדיקה.</p>
</div>

מ-**v2.1.193 (25 ביוני 2026)**:

```json
{
  "autoMode": {
    "classifyAllShell": true
  }
}
```

כברירת מחדל ה-classifier בודק רק פקודות שנראות חשודות. עם `classifyAllShell: true` - **כל** פקודת Bash ו-PowerShell עוברת דרך ה-safety classifier, גם פקודות שנראות תמימות לחלוטין.

### `autoMode.environment` - ספר ל-classifier מה בטוח

```json
{
  "autoMode": {
    "environment": [
      "Source control: github.example.com/acme-corp",
      "Trusted buckets: s3://acme-builds, gs://acme-datasets",
      "Trusted internal domains: *.corp.example.com",
      "Key services: Jenkins at ci.example.com"
    ]
  }
}
```

פרוזה, לא regex. ה-classifier קורא כהוראות טבעיות.

### `autoMode.soft_deny` / `autoMode.allow`

```json
{
  "autoMode": {
    "allow": ["Deploying to staging is allowed: isolated, resets nightly"],
    "soft_deny": ["Never run DB migrations outside the migrations CLI"]
  }
}
```

**⚠️ Danger:** `allow` או `soft_deny` שאתה מגדיר **מחליפים את כל ה-defaults.** חובה:

```bash
claude auto-mode defaults     # מדפיס defaults
```

העתק, ערוך, הדבק. לעולם אל תתחיל מרשימה ריקה.

### בדיקה

```bash
claude auto-mode defaults      # defaults
claude auto-mode config        # effective rules
claude auto-mode critique      # AI review של ה-rules שלך
```

### `PermissionDenied` hook

גישה פרוגרמטית ל-denials ב-auto mode.

### `/permissions` → Recently denied

`r` על action → mark for retry. Claude מקבל הודעה שמותר לנסות שוב.

### סיבות הדחייה - עכשיו גלויות

מ-**v2.1.193 (25 ביוני 2026)**, כש-Auto mode חוסם action - **הסיבה לדחייה מוצגת במפורש** בשלושה מקומות:

- ב-transcript (תיעוד השיחה)
- ב-denial toast (ההודעה הקופצת)
- וב-`/permissions`

קודם ראית שמשהו נחסם בלי לדעת למה. עכשיו ה-classifier מסביר את ההחלטה - קל יותר להבין מה קרה ולתקן את ה-rules בהתאם.

---

## דוגמאות

### Minimal safe setup (project)

```json
{
  "permissions": {
    "defaultMode": "default",
    "allow": [
      "Bash(npm test)",
      "Bash(npm run *)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Read(src/**)",
      "Read(tests/**)",
      "Edit(src/**)",
      "Edit(tests/**)"
    ],
    "deny": [
      "Read(**/*.env)",
      "Read(**/secrets/**)",
      "Bash(rm -rf *)",
      "Bash(git push *)"
    ]
  }
}
```

### Relaxed dev setup

```json
{
  "permissions": {
    "defaultMode": "acceptEdits",
    "allow": ["Bash"],
    "deny": [
      "Bash(rm -rf /*)",
      "Bash(:(){:|:&};:)",
      "Read(**/*.env*)",
      "Bash(sudo *)"
    ]
  }
}
```

### Enterprise lockdown (managed)

```json
{
  "allowManagedPermissionRulesOnly": true,
  "allowManagedMcpServersOnly": true,
  "allowManagedHooksOnly": true,
  "disableBypassPermissionsMode": "disable",
  "disableAutoMode": "disable",
  "permissions": {
    "defaultMode": "default",
    "allow": [
      "Read",
      "Grep",
      "Bash(npm run *)",
      "Edit(/src/**)"
    ],
    "deny": [
      "Bash(curl *)",
      "Bash(wget *)",
      "Read(//etc/**)",
      "Edit(//**)"
    ]
  }
}
```

---

## Best practices

1. **התחל ב-default mode.** החלף ל-`acceptEdits` רק כש-trust גבוה.
2. **לעולם אל תתחיל `bypassPermissions` ב-codebase של production.** רק VMs מבודדים.
3. **Deny rules תמיד על secrets.** `Read(**/*.env*)`, `Read(**/secrets/**)`.
4. **Allow specific Bash, not `Bash`.** רשימה של allowed prefix - `Bash(npm *)`, `Bash(git status)`, וכו'. `Bash` פתוח = הכל.
5. **Sandbox ל-defense-in-depth.** אם אתה עובד עם קוד לא מוכר - הפעל.
6. **Managed settings לארגונים.** user/project יכולים לטפס; managed לא.
7. **Review `/permissions` תקופתית.** להבין מה הצטבר.
8. **Auto mode עם environment רחב.** ספר ל-classifier את ה-infrastructure שלך - פחות prompts מיותרים.

---

## חזור ל-[Foundations](#/deep/agent-loop) · המשך ל-[03_sdk/](#/deep/sdk-overview)
