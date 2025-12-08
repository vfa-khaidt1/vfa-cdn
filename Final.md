MouseCommandManager manages three layers of commands:
- Helper commands (e.g. snap, highlight)
- View commands (e.g. pan, rotate, wheel zoom)
- a single active Primary command (e.g. distance measurement, selection) 
It also keeps a single runningCommand, which captures subsequent events while it is in the Running state and blocks them from going down to lower layers.

When a mouse event is received, MouseCommandManager:
1. Send event to runningCommand (if not null). If result == Running → stop.
2. Else dispatch to below commands in order: Helper → View → Primary.
3. If a command that returns Running state. Set as new runningCommand.

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
        +onWheel(event)
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
        -ICommand* runningCommand // current running command, If not null it will block event rundown other commands below
        -vector~ICommand~ helperCommands // Snap, Highlight
        -vector~ICommand~ viewCommands // Multiple view command can run Pan/ Rotate/ Wheel mouse zoom.
        -ICommand* primaryCommand // 1 primary command at a time

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


Here is the example of a mouse event routing

```
CommandStatus DispatchMouseDown(MouseEvent& e) {
        // If has running command
        if(runningCommand!=nullptr)
        {    
            CommandStatus status = runningCommand->OnMouseDown(e);
            if (status == CommandStatus::Finished || status == CommandStatus::Canceled) {
                runningCommand = nullptr;
                // Allow event from go down below
            } else if (status == CommandStatus::Running )
                return status; // Stop event from go down below
        }
    
        // [LAYER 1] Helper
        for (auto helper : helperCommands) {
             CommandStatus status =helper->OnMouseDown(e);
             if (status == CommandStatus::Running) {
                runningCommand = helper;
                return status; 
            }
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

