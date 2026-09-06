# Troubleshooting - פתרונות לבעיות נפוצות

**מקור:** [troubleshooting](https://code.claude.com/docs/en/troubleshooting)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>משהו לא עובד? הקובץ הזה הוא <strong>"מה לעשות כש..."</strong>. אוסף של בעיות נפוצות עם פתרונות:</p>

<p>❌ <code>claude: command not found</code> → נתיבי ההתקנה.<br>
❌ Login נכשל → Firewall, proxy, אישורים.<br>
❌ Context מתמלא מהר → איך לצמצם, מה לא לטעון.<br>
❌ Skill לא מופעל → איך לבדוק למה, איך לתקן.<br>
❌ MCP server לא מתחבר → Logs, timeouts, הגדרות.<br>
❌ Plugin לא עובד → reload, permissions.</p>

<p>בנוסף: <strong>כלי אבחון אוטומטי</strong> (<code>claude doctor</code>) שבודק בעצמו אם ההתקנה שלך תקינה, ו-<strong>debug mode</strong> שמראה מה באמת קורה מתחת למכסה המנוע.</p>

<p>כשיש בעיה - בוא לכאן קודם לפני שמחפשים ב-Google.</p>
</div>

---

## Installation

### `claude: command not found`

- Native install: `curl -fsSL https://claude.ai/install.sh | bash` - add to PATH
- Homebrew: `brew install --cask claude-code`
- Windows: `winget install Anthropic.ClaudeCode` או PowerShell installer
- `echo $PATH` - בדוק ש-`~/.local/bin` או install dir נמצא

### עדכון נכשל

- Native: self-updating, בדוק logs
- Homebrew: `brew upgrade claude-code` או `@latest`
- WinGet: `winget upgrade Anthropic.ClaudeCode`
- Manual: `claude update`

### Windows - "token && is not a valid statement separator"

אתה ב-PowerShell, לא ב-CMD. Prompt שמראה `PS C:\` = PowerShell. Use the PowerShell install command.

---

## Authentication

### Login fails

```bash
claude auth status --text
```

בעיות נפוצות:
- Firewall/proxy - ראה [network-config](https://code.claude.com/docs/en/network-config)
- OAuth callback blocked - `--callback-port` ל-port זמין
- Corporate network - MDM settings עוקפות

### "Invalid API key"

```bash
echo $ANTHROPIC_API_KEY    # בדוק
claude auth logout
claude auth login --console    # re-login with API key
```

---

## Runtime

### Auto-compaction loops / thrashing

תסמין: `/compact` רץ שוב ושוב, context לא מתנקה.

סיבה: קובץ/tool output אחד כל כך גדול שמפולא מיד אחרי summary.

פתרון:
- Read עם `offset`/`limit`
- Grep במקום Read
- Subagent לקריאה של הקובץ
- `/clear` והתחלה נקייה

### "Context limit exceeded" לפני שעבדת הרבה

- MCP tools נטענים upfront? Check `ENABLE_TOOL_SEARCH=true`
- Huge CLAUDE.md? קצר ל-200 שורות, hoist reference ל-`.claude/rules/`
- כל CLAUDE.md ב-subdirectories נטענים? `claudeMdExcludes` לצמצם
- `/context` - רואה מה צורך

### Skills לא מופעלים אוטומטית

1. `/skills` - נטען?
2. `description` - keywords natural?
3. `SLASH_COMMAND_TOOL_CHAR_BUDGET` - descriptions נחתכים?
4. `disable-model-invocation: true` בטעות?

### Hooks לא רצים

1. `/hooks` - רשום?
2. Exit code: `echo '{...}' | bash script.sh; echo $?`
3. Timeout - default קצר. `"timeout": 600`
4. Plugin hook? plugins לא יכולים hooks ב-frontmatter של agents

### MCP server לא מתחבר

1. `/mcp` - מה ה-status?
2. Logs: `~/.claude/logs/mcp-*.log`
3. `MCP_TIMEOUT=30000 claude` - אם server איטי
4. Windows stdio: wrap עם `cmd /c`
5. OAuth loop: `claude mcp reset-project-choices`

### Bash command blocked בשגיאה מסתורית

1. `/permissions` - deny rule matching?
2. Hook PreToolUse שחוזר exit 2?
3. Sandbox filesystem restriction?
4. Process wrapper לא מוכר? (`devbox run ...` לא match `Bash(npm *)`)

---

## Performance

### Claude איטי

- `/model` → sonnet במקום opus (אלא אם צריך opus)
- Context דחוס? `/context` - אם > 80%, `/compact`
- Too many MCP servers? Remove unused
- Network? check `ANTHROPIC_BASE_URL`

### Tool calls אטיים

- Parallel tools אינדיפנדנטיים ב-single message
- Subagent למחקר ארוך במקום main thread
- Cache warming - prompt כמעט-זהה מקבל cache hit ענק (5-10x זול ומהיר)

### Streaming partial

`--output-format stream-json --include-partial-messages` ל-real-time chunks.

---

## Permission prompts nonstop

- `/permissions` → "Yes, don't ask again" לpatterns rutine
- `defaultMode: "acceptEdits"` ב-settings אם batch pattern
- `additionalDirectories` - אם הפרויקט cross מספר folders

---

## Auto memory

### "לא רואה שהוא זוכר"

1. `/memory` → רשימת memories
2. האם `~/.claude/projects/<hash>/memory/MEMORY.md` מאוכלס?
3. `CLAUDE_CODE_DISABLE_AUTO_MEMORY` set?
4. גרסה מעל 2.1.59? `claude --version`

### Memory ישן / לא רלוונטי

- `/memory` → פתח MEMORY.md, edit או delete
- "בקש מ-Claude: review and prune your memory"

---

## Plugins

### Plugin לא מופעל אחרי install

`~/.claude/settings.json`:
```json
{
  "enabledPlugins": ["marketplace-name:plugin-name"]
}
```

Or UI: `/plugin` → toggle.

### Plugin skill לא מופיע

- `/reload-plugins`
- Restart session אם הוספת new `skills/` directory
- namespace correct? `/plugin-name:skill-name`

### Plugin hook לא רץ

- `hooks/hooks.json` בparse? JSON valid?
- Plugin enabled?
- `/hooks` - רואה את ה-hook?

---

## SDK

### Python SDK נכשל עם "Node.js not found"

Python SDK = wrapper של Node. התקן:
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo bash -
sudo apt-get install -y nodejs
```

### "Session not persisting"

- Container ephemeral? mount `/root/.claude/` or `~/.claude/` ל-volume
- `no_session_persistence=True`? check options

### Hook callback לא נקרא

- `HookMatcher` - matcher pattern?
- `hooks={"PreToolUse": [HookMatcher(matcher="Bash", hooks=[callback])]}`
- async function?

### Custom tool לא נמצא

- `mcp__{server_name}__{tool_name}` - check naming
- `allowed_tools` - חייב להכיל את ה-tool
- Server registered in `mcp_servers`

---

## `--safe-mode` - בידוד התקלה (הצעד הראשון)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>הסשן מתנהג מוזר - קורס, מתעלם מהוראות, או עושה משהו שלא ביקשת - ואתה לא יודע למה. לפני שאתה מתחיל לחפש את האשם, יש שאלה אחת שצריך לענות עליה ראשונה: <strong>האם הבעיה ב-Claude עצמו, או באחת ההתאמות האישיות שלך</strong> (hook שכתבת, plugin שהתקנת, skill, MCP server, או ה-CLAUDE.md שלך)?</p>

<p>הדגל <code>--safe-mode</code> מפעיל את Claude Code <strong>בלי שום התאמה אישית</strong> - כאילו התקנת אותו עכשיו, נקי. אם בלי כל ההגדרות שלך הבעיה נעלמה - אתה יודע שהאשם הוא בקונפיג שלך, וצמצמת את החיפוש דרמטית. אם הבעיה ממשיכה גם נקי - היא לא בקונפיג שלך, וכדאי לדווח עליה ל-Anthropic.</p>

<p>זה בדיוק כמו להפעיל את Windows ב-Safe Mode: מכבים הכול חוץ מהבסיס, רואים אם הבעיה נשארה, וכך מגלים מי גורם לה.</p>
</div>

זמין מגרסה **2.1.169** (8 ביוני 2026).

```bash
claude --safe-mode
```

מה הוא מכבה (הכול בבת אחת):
- **Hooks** - כל ה-hooks שלך לא רצים
- **Plugins** - אף plugin לא נטען
- **Skills** - שום skill לא זמין
- **MCP servers** - אף server חיצוני לא מתחבר
- **Custom CLAUDE.md** - ההוראות האישיות שלך לא נטענות

איך משתמשים בזה לאבחון:

1. הסשן הרגיל מתנהג רע → סגור והרץ `claude --safe-mode`.
2. **הבעיה נעלמה?** האשם הוא אחת ההתאמות שלך. עכשיו תחזיר אותן אחת-אחת (`/hooks`, `/plugin`, `/skills`, `/mcp`, CLAUDE.md) עד שהבעיה חוזרת - וזיהית את האשם.
3. **הבעיה נשארה?** היא לא בקונפיג שלך - זה Claude עצמו. תעד מה קורה ודווח ב-[GitHub issues](https://github.com/anthropics/claude-code/issues).

> זה הצעד הראשון המומלץ כמעט לכל תקלה עמומה - לפני `claude doctor` ולפני `--debug` - כי הוא חוסך לך לנחש: במכה אחת אתה יודע באיזה צד של הגדר לחפש.

---

## `/doctor` - אבחון אוטומטי

```bash
claude doctor
```

בודק:
- Installation integrity
- Node.js version (SDK)
- `~/.claude/` permissions
- Credentials store
- MCP server health
- Plugin conflicts
- Settings JSON validity

---

## Debug mode

```bash
claude --debug "api,hooks,mcp"
# או
claude --debug-file /tmp/claude.log
```

Categories:
- `api` - API calls
- `hooks` - hook execution
- `mcp` - MCP transactions
- `permissions` - permission decisions
- `memory` - CLAUDE.md / auto memory
- `tool` - tool calls
- `statsig` - feature flags
- `!category` - exclude

### Log locations

- `~/.claude/logs/` - general
- Specific via `--debug-file`
- `~/.claude/projects/<hash>/<session>.jsonl` - session log

---

## Graceful recovery

### אפשר לשחזר session שסגרתי בטעות?

```bash
claude --resume     # בוחר אינטראקטיבית
# או עם שם:
claude --resume my-feature
```

Session נשמר גם אם crash.

### "חזרתי מsession ויש משהו שגוי"

- Esc Esc - undo
- `git status` / `git diff` - רואה מה השתנה
- `git checkout -- <file>` לhard revert
- `/rewind` - רואה snapshots, חוזר לנקודה ספציפית

### crashed באמצע טעם

Session נשמר up to crash. `claude --continue` - ממשיך.

---

## כיוון הראשון

1. **`claude --safe-mode`** - בידוד: האם הבעיה בקונפיג שלך או ב-Claude עצמו.
2. **Google עם המסר המדויק.**
3. **`claude doctor`** - auto-check.
4. **`/status`** - session meta.
5. **`--debug`** - מה קורה בפועל.
6. **Discord / GitHub issues** - [anthropics/claude-code](https://github.com/anthropics/claude-code/issues).
7. **`/help`** - in-session.

---

## חזור ל-[INDEX.md](#/deep)
