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

## 3. 4+1 Architecture Views

The proposal already defines the logical view: a hierarchical architecture with Routing Services, Global Coordination Services, and Atomic Modular Services. This document adds the development, process, physical, and scenario views for the first prototype.

```mermaid
flowchart TB
    CENTER[Free6GC Initial Registration Prototype]
    LOGICAL[Logical View<br/>SRF, GCF, AMCF atomic services]
    DEV[Development View<br/>hybrid multi-repo and proto contracts]
    PROCESS[Process View<br/>initial registration runtime messages]
    PHYSICAL[Physical View<br/>compose and Kubernetes deployment]
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
        PCF[PCF optional]
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
    MR -. policy input .-> PCF
```

## 4. Development View

### 4.1 Repository Model

Use a hybrid multi-repo model.

```mermaid
flowchart TB
    SYS[free6gc-system<br/>integration authority]
    API[free6gc-api-go<br/>generated Go gRPC module]

    SYS --> SPECS[specs/proto<br/>source .proto contracts]
    SYS --> PROMPTS[prompts<br/>Codex task prompts]
    SYS --> DEPLOY[deployments<br/>compose and k8s]
    SYS --> TESTS[tests<br/>contract and e2e]
    SYS --> DOCS[docs/architecture]

    SPECS --> API

    API --> SRF[free6gc-srf<br/>NGAP/SCTP termination]
    API --> GCF[free6gc-gcf<br/>procedure orchestration]
    API --> NASC[free6gc-nas-codec-service<br/>NAS encode/decode]
    API --> NASS[free6gc-nas-security-service<br/>NAS security context]
    API --> AUTH[free6gc-authentication-service<br/>Authentication/SEAF]
    API --> SUB[free6gc-subscription-service<br/>registration subscription data]
    API --> MR[free6gc-mobility-restriction-service<br/>access decisions]
    API --> MM[free6gc-mobility-management-service<br/>GUTI/TA/registration context]

    TESTS --> SRF
    TESTS --> GCF
    TESTS --> NASC
    TESTS --> NASS
    TESTS --> AUTH
    TESTS --> SUB
    TESTS --> MR
    TESTS --> MM
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
        SBI[NFs/amf/internal/sbi/consumer<br/>AUSF/UDM/PCF/NRF clients]
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

    GCF->>MM: CommitRegistration(SUPI, RAN association, completion)
    MM-->>GCF: registration committed

    GCF->>SRF: MarkProcedureComplete / optional UE context release
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

    AUTH -->|SBI or configured endpoint| AUSF[AUSF]
    SUB -->|SBI or configured endpoint| UDM[UDM/UDR]
    MR -->|optional policy input| PCF[PCF]
```

### 5.3 Process Rules

- SRF must not branch on NAS registration semantics except for minimal correlation required to route messages.
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
            NRF[free5gc-nrf or static endpoints]
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
    AUTH -->|SBI/configured endpoint| AUSF
    SUB -->|SBI/configured endpoint| UDM
    UDM --> UDR
    UDR --> DB
    AUTH -. optional .-> NRF
    SUB -. optional .-> NRF
```

Milestone 1 should not run legacy `free5gc-amf` for the registration path. It may remain available only as a reference or comparison target.

SMF and UPF are not required for initial registration unless the scenario is extended to PDU session establishment.

### 6.2 Target Kubernetes Deployment

```mermaid
flowchart TB
    subgraph NS[Namespace: free6gc]
        subgraph INGRESS[External access]
            NGAP[SRF NGAP exposure<br/>hostNetwork / NodePort / LoadBalancer]
        end

        subgraph CORE[Free6GC deployments]
            SRF[Deployment: free6gc-srf]
            GCF[Deployment: free6gc-gcf]
            NASC[Deployment: nas-codec]
            NASS[Deployment: nas-security]
            AUTH[Deployment: authentication]
            SUB[Deployment: subscription]
            MR[Deployment: mobility-restriction]
            MM[Deployment: mobility-management]
        end

        subgraph LEGACY[Free5GC backend services]
            AUSF[Deployment: ausf]
            UDM[Deployment: udm]
            UDR[Deployment: udr]
            DB[(StatefulSet: mongodb)]
        end

        CM[ConfigMaps<br/>service endpoints, PLMN, TAC, slice, timers, algorithms]
        SEC[Secrets<br/>TLS material and test subscriber credentials]
    end

    NGAP --> SRF
    SRF -->|ClusterIP gRPC| GCF
    GCF -->|ClusterIP gRPC| NASC
    GCF -->|ClusterIP gRPC| NASS
    GCF -->|ClusterIP gRPC| AUTH
    GCF -->|ClusterIP gRPC| SUB
    GCF -->|ClusterIP gRPC| MR
    GCF -->|ClusterIP gRPC| MM
    AUTH --> AUSF
    SUB --> UDM
    UDM --> UDR
    UDR --> DB
    CM -. mounted/read .-> GCF
    CM -. mounted/read .-> SRF
    SEC -. mounted/read .-> CORE
```

### 6.3 Scaling Model

```mermaid
flowchart LR
    SRF[SRF<br/>stateful SCTP/RAN associations] -->|sticky procedure routing| GCF[GCF<br/>stateful procedure transactions]
    GCF --> NASC[NAS Codec<br/>stateless]
    GCF --> NASS[NAS Security<br/>stateful in-memory security context]
    GCF --> AUTH[Authentication<br/>stateful active auth sessions]
    GCF --> SUB[Subscription<br/>mostly stateless, optional cache]
    GCF --> MR[Mobility Restriction<br/>stateless decision engine]
    GCF --> MM[Mobility Management<br/>stateful registration context]

    NASC -. easiest to scale .-> NASC2[NAS Codec replica]
    MR -. easiest to scale .-> MR2[Mobility Restriction replica]
    NASS -. needs context affinity or persistence later .-> NASS2[NAS Security replica]
    MM -. needs persistence later .-> MM2[Mobility Management replica]
```

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
- SRF can correlate all uplink/downlink NAS transports for the RAN UE association.

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
  string ran_assoc_id = 2;
  string ran_ue_id = 3;
}

message RanMetadata {
  string plmn_id = 1;
  string tac = 2;
  string cell_id = 3;
  string rat_type = 4;
}

service GcfIngress {
  rpc StartOrContinueProcedure(UplinkNasEvent) returns (GcfIngressResult);
}

message UplinkNasEvent {
  ProcedureRef ref = 1;
  RanMetadata ran = 2;
  bytes nas_pdu = 3;
}

message GcfIngressResult {
  string procedure_id = 1;
  string status = 2;
}

service SrfControl {
  rpc SendDownlinkNas(DownlinkNasCommand) returns (DownlinkNasResult);
  rpc MarkProcedureComplete(ProcedureRef) returns (MarkProcedureCompleteResult);
}

message DownlinkNasCommand {
  ProcedureRef ref = 1;
  bytes nas_pdu = 2;
}

message DownlinkNasResult {
  bool accepted = 1;
}

message MarkProcedureCompleteResult {
  bool accepted = 1;
}
```

Suggested proto files:

```mermaid
flowchart TB
    COMMON[common.proto<br/>ProcedureRef, RanMetadata, causes, identifiers]
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
- SRF owns RAN association and RAN UE routing state.
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
- Keyed by procedure_id/security_context_id.
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

1. Should milestone 1 use mTLS between services, or plaintext gRPC on an isolated Docker network?
2. Should `free6gc-api-go` be generated in CI only, or checked in for easier Codex sessions?
3. Should service-local memory be reset by admin API for tests?
4. Should GCF transaction logs be persisted to disk for replay in milestone 1?
5. How much of NRF remains in milestone 1 if GCF uses static config?
6. Should PCF be included in the happy path, or deferred until access restriction/policy tests require it?
7. Should Registration Complete trigger UE context release behavior in milestone 1, or just commit registration?
