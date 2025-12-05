```mermaid
classDiagram
    class ScreenEventHandler {
        -PointerState _pointerState
        -GestureRecognizer _gesture
        -MouseCommandOrchestrator* _orchestrator
        +onMouseDown(event)
        +onMouseMove(event)
        +onMouseUp(event)
        +onWheel(event)
        +onKeyDown(event)
        +onKeyUp(event)
    }

    class GestureRecognizer {
        +updatePointer(event) void
        +classifyMouse(event) GestureType
    }

    class MouseCommandOrchestrator {
        -ICommand* _activePrimary
        -ICommand* _viewPan
        -ICommand* _viewRotate
        -ICommand* _viewZoomWheel
        -ICommand* _viewOther
        -ICommand* _helperStack[ ]
        +setPrimaryCommand(cmd : ICommand*) void
        +pushHelperCommand(cmd : ICommand*) void
        +popHelperCommand() void
        +dispatchMouseDown(event, gesture) bool
        +dispatchMouseMove(event, gesture) bool
        +dispatchMouseUp(event, gesture) bool
        +dispatchWheel(event) bool
        +dispatchKeyDown(event) bool
        +dispatchKeyUp(event) bool
    }

    class ICommand {
        <<interface>>
        +name() string
        +getKind() CommandKind
        +onStart(context : CommandContext) void
        +onEnd(context : CommandContext) void
        +onCancel(context : CommandContext) void
        +onMouseDown(event, context : CommandContext) bool
        +onMouseMove(event, context : CommandContext) bool
        +onMouseUp(event, context : CommandContext) bool
        +onWheel(event, context : CommandContext) bool
        +onKeyDown(event, context : CommandContext) bool
        +onKeyUp(event, context : CommandContext) bool
    }

    class CommandKind {
        <<enumeration>>
        View
        Primary
        Helper
    }

    class CommandContext {
        -App* app
        -Camera* camera
        -SelectionManager* selection
        -SnapManager* snap
        +getApp() App*
        +getCamera() Camera*
        +getSelection() SelectionManager*
        +getSnap() SnapManager*
    }

    class DistanceMeasureCommand {
        -DistanceMeasureState _state
    }

    class AreaMeasureCommand {
        -AreaMeasureState _state
    }

    class PanCommand {
        -PanState _state
    }

    class RotateCommand {
        -RotateState _state
    }

    class ZoomWheelCommand {
    }

    class MidpointHelperCommand {
    }

    class HoverHighlightHelperCommand {
    }

    class DistanceMeasureState {
        <<enumeration>>
        Idle
        AwaitFirstPoint
        AwaitSecondPoint
        Completed
        Canceled
    }

    class PanState {
        <<enumeration>>
        Idle
        PendingDrag
        Dragging
    }

    ICommand <|.. DistanceMeasureCommand
    ICommand <|.. AreaMeasureCommand
    ICommand <|.. PanCommand
    ICommand <|.. RotateCommand
    ICommand <|.. ZoomWheelCommand
    ICommand <|.. MidpointHelperCommand
    ICommand <|.. HoverHighlightHelperCommand

    ScreenEventHandler --> GestureRecognizer
    ScreenEventHandler --> MouseCommandOrchestrator : dispatches
    MouseCommandOrchestrator --> ICommand : routes events
    ICommand --> CommandContext : uses

```

```mermaid
flowchart LR
    subgraph Input["Browser Canvas"]
        Raw["Emscripten Mouse / Key Events"]
    end

    Raw --> SEH["ScreenEventHandler\n(pointer state + click/drag threshold)"]
    SEH --> GR["GestureRecognizer\n(click / drag / wheel)"]
    GR --> ORCH["MouseCommandOrchestrator"]

    subgraph Layers["Command Layers"]
        Primary["PrimaryCommandSlot\n(Draw / Measure)"]
        View["ViewCommandLayer\n(Pan / Rotate / Zoom / Wheel)"]
        Helper["HelperCommandStack\n(Interrupt / Helper)"]
    end

    ORCH --> Primary
    ORCH --> View
    ORCH --> Helper

```

```mermaid
classDiagram
    %% --- Data Structures ---
    class InputContext {
        +vec2 ScreenPos
        +vec3 WorldPos
        +vec3 SnappedWorldPos
        +EntityID HoveredEntity
        +vec2 Delta
        +bool IsDragging
        +bool IsClick
    }

    class InputResult {
        <<enumeration>>
        Ignored
        Consumed
        Passthrough
    }

    %% --- Core Handler ---
    class ScreenEventHandler {
        -GestureRecognizer _recognizer
        -CommandOrchestrator* _orchestrator
        -App* _app
        +OnRawEvent(eventType, x, y)
        -BuildInputContext() : InputContext
    }

    class GestureRecognizer {
        -vec2 _mouseDownPos
        -float _dragThreshold
        +UpdateState(x, y, isDown)
        +IsDragging() : bool
    }

    %% --- Command Interface ---
    class ICommand {
        <<interface>>
        +GetName() : string
        +OnEnter()
        +OnExit()
        +HandleInput(InputContext) : InputResult
        +DrawGizmos()
    }

    %% --- The Manager ---
    class CommandOrchestrator {
        -vector~ICommand*~ _helpers  // Snapping, Highlight
        -ICommand* _activeTool       // Measure, DrawLine
        -ICommand* _navigation       // Pan/Zoom/Rotate
        
        +SetTool(ICommand*)
        +DispatchInput(InputContext)
    }

    %% --- Concrete Commands ---
    class SnappingHelper {
        -Scene* _scene
        +HandleInput(ctx) : InputResult
        %% Modifies ctx.SnappedWorldPos
    }

    class DistanceMeasureCommand {
        <<State Machine>>
        -enum State { IDLE, WAITING_PT2, FINISHED }
        -State _state
        -vec3 _startPoint
        -vec3 _currentEndPoint
        +HandleInput(ctx) : InputResult
    }

    class NavigationCommand {
        -Camera* _cam
        +HandleInput(ctx) : InputResult
    }

    %% --- Relationships ---
    ScreenEventHandler --> GestureRecognizer : Uses
    ScreenEventHandler --> CommandOrchestrator : Drives
    CommandOrchestrator --> ICommand : Manages
    CommandOrchestrator ..> InputContext : Creates/Passes

    ICommand <|.. SnappingHelper
    ICommand <|.. DistanceMeasureCommand
    ICommand <|.. NavigationCommand
    
    SnappingHelper --|> DistanceMeasureCommand : Provides Data via Context
```

```mermaid
flowchart TD
    subgraph Browser["Browser & Emscripten"]
        Raw["Raw Mouse/Key Events"]
    end

    subgraph InputProcessing["Input Pre-Processing"]
        Gesture["GestureRecognizer\n(Threshold Logic: Click vs Drag)"]
        ContextBuilder["InputContext Builder\n(Raycast, World Coordinates)"]
    end

    subgraph Orchestration["Command Orchestrator"]
        Priority1["1. Helper Layer\n(Snapping, Highlighting)"]
        Priority2["2. Active Tool Layer\n(Draw, Measure)"]
        Priority3["3. Navigation Layer\n(Pan, Rotate, Zoom)"]
    end

    Raw --> Gesture
    Gesture --> ContextBuilder
    ContextBuilder -- "InputContext (Raw)" --> Priority1
    
    Priority1 -- "Modifies Context\n(e.g. Snapped Pos)" --> Priority2
    
    Priority2 -- "Event Consumed?" --> Decision{Consumed?}
    
    Decision -- No --> Priority3
    Decision -- Yes --> End((Stop))
    Priority3 --> End
```
