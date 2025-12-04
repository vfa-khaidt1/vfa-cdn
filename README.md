CHART:

```mermaid
flowchart LR
    subgraph Input["Window / OS"]
        Raw["Raw Mouse Events"]
    end

    Raw --> MIR["MouseInputRouter"]

    subgraph Core["Core Input Layer"]
        MIR --> CM["CommandManager\n(per-button slots)"]
    end

    subgraph Slots["Command Slots"]
        LSlot["LeftCommand\n(e.g. DistanceMeasureCommand)"]
        RSlot["RightCommand\n(e.g. ViewRotateCommand)"]
        MSlot["MiddleCommand\n(optional)"]
    end

    CM --> LSlot
    CM --> RSlot
    CM --> MSlot

    subgraph Commands["Concrete Commands"]
        DMC["DistanceMeasureCommand"]
        AMC["AreaMeasureCommand"]
        VRC["ViewRotateCommand"]
    end

    LSlot --> DMC
    LSlot --> AMC
    RSlot --> VRC


```mermaid
sequenceDiagram
  participant OS
  participant MIR as MouseInputRouter
  participant CM as CommandManager
  participant LCmd as LeftCommand
  participant RCmd as RightCommand

  Note over CM: Left = DistanceMeasure, Right = ViewRotate

  OS->>MIR: MouseDown (Right)
  MIR->>CM: handleMouseDown(Right)
  CM->>RCmd: onMouseDown()

  OS->>MIR: MouseMove (Right)
  MIR->>CM: handleMouseMove(Right)
  CM->>RCmd: onMouseMove()

  OS->>MIR: MouseUp (Right)
  MIR->>CM: handleMouseUp(Right)
  CM->>RCmd: onMouseUp()

  OS->>MIR: MouseDown (Left)
  MIR->>CM: handleMouseDown(Left)
  CM->>LCmd: onMouseDown()

  OS->>MIR: MouseUp (Left)
  MIR->>CM: handleMouseUp(Left)
  CM->>LCmd: onMouseUp()
