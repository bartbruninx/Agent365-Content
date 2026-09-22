# Agent Telemetry KQL Guide

A short guide to understanding agent telemetry and investigating **what an agent did, which tools it used, which accounts were associated, and what deserves a closer look**.

Examples only use native KQL for **Defender XDR Advanced Hunting**.

## 1. Understand the data

| Table | Question it answers | Important limitation |
|---|---|---|
| `CloudAppEvents` | What activity was observed? | Also contains non-agent activity. Filter by time and agent action types. |
| `AgentsInfo` | What inventory/configuration metadata is reported? | Can be incomplete, outdated, or a related blueprint rather than the executing instance. |

Activity and inventory are **different evidence sources**. An agent can have activity without an inventory match, or inventory without recent activity.

### The shape of an activity event

```text
CloudAppEvents
  Timestamp, ActionType, Application
  AccountObjectId, AccountId, AccountDisplayName, AccountType
  IPAddress, CountryCode, IsExternalUser, IsAnonymousProxy, IsImpersonated
  RawEventData
    AgentId / TargetAgentId
    ToolName, ToolType, ToolServerName, ServerAddress
    TraceId, ConversationId, OpId, ParentId, ErrorMessage ...
    CopilotEventData
      AgentId, TraceId, ConversationId ...
      ModelTransparencyDetails
        ModelName, ModelProviderName
  AdditionalFields
    IsSatelliteProvider ... when reported
```

`RawEventData` contains most of the useful agent-specific detail we observed. Some inference fields are nested inside `CopilotEventData`; tool and output fields can be at the raw root. In our exports, `AdditionalFields` was mostly empty or contained `IsSatelliteProvider`.

These are **observed paths**, not a guarantee that every emitter populates every field.

### The main event families

| `ActionType` | How to read it |
|---|---|
| `InvokeAgent` | An agent invocation. For the agent being invoked, use **`TargetAgentId`**, not the caller's ID. |
| `InferenceCall` | A reported inference event. Model/provider metadata may be nested or missing. |
| `ExecuteToolBySDK` | SDK-reported tool activity; can include MCP initialization and discovery. |
| `ExecuteToolByGateway` | Gateway-reported tool activity. |
| `ExecuteToolByMCPServer` | MCP-server-reported tool activity, when available. |
| `AISpanOutput` | An output span observed in our data; not proof that output reached a user. |

**One operation can appear at several reporting layers. Event counts are not unique calls or successful operations.**

## 2. The fields worth knowing

Here, `R` means `RawEventData`, and `C` means its parsed `CopilotEventData` object.

| Purpose | Useful fields | What to watch for |
|---|---|---|
| Agent identity | `R.TargetAgentId` on invocation; otherwise `R.AgentId`, then `C.AgentId` | Caller and target are not interchangeable. |
| Name/platform | `R.TargetAgentName`, `R.PlatformTargetAgentType`; otherwise `R.AgentName`, `R.PlatformAgentType`, nested equivalents | A name is not an identity key. Multiple platform labels do not prove a collision. |
| Platform-native identity | `R.PlatformTargetAgentId` on invocation; otherwise `R.PlatformAgentId`, `C.PlatformAgentId` | Keep separate from the executing ID. Native IDs need platform context for inventory matching. |
| Blueprint relationship | `R.TargetAgentBlueprintId` / `R.TargetAgentBlueprintID`, `R.AgentBlueprintId` | A blueprint is not the executing instance. JSON key casing matters. |
| Normalized account | `AccountObjectId`, `AccountId`, `AccountDisplayName`, `AccountType` | Missing object ID does not mean anonymous. An "admin" name does not establish privileges. |
| Other identity roles | `R.UserId`, `R.UserKey`; `R.TargetAgentUserId`, `R.TargetAgentUserKey` | Raw reported user and target agent user are separate roles, not automatic aliases. |
| Tools/servers | `R.ToolName`, `R.ToolType`, `R.ToolServerName`, `R.ServerAddress`, `R.ToolDescription` | A reported address is not proof of a network connection. A tool name does not prove data retrieval. |
| Correlation | `R.TraceId` / `C.TraceId`, `ConversationId`, `ThreadId`, `RequestId`, `R.OpId`, `R.ParentId` | Preserve each ID separately. `OpId`/`ParentId` are useful span pivots; do not assume complete traces. |
| Errors | `R.ErrorMessage`, `R.ErrorCode`, `R.ErrorType`, nested error fields | Error-bearing events can describe the same failure at different layers. |
| Models | `C.ModelTransparencyDetails[].ModelName`, `.ModelProviderName` | Observed in some exports, missing in others. Do not infer model identity from an agent name. |
| Timing | `R.CreationTime`, `R.CompletionTime`, `C.CompletionTime` | Only calculate duration when both timestamps exist and end is not before start. Missing is not zero. |
| Review signals | `AccountType`, `IsExternalUser`, `IsAnonymousProxy`, `IsImpersonated`, `IPAddress`, `CountryCode` | Investigation context, not proof of compromise. |

## 3. Before you start

- Start with **24 hours**, then widen deliberately.
- Check the schema once if fields do not match. Run each of these separately:

```kusto
CloudAppEvents
| getschema
```

```kusto
AgentsInfo
| getschema
```

## 4. Find active agents

**Run this query on its own.** It lists observed executing/target IDs, with names for context.

```kusto
CloudAppEvents
| where Timestamp > ago(24h)
| where ActionType in ("InvokeAgent", "InferenceCall", "AISpanOutput",
    "ExecuteToolBySDK", "ExecuteToolByGateway", "ExecuteToolByMCPServer")
| extend R = RawEventData, C = parse_json(tostring(RawEventData.CopilotEventData))
| extend AgentId = iff(ActionType == "InvokeAgent", tostring(R.TargetAgentId),
        coalesce(tostring(R.AgentId), tostring(C.AgentId))),
    AgentName = iff(ActionType == "InvokeAgent", tostring(R.TargetAgentName),
        coalesce(tostring(R.AgentName), tostring(C.AgentName)))
| where isnotempty(AgentId)
| summarize Events = count(), LastSeen = max(Timestamp),
    Names = make_set_if(AgentName, isnotempty(AgentName), 10) by AgentId
| top 50 by LastSeen desc
```

This is a top-50 activity list, not a complete inventory. Events with only a native platform ID will not appear here; inspect their raw fields if needed.

## 5. A small starting block for one agent

For each example **A-H**, paste this block first, replace the ID, and append **one** example. Run the combined text together: the `Events` variable does not persist between queries.

**Do not run this starting block alone.** It only defines variables; it does not return a table. Running just the `let` statements produces **"No tabular expression statement found."** For a quick check, append `Events | top 100 by Timestamp desc` after the final semicolon. For an investigation, append one of A-H instead. Select and run the whole combined query, including the `let` statements.

<!-- kql:agent-base -->
```kusto
let agentId = "PASTE-AGENT-ID-HERE";
let Events =
    CloudAppEvents
    | where Timestamp > ago(24h)
    | where isnotempty(agentId)
    | where ActionType in ("InvokeAgent", "InferenceCall", "AISpanOutput",
        "ExecuteToolBySDK", "ExecuteToolByGateway", "ExecuteToolByMCPServer")
    | extend R = RawEventData, C = parse_json(tostring(RawEventData.CopilotEventData))
    | extend EventAgentId = iff(ActionType == "InvokeAgent", tostring(R.TargetAgentId),
        coalesce(tostring(R.AgentId), tostring(C.AgentId)))
    | where EventAgentId == agentId;
```

Keep it simple: copy the exact ID, including casing, from the discovery query. This block uses exact string matching and intentionally does not merge names, native IDs, or blueprint IDs. For a native-ID investigation, extract `PlatformTargetAgentId` on invocation and `PlatformAgentId` on other events, and filter the platform as well.

### A. What happened most recently?

Start here when you do not yet understand an agent's payload.

```kusto
Events
| top 100 by Timestamp desc
| project Timestamp, ActionType, EventAgentId, Application,
    AccountDisplayName, AccountId, AccountType,
    RawEventData, AdditionalFields
```

Expand the JSON before assuming a field is absent. This is the latest 100 events, not a full history.

### B. When was it active, and doing what?

```kusto
Events
| summarize Events = count() by bin(Timestamp, 1h), ActionType
| order by Timestamp asc
| render timechart
```

Useful for bursts, changes in activity mix, or unexpected activity windows. A spike needs operational context; it is not automatically suspicious.

### C. Which tools and servers were reported?

```kusto
Events
| where ActionType startswith "ExecuteTool"
| extend Tool = coalesce(tostring(R.ToolName), tostring(R.toolName),
        tostring(R["gen_ai.tool.name"])),
    ToolType = tostring(R.ToolType),
    Server = tostring(R.ToolServerName), Address = tostring(R.ServerAddress)
| summarize Events = count(), LastSeen = max(Timestamp)
    by Tool, ActionType, ToolType, Server, Address
| top 50 by Events desc
```

Keep `ActionType` and `ToolType` visible. `MCP:Initialize*` and `MCP:ListTools*` are setup/discovery; `MCP:CallTool*` is an explicit call marker. `function`, `RemoteMCP`, and `CodefulServer` alone do not establish a unique successful execution.

Each row groups fields reported **on the same events**. Do not build relationships by combining unrelated "latest values" from different events.

### D. Which accounts were associated?

```kusto
Events
| top 100 by Timestamp desc
| project Timestamp, ActionType,
    AccountDisplayName, AccountObjectId, AccountId, AccountType,
    RawUserId = tostring(R.UserId), RawUserKey = tostring(R.UserKey),
    TargetAgentUserId = tostring(R.TargetAgentUserId),
    TargetAgentUserKey = tostring(R.TargetAgentUserKey)
```

Read across each event, keeping the three roles separate: **normalized account**, **raw reported user**, and **target agent user**. Service identities and notification accounts can legitimately appear.

### E. Which events deserve an identity/network review?

```kusto
Events
| where AccountType =~ "Admin"
    or IsExternalUser == true
    or IsAnonymousProxy == true
    or IsImpersonated == true
| top 100 by Timestamp desc
| project Timestamp, ActionType, AccountDisplayName, AccountId, AccountType,
    IsExternalUser, IsAnonymousProxy, IsImpersonated,
    IPAddress, CountryCode, Tool = tostring(R.ToolName)
```

Use this as a **review queue**, not an alert verdict. Verify whether the identity, tool, location, and timing fit the expected workflow. `AccountType` is not an effective-permissions assessment.

### F. Which errors keep appearing?

```kusto
Events
| extend Tool = coalesce(tostring(R.ToolName), tostring(R.toolName),
        tostring(R["gen_ai.tool.name"])),
    Error = coalesce(tostring(R.ErrorMessage), tostring(C.ErrorMessage)),
    Code = coalesce(tostring(R.ErrorCode), tostring(R.ErrorType), tostring(C.ErrorType))
| where isnotempty(Error) or isnotempty(Code)
| summarize ErrorEvents = count(), LastSeen = max(Timestamp)
    by ActionType, Tool, Code, Error
| top 30 by ErrorEvents desc
```

Repeated mail/search failures, for example, may indicate a tool, permissions, or configuration problem rather than an attack. No error field does not prove success.

### G. What model/provider metadata is available?

```kusto
Events
| where ActionType == "InferenceCall"
| extend Models = parse_json(tostring(C.ModelTransparencyDetails))
| mv-expand ModelEntry = Models
| extend Model = tostring(ModelEntry.ModelName),
    Provider = tostring(ModelEntry.ModelProviderName)
| where isnotempty(Model) or isnotempty(Provider)
| summarize MetadataEntries = count(), LastSeen = max(Timestamp) by Model, Provider
| order by MetadataEntries desc
```

This small query targets the **array shape observed in our inference exports**. Inspect raw JSON if your emitter uses another shape. It shows reported names only; no rows means no extracted named metadata, not no inference activity. Multiple model entries in one event can contribute multiple rows.

### H. Follow a reported trace

Replace the trace ID after finding it in raw evidence. Keep the starting block above this example.

```kusto
let traceId = "PASTE-TRACE-ID-HERE";
Events
| extend Trace = coalesce(tostring(R.TraceId), tostring(C.TraceId), tostring(R.traceId))
| where isnotempty(traceId) and Trace == traceId
| project Timestamp, ActionType,
    Tool = tostring(R.ToolName),
    ConversationId = coalesce(tostring(R.ConversationId), tostring(C.ConversationId)),
    RequestId = coalesce(tostring(R.RequestId), tostring(C.RequestId)),
    SpanId = tostring(R.OpId), ParentSpanId = tostring(R.ParentId),
    RawEventData
| order by Timestamp asc
| take 200
```

This shows the earliest 200 matching events **for the selected agent and window**, not a guaranteed complete cross-agent run. We observed both traces spanning several conversation IDs and agents with no extracted trace IDs. If trace data is missing, pivot on a reported conversation or request ID, but do not equate that with a complete trace.

## 6. Check inventory without overcomplicating the match

**Run this query on its own.** Start with an exact inventory `AgentId` lookup. It keeps the latest snapshot per ID/platform in the last 30 days and returns the full row so you can expand metadata.

```kusto
let inventoryId = "PASTE-INVENTORY-AGENT-ID-HERE";
AgentsInfo
| where Timestamp > ago(30d)
| where isnotempty(inventoryId) and tostring(AgentId) == inventoryId
| summarize arg_max(Timestamp, *) by AgentId, Platform
```

If the activity ID returns no inventory row, that is **not proof the agent is unmanaged**. Check the identity fields and schema:

| Inventory fields | What to examine |
|---|---|
| `AgentId`, `Platform` | The inventory record's identity and source platform. |
| `AgentName` / `Name`, `AgentDescription` / `Description` | Friendly metadata; our live schema differed from the reference names. |
| `EntraAgentId` / `EntraAgentID` | A possible instance identity match; compare explicitly with the activity ID. |
| `SourceAgentId` | A native ID; match it with platform context, not ID alone. |
| `EntraBlueprintId` / `EntraBlueprintID` | A related template identity. Do not treat it as the runtime instance. |
| `ObservabilityId` / `ObservabilityID` | Inspect the value/type before using it as a correlation key. |
| `Owners`, `Permissions`, `SharedWith`, `ToolsAuthenticationType` | Reported ownership/access metadata; not a live authorization test. |
| `DeclaredTools`, `McpServers`, `Endpoints`, `DeclaredDataSources` | Compare declared configuration with observed activity. |
| `Model`, `Instructions`, `Guardrails`, `RawAgentInfo` | Reported configuration and additional raw details. |

Our workbook investigation found a **related blueprint-only record without an instance match**. Its publication status and sharing settings could not safely be applied to the executing agent. Likewise, empty permissions, owners, or guardrails mean **not reported here**, not necessarily absent.

## 7. A simple investigation order

1. **Discover the ID** and confirm the time window.
2. **Read a few raw events** to understand the actual field layout.
3. **Review activity and tools**, preserving reporting layers.
4. **Check associated accounts, review signals, and recurring errors.**
5. **Follow correlation IDs**, where reported.
6. **Compare inventory separately**, distinguishing instance identity from blueprint relationships.

If an investigation starts from an alert, also inspect `AlertInfo` and `AlertEvidence` using that alert's actual identifiers. Do not invent a join between agent IDs and alert entities.

## References

- [CloudAppEvents schema](https://learn.microsoft.com/defender-xdr/advanced-hunting-cloudappevents-table)
- [AgentsInfo schema](https://learn.microsoft.com/defender-xdr/advanced-hunting-agentsinfo-table)
- [Agent 365 observability concepts](https://learn.microsoft.com/microsoft-agent-365/developer/observability-concepts)
- [Agent 365 attribute reference](https://learn.microsoft.com/microsoft-agent-365/developer/observability-attribute-reference)
