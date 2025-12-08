```mermaid
classDiagram
    class MouseButton {
        <<enumeration>>
        Left
        Right
        Middle
        Any
    }

    class MouseEventType {
        <<enumeration>>
        Down
        Move
        Up
        Wheel
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
        +type : MouseEventType
        +button : MouseButton
        +gesture : GestureType
        +x : double
        +y : double
        +dx : double
        +dy : double
        +wheelDelta : double
    }

    class CommandStatus {
        <<enumeration>>
        None
        Running
        Finished
        Cancelled
    }

    class CommandContext {
        +camera : Camera
        +app : App
    }

    class ScreenEventHandler {
        -pointerState : PointerState
        -gestureDetector : GestureDetector
        -commandManager : MouseCommandManager
        +onMouseDown(event : MouseEvent) : void
        +onMouseMove(event : MouseEvent) : void
        +onMouseUp(event : MouseEvent) : void
        +onWheel(event : MouseEvent) : void
    }

    class GestureDetector {
        -startPos : vec2
        -startTimeMs : double
        -dragActive : bool
        -dragThresholdPx : double
        +reset() : void
        +onMouseDown(event : MouseEvent) : void
        +onMouseMove(event : MouseEvent) : void
        +onMouseUp(event : MouseEvent) : void
        +onWheel(event : MouseEvent) : void
        +getGestureType() : GestureType
    }

    class ICommand {
        <<interface>>
        -name : string
        -mouseButton : MouseButton
        -enabled : bool
        +ICommand(context : CommandContext)
        +GetPriority() : int
        +OnStart() : void
        +OnEnd() : void
        +OnCancel() : void
        +OnMouseDown(e : MouseEvent) : CommandStatus
        +OnMouseMove(e : MouseEvent) : CommandStatus
        +OnMouseUp(e : MouseEvent) : CommandStatus
        +OnWheel(e : MouseEvent) : CommandStatus
    }

    class MouseCommandManager {
        -runningCommand : ICommand
        -helperCommands : List~ICommand~
        -viewCommands : List~ICommand~
        -primaryCommand : ICommand
        -context : CommandContext

        +DispatchEvent(e : MouseEvent) : CommandStatus
        +PushCommand(type : CommandType, cmd : ICommand) : bool
    }

    ScreenEventHandler --> GestureDetector
    ScreenEventHandler --> MouseCommandManager
    GestureDetector ..> GestureType

    MouseCommandManager --> ICommand
    ICommand ..> CommandStatus
    ICommand ..> CommandType
    ICommand ..> CommandContext
    ICommand ..> MouseButton
    ICommand ..> MouseEvent
```

```mermaid
CommandStatus MouseCommandManager::DispatchEvent(const MouseEvent& e)
{
    auto invoke = [&](ICommand* cmd) -> CommandStatus {
        switch (e.type) {
        case MouseEventType::Down:  return cmd->OnMouseDown(e);
        case MouseEventType::Move:  return cmd->OnMouseMove(e);
        case MouseEventType::Up:    return cmd->OnMouseUp(e);
        case MouseEventType::Wheel: return cmd->OnWheel(e);
        }
        return CommandStatus::None;
    };

    // Running command (capture)
    if (runningCommand != nullptr) {
        CommandStatus status = invoke(runningCommand);
        if (status == CommandStatus::Finished || status == CommandStatus::Cancelled) {
            runningCommand = nullptr; // allow event to go down
        } else if (status == CommandStatus::Running) {
            return status;            // stop here
        }
    }

    // [LAYER 1] Helper
    for (auto* helper : helperCommands) {
        CommandStatus status = invoke(helper);
        if (status == CommandStatus::Running) {
            runningCommand = helper;
            return status;
        }
    }

    // [LAYER 2] View (Pan, Rotate, Wheel zoom)
    for (auto* view : viewCommands) {
        CommandStatus status = invoke(view);
        if (status == CommandStatus::Running) {
            runningCommand = view;
            return status;
        }
    }

    // [LAYER 3] Primary (Draw Line, Select, Measure)
    if (primaryCommand != nullptr) {
        CommandStatus status = invoke(primaryCommand);
        if (status == CommandStatus::Running) {
            runningCommand = primaryCommand;
        } else if (status == CommandStatus::Finished) {
            primaryCommand->OnEnd();
        } else if (status == CommandStatus::Cancelled) {
            primaryCommand->OnCancel();
        }
        return status;
    }

    return CommandStatus::None;
}

```
