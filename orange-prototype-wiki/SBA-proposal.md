# 1. Introduction

This contribution addresses architectural requirements (23801-01: 4.2 minimize inter dependencies between NFs) and Key Issue #2 (SBA framework)

- **Chapter 4.2-architectual requirements: minimize dependencies**

- *Bullet 1: The 6G System should: Minimize inter dependencies between NFs when introducing a new feature or procedure for an NF. The architectural requirements apply to all WTs and corresponding key issues in this study.*

- The following bullet numbers within KI#2: SBA framework are also addressed.

  - *Bullet 2: Study improve NF/NF service resiliency, scalability, efficiency and load balancing.*

While the 5GC introduced service-oriented interfaces, a significant gap remains in actual deployments. To achieve the goals of network agility and flexibility, there are still three key issues to address.

- **Monolithic architecture and Granularity Mismatch:** in 5G, network functions remain essentially monolithic. Although the concept of network function services was introduced, the role and responsibilities of each service were never clearly defined. The specification focuses on how a network function as a whole handles service requests received from the UE, RAN, or other NFs through its service-based interface, but stops short of defining what each individual network function service is actually responsible for. As a result, the service-based interface (SBI) serves merely as an exposure mechanism toward other NFs, rather than as a true architectural boundary. A single coarse-grained NF Service often encapsulates multiple distinct internal functionalities with vastly different processing characteristics, preventing the network from performing fine-grained scaling, isolating micro-failures, or upgrading specific processing logic (e.g., routing rules) without disrupting active user connections. In this sense, 5G SBA defines service-based interfaces, but falls short of delivering a genuinely service-based architecture.

- **Internal Coupling between Services:** this architectural deficiency is further reflected in the granularity of SBI interface definitions. Rather than exposing fine-grained, atomic services, SBI interfaces are defined at the granularity of complete procedures. A single SBI interface invocation triggers an entire end-to-end procedure, which internally involves multiple sequential interactions across different network entities. What appears to be a single service call is in fact a tightly coupled chain of operations, with the invocation conditions, input parameters, and triggered processes all pre-defined and bound to a specific procedure. Consequently, SBI interfaces cannot be invoked outside of their prescribed procedural context — the interface and the procedure it represents are inseparable. For example, the standard defines 49 usage scenarios and procedures for the Update SM Context service operation, and different procedures and scenarios require different parameters to be carried. As a consequence of this coarse-grained design, SBI interfaces are not reusable building blocks, restricting deployment flexibility and violating the isolation principles. Since each interface encapsulates a complete procedure rather than a discrete, self-contained function, it cannot be freely combined with other interfaces to compose new behaviors. Introducing a new feature therefore requires defining an entirely new procedure from scratch, rather than assembling existing services on demand. This fundamentally contradicts the promise of a service-oriented architecture, where fine-grained, loosely coupled services should be composable in flexible ways to realize new functionalities.

- **Inter-NF dependency:** the SBA architecture exhibits limitations in minimizing inter-NF dependencies and achieving effective topology hiding, as evidenced by the following coupling issues:

- **RAN-AMF Coupling via the N2 Interface:** The N2 interface establishes a direct, point-to-point association between the RAN and the AMF, requiring the RAN to maintain explicit awareness of AMF identity and reachability. As a result, AMF scaling, migration, or failover events must be coordinated with the RAN, creating operational dependencies that contradict the principle of topology hiding and limit the independent deployability of core network functions.

- **Inter-Domain/Inter-PLMN Topology Exposure:** Cross-domain and inter-PLMN procedures require network functions to maintain awareness of the internal topology of peer domains. In roaming scenarios, for instance, the visited network's AMF and SMF must be aware of the home network's SMF topology in order to make appropriate NF selection decisions. Similarly, during UE mobility across service areas, the target AMF and SMF must interact directly with their source counterparts to retrieve UE context and session state, exposing internal topology information across NF boundaries. This inter-domain topology awareness introduces complexity, increases signalling overhead, and makes the architecture sensitive to the internal structure of peer networks.

- **Inter-NF Service Coupling:** The coarse-grained nature of current services leads to heavy external interface entanglement, forcing NFs to execute cross-domain logic and understand each other's internal state machines and contexts. A prime example is the coupling between the AMF and SMF during mobility procedures. This rigid inter-dependency is fundamentally driven by two architectural flaws: the AMF implicitly acts as a monolithic signaling router for Session Management (SM) messages, and it is forced to parse N2 SM containers. Since the AMF must understand SM states to trigger the correct updates, the states and contexts of the AMF and SMF become tightly interlocked. Consequently, a change in one domain mandates structural modifications in the other. This deep entanglement prevents hitless, independent upgrades and directly violates the 6G principle of minimizing inter-dependencies.

These aforementioned dependencies, both within a single domain (as in point 1) and across multiple domains (as in point 2), significantly increase the complexity of UE mobility management procedures.

This architectural rigidity has practical consequences. As discussed in the previous S2-2602326 document , the 5G SBA faces several challenges that limit its flexibility, particularly in terms of independently scaling specific functions or introducing new features without impacting the broader system. To address these limitations, a **Modular NF Service Architecture** has been proposed to eliminate internal and external entanglement, achieve on-demand service delivery, and provide strong resiliency, advanced scalability, enhanced efficiency, optimized load balancing, and unprecedented flexibility. Taking the 6G AMF (referred to here as the AMCF) as an illustrative example, it may be decomposed into the following modular services: Authentication Service, NAS Security Service, Mobility Service, Paging Service, Reachability Service, and Mobility Restriction Service.

Once Network Functions are decomposed into discrete, loosely coupled modular services, it becomes necessary to clarify how the Core Network interfaces with external systems — such as the RAN and UE — to execute the procedures required to fulfil their requests. This solution further introduces a service interaction model with some middle layer NFs to simplify network dependencies, NFs topology-hiding, decoupling service relationship, optimize signaling latency, which fulfill the procedure-level architecture design.

- Global Coordination Function: topology hiding between networks/sub-networks, procedure control and dynamic scheduling, parallel NFs execution, user-centric closed-loop processing.

- SRF (Signaling Routing Function): bidirectional topology hiding and link convergence between 6G AN and 6G CN


# 2. Solution discussion

## 2.1 General Description

### 2.1.1 Objectives

The new architecture shall fulfill the following objectives:

- **Enable flexible composition and invocation of services**: Future new business processes may not be fully anticipated or pre-defined in the standards. Decoupled **services** and interfaces shall support the capability to compose new flows - either manually or intelligently - allowing new functionalities to be realized through flexible, on-demand combination and sequential invocation of fine-grained services.

- **Achieve decoupling of function changes between RAN and CN**: Following function refactoring, changes to CN functions shall not impact the RAN or its connectivity. The RAN shall remain agnostic to the internal structure and topology of the CN, such that CN scaling, migration, or functional upgrades can be performed independently without requiring coordination with or reconfiguration of the RAN.

- **Achieve cross-domain topology hiding:** The internal functional topology of each domain shall remain opaque to peer domains, whether in roaming scenarios or during inter-domain mobility. This aims to reduce inter-domain functional dependencies.

### 2.1.2 Principles

To achieve the aforementioned objectives, the decoupling and reconstruction of network functions and services shall adhere to the following principles:

- **Context Isolation**: Decoupled services within a Network Function (NF) shall be strictly isolated at the context level. Each service shall expose only standardized service interfaces to other services, and shall not share any user context, session context, and state data. All interactions between services shall occur exclusively through standardized interfaces or via third-party services. Implicit state dependencies, shared data structures, or any other form of tight coupling shall be prohibited, ensuring that each service can evolve, scale, and be replaced independently without affecting others.

- **Single Responsibility of service:** Each decoupled service shall have a single responsibility, with well-defined boundaries, performing cohesive and deterministic tasks with specified inputs and outputs. The bundling of unrelated functionalities and logic within a single module, leading to logical overload and confusion, shall be strictly avoided. Technical measures, such as interface design and context isolation, shall be employed to ensure the stability of functional boundaries.

- **Minimal Inter-Service Dependency:** Each service shall fulfill its own responsibilities independently, without relying on the internal implementation, operational state, or existence of other services. Inter-service interactions must be completely decoupled, especially from the internal state machines and cross-domain routing obligations of the interacting NFs. Consumer NFs shall not be forced to construct highly conditional payloads to understand the producer's internal state. Minimal dependency shall be achieved through high cohesion, low coupling, unidirectional dependencies, and the utilization of third-party dependencies.

### 2.1.3 Architecture and Atomic Modular Services

#### 2.1.3.1 Hierarchical Architecture

Currently, the SBA architecture adopts a full mesh model between network functions (NFs). Functions are distributed across each NF, and completing a procedure requires chaining multiple NFs in series. Different procedures involve different chained NFs, resulting in a complex mesh of relationships among NFs in the network. To simplify the functions of each NF and achieve minimized coupling between NFs for 6G networks, the target network architecture is a hierarchical one, consisting of the following functional entities:

**Atomic Modular Services:** This type of function is highly cohesive. Even if it internally relies on invocations to other network elements, it presents itself externally as an atomic operation characterized by “one request, one response”. Services such as User Authentication, Access Restriction, NAS Security, Reachability Management, belong to Atomic Functions.

Atomic services may reside within different network entities based on their functional cohesion and deployment affinity. For example, User Authentication, Access Restriction, and Reachability Management may reside within the AMCF, while services such as UP Selection, Address Management, and QoS Flow Management may reside within the SCF.

**Global Coordination Services:** This type of service is responsible for the end-to-end composition and interaction of control procedures. It makes decisions and performs scheduling based on request message, corresponding user attributes, subscription, and service configuration, possessing the control over the procedure (determining “which service is invoked first, which next, and how to handle errors”). Participating atomic modular services do not need to be aware of the global procedure. These services manage the procedure state and transaction contexts but do not themselves hold the core service context of the business function. Simultaneously, these services also hide inter-domain topology information from the Atomic Modular Services, thereby keeping Atomic Services simpler. Services such as Mobility Procedure Control, and Session Procedure Control belong to this category. According to functional cohesion and affinity relationships, this type of services may also be managed within an independent NF. The Global Coordination Service does not eliminate the inherent complexity of mobile network procedures; rather, it externalises and centralises the procedure logic that was previously embedded within each monolithic Network Function. This separation of procedure control and management from service execution is the key architectural principle, enabling individual services to remain simple, independently scalable, and reusable across different procedures.

**Routing Services:** This type of service does not carry business semantics. It is solely responsible for the correct routing and distribution of messages/signaling without modifying message content or holding business state. It is essentially an infrastructure-layer function for information transfer. This type of service primarily performs protocol conversion to achieve the goals of inter-domain decoupling and topology hiding. This type of function can also be managed via an independent NF.

![image1](<NGA.T1839 Service-Granularity Resilience_v3_assets/image1.png>)

SBA architecture (Full mesh)

![image2](<NGA.T1839 Service-Granularity Resilience_v3_assets/image2.png>)

Target architecture (Hierarchical)

#### 2.1.3.2 Atomic Services

The following examples illustrate how the modular architecture structurally resolves the limitations of the 5GS architecture during basic connectivity procedures:

- **Minimal Inter Dependencies:** In legacy 5GS, introducing new features often requires standardizing and upgrading the entire monolithic NFs. In this modular architecture, capabilities are strictly isolated. For instance, the monolithic legacy AMF functionalities are decomposed into isolated and independent modular services, specifically separating Registration Management (RM), Connection Management (CM), Mobility Management (MM), Security Anchor Function (SEAF), and Routing capabilities. This ensures each can be updated or scaled independently according to its specific traffic patterns. Then operators can seamlessly upgrade authentication services (hosted in the separated SEAF entity) for emerging devices requiring Post-Quantum Cryptography without modifying the baseline signaling logic. Furthermore, by decoupling the Routing functionality from the rest of the legacy AMF, updates or independent scaling of the NAS routing logic will have a minor impact on existing RAN Connections and significantly reduce disruptive impacts on ongoing CM states. This enables **Advanced Scalability and Efficiency:** modular approach enables targeted scaling of specific capabilities (e.g., scaling only CM/MM modules without over-provisioning RM modules) based on real-time traffic demands, maximizing resource utilization. Additionally, this also contributes to **Unprecedented Flexibility and Agility:** Instead of modifying tightly coupled logic embedded within legacy monolithic NFs, introducing a new feature simply requires adding or updating an independent atomic service. The network could provision flexible and tailored services, and rapidly adapt to diverse 6G use cases.

- **Service-Granularity Resilience**: In the proposed modular architecture, resilience is fundamentally shifted to the highly granular Service level thus to achieve **Enhanced Resiliency and Load Balancing**: Load balancing and error-handling are achieved at the modular service level, each service instance can be dynamically isolated or bypassed or replaced in case of failure or congestion. Because services are strictly isolated and deterministically defined, this architecture naturally synergizes with new NF inter-connection framework as proposed in S2-2602325. Though dynamically evaluates the operational status of the network function, the Signalling Forwarding Function SFF could provide service instance information to the target NF. This intrinsic, fine-grained service-level redundancy ensures continuous, uninterrupted processing and guarantees resilience procedure execution for the 6G control plane.

Based on the principles of network service singularity and context isolation, the basic connection network functions and services can be reconstructed into the following independent services:

Note: The service examples are not intended to prescribe the final set or granularity. The actual level of reconstruction—whether broader or finer‑grained—will be refined as the study progresses.

- **Mobility Management Services**

| Service                      | Service Description                                                                                                                                                                                                                                        | Context                                                                                                                    |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Authentication               | Responsible for UE authentication/re-authentication, processing authentication results, and managing authentication failure handling; responsible for key derivation to support NAS security and RAN security                                              | SUPI, SUCI, AUTH-Result, ABBA, RAND                                                                                        |
| Mobility management          | Responsible for TA list and UE 6G GUTI management                                                                                                                                                                                                          | SUPI, SUCI                                                                                                                 |
| Mobility Restriction         | Determines if a UE is permitted to access a specific RAT, area, or slice based on subscription data and operator policies. Manages Forbidden Areas, Non-Allowed Areas, and Service Area Restrictions.                                                      | Allowed NSSAI, Forbidden TAC List, RAT Restriction, Roaming Restriction, Service Area Restriction                          |
| Paging management            | Responsible for paging, determines the paging strategy (paging area, paging priority).                                                                                                                                                                     | AI List, Registration Area, Paging Priority, UE Paging DRX                                                                 |
| UE 6GAP Management           | Manages the UE 6G Association (6GAP) between RAN and CN, including 6G connection establishment, release, and reconfiguration. Responsible for 6GAP message encoding and decoding.                                                                          | UE Association                                                                                                            |
| 6G Link Management           | Manages the link layer connection between RAN node and CN, including connection Setup and release, RAN node level info management, such as TA list.                                                                                                        | 6G Link State, SCTP Association                                                                                            |
| NAS Security                 | Manages the NAS layer security context, including the Security Mode Control (SMC) procedure, and the selection of algorithms for NAS ciphering and integrity protection, and the NAS message encryption/decryption.                                        | NAS Security Context, NAS INT/ENC Algorithm                                                                                |
| Reachability                 | Monitors and manages the reachability state of the UE, including monitoring periodic registration, reachability timer and implicit de-registration timer management                                                                                        | Reachability State, UE Activity Status, Maximum Latency/Response Time                                                      |
| Subscription Data Management | Subscribes to and retrieves UE subscription data from the UDM.                                                                                                                                                                                             | SUPI, Subscribed DNN, Subscribed S-NSSAI, Subscribed Session-AMBR                                                          |
| UE Policy                    | Retrieves UE-level mobility policies from the PCF, including dynamic policies such as access barring.                                                                                                                                                      | Policy Association ID (UE-level), UE Policy                                                                                |
| Mobility Procedure Control   | Controls the complete procedure for UE mobility: Registration procedures (Initial, Update), Service Request procedure, Handover procedure, and Mobility Registration Updates. Manages the mobility state machine and state transitions.                    | Registration Type, Registration State, Mobility State, Handover Type, Source/Target AMF, TAI List, Service Request Trigger |
| Network Function Selection   | Selects the target Network Function (e.g., SMF, PCF, AMF) during registration and mobility procedures. Selects the target SMF, PCF, and NSSF (for slice selection). Decisions are based on conditions such as UE location, slice information, and NF load. | DNN, S-NSSAI, UE Location (TAI), SMF Set ID, PCF ID, NSSF ID, NRF Query Result                                             |

- **Session Management Services**

| Service                   | Service Description                                                                                                                                                                                                          | Context                                                                                                         |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Session Procedure Control | Controls the complete lifecycle of a PDU Session: Establishment, Modification, and Release.                                                                                                                                  | PDU Session ID, PDU Session Type (IP/Ethernet/Unstructured), PDU Session State                                  |
| UP Session Control        | Manages the 6G N4 session between the SMF and the UPF, including 6G N4 Session Establishment, Modification, and Deletion. Configures PDR, FAR, QER, and URR rules, and manages UP Tunnels between UPs and between UP and RAN | 6G N4 Session ID, PDR ID, FAR ID, QER ID, URR ID, F-TEID, Apply Action, Precedence                              |
| Policy Control            | Retrieves PCC rules at the PDU Session level from the PCF.                                                                                                                                                                   | PCC Rule ID, 5QI, ARP, Session-AMBR, Gating Status, Reflective QoS Indicator                                    |
| Secondary Authentication  | Performs secondary authentication/authorization for a specific DNN (e.g., enterprise network) by validating the UE’s authorization to access a particular Data Network via the DN-AAA server.                                | SUPI, DNN, DN-AAA Server Address, EAP Message, Auth-Result, Authorization Data                                  |
| Charging Management       | Manages charging at the PDU Session level: interacts with the CHF to create charging sessions, generate CDRs, and report usage (Volume/Time/Event).                                                                          | Charging ID, CHF Address, Rating Group, Service ID, Usage Report, CDR Trigger, Charging Method (Online/Offline) |
| QoS Flow Management       | Establishes, modifies, and releases QoS Flows, and manages both the default and non-default QoS Flows.                                                                                                                       | QFI, 5QI, ARP, QoS Flow State, Default QoS Rule, GFBR/MFBR                                                      |
| Subnet Management         | Manages UE subscriptions for IP multicast/broadcast, supporting 6G LAN-type Services and Group Communication. Manages membership and user-plane topology for 5G VN Groups.                                                   | 6G VN Group ID, Group Member List, Multicast/Broadcast Session ID, 5G VN Group Data, DNN (LAN-type)             |
| RAN Session Control       | Manages RAN Session, including RAN session establishment, modification, and release, including configuration and reconfiguration of RAN session QoS profile                                                                  | PDU Session ID, QoS Profile                                                                                     |
| UE Control                | Manages UE session, including configuration and reconfiguration of UE Session QoS rules                                                                                                                                      | PDU Session ID, QoS rules                                                                                       |

- **UP Node Control Services**

| Service              | Service Description                                                                                    | Context                                                                                                    |
| -------------------- | --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| UP Node Management    | Manages the state of UPF nodes and the links between the SMF and UPFs.                                    | UPF ID, UPF State                                                                                    |
| --------------------- | --------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| UP Selection          | Selects the target UPF based on conditions such as DNN, S-NSSAI, UE location, DNAI, and UPF capabilities. | DNAI, UPF Profile, DNN, S-NSSAI, UE TAI, UPF Load, UPF Service Area                                   |
| IP Address Management | Allocates and manages IP addresses (IPv4/IPv6) or an Ethernet MAC address for the PDU Session.            | UE IP Address, IPv4 Address Pool, IPv6 Prefix, DHCP Server Config, MAC Address, IP Address Lease Time |

Based on the preceding analysis of service decoupling and typology, the target architecture is proposed as follows.

![image3](<NGA.T1839 Service-Granularity Resilience_v3_assets/image3.png>)

- **SRF (Signaling Routing Function):** The SRF is a signaling routing and distribution NF that shields the RAN and CN from the impact of functional changes. It is responsible for the uplink and downlink distribution of NAS and N2 messages.

- **AMCF:** The AMCF is a collection of user access-related Atomic Services. It encompasses functionalities such as user authentication, access restrictions, feasibility management, and paging.

- **SCF:** The SCF is a collection of user session-related Atomic Services. It includes functionalities such as UP management, QoS Flow management, N4 session management, address management, and secondary authentication.

- **Global Coordination Function:** Corresponding to the above mentioned Global Coordination Services. The Global Coordination Function is responsible for coordination across Atomic Services and domains, hiding inter-domain topology information to simplify signaling procedures. Its functionalities include mobility procedure control and session procedure control. Whether the Connection Function constitutes an independent NF is detailed in an alternative solution.

## 2.2 Standalone Global Coordination Function vs. Global Coordination Function within Network Functions

Two potential deployment options are identified for the Global Coordination Function: **Standalone** or **Collocated with Network Functions** (e.g., collocated with AMCF, SCF, etc.), as illustrated in Figure 2.2-1.

![image4](<NGA.T1839 Service-Granularity Resilience_v3_assets/image4.png>)

![image5](<NGA.T1839 Service-Granularity Resilience_v3_assets/image5.png>)

**Standalone Connection Function                   Connection Function Collocated with NFs**

**Figure 2.2-1 Deployment Options of Connection Function**

**Option 1: Standalone Global Coordination Function**

In this option, Global Coordination Function is deployed as an independent entity. It can invoke services provided by different Network Functions — for example, mobility-related services from the AMCF and session-related services from the SCF.

A key advantage of this option is that it **abstracts core network complexity from external entities**. In practice, most procedures require the coordination of multiple service types spanning different Network Functions. For example, a handover procedure involves both mobility-related handling — such as mobility restriction enforcement and security context management — and session-related handling — such as session path preparation, bearer switching, and indirect forwarding tunnel establishment. Similarly, the service request procedure requires both mobility-related handling to transition the UE's CM state and session-related handling to reactivate the user plane. With a standalone Connection Function acting as the single entry point, an external entity such as the RAN only needs to send a single request message — for instance, a HANDOVER REQUIRED message. The Global Coordination Function autonomously orchestrates the required services across the AMCF and SCF, consolidates the results, and returns a single response to the requesting entity. This single-interface model significantly reduces the signalling burden and procedural complexity imposed on external entities.

Furthermore, the standalone Global Coordination Function also simplifies **inter-domain and inter-PLMN mobility procedures**. The Global Coordination Function can aggregate the full UE context — encompassing both mobility management (MM) and session management (SM) context — and interact directly with the Global Coordination Function in the target domain or PLMN through a single, well-defined interface. This eliminates the need for multiple cross-domain interfaces and reduces the number of message exchanges required to complete the procedure, resulting in lower signalling overhead and faster procedure execution.

In addition, the standalone deployment offers superior **scalability and flexibility**. As an independent entity, the Global Coordination Function can be scaled horizontally based on traffic load without being constrained by the resource profile of any individual Network Function. It also provides a natural extension point for introducing new procedures or features, as changes to orchestration logic are localised within the Global Coordination Function and do not require modifications to the underlying microservices.

**Option 2: Connection Function Collocated with NFs**

This option **exposes core network complexity to external entities**. In this option, under a distributed NAS transmission assumption, external entities are required to interact with multiple Global Coordination Functions residing in different Network Functions, rather than a single entry point.

For example, to execute a handover procedure, the RAN must decompose the request into a mobility-related HANDOVER REQUIRED message directed to the Global Coordination Function in the AMCF, and a session-related HANDOVER REQUIRED message directed to the Global Coordination Function in the SCF. The Global Coordination Function in the AMCF handles mobility-related aspects — such as mobility restriction control and security context management — while the Global Coordination Function in the SCF handles session-related aspects — such as session path preparation and bearer switching. Both return independent responses to the RAN, which must then wait for and correlate the two responses before proceeding to the next step. This introduces response correlation logic into the RAN, increasing its implementation complexity and potentially introducing additional latency due to the need to synchronise across two independent response paths.

The same issue arises for UE-initiated procedures. When triggering a service request, the UE may need to send separate messages to the Global Coordination Function in the AMCF to activate its CM state, and to the Global Coordination Function in the SCF to reactivate its session. This imposes additional complexity on the UE protocol stack and increases the number of over-the-air message exchanges, which is particularly undesirable in resource-constrained or latency-sensitive scenarios.

Inter-domain and inter-PLMN mobility procedures are similarly affected. In this option, both the AMCF and SCF in the source domain must independently establish interactions with their counterparts in the target domain, necessitating the definition of multiple cross-domain interfaces and a greater number of message exchanges to complete the procedure. This not only increases standardisation effort but also introduces additional points of failure and coordination overhead.

**Conclusion**

**Option 1 is preferable to Option 2.** The standalone Global Coordination Function provides a clean single-entry-point model that abstracts core network complexity from external entities, reduces signalling overhead, and simplifies inter-domain procedures. Option 2, while architecturally simpler in terms of co-location, redistributes complexity onto external entities such as the RAN and UE — entities that should remain agnostic to the internal structure of the core network. The principle that core network complexity should be contained within the core network strongly favours the standalone deployment model.


# 3. Text Proposal

It is proposed to capture the following changes vs. TR 23.801-01.

* * * * First change * * * *

## 6.0 Mapping of Solutions to Key Issues

Guidance – Fill out the table describing how the solutions map to KIs

Table 6.0-1: Mapping of Solutions to Key Issues

|           | Key Issues |     |     |     |     |     |     |     |     |     |     |     |     |     |     |     |     |     |     |
| --------- | ---------- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Solutions | #2         | #Q  |     |     |     |     |     |     |     |     |     |     |     |     |     |     |     |     |     |
| #X.Y      | x          |     |     |     |     |     |     |     |     |     |     |     |     |     |     |     |     |     |     |

* * * * Second change * * * *

### 6.2.Y Solution #2.X: NF Service Reconstruction

#### 6.2.Y.1 Objectives

The new architecture shall fulfill the following objectives:

- **Enable flexible composition and invocation of services**: Future new business processes may not be fully anticipated or pre-defined in the standards. Decoupled **services** and interfaces shall support the capability to compose new flows - either manually or intelligently - allowing new functionalities to be realized through flexible, on-demand combination and sequential invocation of fine-grained services.

- **Achieve decoupling of function changes between RAN and CN**: Following function refactoring, changes to CN functions shall not impact the RAN or its connectivity. The RAN shall remain agnostic to the internal structure and topology of the CN, such that CN scaling, migration, or functional upgrades can be performed independently without requiring coordination with or reconfiguration of the RAN.

- **Achieve cross-domain topology hiding:** The internal functional topology of each domain shall remain opaque to peer domains, whether in roaming scenarios or during inter-domain mobility. This aims to reduce inter-domain functional dependencies.

#### 6.2.Y.2 Principles

To achieve the aforementioned objectives, the decoupling and reconstruction of network functions and services shall adhere to the following principles:

- **Context Isolation**: Decoupled services within a Network Function (NF) shall be strictly isolated at the context level. Each service shall expose only standardized service interfaces to other services, and shall not share any user context, session context, and state data. All interactions between services shall occur exclusively through standardized interfaces or via third-party services. Implicit state dependencies, shared data structures, or any other form of tight coupling shall be prohibited, ensuring that each service can evolve, scale, and be replaced independently without affecting others.

- **Single Responsibility of service:** Each decoupled service shall have a single responsibility, with well-defined boundaries, performing cohesive and deterministic tasks with specified inputs and outputs. The bundling of unrelated functionalities and logic within a single module, leading to logical overload and confusion, shall be strictly avoided. Technical measures, such as interface design and context isolation, shall be employed to ensure the stability of functional boundaries.

- **Minimal Inter-Service Dependency:** Each service shall fulfill its own responsibilities independently, without relying on the internal implementation, operational state, or existence of other services. Inter-service interactions must be completely decoupled, especially from the internal state machines and cross-domain routing obligations of the interacting NFs. Consumer NFs shall not be forced to construct highly conditional payloads to understand the producer's internal state. Minimal dependency shall be achieved through high cohesion, low coupling, unidirectional dependencies, and the utilization of third-party dependencies.

#### 6.2.Y.3 Hierarchical Architecture

Currently, the SBA architecture adopts a full mesh model between network functions (NFs). Functions are distributed across each NF, and completing a procedure requires chaining multiple NFs in series. Different procedures involve different chained NFs, resulting in a complex mesh of relationships among NFs in the network. To simplify the functions of each NF and achieve minimized coupling between NFs for 6G networks, the target network architecture is a hierarchical one, consisting of the following functional entities:

**Atomic Modular Services:** This type of function is highly cohesive. Even if it internally relies on invocations to other network elements, it presents itself externally as an atomic operation characterized by “one request, one response”. Services such as User Authentication, Access Restriction, NAS Security, Reachability Management, belong to Atomic Functions.

Atomic services may reside within different network entities based on their functional cohesion and deployment affinity. For example, User Authentication, Access Restriction, and Reachability Management may reside within the AMCF, while services such as UP Selection, Address Management, and QoS Flow Management may reside within the SCF.

**Global Coordination Services:** This type of service is responsible for the end-to-end composition and interaction of control procedures. It makes decisions and performs scheduling based on request message, corresponding user attributes, subscription, and service configuration, possessing the control over the procedure (determining “which service is invoked first, which next, and how to handle errors”). Participating atomic modular services do not need to be aware of the global procedure. These services manage the procedure state and transaction contexts but do not themselves hold the core service context of the business function. Simultaneously, these services also hide inter-domain topology information from the Atomic Modular Services, thereby keeping Atomic Services simpler. Services such as Mobility Procedure Control, and Session Procedure Control belong to this category. According to functional cohesion and affinity relationships, this type of services may also be managed within an independent NF.

Note: The Global Coordination Service does not eliminate the inherent complexity of mobile network procedures; rather, it externalises and centralises the procedure logic that was previously embedded within each monolithic Network Function. This separation of procedure control and management from service execution is the key architectural principle, enabling individual services to remain simple, independently scalable, and reusable across different procedures.

**Routing Services:** This type of service does not carry business semantics. It is solely responsible for the correct routing and distribution of messages/signaling without modifying message content or holding business state. It is essentially an infrastructure-layer function for information transfer. This type of service primarily performs protocol conversion to achieve the goals of inter-domain decoupling and topology hiding. This type of function can also be managed via an independent NF.

#### 6.2.Y.4 Atomic Services of 6G Mobility Management Function and 6G Session Management Function

The following examples illustrate how the modular architecture structurally resolves the limitations of the 5GS architecture during basic connectivity procedures:

- **Minimal Inter Dependencies:** In legacy 5GS, introducing new features often requires standardizing and upgrading the entire monolithic NFs. In this modular architecture, capabilities are strictly isolated. For instance, the monolithic legacy AMF functionalities are decomposed into isolated and independent modular services, specifically separating Registration Management (RM), Connection Management (CM), Mobility Management (MM), Security Anchor Function (SEAF), and Routing capabilities. This ensures each can be updated or scaled independently according to its specific traffic patterns. Then operators can seamlessly upgrade authentication services (hosted in the separated SEAF entity) for emerging devices requiring Post-Quantum Cryptography without modifying the baseline signaling logic. Furthermore, by decoupling the Routing functionality from the rest of the legacy AMF, updates or independent scaling of the NAS routing logic will have a minor impact on existing RAN Connections and significantly reduce disruptive impacts on ongoing CM states. This enables **Advanced Scalability and Efficiency:** modular approach enables targeted scaling of specific capabilities (e.g., scaling only CM/MM modules without over-provisioning RM modules) based on real-time traffic demands, maximizing resource utilization. Additionally, this also contributes to **Unprecedented Flexibility and Agility:** Instead of modifying tightly coupled logic embedded within legacy monolithic NFs, introducing a new feature simply requires adding or updating an independent atomic service. The network could provision flexible and tailored services, and rapidly adapt to diverse 6G use cases.

- **Service-Granularity Resilience**: In the proposed modular architecture, resilience is fundamentally shifted to the highly granular Service level thus to achieve **Enhanced Resiliency and Load Balancing**: Load balancing and error-handling are achieved at the modular service level, each service instance can be dynamically isolated or bypassed or replaced in case of failure or congestion. Because services are strictly isolated and deterministically defined, this architecture naturally synergizes with new NF inter-connection framework as proposed in S2-2602325. Though dynamically evaluates the operational status of the network function, the Signalling Forwarding Function SFF could provide service instance information to the target NF. This intrinsic, fine-grained service-level redundancy ensures continuous, uninterrupted processing and guarantees resilience procedure execution for the 6G control plane.

Based on the principles of network service singularity and context isolation, the basic connection network functions and services can be reconstructed into the following independent services:

Note: The service examples are not intended to prescribe the final set or granularity. The actual level of reconstruction—whether broader or finer‑grained—will be refined as the study progresses.

- **Mobility Management Services**

| Service                      | Service Description                                                                                                                                                                                                                                        | Context                                                                                                                    |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Authentication               | Responsible for UE authentication/re-authentication, processing authentication results, and managing authentication failure handling; responsible for key derivation to support NAS security and RAN security                                              | SUPI, SUCI, AUTH-Result, ABBA, RAND                                                                                         |
| Mobility management          | Responsible for TA list and UE 6G GUTI management                                                                                                                                                                                                          | SUPI, SUCI                                                                                                                 |
| Mobility Restriction         | Determines if a UE is permitted to access a specific RAT, area, or slice based on subscription data and operator policies. Manages Forbidden Areas, Non-Allowed Areas, and Service Area Restrictions.                                                      | Allowed NSSAI, Forbidden TAC List, RAT Restriction, Roaming Restriction, Service Area Restriction                          |
| Paging management            | Responsible for paging, determines the paging strategy (paging area, paging priority).                                                                                                                                                                     | AI List, Registration Area, Paging Priority, UE Paging DRX                                                                 |
| UE 6GAP Management           | Manages the UE 6G Association (6GAP) between RAN and CN, including 6G connection establishment, release, and reconfiguration. Responsible for 6GAP message encoding and decoding.                                                                          | UE Association                                                                                                            |
| 6G Link Management           | Manages the link layer connection between RAN node and CN, including connection Setup and release, RAN node level info management, such as TA list.                                                                                                        | 6G Link State, SCTP Association                                                                                            |
| NAS Security                 | Manages the NAS layer security context, including the Security Mode Control (SMC) procedure, and the selection of algorithms for NAS ciphering and integrity protection, and the NAS message encryption/decryption.                                        | NAS Security Context, NAS INT/ENC Algorithm                                                                                |
| Reachability                 | Monitors and manages the reachability state of the UE, including monitoring periodic registration, reachability timer and implicit de-registration timer management                                                                                        | Reachability State, UE Activity Status, Maximum Latency/Response Time                                                      |
| Subscription Data Management | Subscribes to and retrieves UE subscription data from the UDM.                                                                                                                                                                                             | SUPI, Subscribed DNN, Subscribed S-NSSAI, Subscribed Session-AMBR                                                          |
| UE Policy                    | Retrieves UE-level mobility policies from the PCF, including dynamic policies such as access barring.                                                                                                                                                      | Policy Association ID (UE-level), UE Policy                                                                                |
| Mobility Procedure Control   | Controls the complete procedure for UE mobility: Registration procedures (Initial, Update), Service Request procedure, Handover procedure, and Mobility Registration Updates. Manages the mobility state machine and state transitions.                    | Registration Type, Registration State, Mobility State, Handover Type, Source/Target AMF, TAI List, Service Request Trigger |
| Network Function Selection   | Selects the target Network Function (e.g., SMF, PCF, AMF) during registration and mobility procedures. Selects the target SMF, PCF, and NSSF (for slice selection). Decisions are based on conditions such as UE location, slice information, and NF load. | DNN, S-NSSAI, UE Location (TAI), SMF Set ID, PCF ID, NSSF ID, NRF Query Result                                             |

- **Session Management Services**

| Service                   | Service Description                                                                                                                                                                                                          | Context                                                                                                         |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Session Procedure Control | Controls the complete lifecycle of a PDU Session: Establishment, Modification, and Release.                                                                                                                                  | PDU Session ID, PDU Session Type (IP/Ethernet/Unstructured), PDU Session State                                  |
| UP Session Control        | Manages the 6G N4 session between the SMF and the UPF, including 6G N4 Session Establishment, Modification, and Deletion. Configures PDR, FAR, QER, and URR rules, and manages UP Tunnels between UPs and between UP and RAN | 6G N4 Session ID, PDR ID, FAR ID, QER ID, URR ID, F-TEID, Apply Action, Precedence                              |
| Policy Control            | Retrieves PCC rules at the PDU Session level from the PCF.                                                                                                                                                                   | PCC Rule ID, 5QI, ARP, Session-AMBR, Gating Status, Reflective QoS Indicator                                    |
| Secondary Authentication  | Performs secondary authentication/authorization for a specific DNN (e.g., enterprise network) by validating the UE’s authorization to access a particular Data Network via the DN-AAA server.                                | SUPI, DNN, DN-AAA Server Address, EAP Message, Auth-Result, Authorization Data                                  |
| Charging Management       | Manages charging at the PDU Session level: interacts with the CHF to create charging sessions, generate CDRs, and report usage (Volume/Time/Event).                                                                          | Charging ID, CHF Address, Rating Group, Service ID, Usage Report, CDR Trigger, Charging Method (Online/Offline) |
| QoS Flow Management       | Establishes, modifies, and releases QoS Flows, and manages both the default and non-default QoS Flows.                                                                                                                       | QFI, 5QI, ARP, QoS Flow State, Default QoS Rule, GFBR/MFBR                                                      |
| Subnet Management         | Manages UE subscriptions for IP multicast/broadcast, supporting 6G LAN-type Services and Group Communication. Manages membership and user-plane topology for 5G VN Groups.                                                   | 6G VN Group ID, Group Member List, Multicast/Broadcast Session ID, 5G VN Group Data, DNN (LAN-type)             |
| RAN Session Control       | Manages RAN Session, including RAN session establishment, modification, and release, including configuration and reconfiguration of RAN session QoS profile                                                                  | PDU Session ID, QoS Profile                                                                                     |
| UE Control                | Manages UE session, including configuration and reconfiguration of UE Session QoS rules                                                                                                                                      | PDU Session ID, QoS rules                                                                                       |

- **UP Node Control Services**

| Service              | Service Description                                                                                    | Context                                                                                                    |
| -------------------- | --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| UP Node Management    | Manages the state of UPF nodes and the links between the SMF and UPFs.                                    | UPF ID, UPF State                                                                                    |
| --------------------- | --------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| UP Selection          | Selects the target UPF based on conditions such as DNN, S-NSSAI, UE location, DNAI, and UPF capabilities. | DNAI, UPF Profile, DNN, S-NSSAI, UE TAI, UPF Load, UPF Service Area                                   |
| IP Address Management | Allocates and manages IP addresses (IPv4/IPv6) or an Ethernet MAC address for the PDU Session.            | UE IP Address, IPv4 Address Pool, IPv6 Prefix, DHCP Server Config, MAC Address, IP Address Lease Time |

#### 6.2.Y.4 Example Procedure(s)

The following is an example of 6G Service Request procedure, controlled by the Global Coordination Function:

![image6](<NGA.T1839 Service-Granularity Resilience_v3_assets/image6.png>)

1. UE sends Service Request to the Global Coordination Function. To enable flexibility handling in Global Coordination Function, the mobility and session related parameters are composed into MM container and SM container respectively.

2. Global Coordination Function determines the signaling procedures to fulfil the UE request based on the request type and requirements, i.e., the services to be invoked, their invocation order, and the invocation conditions. This enables both static predefined procedures as a fixed set of services with determined order, and flexible procedures with complex branching based on runtime conditions and non-disruptive introduction of new features.

3. The Global Coordination Function sequentially invokes the atomic modular services, and waits for the result of each service invocation before proceeding. During the execution process, the Global Coordination Function is also capable of dynamically adjusting the invocation sequence or composing alternative service branches. This runtime adaptation is triggered by 3GPP pre-defined rules or operator runtime customized policies (e.g., from PCF or NWDAF), utilizing the intermediate outputs from previous services, prevailing network configurations, or error-handling procedures.

4. The Global Coordination Function finally determines whether to accept or reject the request based on the consolidated outcomes.
