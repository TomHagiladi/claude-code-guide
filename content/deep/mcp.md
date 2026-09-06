# MCP - Model Context Protocol

> סטנדרט פתוח לחיבור Claude Code לשירותים חיצוניים. Slack, GitHub, DBs, design tools - כולם דרך ממשק אחיד. Claude מקבל tools חדשים, resources חדשים, prompts חדשים.

**מקור:** [mcp](https://code.claude.com/docs/en/mcp) · [modelcontextprotocol.io](https://modelcontextprotocol.io)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>תחשוב על <strong>תוספים לדפדפן</strong> (extensions). ברירת המחדל של הדפדפן זה גלישה בסיסית - אבל אתה מוסיף תוסף ומקבל יכולות חדשות: מנהל סיסמאות, מתרגם, חוסם פרסומות.</p>

<p><strong>MCP זה אותו רעיון עבור Claude</strong>. ברירת המחדל: Claude קורא קבצים, מריץ פקודות, מחפש ברשת. מחבר אותו ל-MCP server של Slack - פתאום הוא יכול לשלוח הודעות. מחבר ל-GitHub - הוא רואה PRs ו-issues. מחבר ל-database שלך - הוא שולף נתונים.</p>

<p>המילה "MCP" מייצגת <strong>Model Context Protocol</strong> - סטנדרט פתוח לחיבורים האלה. יש מאות MCP servers מוכנים (Slack, Notion, Jira, Postgres, Playwright). אפשר גם לכתוב בעצמך.</p>

<p>הקובץ הזה מסביר איך מתקינים servers, איך מצמצמים את העלות בזיכרון, ואיך שומרים על אבטחה (ניהול OAuth, הגבלת הרשאות).</p>
</div>

---

## מה MCP נותן ל-Claude

שרת MCP חושף שלושה דברים:

1. **Tools** - functions ש-Claude יכול לקרוא. "send_slack_message", "query_database".
2. **Resources** - data ש-Claude יכול לקבל עם `@server:protocol://path`.
3. **Prompts** - slash commands ב-format `/mcp__server__prompt`.

---

## Transport types - איך השרת רץ

| Type | תיאור | מתי |
|------|-------|-----|
| `stdio` | subprocess מקומי, תקשורת דרך stdin/stdout | מקומי, trusted, ישן |
| `http` | HTTP POST עם streaming | remote, modern |
| `sse` | Server-Sent Events | remote, legacy |
| `ws` | WebSocket | remote, bidirectional |

---

## התקנה

### Remote HTTP

```bash
claude mcp add --transport http notion https://mcp.notion.com/mcp
claude mcp add --transport http secure-api https://api.example.com/mcp \
  --header "Authorization: Bearer $TOKEN"
```

### Remote SSE

```bash
claude mcp add --transport sse asana https://mcp.asana.com/sse
```

### Local stdio

```bash
claude mcp add [options] <name> -- <command> [args...]

claude mcp add --transport stdio --env AIRTABLE_API_KEY=YOUR_KEY airtable \
  -- npx -y @airtable/mcp-server
```

**Windows gotcha:** stdio דורש cmd wrapper:
```bash
claude mcp add --transport stdio myserver -- cmd /c npx -y @some/package
```

### JSON

```bash
claude mcp add-json weather '{"type":"http","url":"https://api.weather.com/mcp","headers":{"Authorization":"Bearer token"}}'
```

### ניהול

```bash
claude mcp list
claude mcp remove <name>
claude mcp reset-project-choices   # reset approvals for .mcp.json
```

---

## Scopes - שלושה

### Local (default)

`~/.claude.json` → ה-server זמין רק לך, רק בפרויקט הזה.

### Project

`.mcp.json` ב-project root. משותף בצוות דרך git:

```bash
claude mcp add --transport http paypal --scope project https://mcp.paypal.com/mcp
```

Format של `.mcp.json`:
```json
{
  "mcpServers": {
    "paypal": {
      "type": "http",
      "url": "https://mcp.paypal.com/mcp"
    }
  }
}
```

**Security:** Claude Code שואל אישור לפני שימוש בשרת מ-`.mcp.json`. reset דרך `claude mcp reset-project-choices`.

### User

כל הפרויקטים שלך:
```bash
claude mcp add --transport http hubspot --scope user https://mcp.hubspot.com/anthropic
```

### Precedence

Local > Project > User - ספציפי דורס כללי.

---

## Environment variable expansion

ב-`.mcp.json`:

```json
{
  "mcpServers": {
    "myserver": {
      "command": "${MY_CUSTOM_PATH:-/usr/local/bin}/my-command",
      "env": {
        "API_KEY": "${API_KEY}",
        "DB_PATH": "${DB_PATH:-/default/path}"
      }
    }
  }
}
```

- `${VAR}` - חובה
- `${VAR:-default}` - עם default
- Teams משתפים `.mcp.json` אחד, כל אחד עם env vars משלו.

---

## Tool naming: `mcp__<server>__<tool>`

Claude רואה tools בפורמט הזה. שימושי ל-matchers ב-hooks ו-permissions:

```json
{"matcher": "mcp__github__.*"}        // כל tool של github server
{"matcher": "mcp__.*__write.*"}       // tools שמכילים "write"
```

### MCP prompts כ-slash commands

```
/mcp__github__list_prs
/mcp__jira__create_issue "Bug" high
```

---

## Tool Search - חיסכון בקונטקסט

Default: **MCP tools נטענים deferred.** רק שמות בקונטקסט. סכמה מלאה נטענת כש-`ToolSearch` מגלה tool ספציפי.

```bash
ENABLE_TOOL_SEARCH=true       # הכל deferred (default)
ENABLE_TOOL_SEARCH=auto       # טען upfront אם < 10% של context, שאר deferred
ENABLE_TOOL_SEARCH=auto:5     # 5% במקום 10%
ENABLE_TOOL_SEARCH=false      # טען הכל תמיד (רק במיוחד)
```

**דרוש:** Sonnet 4+ או Opus 4+. Haiku **לא תומך**.

**Non-first-party ANTHROPIC_BASE_URL:** tool search **disabled by default** (רוב ה-proxies לא מעבירים `tool_reference`). `ENABLE_TOOL_SEARCH=true` להכריח.

### לכותבי MCP servers

אתה בונה שרת? הוסף `server instructions` ברור:
- מה קטגוריית המשימות
- מתי Claude צריך לחפש את ה-tools שלך
- יכולות מרכזיות

Truncation ב-2KB. front-load.

---

## Output limits

| Limit | Default | שליטה |
|-------|---------|--------|
| Warning threshold | 10,000 tokens | - |
| Max per call | 25,000 tokens | `MAX_MCP_OUTPUT_TOKENS` |
| Hard ceiling | 500,000 chars | - |

### `anthropic/maxResultSizeChars` - per-tool override

כ-MCP server author:

```json
{
  "name": "get_schema",
  "description": "Full DB schema",
  "_meta": {
    "anthropic/maxResultSizeChars": 200000
  }
}
```

ה-annotation דורסת את `MAX_MCP_OUTPUT_TOKENS` לtext. לא משפיע על image content.

**Large outputs שלא מסומנים** → persisted to disk, Claude מקבל file reference.

---

## OAuth & authentication

### Remote servers עם OAuth

Claude Code מטפל אוטומטית - מציג URL, אתה מתחבר בדפדפן, token נשמר.

### `claude mcp login` / `claude mcp logout` - auth ישירות מהshell

לפעמים שרת MCP מאבד את ה-auth שלו - ה-token פג, או נותקת. עד עכשיו היית צריך לפתוח את התפריט האינטראקטיבי `/mcp`, למצוא שם את השרת, ולהתחבר מחדש. מ-**v2.1.186 (22 ביוני 2026)** יש שתי פקודות CLI שעושות את זה ישר מה-shell, בלי לפתוח שום תפריט:

```bash
claude mcp login <name>     # מריץ את ה-OAuth flow לשרת מוגדר
claude mcp logout <name>    # מנקה את ה-credentials של אותו שרת
```

- `login` - מריץ את אותו OAuth flow הרגיל (URL בדפדפן, token נשמר) עבור שרת שכבר מוגדר אצלך, בלי לעבור דרך `/mcp`.
- `logout` - מוחק את ה-token השמור של אותו שרת. שימושי כשרוצים להתחבר מחדש עם חשבון אחר, או כששרת תקוע על credentials ישנים.

זה הפתרון המהיר כששרת MCP מאבד אימות באמצע העבודה.

### Fixed callback port

```bash
claude mcp add --transport http \
  --callback-port 8080 \
  myserver https://mcp.example.com/mcp
```

### Pre-configured credentials

```bash
claude mcp add-json my-server \
  '{"type":"http","url":"https://mcp.example.com/mcp","oauth":{"clientId":"xyz","callbackPort":8080}}' \
  --client-secret
```

או env var:
```bash
MCP_CLIENT_SECRET=your-secret claude mcp add --transport http ...
```

### Override metadata discovery

```json
{
  "mcpServers": {
    "myserver": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "oauth": {
        "authServerMetadataUrl": "https://auth.example.com/.well-known/oauth-authorization-server"
      }
    }
  }
}
```

### Dynamic headers

```json
{
  "mcpServers": {
    "myserver": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "X-Dynamic-Token": "${CUSTOM_TOKEN}"
      }
    }
  }
}
```

---

## Managed configuration (ארגוני)

### Option 1: Exclusive control

`managed-mcp.json` ב-managed settings dir → משתמשים לא יכולים להוסיף servers נוספים:

```json
{
  "mcpServers": {
    "company-api": {
      "type": "http",
      "url": "https://api.company.com/mcp"
    }
  }
}
```

### Option 2: Allow/deny lists

ב-managed settings:

```json
{
  "allowedMcpServers": [
    {"type": "url", "pattern": "https://*.company.com/*"},
    {"type": "command", "pattern": "npx @company/*"}
  ],
  "deniedMcpServers": [
    {"type": "url", "pattern": "https://sketchy.example.com/*"}
  ]
}
```

---

## MCP Resources - `@server:protocol://`

```
Can you analyze @github:issue://123 and suggest a fix?
Review the API docs at @docs:file://api/authentication
Compare @postgres:schema://users with @docs:file://database/user-model
```

- type `@` → autocomplete מציג resources מכל servers + קבצים מקומיים
- Fuzzy search
- Fetched אוטומטית, attached כ-attachments

---

## Elicitation - שרת מבקש input

שרת MCP יכול לבקש מידע בזמן ריצה. Claude Code מציג:

- **Form mode** - דיאלוג עם שדות שהשרת מגדיר
- **URL mode** - פתיחת דפדפן ל-auth, חזרה לCLI לאישור

Auto-respond: [`Elicitation` hook](#/deep/hooks).

---

## Claude Code as MCP server

`.mcp.json`:
```json
{
  "mcpServers": {
    "claude-code": {
      "type": "stdio",
      "command": "claude",
      "args": ["mcp", "serve"]
    }
  }
}
```

מאפשר clients אחרים (Claude Desktop, Cursor) לקרוא tools של Claude Code.

---

## Channels - push לתוך session

לא WebFetch, לא polling - שרת MCP יכול לשלוח הודעות ל-session פעיל.

Use cases: CI results, monitoring alerts, Slack messages.

ראה: [channels](#/deep/channels) (בהמשך)

---

## דוגמאות מעשיות

### Playwright browser automation

```bash
claude mcp add --transport stdio playwright -- npx -y @playwright/mcp@latest
```

### Sentry for errors

```bash
claude mcp add --transport http sentry https://mcp.sentry.dev/mcp
```

### GitHub

```bash
claude mcp add --transport http github https://api.githubcopilot.com/mcp/
```

### PostgreSQL

```bash
claude mcp add --transport stdio db -- npx -y @bytebase/dbhub \
  --dsn "postgresql://user:pass@host:5432/db"
```

### Import from Claude Desktop

```bash
claude mcp add-from-claude-desktop
```

---

## Security notes

1. **Project-scope servers require approval** - הפעלה ראשונה שואלת.
2. **OAuth tokens נשמרים** ב-Claude Code credential store.
3. **Plugins עם MCP:** חייבים להצהיר ב-`plugin.json`.
4. **Subagent-scoped MCP** - server מופיע רק ל-subagent, לא ל-main.
5. **Managed allowlists** - ארגון יכול לאכוף.

---

## Debugging

1. `/mcp` - רואה אילו servers מחוברים, כמה context כל אחד תופס, status.
2. `MCP_TIMEOUT=10000 claude` - timeout להתחברות (ms).
3. `claude mcp list` - רשימה מלאה ב-CLI.
4. Logs: `~/.claude/logs/` - טרנזקציות MCP ל-debug.

---

## המשך → [permissions.md](#/deep/permissions)
