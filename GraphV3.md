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
        +UnbindCommand(button): void
        +CancelAll(): void
        +DispatchMouseDown(event): bool
        +DispatchMouseMove(event): bool
        +DispatchMouseUp(event): bool
        +GetCommand(button): ICommand
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

    ICommand <|.. DistanceMeasureCommand
    ICommand <|.. ViewNavigationCommand
    ICommand <|.. AreaMeasureCommand

    ScreenEventHandler --> MouseCommandManager : dispatches events
    MouseCommandManager --> ICommand : per-button slots
    ViewNavigationCommand --> Camera : drives
```

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
    LSlot --> AMC
    RSlot --> RNC
    MSlot --> PNC

    %% --- New: Binding / Settings path ---

    subgraph Binding["Mouse Binding UI / Settings"]
        Settings["Mouse Settings Panel"]
        Profile["MouseBindingProfile (saved config)"]
    end

    Settings -->|user chooses button + command| CM
    Profile -->|ApplyProfile()| CM

    CM -->|BindCommand(Left, DistanceMeasure)| LSlot
    CM -->|BindCommand(Right, RotationNav)| RSlot
    CM -->|BindCommand(Middle, PanNav)| MSlot
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

    class MouseButton {
        <<enum>>
        Left
        Right
        Middle
    }

    class MouseCommand {
        <<interface>>
        +Name() const : const char*
        +OnStart()
        +OnEnd()
        +OnCancel()
        +OnMouseDown(event): bool
        +OnMouseMove(event): bool
        +OnMouseUp(event): bool
    }

    class MouseBindingProfile {
        -bindings : map
        +Get(button: MouseButton) : string
        +Set(button: MouseButton, commandName: string) : void
    }

    class MouseCommandFactory {
        +Create(commandName: string, app: App*, camera: Camera*): MouseCommand*
    }

    class MouseCommandManager {
        -App* _app
        -MouseBindingProfile _profile
        -MouseCommandFactory _factory
        -unique_ptr~MouseCommand~ leftCommand
        -unique_ptr~MouseCommand~ rightCommand
        -unique_ptr~MouseCommand~ middleCommand
        +BindCommand(button: MouseButton, commandName: string): bool
        +UnbindCommand(button: MouseButton): void
        +ApplyProfile(profile: MouseBindingProfile): void
        +CancelAll(): void
        +DispatchMouseDown(event): bool
        +DispatchMouseMove(event): bool
        +DispatchMouseUp(event): bool
        +GetCommand(button: MouseButton): MouseCommand*
    }

    class DistanceMeasureCommand {
        -App* _app
        -optional~vec3~ _startWorld
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

    MouseCommand <|.. DistanceMeasureCommand
    MouseCommand <|.. ViewNavigationCommand

    ScreenEventHandler --> MouseCommandManager : dispatches events
    MouseCommandManager --> MouseCommand : per-button slots
    MouseCommandManager --> MouseBindingProfile : load/save bindings
    MouseCommandManager --> MouseCommandFactory : creates commands
    MouseCommandManager --> MouseButton
    ViewNavigationCommand --> Camera : drives
```
