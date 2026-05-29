# cmux — Browser Automation Flow

> Derived from a graphify knowledge graph of `agisota/cmux` (fork of `manaflow-ai/cmux`).
> Edges are AST-EXTRACTED unless noted. cmux ships an in-app, scriptable browser whose
> command surface is a WKWebView port of [`vercel-labs/agent-browser`](https://github.com/vercel-labs/agent-browser),
> exposed to AI agents over the same Unix-socket control-plane as the terminal.

```mermaid
flowchart TD
    AGENT["AI agent / CLI / browser-skill<br/>(cmux browser ...)"]
    SOCK{{"Unix socket · JSON-RPC v2"}}
    PV2["processV2Command()<br/>TerminalController.swift · dispatcher"]
    AGENT --> SOCK --> PV2

    %% ~85 v2Browser* verbs, grouped by family
    subgraph verbs["v2Browser* verbs (~85, agent-browser parity)"]
        NAV["Navigation<br/>OpenSplit · Navigate · Back/Forward · Reload · GetURL"]
        LOC["Locators<br/>FindRole/Text/Label/Placeholder/TestId · First/Last/Nth"]
        ACT["Interaction<br/>Click · DblClick · Hover · Type · Fill · Press · Check · Select · Scroll"]
        QRY["Query / Read<br/>GetText/HTML/Value/Attr/Box/Styles · IsVisible/Enabled/Checked"]
        CAP["Capture<br/>Snapshot (a11y tree) · Screenshot · Screencast · Highlight"]
        STATE["State<br/>Cookies · Storage · StateSave/Load · AddInitScript/Script/Style"]
        NET["Network / Trace<br/>NetworkRoute · NetworkRequests · TraceStart/Stop"]
        TABS["Tabs / Frames<br/>TabNew/List/Switch/Close · FrameSelect/Main"]
        INPUT["Low-level input<br/>InputMouse · InputKeyboard · InputTouch"]
    end
    PV2 -->|calls| NAV & LOC & ACT & QRY & CAP & STATE & NET & TABS & INPUT

    %% Resolution: element/surface handles -> live objects
    REF["element refs (e1, e2 ...) + surface handle<br/>resolved via v2Ref / V2HandleKind.surface"]
    verbs -.resolve.-> REF

    %% Panel + web engine
    subgraph panel["Browser panel subsystem · Sources/Panels"]
        BP["BrowserPanel<br/>(deg 83) · state, history, search"]
        BPV["BrowserPanelView<br/>omnibar, suggestions"]
        CWV["CmuxWebView<br/>WKWebView wrapper"]
        HIST["BrowserHistoryStore<br/>frecency"]
        SEARCH["BrowserSearchEngine<br/>+ suggestion service"]
    end
    REF --> BP
    BP --> BPV --> CWV
    BP --> HIST
    BPV --> SEARCH
    BP -.hosted by.-> PORTAL["BrowserWindowPortal<br/>AppKit portal over SwiftUI"]

    %% Execution at the engine
    CWV -->|evaluateJavaScript| JS["page JS / DOM"]
    CWV -->|accessibility tree| A11Y["a11y snapshot → element refs"]

    %% Lineage
    SPEC["Agent-Browser Port Spec<br/>docs/agent-browser-port-spec.md"]
    SPEC -. references .-> UPSTREAM["agent-browser (vercel-labs)"]
    SPEC -. specifies .-> verbs
```

## How a command flows (example: `cmux browser click e3`)

1. Agent sends JSON-RPC over the socket → `processV2Command()` dispatches to `v2BrowserClick()`.
2. The verb resolves the **element ref** `e3` (and the target **surface handle**) via `v2Ref` / `V2HandleKind.surface` to a live `BrowserPanel`.
3. `BrowserPanel` → `CmuxWebView` (WKWebView) performs the action by `evaluateJavaScript` against the resolved DOM node.
4. Read-style verbs (`Snapshot`, `GetText`) return the **accessibility tree**, which is how agents obtain stable element refs in the first place.

## Key facts & WKWebView constraints

| Element | Note |
|---|---|
| ~85 `v2Browser*` verbs | Near-complete parity with `vercel-labs/agent-browser`, ported onto WKWebView. |
| Element refs (`e1`, `e2`…) | Ephemeral handles minted from the a11y snapshot; stable within a snapshot generation. |
| `CmuxWebView` | The single WKWebView bridge — all DOM interaction funnels through `evaluateJavaScript`. |
| Platform gaps (rationale) | WKWebView lacks CDP-style per-context proxy and a CDP video pipeline → some upstream agent-browser features are approximated or unavailable. |
| Same control-plane | Browser verbs share the socket dispatcher with terminal/window verbs — see [control-plane.md](./control-plane.md). |

## Why this matters for the fork

The browser surface is half of cmux's agent-automation value. Any fork work on
new `v2Browser*` verbs, locator strategies, or WKWebView workarounds lands in
`Sources/Panels/BrowserPanel*.swift` + `CmuxWebView.swift`, is dispatched from
`processV2Command()`, and should track the `agent-browser` port spec for parity.
