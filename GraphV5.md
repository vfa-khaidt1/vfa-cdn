```mermaidclassDiagram
    class MouseButton {
        <<enumeration>>
        Left
        Right
        Middle
    }

    class GestureType {
        <<enumeration>>
        None
        Click
        Drag
    }

    class ScreenEventHandler {
        -PointerState pointerState
        -GestureDetector gestureDetector
        -MouseCommandManager commandManager
        +onMouseDown(event)
        +onMouseMove(event)
        +onMouseUp(event)
        +onWheel(event)
    }

    class GestureDetector {
        -vec2 startPos
        -double startTimeMs
        -bool dragActive
        -double dragThresholdPx
        +reset()
        +onMouseDown(event)
        +onMouseMove(event)
        +onMouseUp(event)
        +getGestureType(): GestureType
    }

    class ICommand {
        <<interface>>
        +OnStart()
        +OnEnd()
        +OnCancel()
        +OnMouseDown(event): bool
        +OnMouseMove(event): bool
        +OnMouseUp(event): bool
        +OnWheel(event): bool
        +Update(deltaTime)
    }

    class MouseButtonSlots {
        -MouseButton button
        -ICommand primaryCommand
        -ICommand viewCommand
        -ICommand helperCommand
        +GetButton(): MouseButton
        +GetPrimary(): ICommand
        +GetView(): ICommand
        +GetHelper(): ICommand
        +SetPrimary(cmd: ICommand)
        +SetView(cmd: ICommand)
        +SetHelper(cmd: ICommand)
    }

    class MouseCommandManager {
        -MouseButtonSlots leftSlots
        -MouseButtonSlots rightSlots
        -MouseButtonSlots middleSlots
        +BindPrimary(button: MouseButton, cmd: ICommand)
        +BindView(button: MouseButton, cmd: ICommand)
        +BindHelper(button: MouseButton, cmd: ICommand)
        +DispatchMouseDown(event, gesture: GestureType): bool
        +DispatchMouseMove(event, gesture: GestureType): bool
        +DispatchMouseUp(event, gesture: GestureType): bool
        +DispatchWheel(event): bool
        +UpdateAll(deltaTime)
        -getSlots(button: MouseButton): MouseButtonSlots
    }

    ScreenEventHandler --> GestureDetector
    ScreenEventHandler --> MouseCommandManager
    GestureDetector --> GestureType

    MouseCommandManager --> MouseButtonSlots
    MouseButtonSlots --> MouseButton
    MouseCommandManager --> ICommand
```


```mermaidclassDiagram
    class CommandCategory {
        <<enumeration>>
        Primary
        View
        Helper
    }

    class ICommand {
        <<interface>>
        +GetCategory() CommandCategory
        +OnStart()
        +OnEnd()
        +OnCancel()
        +OnMouseDown(event) bool
        +OnMouseMove(event) bool
        +OnMouseUp(event) bool
        +OnWheel(event) bool
        +Update(deltaTime)
    }

    %% ===== Primary commands (long living, can use state machine internally) =====
    class DistanceMeasureCommand {
        -vec3 startWorld
        -vec3 endWorld
        -bool hasStart
        +GetCategory() CommandCategory
        +OnStart()
        +OnEnd()
        +OnCancel()
        +OnMouseDown(event) bool
        +OnMouseMove(event) bool
        +OnMouseUp(event) bool
    }

    class LineDrawingState {
        <<enumeration>>
        Idle
        AwaitFirstPoint
        Drawing
        Completed
        Canceled
    }

    class LineDrawingCommand {
        -LineDrawingState state
        -list points
        +GetCategory() CommandCategory
        +OnStart()
        +OnEnd()
        +OnCancel()
        +OnMouseDown(event) bool
        +OnMouseMove(event) bool
        +OnMouseUp(event) bool
        +Update(deltaTime)
    }

    %% ===== View commands (pan, rotate, wheel) =====
    class RotateViewCommand {
        -Camera camera
        -vec2 lastPos
        +GetCategory() CommandCategory
        +OnStart()
        +OnEnd()
        +OnMouseDown(event) bool
        +OnMouseMove(event) bool
        +OnMouseUp(event) bool
    }

    class PanViewCommand {
        -Camera camera
        -vec2 lastPos
        +GetCategory() CommandCategory
        +OnStart()
        +OnEnd()
        +OnMouseDown(event) bool
        +OnMouseMove(event) bool
        +OnMouseUp(event) bool
    }

    class ZoomWheelCommand {
        -Camera camera
        +GetCategory() CommandCategory
        +OnStart()
        +OnEnd()
        +OnWheel(event) bool
    }

    %% ===== Helper commands (お助けコマンド) =====
    class MidpointHelperCommand {
        -vec3 p1
        -vec3 p2
        -bool active
        +GetCategory() CommandCategory
        +OnStart()
        +OnEnd()
        +OnMouseDown(event) bool
        +GetMidpoint() vec3
    }

    class HoverHighlightCommand {
        -Entity hovered
        +GetCategory() CommandCategory
        +OnStart()
        +OnEnd()
        +OnMouseMove(event) bool
    }

    class Camera {
        +Rotate(dx, dy)
        +Pan(dx, dy)
        +Zoom(delta)
    }

    class Entity {
    }

    %% ===== Inheritance =====
    ICommand <|.. DistanceMeasureCommand
    ICommand <|.. LineDrawingCommand
    ICommand <|.. RotateViewCommand
    ICommand <|.. PanViewCommand
    ICommand <|.. ZoomWheelCommand
    ICommand <|.. MidpointHelperCommand
    ICommand <|.. HoverHighlightCommand

    %% ===== Associations =====
    LineDrawingCommand --> LineDrawingState
    DistanceMeasureCommand ..> CommandCategory
    LineDrawingCommand ..> CommandCategory
    RotateViewCommand ..> CommandCategory
    PanViewCommand ..> CommandCategory
    ZoomWheelCommand ..> CommandCategory
    MidpointHelperCommand ..> CommandCategory
    HoverHighlightCommand ..> CommandCategory

    RotateViewCommand --> Camera
    PanViewCommand --> Camera
    ZoomWheelCommand --> Camera

    HoverHighlightCommand --> Entity
    MidpointHelperCommand --> Entity
```
