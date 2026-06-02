# Free6GC Initial Prototype Design

Status: initial review draft

Source proposal: `proposal.md`

This document defines the first executable prototype design for a Free6GC core network based on the Free5GC codebase. It is intentionally scoped to a narrow initial-registration vertical slice. The goal is to prove the proposed 6G architecture through a deep refactor, not through an overlay around the existing monolithic AMF/SMF.

## 1. Codex Initial Context Prompt

Use this prompt at the start of each Codex session working on this project:

```text
We are building a 6G core network prototype, called Free6GC, based on our fork of Free5GC.

Read these files first:
- /root/proj/go/orange-prototype-wiki/proposal.md
- /root/proj/go/orange-prototype-wiki/free6gc-initial-design.md
- Relevant Free5GC source under /root/proj/go/free5gc-compose/base/free5gc

Architectural direction:
- This is a deep refactor, not an overlay prototype.
- Keep UE/gNB compatibility with current Free5GC/UERANSIM for milestone 1.
- SRF terminates NGAP/SCTP and forwards raw NAS PDUs plus RAN metadata to GCF.
- GCF owns procedure transaction state and orchestrates atomic services.
- Atomic services expose gRPC/Protobuf APIs.
- Atomic services keep service-local in-memory UE context for milestone 1.
- Source .proto contracts live in free6gc-system/specs/proto.
- Generated Go API clients should be published through a shared module such as free6gc-api-go.
- GCF uses static config for service endpoints in milestone 1.

Milestone 1 scope:
- Implement initial registration only.
- Do not implement PDU session establishment unless explicitly assigned.
- Do not keep AMF as the runtime owner of registration logic.
- Extract or reuse Free5GC code inside new service boundaries where possible.

Primary principle:
Keep procedure logic in GCF, protocol routing in SRF, NAS encode/decode in NAS Codec Service, NAS crypto/security context in NAS Security Service, authentication/SEAF in Authentication Service, subscription access in Subscription Service, access restriction in Mobility Restriction Service, and GUTI/TA/registration context in Mobility Management Service.

Before editing, inspect the existing code path and identify the exact Free5GC functions being moved or adapted. Preserve unrelated user changes.
```

## 2. Architecture Decisions

1. Prototype style: deep refactor, not overlay.
2. First milestone: narrow initial-registration vertical slice.
3. External compatibility: keep Free5GC/UERANSIM NGAP/NAS compatibility.
4. Runtime entry point: SRF terminates NGAP/SCTP.
5. NAS handling boundary: SRF forwards raw NAS PDUs plus metadata to GCF.
6. Procedure ownership: GCF owns procedure transaction state.
7. Internal API style: gRPC/Protobuf.
8. Contract authority: `.proto` source files live in `free6gc-system/specs/proto`.
9. Generated clients: a shared Go API module, for example `free6gc-api-go`.
10. Endpoint resolution: static config for milestone 1.
11. NAS protocol boundary: NAS Codec Service handles NAS encode/decode.
12. NAS security boundary: NAS Security Service handles NAS security context, integrity, ciphering, and NAS counts.
13. Authentication boundary: Authentication Service owns Authentication/SEAF responsibilities.
14. Atomic service state: service-local in-memory UE context for milestone 1.
15. PCF and AM Policy Control are deferred from the happy-path initial registration.
16. Authentication and Subscription Services use static configured endpoints for Free5GC AUSF/UDM/UDR.
17. `free6gc-api-go` is generated, committed, and consumed as its own sibling repo.
18. Local multi-repo development uses `go.work` or temporary local `replace`; committed PRs pin `free6gc-api-go` by tag or commit pseudo-version.
19. `free6gc-system` provides `go.work.example`, not a committed active `go.work`.
20. Milestone 1 uses plaintext gRPC on isolated Docker networks; no mTLS.
21. GCF transaction logs are in memory, with optional debug/replay export.
22. Debug snapshot/reset APIs are allowed for milestone 1 and gated by config.
23. E2E harness has two layers: synthetic SRF replay and full UERANSIM through SRF.
24. Synthetic SRF replay uses real encoded NAS bytes, initially generated provisionally, then replaced by golden UERANSIM trace JSON.
25. Golden trace replay covers only the successful initial-registration path.
26. Milestone 1 stops at Registration Complete committed.
27. Happy-path Registration Request assumes SUCI is present.
28. Milestone 1 supports 5G-AKA only.
29. Authentication Service returns SUPI and KSEAF; NAS Security Service derives KAMF and NAS keys.
30. NAS Security Service selects NAS algorithms.
31. NAS Security contexts use generated `security_context_id`.
32. Mobility Management committed registration context uses SUPI as primary key.
33. GCF exposes one generic ingress API: `StartOrContinueProcedure`.
34. GCF allocates `procedure_id` and correlates uplink events using RAN association metadata.
35. SRF does not store procedure correlation state in milestone 1.
36. SRF downlink delivery is addressed by RAN routing metadata.
37. SRF, GCF, and all atomic services are single-instance in milestone 1.
38. Milestone 1 uses Docker Compose only.

## 3. 4+1 Architecture Views

The proposal already defines the logical view: a hierarchical architecture with Routing Services, Global Coordination Services, and Atomic Modular Services. This document adds the development, process, physical, and scenario views for the first prototype.

```mermaid
flowchart TB
    CENTER[Free6GC Initial Registration Prototype]
    LOGICAL[Logical View<br/>SRF, GCF, AMCF atomic services]
    DEV[Development View<br/>hybrid multi-repo and proto contracts]
    PROCESS[Process View<br/>initial registration runtime messages]
    PHYSICAL[Physical View<br/>Docker Compose deployment]
    SCENARIO[Scenario View<br/>success and failure registration flows]

    CENTER --> LOGICAL
    CENTER --> DEV
    CENTER --> PROCESS
    CENTER --> PHYSICAL
    CENTER --> SCENARIO
```

### 3.1 Logical View Summary

```mermaid
flowchart TB
    subgraph ACCESS[Access side]
        UE[UE]
        GNB[gNB/UERANSIM]
        UE --> GNB
    end

    subgraph ROUTING[Routing Service Layer]
        SRF[SRF<br/>Signaling Routing Function]
    end

    subgraph COORD[Global Coordination Layer]
        GCF[GCF<br/>initial registration procedure control]
    end

    subgraph ATOMIC[Atomic Modular Services]
        NASC[NAS Codec Service]
        NASS[NAS Security Service]
        AUTH[Authentication Service]
        SUB[Subscription Service]
        MR[Mobility Restriction Service]
        MM[Mobility Management Service]
    end

    subgraph BACKEND[Existing Free5GC backend services]
        AUSF[AUSF]
        UDM[UDM/UDR]
    end

    GNB -->|NGAP/SCTP| SRF
    SRF -->|raw NAS + metadata| GCF
    GCF --> NASC
    GCF --> NASS
    GCF --> AUTH
    GCF --> SUB
    GCF --> MR
    GCF --> MM
    AUTH --> AUSF
    SUB --> UDM
```

## 4. Development View

### 4.1 Repository Model

Use a hybrid multi-repo model. These are sibling repositories in the same workspace, not parent/child directories. `free6gc-system` is the coordination and integration authority, but it does not contain the service implementation repositories.

```mermaid
flowchart LR
    subgraph WORKSPACE[Workspace, for example /root/proj/go]
        direction TB
        SYS[free6gc-system<br/>specs, deployments, tests, prompts]
        API[free6gc-api-go<br/>generated API module]
        SRF[free6gc-srf]
        GCF[free6gc-gcf]
        NASC[free6gc-nas-codec-service]
        NASS[free6gc-nas-security-service]
        AUTH[free6gc-authentication-service]
        SUB[free6gc-subscription-service]
        MR[free6gc-mobility-restriction-service]
        MM[free6gc-mobility-management-service]
    end
```

Recommended workspace shape:

```text
/root/proj/go/
  free6gc-system/                  # sibling repo, owns specs/deployments/tests/prompts
  free6gc-api-go/                  # sibling repo, generated from free6gc-system/specs/proto
  free6gc-srf/                     # sibling repo
  free6gc-gcf/                     # sibling repo
  free6gc-nas-codec-service/       # sibling repo
  free6gc-nas-security-service/    # sibling repo
  free6gc-authentication-service/  # sibling repo
  free6gc-subscription-service/    # sibling repo
  free6gc-mobility-restriction-service/
  free6gc-mobility-management-service/
```

The dependency direction is contract-first, not directory-parent-first:

```mermaid
flowchart TB
    SYS[free6gc-system]
    PROTO[specs/proto<br/>source contracts]
    TESTS[tests<br/>contract and e2e]
    API[free6gc-api-go<br/>generated bindings]
    SERVICES[service repos<br/>srf, gcf, nas-security, nas-codec, auth, subscription, mobility]

    SYS --> PROTO
    SYS --> TESTS
    PROTO --> API
    API --> SERVICES
    TESTS --> SERVICES
```

### 4.2 Source Extraction Map

Initial Free5GC reference points:

```mermaid
flowchart LR
    subgraph AMF[Free5GC AMF reference code]
        NGAP[NFs/amf/internal/ngap<br/>NGAP/SCTP and NAS transport]
        NAS[NFs/amf/internal/nas<br/>NAS dispatch]
        GMMMSG[NFs/amf/internal/gmm/message<br/>NAS builders]
        SEC[NFs/amf/internal/nas/nas_security<br/>NAS protection]
        UECTX[NFs/amf/internal/context/amf_ue.go<br/>UE security/context fields]
        GMM[NFs/amf/internal/gmm/handler.go<br/>registration procedure]
        SBI[NFs/amf/internal/sbi/consumer<br/>AUSF/UDM client reference]
    end

    NGAP --> SRF[free6gc-srf]
    NAS --> NASC[free6gc-nas-codec-service]
    GMMMSG --> NASC
    SEC --> NASS[free6gc-nas-security-service]
    UECTX --> NASS
    GMM --> GCF[free6gc-gcf]
    GMM --> MM[free6gc-mobility-management-service]
    GMM --> MR[free6gc-mobility-restriction-service]
    SBI --> AUTH[free6gc-authentication-service]
    SBI --> SUB[free6gc-subscription-service]
    SBI --> MR
```

The first refactor should not copy an entire AMF into every service. Each service should import or extract only the code needed for its own responsibility.

### 4.3 Ownership Rules

```mermaid
mindmap
  root((Runtime Ownership))
    SRF
      SCTP connection to gNB
      NGAP association state
      RAN UE identifiers
      Downlink NAS delivery
      No NAS business interpretation
    GCF
      Procedure ID
      Initial registration transaction
      Step ordering
      Branch decisions
      Retry and timeout policy
      Accept or reject decision
    NAS Codec Service
      NAS parsing
      NAS construction
      Typed decoded facts
      No cryptographic state
    NAS Security Service
      NAS security context
      Algorithm selection
      KAMF-derived NAS keys
      Uplink and downlink NAS counts
      Integrity and ciphering
    Authentication Service
      AUSF access
      Authentication start
      RES star and HRES star validation
      AUSF confirmation
      Resynchronization handling
      SUPI and KSEAF result
    Subscription Service
      UDM and UDR access
      Subscribed NSSAI
      AM data
      Optional local cache
    Mobility Restriction Service
      RAT access decision
      TA access decision
      Slice access decision
      Restriction evaluation
    Mobility Management Service
      GUTI allocation
      TAI list allocation
      Registration timers
      Registration context commit
```

### 4.4 Spec Management for SDD

Spec-driven development uses `free6gc-system` as the source of truth for cross-repo behavior, public contracts, integration tests, and Codex work decomposition. Service repos own only service-local specs that do not change public contracts or another repo's behavior.

```mermaid
flowchart TB
    REQ[New requirement]
    CROSS{Touches more than one repo<br/>or public API/deployment/e2e behavior?}
    SYSTEM[Write system spec<br/>free6gc-system/specs/requirements/*.md]
    LOCAL[Write local spec<br/>owning-service/specs/requirements/*.md]
    PROTO{Requires gRPC contract change?}
    PROTOS[Update source proto<br/>free6gc-system/specs/proto/*.proto]
    APIGO[Regenerate/publish<br/>free6gc-api-go]
    FANOUT[Start repo-specific Codex sessions<br/>from each implementation repo]
    LOCALCODE[Implement and test<br/>inside owning service repo]
    CONTRACT[Run contract tests<br/>free6gc-system/tests/contract]
    E2E[Run e2e tests<br/>free6gc-system/tests/e2e]

    REQ --> CROSS
    CROSS -->|yes| SYSTEM
    CROSS -->|no| LOCAL
    SYSTEM --> PROTO
    PROTO -->|yes| PROTOS --> APIGO --> FANOUT
    PROTO -->|no| FANOUT
    FANOUT --> CONTRACT --> E2E
    LOCAL --> LOCALCODE
    LOCALCODE -->|if behavior becomes cross-repo| SYSTEM
```

Recommended layout:

```text
free6gc-system/
  specs/
    requirements/
      initial-registration.md
      auth-failure.md
      access-restricted.md
    scenarios/
      initial-registration-success.md
    decisions/
      adr-0001-hybrid-multirepo.md
    proto/
      common.proto
      srf_gcf.proto
      nas_codec.proto
      nas_security.proto
      authentication.proto
      subscription.proto
      mobility_restriction.proto
      mobility_management.proto
  tests/
    contract/
    e2e/
  deployments/
    compose/
  prompts/
    cross-repo-task-template.md

free6gc-nas-security-service/
  specs/
    requirements/
      replay-protection.md
      security-context-lifecycle.md
```

Spec placement rules:

- Cross-repo behavior: `free6gc-system/specs/requirements/`.
- Public gRPC/protobuf API: `free6gc-system/specs/proto/`.
- Integration scenarios: `free6gc-system/specs/scenarios/`.
- Architecture decisions: `free6gc-system/specs/decisions/`.
- Contract and E2E acceptance tests: `free6gc-system/tests/`.
- Milestone 1 deployment manifests: `free6gc-system/deployments/compose/`.
- Purely local service behavior: owning service repo under `specs/requirements/`.
- Internal refactor with no behavior change: a service-local note or issue is enough; no system spec is required.

For cross-repo requirements, start the first Codex session in `free6gc-system`, write or update the system spec, update proto contracts if needed, then fan out implementation sessions to the service repos. For local requirements, start Codex in the owning service repo and keep the spec there.

## 5. Process View

### 5.1 Initial Registration Sequence

```mermaid
sequenceDiagram
    autonumber
    participant UE as UE
    participant GNB as gNB/UERANSIM
    participant SRF as SRF<br/>NGAP Termination
    participant GCF as GCF<br/>Procedure Orchestrator
    participant NASC as NAS Codec Service
    participant AUTH as Authentication Service
    participant NASS as NAS Security Service
    participant SUB as Subscription Service
    participant MR as Mobility Restriction Service
    participant MM as Mobility Management Service

    UE->>GNB: NAS Registration Request
    GNB->>SRF: NGAP InitialUEMessage<br/>raw NAS Registration Request
    SRF->>GCF: StartOrContinueProcedure<br/>RAN metadata + raw NAS

    GCF->>NASC: DecodeUplinkNas(raw NAS)
    NASC-->>GCF: RegistrationRequest facts<br/>SUCI/SUPI, reg type, capabilities

    GCF->>AUTH: StartAuthentication(SUCI/SUPI, serving network)
    AUTH-->>GCF: Auth challenge<br/>RAND, AUTN, HXRES*, ABBA, ngKSI

    GCF->>NASC: BuildAuthenticationRequest(RAND, AUTN, ABBA, ngKSI)
    NASC-->>GCF: raw NAS Authentication Request
    GCF->>SRF: SendDownlinkNas(raw NAS)
    SRF->>GNB: NGAP DownlinkNASTransport
    GNB->>UE: NAS Authentication Request

    UE->>GNB: NAS Authentication Response
    GNB->>SRF: NGAP UplinkNASTransport<br/>raw NAS Authentication Response
    SRF->>GCF: ContinueProcedure(raw NAS)

    GCF->>NASC: DecodeUplinkNas(raw NAS)
    NASC-->>GCF: AuthenticationResponse facts<br/>RES*
    GCF->>AUTH: ConfirmAuthentication(RES*)
    AUTH-->>GCF: Auth success<br/>SUPI, KSEAF

    GCF->>NASS: CreateNasSecurityContext(SUPI, KSEAF, UE capabilities)
    NASS-->>GCF: selected algorithms, security context id

    GCF->>NASC: BuildSecurityModeCommand(selected algorithms)
    NASC-->>GCF: raw NAS Security Mode Command
    GCF->>NASS: ProtectDownlinkNas(context id, raw NAS)
    NASS-->>GCF: protected NAS Security Mode Command
    GCF->>SRF: SendDownlinkNas(protected NAS)
    SRF->>GNB: NGAP DownlinkNASTransport
    GNB->>UE: NAS Security Mode Command

    UE->>GNB: NAS Security Mode Complete
    GNB->>SRF: NGAP UplinkNASTransport<br/>protected NAS
    SRF->>GCF: ContinueProcedure(protected NAS)

    GCF->>NASS: UnprotectUplinkNas(context id, protected NAS)
    NASS-->>GCF: raw NAS Security Mode Complete
    GCF->>NASC: DecodeUplinkNas(raw NAS)
    NASC-->>GCF: SecurityModeComplete facts

    GCF->>SUB: FetchRegistrationSubscriptionData(SUPI)
    SUB-->>GCF: subscription, subscribed NSSAI, DNN data

    GCF->>MR: EvaluateAccess(TAI, RAT, requested NSSAI, subscription)
    MR-->>GCF: access allowed, allowed NSSAI, restrictions

    GCF->>MM: AllocateRegistrationContext(SUPI, TAI, allowed NSSAI)
    MM-->>GCF: GUTI, TAI list, registration timers

    GCF->>NASC: BuildRegistrationAccept(GUTI, TAI list, allowed NSSAI, timers)
    NASC-->>GCF: raw NAS Registration Accept
    GCF->>NASS: ProtectDownlinkNas(context id, raw NAS)
    NASS-->>GCF: protected NAS Registration Accept
    GCF->>SRF: SendDownlinkNas(protected NAS)
    SRF->>GNB: NGAP DownlinkNASTransport
    GNB->>UE: NAS Registration Accept

    UE->>GNB: NAS Registration Complete
    GNB->>SRF: NGAP UplinkNASTransport<br/>protected NAS
    SRF->>GCF: ContinueProcedure(protected NAS)

    GCF->>NASS: UnprotectUplinkNas(context id, protected NAS)
    NASS-->>GCF: raw NAS Registration Complete
    GCF->>NASC: DecodeUplinkNas(raw NAS)
    NASC-->>GCF: RegistrationComplete facts

    GCF->>MM: CommitRegistration(SUPI, RAN routing context, completion)
    MM-->>GCF: registration committed
```

### 5.2 Communication Diagram

```mermaid
flowchart LR
    UE[UE] --> GNB[gNB/UERANSIM]
    GNB -->|NGAP/SCTP<br/>raw NAS container| SRF[SRF<br/>owns RAN association<br/>NGAP state]
    SRF -->|gRPC<br/>raw NAS + RAN metadata| GCF[GCF<br/>owns procedure transaction state]

    GCF -->|gRPC| NASC[NAS Codec Service<br/>NAS encode/decode only]
    GCF -->|gRPC| AUTH[Authentication Service<br/>auth/SEAF context]
    GCF -->|gRPC| NASS[NAS Security Service<br/>NAS security context]
    GCF -->|gRPC| SUB[Subscription Service<br/>UDM/SDM adapter context]
    GCF -->|gRPC| MR[Mobility Restriction Service<br/>access decision logic]
    GCF -->|gRPC| MM[Mobility Management Service<br/>GUTI/TA/registration context]

    AUTH -->|static configured endpoint| AUSF[AUSF]
    SUB -->|static configured endpoint| UDM[UDM/UDR]
```

### 5.3 Process Rules

- SRF must not branch on NAS registration semantics and must not own procedure correlation state in milestone 1.
- SRF forwards uplink NAS events with RAN metadata to the single GCF instance.
- GCF correlates uplink events using RAN association metadata until SUPI is known.
- GCF stores a `ran_routing_context` in procedure transaction state and passes it back to SRF for downlink NAS delivery.
- GCF must not parse ASN.1 NGAP.
- GCF should not import NAS bit-level encoding packages directly.
- GCF should be able to replay a procedure from a transaction log in tests.
- Atomic services should return typed outcomes, not direct NAS messages, unless their responsibility is NAS encoding/security.
- Every GCF to service request should carry `procedure_id`.
- Every service response should carry enough decision data for GCF to advance or reject the procedure.

## 6. Physical View

### 6.1 Milestone 1 Docker Compose Deployment

```mermaid
flowchart TB
    subgraph HOST[Host]
        subgraph RAN[Docker network: ran-core]
            UE[UERANSIM UE]
            GNB[UERANSIM gNB]
            UE --> GNB
        end

        subgraph CTRL[Docker network: free6gc-control]
            SRF[free6gc-srf<br/>NGAP/SCTP endpoint]
            GCF[free6gc-gcf<br/>static endpoint config]
            NASC[free6gc-nas-codec-service]
            NASS[free6gc-nas-security-service]
            AUTH[free6gc-authentication-service]
            SUB[free6gc-subscription-service]
            MR[free6gc-mobility-restriction-service]
            MM[free6gc-mobility-management-service]
            AUSF[free5gc-ausf]
            UDM[free5gc-udm]
            UDR[free5gc-udr]
            DB[(mongodb)]
        end
    end

    GNB -->|NGAP/SCTP| SRF
    SRF -->|gRPC| GCF
    GCF -->|gRPC| NASC
    GCF -->|gRPC| NASS
    GCF -->|gRPC| AUTH
    GCF -->|gRPC| SUB
    GCF -->|gRPC| MR
    GCF -->|gRPC| MM
    AUTH -->|static configured endpoint| AUSF
    SUB -->|static configured endpoint| UDM
    UDM --> UDR
    UDR --> DB
```

Milestone 1 should not run legacy `free5gc-amf` for the registration path. It may remain available only as a reference or comparison target.

SMF and UPF are not required for initial registration unless the scenario is extended to PDU session establishment.

### 6.2 Milestone 1 Deployment Constraints

```mermaid
flowchart TB
    COMPOSE[Milestone 1 runtime<br/>Docker Compose only]
    NET[Isolated Docker networks]
    GRPC[Plaintext gRPC<br/>no mTLS]
    SINGLE[Single instance per Free6GC service]
    SRF[Single SRF<br/>forwards NAS + RAN metadata]
    GCF[Single GCF<br/>owns correlation and transaction state]
    ATOMIC[Single atomic service instances<br/>service-local in-memory context]
    BACKEND[Existing Free5GC AUSF/UDM/UDR<br/>static configured endpoints]

    COMPOSE --> NET
    COMPOSE --> GRPC
    COMPOSE --> SINGLE
    SINGLE --> SRF
    SINGLE --> GCF
    SINGLE --> ATOMIC
    ATOMIC --> BACKEND
```

Multi-instance scaling, mTLS, NRF discovery, PCF policy integration, and post-registration UE context release are deferred from milestone 1.

## 7. Scenario View

### 7.1 Scenario: Successful Initial Registration

Preconditions:

- UERANSIM UE and gNB are configured with a test SUPI/SUCI compatible with subscriber data.
- SRF is reachable by gNB over NGAP/SCTP.
- GCF has static endpoints for all atomic services.
- Authentication Service can reach AUSF or equivalent authentication backend.
- Subscription Service can reach UDM/UDR or equivalent subscriber backend.
- Atomic services are empty or reset before test start.

Main flow:

```mermaid
stateDiagram-v2
    [*] --> RegistrationRequestReceived
    RegistrationRequestReceived --> AuthenticationStarted: decode registration request
    AuthenticationStarted --> AuthenticationChallengeSent: build/send auth request
    AuthenticationChallengeSent --> AuthenticationConfirmed: receive RES star and confirm
    AuthenticationConfirmed --> SecurityContextCreated: create NAS security context
    SecurityContextCreated --> SecurityModeCommandSent: build/protect/send SMC
    SecurityModeCommandSent --> SecurityModeCompleteReceived: unprotect/decode SMC complete
    SecurityModeCompleteReceived --> SubscriptionFetched: fetch registration subscription data
    SubscriptionFetched --> AccessEvaluated: evaluate TAI/RAT/slice restrictions
    AccessEvaluated --> RegistrationContextAllocated: allocate GUTI/TAI/timers
    RegistrationContextAllocated --> RegistrationAcceptSent: build/protect/send accept
    RegistrationAcceptSent --> RegistrationCompleteReceived: receive registration complete
    RegistrationCompleteReceived --> RegistrationCommitted: commit registration
    RegistrationCommitted --> ProcedureComplete
    ProcedureComplete --> [*]
```

Success criteria:

- UE reaches registered state.
- GCF transaction state is complete.
- Mobility Management Service has committed registration context.
- NAS Security Service has a valid security context.
- GCF can correlate all uplink NAS events through RAN metadata and can address downlink NAS delivery through SRF.

### 7.2 Scenario: Authentication Failure

```mermaid
stateDiagram-v2
    [*] --> RegistrationRequestReceived
    RegistrationRequestReceived --> AuthenticationStarted
    AuthenticationStarted --> AuthenticationChallengeSent
    AuthenticationChallengeSent --> AuthenticationFailed: invalid RES star or AUSF failure
    AuthenticationFailed --> RejectBuilt: NAS Codec builds reject
    RejectBuilt --> RejectDelivered: SRF sends downlink NAS
    RejectDelivered --> ProcedureFailed
    ProcedureFailed --> [*]
```

### 7.3 Scenario: Access Restricted

```mermaid
stateDiagram-v2
    [*] --> SecurityModeCompleteReceived
    SecurityModeCompleteReceived --> SubscriptionFetched
    SubscriptionFetched --> AccessDenied: Mobility Restriction denies TAI/RAT/slice
    AccessDenied --> RegistrationRejectBuilt
    RegistrationRejectBuilt --> RegistrationRejectProtected
    RegistrationRejectProtected --> RejectDelivered
    RejectDelivered --> ProcedureFailed
    ProcedureFailed --> [*]
```

## 8. Initial Proto Contract Sketch

This is not final IDL. It defines the first contract shape.

```proto
syntax = "proto3";

package free6gc.v1;

message ProcedureRef {
  string procedure_id = 1;
}

message RanRoutingContext {
  string ran_assoc_id = 1;
  string ran_ue_id = 2;
  string amf_ue_ngap_id = 3;
  string ran_ue_ngap_id = 4;
  string plmn_id = 5;
  string tac = 6;
  string cell_id = 7;
  string rat_type = 8;
}

service GcfIngress {
  rpc StartOrContinueProcedure(UplinkNasEvent) returns (GcfIngressResult);
}

message UplinkNasEvent {
  RanRoutingContext ran = 1;
  bytes nas_pdu = 2;
}

message GcfIngressResult {
  string procedure_id = 1;
  string status = 2;
}

service SrfControl {
  rpc SendDownlinkNas(DownlinkNasCommand) returns (DownlinkNasResult);
}

message DownlinkNasCommand {
  RanRoutingContext ran = 1;
  bytes nas_pdu = 2;
}

message DownlinkNasResult {
  bool accepted = 1;
}
```

Suggested proto files:

```mermaid
flowchart TB
    COMMON[common.proto<br/>ProcedureRef, RanRoutingContext, causes, identifiers]
    SRFGCF[srf_gcf.proto<br/>GcfIngress and SrfControl]
    NASC[nas_codec.proto<br/>Decode and build NAS]
    NASS[nas_security.proto<br/>protect/unprotect/context]
    AUTH[authentication.proto<br/>start/confirm/failure]
    SUB[subscription.proto<br/>registration subscription data]
    MR[mobility_restriction.proto<br/>access evaluation]
    MM[mobility_management.proto<br/>allocation and commit]

    COMMON --> SRFGCF
    COMMON --> NASC
    COMMON --> NASS
    COMMON --> AUTH
    COMMON --> SUB
    COMMON --> MR
    COMMON --> MM
```

## 9. Codex Session Prompts

### 9.1 SRF Session Prompt

```text
You are implementing free6gc-srf for milestone 1.

Read:
- proposal.md
- free6gc-initial-design.md
- Free5GC AMF NGAP code under NFs/amf/internal/ngap

Task:
Extract or adapt the NGAP/SCTP-facing behavior needed for initial registration.

Rules:
- SRF terminates NGAP/SCTP.
- SRF forwards raw NAS PDUs plus RAN metadata to GCF over gRPC.
- SRF does not decode NAS business semantics.
- SRF owns RAN association and downlink NGAP delivery state.
- SRF does not own procedure correlation state in milestone 1.
- SRF exposes a gRPC control API for GCF to send downlink NAS.

Deliverables:
- Compilable service binary.
- Minimal config.
- Unit tests for InitialUEMessage and UplinkNASTransport forwarding.
- Fake GCF test server.
```

### 9.2 GCF Session Prompt

```text
You are implementing free6gc-gcf for milestone 1 initial registration.

Read:
- proposal.md
- free6gc-initial-design.md
- Free5GC AMF GMM registration code under NFs/amf/internal/gmm
- Existing TestInitialRegistrationProcedure in NFs/amf/internal/ngap/registration_procedure_test.go

Task:
Implement the initial registration orchestrator.

Rules:
- GCF owns procedure transaction state.
- GCF allocates procedure_id and correlates uplink events by RAN association metadata.
- GCF stores ran_routing_context and uses it when calling SRF downlink APIs.
- GCF calls atomic services over gRPC using static config.
- GCF does not terminate NGAP.
- GCF does not directly parse/build NAS bytes.
- GCF does not own service-local UE contexts.

Deliverables:
- Procedure state machine for successful initial registration.
- gRPC ingress from SRF.
- gRPC clients for NAS Codec, NAS Security, Authentication, Subscription, Mobility Restriction, Mobility Management, and SRF control.
- Unit tests using fake atomic services.
```

### 9.3 NAS Codec Session Prompt

```text
You are implementing free6gc-nas-codec-service for milestone 1.

Read:
- free6gc-initial-design.md
- Free5GC AMF NAS handling and message building code
- github.com/acore2026/nas usage in AMF

Task:
Expose gRPC APIs for decoding and building NAS messages needed by initial registration.

Rules:
- No cryptographic context.
- No procedure state.
- No NGAP.
- Return typed decoded facts to GCF.

Required operations:
- DecodeUplinkNas
- BuildAuthenticationRequest
- BuildSecurityModeCommand
- BuildRegistrationAccept
- BuildRegistrationReject or equivalent error path support
```

### 9.4 NAS Security Session Prompt

```text
You are implementing free6gc-nas-security-service for milestone 1.

Read:
- free6gc-initial-design.md
- Free5GC AMF NAS security code
- Free5GC AMF UE security context code

Task:
Expose gRPC APIs for NAS security context creation, uplink unprotection, and downlink protection.

Rules:
- Service-local in-memory context.
- Primary key is generated security_context_id, with indexes for procedure_id and SUPI when known.
- Own NAS counts and selected algorithms.
- Do not decode NAS message semantics.

Required operations:
- CreateNasSecurityContext
- ProtectDownlinkNas
- UnprotectUplinkNas
- GetSecurityContextDebugSnapshot for tests
```

### 9.5 Authentication Session Prompt

```text
You are implementing free6gc-authentication-service for milestone 1.

Read:
- free6gc-initial-design.md
- Free5GC AMF AuthenticationProcedure and HandleAuthenticationResponse
- Free5GC AMF AUSF consumer code

Task:
Extract Authentication/SEAF responsibility behind gRPC.

Rules:
- Own auth session state in memory.
- Interact with AUSF or configured test backend.
- Validate RES*/HRES*.
- Confirm auth with AUSF.
- Return SUPI and KSEAF to GCF.

Required operations:
- StartAuthentication
- ConfirmAuthentication
- HandleAuthenticationFailure
```

### 9.6 Subscription Session Prompt

```text
You are implementing free6gc-subscription-service for milestone 1.

Read:
- free6gc-initial-design.md
- Free5GC AMF UDM/SDM consumer paths used during initial registration

Task:
Expose registration subscription retrieval behind gRPC.

Rules:
- Hide UDM/UDR API details from GCF.
- Return only data needed by initial registration.
- Service-local cache is allowed but not required.

Required operation:
- FetchRegistrationSubscriptionData
```

### 9.7 Mobility Restriction Session Prompt

```text
You are implementing free6gc-mobility-restriction-service for milestone 1.

Read:
- proposal.md
- free6gc-initial-design.md
- Free5GC AMF registration restriction/policy handling paths

Task:
Expose access evaluation behind gRPC.

Rules:
- Decide allowed/denied for TAI, RAT, and requested NSSAI.
- Return allowed NSSAI and rejection cause when denied.
- Do not allocate GUTI or registration area.

Required operation:
- EvaluateAccess
```

### 9.8 Mobility Management Session Prompt

```text
You are implementing free6gc-mobility-management-service for milestone 1.

Read:
- free6gc-initial-design.md
- Free5GC AMF registration area, GUTI, and registration commit paths

Task:
Expose registration context allocation and commit behind gRPC.

Rules:
- Own GUTI allocation and TAI list assignment.
- Own committed registration context in memory.
- Do not evaluate access permission.
- Do not perform NAS encoding.

Required operations:
- AllocateRegistrationContext
- CommitRegistration
- GetRegistrationDebugSnapshot for tests
```

## 10. Immediate Work Plan

```mermaid
flowchart TB
    A[1. Create free6gc-system<br/>design, proto stubs, compose skeleton, test plan]
    B[2. Create free6gc-api-go<br/>generation workflow]
    C[3. Implement fake atomic services<br/>for GCF/SRF tests]
    D[4. Implement SRF NGAP path<br/>from Free5GC AMF reference]
    E[5. Implement GCF happy path<br/>initial registration orchestration]
    F[6. Extract NAS Codec Service]
    G[7. Extract NAS Security Service]
    H[8. Extract Authentication Service]
    I[9. Extract Subscription Service]
    J[10. Extract Mobility Restriction Service]
    K[11. Extract Mobility Management Service]
    L[12. Replace fakes one by one<br/>in e2e compose]
    M[13. Run UERANSIM initial registration<br/>through SRF/GCF without legacy AMF]

    A --> B
    B --> C
    C --> D
    C --> E
    D --> M
    E --> M
    F --> L
    G --> L
    H --> L
    I --> L
    J --> L
    K --> L
    L --> M
```

## 11. Open Questions

These are intentionally left for later review:

1. What exact JSON schema should golden trace replay use?
2. Should debug snapshot/reset APIs live in the same proto package or in separate debug-only proto files?
3. How should provisional generated NAS fixtures be marked and prevented from becoming permanent golden traces?
4. What command should capture the first UERANSIM golden trace?
5. What release/tag process should publish `free6gc-api-go` after proto updates?
6. Should Compose health checks be required before the E2E harness starts replay?
7. Which service owns operator security algorithm policy config consumed by NAS Security Service?
