# Enterprise - providers, deploy, monitoring

> Claude Code ב-organizations. Providers חלופיים, deployment patterns, admin tooling.

**מקורות:** [amazon-bedrock](https://code.claude.com/docs/en/amazon-bedrock) · [google-vertex-ai](https://code.claude.com/docs/en/google-vertex-ai) · [microsoft-foundry](https://code.claude.com/docs/en/microsoft-foundry) · [llm-gateway](https://code.claude.com/docs/en/llm-gateway) · [third-party-integrations](https://code.claude.com/docs/en/third-party-integrations) · [network-config](https://code.claude.com/docs/en/network-config) · [analytics](https://code.claude.com/docs/en/analytics) · [monitoring-usage](https://code.claude.com/docs/en/monitoring-usage) · [costs](https://code.claude.com/docs/en/costs)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>מעבר למפתח יחיד - <strong>חברות שלמות</strong> שרוצות להטמיע את Claude Code. הקובץ הזה על מה שחשוב להן:</p>

<p>☁️ <strong>ספקים חלופיים</strong>: AWS Bedrock, Google Vertex, Azure Foundry. החברה כבר משלמת ל-AWS? אפשר להריץ Claude דרך AWS ולקבל חשבון אחד. בנוסף - private endpoints, compliance מסוימים.</p>

<p>🔒 <strong>שליטה ריכוזית</strong>: IT מגדיר פעם אחת מדיניות (אילו servers מותרים, אילו פקודות חסומות, מי יכול להפעיל מה), והיא חלה על כל העובדים. אי אפשר לעקוף.</p>

<p>📊 <strong>ניטור</strong>: דשבורד שמראה כמה כל עובד משתמש, כמה זה עולה, אילו כלים הכי נפוצים. בנוסף - OpenTelemetry לגרפנה/דטהדוג.</p>

<p>🌐 <strong>רשת ארגונית</strong>: Proxies, certificates, mTLS - ה-"כאבי ראש" של IT בארגון גדול.</p>

<p>📜 <strong>Compliance</strong>: SOC 2, GDPR, HIPAA, FedRAMP - כל התקנים.</p>
</div>

---

## Providers - חלופות ל-Anthropic API

### Amazon Bedrock

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=us-west-2
# Optional: AWS_PROFILE, AWS_BEARER_TOKEN_BEDROCK
claude
```

**יתרונות:**
- Billing דרך AWS (existing contracts)
- Private endpoints / VPC
- FedRAMP compliance
- Same Claude model IDs with Bedrock prefix

**Model IDs:**
```
anthropic.claude-opus-4-8-v1:0
anthropic.claude-sonnet-4-6-v1:0
```

**IAM:**
- `bedrock:InvokeModel`
- `bedrock:InvokeModelWithResponseStream`
- Cross-region inference - model-specific

---

### Google Vertex AI

```bash
export CLAUDE_CODE_USE_VERTEX=1
export ANTHROPIC_VERTEX_PROJECT_ID=my-project
export CLOUD_ML_REGION=us-east5
claude
```

**יתרונות:**
- GCP billing
- Integration עם GCP services
- Private access via Service Connect

**Auth:** `gcloud auth application-default login`

---

### Microsoft Foundry (Azure AI)

```bash
export CLAUDE_CODE_USE_FOUNDRY=1
export AZURE_ENDPOINT=https://...
claude
```

Same model access, Azure billing.

---

### LLM Gateway

Custom proxy / gateway:

```bash
export ANTHROPIC_BASE_URL=https://gateway.company.com
export ANTHROPIC_API_KEY=gateway-key
claude
```

**Requirements:**
- Must forward `tool_reference` blocks (אחרת tool search disabled)
- Must support streaming
- Must support caching headers for cost efficiency

Popular gateways: Portkey, Helicone, Litellm, custom.

**Feature support:**
- `ENABLE_TOOL_SEARCH=true` explicit אם gateway תומך
- Otherwise tool search disabled (לדאוג לcontext)

---

## Network configuration

### Proxy

```json
// settings.json
{
  "network": {
    "proxy": "http://proxy.company.com:8080",
    "noProxy": ["*.internal", "localhost"],
    "caBundle": "/etc/ssl/company-ca.crt"
  }
}
```

Env vars:
```bash
export HTTP_PROXY=http://proxy:8080
export HTTPS_PROXY=http://proxy:8080
export NO_PROXY=*.internal,localhost
export NODE_EXTRA_CA_CERTS=/etc/ssl/ca.crt   # for Node SDK
```

### mTLS

```json
{
  "network": {
    "mtls": {
      "cert": "/etc/ssl/client.crt",
      "key": "/etc/ssl/client.key",
      "ca": "/etc/ssl/ca.crt"
    }
  }
}
```

---

## Managed deployment

### MDM (macOS, Windows)

Managed settings deployed ל-endpoints:

- macOS: `/Library/Application Support/ClaudeCode/managed-settings.json`
- Windows: `C:\ProgramData\ClaudeCode\managed-settings.json`
- Linux: `/etc/claude-code/managed-settings.json`

Deploy via:
- Jamf / Intune / Kandji
- Ansible / Puppet / Chef
- Group Policy

### Server-managed settings

Pulled from Anthropic backend (Team/Enterprise):

```json
{
  "forceRemoteSettingsRefresh": true
}
```

Admin changes settings ב-admin dashboard → auto-applied ל-endpoints on next startup.

---

## Analytics dashboard

```
claude.ai/admin-settings/claude-code/analytics
```

Team / Enterprise plans:

- Per-user usage
- Tool call distribution
- Cost per project / user
- Error rates
- Adoption metrics
- Engineering velocity proxy

---

## OpenTelemetry monitoring

```json
{
  "otel": {
    "endpoint": "http://otel-collector:4317",
    "serviceName": "claude-code",
    "headers": {
      "Authorization": "Bearer ${OTEL_TOKEN}"
    }
  }
}
```

Exports:
- **Traces** - session events, tool calls, subagent spawns
- **Metrics** - tokens/sec, duration, errors
- **Logs** - structured events

Compatible: Grafana, Datadog, Jaeger, Honeycomb, New Relic.

---

## Cost management

### Team spend caps

```json
{
  "costs": {
    "maxBudgetUSD": 100,
    "budgetPeriod": "monthly",
    "alertThresholds": [50, 80, 100]
  }
}
```

Per-user default via managed settings.

### Per-session cap (SDK)

```python
ClaudeAgentOptions(max_budget_usd=2.00)
```

### Context hygiene for cost

- **Prompt caching** - identical startup context → 5-10x cheaper after first run
- **Subagents** - research work out of main context
- **Model selection** - sonnet for most, opus for hard cases
- **Haiku** ל-pre-filters
- **`--exclude-dynamic-system-prompt-sections`** ב-CI → cache reuse across jobs

### Cost tracking per team

Analytics dashboard + OTel metrics → attribute spend to:
- User
- Project
- Tool (Bash vs Read vs WebFetch)
- Model

---

## Third-party integrations summary

| Integration | Type | Use case |
|-------------|------|----------|
| GitHub Actions | CI/CD | Auto-review PRs, triage issues |
| GitHub Enterprise Server | Self-hosted Git | Same as GitHub |
| GitLab CI/CD | CI/CD | MR review, issue triage |
| Slack | Chat | @Claude in channels |
| Chrome | Browser | Debug web apps |
| Code review (auto) | CI bot | Multi-agent PR analysis |
| Jira / Linear | Issues | Via MCP servers |
| Notion / Confluence | Docs | Via MCP |

---

## Devcontainers

Team-wide consistent dev environment:

`.devcontainer/devcontainer.json`:
```json
{
  "image": "anthropic/claude-code:latest",
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },
  "mounts": [
    "source=${localEnv:HOME}/.claude,target=/home/vscode/.claude,type=bind"
  ],
  "customizations": {
    "vscode": {
      "extensions": ["anthropic.claude-code"]
    }
  },
  "postCreateCommand": "claude auth login"
}
```

---

## Compliance

| Standard | Coverage |
|----------|----------|
| SOC 2 Type II | ✅ |
| GDPR | ✅ |
| CCPA | ✅ |
| HIPAA | ✅ (Enterprise + BAA) |
| FedRAMP | ✅ (via AWS Bedrock) |
| ISO 27001 | ✅ |

Data handling:
- US-based processing default
- EU region available (Enterprise)
- ZDR (Zero Data Retention) per-org option

---

## Audit logging

### Per-session audit

Hook `PreToolUse` + `PostToolUse` → log every action:

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "scripts/audit.sh",
        "async": true
      }]
    }]
  }
}
```

### Organization-wide

Managed settings enforce hook:
```json
{
  "allowManagedHooksOnly": true,
  "hooks": {
    "PreToolUse": [{
      "matcher": "*",
      "hooks": [{"type": "http", "url": "https://audit.company.com/log"}]
    }]
  }
}
```

Managed hooks - users cannot disable.

---

## Security hardening checklist (enterprise)

- [ ] Managed settings deployed via MDM
- [ ] `allowManagedPermissionRulesOnly: true`
- [ ] `allowManagedMcpServersOnly: true`
- [ ] `allowManagedHooksOnly: true`
- [ ] `disableBypassPermissionsMode: "disable"`
- [ ] `disableAutoMode: "disable"` (עד שroll out)
- [ ] `forceLoginMethod` + `forceLoginOrgUUID`
- [ ] Sandbox enabled with tight allowlist
- [ ] OTel audit logging active
- [ ] `blockedMarketplaces` + `strictKnownMarketplaces`
- [ ] `allowedChannelPlugins` limited
- [ ] CA / proxy configured
- [ ] ZDR enabled אם compliance דורש
- [ ] Cost caps per user
- [ ] Analytics monitored
- [ ] Onboarding documentation

---

## חזור ל-[INDEX.md](#/deep)
