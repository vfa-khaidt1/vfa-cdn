
Tasks break down:
1. Define behavior + state machine (Center -> pick P1 -> pick P2-valid-opposite-side) + rubberband preview: 1 day

2. Implement CrossSectionCommand logic (opposite-side validation, plane build, output result): 1 day

3. Implement profile extraction in CrossSectionManager (highest height per bin along P1->P2, thickness filter): 2 days

4. Cross-section view update. Send CuttingPlane and CrossSectionResult to CrossSectionCanvas View: 2 day

5. Testing and Backup: 2 day

Total: 8 days

```mermaid
classDiagram
    class ICommand {
        <<interface>>
        +OnStart()
        +OnEnd()
        +OnCancel()
        +OnInput(input: IInput): CommandStatus
    }

    class AbstractPointerCommand {
        <<abstract>>
        +OnInput(input: IInput): CommandStatus
        #OnClick(pos: vec2): CommandStatus
        #OnHoverInput(pos: vec2): CommandStatus
        #OnDrag(pos: vec2): CommandStatus
        #OnWheelInput(delta: float, pos: vec2): CommandStatus
        #OnPinchInput(scaleDelta: float, center: vec2): CommandStatus
    }

    class CoordinatePickingCommand {
        <<abstract>>
        #bool m_enableHover
        #PickingState m_state
        #ResetPickingState(): void
        #OnPointPicked(worldPoint: glm::vec3): CommandStatus
        #OnHoverPoint(worldPoint: glm::vec3): CommandStatus
        #OnClick(pos: vec2): CommandStatus
        #OnHoverInput(pos: vec2): CommandStatus
        #IsOppositeSide(center: vec3, p1: vec3, p2: vec3): bool
    }

    class PickingState {
        <<enumeration>>
        WaitingForFirstPoint
        WaitingForNextPoint
        Completed
    }

    ICommand <|-- AbstractPointerCommand
    AbstractPointerCommand <|-- CoordinatePickingCommand
    CoordinatePickingCommand --> PickingState

    class CuttingPlane {
        +glm::vec3 point1
        +glm::vec3 point2
        +glm::vec3 center
        +glm::vec3 normal
        +glm::vec2 extentSize
        +float thickness
        +float zoomLevel
    }

    class CrossSectionResult {
        +std::vector~glm::vec3~ profilePoints
    }

    class CrossSectionManager {
        +PointCloudModel* sourceCloud
        +CuttingPlane activePlane
        +CrossSectionResult lastResult
        +BuildTerrainProfileForSegment(p1: vec3, p2: vec3, plane: CuttingPlane): CrossSectionResult
        +UpdateCrossSectionView(data: CrossSectionResult): void
    }

    class CommandContext {
        +RubberBandManager* rubberBand
        +CrossSectionManager* crossSectionManager
        +pickNearestPoint(x: double, y: double): glm::vec3
    }

    class CrossSectionCommand {
        -Step m_step
        -glm::vec3 m_center
        -glm::vec3 m_p1
        -glm::vec3 m_p2
        -int m_binCount
        -float m_halfThickness
        +OnStart(): void
        +OnEnd(): void
        #OnPointPicked(worldPoint: glm::vec3): CommandStatus
        #OnHoverPoint(worldPoint: glm::vec3): CommandStatus
        -BuildVerticalPlaneFromSegment(p1: vec3, p2: vec3): CuttingPlane
        -IsOppositeSide(center: vec3, p1: vec3, p2: vec3): bool
    }

    class Step {
        <<enumeration>>
        PickCenter
        PickP1
        PickP2
        Completed
    }

    CoordinatePickingCommand <|-- CrossSectionCommand
    CrossSectionCommand --> Step

    CoordinatePickingCommand --> CommandContext
    CrossSectionCommand --> CrossSectionManager
    CrossSectionManager --> CuttingPlane
    CrossSectionManager --> CrossSectionResult

```
