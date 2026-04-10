
**FMEA의 유스케이스**
```
“FMEA 분석서를 생성하고, 실패모드와 위험평가를 등록하고, 개선조치를 추가한다.”
```

**표현 영역**
- `FmeaController`
- `CreateFmeaRequest`
- `AddFailureModeRequest`
- `AddActionRequest`
- `FmeaResponse`

**응용 영역**
- `CreateFmeaService`
- `AddFailureModeService`
- `AddActionService`
- `ApproveFmeaService`

**도메인 영역**
- `FmeaAnalysis`
- `FmeaItem`
- `FunctionAnalysis`
- `FailureModeAnalysis`
- `RecommendedAction`
- `RiskAssessment`
- `FmeaStatus`
- `FmeaType`
- `ItemType`
- `FmeaRepository 인터페이스`

**인프라스트럭처 영역**
- `JpaFmeaRepository`
- `FmeaJpaEntity`
- `FmeaEntityMapper`
- `KafkaRiskEventPublisher`
- `SmtpNotificationSender`
- `ExternalPartClient`
- `DB, Kafka, SMTP, 외부 PLM/QMS 연동 구현`

```mermaid
flowchart TD
    UI[사용자 또는 외부 시스템]

    subgraph P[표현 영역]
        FC[FmeaController]
        CFR[CreateFmeaRequest DTO]
        AFR[AddFailureModeRequest DTO]
        AAR[AddActionRequest DTO]
        FR[FmeaResponse DTO]
    end

    subgraph A[응용 영역]
        CFS[CreateFmeaService]
        AFMS[AddFailureModeService]
        AAS[AddActionService]
        APS[ApproveFmeaService]
    end

    subgraph D[도메인 영역]
        FA[FmeaAnalysis]
        FI[FmeaItem]
        FUN[FunctionAnalysis]
        FM[FailureModeAnalysis]
        RA[RiskAssessment]
        ACT[RecommendedAction]
        FT[FmeaType]
        FS[FmeaStatus]
        IT[ItemType]
        REP[FmeaRepository 인터페이스]
    end

    subgraph I[인프라스트럭처 영역]
        JREP[JpaFmeaRepository]
        JE[FmeaJpaEntity]
        MAP[FmeaEntityMapper]
        KAFKA[KafkaRiskEventPublisher]
        SMTP[SmtpNotificationSender]
        EXT[ExternalPartClient]
    end

    UI --> FC
    FC --> CFS
    FC --> AFMS
    FC --> AAS
    FC --> APS

    CFS --> FA
    CFS --> REP

    AFMS --> REP
    AFMS --> FM
    AFMS --> RA

    AAS --> REP
    AAS --> ACT

    APS --> REP
    APS --> FA

    JREP -.implements.-> REP

    JREP --> JE
    JREP --> MAP
    AAS --> KAFKA
    APS --> SMTP
    AFMS --> EXT
```

간단한 버전
```mermaid
flowchart LR
    subgraph 표현
        C[Controller]
        DTO[Request/Response DTO]
    end

    subgraph 응용
        S[Application Service]
    end

    subgraph 도메인
        AGG[FmeaAnalysis Aggregate]
        FM[FailureModeAnalysis]
        RA[RiskAssessment]
        ACT[RecommendedAction]
        REPO[FmeaRepository Interface]
    end

    subgraph 인프라스트럭처
        JPA[JpaFmeaRepository]
        DB[(DB)]
        MQ[Kafka / SMTP / External API]
    end

    C --> S
    DTO --> S
    S --> AGG
    S --> REPO
    JPA -.implements.-> REPO
    JPA --> DB
    S --> MQ
```

---
**클래스 구조**

```mermaid
classDiagram
    direction LR

    class FmeaController {
        +createFmea(request)
        +addFailureMode(fmeaId, request)
        +addAction(fmeaId, request)
        +approve(fmeaId)
    }

    class CreateFmeaService {
        +handle(command)
    }

    class AddFailureModeService {
        +handle(command)
    }

    class AddActionService {
        +handle(command)
    }

    class ApproveFmeaService {
        +handle(command)
    }

    class FmeaAnalysis {
        +addItem(item)
        +addFailureMode(itemId, functionId, failureMode)
        +addAction(failureModeId, action)
        +approve()
    }

    class FmeaItem
    class FunctionAnalysis
    class FailureModeAnalysis
    class RecommendedAction
    class RiskAssessment {
        +rpn() int
    }

    class FmeaType
    class FmeaStatus
    class ItemType

    class FmeaRepository {
        <<interface>>
        +save(fmea)
        +findById(id)
    }

    class JpaFmeaRepository {
        +save(fmea)
        +findById(id)
    }

    FmeaController --> CreateFmeaService
    FmeaController --> AddFailureModeService
    FmeaController --> AddActionService
    FmeaController --> ApproveFmeaService

    CreateFmeaService --> FmeaAnalysis
    AddFailureModeService --> FmeaAnalysis
    AddActionService --> FmeaAnalysis
    ApproveFmeaService --> FmeaAnalysis

    CreateFmeaService --> FmeaRepository
    AddFailureModeService --> FmeaRepository
    AddActionService --> FmeaRepository
    ApproveFmeaService --> FmeaRepository

    JpaFmeaRepository ..|> FmeaRepository

    FmeaAnalysis --> FmeaItem
    FmeaItem --> FunctionAnalysis
    FunctionAnalysis --> FailureModeAnalysis
    FailureModeAnalysis --> RecommendedAction
    FailureModeAnalysis --> RiskAssessment
    FmeaAnalysis --> FmeaType
    FmeaAnalysis --> FmeaStatus
    FmeaItem --> ItemType
```

