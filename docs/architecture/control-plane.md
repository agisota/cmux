# cmux — Socket Control-Plane Architecture

> Derived from a graphify knowledge graph of `agisota/cmux` (fork of `manaflow-ai/cmux`).
> All edges below are AST-EXTRACTED (verified), not inferred.
> The socket/CLI control-plane — not the UI — is the architectural spine: it is how
> the CLI, shell integration, and AI coding agents programmatically drive terminals,
> the in-app browser, and windows.

```mermaid
flowchart TD
    %% ---- Clients ----
    subgraph clients["External clients"]
        CLI["cmux CLI<br/>(CMUXCLI / SocketClient)"]
        SHELL["shell integration<br/>(cmux-bash-integration)"]
        AGENT["AI coding agent<br/>(claude / open wrappers)"]
    end

    %% ---- Transport ----
    SOCK{{"Unix socket<br/>JSON-RPC, ownership-checked"}}
    CLI -->|sendV2| SOCK
    SHELL -->|_cmux_send| SOCK
    AGENT -->|cmux ...| SOCK

    %% ---- Dispatch ----
    subgraph dispatch["TerminalController.swift — dispatch"]
        PC["processCommand()<br/>v1 entry"]
        PV2["processV2Command()<br/>v2 dispatcher · 175 edges"]
        PC -->|calls| PV2
    end
    SOCK --> PC

    %% ---- Handler families (fan-out from processV2Command) ----
    subgraph handlers["v2* handler families"]
        WS["v2Workspace*<br/>List/Create/Select/Close/Move/Reorder/Action"]
        TAB["v2TabAction()"]
        PANE["v2Pane*<br/>Create/Resize/Focus/Swap/Break/Join"]
        SURF["v2Surface*<br/>Split/Move/Focus/SendText/SendKey/ReadText"]
        BR["v2Browser*<br/>OpenSplit/Snapshot/Tab*"]
    end
    PV2 -->|calls| WS
    PV2 -->|calls| TAB
    PV2 -->|calls| PANE
    PV2 -->|calls| SURF
    PV2 -->|calls| BR

    %% ---- Handle model + resolvers ----
    HK["V2HandleKind (enum:String)<br/>window · workspace · pane · surface"]
    subgraph resolve["Resolvers (handle → live object)"]
        RTM["v2ResolveTabManager()"]
        RWIN["v2ResolveWindowId()"]
        REF["v2Ref() / v2MainSync()"]
    end
    WS -->|calls| RTM
    TAB -->|calls| RTM
    HK -.addresses.-> resolve

    %% ---- Subsystems ----
    subgraph subsystems["Resolved targets"]
        TM["TabManager"]
        WSP["Workspace<br/>terminal panels + portals"]
        BP["BrowserPanel<br/>WKWebView automation"]
        MWC["MainWindowContext<br/>window routing"]
    end
    RTM --> TM
    TM -->|owns/mutates| WSP
    BR --> BP
    RWIN --> MWC
    TC2["TerminalController"] -->|calls| TM
```

## Legend / invariants

| Element | Meaning |
|---|---|
| `processV2Command()` | Single dispatch bottleneck (degree 175). Every external command enters here — one place to enforce focus/access/idempotency policy. |
| `V2HandleKind` | Opaque handle model `{window, workspace, pane, surface}`. The external API speaks string handles (`surface:N`); resolvers map them to live Swift objects. |
| `v2ResolveTabManager()` | The bridge: turns a workspace/tab handle into a concrete `TabManager`. The one true edge from the protocol layer to terminal state. |
| v1 → v2 | `processCommand()` (flat v1 API) delegates to `processV2Command()` (handle-based JSON-RPC v2) — a protocol-evolution refactor. |

## Why this matters for the fork

The control-plane is the product's reason to exist: AI agents drive cmux through this
socket surface. Any fork change touching agent automation, the socket API, or new
`v2*` verbs lands in `Sources/TerminalController.swift` around `processV2Command()`,
and must respect the handle-resolution contract above.
