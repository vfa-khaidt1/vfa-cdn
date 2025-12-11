```mermaid
classDiagram
    class ICommand {
        <<interface>>
        +OnStart()
        +OnEnd()
        +OnCancel()
        +OnMouseMove(e: MouseEvent): CommandStatus
        +OnMouseUp(e: MouseEvent): CommandStatus
    }

    class CoordinatePickingCommand {
        <<abstract>>
        #bool m_enableHover
        #PickingState m_state
        +OnMouseMove(e: MouseEvent): CommandStatus
        +OnMouseUp(e: MouseEvent): CommandStatus
        #ResetPickingState(): void
        #OnPointPicked(worldPoint: glm::vec3): CommandStatus*
        #OnHoverPoint(worldPoint: glm::vec3): CommandStatus
    }

    class PickingState {
        <<enumeration>>
        WaitingForFirstPoint
        WaitingForNextPoint
        Completed
    }

    class DistanceMeasureCommand {
        -glm::vec3 m_startPoint
        +OnStart()
        +OnEnd()
        #OnPointPicked(worldPoint: glm::vec3): CommandStatus
        #OnHoverPoint(worldPoint: glm::vec3): CommandStatus
    }

    class AreaMeasureCommand {
        -std::vector<glm::vec3> m_points
        +OnStart()
        +OnEnd()
        #OnPointPicked(worldPoint: glm::vec3): CommandStatus
        #OnHoverPoint(worldPoint: glm::vec3): CommandStatus
    }

    class CommandContext {
        +RubberBandManager* rubberBand
        +pickNearestPoint(x: double, y: double): glm::vec3
    }

    class RubberBandManager {
        +Clear()
        +ShowLine(a: glm::vec3, b: glm::vec3)
        +ShowPolyline(points: std::vector<glm::vec3>, hover: glm::vec3)
    }

    class MeasurementService {
        +MeasureDistance(start: glm::vec3, end: glm::vec3): DistanceResult
        +MeasureArea(vertices: std::vector<glm::vec3>): std::optional~AreaResult~
    }

    class MouseEvent {
        +MouseButton button
        +GestureType gesture
        +double x
        +double y
    }

    class CommandStatus {
        <<enumeration>>
        None
        Running
        Finished
        Cancelled
    }

    ICommand <|-- CoordinatePickingCommand
    CoordinatePickingCommand <|-- DistanceMeasureCommand
    CoordinatePickingCommand <|-- AreaMeasureCommand

    CoordinatePickingCommand --> PickingState

    CoordinatePickingCommand --> CommandContext
    CommandContext --> RubberBandManager

    DistanceMeasureCommand --> MeasurementService
    AreaMeasureCommand --> MeasurementService

```


**Base class:** CoordinatePickingCommand
```
class CoordinatePickingCommand : public ICommand {
protected:
    enum class PickingState {
        WaitingForFirstPoint,
        WaitingForNextPoint,
        Completed
    };

    bool m_enableHover = false;
    PickingState m_state{PickingState::WaitingForFirstPoint};

    virtual CommandStatus OnPointPicked(const glm::vec3& worldPoint) = 0;
    virtual CommandStatus OnHoverPoint(const glm::vec3& worldPoint) = 0;

    void ResetPickingState() {
        m_state = PickingState::WaitingForFirstPoint;
    }

public:
    using ICommand::ICommand;

    CommandStatus OnMouseMove(const MouseEvent& e) override {
        if (!m_enableHover)
            return CommandStatus::Running;

        glm::vec3 hoverPoint = m_context.pickNearestPoint(e.x, e.y);
        return OnHoverPoint(hoverPoint);
    }

    CommandStatus OnMouseUp(const MouseEvent& e) override {
        if (e.button != mouseButton || e.gesture != GestureType::Click)
            return CommandStatus::None;

        glm::vec3 hitPoint = m_context.pickNearestPoint(e.x, e.y);
        return OnPointPicked(hitPoint);
    }
};
```

```
class DistanceMeasureCommand : public CoordinatePickingCommand {
private:
    glm::vec3 m_startPoint{0.0f};

public:
    using CoordinatePickingCommand::CoordinatePickingCommand;

    const char* GetName() const override { return "Distance Measure"; }

    void OnStart() override {
        ResetPickingState();
        m_context.rubberBand->Clear();
        m_enableHover = true;
    }

    void OnEnd() override {
        m_context.rubberBand->Clear();
    }

protected:
    CommandStatus OnHoverPoint(const glm::vec3& hoverPoint) override {
        if (m_state == PickingState::WaitingForNextPoint) {
            m_context.rubberBand->ShowLine(m_startPoint, hoverPoint);
        }
        return CommandStatus::Finished;
    }

    CommandStatus OnPointPicked(const glm::vec3& pickedPoint) override {
        switch (m_state) {
        case PickingState::WaitingForFirstPoint:
            m_startPoint = pickedPoint;
            m_state = PickingState::WaitingForNextPoint;
            return CommandStatus::Finished;

        case PickingState::WaitingForNextPoint: {
            auto result = MeasurementService::MeasureDistance(m_startPoint, pickedPoint); 

            m_context.rubberBand->Clear();
            m_state = PickingState::Completed;
            return CommandStatus::Finished;
        }

        case PickingState::Completed:
            return CommandStatus::Finished;
        }

        return CommandStatus::None;
    }
};
```

```
class AreaMeasureCommand : public CoordinatePickingCommand {
private:
    std::vector<glm::vec3> m_points;
public:
    using CoordinatePickingCommand::CoordinatePickingCommand;

    const char* GetName() const override { return "Area Measure"; }

    void OnStart() override {
        m_points.clear();
        m_context.rubberBand->Clear();
        ResetPickingState();         
        m_enableHover = true;
    }

    void OnEnd() override {
        m_context.rubberBand->Clear();
    }

protected:
    CommandStatus OnHoverPoint(const glm::vec3& hoverPoint) override {
        if (!m_points.empty()) {
            m_context.rubberBand->ShowPolyline(m_points, hoverPoint);
        }
        return CommandStatus::Running;
    }

    CommandStatus OnPointPicked(const glm::vec3& pickedPoint) override {
        if (m_state == PickingState::Completed) {
            return CommandStatus::Finished;
        }

        if (!m_points.Contains(pickedPoint)) {
            m_points.push_back(pickedPoint);
        }

        if (m_state == PickingState::WaitingForFirstPoint) {
            m_state = PickingState::WaitingForNextPoint;
        }

        if (m_points.size() >= 3) {
            auto resOpt = MeasurementService::MeasureArea(m_points);
            if (resOpt) {
                const auto& res = *resOpt;
                printf("Area: %f, perimeter: %f\n", res.area, res.perimeter);
            }
            m_state = PickingState::Completed;
            return CommandStatus::Finished;
        }

        return CommandStatus::Running;
    }
};
```

```
class MeasurementService {
public:
    struct DistanceResult {
        double length = 0.0;
        glm::vec3 start{}, end{};
    };

    struct AreaResult {
        double area = 0.0;
        double perimeter = 0.0;
        glm::vec3 normal{};
        std::vector<glm::vec3> vertices;
    };

    static DistanceResult MeasureDistance(const glm::vec3& start,
                                          const glm::vec3& end);

    static std::optional<AreaResult> MeasureArea(
        const std::vector<glm::vec3>& vertices);
};

```
