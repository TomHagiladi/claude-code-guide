# Plugins - אריזה להפצה

> Plugin = אריזת כל ההרחבות שלך (skills, agents, hooks, MCP, LSP, executables) למודול אחד ניתן להפצה. נגד `.claude/` "סטנדאלון" - plugins מאפשרים גרסאות, marketplaces, shared namespaces.

**מקורות:** [plugins](https://code.claude.com/docs/en/plugins) · [plugins-reference](https://code.claude.com/docs/en/plugins-reference) · [plugin-marketplaces](https://code.claude.com/docs/en/plugin-marketplaces) · [discover-plugins](https://code.claude.com/docs/en/discover-plugins)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>בנית לעצמך hooks שעוזרים בעבודה, skills מוצלחים, ו-subagent מתמחה. <strong>איך משתפים את זה עם הצוות או עם העולם?</strong></p>

<p><strong>Plugin זה אריזה אחת שכוללת הכל</strong> - כמו app בחנות אפליקציות. אתה אורז את ה-skills, agents, hooks וחיבורי ה-MCP שלך לתיקייה אחת, מעלה ל-GitHub, וכל אחד מהצוות יכול להתקין עם פקודה אחת.</p>

<p>יש <strong>marketplace רשמי</strong> של Anthropic עם plugins מוכנים, ואתה יכול להקים marketplace משלך לצוות.</p>

<p>הקובץ מסביר את מבנה התיקייה, את ה-manifest (קובץ הקונפיג), איך להפיץ, ואיך לוודא שזה בטוח (plugins לא יכולים לעשות הכל - יש הגבלות מיוחדות).</p>
</div>

---

## מתי plugin, מתי standalone

| גישה | שם skill | מתאים ל |
|------|----------|---------|
| Standalone (`.claude/`) | `/hello` | personal, project-specific, experimentation |
| Plugin | `/plugin-name:hello` | team sharing, community distribution, versioning, reuse |

התחל standalone, שדרג ל-plugin כשבגרת ל-share.

---

## מבנה מינימלי

```
my-plugin/
└── .claude-plugin/
    └── plugin.json
```

### plugin.json minimal

```json
{
  "name": "my-plugin",
  "description": "What it does",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  }
}
```

### plugin.json מלא

```json
{
  "name": "my-plugin",
  "description": "...",
  "version": "1.0.0",
  "author": {
    "name": "...",
    "email": "...",
    "url": "..."
  },
  "homepage": "https://...",
  "repository": "https://github.com/...",
  "license": "MIT",
  "keywords": ["keyword1"],
  "mcpServers": { /* inline - אלטרנטיבה ל-.mcp.json */ },
  "settings": { /* defaults - נדרס ע"י settings.json */ }
}
```

---

## מבנה מלא - תיקיות

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json               # המניפסט (חובה רק בשורש .claude-plugin/)
├── skills/                        # skills כ-directories
│   └── hello/
│       └── SKILL.md
├── commands/                      # flat markdown (legacy, תעדיף skills/)
│   └── deploy.md
├── agents/                        # subagents
│   └── reviewer.md
├── hooks/
│   └── hooks.json                 # hooks configuration
├── .mcp.json                      # MCP servers
├── .lsp.json                      # LSP servers (code intelligence)
├── bin/                           # executables - נוספים ל-PATH
│   └── my-tool
└── settings.json                  # defaults
```

**⚠️ Common mistake:** אל תשים `commands/`, `agents/`, `skills/` בתוך `.claude-plugin/`. רק `plugin.json` שם.

---

## Namespacing

Skills של plugins **תמיד ממוספרים**:

```
plugin "my-plugin" + skill "hello" → /my-plugin:hello
```

מונע התנגשות בין plugins. אי אפשר להשבית (חוץ משינוי `name` ב-plugin.json).

---

## Hooks ב-plugin

```
my-plugin/hooks/hooks.json
```

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [{
          "type": "command",
          "command": "jq -r '.tool_input.file_path' | xargs npm run lint:fix"
        }]
      }
    ]
  }
}
```

אותה סכמה בדיוק כמו ב-settings.json. רצים כש-plugin פעיל.

### Security - מה plugins לא יכולים

**Plugin subagents לא תומכים:**
- `hooks` frontmatter
- `mcpServers` frontmatter
- `permissionMode` frontmatter

הסיבה: plugin לא אמור להריץ shell שרירותי ללא אישור ברור. רוצה? העתק את ה-agent ל-`.claude/agents/` או `~/.claude/agents/`.

---

## MCP בplugin

שתי דרכים:

### 1. `.mcp.json` בשורש plugin

```json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "${CLAUDE_PLUGIN_ROOT}/bin/my-mcp-server",
      "args": ["--verbose"]
    }
  }
}
```

### 2. Inline ב-plugin.json

```json
{
  "name": "...",
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "${CLAUDE_PLUGIN_ROOT}/bin/my-mcp-server"
    }
  }
}
```

---

## Environment variables ב-plugin

| Var | תוכן |
|-----|------|
| `${CLAUDE_PLUGIN_ROOT}` | install directory של plugin |
| `${CLAUDE_PLUGIN_DATA}` | persistent data directory (נקי בין runs?) |
| `$CLAUDE_PROJECT_DIR` | project root |

השתמש ב-paths מוחלטים יחסית ל-`${CLAUDE_PLUGIN_ROOT}` - לעולם אל תניח cwd.

---

## Default settings

```json
// my-plugin/settings.json
{
  "agent": "security-reviewer",
  "subagentStatusLine": {...}
}
```

**תמיכה מוגבלת.** כרגע רק `agent` ו-`subagentStatusLine` נתמכים. מפתחות אחרים מתעלמים.

`agent: "security-reviewer"` → הפעלת plugin משנה את ה-main thread ל-agent הזה (System prompt, tools, model). זה plugin שמשנה את התנהגות Claude Code ברירת מחדל.

---

## LSP servers - code intelligence

```json
// my-plugin/.lsp.json
{
  "go": {
    "command": "gopls",
    "args": ["serve"],
    "extensionToLanguage": {
      ".go": "go"
    }
  }
}
```

משתמש חייב שה-LSP binary מותקן. לשפות common - יש plugins רשמיים, אל תבנה חדש.

---

## bin/ - executables ל-PATH

כל מה ש-`bin/` נוסף ל-PATH של Bash כש-plugin מופעל:

```
my-plugin/bin/my-cli        # chmod +x
```

Claude יכול:
```bash
my-cli --help               # רץ
```

ב-plugin.json לא צריך declare - רק לשים שם ולתת execute permission.

---

## Testing

```bash
claude --plugin-dir ./my-plugin
```

- טוען plugin ישירות, בלי install
- אפשר מספר: `claude --plugin-dir ./p1 --plugin-dir ./p2`
- שם זהה של local עם marketplace → **local מנצח** (חוץ מ-managed force-enabled)

### `/reload-plugins`

אחרי שינוי קבצי plugin - pick up changes בלי restart. מטעין מחדש: plugins, skills, agents, hooks, MCP servers, LSP servers.

---

## Marketplaces

### `marketplace.json`

```json
{
  "name": "my-org-marketplace",
  "description": "...",
  "plugins": [
    {
      "name": "plugin-1",
      "source": {
        "type": "github",
        "repo": "myorg/plugin-1",
        "ref": "v1.0.0"
      }
    },
    {
      "name": "plugin-2",
      "source": {
        "type": "directory",
        "path": "./plugins/plugin-2"
      }
    }
  ]
}
```

### הוספת marketplace

```bash
claude plugin marketplace add github.com/org/marketplace-repo
claude plugin marketplace add --path ./local-marketplace
```

או ב-`settings.json`:
```json
{
  "extraKnownMarketplaces": [
    {"type": "github", "repo": "org/my-marketplace"}
  ]
}
```

### התקנת plugin

```bash
claude plugin install marketplace-name:plugin-name
claude plugin install marketplace-name:plugin-name@v1.0.0
```

או interactive: `/plugin install`.

### הפעלה

Plugins **מותקנים לא אוטומטית פעילים**. הפעלה:

```json
// ~/.claude/settings.json
{
  "enabledPlugins": ["marketplace-name:plugin-name"]
}
```

או UI: `/plugin` → toggle.

---

## Managed marketplaces (ארגוני)

```json
// managed settings
{
  "blockedMarketplaces": [
    {"type": "github", "repo": "untrusted/*"}
  ],
  "strictKnownMarketplaces": true,     // רק ממש allowed
  "enabledPlugins": ["trusted-market:security-plugin"]    // force-enabled
}
```

`pluginTrustMessage` - הודעה מותאמת לdialog האישור.

---

## Trust model

- **Installing plugin ≠ enabling.** install מוריד לדיסק, enable מפעיל.
- **Enable דורש אישור ראשוני.** משתמש רואה מה הplugin עושה (skills, hooks, MCP), מאשר.
- **Plugins לא יכולים לערוך settings שלך אוטומטית.** רק דרך hooks שדורשים exec explicit.
- **Plugins עם MCP servers** - Claude Code מציג אותם בdialog.

---

## Version management

Semantic versioning - `major.minor.patch`:
- Major - breaking changes
- Minor - new features, backward-compat
- Patch - bug fixes

Marketplace יכול לspecify `ref: "v1.2.3"` או `ref: "main"` (latest).

```bash
claude plugin update <name>                    # latest
claude plugin update <name>@v2.0.0             # ספציפי
```

---

## Migration - מ-`.claude/` ל-plugin

```bash
mkdir -p my-plugin/.claude-plugin
# copy files:
cp -r .claude/commands my-plugin/
cp -r .claude/agents my-plugin/
cp -r .claude/skills my-plugin/
# hooks - מ-settings.json ל-hooks/hooks.json
```

Test:
```bash
claude --plugin-dir ./my-plugin
```

---

## הפצה רשמית (Anthropic marketplace)

- Claude.ai: [claude.ai/settings/plugins/submit](https://claude.ai/settings/plugins/submit)
- Console: [platform.claude.com/plugins/submit](https://platform.claude.com/plugins/submit)

---

## המשך → [channels_routines.md](#/deep/channels) · [advanced_features.md](#/deep/advanced)
