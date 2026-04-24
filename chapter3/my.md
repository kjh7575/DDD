---

```mermaid
flowchart TD
    subgraph FMEA_AGG["FMEA Analysis Aggregate"]
        FMEA["FmeaAnalysis <<Aggregate Root>>"]

        ITEM["FmeaItem"]
        FUNC["FunctionAnalysis"]
        FM["FailureMode"]
        EFFECT["Effect"]
        CAUSE["Cause"]
        CONTROL["CurrentControl"]
        RISK["RiskAssessment <<Value Object>>"]
        ACTION["RecommendedAction"]

        FMEA --> ITEM
        ITEM --> FUNC
        FUNC --> FM
        FM --> EFFECT
        FM --> CAUSE
        CAUSE --> CONTROL
        FM --> RISK
        FM --> ACTION
    end

    PART["Part Aggregate"]
    USER["Member/User Aggregate"]
    ECR["ECR Aggregate"]

    FMEA -. "partId 참조" .-> PART
    ACTION -. "ownerId 참조" .-> USER
    FMEA -. "relatedEcrId 참조" .-> ECR
```
```mermaid
classDiagram
    direction TB

    class FmeaAnalysis {
        <<Aggregate Root>>
        -FmeaId id
        -String title
        -FmeaType type
        -FmeaStatus status
        -List~FmeaItem~ items
        +addItem(FmeaItem item)
        +addFunction(FmeaItemId itemId, FunctionAnalysis function)
        +addFailureMode(FmeaItemId itemId, FunctionId functionId, FailureMode failureMode)
        +changeRiskAssessment(FailureModeId failureModeId, RiskAssessment risk)
        +addRecommendedAction(FailureModeId failureModeId, RecommendedAction action)
        +approve()
        +close()
    }

    class FmeaItem {
        -FmeaItemId id
        -String name
        -ItemType itemType
        -List~FunctionAnalysis~ functions
    }

    class FunctionAnalysis {
        -FunctionId id
        -String description
        -List~FailureMode~ failureModes
    }

    class FailureMode {
        -FailureModeId id
        -String description
        -List~Effect~ effects
        -List~Cause~ causes
        -RiskAssessment riskAssessment
        -List~RecommendedAction~ actions
    }

    class Cause {
        -CauseId id
        -String description
        -List~CurrentControl~ controls
    }

    class Effect {
        -String description
    }

    class CurrentControl {
        -String description
        -ControlType type
    }

    class RiskAssessment {
        <<Value Object>>
        -int severity
        -int occurrence
        -int detection
        +rpn() int
    }

    class RecommendedAction {
        -ActionId id
        -String description
        -MemberId ownerId
        -LocalDate dueDate
        -ActionStatus status
        +complete()
        +cancel()
    }

    FmeaAnalysis "1" --> "*" FmeaItem
    FmeaItem "1" --> "*" FunctionAnalysis
    FunctionAnalysis "1" --> "*" FailureMode
    FailureMode "1" --> "*" Effect
    FailureMode "1" --> "*" Cause
    Cause "1" --> "*" CurrentControl
    FailureMode "1" --> "1" RiskAssessment
    FailureMode "1" --> "*" RecommendedAction
```

FmeaAnalysis = 애그리거트 루트  
FmeaItem, FunctionAnalysis, FailureMode, Cause, Effect, Action = 내부 구성요소  
RiskAssessment = Value Object  
Part, User, ECR = 다른 애그리거트라서 ID로 참조

repository는 루트 단위로 둔다.
```mermaid
flowchart LR
    APP["Application Service"]
    REPO["FmeaAnalysisRepository"]
    ROOT["FmeaAnalysis << Aggregate Root >>"]
    DB[("FMEA tables")]

    APP --> REPO
    REPO --> ROOT
    REPO --> DB
```

## FMEA Analysis가 루트인 이유?

- 승인된 FMEA는 실패모드를 추가/수정할 수 없다.
- RiskAssessment는 severity, occurrence, detection이 1~10이어야 한다.
- FailureMode가 추가되면 위험도 계산이 가능해야 한다.
- RecommendedAction은 담당자, 기한, 상태 규칙을 가져야 한다.
- FMEA가 Close 상태면 Action을 추가할 수 없다.

## 애그리거트 -  팩토리로 사용하는 부분

1. fmea 생성
```java
public static FmeaAnalysis createPfmea(
        FmeaId id,
        String title,
        PartId targetPartId
) {
    return new FmeaAnalysis(id, title, FmeaType.PFMEA, targetPartId);
}
```

2. 실패 모드 생성 - Effect, Cause, RiskAssessment가 따라옴
```java
public class FailureModeFactory {
    public static FailureMode create(
            String description,
            List<Effect> effects,
            List<Cause> causes,
            RiskAssessment riskAssessment
    ) {
        return new FailureMode(description, effects, causes, riskAssessment);
    }
}
```
