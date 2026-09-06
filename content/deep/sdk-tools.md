# Custom Tools - כלים משלך

> כלים שאתה מגדיר בקוד שלך. SDK מריץ אותם in-process (לא subprocess, לא MCP server חיצוני). Claude רואה אותם כמו כל tool אחר.

**מקור:** [agent-sdk/custom-tools](https://code.claude.com/docs/en/agent-sdk/custom-tools)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>Claude מגיע עם <strong>כלים מובנים</strong>: קריאת קבצים, הרצת פקודות, חיפוש ברשת. אבל מה אם אתה רוצה שיוכל לעשות משהו שרק התוכנה שלך יודעת - לשלוף מ-DB ספציפי, לחשב משהו עסקי, לקרוא למערכת פנימית?</p>

<p><strong>כלים מותאמים אישית</strong>: אתה כותב פונקציה ב-Python או TypeScript - "get_customer(id)" - ו-Claude יכול לקרוא לה כמו שהוא קורא לכל כלי אחר.</p>

<p>למשל: תוכנת שירות לקוחות. אתה מגדיר כלי <code>lookup_order(order_id)</code> שמחובר ל-DB שלך. עכשיו כשלקוח שואל "איפה ההזמנה שלי", Claude יכול לקרוא לכלי הזה, לקבל את המידע האמיתי, ולענות - בלי שאתה כתבת <em>שום</em> לוגיקה של "מה הוא אמר, מה לעשות".</p>

<p>הקובץ מסביר איך להגדיר כלי, לטפל בשגיאות, ולהחזיר סוגי תוכן שונים (טקסט, תמונות, נתונים מובנים).</p>
</div>

---

## המבנה - 4 חלקים

כל tool מוגדר עם:

1. **Name** - identifier ייחודי
2. **Description** - Claude קורא כדי להחליט מתי לקרוא
3. **Input schema** - הארגומנטים
4. **Handler** - async function שרצה

---

## TypeScript - Zod schema

```typescript
import { tool, createSdkMcpServer } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

const getTemperature = tool(
  "get_temperature",
  "Get the current temperature at a location",
  {
    latitude: z.number().describe("Latitude coordinate"),
    longitude: z.number().describe("Longitude coordinate")
  },
  async (args) => {
    // args typed from schema: { latitude: number; longitude: number }
    const response = await fetch(
      `https://api.open-meteo.com/v1/forecast?latitude=${args.latitude}&longitude=${args.longitude}&current=temperature_2m`
    );
    const data: any = await response.json();

    return {
      content: [{ type: "text", text: `Temperature: ${data.current.temperature_2m}°F` }]
    };
  }
);

const weatherServer = createSdkMcpServer({
  name: "weather",
  version: "1.0.0",
  tools: [getTemperature]
});
```

**Optional parameters:** `.default()` ב-Zod:
```typescript
hours: z.number().int().min(1).max(24).default(12)
```

---

## Python - dict schema

```python
from claude_agent_sdk import tool, create_sdk_mcp_server
from typing import Any
import httpx

@tool(
    "get_temperature",
    "Get the current temperature at a location",
    {"latitude": float, "longitude": float}
)
async def get_temperature(args: dict[str, Any]) -> dict[str, Any]:
    async with httpx.AsyncClient() as client:
        response = await client.get(
            "https://api.open-meteo.com/v1/forecast",
            params={
                "latitude": args["latitude"],
                "longitude": args["longitude"],
                "current": "temperature_2m"
            }
        )
        data = response.json()
    return {
        "content": [{
            "type": "text",
            "text": f"Temperature: {data['current']['temperature_2m']}°F"
        }]
    }

weather_server = create_sdk_mcp_server(
    name="weather",
    version="1.0.0",
    tools=[get_temperature]
)
```

**Optional parameters:** ב-Python, ה-dict schema **מתייחס לכל שדה כחובה**. Workaround:
- השמט מה-schema
- הזכר ב-description
- `args.get("key", default)` ב-handler

**Enums ו-ranges ב-Python:** ה-dict לא תומך. השתמש ב-JSON Schema dict מלא:

```python
@tool(
    "convert_units",
    "Convert value between units",
    {
        "type": "object",
        "properties": {
            "unit_type": {
                "type": "string",
                "enum": ["length", "temperature", "weight"]
            },
            "value": {"type": "number"}
        },
        "required": ["unit_type", "value"]
    }
)
```

---

## Handler return value

Handler חייב להחזיר dict/object עם:

```typescript
{
  content: [              // required
    { type: "text", text: "..." },
    { type: "image", data: "base64...", mimeType: "image/png" },
    { type: "resource", resource: { uri: "...", text: "..." } }
  ],
  isError: false          // optional
}
```

Python - `is_error` (snake_case).

---

## Content blocks

### Text
```typescript
{ type: "text", text: "result" }
```

### Image
```typescript
{
  type: "image",
  data: base64String,                  // raw base64, NO "data:image/..." prefix
  mimeType: "image/png"                // required
}
```

Claude רואה את התמונה ויזואלית. יכול לתאר, לענות שאלות עליה.

### Resource

Resource = תוכן שמקושר עם URI. ה-URI הוא label, לא path אמיתי שה-SDK קורא.

```typescript
{
  type: "resource",
  resource: {
    uri: "file:///tmp/report.md",      // label
    mimeType: "text/markdown",
    text: "# Report\n..."              // or blob: base64
  }
}
```

Use case: generated artifact שChaude יכול להזכיר מאוחר יותר.

---

## Registering עם query()

```python
options = ClaudeAgentOptions(
    mcp_servers={"weather": weather_server},
    allowed_tools=["mcp__weather__get_temperature"]
)
```

### Tool naming format

`mcp__{server_name}__{tool_name}`

Server בשם `weather` עם tool `get_temperature` → `mcp__weather__get_temperature`.

**Wildcard:** `mcp__weather__*` - כל tools מ-weather server.

---

## `allowed_tools` vs `tools`

**שני layers שונים:**

| Option | Layer | מה עושה |
|--------|-------|---------|
| `tools: ["Read", "Grep"]` | **Availability** | רק אלה built-ins קיימים בקונטקסט |
| `tools: []` | Availability | אין built-ins. רק ה-MCP שלך. |
| `allowed_tools: [...]` | **Permission** | רצים בלי prompt. tools אחרים זמינים אבל עוברים permission flow |
| `disallowed_tools: [...]` | Permission | תמיד נדחה. tool עדיין בקונטקסט, Claude עשוי לנסות ולהיכשל |

**Rule of thumb:** להוציא built-in מהראות של Claude → שים ב-`tools` בלי הlist. להגביל permission → `allowed_tools` / `disallowed_tools`.

---

## Tool annotations

Metadata על התנהגות:

| Field | Default | משמעות |
|-------|---------|---------|
| `readOnlyHint` | `false` | לא משנה env. מאפשר parallel execution עם אחרים ש-read-only. |
| `destructiveHint` | `true` | עשוי לבצע שינויים הרסניים. אינפורמטיבי. |
| `idempotentHint` | `false` | קריאה חוזרת עם אותם args = אותה השפעה. |
| `openWorldHint` | `true` | פונה למערכות חיצוניות. |

```typescript
tool(
  "get_temperature",
  "...",
  { /* schema */ },
  async (args) => ({ /* ... */ }),
  { annotations: { readOnlyHint: true } }
);
```

Python:
```python
from claude_agent_sdk import tool, ToolAnnotations

@tool(
    "get_temperature",
    "...",
    {"latitude": float, "longitude": float},
    annotations=ToolAnnotations(readOnlyHint=True)
)
async def get_temperature(args):
    ...
```

**⚠️ Annotations = metadata, לא enforcement.** `readOnlyHint: true` לא מונע מ-handler לכתוב לדיסק. שמור על אמת.

**למה `readOnlyHint: true` חשוב:** Claude יכול לקרוא כמה tools עם `readOnlyHint: true` **במקביל** - משמעותית יותר מהיר.

---

## Error handling

### חשוב מאוד: return error vs throw

| מה קורה | תוצאה |
|---------|--------|
| Handler זורק exception | **agent loop עוצר**. Claude לא רואה. `query` נכשל. |
| Handler catches ומחזיר `isError: true` | agent loop **ממשיך**. Claude רואה את השגיאה, יכול לנסות אחרת. |

### Pattern נכון

```python
async def fetch_data(args):
    try:
        async with httpx.AsyncClient() as client:
            response = await client.get(args["endpoint"])
            if response.status_code != 200:
                return {
                    "content": [{
                        "type": "text",
                        "text": f"API error: {response.status_code} {response.reason_phrase}"
                    }],
                    "is_error": True
                }
            data = response.json()
            return {"content": [{"type": "text", "text": json.dumps(data)}]}
    except Exception as e:
        return {
            "content": [{"type": "text", "text": f"Failed: {str(e)}"}],
            "is_error": True
        }
```

---

## Scaling - tool search

אם יש לך **עשרות tools**, כולם נכנסים לקונטקסט בכל turn. עלויות גדלות.

פתרון: [tool search](https://code.claude.com/docs/en/agent-sdk/tool-search) - tools נטענים on demand. Claude מקבל רק שמות, טוען סכמות כשצריך.

```python
options = ClaudeAgentOptions(
    enable_tool_search="auto"    # threshold-based
    # או "true" - always defer
)
```

---

## In-process MCP vs external MCP

| in-process (custom tool) | external MCP server |
|--------------------------|----------------------|
| async function בקוד שלך | תהליך נפרד, stdio/http |
| latency כמעט אפס | network/IPC overhead |
| גישה מלאה ל-app state | מבודד |
| מתחיל עם ה-query | רץ נפרד |
| שימוש: business logic שלך | שימוש: Slack, GitHub, DB connectors |

שניהם יחד ב-`mcp_servers`:
```python
mcp_servers={
    "my_custom": weather_server,              # in-process
    "github": {                               # external
        "type": "http",
        "url": "https://api.githubcopilot.com/mcp/"
    }
}
```

---

## דוגמה מלאה - unit converter

```python
@tool(
    "convert_units",
    "Convert a value from one unit to another",
    {
        "type": "object",
        "properties": {
            "unit_type": {
                "type": "string",
                "enum": ["length", "temperature", "weight"]
            },
            "from_unit": {"type": "string"},
            "to_unit": {"type": "string"},
            "value": {"type": "number"}
        },
        "required": ["unit_type", "from_unit", "to_unit", "value"]
    }
)
async def convert_units(args):
    conversions = {
        "length": {
            "kilometers_to_miles": lambda v: v * 0.621371,
            "miles_to_kilometers": lambda v: v * 1.60934,
        },
        "temperature": {
            "celsius_to_fahrenheit": lambda v: v * 9/5 + 32,
            "fahrenheit_to_celsius": lambda v: (v - 32) * 5/9,
        }
    }
    key = f"{args['from_unit']}_to_{args['to_unit']}"
    fn = conversions.get(args["unit_type"], {}).get(key)

    if not fn:
        return {
            "content": [{"type": "text", "text": f"Unsupported: {args['from_unit']} → {args['to_unit']}"}],
            "is_error": True
        }

    result = fn(args["value"])
    return {
        "content": [{
            "type": "text",
            "text": f"{args['value']} {args['from_unit']} = {result:.4f} {args['to_unit']}"
        }]
    }
```

---

## המשך → [03_hooks_permissions.md](#/deep/sdk-hooks)
