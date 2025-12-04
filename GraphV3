```mermaid
flowchart LR
    subgraph Input["Browser Canvas"]
        Raw["Emscripten Mouse Events"]
    end

    Raw --> Router["ScreenEventHandler\n(click/drag threshold, per-button routing)"]

    subgraph Core["Core Input Layer"]
        Router --> CM["MouseCommandManager\n(per-button slots)"]
    end

    subgraph Slots["Command Slots"]
        LSlot["Left Slot\n(measure if rotation=Right)"]
        RSlot["Right Slot\n(measure if rotation=Left,\nrotate if rotation=Right)"]
        MSlot["Middle Slot\n(pan)"]
    end

    CM --> LSlot
    CM --> RSlot
    CM --> MSlot

    subgraph Commands["Concrete Commands"]
        DMC["DistanceMeasureCommand\n(opposite rotation button)"]
        VNC["ViewNavigationCommand\n(rotate/pan)"]
        AMC["Future: AreaMeasureCommand"]
    end

    LSlot --> DMC
    RSlot --> VNC
    MSlot --> VNC
    LSlot --> AMC
    RSlot --> AMC

```
