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
