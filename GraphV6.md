```mermaid
classDiagram
    class MouseButton {
        <<enumeration>>
        Left
        Right
        Middle
        Any // For Hover...
    }

    class GestureType {
        <<enumeration>>
        None
        Click
        Drag
    }

    class CommandType {
        <<enumeration>>
        HELPER
        PRIMARY 
        VIEW
    }

    class MouseActionState {
        -bool success
        -bool isBlock // Block lower priority command
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
        +Name() const: const char*
        +CommandType() const: CommandType
        +GetMouseButton() const: MouseButton
        +IsEnable() const: bool
        +GetPriority() const: int // index
        +OnStart()
        +OnEnd()
        +OnCancel()
        +OnMouseDown(event): MouseActionState
        +OnMouseMove(event): MouseActionState
        +OnMouseUp(event): MouseActionState
        +OnWheel(event): MouseActionState
    }


    class MouseCommandManager {
        -vector~ICommand~ primaryCommands
        -vector~ICommand~ viewCommands
        -vector~ICommand~ helperCommands
        +DispatchMouseDown(event): bool
        +DispatchMouseMove(event): bool
        +DispatchMouseUp(event): bool
        +DispatchWheel(event): bool
        +PushCommand(CommandType, ICommand): bool
    }

    ScreenEventHandler --> GestureDetector
    ScreenEventHandler --> MouseCommandManager
    GestureDetector ..> GestureType

    MouseCommandManager --> ICommand
    ICommand ..> MouseActionState
    ICommand ..> CommandType
    ICommand ..> MouseButton
```

```mermaid
flowchart TB
    Event[Mouse event for this button]
    End[End]
    subgraph CommandList
        FirstCommand
        Block1{Block?}
        SecondCommand
        Block2{Block?}
        ThirdCommand
    end


    Event --> FirstCommand
    FirstCommand -->Block1 --> |No| SecondCommand
    Block1 --> |Yes| End
    SecondCommand -->Block2 --> |No| ThirdCommand
    Block2 --> |Yes| End
    ThirdCommand --> End
```

```mermaid
classDiagram
   class CommandState {
        <<enumeration>>
        IDLE
        ACTIVE
        WAITING_INPUT
        EXECUTING
        COMPLETED
        CANCELLED
    }
      class ICommand {
        <<interface>>
        +Name() const: const char*
        +OnStart()
        +OnEnd()
        +OnCancel()
        +OnMouseDown(event): MouseActionState
        +OnMouseMove(event): MouseActionState
        +OnMouseUp(event): MouseActionState
        +OnWheel(event): MouseActionState
    }
   class StatefulCommand {
        <<abstract>>
        -CommandState _state
        -StateMachine* _stateMachine
        +GetState(): CommandState
        +TransitionTo(state)
        #onStateEnter(state)*
        #onStateExit(state)*
    }
    
    class DistanceMeasureCommand {
        -vec3 _startWorld
        -vec3 _endWorld
        -bool _hasStart
        #onStateEnter(state)
        #onStateExit(state)
    }
    
    class AreaMeasureCommand {
        -vector~vec3~ _points
        -bool _isClosed
    }
    
    class LineDrawingCommand {
        -vector~vec3~ _points
        -bool _continuous
        +ContinueDrawing()
    }
    
    class RotateNavigationCommand {
        -Camera* _camera
    }
    
    class PanNavigationCommand {
        -Camera* _camera
    }
    
    class ZoomNavigationCommand {
        -Camera* _camera
        +OnWheel(delta)
    }
    
    class SnapToMidpointCommand {
        -vec3 _point1
        -vec3 _point2
        +GetMidpoint(): vec3
    }
    
    class HoverHighlightCommand {
        -Color _originalColor
        -Color _highlightColor
        +OnMouseMove(event)
    }

    ICommand <|.. StatefulCommand
    StatefulCommand <|-- DistanceMeasureCommand
    StatefulCommand <|-- AreaMeasureCommand
    StatefulCommand <|-- LineDrawingCommand
    ICommand <|.. RotateNavigationCommand
    ICommand <|.. PanNavigationCommand
    ICommand <|.. ZoomNavigationCommand
    ICommand <|.. SnapToMidpointCommand
    ICommand <|.. HoverHighlightCommand
```
