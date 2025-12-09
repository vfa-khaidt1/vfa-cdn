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
    class MouseEventType {
        <<enumeration>>
        Down
        Move
        Up
        Wheel
    }
    
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
        +MouseEventType type
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
        -ICommand* runningCommand // current running command, will block event rundown commands below
        -vector~ICommand~ helperCommands // Snap, Highlight
        -vector~ICommand~ viewCommands // Multiple view command can run Pan/ Rotate/ Wheel mouse zoom.
        -ICommand* primaryCommand // 1 primary command at a time

        -CommandContext context

        +DispatchEvent(MouseEvent e): CommandStatus // DispatchMouseDown, DispatchMouseUp, DispatchWheel....
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

Event routing chart
```mermaid
flowchart TD
    Start([Start: ScreenEventHandler receives Event]) --> SwitchEvent{Event Type?}

    %% PHASE 0: GestureDetector
    subgraph Phase0 [Phase 0: GestureDetector]
        direction TB
        
        %% MOUSE DOWN
        SwitchEvent -- OnMouseDown --> ResetState["startPos = currentPos <br/> dragActive = false"]
        ResetState --> DetectNone1[return None]

        %% MOUSE MOVE
        SwitchEvent -- OnMouseMove --> CheckDragActive{dragActive == true?}
        
        CheckDragActive -- Yes --> DetectDrag1[return Drag]
        
        CheckDragActive -- No --> CalcDist["dist = length(currentPos - startPos)"]
        CalcDist --> CheckThresh{dist > dragThresholdPx?}
        
        CheckThresh -- Yes --> SetActive[dragActive = true]
        SetActive --> DetectDrag2[return Drag]
        
        CheckThresh -- No --> DetectNone2[return None]

        %% MOUSE UP
        SwitchEvent -- OnMouseUp --> CheckActiveUp{dragActive == true?}
        
        CheckActiveUp -- Yes --> DetectDragEnd["return Drag <br/> (Finish Drag)"]
        CheckActiveUp -- No --> DetectClick[return Click]
        
        DetectDragEnd -.-> ResetInternal[reset internal state]
        DetectClick -.-> ResetInternal

        %% WHEEL
        SwitchEvent -- OnWheel --> DetectWheel[return Wheel]
    end

    DetectNone1 & DetectNone2 --> SetEvtNone[e.gestureType = None]
    DetectDrag1 & DetectDrag2 & DetectDragEnd --> SetEvtDrag[e.gestureType = Drag]
    DetectClick --> SetEvtClick[e.gestureType = Click]
    DetectWheel --> SetEvtWheel[e.gestureType = Wheel]

    SetEvtNone & SetEvtDrag & SetEvtClick & SetEvtWheel --> InputCheck{Has runningCommand?}

    %% PHASE 1: ACTIVE COMMAND
    subgraph Phase1 [Phase 1: Active Command]
        direction TB
        InputCheck -- Yes --> DispatchRunning[runningCommand->OnEvent]
        DispatchRunning --> CheckStatus{Status == Finished OR Cancel?}
        CheckStatus -- Yes --> ResetRunning[runningCommand = nullptr]
        CheckStatus -- No --> RetRunning[Return Status]
        ResetRunning --> RetRunning
    end

    %% PHASE 2: HELPERS
    subgraph Phase2 [Phase 2: Helpers]
        InputCheck -- No --> LoopHelpers[Loop: helperCommands]
        LoopHelpers --> HelperExec[helper->OnEvent]
        HelperExec --> NextHelper{Next Helper?}
        NextHelper -- Yes --> LoopHelpers
        NextHelper -- No --> Phase3Entry
    end

    %% PHASE 3: VIEW LAYER
    subgraph Phase3 [Phase 3: View Commands]
        Phase3Entry([Start View Layer]) --> LoopView[Loop: viewCommands]
        LoopView --> ViewExec[view->OnEvent]
        ViewExec --> CheckViewStatus{Status == Running?}
        CheckViewStatus -- Yes --> SetViewRunning[runningCommand = view]
        SetViewRunning --> RetView[Return Running]
        CheckViewStatus -- No --> NextView{Next View?}
        NextView -- Yes --> LoopView
        NextView -- No --> Phase4Entry
    end

    %% PHASE 4: PRIMARY LAYER
    subgraph Phase4 [Phase 4: Primary Command]
        Phase4Entry([Start Primary Layer]) --> CheckPrimary{Has primaryCommand?}
        CheckPrimary -- Yes --> PrimaryExec[primaryCommand->OnEvent]
        PrimaryExec --> CheckPrimStatus{Status == Running?}
        
        CheckPrimStatus -- Yes --> SetPrimRunning[runningCommand = primaryCommand]
        CheckPrimStatus -- No --> CheckPrimFinish{Status == Finished?}
        CheckPrimFinish -- Yes --> PrimEnd[primaryCommand->OnEnd]
        CheckPrimFinish -- No --> RetPrim[Return Status]
        
        SetPrimRunning --> RetPrim
        PrimEnd --> RetPrim
        CheckPrimary -- No --> RetNone[Return None]
    end

    RetRunning --> End([End])
    RetView --> End
    RetPrim --> End
    RetNone --> End

    style Phase0 fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style Phase1 fill:#f9f1f1,stroke:#333,stroke-width:2px
```

```mermaid
classDiagram
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
    
    class DistanceMeasureCommand {
        -bool hasFirstPoint
        -vec3 firstPoint
        -vec3 currentPoint
        +OnStart()
        +OnMouseDown(MouseEvent e): CommandStatus
        +OnMouseMove(MouseEvent e): CommandStatus
        +OnEnd()
        -UpdateRubberBandLine()
    }

    class AreaMeasureCommand {
        -vector~vec3~ points
        +OnStart()
        +OnMouseDown(MouseEvent e): CommandStatus
        +OnMouseMove(MouseEvent e): CommandStatus
        +OnEnd()
        -UpdateRubberBandPolygon()
    }

    class DebugCommand {
        -int clickCount
        +OnStart()
        +OnMouseDown(MouseEvent e): CommandStatus
        +GetContextMenuItems(): vector~ContextMenuItem~
        +OnContextMenuItemClicked(string itemId)
    }

    class ContextMenuCommand {
        -ICommand* targetCommand  // command whose menu is shown
        -vector~ContextMenuItem~ currentItems
        +OnStart()
        +OnMouseDown(MouseEvent e): CommandStatus
        +OnContextMenuItemClicked(string itemId)
        +GetContextMenuItems(): vector~ContextMenuItem~ // usually empty; it shows target's items
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
    
    class HoverHighlightCommand {
        -Color _originalColor
        -Color _highlightColor
        +OnMouseMove(event)
    }

    ICommand <|-- DistanceMeasureCommand
    ICommand <|-- AreaMeasureCommand
    ICommand <|-- LineDrawingCommand
    ICommand <|-- DebugCommand
    ICommand <|-- ContextMenuCommand
    ICommand <|.. RotateNavigationCommand
    ICommand <|.. PanNavigationCommand
    ICommand <|.. ZoomNavigationCommand
    ICommand <|.. HoverHighlightCommand
```
