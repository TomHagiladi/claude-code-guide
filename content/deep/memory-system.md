# Memory System - מערכת הזיכרון

> Claude Code פותח כל session עם context window ריק. שתי מערכות זיכרון משלימות גוברות על זה: CLAUDE.md (אתה כותב) ו-Auto Memory (Claude כותב).

**מקור:** [memory](https://code.claude.com/docs/en/memory)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>תחשוב על <strong>עובד חדש שמגיע בכל יום לעבודה כאילו זה היום הראשון שלו</strong>. הוא לא זוכר מה עשה אתמול. יש לך שתי דרכים לעזור לו:</p>

<p><strong>הדרך הראשונה - אתה כותב לו הוראות.</strong> יוצר קובץ שנקרא <code>CLAUDE.md</code> עם דברים כמו "אני כותב בפייתון, תמיד תשתמש ב-2 רווחים להזחה, לבדיקות רוץ <code>npm test</code>". הוא קורא את הקובץ הזה בתחילת כל יום עבודה.</p>

<p><strong>הדרך השנייה - הוא כותב לעצמו פתקים.</strong> תוך כדי שהוא עובד, Claude מזהה דברים ששווה לזכור ("הפרויקט דורש Redis", "לחצי יום תיקנתי שגיאה דומה") - ושומר לעצמו. בפעם הבאה הוא יקרא את הפתקים ויחסוך לעצמו את הבלבול.</p>

<p>הקובץ הזה מסביר איך שתי המערכות עובדות, איפה לכתוב הוראות, ואיך לעצב אותן כדי שClaude באמת יעקוב אחריהן.</p>
</div>

---

## שתי המערכות - ההבדל המהותי

|                | CLAUDE.md | Auto Memory |
|----------------|-----------|-------------|
| מי כותב | אתה | Claude עצמו |
| מה מכיל | הוראות, standards, architecture | למידות, דפוסים, preferences שגילה |
| Scope | Project / user / org | Per git repo |
| נטען ב-session | במלואו | 200 שורות ראשונות / 25KB |
| שימוש | "תמיד תעשה X" | "Claude גילה שהבדיקות דורשות Redis" |

שתיהן **context, לא enforcement**. Claude קורא ומנסה לעקוב - אין כלי שכופה. ככל שההוראות יותר ספציפיות ומובנות → עקביות גבוהה יותר.

---

## CLAUDE.md - היררכיה של מיקומים

| Scope | מיקום | מתי להשתמש |
|-------|-------|-------------|
| **Managed policy** | `/Library/Application Support/ClaudeCode/CLAUDE.md` (macOS)<br>`/etc/claude-code/CLAUDE.md` (Linux/WSL)<br>`C:\Program Files\ClaudeCode\CLAUDE.md` (Windows) | ארגון - לא ניתן לדרוס |
| **Project** | `./CLAUDE.md` או `./.claude/CLAUDE.md` | משותף בצוות דרך git |
| **User** | `~/.claude/CLAUDE.md` | preferences אישיות לכל הפרויקטים |
| **Local** | `./CLAUDE.local.md` | אישי לפרויקט, ב-.gitignore |

### איך הם נטענים (סדר חשוב!)

Claude Code **מטפס בעץ התיקיות** מה-cwd כלפי מעלה, ומחפש CLAUDE.md ו-CLAUDE.local.md בכל תיקייה. **כולם משורשרים, לא נדרסים**. בתוך תיקייה, CLAUDE.local.md נטען **אחרי** CLAUDE.md (כלומר גובר אם יש סתירה).

דוגמה - אם אתה ב-`/foo/bar/baz/`:
```
/foo/CLAUDE.md          ← loaded
/foo/bar/CLAUDE.md       ← loaded, concatenated
/foo/bar/baz/CLAUDE.md   ← loaded, concatenated
/foo/bar/baz/CLAUDE.local.md ← loaded last (highest priority)
```

קבצים ב-**subdirectories** של cwd נטענים **lazily** - רק כש-Claude קורא קובץ באותה תיקייה.

### Imports - `@path/to/file`

```markdown
See @README for project overview.
Git rules: @docs/git-workflow.md
Personal: @~/.claude/my-prefs.md
```

- Relative paths - יחסית לקובץ שמכיל את ה-import, לא ל-cwd.
- Absolute - מותר.
- Recursive - מותר, max depth 5.
- בפעם הראשונה שיש external import - Claude Code שואל אישור. סירוב → לא ישאל שוב.

### AGENTS.md

Claude Code קורא רק CLAUDE.md. אם הפרויקט משתמש ב-AGENTS.md (standard של Cursor/אחרים):

```markdown
# CLAUDE.md
@AGENTS.md

## Claude-specific additions
Use plan mode for changes under src/billing/
```

### `claudeMdExcludes` - דילוג על קבצים

במונורפוז גדול, CLAUDE.md של צוותים אחרים יכול להטעין רעש. ב-`.claude/settings.local.json`:

```json
{
  "claudeMdExcludes": [
    "**/monorepo/CLAUDE.md",
    "/home/user/monorepo/other-team/.claude/rules/**"
  ]
}
```

Managed policy CLAUDE.md **לא ניתן להוציא** מהרשימה - ארגון תמיד גובר.

### Block-level HTML comments

```markdown
<!-- maintainer note: update this when schema changes -->
```

מוסרים מה-context. שימושי להערות ל-human בלי לבזבז tokens.

---

## כתיבת CLAUDE.md יעיל

### גודל

- **Target: מתחת 200 שורות.** גדול מדי → Claude פחות דבק בו.
- גדל? → פצל ל-`.claude/rules/` או imports.
- מודדים ב-tokens, לא בשורות. טבלת הקונטקסט ב-[context-window.md](#/deep/context-window) מראה.

### ספציפיות

| ❌ עמום | ✅ ספציפי |
|--------|----------|
| "Format code properly" | "Use 2-space indentation" |
| "Test your changes" | "Run `npm test` before committing" |
| "Keep files organized" | "API handlers live in `src/api/handlers/`" |

### עקביות

סתירות בין CLAUDE.md שונים (project root vs subdirectory) → Claude בוחר שרירותית. בדוק תקופתית.

### `/init` ליצירה ראשונית

Claude סורק את הקוד ומייצר CLAUDE.md התחלתי. אם כבר קיים - מציע שיפורים במקום לדרוס.

`CLAUDE_CODE_NEW_INIT=1` → flow אינטראקטיבי: שואל אילו artifacts (CLAUDE.md, skills, hooks), מפעיל subagent לחקירה, מציג proposal לפני כתיבה.

---

## `.claude/rules/` - פירוק מודולרי

במקום CLAUDE.md ענק:

```
your-project/
├── .claude/
│   ├── CLAUDE.md           # הוראות בסיס
│   └── rules/
│       ├── code-style.md
│       ├── testing.md
│       ├── api-design.md
│       └── security.md
```

`.md` מתגלים **רקורסיבית**. תתי-תיקיות כמו `frontend/` ו-`backend/` - עובדות.

### Path-specific rules - הפיצ'ר המוצלח ביותר

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API Rules
- All endpoints must include input validation
- Use standard error response format
```

ה-rule **לא נטען בתחילת session**. נטען רק כש-Claude קורא קובץ שמתאים ל-pattern. חיסכון אדיר בקונטקסט.

Patterns:
| Pattern | Matches |
|---------|---------|
| `**/*.ts` | כל TypeScript |
| `src/**/*` | הכל תחת src/ |
| `*.md` | root markdown files |
| `src/**/*.{ts,tsx}` | brace expansion |

### User-level rules

`~/.claude/rules/*.md` - נטענים בכל פרויקט. project rules גוברים אם יש התנגשות.

### Symlinks

`.claude/rules/` תומך symlinks - קישור rule משותף למספר פרויקטים:

```bash
ln -s ~/shared-rules/security.md .claude/rules/security.md
```

---

## Auto Memory - Claude כותב לעצמו

### איפה

```
~/.claude/projects/<project>/memory/
├── MEMORY.md          # אינדקס, 200 שורות/25KB נטענות
├── debugging.md       # detailed notes
├── api-conventions.md
└── ...
```

`<project>` נגזר מה-git repo - כל worktrees של אותו repo חולקים auto memory אחד. מחוץ ל-git - ה-project root משמש.

**Machine-local**. לא משותף בין מכונות או cloud environments.

### איך זה עובד בפועל

- Claude מחליט מה שווה לשמור על בסיס "האם יהיה שימושי בשיחה עתידית".
- `MEMORY.md` - אינדקס קצר.
- קבצי נושא (`debugging.md` וכו') - נטענים on demand עם Read.
- רואים "Writing memory" או "Recalled memory" ב-UI כשזה קורה.

### שליטה

```json
// settings.json
{
  "autoMemoryEnabled": false
}
```

או env: `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`

**שינוי מיקום:**
```json
{
  "autoMemoryDirectory": "~/my-custom-memory-dir"
}
```

לא מקובל ב-project settings (מניעת hijack דרך repo משותף).

### עריכה ידנית

`/memory` - מציג הכל שנטען, מאפשר פתיחה בעורך. כל הקבצים הם markdown רגיל - לערוך, למחוק חופשי.

ביקש מ-Claude "תזכור ש..." → שמור ל-auto memory.
"Add this to CLAUDE.md" → שמור ל-CLAUDE.md.

---

## Subagents עם memory משלהם

Subagent יכול להחזיק auto memory עצמאי:

```yaml
---
name: code-reviewer
memory: user   # או project / local
---
```

| Scope | Location |
|-------|----------|
| `user` | `~/.claude/agent-memory/<agent-name>/` |
| `project` | `.claude/agent-memory/<agent-name>/` |
| `local` | `.claude/agent-memory-local/<agent-name>/` |

ה-subagent יקבל הוראות אוטומטיות לקרוא/לכתוב ל-directory הזה. שימושי לבנות ידע מתמחה לאורך זמן. ראה [subagents.md](#/deep/subagents#persistent-memory).

---

## Debugging - למה Claude לא עוקב אחרי CLAUDE.md

1. `/memory` - בדוק שהקובץ אכן נטען.
2. הבדוק שהמיקום נכון (project root או `.claude/`).
3. ספציפיות - עמום = לא נעקב אחריו.
4. סתירות - חפש rules מנוגדים.
5. `InstructionsLoaded` hook - debugging למה rule נטען/לא נטען.

### "Lost after /compact"

- **Project CLAUDE.md ב-root → שורד** (reinjected מ-disk).
- **Nested CLAUDE.md בתת-תיקייה → לא reinjected** אוטומטית. נטען שוב רק כש-Claude קורא קובץ באותה תיקייה.
- **הוראות שאמרת רק בצ'אט → נעלמו** אחרי compaction. העבר ל-CLAUDE.md.

### `--append-system-prompt`

הוראות שחייבות להיות ב-system prompt (לא ב-user message):

```bash
claude --append-system-prompt "Always respond in Hebrew"
```

צריך להעביר בכל invocation - מתאים ל-scripts.

---

## המלצות ברמת מומחה

1. **התחל מ-`/init`.** טוב להתחלה, ואז refine.
2. **Project CLAUDE.md < 200 שורות.** עודף → rules.
3. **Rules עם `paths`** - כל מה שרלוונטי רק לחלק מהקוד.
4. **User CLAUDE.md** - העדפות אישיות (naming, comments style).
5. **Managed policy** - רק לארגונים שצריכים enforcement.
6. **CLAUDE.local.md** - secrets של dev env, lab URLs, test accounts.
7. **Auto memory** - תן ל-Claude לעבוד. עבור `/memory` מדי כמה שבועות לראות מה נשמר.
8. **בקש מ-Claude לעדכן memory בסוף עבודה מורכבת** - "save what you learned about this architecture".

---

## הרחבה → [session-management.md](#/deep/sessions)
