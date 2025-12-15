

### Description
CrossSectionCommand lets the user define a cross-section line and generates a terrain profile from the point cloud.
It returns a CuttingPlane (for CrossSection camera setup) and a list of profile points (highest points along P1 → P2) used to render the profile line.

### Data it outputs (for rendering)

**CuttingPlane** (`m_activePlane`)  
- `center`  
- `point1`  
- `point2`  
- `normal`  
- `extentSize`  
- `zoomLevel`  
- `thickness`  
- `isBuilt`  

**CrossSectionResult** (`m_lastResult`)  
- `_points` (ordered points forming the terrain profile polyline: highest points along P1 -> P2 near the plane) 

```
//  Data Structure for the Slicing Plane
struct CuttingPlane {
    glm::vec3 point1{0.0f, 0.0f, 0.0f};
    glm::vec3 point2{0.0f, 0.0f, 0.0f};
    glm::vec3 center{0.0f, 0.0f, 0.0f};
    glm::vec3 normal{0.0f, 0.0f, 1.0f};

    // width and height of the plane 
    glm::vec2 extentSize = glm::vec2(100.0f, 80.0f);
    float thickness;
    float zoomLevel = 1.0f;
    bool isBuilt; // isHaveEnought 
};

// Contain points data that in Cutting plane
struct CrossSectionResult {     
        std::vector<Point>  _points; // highest points along P1->P2
};

```

```mermaid
classDiagram
    class ICommand {
        <<interface>>
        +OnStart()
        +OnEnd()
        +OnCancel()
        +OnInput(input: IInput): CommandStatus
    }

    class AbstractMouseCommand {
        <<abstract>>
        +OnInput(input: IInput): CommandStatus
        #OnClick(pos: vec2): CommandStatus
        #OnWheelInput(delta: float): CommandStatus
        #OnHoverInput(pos: vec2): CommandStatus
        #OnDrag(pos: vec2): CommandStatus
    }

    class PickingState {
        <<enumeration>>
        WaitingForFirstPoint
        WaitingForNextPoint
        Completed
    }

    class CoordinatePickingCommand {
        <<abstract>>
        #bool m_enableHover
        #PickingState m_state
        #glm::vec3 m_P1
        #ResetPickingState(): void
        #OnPointPicked(worldPoint: glm::vec3): CommandStatus
        #OnHoverPoint(worldPoint: glm::vec3): CommandStatus
        #OnClick(pos: vec2): CommandStatus
        #OnHoverInput(pos: vec2): CommandStatus
        #OnDrag(pos: vec2): CommandStatus
    }

    ICommand <|-- AbstractMouseCommand
    AbstractMouseCommand <|-- CoordinatePickingCommand
    CoordinatePickingCommand --> PickingState

    class CuttingPlane {
        +glm::vec3 point1
        +glm::vec3 point2
        +glm::vec3 center
        +glm::vec3 normal
        +glm::vec2 extentSize
        +float thickness
        +float zoomLevel
        +bool isBuilt
    }

    class CrossSectionResult {
        +std::vector~Point~ _points // profile Points, highest along the line P1 -> P2
    }

    class CrossSectionService {
        +ApplyCrossSection(plane: CuttingPlane, cloud: PointCloudModel): CrossSectionResult
        +UpdateCrossSectionView(data: CrossSectionResult, plane: CuttingPlane): void
    }

    class CommandContext {
        +RubberBandManager* rubberBand
        +CrossSectionService* crossSectionService
        +GetPointCloud(): PointCloudModel*
    }

    class CrossSectionCommand {
        -bool m_hasP1
        -glm::vec2 m_baseExtent
        -CuttingPlane m_activePlane
        -CrossSectionResult m_lastResult
        -ApplyCrossSection(): void
        #OnHoverPoint(hoverPoint: glm::vec3): CommandStatus
        #OnPointPicked(pickedPoint: glm::vec3): CommandStatus
    }

    CoordinatePickingCommand <|-- CrossSectionCommand
    CoordinatePickingCommand --> CommandContext
    CrossSectionCommand --> CrossSectionService
    CrossSectionService --> CrossSectionResult
    CrossSectionService --> CuttingPlane

```


```
class CrossSectionCommand : public CoordinatePickingCommand {
private:
    bool m_hasP1 = false;

    glm::vec2 m_baseExtent{100.0f, 80.0f};

    CuttingPlane m_activePlane;
    CrossSectionResult m_lastResult;
    void ApplyCrossSection() {
        m_lastResult = m_context.crossSectionService->ApplyCrossSection(
            m_activePlane, *m_context.GetPointCloud());

        m_context.crossSectionService->UpdateCrossSectionView(
            m_lastResult, m_activePlane);
    }
protected:
    CommandStatus OnHoverPoint(const glm::vec3& hoverPoint) override {
        if (m_state == PickingState::WaitingForNextPoint) {
            m_context.rubberBand->ShowLine(m_P1, hoverPoint);
        }
        return CommandStatus::Finished;
    }
    CommandStatus OnPointPicked(const glm::vec3& pickedPoint) override {
        CuttingPlane& plane = m_activePlane;

        if (m_state == PickingState::WaitingForFirstPoint) {
            plane.center = pickedPoint;
            m_state = PickingState::WaitingForNextPoint;
            m_hasP1 = false;
            return CommandStatus::None;
        }

        if (m_state == PickingState::WaitingForNextPoint && !m_hasP1) {
            plane.point1 = pickedPoint;
            m_hasP1 = true;
            return CommandStatus::None;
        }

        if (m_state == PickingState::WaitingForNextPoint && m_hasP1) {
            if (!IsOppositeSide(...)) return CommandStatus::None;

            plane.point2 = pickedPoint;
            plane.normal = dot(point2-point1, Vector.up)
            plane.isBuilt = true;
            ApplyCrossSection();
            m_state = PickingState::Completed;
            return CommandStatus::Finished;
        }

        // Restart
        if (m_state == PickingState::Completed) {
            OnStart();
        }

        return CommandStatus::None;
    }
public:
    void OnStart() override {
        ResetPickingState();
        m_P1 = glm::vec3(0.0f);
        m_P2 = glm::vec3(0.0f);
        m_context.rubberBand->Clear();
        
        m_activePlane = CuttingPlane();
        m_enableHover = true;
    }

    void OnEnd() override {
        m_context.rubberBand->Clear();
    }

};

```

```
class CrossSectionService {
public:
     CrossSectionResult ApplyCrossSection(const CuttingPlane& plane,
                                         const PointCloudModel& sourceCloud)
    {
        return BuildProfilePoint(plane, sourceCloud);
    }


    void UpdateCrossSectionView(const CrossSectionResult& data, const CuttingPlane& cuttingPlane) {
       // Update Cross
    }
private:
    CrossSectionResult BuildProfilePoint(const CuttingPlane& plane, const PointCloudModel& srcCloud){
        CrossSectionResult result;
        //find highest points along the line P1 -> P2
        return result;
    }
};

// CommandContext remains the same
struct CommandContext {
    RubberBandManager* rubberBand;
    CrossSectionService* crossSectionService;
    PointCloudModel* GetPointCloud();
    glm::vec3 pickNearestPoint(double x, double y);
    //...
};

```



Tasks break down:
1. Define behavior + class design: 1 day < Done

2. Write code, Implement CrossSectionCommand logic (opposite-side validation, plane build): 1 day

3. Implement profile extraction in CrossSectionService (highest height along P1->P2, thickness filter): 2 day

4. Cross-section view update. Send CuttingPlane and CrossSectionResult to CrossSectionCanvas View: 1 day

5. Render the profile lines, display infomations: 2 day

6. Testing and Feedback Backup: 3 day

Total: 10 days
