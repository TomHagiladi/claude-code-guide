# Context Window - חלון הקונטקסט

> המשאב המוגבל ביותר בכל session. מי ששולט בקונטקסט שולט בעלות, באיכות התשובות, ובאורך הסשן.

**מקור:** [context-window](https://code.claude.com/docs/en/context-window)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>תחשוב על <strong>שולחן עבודה</strong>. על השולחן יש לClaude מקום מוגבל לערום עליו ניירות: ההוראות הקבועות שלך, קבצים שהוא קורא, תוצאות של פקודות, והשיחה שלכם. כשהשולחן מתמלא - אי אפשר להוסיף עוד בלי להוריד משהו.</p>

<p>ככל שהשולחן יותר מסודר ופחות עמוס, כך Claude עובד טוב יותר, מהיר יותר, וזול יותר. אם אתה מבקש ממנו לתקן באג קטן - עדיף שיקרא רק את הקובץ הרלוונטי, לא 20 קבצים. אם יש לך הוראות שחשובות תמיד, כדאי לשים אותן במקום קבוע במקום להגיד שוב ושוב בצ'אט.</p>

<p>הקובץ הזה מסביר מה נכנס לשולחן, בכמה מקום זה תופס, ואיך לשמור עליו מסודר כדי שClaude יעבוד בצורה הטובה ביותר.</p>
</div>

---

## המספר הגדול: 200K tokens

ברירת המחדל היא 200K. Sonnet 5 רץ על 1,000,000 tokens בלי הגדרה מיוחדת; Fable 5.1, Fable 5, Opus 4.6 ומעלה ו-Sonnet 4.6 תומכים ב-1M דרך בחירת גרסת `[1m]` (הזמינות תלויה במנוי). (עודכן 4.9.2026.)

המפתח: **גם אם יש לך 200K, יעיל הרבה יותר לא להגיע לשם**. ככל שיותר מלא - יותר יקר, יותר איטי, יותר רעש שמפריע ל-reasoning.

---

## מה נטען אוטומטית בתחילת session

לפני שאמרת מילה, הקונטקסט כבר מלא חלקית. ההערכות בטוקנים (משתנות):

| רכיב | ~Tokens | מה זה |
|------|---------|-------|
| **System prompt** | 4,200 | הוראות ליבה של Claude Code. לא רואה, לא משנה. |
| **Auto memory (MEMORY.md)** | ~700 | 200 שורות ראשונות או 25KB, הקטן מבין השניים |
| **Environment info** | 280 | cwd, platform, shell, OS, git status |
| **MCP tools (deferred)** | ~120 | רק שמות. הסכמה המלאה נטענת on demand |
| **Skill descriptions** | ~450 | שורה לכל skill שאפשר להפעיל אוטו |
| **~/.claude/CLAUDE.md** | 320 | ההוראות הגלובליות שלך |
| **Project CLAUDE.md** | 1,800 | הוראות הפרויקט - החלק שהכי חשוב לנהל |

**סה"כ startup ≈ 8K tokens** לפני שהקלדת משהו.

### חשוב: MCP tools הם deferred

שרתי MCP יכולים להוסיף עשרות או מאות tools. אם כל הסכמות שלהם היו נטענות - היה קל להגיע ל-20-50K tokens רק מזה.

לכן: **רק שמות של tools נטענים**. הסכמה המלאה של tool ספציפי נטענת רק כש-Claude רוצה להשתמש בו (דרך `ToolSearch`).

שליטה:
- `ENABLE_TOOL_SEARCH=auto` - טוען הכל אם זה פחות מ-10% מחלון הקונטקסט.
- `ENABLE_TOOL_SEARCH=false` - טוען הכל תמיד (ישן, יקר, לא מומלץ).
- default - deferred.

---

## מה מצטבר במהלך session

כל פעולה מוסיפה tokens:

| סוג פעולה | Token cost |
|-----------|-----------|
| Read file | גודל הקובץ |
| Grep/Glob output | גודל התוצאה |
| Bash command output | כל הפלט (stdout + stderr) |
| Edit | ה-diff + קצת מסביב |
| WebFetch | הפלט המסוכם של הדף |
| Subagent return | רק ה-summary (זו נקודת זכות עצומה) |
| Hook `additionalContext` | מה ש-hook הוסיף |
| Path-scoped rule loaded | תוכן ה-rule |

**File reads הם הצרכנים הגדולים.** בקשה עמומה כמו "fix the bug" יכולה לגרום ל-Claude לקרוא 10-20 קבצים לפני שהוא מבין איפה הבעיה. בקשה ספציפית - "fix the bug in auth.ts" - מקצרת דרמטית.

---

## פקודת `/context`

הפקודה החשובה ביותר לניהול קונטקסט. מראה:

```
System prompt      4.2K   ━━━━━
Auto memory        680    ━
CLAUDE.md          2.1K   ━━━
MCP tools          120    ▪
Skills             450    ━
Conversation       47.3K  ━━━━━━━━━━━━━━━━━━━━━
Available          145K   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

רוץ את זה לפני שאתה מבקש משהו גדול. אם אתה ב-70%+ - שקול `/compact` או `--fork-session`.

פקודה משלימה: `/mcp` - מראה כמה קונטקסט כל שרת MCP תופס.

---

## Auto-compaction - מה קורה כשמתמלא

כשהקונטקסט מתקרב לגבול, Claude Code מפעיל compaction אוטומטית:

1. **מנקה tool outputs ישנים** קודם. זו הדרך הכי זולה.
2. אם זה לא מספיק - **מסכם את השיחה**.

מה ששורד compaction:

| נשמר | לא נשמר |
|------|---------|
| Project CLAUDE.md (reinjected from disk) | הוראות שאמרת רק בצ'אט |
| Auto memory (reloaded) | tool outputs ישנים |
| Skills שהופעלו (5K tokens per, budget 25K) | Skill descriptions (nested reload) |
| הבקשות העיקריות שלך + קוד מרכזי | פרטי ביניים |
| System prompt | nested CLAUDE.md (reload on next read) |

**שליטה ב-compaction:**
- `/compact` ידני.
- `/compact focus on the API changes` - שמירת פוקוס.
- הוספת section "Compact Instructions" ב-CLAUDE.md כדי להדריך מה חשוב.

### Thrashing error

אם קובץ בודד כל כך גדול שאחרי compaction הוא ממלא מיד מחדש - Claude Code מפסיק לנסות ומציג שגיאה. פתרונות: לקרוא רק חלק מהקובץ, להשתמש בtext search במקום read, להפעיל subagent.

---

## אסטרטגיות חיסכון בקונטקסט

### 1. Skills עם `disable-model-invocation: true`

Skill רגיל - ה-description שלו (שורה) נטען בתחילת session. ה-content רק כשמפעילים אותו.

Skill עם `disable-model-invocation: true` - **אפילו ה-description לא נטען**. עולה אפס עד שאתה מקליד `/name`.

מתי להשתמש: פעולות שאתה תמיד מפעיל ידנית - `/commit`, `/deploy`, `/send-slack`.

### 2. Subagents - קונטקסט נפרד

Subagent מקבל חלון קונטקסט עצמאי. הוא קורא 6,000 tokens של קבצים - זה לא נכנס אליך. כשהוא מסיים, רק ה-summary (~400 tokens) חוזר.

שווה זהב למחקר מעמיק. דוגמה: "Research session timeout handling in this codebase" - דלגו ל-subagent.

### 3. Path-scoped rules

במקום הוראה ב-CLAUDE.md שתמיד תופסת מקום, שים ב-`.claude/rules/api.md` עם frontmatter:

```yaml
---
paths:
  - "src/api/**/*.ts"
---
```

הכלל נטען רק כש-Claude נוגע בקבצים שמתאימים ל-pattern. חיסכון משמעותי במיוחד במונורפוז.

### 4. קריאה ממוקדת של קבצים

במקום `Read(huge_file.ts)` - השתמש ב-`offset` ו-`limit` לקריאה של טווח שורות, או ב-Grep למציאת קוד ספציפי.

### 5. `/clear` במקום `/compact` כשאפשר

`/clear` מתחיל session נקי (שומר auto memory אבל מוחק שיחה). טוב למעבר בין משימות לא קשורות.

---

## מה כל feature עולה - טבלת סיכום

| Feature | עלות startup | עלות use |
|---------|-------------|---------|
| CLAUDE.md | נטען תמיד במלואו | - |
| Rules (no `paths`) | נטענים תמיד | - |
| Rules (with `paths`) | 0 | נטען כש-match |
| Skill (default) | ~100 tokens description | ~500-5000 לפי גודל |
| Skill (`disable-model-invocation`) | 0 | כל ה-skill |
| MCP tool | שם בלבד | הסכמה המלאה כש-tool search טוען |
| Subagent | 0 | רק ה-summary בסוף |
| Hook (command) | 0 | `additionalContext` אם קיים |

**Rule of thumb:** כל מה שצריך "תמיד" - CLAUDE.md. כל מה שצריך "לפעמים" - Skill או Rule. כל מה שדורש עבודה כבדה - Subagent.

---

## Bang mode - `!command`

בתוך צ'אט, הקלד `!git status` - הפקודה תרוץ, גם הפקודה וגם הפלט נכנסים לקונטקסט כחלק מההודעה שלך. שימושי: לא Claude מריץ, אלא אתה - ואז Claude רואה את התוצאה.

שונה מ-Claude שמריץ Bash בעצמו: אתה בוחר בדיוק מה להכניס.

---

## הרחבה → [session-management.md](#/deep/sessions)

אם זה חלון אחד - איך עוברים בין חלונות? continue, resume, fork, checkpoints.
