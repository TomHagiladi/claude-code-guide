# Settings & Environment Variables

> Reference לכל ההגדרות ב-settings.json ו-env vars. 150+ keys.

**מקורות:** [settings](https://code.claude.com/docs/en/settings) · [env-vars](https://code.claude.com/docs/en/env-vars) · [server-managed-settings](https://code.claude.com/docs/en/server-managed-settings)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>Claude Code הוא אפליקציה עם <strong>המון הגדרות</strong> - כמו Photoshop או VS Code. איזה מודל להשתמש בו ברירת מחדל, איזה צבע, האם להפעיל memory אוטומטי, אילו פקודות תמיד מותרות, ועוד 150+ הגדרות.</p>

<p>ההגדרות חיות ב<strong>קבצי JSON</strong> ברמות שונות: אישיות (שלך, לכל הפרויקטים), פרויקט (משותפות בצוות דרך Git), מקומי (אישי לפרויקט הזה), ומנוהל (מנהל הארגון קובע, אף אחד לא יכול לעקוף).</p>

<p>בנוסף - <strong>משתני סביבה</strong> (environment variables). אלה הגדרות שאתה מגדיר במערכת ההפעלה (<code>export ANTHROPIC_API_KEY=...</code>). שימושי בעיקר ל<strong>סודות</strong>: אל תשים API key בקובץ - שים במשתנה סביבה.</p>

<p>הקובץ הזה רפרנס: רשימה של כל ההגדרות ו-env vars, מה כל אחד עושה, איפה לשים מה.</p>
</div>

---

## Settings files - hierarchy

| Location | Scope |
|----------|-------|
| `~/.claude/settings.json` | User (all projects) |
| `./.claude/settings.json` | Project (shared, in git) |
| `./.claude/settings.local.json` | Local project (gitignored) |
| Managed policy | Organization (cannot override) |

**Precedence:** Managed > CLI args > Local > Shared project > User.

### Managed policy locations

- macOS: `/Library/Application Support/ClaudeCode/managed-settings.json`
- Linux/WSL: `/etc/claude-code/managed-settings.json`
- Windows: `C:\ProgramData\ClaudeCode\managed-settings.json`

### Server-managed settings

Pulled dynamically from backend. See [server-managed-settings](https://code.claude.com/docs/en/server-managed-settings).

---

## Core settings

### Model & behavior

```json
{
  "model": "sonnet",
  "fastMode": false,
  "effort": "medium",
  "autoCompactThreshold": 0.85,
  "disableAutoCompact": false,
  "theme": "dark",
  "alwaysAllowReadonly": true
}
```

### Memory

```json
{
  "autoMemoryEnabled": true,
  "autoMemoryDirectory": "~/custom-memory",
  "claudeMdExcludes": ["**/monorepo/other-team/**"]
}
```

### Skills / Commands

```json
{
  "disableSkillShellExecution": false,
  "skillSources": ["project", "user"],
  "disableBundledSkills": false
}
```

`disableBundledSkills` (v2.1.169, 8.6.2026) - מסתיר מהמודל את ה-skills, ה-workflows וה-slash commands המובְנים של Claude Code. שימושי ל-troubleshooting, כדי שמובְנים לא "יאפילו" (shadow) על ה-skills שלך, או כדי לחסוך tokens בהפעלה (startup). אפשר גם דרך env var: `CLAUDE_CODE_DISABLE_BUNDLED_SKILLS`.

### Hooks - ראה [hooks.md](#/deep/hooks)

```json
{
  "hooks": { "PreToolUse": [...], "PostToolUse": [...] },
  "disableAllHooks": false
}
```

### MCP

```json
{
  "mcpServers": {...},
  "strictMcpConfig": false,
  "allowedMcpServers": [...],      // managed
  "deniedMcpServers": [...],
  "allowManagedMcpServersOnly": false
}
```

---

## Permissions (ראה [permissions.md](#/deep/permissions))

```json
{
  "permissions": {
    "defaultMode": "default",
    "allow": ["Bash(npm *)"],
    "deny": ["Read(**/*.env)"],
    "ask": ["Bash(git push *)"],
    "additionalDirectories": ["../shared"],
    "disableBypassPermissionsMode": "disable",
    "disableAutoMode": "disable"
  }
}
```

### Auto mode

```json
{
  "autoMode": {
    "environment": ["Source control: github.example.com/acme"],
    "allow": ["..."],
    "soft_deny": ["..."],
    "classifyAllShell": true
  }
}
```

`classifyAllShell` (v2.1.193, 25.6.2026) - מנתב **כל** פקודת Bash ו-PowerShell דרך ה-Auto-mode safety classifier. ברירת המחדל בודקת רק פקודות שנראות חשודות; עם `true` הכול עובר בדיקה. מאותה גרסה: סיבות הדחייה (denial reasons) של Auto-mode מופיעות עכשיו ב-transcript, ב-toast של הדחייה, וגם ב-`/permissions`.

---

## Sandbox

```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "filesystem": {
      "allowRead": ["/workspace"],
      "allowWrite": ["/workspace"],
      "denyRead": ["/secrets"],
      "denyWrite": ["/etc"],
      "allowManagedReadPathsOnly": false
    },
    "network": {
      "allowedDomains": ["*.api.company.com"],
      "allowManagedDomainsOnly": false
    }
  }
}
```

ראה [sandboxing](https://code.claude.com/docs/en/sandboxing).

---

## Plugins

```json
{
  "enabledPlugins": ["marketplace:plugin-name"],
  "extraKnownMarketplaces": [{"type": "github", "repo": "org/repo"}],
  "strictKnownMarketplaces": false,
  "blockedMarketplaces": [],
  "pluginTrustMessage": "Custom warning text"
}
```

---

## Channels (Team/Enterprise)

```json
{
  "channelsEnabled": false,
  "allowedChannelPlugins": ["telegram", "slack"]
}
```

---

## Environment

```json
{
  "env": {
    "API_KEY": "${VAULT_API_KEY}",
    "REGION": "us-east-1"
  }
}
```

Claude Code sets these for subprocesses (Bash, hooks).

---

## Authentication

```json
{
  "forceLoginMethod": "claude_ai",     // or "api_key", "bedrock", "vertex", "foundry"
  "forceLoginOrgUUID": "...",
  "apiKeyHelper": "/path/to/script-that-prints-key",
  "anthropicApiKey": "...",              // discouraged; use env
  "awsBearerToken": "...",
  "bedrockConfiguration": {...}
}
```

---

## IDE & interface

```json
{
  "statusLine": {...},
  "subagentStatusLine": {...},
  "fullscreen": false,
  "voiceEnabled": false,
  "editorCommand": "code",
  "terminalNotifications": true,
  "respondToBashCommands": true
}
```

`respondToBashCommands` (v2.1.186, 22.6.2026) - שולט במצב shell. פקודות שמריצים ב-shell mode עם הקידומת `!` (למשל `! npm test`) מפעילות עכשיו אוטומטית תגובה של Claude ברגע שמופיע פלט - כך שכישלון בטסט מקבל הסבר מיד, בלי לבקש שוב. כדי לכבות את ההתנהגות הזו, הגדר את `respondToBashCommands` ל-false. בנוסף, מצב Bash קיבל השלמה אוטומטית חיה של נתיבי קבצים (live file-path autocomplete).

---

## Managed-only settings

מתעלמים כשמוגדרים בuser/project:

| Setting |
|---------|
| `allowManagedPermissionRulesOnly` |
| `allowManagedMcpServersOnly` |
| `allowManagedHooksOnly` |
| `allowedChannelPlugins` |
| `channelsEnabled` |
| `forceRemoteSettingsRefresh` |
| `pluginTrustMessage` |
| `blockedMarketplaces` |
| `strictKnownMarketplaces` |
| `sandbox.filesystem.allowManagedReadPathsOnly` |
| `sandbox.network.allowManagedDomainsOnly` |

---

## Environment Variables - רשימה מרכזית

### Auth

| Var | משמעות |
|-----|---------|
| `ANTHROPIC_API_KEY` | API key |
| `ANTHROPIC_AUTH_TOKEN` | bearer (claude.ai) |
| `ANTHROPIC_BASE_URL` | proxy / alt endpoint |
| `CLAUDE_CODE_USE_BEDROCK=1` | AWS |
| `CLAUDE_CODE_USE_VERTEX=1` | GCP |
| `CLAUDE_CODE_USE_FOUNDRY=1` | Azure |
| `AWS_REGION`, `AWS_PROFILE`, `VERTEX_REGION`, etc. | provider-specific |

### Memory

| Var | משמעות |
|-----|---------|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` | disable |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` | load CLAUDE.md from --add-dir |

### MCP

| Var | משמעות |
|-----|---------|
| `ENABLE_TOOL_SEARCH` | `true`, `auto`, `auto:N`, `false` |
| `MAX_MCP_OUTPUT_TOKENS` | default 25000 |
| `MCP_TIMEOUT` | connect timeout (ms) |
| `MCP_CLIENT_SECRET` | OAuth secret |

### Behavior

| Var | משמעות |
|-----|---------|
| `CLAUDE_CODE_SUBAGENT_MODEL` | override subagent model |
| `CLAUDE_CODE_SIMPLE=1` | bare mode |
| `CLAUDE_CODE_NEW_INIT=1` | interactive /init |
| `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` | PowerShell (Windows) |
| `CLAUDE_CODE_DISABLE_BUNDLED_SKILLS=1` | hide built-in skills/workflows/commands (v2.1.169) |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | response cap |
| `CLAUDE_CODE_THINK_BUDGET` | thinking tokens |
| `SLASH_COMMAND_TOOL_CHAR_BUDGET` | skill description cap |

### Debug

| Var | משמעות |
|-----|---------|
| `CLAUDE_CODE_DEBUG_LOGS_DIR` | log directory |
| `DEBUG=1` / `CLAUDE_DEBUG=1` | generic debug |

### Runtime context (available in hooks/scripts)

| Var | משמעות |
|-----|---------|
| `CLAUDE_PROJECT_DIR` | project root |
| `CLAUDE_PLUGIN_ROOT` | plugin install dir |
| `CLAUDE_PLUGIN_DATA` | plugin data dir |
| `CLAUDE_ENV_FILE` | env persistence file (SessionStart, CwdChanged, FileChanged hooks) |
| `CLAUDE_CODE_REMOTE=true` | web/cloud environment |
| `CLAUDE_SESSION_ID` | session ID |
| `CLAUDE_SKILL_DIR` | SKILL.md directory (in skills) |

### Remote Control

| Var | משמעות |
|-----|---------|
| `CLAUDE_REMOTE_CONTROL_SESSION_NAME_PREFIX` | default prefix |

---

## Best practices

1. **User settings ל-preferences.** Theme, default model, common allow rules.
2. **Project settings ל-team standards.** Shared permissions, hooks, MCP servers.
3. **Local settings ל-personal dev env.** `claudeMdExcludes`, local MCP secrets.
4. **Managed ל-ארגוני.** Policy שחייב לא לעקוף.
5. **env vars ל-secrets.** לא ב-settings (גם לא local - עלול להתגלגל).
6. **`${VAR_NAME}` expansion** ב-settings.json - שמור secrets בenv.

---

## Validation

```bash
claude auto-mode config   # effective auto-mode rules
claude doctor             # health check
/permissions              # active permission rules
/hooks                    # active hooks
/mcp                      # MCP servers status
```

---

## המשך → [sandboxing_security.md](#/deep/sandbox) · [troubleshooting.md](#/deep/troubleshooting)
