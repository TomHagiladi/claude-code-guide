# Session Management - ניהול סשנים

> מה זה session, איך הוא נשמר, איך חוזרים אליו, ואיך נמנעים מאובדן עבודה.

**מקורות:** [how-claude-code-works#sessions](https://code.claude.com/docs/en/how-claude-code-works), [checkpointing](https://code.claude.com/docs/en/checkpointing)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>כל <strong>שיחה</strong> שלך עם Claude נשמרת אוטומטית במחשב שלך - כמו צ'אטים בוואטסאפ. סגרת את הטרמינל בטעות? לא נורא, אתה יכול לפתוח מחדש ולהמשיך בדיוק מאיפה שעצרת.</p>

<p>יש לך <strong>שלוש אופציות לחזור לשיחה קודמת</strong>: להמשיך את האחרונה, לבחור ספציפית מרשימה, או "לשכפל" שיחה קיימת כדי לנסות כיוון אחר בלי לפגוע במקור.</p>

<p>בונוס - לפני כל שינוי קובץ, Claude שומר <strong>גיבוי אוטומטי</strong>. עשה בלגן? תלחץ Esc פעמיים והוא יחזור אחורה. זה רק לשינויי קבצים - דברים כמו "שלחתי מייל" או "עשיתי push" לא הפיכים, ולכן הוא שואל לפני.</p>

<p>הקובץ הזה מסביר איך לנווט בין שיחות, איך לעבוד על כמה משימות במקביל בלי להתבלבל, ומתי להשתמש בכל אופציה.</p>
</div>

---

## מה זה session

Session = שיחה אחת מלאה עם Claude Code, מהרגע שהפעלת `claude` ועד שסגרת.

- **נשמר אוטומטית** ב-`~/.claude/projects/<project-id>/<session-id>.jsonl`
- **Plaintext** - אפשר לקרוא ב-text editor. כל הודעה, כל tool use, כל response = שורה ב-JSONL.
- **Per-directory** - sessions של פרויקט אחד לא נראים בפרויקט אחר.
- **Independent** - session חדש מתחיל עם context window ריק. אין המשכיות אוטומטית.

---

## Continue / Resume / Fork

### `claude --continue`
חוזר ל-**session האחרון** ב-cwd. ממשיך מאיפה שעצרת, אותו session ID. הודעות חדשות נוספות להיסטוריה.

### `claude --resume`
מציג רשימת sessions בפרויקט, בוחר ספציפי. שימושי כשיש לך כמה קווים מקבילים.

### `claude --continue --fork-session`
**יוצר session ID חדש** עם ההיסטוריה של הקודם. המקור **לא משתנה**. שימושי: "רוצה לנסות גישה אחרת מנקודה מסוימת בלי לאבד את המקורית".

### Session-scoped permissions - לא שורדות

אישרת ל-Claude להריץ `npm test` - זה היה "for this session". כשחוזרים (continue/resume/fork) - תצטרך לאשר שוב. שמירה קבועה: `.claude/settings.json` → `permissions.allow`.

---

## Gotcha: אותו session בשני terminals

```bash
# Terminal 1
claude --resume abc123

# Terminal 2
claude --resume abc123   # גם כן!
```

שניהם כותבים ל-**אותו קובץ JSONL**. הודעות מתערבבות. שום דבר לא נשבר אבל השיחה נהיית ג'יבריש.

**פתרון:** `--fork-session` ב-terminal השני - כל אחד עם session ID משלו, אותה היסטוריה התחלתית.

---

## Branches ו-sessions

Session קשור ל-**directory**, לא ל-git branch.

- מחליף branch בתוך session → Claude רואה את הקבצים החדשים, אבל היסטוריית השיחה נשמרת.
- אותו directory, branch אחר - אותו session.

**עבודה מקבילה אמיתית על branches שונים:** [git worktrees](https://code.claude.com/docs/en/common-workflows#run-parallel-claude-code-sessions-with-git-worktrees) - יוצרים directory נפרד ל-branch, כל אחד עם sessions משלו.

```bash
git worktree add ../feature-x feature-x
cd ../feature-x
claude    # session חדש, directory חדש, נפרד לחלוטין
```

---

## Checkpoints - Undo אוטומטי

**כל עריכת קובץ הפיכה.** לפני שClaude עורך, הוא **snapshot-מצלם** את התוכן הנוכחי.

- `Esc Esc` (double escape) - rewind למצב קודם.
- או בקשה: "undo the last change".

### מה checkpoints כן

- שינויי קבצים
- תוכן לפני Edit/Write
- מצב פנימי של session

### מה checkpoints לא

- **Remote side effects** - commits pushed, API calls, DB writes, deployments.
- לכן Claude שואל לפני `git push`, `curl -X POST`, `npm publish`.
- Checkpoints ב-session only - לא משותפים בין sessions.

---

## התנהגות Context במעבר בין sessions

| פעולה | Context | Permissions | Auto memory | Skills invoked |
|-------|---------|-------------|-------------|----------------|
| New session (`claude`) | ריק (+ startup) | reset | נטענות (200 lines) | none |
| `--continue` | מלא מהקודם | **לא** משמרות | loaded from memory dir | loaded |
| `--resume <id>` | מלא ישן | **לא** משמרות | loaded | loaded |
| `--fork-session` | מלא ישן | **לא** משמרות | loaded | loaded |
| `/clear` | ריק | reset | נטענות שוב | none |
| `/compact` | מתכווץ (summary) | נשמרות | - | חלק מ-re-attach |

---

## פקודות session חיוניות

| Command | פעולה |
|---------|-------|
| `/clear` | מתחיל שיחה נקייה (שומר auto memory) |
| `/compact` | מסכם את השיחה, מפנה context |
| `/compact focus on X` | compaction ממוקד |
| `/context` | מציג שימוש בקונטקסט |
| `/cost` | מציג עלות session עד כה |
| `/memory` | פתיחת memory files לעריכה |
| `/status` | מצב session כללי |
| `/exit` | יציאה (session נשמר) |

---

## Teleport - העברת session בין environments

`claude --teleport` - מעביר session שרץ ב-web/iOS ל-terminal מקומי, ולהפך.

שימושי: התחלת משימה ארוכה ב-web (רץ בענן של Anthropic), ממשיך אותה ב-terminal המקומי כשחוזרים.

פרטים: [surfaces.md](#/deep/surfaces) (בהמשך)

---

## Best practices

1. **משימה גדולה → session משלה.** אל תערבב 3 משימות שונות באותו session. אחרי כל משימה - `/clear`.

2. **אם יצאת באמצע ותחזור - `--continue`.** אל תתחיל חדש "כדי לא להתבלבל".

3. **לפני ניסוי גישה אחרת - `--fork-session`.** אתה ממשיך באותה נקודה, אבל יש לך רשת בטחון.

4. **עבודה מקבילה → worktrees.** לא שני sessions על אותו directory.

5. **במשימות ארוכות - `/compact` לפני פעולה כבדה.** ממקסם את ה-context שנשאר.

6. **`/context` כל כמה זמן.** אם מתקרב ל-70% - החלט: compact, fork-session, או clear.

7. **Checkpoints = רשת בטחון, לא אסטרטגיה.** אל תסמוך עליהם לפני actions destructive. Claude עצמו שואל לפני.

---

## המשך → [02_extensibility/hooks.md](#/deep/hooks)
