# תכונות מתקדמות - plan mode, thinking, agent teams, ultraplan, computer use

> יכולות מתקדמות שמפרידות שימוש בסיסי משימוש ברמת מומחה.

**מקורות:** [permission-modes](https://code.claude.com/docs/en/permission-modes) · [common-workflows](https://code.claude.com/docs/en/common-workflows) · [agent-teams](https://code.claude.com/docs/en/agent-teams) · [ultraplan](https://code.claude.com/docs/en/ultraplan) · [computer-use](https://code.claude.com/docs/en/computer-use) · [checkpointing](https://code.claude.com/docs/en/checkpointing)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>אוסף של <strong>יכולות מתקדמות</strong> ש"מפרידות בין משתמש רגיל למומחה":</p>

<p>📋 <strong>Plan mode</strong>: לפני שClaude עובד על משימה מורכבת, הוא קודם <em>חוקר ומכין תכנית</em>, אתה מאשר, ואז הוא מתחיל לבצע. חוסך המון טעויות.</p>

<p>🧠 <strong>Ultrathink</strong>: Claude "חושב לעומק" לפני שעונה - תהליך חשיבה נסתר שמשפר החלטות על בעיות מורכבות.</p>

<p>👥 <strong>Agent Teams</strong>: כמה Claude-ים עובדים יחד על משימה - אחד מתכנן, אחד מיישם, אחד סוקר.</p>

<p>🖱️ <strong>Computer Use</strong>: Claude "רואה" את המסך שלך ויכול לקליק, להקליד. שימושי לבדיקת אפליקציות UI שאי אפשר לבדוק מה-terminal.</p>

<p>↩️ <strong>Checkpoints</strong>: כל שינוי קובץ ניתן לביטול. Esc Esc וחזרת למצב הקודם.</p>
</div>

---

## Plan Mode

### מה זה

Read-only mode. Claude יכול לנתח, לחפש, להבין - אבל לא לערוך או להריץ shell. מייצר plan, אתה מאשר לפני execution.

### הפעלה

- `Shift+Tab` (tabs through modes)
- `claude --permission-mode plan`
- בתוך session: UI toggle

### מתי

- **משימה מורכבת** - OAuth integration, migration, refactor גדול
- **codebase לא מוכר** - רוצה שClaude יבין לפני שיתקן
- **פעולה קריטית** - database migration, production deploy
- **research before implementation** - "understand first, propose approach"

### Plan subagent

כש-Claude ב-plan mode צריך לחקור codebase, הוא מאציל ל-Plan subagent (מונע recursive nesting).

### Flow

```
User: "Add OAuth to the app" → Plan mode
Claude: [Reads auth code, identifies integration points]
Claude: [Generates plan with steps, files to change, tests to add]
User: [Reviews, refines via conversation]
User: "Looks good, implement it" → Exit plan mode
Claude: [Executes the plan]
```

---

## Extended thinking (ultrathink)

### מה זה

Claude "חושב" לפני שעונה - מריץ reasoning tokens שלא מוצגים. תוצאה: reasoning עמוק יותר על בעיות מורכבות.

### איך

1. **Explicit** - הוסף "think" או "ultrathink" ב-prompt:
   ```
   ultrathink about how to refactor this auth system
   ```

2. **בsettings** - default thinking per project/session.

3. **בsakill** - הוסף "ultrathink" בכל מקום ב-skill content.

### מתי

- Architecture decisions
- Debugging מסובך
- Trade-off analysis
- Algorithmic problems
- Security reviews

### עלות

Thinking tokens נספרים ב-usage (input-side). משמעותי אבל הרבה יותר זול מ-output tokens.

---

## /goal - תנאי סיום אוטומטי

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>אתה מגדיר ל-Claude את <strong>"קו הסיום"</strong> - מתי המשימה נחשבת גמורה (למשל "סיים כשכל הטסטים עוברים"). אחרי כל צעד, מודל מהיר בודק "הגענו?", ו-Claude ממשיך לעבוד לבד עד שהתנאי מתקיים.</p>
</div>

### מה זה

`/goal <condition>` - מגדיר condition לסיום המשימה. אחרי כל turn, מודל מהיר בודק מחדש אם התנאי התקיים, ו-Claude ממשיך לעבוד עד שהוא מתקיים.

### מתי

- משימה ארוכה שאתה לא רוצה ללוות צעד-צעד
- "תמשיך עד שכל הטסטים עוברים" / "עד שאין שגיאות lint"

---

## Agent Teams

### מה זה

Multiple Claude instances (teammates) שעובדים יחד על session משותפת. **שונה מ-subagents** - subagents ב-session אחת, teammates ב-sessions נפרדות שמתאמות.

### Use cases

- Architect + Implementer + Reviewer - each a separate agent
- Parallel work על עבודה שמתחלקת למרכיבים עצמאיים
- "Pair programming" - שני agents, roles שונים

### מושגים

| מושג | תפקיד |
|------|-------|
| Team lead | ה-main session שמצרף teammates |
| Teammate | instance שמצטרף לteam |
| Shared task | TodoWrite משותף לכל ה-teammates |
| Inter-agent messaging | teammates יכולים לשלוח הודעות זה לזה |
| TeammateIdle hook | fires כש-teammate free |

### יצירה

אין פקודת `/team` (נבדק 4.9.2026 מול עמוד הפקודות). מבקשים בשפה חופשית - "צור צוות סוכנים שיעבוד על..." - או programmatically ב-SDK.

---

## Agent View + background sessions

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>אפשר להריץ <strong>הרבה sessions של Claude במקביל</strong> - חלקם רצים "ברקע" בזמן שאתה עושה דברים אחרים. <strong>Agent View</strong> הוא לוח בקרה במסך מלא שמראה את כולם במבט אחד: מי עובד עכשיו, מי מחכה לתשובה ממך, ומי כבר סיים.</p>
</div>

### `claude agents` - לוח הבקרה

Agent View - dashboard במסך מלא של כל ה-sessions שלך, מסווגים לשלוש קבוצות: **Working** (עובד), **Needs input** (מחכה לך), ו-**Completed** (סיים). מתוך הלוח אתה גם שולח sessions חדשים שרצים ברקע (detachable background sessions).

### הפעלה ברקע

- `claude --bg` - מפעיל session חדש ישר ברקע
- `/background` או `/bg` - שולח את ה-session הנוכחי לרקע

Per-user supervisor daemon מנהל את ה-sessions ברקע, גם אחרי שסגרת את החלון.

### ניהול לפי short id

לכל session ברקע יש id קצר:

- `claude attach <id>` - חזרה לתוך ה-session
- `claude logs <id>` - צפייה בלוג שלו
- `claude stop <id>` - עצירה

---

## Dynamic Workflows (`/workflows`)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>אתה מתאר משימה גדולה, ו-Claude כותב בעצמו <strong>תוכנית הרצה קטנה</strong> (script ב-JavaScript) שמפעילה ומתאמת <strong>עשרות עד מאות עוזרי-Claude קטנים</strong> שרצים ברקע במקביל. אפשר לשמור ריצה כזו ולהפוך אותה לפקודה קבועה משלך. (אגב - דוח עדכוני Claude Code שקיבלת נבנה בדיוק ככה.)</p>
</div>

### מה זה

Dynamic Workflows (`/workflows`): Claude כותב script של JavaScript ש"מנצח על התזמורת" - מתזמר (orchestrates) עשרות עד מאות sub-agents שרצים ברקע. אפילו עצם הופעת המילה "workflow" ב-prompt יכולה להפעיל את זה אוטומטית.

### שמירה כפקודה לשימוש חוזר

ריצה אפשר לשמור כקובץ `.claude/workflows/<name>.js`, והיא הופכת לפקודה `/<name>` שאפשר להריץ שוב.

### `/deep-research`

מגיע מובנה workflow בשם `/deep-research` - מחקר עומק רב-מקורות בלחיצה אחת.

---

## Ultraplan

> **אזהרה:** `/ultraplan` **הוסר** מ-Claude Code (עמוד הפקודות הרשמי, נבדק 4.9.2026). במקומו: מצב תכנון רגיל - `/plan` או `Shift+Tab`. הסעיף נשאר כתיעוד היסטורי בלבד.


### מה זה

Plan שמתחיל ב-terminal, ממשיך ב-web (Anthropic infra), ונשמר.

### Flow

1. `claude` ב-terminal - התחל לחשוב
2. `/ultraplan` - ה-plan עובר ל-web
3. ב-[claude.ai/code](https://claude.ai/code) - תכנון deep, גישה ל-MCP servers, resources
4. חזרה ל-terminal עם `claude --teleport <plan-id>` - ממשיך execution לוקלית

### מתי

Planning כבד שדורש:
- Heavy web research
- Long-running analysis
- Access ל-services שלא זמינים לוקלית
- Collaboration על plan - שיתוף URL

---

## Computer Use

### מה זה

Claude שולט במחשב - רואה מסך, קליקים, הקלדות. מ-CLI:

מ-CLI מפעילים דרך `/mcp` (macOS בלבד). באפליקציית הדסקטופ: `Settings → General → Computer use`, ב-macOS וב-Windows. דורש מנוי Pro או Max - **לא זמין ב-Team או Enterprise**. (עודכן 4.9.2026.)

### מה מאפשר

- **Testing native apps** - אפליקציות שלא ניתן לדבג מ-terminal
- **Automating GUI-only tools** - כלים ללא CLI/API
- **Visual bug reproduction** - screenshots, click paths
- **Accessibility testing**

### איך

Screenshot tool + mouse/keyboard tools מוסיפים ל-Claude:
- `screenshot` - צילום מסך
- `click` - mouse click
- `type` - הקלדה
- `key` - מקש

### Safety

Sandboxing קריטי. Computer use יכול לעשות הכל במחשב שלך.

---

## Checkpointing - עמוק

### מודל

לפני כל Edit/Write:
- Snapshot של כל הקבצים שעומדים להשתנות
- נשמרים ב-memory של session (לא לדיסק)
- Live רק עד סוף session

### השימוש

- `Esc Esc` - undo last
- בקשה: "undo the last change" או "revert to before the refactor"
- `/rewind` - רואה את נקודות השמירה, בוחר לחזור (אין פקודת `/checkpoints`)

### מה לא כיסוי

- Git operations (commits, push) - לא הפיכים ע"י checkpoint
- API calls, DB writes - לא ניתן undo
- Deletes - תיקיות שנמחקו לא שוחזרות (Claude בד"כ משתמש ב-mv לtrash)

### Git integration

Checkpoints **נפרדים מ-git.** אפשר ליצור commit אחרי שהכל עובד, ולהישאר עם checkpoints למקרה שצריך לחזור.

### `/rewind` לפני `/clear` (v2.1.191, 24 ביוני 2026)

עד עכשיו, `/clear` (ניקוי השיחה) היה מוחק את ה-context לתמיד. מ-v2.1.191 הפקודה `/rewind` יכולה לחזור לנקודה *לפני* שהרצת `/clear` - ובכך לשחזר את ההקשר שהיה הולך לאיבוד. שימושי כשניקית בטעות, או כשהבנת שאתה עדיין צריך משהו מהשיחה שנמחקה.

---

## Interactive features

### Bang mode - `!command`

```
!git status
!ls -la
```

הפקודה רצה מיד (אתה מריץ, לא Claude). Output + command שניהם נכנסים ל-context.

**Auto-response (v2.1.186, 22 ביוני 2026):** פקודות shell שאתה מריץ עם prefix של `!` עכשיו מפעילות *תגובה אוטומטית* של Claude ברגע שמופיע output - בלי שתצטרך לכתוב prompt נוסף. למשל `! npm test` שנכשל → Claude מסביר לבד מה נשבר ולמה. אם אתה לא רוצה שהוא יגיב על כל פקודה, מכבים עם ה-setting בשם `respondToBashCommands`.

בנוסף, Bash mode קיבל autocomplete חי לנתיבי קבצים תוך כדי הקלדה.

### @-mentions

```
@src/auth.ts       # refer to file
@github:issue://123   # MCP resource
```

אוטומטי attached כ-context.

### Slash commands

- Built-in: `/help`, `/clear`, `/compact`, `/memory`, `/cost`, `/doctor`, `/context`, `/init`, `/agents`, `/permissions`, `/mcp`, `/hooks`, `/skills`, `/model`, `/team`, `/schedule`, `/plugin`, `/goal`, `/workflows`, `/code-review`, `/rewind`, `/background`
- Skills: `/skill-name` או `/plugin:skill-name`
- MCP prompts: `/mcp__server__prompt`

### Keyboard shortcuts

| מקש | פעולה |
|-----|-------|
| `Enter` | send |
| `Shift+Enter` | newline |
| `Esc` | stop current action |
| `Esc Esc` | undo (rewind checkpoint) |
| `Shift+Tab` | cycle permission modes |
| `Ctrl+C` (twice) | exit |
| `Ctrl+R` | search transcript |
| `Up/Down` | history of prompts |

ב-[keybindings](https://code.claude.com/docs/en/keybindings) - customize.

---

## Voice dictation

Push-to-talk:
- macOS / Linux: native
- hold key, speak, release → text מוזן

הפעלה: `/voice` או ב-settings.

---

## Fast mode

`/fast` - מפעיל fast mode. **אותו מודל** (Opus 5, וגם Opus 4.8), פלט מהיר יותר.

לא מתחלף לHaiku. שינוי הפורמט של streaming + optimizations.

---

## Fullscreen rendering

`/tui fullscreen` - עובר ל-flicker-free rendering (הפקודה הישנה `/fullscreen` כבר לא קיימת) עם mouse support. stable memory usage בשיחות ארוכות.

---

## Transcript search

`Ctrl+R` - חיפוש בהיסטוריית session. כולל tool outputs, code, הודעות שלך.

---

## Auto-fix PR

`/autofix-pr <number>` - Claude עובד על PR:
- Fetches PR
- Reads review comments
- Makes requested changes
- Pushes updates

זמין ב-terminal, ב-desktop, וב-routines.

---

## `/code-review`

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>Claude קורא את <strong>השינויים שעשית בקוד</strong> (ה-diff) ומצביע על באגים - ויכול גם לתקן אותם לבד או להשאיר הערות על ה-PR. הגרסה החזקה, <strong>ultra</strong>, שולחת "צוות" שלם לחפש באגים עמוקים בענן.</p>
</div>

### `/code-review` - סוקר מקומי

סוקר את ה-diff המקומי ומוצא באגים:

- `--fix` - מיישם את הממצאים על ה-working tree
- `--comment` - מפרסם הערות inline על ה-PR

### `/code-review ultra`

מפעיל deep bug-hunt בענן עם multi-agent על כל ה-branch. **3 ריצות חינם בחודש** למנויי Max. החליף את ה-alias הישן `/ultrareview`.

---

## `/team-onboarding`


מייצר **מדריך קליטה לצוות** מתוך היסטוריית השימוש שלך: Claude מנתח את הסשנים, הפקודות ושרתי ה-MCP מ-30 הימים האחרונים ומסכם איך אתה עובד, כדי שחבר צוות חדש יוכל להתחיל מאותה נקודה.

```
/team-onboarding
```

(תוקן 4.9.2026 מול עמוד הפקודות הרשמי - תיאור קודם בדף הזה, על אריזת ההגדרות כ-plugin, לא היה מדויק.)

---

## סיום → [05_reference/](#/deep/cli)
