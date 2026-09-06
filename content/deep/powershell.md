# PowerShell Tool - Windows-native shell

> ב-Windows, Claude Code מקבל כלי PowerShell מובְנה לצד כלי ה-Bash. בלי להתקין Git for Windows.

**מקורות:** [tools-reference](https://code.claude.com/docs/en/tools-reference) · [env-vars](https://code.claude.com/docs/en/env-vars) · [skills](https://code.claude.com/docs/en/skills) · [changelog](https://code.claude.com/docs/en/changelog) - נבדק 27.6.2026

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>כשClaude צריך להריץ פקודה במחשב - להעתיק קובץ, להריץ סקריפט, לבדוק מצב git - הוא עושה את זה דרך <strong>shell</strong> (מתורגמן פקודות). עד עכשיו, ב-Windows, ה-shell היחיד היה <strong>Bash</strong>, וכדי שהוא יעבוד היית חייב להתקין תוכנה נפרדת בשם Git for Windows.</p>

<p>עכשיו Claude Code מכיר גם את <strong>PowerShell</strong> - ה-shell הטבעי של Windows שכבר מותקן אצלך מהיום הראשון. זה כלי שווה-ערך (sibling) ל-Bash: Claude יכול לבחור איזה מהשניים מתאים יותר למשימה.</p>

<p>למה זה טוב בשבילך, כמשתמש Windows? <strong>בלי התקנות נוספות</strong>, ועם גישה ישירה לפקודות הטבעיות של Windows (cmdlets כמו <code>Get-ChildItem</code> או <code>Copy-Item</code>). הקובץ הזה מסביר איך מפעילים את זה, ומה ההבדל בתחביר בין שני ה-shells.</p>
</div>

---

## שני shells, כלים נפרדים

Claude Code מתייחס ל-PowerShell ול-Bash כ**שני כלים נפרדים** - לא flag אחד שמחליף את ה-shell, אלא שני tools שעומדים זה לצד זה. כשהכלי מופעל, Claude רואה גם `Bash` וגם `PowerShell` ברשימת הכלים שלו, ובוחר את המתאים למשימה.

המשמעות המעשית:

- ב-**macOS / Linux** - ה-Bash tool הוא ברירת המחדל, כמו תמיד.
- ב-**Windows** - אפשר להוסיף את ה-PowerShell tool, והוא הופך ל-shell טבעי בלי תלות ב-Git for Windows.
- כל פקודה רצה ב-shell שאליו היא נשלחה - אין "תרגום" אוטומטי בין השניים.

---

## הפעלה - `CLAUDE_CODE_USE_POWERSHELL_TOOL=1`

הכלי מופעל דרך environment variable:

```powershell
$env:CLAUDE_CODE_USE_POWERSHELL_TOOL = "1"
claude
```

או קבוע, ב-`~/.claude/settings.json`, תחת `env`:

```json
{
  "env": {
    "CLAUDE_CODE_USE_POWERSHELL_TOOL": "1"
  }
}
```

מרגע ההפעלה, ה-PowerShell tool זמין לצד ה-Bash tool בכל session על המכונה הזו.

---

## תחביר - כל shell והכללים שלו

זו הנקודה החשובה: **כל כלי מקבל את התחביר שלו**. אסור לערבב. אם Claude שולח פקודה ל-PowerShell tool, היא חייבת להיות PowerShell תקני; אם ל-Bash tool - POSIX תקני.

| נושא | PowerShell | Bash (POSIX) |
|------|------------|--------------|
| "כלום" / null | `$null` | `/dev/null` |
| משתנה סביבה | `$env:VAR` | `$VAR` |
| המשך שורה | backtick \` בסוף שורה | backslash `\` בסוף שורה |
| משתנה | `$myVar = "x"` | `myVar=x` |
| תנאי | `if (Test-Path x) {...}` | `if [ -f x ]; then ...; fi` |
| לולאה | `foreach ($i in ...) {...}` | `for i in ...; do ...; done` |

דוגמה - אותה משימה (להשתיק שגיאות) בשני ה-shells:

```powershell
# PowerShell
Get-Content missing.txt 2>$null
```

```bash
# Bash
cat missing.txt 2>/dev/null
```

טעות נפוצה: לשלוח תחביר Bash (כמו `$VAR` או `/dev/null`) לכלי ה-PowerShell. זה פשוט לא יעבוד - כל כלי מצפה לדקדוק שלו.

---

## Skills - `shell: powershell`

Skill יכול להכריז באיזה shell הפקודות שלו רצות, דרך שדה `shell` ב-frontmatter (ראה [skills.md](#/deep/skills)):

```yaml
---
name: my-windows-skill
description: ...
shell: powershell
---
```

| ערך | משמעות |
|-----|---------|
| `bash` | ברירת מחדל - פקודות ה-skill רצות ב-Bash |
| `powershell` | פקודות ה-skill רצות ב-PowerShell (Windows) |

כך אפשר לכתוב skill שמשתמש ב-cmdlets טבעיים של Windows, והוא יעבוד אצל כל משתמש Windows בלי דרישה ל-Git for Windows.

---

## למה זה חשוב למשתמש Windows

1. **בלי התקנות נוספות.** PowerShell מגיע מובְנה עם Windows. אין צורך ב-Git for Windows רק כדי לתת ל-Claude shell לעבוד איתו.

2. **cmdlets טבעיים.** גישה ישירה לפקודות Windows - `Get-ChildItem`, `Copy-Item`, `Get-Process`, `Test-Path` - במקום להתאמץ עם מקבילות POSIX דרך שכבת תאימות.

3. **התנהגות צפויה.** הנתיבים, הציטוטים (quoting) ומשתני הסביבה מתנהגים כמו שמשתמש Windows מצפה, לא כמו ב-emulation של Unix.

> שים לב: ב-environment הזה (המכונה של תום) ה-PowerShell tool הוא ה-shell הראשי, וה-Bash tool זמין בנוסף עבור סקריפטים בסגנון POSIX. זו בדיוק הסיטואציה של שני כלים אחים זה לצד זה.

---

## יוני 2026 - `classifyAllShell` (Auto-mode)

כשמפעילים shell חדש, נכנסת לתמונה גם שכבת הבטיחות. ב-**v2.1.193 (25.6.2026)** נוספה הגדרה שמרחיבה את ה-Auto-mode safety classifier כך שיכסה גם פקודות PowerShell, לא רק Bash:

```json
{
  "autoMode": {
    "classifyAllShell": true
  }
}
```

- `classifyAllShell: true` מנתב **כל** פקודת Bash ו-PowerShell דרך ה-Auto-mode safety classifier.
- ברירת המחדל (בלי ההגדרה) בודקת רק פקודות ש**נראות חשודות**; עם `true` - הכול עובר בדיקה.

מאותה גרסה שופרה גם השקיפות של דחיות: **סיבות הדחייה (denial reasons)** של Auto-mode מופיעות עכשיו בשלושה מקומות - ב-transcript של השיחה, ב-toast של הדחייה, וגם ב-`/permissions`. כך, אם פקודה נחסמה, רואים בדיוק למה.

ההגדרה המלאה של `autoMode` מתועדת ב-[settings_env.md](#/deep/settings#auto-mode).

---

## המשך → [tools_reference.md](#/deep/tools) · [settings_env.md](#/deep/settings) · [skills.md](#/deep/skills)
