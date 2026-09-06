# Surfaces - איפה Claude Code רץ

> אותו engine, ממשקים שונים. CLAUDE.md, settings, MCP servers עובדים בכל מקום.

**מקורות:** [platforms](https://code.claude.com/docs/en/platforms) · [desktop](https://code.claude.com/docs/en/desktop) · [vs-code](https://code.claude.com/docs/en/vs-code) · [jetbrains](https://code.claude.com/docs/en/jetbrains) · [claude-code-on-the-web](https://code.claude.com/docs/en/claude-code-on-the-web) · [chrome](https://code.claude.com/docs/en/chrome) · [slack](https://code.claude.com/docs/en/slack) · [remote-control](https://code.claude.com/docs/en/remote-control)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>Claude Code זה <strong>מנוע אחד, אבל הרבה "פרצופים"</strong>. אותו Claude, אותם יכולות - רק הממשק משתנה:</p>

<p>💻 <strong>Terminal</strong> - הקלאסי, הכי מהיר, הכי חזק.<br>
🎨 <strong>VS Code extension</strong> - רואה diffs בעורך, נוח לעבודה יומיומית.<br>
🧠 <strong>JetBrains</strong> - למי שמשתמש ב-IntelliJ/PyCharm/וכו'.<br>
🖥️ <strong>Desktop app</strong> - מראה diffs ויזואלי, כמה sessions במקביל, "Computer Use".<br>
🌐 <strong>Web (claude.ai/code)</strong> - בלי התקנה, רץ בענן, טוב למשימות ארוכות.<br>
📱 <strong>Remote Control</strong> - לשלוט ב-Claude של המחשב שלך מהטלפון.<br>
💬 <strong>Slack</strong> - לקרוא לClaude עם @mention.<br>
🔌 <strong>Chrome integration</strong> - לדבג אתרי web.</p>

<p>הטוב? <strong>ההגדרות שלך (CLAUDE.md, skills, MCP) עובדות בכל מקום</strong>. אתה מגדיר פעם אחת - ויש לך בכל הפרצופים.</p>
</div>

---

## Terminal (CLI)

### היתרונות

- **Full capability** - הכל זמין, שום הגבלה
- **Unix pipeable** - `|`, `>`, `<`, CI-friendly
- **Fast** - שום overhead UI
- **Scriptable** - אוטומציה trivial

### מתי

- משימות מורכבות
- רב-קבציות
- CI/CD
- scripting

### Key features

- Interactive mode + `-p` print mode
- Bash prefix (`!cmd`)
- `@` mentions לfiles + MCP resources
- Extensive keybindings
- Voice dictation (push-to-talk)

---

## VS Code extension

### היתרונות

- **Inline diffs** - רואה שינויים בעורך
- **@-mentions** משולבים עם files picker
- **Plan review** בUI
- **Conversation history** באקורדיון
- **Keyboard shortcuts** native (Cmd+Shift+P)

### Install

```
cmd+shift+P → "Install Extension" → search "Claude Code"
```

### שילוב עם terminal

הarchitecture: Extension מפעיל instance של Claude שרץ ברקע. אתה יכול:
- Chat panel בsidebar
- Split - terminal + chat
- Open in new tab - fullwindow mode

### Context מהעורך

- **Selection** - הסלקט נוכחי נשלח אוטומטית כ-context
- **Current file** - ידוע ל-Claude
- **Open files** - available via Read

### Settings

VS Code settings.json:
```json
{
  "claudeCode.defaultModel": "sonnet",
  "claudeCode.showDiffInline": true,
  "claudeCode.autoOpenInNewTab": false
}
```

### לימיטציות

- UI של inline diffs יכול להתעדכן לאט ב-codebases ענקיים
- TTY-dependent features (voice, keybindings מיוחדים) - לא בextension

---

## JetBrains IDEs

תמיכה ב: IntelliJ IDEA, PyCharm, WebStorm, GoLand, Rider, וכולם.

### Install

JetBrains Marketplace → "Claude Code"

### יכולות

- Interactive diff viewing
- Selection context sharing
- Tool windows integration
- Run configurations - Claude Code actions
- Git integration

### הבדלים מ-VS Code

- UI native של JetBrains (tool windows, not sidebar)
- Integration עם built-in tools (refactoring, debugging)
- Slower startup than VS Code ext

---

## Desktop app

App עצמאית (macOS, Windows x64, Windows ARM64, Linux בבטא). אותו מנוע כמו ה-terminal, בממשק גרפי.

### מה יש ב-desktop

- **Visual diff review** - סקירת שינויים עם הערות על שורות
- **Parallel sessions** - כמה session בו-זמנית, split view
- **Git isolation** - כל session מקבל worktree משלו, אוטומטית
- **Panes** - chat, diff, browser, terminal, file editor, tasks
- **App preview** - שרת פיתוח בחלון משותף, עם אימות עצמי
- **PR monitoring** - ניטור CI עם auto-fix ו-auto-merge
- **Connectors** - אשף גרפי ל-MCP
- **Dispatch** - start session מהטלפון, ממשיך ב-desktop
- **Computer use** - Claude רואה/קליק על המסך
- **Scheduled tasks** - ריצה על schedule עם גישה לקבצים מקומיים

> ב-Windows, **Git הוא תנאי הכרחי** להפעלת לשונית ה-Code - בשונה מה-terminal.

הפירוט המלא של כל אלה - כולל ארבע סביבות ההרצה, מצבי ההרשאה, ההגדרות הארגוניות
וטבלת ההשוואה מול ה-CLI - נמצא בדף הייעודי:

**→ [desktop_app.md](#/deep/desktop)**

---

## Web (claude.ai/code)

Claude בענן של Anthropic. שום install, שום local setup.

### יתרונות

- Mobile (iOS app), browser
- Kick off long tasks, close computer
- Repos שאין לוקלית
- Multiple parallel tasks

### הגבלות

- Sandbox מבוקר של Anthropic
- Docker configurable (setup scripts, network access)
- Integration עם GitHub (auth + PRs)
- עדיין מוגבל - chrome extension, MCP stdio local לא עובדים

### Teleport - חיבור ל-terminal

```bash
claude --teleport
# מחבר web session ל-terminal מקומי
```

### Remote - push task to web

```bash
claude --cloud "Fix login bug"   # --remote הוא כינוי ישן שעדיין עובד
# יוצר web session, פותח URL
```

---

## Artifacts - live published page

חדש: v2.1.178 (Week 25, 15-19 ביוני 2026). Beta - זמין ב-Team ו-Enterprise plans.

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>תאר לעצמך ש-Claude עובד על משהו, ובמקביל בונה לך <strong>דף אינטרנט חי</strong> שמראה את ההתקדמות - ומתעדכן מעצמו תוך כדי עבודה, בלי שתצטרך לרענן או לקבל קישור חדש.</p>

<p>זה מה ש-Artifacts עושה: Claude מפרסם מתוך ה-session דף <strong>אינטראקטיבי</strong> ל-URL פרטי על claude.ai. כל עוד ה-session ממשיך לעבוד, הדף משתנה "במקום" - אותו קישור בדיוק, רק שהתוכן מתעדכן. למשל: סקירת PR עם diffs מסומנים, dashboard של מצב משימה, או checklist חי שמסמן ✓ ליד כל צעד שכבר נעשה.</p>

<p>שים לב: זה <strong>Beta</strong>, וזמין רק ב-Team ו-Enterprise plans. כלומר ב-Max plan אישי (כמו שלך) זה כנראה <strong>לא</strong> זמין כרגע.</p>
</div>

### מה זה נותן

- **Live, in-place updates** - אותו URL, התוכן מתעדכן תוך כדי ש-ה-session רץ. לא צריך לרענן, ולא נוצר קישור חדש בכל פעם.
- **Interactive** - לא רק טקסט סטטי. דף שאפשר ללחוץ בו, לגלול ולסנן.
- **Private URL on claude.ai** - לא צריך לפרוס שום דבר (no deploy), והקישור פרטי.

### שימושים טובים

- **PR walkthroughs** - סקירת PR עם diffs מוערים (annotated), במקום קיר טקסט.
- **Dashboards** - תצוגה חיה של מצב משימה או נתונים.
- **Checklists** - רשימת צעדים שמסמנת מה כבר נעשה תוך כדי ש-Claude מתקדם.

### זמינות

- **Beta** - עדיין בפיתוח, ההתנהגות יכולה להשתנות.
- **Team + Enterprise plans בלבד** - נכון ל-v2.1.178 (יוני 2026). על Max plan אישי זה כנראה לא יופיע.

---

## Remote Control

Control local Claude Code מ-browser / iOS app.

### Setup

```bash
claude remote-control --name "My Project"
# server mode, לא interactive
```

או combined:
```bash
claude --remote-control "My Project"
# interactive + RC
```

### Use case

אתה מחוץ לבית, לפטופ שלך פתוח. רוצה להזיז session, להזין prompt חדש. iOS app / browser → מחובר ל-local.

### Restrictions

- Team / Enterprise ב-admin settings
- Defaults: on for individuals ב-API plans

---

## Chrome integration

Claude מדבר עם Chrome - רואה עמוד, console logs, DOM.

```bash
claude --chrome
```

### Use cases

- Debug live web app
- Automate form filling
- Extract data from pages
- E2E testing

### איך

Chrome extension + Claude Code extension mdebrים. Claude יכול:
- Navigate
- Click
- Read DOM
- Execute JS
- Screenshot
- Read console

---

## Slack

Claude ב-Slack - `@Claude` ב-channel או DM.

### Setup

Install Claude app ל-workspace → link GitHub org.

### Use cases

- "@Claude fix this bug in my-repo" - Claude מקבל, עובד, חוזר עם PR
- Triage: Slack message → Claude יוצר issue
- Meeting summaries, task assignments

### Configuration

Per-channel או workspace-wide settings:
- Allowed repos
- Default permissions
- Rate limits

---

## CI/CD

### GitHub Actions

```yaml
- uses: anthropics/claude-code-action@v1
  with:
    task: "Review this PR for security issues"
    repo: ${{ github.repository }}
    pr: ${{ github.event.pull_request.number }}
```

Pre-built actions:
- Code review
- Issue triage
- PR auto-fix

### GitHub Enterprise Server

Self-hosted GitHub? Claude Code יכול להתחבר. Setup ב-admin settings.

### GitLab CI/CD

Docker image + env vars:
```yaml
claude-review:
  image: anthropic/claude-code:latest
  script:
    - claude -p --output-format json "review the MR"
```

---

## Docker / Dev Containers

### DevContainer

`.devcontainer/devcontainer.json`:
```json
{
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
  "features": {
    "ghcr.io/anthropics/features/claude-code:1": {}
  },
  "mounts": [
    "source=${localEnv:HOME}/.claude,target=/home/vscode/.claude,type=bind"
  ]
}
```

Sessions ו-credentials persist דרך mount.

---

## Which surface - decision table

| תרחיש | הכי טוב |
|-------|---------|
| פיתוח יומיומי | Terminal או VS Code ext |
| Visual diff review | Desktop |
| Long-running task בענן | Web |
| Live progress page / PR walkthrough | Artifacts (Team/Enterprise, Beta) |
| Mobile / away from computer | Remote Control + iOS |
| CI/CD automation | SDK / GitHub Actions |
| Debug web app live | Chrome integration |
| Team collaboration async | Slack |
| Scheduled automation | Routines (cloud) או Desktop tasks |
| Data science notebooks | JetBrains (PyCharm) + VS Code |
| Monorepo עם ריבוי branches | Terminal + worktrees |

---

## Cross-surface config

כל ה-surfaces משתמשים ב:
- `~/.claude/settings.json` (user)
- `.claude/settings.json` (project)
- `CLAUDE.md` files
- `.claude/skills/`
- `.mcp.json`

**השינוי היחיד:** איפה sessions נשמרים + אילו tools זמינים (Chrome integration רק ב-terminal; computer use רק ב-desktop; וכו').
