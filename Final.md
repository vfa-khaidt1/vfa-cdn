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
        Wheel
    }

    class CommandType {
        <<enumeration>>
        HELPER
        PRIMARY 
        VIEW
    }

   class MouseEvent {
        +MouseButton button
        +GestureType gesture
        +double x
        +double y
        +double dx
        +double dy
        +double wheelDelta
    }

    class CommandStatus {
        <<enumeration>>
        None
        Running
        Finished
        Cancelled
    }

    class CommandContext {
        +Camera camera
        +App app
        +ProjectPoint(screenPos): vec2
        +UnprojectPoint(worldPos): vec2
        +Redraw()
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
        -const char* name
        -MouseButton mouseButton
        -bool isEnable 
        -IConmand(CommandContext context)
        +GetPriority() const: int // index
        +OnStart()
        +OnEnd()
        +OnCancel()
        +OnMouseDown(MouseEvent e): CommandStatus
        +OnMouseMove(MouseEvent e): CommandStatus
        +OnMouseUp(MouseEvent e): CommandStatus
        +OnWheel(MouseEvent e): CommandStatus
    }


    class MouseCommandManager {
        -ICommand* runningCommand // current running command
        -ICommand* primaryCommand // 1 primary command at a time
        -vector~ICommand~ viewCommands // Multiple view command can run Pan/ Rotate/ Wheel mouse zoom.
        -vector~ICommand~ helperCommands // Snap, Highlight
        -CommandContext context

        +DispatchMouseDown(MouseEvent e): CommandStatus
        +DispatchMouseMove(MouseEvent e): CommandStatus
        +DispatchMouseUp(MouseEvent e): CommandStatus
        +DispatchWheel(MouseEvent e): CommandStatus

        +PushCommand(CommandType, ICommand): bool
    }

    ScreenEventHandler --> GestureDetector
    ScreenEventHandler --> MouseCommandManager
    GestureDetector ..> GestureType

    MouseCommandManager --> ICommand
    ICommand ..> CommandStatus : returns
    ICommand ..> CommandType
    ICommand ..> CommandContext : uses
    ICommand ..> MouseButton : holds
    ICommand ..> MouseEvent : input

```

```
CommandStatus DispatchMouseDown(MouseEvent& e) {
        // If has running command
        if(runningCommand!=nullptr)
        {    
            CommandStatus status = runningCommand->OnMouseDown(e);
            if (status == CommandStatus::Finished || status == CommandStatus::Cancel) {
                runningCommand = nullptr;
            }
            return status; 
        }
    
        // [LAYER 1] Helper
        for (auto helper : helperCommands) {
            helper->OnMouseDown(e); 
        }

        // [LAYER 2] View Commands (Pan, Rotate)
        for (auto view : viewCommands) {
            CommandStatus status = view->OnMouseDown(e);
            if (status == CommandStatus::Running) {
                runningCommand = view;
                return status; 
            }
        }

        // [LAYER 3] Primary Command (Draw Line, Select)
        if (primaryCommand) {
            CommandStatus status = primaryCommand->OnMouseDown(e);
            if(status == CommandStatus::Running){
                 runningCommand = primaryCommand;
            }
            else if (status == CommandStatus::Finished) {
                primaryCommand->OnEnd();
            } else if (status == CommandStatus::Cancelled) {
                primaryCommand->OnCancel();
            }
            
            return status;
        }

        return CommandStatus::None;
    }
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
