# Agent Loop - לולאת הסוכן

> המודל המנטלי הבסיסי ביותר של Claude Code. אם לא מבינים אותו לעומק, כל השאר נשאר מטושטש.

**מקור:** [how-claude-code-works](https://code.claude.com/docs/en/how-claude-code-works)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>תחשוב על <strong>שף במטבח</strong>. כשאתה מבקש ממנו מנה: קודם הוא קורא את המתכון ובודק מה יש במזווה (<strong>גיבוש קונטקסט</strong>), אחר כך הוא מבשל (<strong>פעולה</strong>), ואז טועם כדי לוודא שיצא טוב (<strong>בדיקה</strong>). אם משהו לא בסדר, הוא מתקן ומנסה שוב.</p>

<p>Claude Code עובד בדיוק ככה. אתה נותן לו משימה, והוא עובר שוב ושוב את שלושת השלבים האלה: <strong>בודק → עושה → מוודא</strong> - עד שהמשימה מסתיימת. <strong>הוא זה שמחליט</strong> מה הצעד הבא על סמך מה שגילה בצעד הקודם. זה מה שהופך אותו ל"סוכן חכם" ולא רק לצ'אט בוט שעונה תשובות.</p>

<p>אתה יכול לעצור אותו באמצע בכל רגע, לכוון אותו בכיוון אחר, או לתת לו עוד מידע. הוא עצמאי - אבל מקשיב לך.</p>
</div>

---

## מה זה agent loop ולמה זה חשוב

Claude Code לא "עורך קוד". Claude Code הוא **harness** - מסגרת תוכנה שעוטפת מודל שפה והופכת אותו לסוכן פעולתי. הלולאה עצמה היא הלב של זה.

כל בקשה שלך עוברת שלושה שלבים שחוזרים עד שהמשימה מסתיימת:

```
User Prompt → [Gather Context] → [Take Action] → [Verify Results] → repeat
                      ↑                                    ↓
                      └────────────── loop ────────────────┘
```

- **Gather context** - קריאת קבצים, חיפושים, בדיקת git status, הבנה.
- **Take action** - עריכה, הרצה, יצירה.
- **Verify results** - הרצת טסטים, קריאה חוזרת של הפלט, בדיקה ויזואלית.

השלבים מתערבבים. קריאה של test.js יכולה להיות "gather" בלולאה אחת ו-"verify" בלולאה הבאה. **Claude מחליט בכל צעד מה הצעד הבא** על בסיס מה שלמד בצעד הקודם.

### איך זה שונה מ-autocomplete

Autocomplete (Copilot, Cursor inline) - רואה את הקובץ הנוכחי ומשלים. אין לולאה, אין החלטות, אין verify.

Claude Code - **בוחר לאיזה קובץ לקרוא, מה לחפש, מה להריץ, מתי לעצור**. זו ההבחנה בין "AI assistant" ל-"agentic system".

---

## שני המרכיבים: Models + Tools

### Models (הנימוק)

המודל הוא החלק החושב. כשכתוב "Claude decides" - זה המודל. זמינים:

| Alias | Full ID | שימוש |
|-------|---------|-------|
| `opus` | `claude-opus-4-8` | משימות מורכבות, ארכיטקטורה, reasoning עמוק - default מומלץ ליום-יום |
| `sonnet` | `claude-sonnet-4-6` | רוב המשימות - מהיר ומאוזן |
| `haiku` | `claude-haiku-4-5-20251001` | משימות מהירות, זולות |
| `fable` | `claude-fable-5` | "Mythos-class" - המשימות הכי קשות וארוכות (long-horizon), במחיר פרימיום |

> **לגבי Fable:** המודל החזק ביותר, מסוג "Mythos". Fable 5 שוחרר ב-9 ביוני 2026 (v2.1.170) ו-Fable 5.1 אחריו (דורש v2.1.255 ומעלה). מתומחר מעל Opus: ‎$10 לכל 1M input ו-‎$50 לכל 1M output, מול ‎$5/$25 של Opus 5 ו-‎$2/$10 של Sonnet 5. לעבודת היומיום Anthropic מגדירה את **Sonnet 5 כברירת המחדל** במנויי Pro ו-Team Standard; Opus 5 לחשיבה מורכבת; Fable רק למשימות long-horizon הקשות באמת. (עודכן 4.9.2026 מול model-config.)

החלפה: `/model` בתוך סשן, או `claude --model <name>` בהפעלה.

### Tools (הפעולה)

בלי tools, Claude יכול רק לייצר טקסט. עם tools - לקרוא קוד, לערוך, להריץ, לחפש ברשת. **כל שימוש בכלי מחזיר מידע שמוזן חזרה ללולאה**.

חמש קטגוריות tools מובנים:

| קטגוריה | מה Claude יכול לעשות | כלים מרכזיים |
|---------|---------------------|---------------|
| **File operations** | קריאה, עריכה, יצירה, שינוי שם | Read, Edit, Write |
| **Search** | חיפוש קבצים ותוכן | Glob, Grep |
| **Execution** | הרצת shell, טסטים, git | Bash |
| **Web** | חיפוש ברשת, הבאת תיעוד | WebSearch, WebFetch |
| **Code intelligence** | שגיאות טיפוס, jump-to-definition | דורש Code Intelligence plugin |

מעבר לאלה: Agent (spawn subagent), AskUserQuestion, TodoWrite, Skill, ועוד. רשימה מלאה: [tools-reference](https://code.claude.com/docs/en/tools-reference).

---

## הלולאה בפועל - דוגמה

בקשה: *"fix the failing tests"*

```
1. Bash("npm test")                     → רואה אילו טסטים נכשלים
2. Read(failing_test.js)                → קורא את הטסט
3. Grep("functionName", "src/")         → מוצא את המימוש
4. Read(src/impl.js)                    → קורא את הקוד
5. Edit(src/impl.js, fix)               → מתקן
6. Bash("npm test")                     → מאמת
7. [כל הטסטים עוברים] → סיום
```

**אם טסט עדיין נכשל אחרי הצעד 6**, Claude חוזר ל-gather עם מידע חדש (הודעת שגיאה) ומנסה גישה אחרת. זה ה-**course correction** של הלולאה.

---

## איך אתה משתלב בלולאה

Claude עובד autonomously - אבל אתה יכול לקטוע בכל רגע:

- **Esc** - עוצר את הפעולה הנוכחית.
- **Esc Esc** - rewind לנקודת checkpoint קודמת (מבטל שינויי קבצים).
- **Type מיד ו-Enter** - מזרים הודעה חדשה, Claude מעדכן גישה.

**רעיון מפתח:** אתה לא "מפעיל" את Claude ומחכה. אתה משתתף בלולאה. תן feedback מוקדם - "לא, זה לא השורש של הבעיה, בדוק את middleware" - וחסוך המון זמן.

---

## Sessions - מה נשמר, מה לא

כל שיחה נשמרת ב-`~/.claude/projects/` כ-JSONL.

- **New session** - context window ריק. אין היסטוריה מסשן קודם.
- **Continue** - `claude --continue` - חוזר לאותו session ID, הודעות חדשות נוספות.
- **Resume** - `claude --resume` - בוחר session ישן ספציפי.
- **Fork** - `claude --continue --fork-session` - יוצר session ID חדש עם היסטוריה זהה. שימושי כש-"רוצים לנסות כיוון אחר בלי לאבד את המקורי".

### Gotcha - same session בשני terminals

אם תריץ `--resume` של אותו session בשני terminals, שניהם כותבים לאותו קובץ. ההודעות יתערבבו. פתרון: `--fork-session` עבור ה-terminal השני.

### Session מול branch

Session קשור ל-**directory**, לא ל-branch. מחליף branch → Claude רואה קבצים חדשים, אבל היסטוריית השיחה נשמרת. לעבודה מקבילה על branches שונים: [git worktrees](https://code.claude.com/docs/en/common-workflows#run-parallel-claude-code-sessions-with-git-worktrees).

---

## הרחבות על גבי הלולאה

הלולאה הבסיסית עם tools מובנים - זה הבסיס. מעליה נבנות ההרחבות:

| הרחבה | מה היא עושה | מתי |
|-------|-------------|-----|
| **[CLAUDE.md](#/deep/memory-system)** | הוראות קבועות שנטענות בכל session | project conventions |
| **[Skills](#/deep/skills)** | playbooks שנטענים on demand | "deploy", "review-pr" |
| **[Hooks](#/deep/hooks)** | שמפעילים shell ב-lifecycle events | format after edit |
| **[MCP](#/deep/mcp)** | שרתים חיצוניים עם tools | Slack, DB, custom APIs |
| **[Subagents](#/deep/subagents)** | context window נפרד למשימה | research heavy |
| **[Plugins](#/deep/plugins)** | אריזה של כל הנ"ל להפצה | distribution |

כולן שכבות על הלולאה - לא שינוי שלה. המודל עדיין מחליט, tools עדיין מריצים, הלולאה עדיין gather→action→verify.

---

## טיפים ברמת מומחה

1. **Delegate, don't dictate.** "Fix the auth bug, look in src/payments" - לא "open auth.ts line 42, change X to Y". אתה נותן קונטקסט; Claude מחליט איך.

2. **Be specific upfront.** ככל שה-prompt הראשון מדויק יותר (שמות קבצים, constraints, expected output), פחות course-correction תצטרך.

3. **Give it something to verify against.** Test cases, screenshots, output specs. הלולאה עובדת טוב יותר כשיש "האם זה נכון?" מדיד.

4. **Plan before implement.** ב-Shift+Tab כניסה ל-Plan mode: Claude חוקר read-only, מייצר plan, אתה מאשר. עבור משימות מורכבות זה שווה את הזמן הנוסף.

5. **Two-phase for big tasks.** Phase 1: "Read src/auth/ and understand how sessions work". Phase 2: "Now add OAuth support based on what you found". ביניהן אתה יכול להתערב.

---

## הרחבה → [context-window.md](#/deep/context-window)

הלולאה רצה בתוך context window. מה נטען, מתי, ובאיזה מחיר - זה הנושא הבא.
