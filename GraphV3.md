

**CommandPattern:**
- We will use CommandPattern for main Pattern to store active commands of left/right/middle
- We may need to combine with StateMachine and Event Dispatcher 

**State Machine:**
=> Use to store state of each command : 
For example distance measurement: 
```mermaid
flowchart TD
    A(Prepare) --> B(Picked Point 1)
    B --> C(Picked Point 2)
    C --> X(End)
    Z[AnyState] --> |Cancel| X(End)
```

**View operatation(rotate/ pan) vs Click operation**
=> Right now we treated the same
**Middle Mouse Scroll**
- Dont have interface like click action?
  => need to create a separated Command.

**Temporary Architecture Graph ( In Progress )**
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
        //Active commands for each mouse
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
        -vec3~ _startWorld
        -vec3~ _endWorld
        +Measure(): void
    }

    class AreaMeasureCommand {
        -App* _app
        +Measure() : void
    }

    class PanNavigationCommand {
        -Camera* _camera
        -App* _app
    }

    class RotateNavigationCommand {
        -Camera* _camera
        -App* _app
    }

    class Camera {
        +Rotate(dx,dy): void
        +Pan(dx,dy): void
        +Zoom(...): void
    }


    ICommand <|.. DistanceMeasureCommand
    ICommand <|.. AreaMeasureCommand
    ICommand <|.. RotateNavigationCommand
    ICommand <|.. PanNavigationCommand

    ScreenEventHandler --> MouseCommandManager : dispatches events
    MouseCommandManager --> ICommand : per-button slots
    RotateNavigationCommand --> Camera : drives
    PanNavigationCommand --> Camera : drives
```


**Implementing event routing for dragging and clicking ( In Progress )**

```mermaid
flowchart TD
    A([Mouse Event Received]) --> B{Event Type?}

    B -->|MouseDown| C[Save Mouse Position]
    C --> D[isPossibleClick = TRUE]
    D --> E[onMouseDown]

    B -->|MouseMove| F[Calculate travel distance d]
    F --> G{d > Threshold?}
    G -->|Yes Drag| H[isPossibleClick = FALSE]
    H --> I[Update camera: Rotate View]
    G -->|No| J[onMouseMove]
    J --> K[Rubber Banding]

    B -->|MouseUp| L{isPossibleClick?}
    L -->|TRUE Click| M[onMouseUp]
    M --> N[Take measurements]
    L -->|FALSE End Drag| O[Stop Camera Rotation]


```


