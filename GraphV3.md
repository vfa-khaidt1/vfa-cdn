```mermaid
flowchart LR
    subgraph Input["Browser Canvas"]
        Raw["Emscripten Mouse Events"]
    end

    Raw --> Router["ScreenEventHandler"]

    subgraph Core["Core Input Layer"]
        Router --> CM["MouseCommandManager"]
    end

    subgraph Slots["Command Slots"]
        LSlot["Left Mouse Slot"]
        RSlot["Right Mouse Slot"]
        MSlot["Middle Mouse Slot"]
    end

    CM --> LSlot
    CM --> RSlot
    CM --> MSlot

    subgraph Commands["Concrete Commands"]
        DMC["DistanceMeasureCommand"]
        RNC["RotationNavigationCommand"]
        AMC["AreaMeasureCommand"]
        PNC["PanNavigationCommand"]
    end

    LSlot --> DMC
    RSlot --> RNC
    MSlot --> PNC
    LSlot --> AMC
```

```mermaid
classDiagram
    class ScreenEventHandler {
        -PointerState _pointerState
        -Camera* _camera
        -App* _app
        -MouseCommandManager* _commandManager
        +onMouseDown(...)
        +onMouseMove(...)
        +onMouseUp(...)
    }

    class ICommand {
        <<interface>>
        +Name() const : const char*
        +OnStart()
        +OnEnd()
        +OnCancel()
        +OnMouseDown(event): bool
        +OnMouseMove(event): bool
        +OnMouseUp(event): bool
    }

    class MouseCommandManager {
        -App* _app
        -unique_ptr~ICommand~ leftCommand
        -unique_ptr~ICommand~ rightCommand
        -unique_ptr~ICommand~ middleCommand
        +BindCommand(button, command): bool
        +DispatchMouseDown(event): bool
        +DispatchMouseMove(event): bool
        +DispatchMouseUp(event): bool
    }

    class DistanceMeasureCommand {
        -App* _app
        -optional~vec3~ _startWorld
        -optional~Measurement~ _lastMeasurement
        +measureButton() const : MouseButton
    }

    class AreaMeasureCommand {
        -App* _app
        -optional~Measurement~ _lastMeasurement
        +measureButton() const : MouseButton
    }

    class ViewNavigationCommand {
        -Camera* _camera
        -App* _app
    }

    class Camera {
        +Rotate(dx,dy): void
        +Pan(dx,dy): void
        +Zoom(...): void
    }

    class InputState {
        <<interface>>
        + Id() string
        + OnEnter(manager : MouseCommandManager)
        + OnExit(manager : MouseCommandManager)
    }

    class NormalState {
        + Id() string
        + OnEnter(manager)
        + OnExit(manager)
    }

    class MeasurementState {
        + Id() string
        + OnEnter(manager)
        + OnExit(manager)
    }

    class MouseInputStateMachine {
        - currentState : InputState*
        - MouseCommandManager* cmdManager
        + SetMode(modeId : string)
    }

    ICommand <|.. DistanceMeasureCommand
    ICommand <|.. ViewNavigationCommand
    ICommand <|.. AreaMeasureCommand

    ScreenEventHandler --> MouseCommandManager : dispatches events
    MouseCommandManager --> ICommand : per-button slots
    ViewNavigationCommand --> Camera : drives

    InputState <|.. NormalState
    InputState <|.. MeasurementState

    MouseInputStateMachine --> InputState : holds currentMode
    MouseInputStateMachine --> MouseCommandManager : rebinds slots
```

```mermaid
flowchart LR
    %% === Input side ===
    subgraph Input["Browser Canvas"]
        Raw["Emscripten Mouse Events"]
    end

    Raw --> Handler["ScreenEventHandler"]

    %% === Core input layer ===
    subgraph Core["Core Input Layer"]
        Handler --> CM["MouseCommandManager"]
        MIM["MouseInputStateMachine"]
    end

    MIM --> CM

    %% === Modes / state machine ===
    subgraph Modes["Input Modes (state machine)"]
        Normal["NormaState"]
        Measure["MeasurementState"]
    end

    MIM --> Normal
    MIM --> Measure

    %% === Per-button command slots ===
    subgraph Slots["Command Slots (per mouse button)"]
        LSlot["Left Mouse Slot"]
        RSlot["Right Mouse Slot"]
        MSlot["Middle Mouse Slot"]
    end

    CM --> LSlot
    CM --> RSlot
    CM --> MSlot

    %% === Concrete commands ===
    subgraph Commands["Concrete Commands"]
        DMC["DistanceMeasureCommand"]
        AMC["AreaMeasureCommand"]
        VNC["ViewNavigationCommand (Rotate/Pan/Zoom)"]
    end

    %% Normal state: view navigation on right/middle
    Normal -.binds.-> RSlot
    Normal -.binds.-> MSlot
    RSlot -.uses.-> VNC
    MSlot -.uses.-> VNC

    %% Measurement state: rebind to measurement commands
    Measure -.rebinds.-> RSlot
    Measure -.rebinds.-> LSlot
    RSlot -.may use.-> DMC
    LSlot -.may use.-> AMC
```

```mermaid
flowchart LR
    %% === Input side ===
    subgraph Input["Browser Canvas"]
        Raw["Emscripten Mouse Events"]
    end

    Raw --> Handler["ScreenEventHandler"]

    %% === Core input layer ===
    subgraph Core["Core Input Layer"]
        Handler --> CM["MouseCommandManager\n(holds current mode + bindings)"]
    end

    %% Handler changes mode (e.g. keyboard / UI)
    Handler -.SetMode(Normal/Measurement).-> CM

    %% === Per-button slots inside CM ===
    subgraph Slots["Per-Mode Command Slots"]
        NLeft["Normal.Left → ViewNavigation"]
        NRight["Normal.Right → ViewNavigation"]
        MLeft["Measurement.Left → AreaMeasure"]
        MRight["Measurement.Right → DistanceMeasure"]
    end

    CM -.uses.-> NLeft
    CM -.uses.-> NRight
    CM -.uses.-> MLeft
    CM -.uses.-> MRight

    %% === Commands ===
    subgraph Commands["Concrete Commands"]
        VNC["ViewNavigationCommand"]
        DMC["DistanceMeasureCommand"]
        AMC["AreaMeasureCommand"]
    end

    NLeft --> VNC
    NRight --> VNC
    MLeft --> AMC
    MRight --> DMC

```

```mermaid
classDiagram
    class ScreenEventHandler {
        -PointerState _pointerState
        -Camera* _camera
        -App* _app
        -MouseCommandManager* _commandManager
        +onMouseDown(...)
        +onMouseMove(...)
        +onMouseUp(...)
        +setMode(mode : InputMode)
    }

    class InputMode {
        <<enum>>
        Normal
        Measurement
    }

    class ICommand {
        <<interface>>
        +Name() const : const char*
        +OnStart()
        +OnEnd()
        +OnCancel()
        +OnMouseDown(event): bool
        +OnMouseMove(event): bool
        +OnMouseUp(event): bool
    }

    class MouseCommandManager {
        -App* _app
        -InputMode _currentMode
        -unique_ptr~ICommand~ _leftNormal
        -unique_ptr~ICommand~ _rightNormal
        -unique_ptr~ICommand~ _middleNormal
        -unique_ptr~ICommand~ _leftMeasure
        -unique_ptr~ICommand~ _rightMeasure
        -unique_ptr~ICommand~ _middleMeasure
        +SetMode(mode : InputMode): void
        +BindCommand(mode : InputMode, button, command): bool
        +DispatchMouseDown(event): bool
        +DispatchMouseMove(event): bool
        +DispatchMouseUp(event): bool
    }

    class DistanceMeasureCommand {
        -App* _app
        -optional~vec3~ _startWorld
        -optional~Measurement~ _lastMeasurement
    }

    class AreaMeasureCommand {
        -App* _app
        -optional~Measurement~ _lastMeasurement
    }

    class ViewNavigationCommand {
        -Camera* _camera
        -App* _app
    }

    class Camera {
        +Rotate(dx,dy): void
        +Pan(dx,dy): void
        +Zoom(...): void
    }

    ICommand <|.. DistanceMeasureCommand
    ICommand <|.. ViewNavigationCommand
    ICommand <|.. AreaMeasureCommand

    ScreenEventHandler --> MouseCommandManager : dispatches events / sets mode
    MouseCommandManager --> ICommand : uses per-mode slots
    ViewNavigationCommand --> Camera : drives camera

```
