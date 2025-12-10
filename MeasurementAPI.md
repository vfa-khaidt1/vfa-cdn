
Class for DistanceMeasureCommand
```
class DistanceMeasureCommand : public ICommand {
private:
    bool m_hasFirstPoint = false;
    glm::vec3 m_firstPoint{0};
    glm::vec3 m_currentHoverPoint{0};

public:
    using ICommand::ICommand;
    const char* GetName() const override { return "Distance Measure"; }

    void OnStart() override {
        m_hasFirstPoint = false;
        m_context.rubberBand->Clear();
    }

    void OnEnd() override {
        m_context.rubberBand->Clear();
    }

    CommandStatus OnMouseMove(const MouseEvent& e) override {
        // Always track the hover point for rubber banding
        m_currentHoverPoint = m_context.pickNearestPoint(e.x, e.y);

        if (m_hasFirstPoint) {
            m_context.rubberBand->ShowLine(m_firstPoint, m_currentHoverPoint);
        }
        return CommandStatus::None;
    }

    CommandStatus OnMouseUp(const MouseEvent& e) override {
        // KEY LOGIC: Only react if it was a distinct CLICK, not a DRAG.
        if (e.button == mouseButton && e.gesture == GestureType::Click) {
            
            glm::vec3 hitPoint =  m_context.pickNearestPoint(e.x, e.y);

            if (!m_hasFirstPoint) {
                // Set Start Point
                m_firstPoint = hitPoint;
                m_hasFirstPoint = true;
            } else {
                // Set End Point & Calculate
                auto result = MeasurementService::MeasureDistance(m_firstPoint, hitPoint);                

                // Update distance
                return CommandStatus::Finished; 
                
            }
        }
        return CommandStatus::None;
    }
};
```

AreaMeasurement Commmand
```
class AreaMeasureCommand : public ICommand {
private:
    std::vector<glm::vec3> m_points;

public:
    using ICommand::ICommand;
    const char* GetName() const override { return "Area Measure"; }

    void OnStart() override {
        m_points.clear();
        m_context.rubberBand->Clear();
    }

    CommandStatus OnMouseMove(const MouseEvent& e) override {
        if (!m_points.empty()) {
            glm::vec3 hover =  m_context.pickNearestPoint(e.x, e.y);
            m_context.rubberBand->ShowPolyline(m_points, hover);
        }
        return CommandStatus::Running;
    }

    CommandStatus OnMouseUp(const MouseEvent& e) override {
        if (e.button != mouseButton || e.gesture != GestureType::Click)
          return CommandStatus::None;
 
        glm::vec3 hitPoint = m_context.pickNearestPoint(e.x, e.y);

        if (!m_points.Contains(hitPoint) ) 
             m_points.push_back(hitPoint);

            if (m_points.size() >= 3) {
                auto res = MeasurementService::MeasureArea(m_points);
                if (res) {
                    // Update area here
                     printf("Area: %f, res->area);
                }
            }
            return CommandStatus::Finished;
        }

        return CommandStatus::None;
    }
};
```
MeasurementService

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

    static DistanceResult MeasureDistance(const glm::vec3& start, const glm::vec3& end);

    static std::optional<AreaResult> MeasureArea(const std::vector<glm::vec3>& vertices);
};
```
