# Phase 4: End-to-End Actor Workflows & State Machines — Voluntary Carbon Credit Marketplace

**Date**: 18 August 2026
**Author Role**: Lead Technical Product Manager & Systems Designer
**Scope**: End-to-end actor workflows and state transitions only. Phases 2–3 are treated as FIXED. No architecture redesign; no API request/response schemas (Phase 5).
**Baseline**:
- Domain facts: [phase1-voluntary-carbon-marketplace-research.md](file:///c:/carbon-credit-marketplace/research/phase1-voluntary-carbon-marketplace-research.md)
- Core Web2 architecture: [phase2-core-architecture.md](file:///c:/carbon-credit-marketplace/architecture/phase2-core-architecture.md)
- Blockchain domain research: [phase3a-blockchain-domain-refresh.md](file:///c:/carbon-credit-marketplace/research/phase3a-blockchain-domain-refresh.md)
- Blockchain architecture: [phase3-blockchain-architecture.md](file:///c:/carbon-credit-marketplace/architecture/phase3-blockchain-architecture.md)

---

## 0. Governing Principles & Cross-Cutting Conventions

### 0.1 Tokenization Starting Condition

Per Phase 3 §7 and the custody account status table (Phase 3 §4):

> **Tokenization is DISABLED for every registry at launch.** Gold Standard, Puro.earth, and Isometric tokenization is gated behind bilateral agreements that are ⚠ PROVISIONAL — not yet executed. Verra, ACR, and CAR are explicitly blocked with no published tokenization framework.

**Design rule applied throughout this document**: Every workflow below is designed to be **fully functional on the Phase 2 Web2 path alone**, with zero dependency on tokenization being active. Where tokenization is relevant, it appears as a clearly demarcated **optional sub-flow** gated by `RegistryConfig.registryTokenizationEnabled == true` for the credit's specific registry.

### 0.2 Actor Reference Matrix

| Actor | Keycloak Role | Permissions Summary | Device / Connectivity Profile |
|:---|:---|:---|:---|
| **Project Developer — Smallholder Farmer** | `PROJECT_DEVELOPER` | Register projects, upload monitoring data, submit spatial boundaries, view credit status | Mobile-first (Android); intermittent 2G/3G connectivity; may lack formal ID documents in some regions |
| **Project Developer — Institutional** | `PROJECT_DEVELOPER` | Same functional permissions; different KYC track | Desktop/laptop; reliable broadband; standard corporate documentation (DUNS, LEI, tax filings) |
| **VVB Auditor** | `VVB_AUDITOR` | Receive assignments, access project data and D-MRV summaries, submit validation/verification reports, approve/reject verification rounds | Desktop; reliable connectivity; accredited under ISO 14065 |
| **Registry** | *(External system — not a platform actor)* | Source-of-truth for credit state; issues serial numbers; confirms retirements | Interacted with via Registry Sync Service adapters (Phase 2 §4) |
| **Corporate Buyer** | `CORPORATE_BUYER` | Browse marketplace, place orders, execute purchases, retire credits, generate VCMI claim reports | Desktop; reliable connectivity; corporate procurement workflows |
| **Retail Buyer** | `RETAIL_BUYER` | Browse marketplace, purchase fractional/small volumes, retire credits | Mobile or desktop; consumer-grade connectivity |
| **Broker/Intermediary** | `BROKER` | List credits on behalf of sellers, match OTC counterparties, execute brokered trades, manage client portfolios | Desktop; reliable connectivity; may hold inventory positions |
| **Platform Admin/Operations** | `PLATFORM_ADMIN` | Manage registry sync, review flagged reconciliation issues, override stuck workflows, manage user KYC escalations, configure feature flags | Desktop; internal tooling access |

### 0.3 Failure State Conventions

Every workflow step specifies failure handling using three categories:

1. **Timeout**: What happens if a step does not complete within its expected SLA.
2. **Rejection/Dispute**: What happens if an actor contests or rejects an outcome.
3. **System Override**: What happens if Phase 2's registry-authoritative conflict resolution (Phase 2 §4.5) or Phase 3's emergency revocation (Phase 3 §4 Transition 5) fires mid-workflow.

### 0.4 Notation for Optional Tokenization Sub-Flows

Throughout this document, tokenization-dependent steps are enclosed in clearly marked blocks:

> 🔗 **OPTIONAL: Tokenization Sub-Flow** — *Activates only when `RegistryConfig.registryTokenizationEnabled == true` for this credit's registry.*

---

## 1. Workflow 1: Project Developer Onboarding & Land Verification

### 1.1 Overview

This workflow covers the complete journey from a new project developer (farmer or institution) arriving at the platform to having a verified, spatially-bounded project registered and eligible for credit issuance. It spans two Phase 2 services: **Identity & Access Service** (KYC) and **Project Lifecycle Service** (registration + D-MRV data submission).

### 1.2 KYC Track Differentiation

| KYC Dimension | Smallholder Farmer Track | Institutional Developer Track |
|:---|:---|:---|
| **Identity Documents** | Government ID (Aadhaar, national ID, voter ID); may accept community attestation + field agent verification where formal documents are unavailable | Corporate registration certificate, LEI/DUNS number, director IDs, UBO disclosure |
| **KYC Provider Flow** | Simplified e-KYC (Aadhaar-based OTP/biometric in India; SMS-OTP + photo ID upload elsewhere); field agent assisted onboarding option | Standard corporate KYC via third-party provider (Jumio/Onfido); automated document verification + sanctions screening |
| **Connectivity Accommodation** | Offline-capable mobile app that queues submissions; sync on connectivity; SMS-based status notifications | Standard web portal; email notifications |
| **Consent Management** | Plain-language consent in regional language (Hindi, Swahili, Portuguese per region); audio consent option for low-literacy users (Phase 2 §3.2 — DPDP Act requirement) | Standard digital consent form; corporate data-sharing agreement |
| **Sanctions/PEP Screening** | Basic screening against OFAC/EU/UN lists | Full screening: OFAC/EU/UN sanctions, PEP database, adverse media, UBO analysis |
| **KYC Refresh Cycle** | Every 24 months (lower risk profile) | Every 12 months (higher transaction volumes) |

### 1.3 Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Project Developer<br/>(Farmer or Institutional)
    participant App as Platform App<br/>(Mobile / Web)
    participant GW as API Gateway<br/>(Kong)
    participant IAS as Identity & Access<br/>Service (Keycloak)
    participant KYC as KYC/AML Provider<br/>(Jumio / Onfido)
    participant PII as PII Vault
    participant PLS as Project Lifecycle<br/>Service
    participant DMRV as D-MRV Ingestion<br/>Service
    participant KF as Kafka
    participant NRS as Notification &<br/>Reporting Service
    participant Admin as Platform Admin

    Note over Dev,Admin: === PHASE A: Identity Verification (KYC) ===

    Dev->>App: Sign up (email/phone + password or passkey)
    App->>GW: POST /auth/register
    GW->>IAS: Create user account (Keycloak)
    IAS->>IAS: Assign role: PROJECT_DEVELOPER<br/>Status: KYC_PENDING
    IAS-->>App: Account created, KYC required

    alt Smallholder Farmer Track
        Dev->>App: Upload photo ID (or Aadhaar OTP)
        App->>App: Offline queue if no connectivity
        App->>GW: POST /kyc/submit (when connected)
        GW->>IAS: Forward KYC submission
        IAS->>KYC: Simplified e-KYC verification
        IAS->>PII: Store identity docs (AES-256 encrypted)
    else Institutional Developer Track
        Dev->>App: Upload corporate docs + director IDs + UBO
        App->>GW: POST /kyc/submit
        GW->>IAS: Forward KYC submission
        IAS->>KYC: Full corporate KYC + sanctions + PEP screening
        IAS->>PII: Store corporate KYC package
    end

    alt KYC Approved
        KYC-->>IAS: Verification PASSED
        IAS->>IAS: Status: KYC_VERIFIED
        IAS->>PII: Store consent record + pseudonym mapping
        IAS->>KF: Publish UserKYCVerified event
        KF->>NRS: Consume event
        NRS-->>Dev: "KYC approved — you can now register projects"
    else KYC Requires Manual Review
        KYC-->>IAS: Verification INCONCLUSIVE
        IAS->>IAS: Status: KYC_MANUAL_REVIEW
        IAS->>KF: Publish KYCEscalation event
        KF->>Admin: Manual review queue entry
        Admin->>IAS: Approve / Reject / Request additional docs
        alt Admin Approves
            IAS->>IAS: Status: KYC_VERIFIED
            NRS-->>Dev: "KYC approved"
        else Admin Rejects
            IAS->>IAS: Status: KYC_REJECTED
            NRS-->>Dev: "KYC rejected — reason: [X]. You may resubmit."
        else Admin Requests More Docs
            IAS->>IAS: Status: KYC_DOCS_REQUESTED
            NRS-->>Dev: "Additional documents required: [list]"
            Note over Dev,IAS: Developer resubmits → loop back
        end
    else KYC Failed (sanctions match / fraud)
        KYC-->>IAS: Verification FAILED — sanctions hit
        IAS->>IAS: Status: KYC_BLOCKED
        IAS->>KF: Publish KYCBlocked event (audit trail)
        NRS-->>Dev: "Account blocked — contact support"
        Note over IAS: Account permanently blocked.<br/>Requires legal review to unblock.
    end

    Note over Dev,Admin: === PHASE B: Project Registration ===

    Dev->>App: Create new project (metadata form)
    App->>GW: POST /projects
    GW->>PLS: Validate project data
    PLS->>PLS: Assign project_id (UUID)<br/>Status: DRAFT
    PLS-->>App: Project created (DRAFT)

    Dev->>App: Upload Project Design Document (PDD)
    App->>GW: POST /projects/{id}/documents
    GW->>PLS: Store PDD reference
    PLS->>PLS: PDD uploaded → Status: DRAFT (awaiting boundary data)

    Note over Dev,Admin: === PHASE C: Spatial Boundary & D-MRV Data ===

    Dev->>App: Draw/upload project boundary (GeoJSON polygon)
    App->>GW: POST /projects/{id}/boundary
    GW->>PLS: Forward spatial data
    PLS->>DMRV: Ingest boundary polygon
    DMRV->>DMRV: Validate polygon (topology check,<br/>overlap detection with existing projects)

    alt Boundary Valid
        DMRV->>DMRV: Store in PostGIS
        DMRV->>DMRV: Trigger satellite baseline acquisition<br/>(Sentinel-2/PlanetScope for AOI)
        DMRV->>KF: Publish BoundaryValidated event
        KF->>PLS: Update project status: BOUNDARY_VERIFIED
        PLS-->>App: Boundary accepted
    else Boundary Invalid (overlap / topology error)
        DMRV-->>PLS: Validation failed (reason: overlap with Project X)
        PLS-->>App: "Boundary rejected — [reason]. Please re-submit."
        Note over Dev: Developer corrects boundary → resubmit
    end

    Dev->>App: Submit project for registration review
    App->>GW: POST /projects/{id}/submit
    GW->>PLS: Transition status: DRAFT → SUBMITTED
    PLS->>KF: Publish ProjectSubmitted event
    KF->>NRS: Notify admin of new project submission
    KF->>NRS: Confirm to developer: "Project submitted for review"

    Note over Dev,Admin: === PHASE D: Platform Review & Registry Listing ===

    Admin->>PLS: Review project submission
    alt Project Approved
        Admin->>PLS: Approve → Status: LISTED
        PLS->>KF: Publish ProjectListed event
        NRS-->>Dev: "Project approved and listed"
    else Project Needs Revision
        Admin->>PLS: Request revision → Status: REVISION_REQUESTED
        NRS-->>Dev: "Revision needed: [details]"
        Note over Dev: Developer revises → resubmit loop
    else Project Rejected
        Admin->>PLS: Reject → Status: REJECTED
        NRS-->>Dev: "Project rejected: [reason]"
    end
```

### 1.4 Failure States & Recovery

| Step | Failure Scenario | Recovery Action |
|:---|:---|:---|
| **KYC submission** | Timeout: KYC provider does not respond within 120s | Retry with exponential backoff (3 attempts). If persistent, queue for manual review. User sees "Verification in progress — we'll notify you." |
| **KYC submission** | Farmer loses connectivity mid-upload | Offline-capable app stores submission locally. Auto-retries on reconnection. Partial uploads resume via chunked upload protocol. |
| **KYC submission** | Farmer has no formal ID documents | Field agent assisted flow: community leader attestation + field agent photo verification + GPS-stamped site visit. Escalated to admin manual review track. |
| **Boundary validation** | D-MRV Service detects overlap with existing project | Developer is shown the conflicting project boundary (anonymized) on a map. Must adjust polygon and resubmit. |
| **Boundary validation** | Satellite data unavailable for AOI (cloud cover, no recent pass) | D-MRV Service queues a future acquisition request. Project proceeds to SUBMITTED without baseline — satellite data is acquired asynchronously and attached when available. |
| **Project review** | Admin does not act within 5 business days | Escalation alert to senior admin. Developer notified: "Your project is under extended review." SLA: 10 business days max before escalation to operations lead. |
| **Registry-authoritative override mid-flow** | Registry Sync detects that the project was rejected/delisted on the source registry after platform listing | Credit Ledger Service transitions project to CANCELLED. Developer notified with registry reason. No credits can be issued. |

---

## 2. Workflow 2: VVB Audit Workflow

### 2.1 Overview

This workflow covers the end-to-end VVB (Validation and Verification Body) engagement: from assignment to final report submission. It maps directly to Phase 2's `VERIFICATION` entity and the Project Lifecycle Service's `LISTED → VALIDATED → VERIFIED` state transitions.

### 2.2 VVB Assignment Model

| Dimension | Design |
|:---|:---|
| **Assignment mechanism** | Developer selects from a platform-curated list of accredited VVBs eligible for the project's methodology. Platform enforces no conflict-of-interest rules (validator ≠ verifier for same monitoring period). |
| **VVB eligibility filtering** | ABAC policy in Keycloak: VVB must hold accreditation for the project's registry + methodology + geography (Phase 2 §2.1 ABAC extensions). |
| **Engagement contract** | Off-platform bilateral agreement between developer and VVB. Platform records the assignment but does not intermediate payment. |
| **Rotation requirement** | Platform enforces registry-mandated VVB rotation rules (e.g., Verra requires rotation every N verification periods). |

### 2.3 Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Project Developer
    participant PLS as Project Lifecycle<br/>Service
    participant VVB as VVB Auditor
    participant DMRV as D-MRV Ingestion<br/>Service
    participant ObjStore as Object Storage<br/>(MinIO/S3)
    participant KF as Kafka
    participant NRS as Notification &<br/>Reporting Service
    participant Admin as Platform Admin

    Note over Dev,Admin: === PHASE A: Validation (Pre-Issuance) ===

    Dev->>PLS: Request VVB assignment for VALIDATION
    PLS->>PLS: List eligible VVBs (methodology + accreditation filter)
    PLS-->>Dev: VVB shortlist presented
    Dev->>PLS: Select VVB-A for validation engagement
    PLS->>PLS: Record assignment (VVB-A, project_id, type=VALIDATION)<br/>Create VERIFICATION record (status: ASSIGNED)
    PLS->>KF: Publish VVBAssigned event
    KF->>NRS: Notify VVB-A: "New validation assignment"

    VVB->>PLS: Accept assignment
    PLS->>PLS: VERIFICATION status: IN_PROGRESS

    VVB->>PLS: Access project data (PDD, boundary, methodology)
    VVB->>DMRV: Access D-MRV baseline data (satellite imagery, IoT summaries)

    alt Validation Requires Site Visit
        VVB->>VVB: Conduct site visit (off-platform)
        VVB->>PLS: Upload site visit report
    end

    alt Validation Passes
        VVB->>ObjStore: Upload validation report (PDF)
        VVB->>PLS: Submit validation decision: APPROVED
        PLS->>PLS: VERIFICATION status: COMPLETED (type=VALIDATION)
        PLS->>PLS: Project status: LISTED → VALIDATED
        PLS->>KF: Publish ProjectValidated event
        KF->>NRS: Notify developer: "Project validated by VVB-A"
    else Validation Fails — Correctable Issues
        VVB->>PLS: Submit validation decision: REVISIONS_REQUIRED
        VVB->>PLS: Attach revision notes (specific issues)
        PLS->>PLS: VERIFICATION status: REVISIONS_REQUESTED
        KF->>NRS: Notify developer: "Validation revisions needed"
        Dev->>PLS: Address issues, upload revised PDD
        Dev->>PLS: Resubmit for validation
        PLS->>PLS: VERIFICATION status: IN_PROGRESS (new round)
        Note over VVB: VVB re-reviews → loop back to decision
    else Validation Rejected — Fundamental Issues
        VVB->>PLS: Submit validation decision: REJECTED
        VVB->>ObjStore: Upload rejection report
        PLS->>PLS: VERIFICATION status: REJECTED
        PLS->>PLS: Project status remains LISTED (cannot advance)
        KF->>NRS: Notify developer: "Validation rejected — [reason]"
        Note over Dev: Developer may appeal to platform admin<br/>or engage a different VVB for fresh validation
    end

    Note over Dev,Admin: === PHASE B: Verification (Post-Monitoring) ===

    Dev->>PLS: Submit monitoring report for verification period
    Dev->>ObjStore: Upload monitoring report (PDF + data appendices)
    Dev->>DMRV: D-MRV data already ingested (continuous pipeline)
    PLS->>PLS: Create MONITORING_REPORT record

    Dev->>PLS: Request VVB assignment for VERIFICATION
    PLS->>PLS: List eligible VVBs (must differ from validator if registry requires)
    Dev->>PLS: Select VVB-B for verification engagement
    PLS->>PLS: Create VERIFICATION record (type=VERIFICATION, status: ASSIGNED)
    KF->>NRS: Notify VVB-B: "New verification assignment"

    VVB->>PLS: Accept assignment
    PLS->>PLS: VERIFICATION status: IN_PROGRESS

    VVB->>PLS: Access monitoring report + project data
    VVB->>DMRV: Access D-MRV telemetry for verification period<br/>(satellite change detection, IoT time-series, biomass estimates)

    alt Verification Passes
        VVB->>ObjStore: Upload verification report + verification statement
        VVB->>PLS: Submit verification decision: APPROVED<br/>(confirmed volume: N tCO₂e)
        PLS->>PLS: VERIFICATION status: COMPLETED (type=VERIFICATION)
        PLS->>PLS: Project status: VALIDATED → VERIFIED
        PLS->>KF: Publish ProjectVerified event (volume: N tCO₂e)
        KF->>NRS: Notify developer: "Verification complete — N credits eligible for issuance"
    else Verification — Volume Adjustment
        VVB->>PLS: Submit verification: APPROVED with ADJUSTED volume
        VVB->>PLS: Provide justification for volume reduction
        PLS->>PLS: Record adjusted volume (developer can view rationale)
        Note over Dev: Developer can dispute volume adjustment<br/>(see §2.4 Dispute Handling)
    else Verification Rejected
        VVB->>PLS: Submit verification decision: REJECTED
        VVB->>ObjStore: Upload rejection report
        PLS->>PLS: VERIFICATION status: REJECTED
        PLS->>PLS: Project status remains VALIDATED (cannot advance to VERIFIED)
        KF->>NRS: Notify developer: "Verification rejected — [reason]"
        Note over Dev: Developer can appeal (see §2.4)<br/>or re-engage after addressing issues
    end
```

### 2.4 Dispute Handling: Developer Contests VVB Decision

| Dispute Scenario | Resolution Process |
|:---|:---|
| Developer disputes **validation rejection** | Developer files dispute with Platform Admin. Admin reviews VVB report + project data. Options: (a) request VVB to reconsider with specific points, (b) allow developer to engage alternate VVB for independent re-validation, (c) uphold rejection. Timeline: 15 business days. |
| Developer disputes **verification volume adjustment** | Developer submits counter-evidence (additional D-MRV data, alternative calculations). Admin mediates between developer and VVB. If unresolved, admin may commission an independent third-party technical review (cost borne by developer). Timeline: 20 business days. |
| Developer disputes **verification rejection** | Same escalation as validation rejection. Developer may engage alternate VVB. If second VVB also rejects, project is flagged and cannot proceed without admin override. |
| VVB fails to deliver report within agreed timeline | After 2× the agreed deadline, developer can request VVB re-assignment. Platform cancels the current VERIFICATION record (status: TIMED_OUT) and developer selects a new VVB. No penalty to developer. |

### 2.5 Failure States & Recovery

| Step | Failure Scenario | Recovery Action |
|:---|:---|:---|
| **VVB assignment** | No eligible VVBs available for methodology/geography | Platform Admin notified. Developer informed of wait time. Platform onboards additional VVBs as needed. |
| **VVB report upload** | Upload fails (large PDF, network timeout) | Chunked upload with resume. ObjStore returns pre-signed upload URL valid for 24h. Retry from last successful chunk. |
| **VVB report submission** | VVB submits incomplete report (missing verification statement) | PLS validates report completeness against checklist before accepting. Returns specific missing items to VVB. |
| **Timeout** | VVB does not accept assignment within 10 business days | Assignment auto-expires. Developer is notified and prompted to select another VVB. |
| **Timeout** | VVB does not complete review within registry-prescribed timeline | Escalation to admin. Warning sent to VVB. If unresolved in 30 days, VVB's platform accreditation is reviewed. |
| **Registry override** | Registry revokes VVB's accreditation during active engagement | Platform Admin immediately reassigns. Active VERIFICATION record is voided (status: CANCELLED — VVB_DISQUALIFIED). Developer selects new VVB. |
| **Registry override** | Registry delists or cancels the project during verification | Project status → CANCELLED via registry-authoritative reconciliation (Phase 2 §4.5). Active verification is voided. Developer and VVB notified. |

---

## 3. Workflow 3: Registry Issuance → Platform Activation

### 3.1 Overview

This workflow traces the path from registry-confirmed credit issuance (detected by the Registry Sync Service per Phase 2 §4) through the credit becoming `ACTIVE` and listable on the marketplace. It then shows the **optional** tokenization sub-flow (Phase 3 §4 lock→mint) as a gated branch.

### 3.2 Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Reg as Source Registry<br/>(e.g., Gold Standard)
    participant Adapter as Registry Adapter<br/>(Phase 2 §4.3)
    participant KF as Kafka
    participant Norm as Event Normalizer<br/>& Conflict Detector
    participant CLS as Credit Ledger<br/>Service
    participant PG as PostgreSQL<br/>(Credits DB)
    participant Search as OpenSearch
    participant NRS as Notification &<br/>Reporting Service
    participant Dev as Project Developer
    participant Admin as Platform Admin

    Note over Reg,Admin: === PHASE A: Registry Issuance Detection ===

    Reg->>Reg: Registry mints credits for Project X<br/>(serial numbers assigned, buffer pool deducted)

    Adapter->>Reg: Poll for new issuances (since last checkpoint)
    Reg-->>Adapter: Issuance data (serial numbers, volume,<br/>vintage, buffer deduction, methodology)
    Adapter->>Adapter: Transform to RegistrySyncEvent schema
    Adapter->>KF: Publish to registry.{name}.raw<br/>(idempotency_key: REG-PROJ-ISSUE-DATE)
    Adapter->>Adapter: Update checkpoint offset

    KF->>Norm: Consume raw event
    Norm->>Norm: Validate schema, normalize serial numbers,<br/>deduplicate against known events
    alt Duplicate Event
        Norm->>Norm: No-op (idempotency_key already processed)
    else New Valid Event
        Norm->>KF: Publish to registry.normalized
    end

    Note over Reg,Admin: === PHASE B: Credit Activation ===

    KF->>CLS: Consume normalized CreditIssuanceEvent
    CLS->>PG: BEGIN TRANSACTION
    CLS->>PG: Insert CREDIT_BATCH (project_id, vintage, quantity,<br/>buffer_pool_deduction, registry_issuance_ref)
    CLS->>PG: Insert CREDIT rows (status=ISSUED, serial_numbers,<br/>quality_labels from project metadata)
    CLS->>PG: Insert CREDIT_EVENT rows (type=ISSUED)
    CLS->>PG: Insert OWNERSHIP_RECORD rows<br/>(owner_ref = developer's pseudonym_hash)
    CLS->>PG: COMMIT
    
    CLS->>CLS: Apply quality labels (CCP status, CORSIA eligibility,<br/>Article 6 authorization — from project metadata)
    CLS->>CLS: Transition: ISSUED → ACTIVE

    CLS->>KF: Publish CreditActivated event (batch_id, count, quality_labels)

    KF->>Search: CDC → Index credits in OpenSearch<br/>(now discoverable in marketplace search)
    KF->>NRS: Consume activation event
    NRS-->>Dev: "X credits issued and activated for Project Y.<br/>They are now listable on the marketplace."

    Note over Reg,Admin: === PHASE C: Quality Label Verification ===

    CLS->>CLS: Cross-reference quality labels against<br/>ICVCM CCP methodology list (cached, refreshed daily)
    alt Labels Verified
        CLS->>CLS: quality_labels confirmed
    else Methodology Not in CCP List
        CLS->>CLS: Remove CCP_APPROVED label if present
        CLS->>KF: Publish QualityLabelUpdated event
        NRS-->>Dev: "Note: credits do not currently carry CCP-Approved status"
    end
```

### 3.3 Optional Tokenization Sub-Flow

> 🔗 **OPTIONAL: Tokenization Sub-Flow** — *Activates only when `RegistryConfig.registryTokenizationEnabled == true` for this credit's registry. Per Phase 3 §7, this is currently `false` for ALL registries pending bilateral agreement execution.*

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Project Developer
    participant P2API as Phase 2 API
    participant CLS as Credit Ledger<br/>Service
    participant RSS as Registry Sync<br/>Service
    participant Reg as Source Registry
    participant KF as Kafka
    participant BBS as Blockchain<br/>Bridge Service
    participant AV as AttestationVerifier<br/>(Polygon)
    participant CV as CreditVault<br/>(Polygon)
    participant IR as IdentityRegistry<br/>(Polygon)
    participant NRS as Notification Service

    Note over Dev,NRS: Gate check: RegistryConfig.registryTokenizationEnabled[registry] must be true

    Dev->>P2API: POST /credits/{batch_id}/tokenize
    P2API->>CLS: Validate: credits are ACTIVE, registry tokenization enabled,<br/>developer has valid ONCHAINID
    
    alt Tokenization Not Enabled for Registry
        CLS-->>P2API: 403 Forbidden — tokenization disabled for this registry
        P2API-->>Dev: "Tokenization is not available for [registry] credits"
        Note over Dev: Workflow ends. Credits remain<br/>as Web2-only ACTIVE credits.
    else Tokenization Enabled
        CLS->>CLS: Lock credits (ACTIVE → ACTIVE_LOCKED,<br/>lock_reason: TOKENIZATION)
        CLS->>KF: Publish CreditLockedForTokenization event

        KF->>RSS: Consume lock event
        RSS->>Reg: Transfer credits to platform custody account (API)
        
        alt Custody Transfer Succeeds
            Reg-->>RSS: Custody confirmed
            RSS->>KF: Publish CustodyConfirmed event

            KF->>BBS: Consume CustodyConfirmed
            BBS->>BBS: Build CreditAttestation (type=MINT, EIP-712 signed)
            BBS->>AV: submitAttestation(signedAttestation)
            AV->>CV: executeMint(tokenId, amount, metadata)
            CV->>IR: isVerified(developer.smartAccount)?
            IR-->>CV: ✓ Verified
            CV->>CV: Mint tokens to developer's smart account
            
            BBS->>KF: Publish CreditTokenized event (txHash, tokenId)
            KF->>CLS: Update status: ACTIVE_LOCKED → TOKENIZED
            NRS-->>Dev: "Credits tokenized on Polygon ✓"
        else Custody Transfer Fails
            Reg-->>RSS: Transfer failed (reason)
            RSS->>KF: Publish CustodyFailed event
            KF->>CLS: Unlock credits (ACTIVE_LOCKED → ACTIVE)
            NRS-->>Dev: "Tokenization failed — credits remain as Web2 credits"
        end
    end
```

### 3.4 Failure States & Recovery

| Step | Failure Scenario | Recovery Action |
|:---|:---|:---|
| **Registry polling** | Adapter cannot reach registry API (timeout / 5xx) | Exponential backoff with jitter. After 3 failures, adapter marked unhealthy. Admin alerted via sync lag dashboard (Phase 2 §4.6). No credits are created from stale data. |
| **Registry polling** | Scraper-based adapter (ACR/CAR) returns >20% fewer records than previous run | Anomaly detection triggers quarantine. Scrape results held for admin review. Not auto-ingested. |
| **Duplicate issuance** | Same issuance event received twice | Idempotency key deduplication in Credit Ledger Service. No-op. |
| **Serial number collision** | Two registries claim same serial number | Cross-registry collision detection (Phase 2 §4.5). Alert + admin manual review. Should be impossible per registry serialization rules. |
| **Quality label conflict** | Registry reports methodology that is not in ICVCM CCP-Approved list | Credit is activated without CCP label. Quality label can be updated later if ICVCM approves the methodology. |
| **Tokenization: custody transfer timeout** | Registry does not confirm custody transfer within 30 minutes | Credit Ledger Service keeps lock for 2 hours. If still unconfirmed, auto-unlock credits back to ACTIVE. Admin alerted for investigation. |
| **Tokenization: on-chain mint fails** | Polygon transaction reverts (gas, contract error) | Credits remain in registry custody. Bridge Service retries 3×. If persistent, admin intervenes. Credits stay ACTIVE_LOCKED until resolved (never lost). |
| **Registry-authoritative override** | Registry cancels/revokes credits after platform activation | Credit Ledger Service transitions credits to CANCELLED (Phase 2 §4.5 — registry wins). If credits were tokenized, emergency revocation fires (Phase 3 §4 Transition 5). |

---

## 4. Workflow 4: Primary Sale and Secondary Trading

### 4.1 Overview

This workflow covers four distinct trade execution paths:
1. **Web2 marketplace purchase** (direct ownership transfer in Credit Ledger Service)
2. **On-chain purchase** (for tokenized credits, ERC-3643 compliant transfer)
3. **Broker-facilitated OTC trade** (bilateral negotiation with broker intermediation)
4. **Discovery/search** via OpenSearch (Phase 2 §2.7)

### 4.2 Marketplace Discovery (All Paths)

```mermaid
sequenceDiagram
    autonumber
    participant Buyer as Corporate/Retail Buyer
    participant GW as API Gateway
    participant MTS as Marketplace &<br/>Trading Service
    participant Search as OpenSearch
    participant Redis as Redis Cache
    participant CLS as Credit Ledger<br/>Service

    Buyer->>GW: GET /marketplace/search?registry=GS&ccp=true&vintage=2025&country=IND
    GW->>MTS: Forward search request
    MTS->>Redis: Check query cache (TTL: 30-60s)
    
    alt Cache Hit
        Redis-->>MTS: Cached results
    else Cache Miss
        MTS->>Search: Faceted search query (OpenSearch)
        Search-->>MTS: Matching credit listings with facets<br/>(project, vintage, price, quality labels, SDG impacts)
        MTS->>Redis: Cache results (TTL: 30s)
    end
    
    MTS-->>Buyer: Search results (paginated, faceted)

    Buyer->>GW: GET /marketplace/credits/{credit_id}/details
    GW->>MTS: Forward detail request
    MTS->>CLS: Fetch credit state + ownership + quality labels
    CLS-->>MTS: Credit details (status, labels, D-MRV summary hash)
    MTS-->>Buyer: Full credit details + pricing + seller info (pseudonymized)
```

### 4.3 Path A: Web2 Marketplace Purchase (Primary Path)

```mermaid
sequenceDiagram
    autonumber
    participant Buyer as Corporate/Retail Buyer
    participant GW as API Gateway
    participant MTS as Marketplace &<br/>Trading Service
    participant CLS as Credit Ledger<br/>Service
    participant PG as PostgreSQL
    participant Redis as Redis
    participant KF as Kafka
    participant Pay as Payment Gateway<br/>⚠ PROVISIONAL
    participant NRS as Notification Service
    participant Seller as Seller<br/>(Developer or Current Owner)

    Buyer->>GW: POST /marketplace/orders (credit_ids, quantity)
    GW->>MTS: Create purchase order

    MTS->>CLS: Validate credits are ACTIVE and available
    CLS->>PG: Check credit status + version (optimistic lock)
    CLS->>CLS: Verify staleness: last_registry_sync < freshness_window (15 min)
    
    alt Credits Stale (sync too old)
        CLS->>CLS: Trigger on-demand re-sync for affected credits
        CLS-->>MTS: "Credits pending freshness verification — retry in 60s"
        MTS-->>Buyer: "Order processing — credit verification in progress"
    else Credits Fresh and Available
        CLS->>Redis: Acquire distributed lock: credit:{serial}:lock (TTL: 30s)
        CLS->>PG: Lock credits (status → SALE_PENDING, version++)
        
        MTS->>MTS: Create ORDER record (status: PAYMENT_PENDING)
        MTS->>KF: Publish OrderCreated event

        Note over Buyer,Pay: ⚠ PROVISIONAL: Payment rail details<br/>require dedicated payments-compliance research

        MTS-->>Buyer: Order created — proceed to payment
        Buyer->>Pay: Execute payment (fiat)
        
        alt Payment Confirmed
            Pay-->>MTS: Payment confirmation (tx_ref)
            MTS->>MTS: ORDER status: PAYMENT_CONFIRMED

            MTS->>CLS: Execute ownership transfer
            CLS->>PG: BEGIN TRANSACTION
            CLS->>PG: Update CREDIT.current_owner_ref → buyer pseudonym_hash
            CLS->>PG: Insert CREDIT_EVENT (type=TRANSFERRED)
            CLS->>PG: Insert OWNERSHIP_RECORD (buyer, acquired_at)
            CLS->>PG: Close OWNERSHIP_RECORD (seller, released_at)
            CLS->>PG: Update CREDIT status: SALE_PENDING → ACTIVE (new owner)
            CLS->>PG: COMMIT
            CLS->>Redis: Release distributed lock

            CLS->>KF: Publish CreditTransferred event
            MTS->>MTS: ORDER status: COMPLETED

            KF->>NRS: Consume transfer event
            NRS-->>Buyer: "Purchase complete — X credits now in your account"
            NRS-->>Seller: "Your credits have been sold and transferred"
        
        else Payment Failed
            Pay-->>MTS: Payment failed (reason)
            MTS->>MTS: ORDER status: PAYMENT_FAILED
            MTS->>CLS: Release credit locks
            CLS->>PG: Unlock credits (SALE_PENDING → ACTIVE)
            CLS->>Redis: Release distributed lock
            NRS-->>Buyer: "Payment failed — credits released. Please retry."
        
        else Payment Timeout (no confirmation within 30 min)
            MTS->>MTS: ORDER status: PAYMENT_TIMED_OUT
            MTS->>CLS: Release credit locks
            CLS->>PG: Unlock credits (SALE_PENDING → ACTIVE)
            CLS->>Redis: Release distributed lock
            NRS-->>Buyer: "Payment window expired — credits released"
        end
    end
```

### 4.4 Path B: On-Chain Transfer (Tokenized Credits Only)

> 🔗 **OPTIONAL: On-Chain Path** — *Only available for credits where tokenization is active.*

```mermaid
sequenceDiagram
    autonumber
    participant Buyer as Buyer<br/>(Smart Account)
    participant SDK as Platform SDK
    participant Bundler as Bundler<br/>(Pimlico)
    participant EP as EntryPoint<br/>(ERC-4337)
    participant PM as Paymaster
    participant CT as CarbonCreditToken<br/>(ERC-1155)
    participant IR as IdentityRegistry
    participant CM as ComplianceModule
    participant BBS as Blockchain<br/>Bridge Service
    participant KF as Kafka
    participant CLS as Credit Ledger<br/>Service

    Note over Buyer,CLS: Preconditions: Both buyer and seller have<br/>verified ONCHAINIDs with valid claims

    Buyer->>SDK: "Buy 5,000 tokens of tokenId X from Seller"
    SDK->>SDK: Build UserOperation<br/>(calldata = safeTransferFrom(seller, buyer, tokenId, 5000))
    SDK->>Bundler: Submit UserOperation
    Bundler->>PM: Verify gas sponsorship (user whitelisted?)
    PM-->>Bundler: Sponsored ✓
    Bundler->>EP: Submit bundled transaction

    EP->>CT: safeTransferFrom(seller, buyer, tokenId, 5000)
    CT->>IR: isVerified(seller)? isVerified(buyer)?
    
    alt Both Verified
        IR-->>CT: ✓ Both verified
        CT->>CM: canTransfer(seller, buyer, tokenId, 5000)?
        alt Compliant
            CM-->>CT: ✓ Compliant
            CT->>CT: Transfer token balances atomically
            CT-->>EP: Transfer complete
            EP-->>Bundler: Transaction receipt
            Bundler-->>SDK: UserOp receipt
            SDK-->>Buyer: "5,000 carbon credits transferred ✓"

            Note over BBS,CLS: Async callback to Phase 2

            BBS->>BBS: Detect Transfer event on-chain
            BBS->>KF: Publish CreditTransferred event
            KF->>CLS: Update ownership (new owner_ref = buyer pseudonym_hash)
        else Non-Compliant (jurisdiction, sanctions, max holding)
            CM-->>CT: ✗ Transfer blocked (rule: RULE_JURISDICTION)
            CT-->>EP: Transaction REVERTED
            EP-->>Bundler: Revert
            Bundler-->>SDK: UserOp failed
            SDK-->>Buyer: "Transfer blocked — compliance check failed: [reason]"
        end
    else Identity Not Verified
        IR-->>CT: ✗ Buyer/seller not verified
        CT-->>EP: Transaction REVERTED
        SDK-->>Buyer: "Transfer blocked — identity verification required"
    end
```

### 4.5 Path C: Broker-Facilitated OTC Trade

```mermaid
sequenceDiagram
    autonumber
    participant Seller as Credit Seller<br/>(Developer or Owner)
    participant Broker as Broker /<br/>Intermediary
    participant GW as API Gateway
    participant MTS as Marketplace &<br/>Trading Service
    participant CLS as Credit Ledger<br/>Service
    participant KF as Kafka
    participant NRS as Notification Service
    participant CorpBuyer as Corporate Buyer

    Note over Seller,CorpBuyer: OTC trades differ from marketplace purchases:<br/>bilateral negotiation, custom pricing,<br/>potentially large volumes, forward commitments

    Seller->>Broker: Off-platform: agree to sell N credits at price P
    Broker->>CorpBuyer: Off-platform: negotiate deal terms

    Note over Broker: Broker structures deal terms off-platform.<br/>Platform records the execution only.

    Broker->>GW: POST /otc/trades (seller_id, buyer_id, credit_ids,<br/>quantity, agreed_price, deal_reference)
    GW->>MTS: Create OTC trade record
    
    MTS->>CLS: Validate credits available and owned by seller
    CLS-->>MTS: Credits validated

    MTS->>MTS: Create OTC_TRADE record (status: PENDING_CONFIRMATION)
    MTS->>KF: Publish OTCTradeCreated event

    KF->>NRS: Notify seller: "Confirm OTC trade #[ref]"
    KF->>NRS: Notify buyer: "Confirm OTC trade #[ref]"

    Seller->>GW: POST /otc/trades/{id}/confirm (seller confirms)
    MTS->>MTS: Seller confirmed

    CorpBuyer->>GW: POST /otc/trades/{id}/confirm (buyer confirms)
    MTS->>MTS: Buyer confirmed

    MTS->>MTS: Both parties confirmed → execute settlement

    Note over MTS,CLS: ⚠ PROVISIONAL: Payment settlement for OTC trades<br/>happens off-platform (wire transfer between parties).<br/>Platform records payment confirmation from broker.

    Broker->>GW: POST /otc/trades/{id}/payment-confirmed
    MTS->>MTS: OTC_TRADE status: SETTLING

    MTS->>CLS: Execute ownership transfer (same as marketplace)
    CLS->>CLS: Transfer ownership: seller → buyer (ACID transaction)
    CLS->>KF: Publish CreditTransferred event
    MTS->>MTS: OTC_TRADE status: COMPLETED

    KF->>NRS: Notify all parties: "OTC trade settled"
```

### 4.6 Failure States & Recovery

| Step | Failure Scenario | Recovery Action |
|:---|:---|:---|
| **Search** | OpenSearch cluster unavailable | MTS falls back to direct PostgreSQL query (slower, limited faceting). Degraded experience, not outage. |
| **Credit freshness** | Credit Ledger detects stale registry sync (>15 min) | On-demand re-sync triggered. Order held in PENDING until freshness confirmed or 5-minute timeout. |
| **Distributed lock** | Redis lock acquisition fails (contention) | Retry with jitter (3 attempts over 5s). If persistent, order rejected with "credits temporarily unavailable — please retry." |
| **Payment** | Payment gateway timeout (>30 min) | Credits auto-released from SALE_PENDING → ACTIVE. Order marked PAYMENT_TIMED_OUT. Buyer can re-attempt. |
| **Payment** | Payment succeeds but Credit Ledger update fails | Saga compensation: MTS detects CLS failure, queues compensating refund via payment gateway. Order marked SETTLEMENT_FAILED — admin review required. |
| **On-chain transfer** | Compliance Module rejects transfer | Transaction reverts cleanly. No state change. Buyer informed of specific rule violation. |
| **On-chain transfer** | Bridge Service fails to detect on-chain Transfer event | Reconciliation job (every 15 min) detects on-chain transfer without matching off-chain record. Auto-replays from event log. |
| **OTC trade** | One party does not confirm within 72 hours | Trade expires. Status → EXPIRED. Credits remain with seller. All parties notified. |
| **OTC trade** | Buyer disputes credit quality after settlement | See §10 (Riskiest Workflow — Dispute Resolution). Quality disputes are the most complex failure mode. |
| **Registry-authoritative override** | Registry retires credit during active sale | Credit Ledger transitions credit to RETIRED (registry wins). If SALE_PENDING, order is cancelled and buyer refunded. If already transferred, buyer is notified of registry retirement — credit is no longer tradeable. |
| **Emergency revocation (Phase 3)** | Registry revokes tokenized credit mid-transfer | Emergency revocation fires (Phase 3 §4 Transition 5). On-chain tokens force-burned. Buyer compensated per incident response process. |

---

## 5. Workflow 5: Credit Retirement

### 5.1 Overview

This workflow covers both the Web2-only retirement path and the on-chain retirement path, producing a retirement certificate in either case. VCMI claims-code reporting content is flagged as ⚠ PROVISIONAL.

### 5.2 Path A: Web2-Only Retirement (Primary Path)

```mermaid
sequenceDiagram
    autonumber
    participant Buyer as Credit Holder<br/>(Corporate/Retail)
    participant GW as API Gateway
    participant CLS as Credit Ledger<br/>Service
    participant PG as PostgreSQL
    participant RSS as Registry Sync<br/>Service
    participant Reg as Source Registry
    participant KF as Kafka
    participant NRS as Notification &<br/>Reporting Service
    participant Admin as Platform Admin

    Buyer->>GW: POST /credits/retire<br/>(credit_ids, quantity, retirement_reason,<br/>beneficiary_name, beneficiary_entity,<br/>reporting_period, claim_type)
    GW->>CLS: Validate retirement request

    CLS->>PG: Verify credits are ACTIVE/TRANSFERRED,<br/>owned by requester (optimistic lock check)
    
    alt Credits Valid and Owned
        CLS->>PG: BEGIN TRANSACTION
        CLS->>PG: Update CREDIT status: ACTIVE/TRANSFERRED → RETIREMENT_PENDING
        CLS->>PG: Insert CREDIT_EVENT (type=RETIREMENT_INITIATED,<br/>metadata: beneficiary, reason, claim_type)
        CLS->>PG: COMMIT
        CLS->>KF: Publish RetirementInitiated event

        CLS->>RSS: Request registry-side retirement
        RSS->>Reg: Execute retirement via API or manual queue

        alt Registry Retirement Confirmed
            Reg-->>RSS: Retirement confirmed<br/>(registry_retirement_ref, registry_certificate_url)
            RSS->>KF: Publish RegistryRetirementConfirmed event
            KF->>CLS: Consume confirmation

            CLS->>PG: BEGIN TRANSACTION
            CLS->>PG: Update CREDIT status: RETIREMENT_PENDING → RETIRED
            CLS->>PG: Update CREDIT.retired_at = now()
            CLS->>PG: Insert CREDIT_EVENT (type=RETIRED,<br/>registry_tx_ref: registry_retirement_ref)
            CLS->>PG: COMMIT
            CLS->>KF: Publish CreditRetired event

            KF->>NRS: Consume CreditRetired event
            NRS->>NRS: Generate Retirement Certificate (PDF)
            Note over NRS: Certificate contains:<br/>- Serial numbers retired<br/>- Vintage year, project, methodology<br/>- Beneficiary name/entity<br/>- Retirement date<br/>- Registry retirement reference<br/>- Quality labels (CCP, CORSIA, etc.)<br/>- ⚠ PROVISIONAL: VCMI claim data fields
            NRS->>NRS: Store certificate in Object Storage (MinIO/S3)
            NRS-->>Buyer: "Credits retired ✓ — Retirement Certificate attached"

        else Registry Retirement Failed
            Reg-->>RSS: Retirement failed (reason)
            RSS->>KF: Publish RegistryRetirementFailed event
            KF->>CLS: Consume failure
            CLS->>PG: Rollback: RETIREMENT_PENDING → ACTIVE/TRANSFERRED
            CLS->>KF: Publish RetirementRolledBack event
            NRS-->>Buyer: "Retirement could not be completed on [registry].<br/>Credits restored. Reason: [reason]"
            KF->>Admin: Alert: registry retirement failure for manual investigation
        end

    else Credits Not Valid or Not Owned
        CLS-->>GW: 400 Bad Request (reason)
        GW-->>Buyer: "Cannot retire: [credits not owned / wrong status]"
    end
```

### 5.3 Path B: On-Chain Retirement (Tokenized Credits)

> 🔗 **OPTIONAL: On-Chain Retirement Path** — *Only for credits that have been tokenized (Phase 3 §4 Transition 3).*

```mermaid
sequenceDiagram
    autonumber
    participant Buyer as Token Holder<br/>(Smart Account)
    participant SDK as Platform SDK
    participant EP as EntryPoint (4337)
    participant PM as Paymaster
    participant CV as CreditVault<br/>(Polygon)
    participant CT as CarbonCreditToken
    participant BBS as Blockchain<br/>Bridge Service
    participant KF as Kafka
    participant CLS as Credit Ledger<br/>Service
    participant RSS as Registry Sync<br/>Service
    participant Reg as Source Registry
    participant NRS as Notification Service

    Buyer->>SDK: "Retire 10,000 credits (tokenId X)"
    SDK->>SDK: Capture retirement data off-chain<br/>(beneficiary, reason, claim_type)
    SDK->>SDK: Build retirementDataHash =<br/>SHA-256(beneficiary + reason + claim_type)
    SDK->>SDK: Build UserOperation<br/>(calldata = initiateRetirement(tokenId, 10000, retirementDataHash))

    SDK->>EP: Submit via Bundler
    EP->>PM: Gas sponsorship check
    PM-->>EP: Sponsored ✓
    EP->>CV: initiateRetirement(tokenId, 10000, retirementDataHash)
    CV->>CT: burn(holder, tokenId, 10000)
    CT-->>CV: Burned ✓
    CV->>CV: Emit RetirementInitiated event
    Note over CT: On-chain state: PENDING_RETIREMENT

    EP-->>SDK: Transaction receipt
    SDK-->>Buyer: "Retirement initiated — pending registry confirmation"

    BBS->>BBS: Detect RetirementInitiated event on-chain
    BBS->>KF: Publish RetirementRequested event
    KF->>CLS: Lock credits for retirement (RETIREMENT_PENDING)

    CLS->>RSS: Execute registry-side retirement
    RSS->>Reg: Retire credits on source registry

    alt Registry Confirms Retirement
        Reg-->>RSS: Confirmed (registry_retirement_ref)
        RSS->>KF: Publish RegistryRetirementConfirmed
        KF->>CLS: Update credits: RETIRED
        KF->>BBS: Consume confirmation
        BBS->>CV: confirmRetirement(tokenId, 10000, registryRef)
        Note over CT: On-chain state: RETIRED_ONCHAIN

        KF->>NRS: Generate Retirement Certificate
        NRS-->>Buyer: "Retirement complete ✓ — Certificate attached"

    else Registry Retirement Failed
        Reg-->>RSS: Failed (reason)
        RSS->>KF: Publish RegistryRetirementFailed
        Note over BBS: On-chain tokens already burned.<br/>Cannot simply un-burn.
        KF->>BBS: Consume failure
        BBS->>BBS: Mint replacement tokens to holder<br/>(attestation type: RETIREMENT_ROLLBACK)
        BBS->>CV: mintReplacement(tokenId, 10000, holder, rollbackRef)
        KF->>CLS: Rollback: RETIREMENT_PENDING → TOKENIZED
        NRS-->>Buyer: "Registry retirement failed. Tokens restored."
        KF->>KF: P1 Alert to Admin: investigate root cause
    end
```

### 5.4 VCMI Claims-Code Reporting

> ⚠ **PROVISIONAL**: The following VCMI reporting structure is architecturally outlined but the specific field requirements have not been independently verified against the latest VCMI Claims Code of Practice. The workflow captures what data must be captured and when the certificate is generated — not the specific VCMI-mandated field names or validation rules.

**Data captured at retirement time** (regardless of Web2 or on-chain path):

| Data Field | Description | Populated By |
|:---|:---|:---|
| Credit serial numbers | Specific serial numbers being retired | System (from Credit Ledger) |
| Vintage year | Year of the emission reduction/removal | System (from credit metadata) |
| Project reference | Registry project ID + platform project ID | System |
| Methodology | Methodology ID and version | System |
| Quality labels | CCP status, CORSIA eligibility, Article 6 authorization, CCB labels | System |
| Beneficiary name | Entity claiming the environmental benefit | User input |
| Beneficiary entity type | Corporate / individual / government | User input |
| Retirement purpose | Voluntary offset, VCMI claim, CORSIA compliance, resale prohibition | User input |
| ⚠ PROVISIONAL: VCMI claim tier | Silver / Gold / Platinum designation | User input (to be validated against VCMI rules) |
| ⚠ PROVISIONAL: Scope coverage | Which emission scopes (1, 2, 3) this retirement addresses | User input |
| ⚠ PROVISIONAL: Internal reduction evidence | Evidence of concurrent internal emission reductions | User-provided document reference |
| Retirement timestamp | UTC timestamp of retirement execution | System |
| Registry retirement reference | Source registry's retirement certificate/reference number | System (from registry confirmation) |
| On-chain transaction hash | If tokenized: Polygon transaction hash of burn | System (if applicable) |

**Certificate generation timing**: The Notification & Reporting Service generates the retirement certificate **only after** registry-side retirement is confirmed. It is never generated on a pending or unconfirmed retirement.

### 5.5 Failure States & Recovery

| Step | Failure Scenario | Recovery Action |
|:---|:---|:---|
| **Retirement request validation** | Credits not owned by requester | Request rejected with 403. No state change. |
| **Retirement request validation** | Credits in SALE_PENDING state (active trade in progress) | Request rejected: "Credits are locked for a pending trade. Complete or cancel the trade first." |
| **Registry-side retirement** | Registry API timeout (portal-only registries like ACR/CAR) | Retirement enters manual queue. Admin manually executes retirement on registry portal. Credits remain in RETIREMENT_PENDING until confirmed. SLA: 48 hours for manual registries. |
| **Registry-side retirement** | Registry rejects retirement (invalid serial number, already retired) | Rollback: credits restored to ACTIVE/TRANSFERRED. Admin investigates discrepancy — may indicate a reconciliation failure. |
| **On-chain burn + registry failure** | Tokens burned on-chain but registry retirement fails | Replacement tokens minted to holder (Phase 3 §4 Transition 3 failure handling). This is a serious incident — admin investigation required. The platform controls the custody account, so registry retirement should succeed unless the registry itself is down. |
| **Certificate generation** | PDF generation fails | Retirement is still valid (state is RETIRED). Certificate generation retried asynchronously. Buyer can request re-generation from account dashboard. |
| **Buyer disputes retirement** | Buyer claims they did not initiate retirement | Audit trail (Kafka) provides immutable record of the retirement request including auth token, timestamp, and IP. If the account was compromised, the incident response process applies. Retirement cannot be reversed (it's permanent on the registry). |
| **Registry-authoritative override** | Registry shows credit as already retired (before our retirement request) | Credit Ledger Service detects the existing retirement during reconciliation. Credits are marked RETIRED with the registry's retirement date. Buyer is notified that credits were retired externally. If tokenized, emergency revocation fires. |

---

## 6. Workflow 6: Corporate Buyer Payment & Settlement

### 6.1 Overview

> ⚠ **PROVISIONAL**: The actual payment rail, cross-border/forex handling, and any FEMA/RBI implications of the India-based operating entity receiving international payments require dedicated payments-compliance research before implementation. This workflow models the structural flow (order → payment confirmation → credit release) without inventing specific banking or payment-provider integration details.

### 6.2 Structural Payment Workflow

```mermaid
sequenceDiagram
    autonumber
    participant CorpBuyer as Corporate Buyer
    participant GW as API Gateway
    participant MTS as Marketplace &<br/>Trading Service
    participant CLS as Credit Ledger<br/>Service
    participant Pay as Payment Service<br/>⚠ PROVISIONAL
    participant KF as Kafka
    participant NRS as Notification Service
    participant Admin as Platform Admin
    participant Finance as Finance/Ops Team

    CorpBuyer->>GW: POST /marketplace/orders<br/>(credit_ids, quantity, currency: USD/EUR/INR)
    GW->>MTS: Create purchase order

    MTS->>CLS: Lock credits (SALE_PENDING)
    MTS->>MTS: Create ORDER (status: AWAITING_PAYMENT)<br/>Calculate total: quantity × unit_price<br/>⚠ PROVISIONAL: FX conversion if needed

    MTS-->>CorpBuyer: Order summary + payment instructions

    Note over CorpBuyer,Pay: ⚠ PROVISIONAL BLOCK: Payment Execution<br/>The specific payment rail is TBD.<br/>Possible options being evaluated:<br/>1. International wire transfer (SWIFT)<br/>2. Payment gateway (Stripe/Razorpay)<br/>3. Escrow service<br/>4. Letter of Credit (for large institutional deals)

    CorpBuyer->>Pay: Execute payment via selected rail
    
    alt Payment Confirmed
        Pay-->>MTS: Payment confirmation<br/>(amount, currency, tx_reference, settlement_date)
        MTS->>MTS: Validate: payment amount ≥ order total
        
        alt Amount Matches
            MTS->>MTS: ORDER status: PAYMENT_CONFIRMED
            MTS->>CLS: Execute ownership transfer
            CLS->>CLS: Transfer credits: seller → buyer (ACID)
            CLS->>KF: Publish CreditTransferred event
            MTS->>MTS: ORDER status: COMPLETED

            KF->>NRS: Notify buyer: "Purchase complete"
            KF->>NRS: Notify seller: "Credits sold and settled"
            KF->>Finance: Settlement record for reconciliation
        
        else Amount Mismatch
            MTS->>MTS: ORDER status: PAYMENT_DISCREPANCY
            KF->>Admin: Alert: payment amount mismatch
            KF->>Finance: Manual reconciliation required
            NRS-->>CorpBuyer: "Payment received but amount doesn't match. Support will contact you."
        end

    else Payment Timeout (configurable: 24-72h for wire transfers)
        MTS->>MTS: ORDER status: PAYMENT_TIMED_OUT
        MTS->>CLS: Release credit locks
        NRS-->>CorpBuyer: "Payment window expired. Order cancelled."
    
    else Payment Failed / Rejected
        Pay-->>MTS: Payment failed (reason)
        MTS->>MTS: ORDER status: PAYMENT_FAILED
        MTS->>CLS: Release credit locks
        NRS-->>CorpBuyer: "Payment failed: [reason]. Credits released."
    end
```

### 6.3 ⚠ PROVISIONAL: Payment & Settlement Considerations

The following require dedicated payments-compliance research and are **not designed in this phase**:

| Topic | Status | Research Required |
|:---|:---|:---|
| **Payment rails** | ⚠ PROVISIONAL | Evaluate: Stripe (card/ACH), Razorpay (India domestic), SWIFT (international wire), stablecoin settlement (for on-chain path). Each has different settlement latency, cost, and regulatory implications. |
| **Cross-border/forex** | ⚠ PROVISIONAL | India-based operating entity receiving USD/EUR payments. FEMA (Foreign Exchange Management Act) compliance: which transactions qualify as current account vs. capital account? Liberalized Remittance Scheme (LRS) limits for Indian retail buyers purchasing international credits? |
| **RBI implications** | ⚠ PROVISIONAL | If platform holds funds in escrow before credit release: does this require a payment aggregator license under RBI guidelines? PA-PG (Payment Aggregator / Payment Gateway) framework applicability. |
| **Tax treatment** | ⚠ PROVISIONAL | GST on platform commission/fees. Withholding tax on international payments to project developers. Carbon credit sale classification: goods, services, or financial instruments? |
| **Escrow model** | ⚠ PROVISIONAL | For large institutional trades: should the platform operate an escrow account? If so, regulatory requirements for escrow licensure. Alternative: use a regulated escrow partner. |
| **Multi-currency pricing** | ⚠ PROVISIONAL | Credits may be listed in USD. Buyers in INR/EUR need FX conversion. Who bears FX risk: buyer, seller, or platform? At what point is the rate locked? |
| **Refund / chargeback handling** | ⚠ PROVISIONAL | Carbon credit sales are generally non-reversible (credits transfer on the registry). How to handle card chargebacks or payment disputes after settlement. |

### 6.4 Failure States & Recovery

| Step | Failure Scenario | Recovery Action |
|:---|:---|:---|
| **Order creation** | Credits no longer available (sold to another buyer) | Optimistic lock detects version mismatch. Order rejected: "Some credits are no longer available." Buyer shown updated availability. |
| **Payment execution** | Wire transfer delayed beyond timeout window | Admin can extend payment window manually for verified institutional buyers. Standard timeout: 72h for wire, 30 min for card/UPI. |
| **Payment confirmation** | Payment gateway webhook fails (doesn't notify platform) | Platform polls payment gateway for order status every 5 min as fallback. If payment detected, order proceeds. If not detected after timeout, credits released. |
| **Settlement** | Credit transfer succeeds but payment later reversed (chargeback) | ⚠ PROVISIONAL: This is the highest-risk payment failure mode. Credits are already transferred and potentially re-sold or retired. Platform must pursue chargeback dispute. May require pre-authorization holds for card payments. Wire transfers are not reversible (lower risk). |
| **Registry override** | Registry retires/cancels credits between payment confirmation and ownership transfer | Transfer fails. Buyer refunded. Order marked SETTLEMENT_FAILED — REGISTRY_OVERRIDE. |
| **FX rate movement** | Exchange rate changes between order creation and payment confirmation | ⚠ PROVISIONAL: Rate lock mechanism TBD. Options: (a) lock rate for payment window duration, (b) re-price at settlement time, (c) buyer bears FX risk with disclosed rate at checkout. |

---

## 7. Cross-Workflow State Machine Summary

### 7.1 Unified Credit State Machine (Phase 2 + Phase 3)

This diagram consolidates all credit states across both the Web2 and on-chain paths, showing every valid transition and the workflow that triggers it.

```mermaid
stateDiagram-v2
    [*] --> LISTED: Workflow 1: Project listed
    LISTED --> VALIDATED: Workflow 2: VVB validation
    VALIDATED --> VERIFIED: Workflow 2: VVB verification
    VERIFIED --> ISSUED: Workflow 3: Registry issuance detected
    ISSUED --> ACTIVE: Workflow 3: Platform activation
    ISSUED --> BUFFERED: Registry buffer pool deduction

    ACTIVE --> SALE_PENDING: Workflow 4: Order placed (lock)
    SALE_PENDING --> ACTIVE: Workflow 4: Payment failed / timeout (unlock)
    SALE_PENDING --> TRANSFERRED: Workflow 4: Payment confirmed + ownership transfer

    ACTIVE --> TRANSFERRED: Workflow 4: Direct transfer (OTC)
    TRANSFERRED --> TRANSFERRED: Workflow 4: Re-transfer

    ACTIVE --> RETIREMENT_PENDING: Workflow 5: Retirement initiated
    TRANSFERRED --> RETIREMENT_PENDING: Workflow 5: Retirement initiated
    RETIREMENT_PENDING --> RETIRED: Workflow 5: Registry retirement confirmed
    RETIREMENT_PENDING --> ACTIVE: Workflow 5: Registry retirement failed (rollback)
    RETIREMENT_PENDING --> TRANSFERRED: Workflow 5: Rollback to transferred state

    ACTIVE --> ACTIVE_LOCKED: Workflow 3 optional: Tokenization lock
    ACTIVE_LOCKED --> ACTIVE: Tokenization failed (unlock)
    ACTIVE_LOCKED --> TOKENIZED: Workflow 3 optional: Tokens minted

    TOKENIZED --> TOKENIZED: Workflow 4 optional: On-chain transfer
    TOKENIZED --> RETIREMENT_PENDING: Workflow 5 optional: On-chain burn
    TOKENIZED --> ACTIVE: Detokenization (un-bridge)

    ACTIVE --> CANCELLED: Registry-authoritative cancellation
    TRANSFERRED --> CANCELLED: Registry-authoritative cancellation
    TOKENIZED --> CANCELLED: Emergency revocation (Phase 3 §4 T5)
    ACTIVE_LOCKED --> CANCELLED: Emergency revocation

    RETIRED --> [*]
    CANCELLED --> [*]
    BUFFERED --> [*]
```

### 7.2 Project State Machine

```mermaid
stateDiagram-v2
    [*] --> DRAFT: Workflow 1: Project created
    DRAFT --> SUBMITTED: Workflow 1: Developer submits
    SUBMITTED --> LISTED: Workflow 1: Admin approves
    SUBMITTED --> REVISION_REQUESTED: Workflow 1: Admin requests changes
    REVISION_REQUESTED --> SUBMITTED: Developer resubmits
    SUBMITTED --> REJECTED: Workflow 1: Admin rejects

    LISTED --> VALIDATED: Workflow 2: VVB validation complete
    VALIDATED --> VERIFIED: Workflow 2: VVB verification complete

    LISTED --> CANCELLED: Registry delistment
    VALIDATED --> CANCELLED: Registry cancellation
    VERIFIED --> CANCELLED: Registry cancellation

    REJECTED --> [*]
    CANCELLED --> [*]
```

---

## 8. Actor-Permission Matrix

This matrix maps every significant action to the actors authorized to perform it and the Phase 2 service that enforces the permission.

| Action | Project Developer | VVB Auditor | Corporate Buyer | Retail Buyer | Broker | Platform Admin | Enforcement Point |
|:---|:---:|:---:|:---:|:---:|:---:|:---:|:---|
| Register account | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | Identity & Access Service |
| Complete KYC | ✓ | ✓ | ✓ | ✓ | ✓ | — | Identity & Access Service |
| Create project | ✓ | — | — | — | — | ✓ | Project Lifecycle Service |
| Upload PDD / boundary data | ✓ | — | — | — | — | — | Project Lifecycle Service |
| Submit project for review | ✓ | — | — | — | — | — | Project Lifecycle Service |
| Approve/reject project | — | — | — | — | — | ✓ | Project Lifecycle Service |
| Accept VVB assignment | — | ✓ | — | — | — | — | Project Lifecycle Service |
| Submit validation/verification report | — | ✓ | — | — | — | — | Project Lifecycle Service |
| Override VVB decision | — | — | — | — | — | ✓ | Project Lifecycle Service |
| List credits for sale | ✓ (own credits) | — | ✓ (own credits) | ✓ (own credits) | ✓ (client credits with auth) | ✓ | Marketplace & Trading Service |
| Search marketplace | ✓ | ✓ (read-only) | ✓ | ✓ | ✓ | ✓ | Marketplace & Trading Service |
| Purchase credits (marketplace) | ✓ | — | ✓ | ✓ | ✓ (on behalf of clients) | — | Marketplace & Trading Service |
| Execute OTC trade | ✓ | — | ✓ | — | ✓ (primary executor) | ✓ | Marketplace & Trading Service |
| Retire credits | ✓ (own credits) | — | ✓ (own credits) | ✓ (own credits) | ✓ (client credits with auth) | ✓ (admin override) | Credit Ledger Service |
| Request tokenization | ✓ | — | ✓ | ✓ | ✓ | ✓ | Credit Ledger Service + Blockchain Bridge |
| Trigger emergency revocation | — | — | — | — | — | ✓ (+ multisig for on-chain) | Blockchain Bridge Service |
| Manage registry sync | — | — | — | — | — | ✓ | Registry Sync Service |
| Review KYC escalations | — | — | — | — | — | ✓ | Identity & Access Service |
| Generate VCMI claim report | — | — | ✓ | — | — | ✓ | Notification & Reporting Service |

---

## 9. Cross-Cutting Failure Modes

These failure modes can fire **during any workflow** and must be handled by every service.

### 9.1 Registry-Authoritative Conflict Resolution (Phase 2 §4.5)

**Trigger**: Registry Sync Service detects that registry state conflicts with local platform state.

**Resolution**: Registry wins, always. The specific impact depends on the active workflow:

| Active Workflow State | Impact of Registry Override |
|:---|:---|
| Credit in SALE_PENDING (active purchase) | Order cancelled. Buyer refunded. Credits moved to registry state. |
| Credit in RETIREMENT_PENDING | If registry already retired: retirement confirmed (no action needed). If registry cancelled: retirement aborted, credits cancelled. |
| Credit TOKENIZED (on-chain) | Emergency revocation fires (Phase 3 §4 Transition 5). Tokens force-burned. Holder compensated per incident response. |
| Project in LISTED/VALIDATED (active VVB engagement) | VVB engagement voided. Developer and VVB notified. Project status follows registry. |

### 9.2 Emergency Revocation (Phase 3 §4 Transition 5)

**Trigger**: Registry sync detects that a credit backing an on-chain token has been retired, cancelled, or revoked on the source registry without going through the platform's flow.

**Execution**:
1. Registry Sync Service publishes CRITICAL ReconciliationAlert
2. Credit Ledger Service immediately locks all operations on affected credits
3. Blockchain Bridge Service calls `CreditVault.emergencyRevoke()` (requires OPERATOR_ROLE multisig)
4. Tokens force-burned from current holder(s)
5. Holder notified with compensation/dispute process
6. Incident logged in immutable audit trail

**SLA**: Execute within 1 hour of detection. Monitoring alert fires if unresolved >30 min.

---

## 10. Riskiest Workflow: Secondary Trading & Quality Disputes

### 10.1 Identification

**The riskiest workflow is Workflow 4 (Primary Sale and Secondary Trading)**, specifically the scenario where a **buyer disputes a credit's quality label after purchase**.

**Why this is the riskiest**:

1. **Actor disagreement**: The buyer, seller, platform, and VVB all have conflicting incentives. The buyer wants compensation; the seller wants finality; the platform wants to preserve marketplace trust; the VVB's reputation is at stake.
2. **Fraud vector**: A seller could list credits with inflated quality labels (e.g., claiming CCP-Approved status for a non-CCP methodology). The damage occurs after settlement — the buyer may have already retired the credits.
3. **Deadlock potential**: If the buyer refuses to accept the credits and the seller refuses to take them back, the credits are in limbo. For on-chain credits, the tokens cannot be force-transferred without emergency mechanisms.
4. **Cascading impact**: A quality dispute undermines marketplace trust. If a corporate buyer retires disputed credits and later discovers the quality label was wrong, their VCMI claim is compromised — creating legal and reputational exposure far beyond the transaction value.

### 10.2 Dispute Resolution — Design A: Automated Rule-Based Resolution

**Mechanism**:

```
1. Buyer files quality dispute (within 30 days of purchase)
2. System automatically pulls:
   - Credit quality labels at time of sale (from immutable CREDIT_EVENT log)
   - Current registry state (live query via Registry Sync Service)
   - VVB verification report (from Object Storage)
   - ICVCM CCP methodology list (cached, timestamp-verified)
3. Automated rules evaluate:
   a) Was the quality label accurate at time of sale? (Cross-reference serial number 
      against registry data + ICVCM list)
   b) Has the quality label changed since sale? (Methodology revoked by ICVCM?)
   c) Was the credit's status accurate? (Registry confirms credit was ACTIVE?)
4. Automated ruling:
   - If label was accurate at sale: DISPUTE REJECTED — buyer accepted the credit 
     as-is. No refund.
   - If label was inaccurate at sale (e.g., CCP claimed but methodology not in 
     CCP list): DISPUTE UPHELD — automatic reversal. Credits returned to seller, 
     buyer refunded.
   - If label changed after sale (e.g., ICVCM revoked methodology): DISPUTE 
     ESCALATED — no clear fault. Platform absorbs loss or mediates.
5. Resolution executed automatically (refund + credit re-transfer) within 48 hours
```

| Dimension | Assessment |
|:---|:---|
| **Speed** | Fast — automated resolution within 48 hours |
| **Cost** | Low — no human review for clear-cut cases |
| **Trust** | Medium — works well for objective facts (label accuracy) but fails for subjective quality assessments ("the D-MRV data looks unreliable") |
| **Coverage** | Only handles disputes about factual label accuracy. Cannot handle subjective quality disagreements or claims of seller misrepresentation in non-label fields. |
| **Fraud deterrence** | Moderate — sellers know that inaccurate labels will be caught. But sophisticated fraud (manipulated D-MRV data underlying a technically accurate label) bypasses this. |
| **Appeal mechanism** | Limited — automated decisions feel impersonal. Buyers may distrust the platform if they cannot escalate to a human. |

### 10.3 Dispute Resolution — Design B: Human-in-the-Loop Arbitration

**Mechanism**:

```
1. Buyer files quality dispute (within 30 days of purchase)
2. Platform assigns a neutral Dispute Reviewer (internal senior analyst or 
   external domain expert from a rotating panel)
3. Dispute Reviewer has access to:
   - All data in Design A (labels, registry state, VVB reports, ICVCM list)
   - D-MRV raw data summaries + satellite imagery for the project
   - Communication thread between buyer and seller
   - Historical trade data for the same project/credit batch
4. Process:
   a) Both parties submit written statements (5 business days)
   b) Reviewer may request additional evidence from VVB
   c) Reviewer may request independent D-MRV assessment 
      (cost: shared equally by parties, refunded to winner)
   d) Reviewer issues binding decision (15 business days)
5. Possible outcomes:
   - DISPUTE UPHELD: full reversal (credits returned, buyer refunded)
   - DISPUTE PARTIALLY UPHELD: credits stay with buyer, partial price 
     adjustment
   - DISPUTE REJECTED: no action, buyer keeps credits as-is
   - DISPUTE ESCALATED TO EXTERNAL ARBITRATION: for disputes >$100K, 
     referred to an independent arbitration body
6. Decision is published (anonymized) on platform for transparency
```

| Dimension | Assessment |
|:---|:---|
| **Speed** | Slow — 15-20 business days for resolution |
| **Cost** | High — reviewer time ($500–2,000 per dispute), potential independent D-MRV assessment ($2,000–10,000) |
| **Trust** | High — both parties feel heard. Human judgment handles nuance and subjective quality concerns. Published decisions create precedent and trust. |
| **Coverage** | Handles all dispute types: factual label errors, subjective quality concerns, D-MRV reliability challenges, seller misrepresentation. |
| **Fraud deterrence** | Strong — human reviewers can detect patterns (same seller repeatedly selling disputed credits) and recommend account sanctions. |
| **Appeal mechanism** | Built-in: external arbitration for high-value disputes. |

### 10.4 Recommendation: Tiered Hybrid

| Dispute Type | Resolution Mechanism | Rationale |
|:---|:---|:---|
| **Factual label accuracy** (CCP status, CORSIA eligibility, serial number validity) | **Design A — Automated** | Objectively verifiable. Speed is critical. No subjectivity involved. |
| **Subjective quality / D-MRV reliability** | **Design B — Human arbitration** | Requires expert judgment. Automated rules cannot assess satellite imagery quality or D-MRV methodology appropriateness. |
| **Seller misrepresentation / fraud allegations** | **Design B — Human arbitration** with potential account suspension | Requires investigation. Pattern analysis across multiple trades. May involve VVB re-engagement. |
| **Disputes > $100K** | **Design B → External arbitration** | Financial materiality warrants independent third-party review. |

### 10.5 Dispute Failure States

| Failure | Recovery |
|:---|:---|
| Buyer disputes after 30-day window | Dispute rejected (time-barred). Buyer can still report quality concerns to platform for future trade warnings on that project. |
| Seller account deleted / inactive during dispute | Platform holds credits in escrow. Dispute proceeds without seller participation (default judgment in buyer's favor for factual disputes). |
| Credit retired during dispute period | Retirement is irreversible. If dispute is upheld, remedy is financial compensation (not credit reversal). |
| Tokenized credit disputed | On-chain tokens are locked in escrow smart contract during dispute. If upheld, tokens returned via CreditVault mechanism. |
| Reviewer conflict of interest discovered | Reviewer replaced. Dispute timeline extended by 5 days. Previous reviewer's analysis discarded. |

---

## 11. Executive Workflow Summary

**Date**: 18 August 2026
**Purpose**: Fixed context for Phase 5 (API schemas) and implementation teams.

---

### Structural Decisions

**1. Web2-First Design**: All six workflows are fully functional without tokenization. Every user-facing flow executes entirely through Phase 2's service mesh (Identity & Access, Project Lifecycle, Credit Ledger, Marketplace & Trading, Registry Sync, D-MRV Ingestion, Notification & Reporting). Tokenization activates as an optional enhancement layer only when `RegistryConfig.registryTokenizationEnabled == true` for a specific registry — which is currently `false` for all registries.

**2. Two KYC Tracks**: Smallholder farmers and institutional developers follow different KYC paths with different document requirements, connectivity accommodations, and consent mechanisms. Both converge to the same `KYC_VERIFIED` status required for marketplace participation.

**3. VVB Audit is Developer-Initiated**: The platform does not auto-assign VVBs. Developers select from a filtered list of eligible, accredited VVBs. The platform enforces accreditation and rotation rules but does not intermediate the VVB-developer financial relationship.

**4. Credits Activate Only After Registry Confirmation**: The credit issuance → activation path is entirely registry-driven. The platform's Registry Sync Service detects issuances; the Credit Ledger Service creates and activates credits only after registry confirmation. There is no platform-initiated credit creation.

**5. Three Trade Execution Paths**: Web2 marketplace purchase (primary), on-chain ERC-3643 compliant transfer (optional, for tokenized credits), and broker-facilitated OTC (bilateral, off-platform negotiation with on-platform settlement recording).

**6. Retirement is Registry-Gated**: Both the Web2 and on-chain retirement paths require registry-side retirement confirmation before a retirement certificate is generated. No certificate is produced on a pending retirement.

**7. Dispute Resolution is Tiered**: Factual label disputes use automated rule-based resolution (48h). Subjective quality disputes use human arbitration (15 business days). Disputes over $100K can escalate to external arbitration.

### Items Flagged ⚠ PROVISIONAL

| Item | Reason | Action Required Before Phase 5 |
|:---|:---|:---|
| **Payment rail integration** | No payment provider selected. FEMA/RBI implications not researched. | Commission dedicated payments-compliance research. Evaluate Stripe, Razorpay, SWIFT, and stablecoin settlement. Determine PA-PG licensing requirements. |
| **Cross-border forex handling** | FX conversion mechanism, rate locking, and risk bearing not designed. | Include in payments-compliance research. Determine LRS applicability for Indian retail buyers. |
| **VCMI claims-code field requirements** | Specific VCMI-mandated reporting fields not independently verified. | Obtain latest VCMI Claims Code of Practice and map exact field requirements. Update retirement certificate template. |
| **Escrow model for large trades** | Whether platform holds funds in escrow is TBD. | Determine regulatory requirements for escrow services (RBI guidelines). Evaluate third-party escrow partners. |
| **Tax treatment of credit sales** | GST applicability, withholding tax, and classification (goods vs. services vs. financial instruments) not determined. | Commission tax opinion from Indian tax counsel familiar with carbon market instruments. |
| **External arbitration body** | No specific arbitration institution selected for high-value disputes. | Evaluate: Singapore International Arbitration Centre (SIAC), ICC Arbitration, or specialized environmental/commodity arbitration bodies. |
| **All registry custody agreements** | Gold Standard, Puro.earth, and Isometric custody agreements are not yet executed. Verra/ACR/CAR have no published tokenization framework. | Tokenization sub-flows cannot be activated until agreements are executed. Web2 workflows are unaffected. |

### Workflow Interfaces for Phase 5 API Design

| Workflow | Primary API Endpoints Required |
|:---|:---|
| **1. Developer Onboarding** | `POST /auth/register`, `POST /kyc/submit`, `POST /projects`, `POST /projects/{id}/documents`, `POST /projects/{id}/boundary`, `POST /projects/{id}/submit` |
| **2. VVB Audit** | `GET /projects/{id}/eligible-vvbs`, `POST /projects/{id}/vvb-assignment`, `POST /verifications/{id}/accept`, `POST /verifications/{id}/submit-report`, `POST /verifications/{id}/decision` |
| **3. Registry Issuance** | Internal (event-driven via Kafka, no user-facing API). Optional: `POST /credits/{batch_id}/tokenize` |
| **4. Primary Sale & Trading** | `GET /marketplace/search`, `GET /marketplace/credits/{id}`, `POST /marketplace/orders`, `POST /marketplace/orders/{id}/pay`, `POST /otc/trades`, `POST /otc/trades/{id}/confirm` |
| **5. Credit Retirement** | `POST /credits/retire`, `GET /credits/{id}/retirement-certificate` |
| **6. Corporate Payment** | `POST /marketplace/orders` (shared with Workflow 4), `POST /marketplace/orders/{id}/payment-confirmed`, `GET /orders/{id}/settlement-status` |

### Key Phase 2 Service Dependencies per Workflow

| Workflow | Primary Services | Supporting Services |
|:---|:---|:---|
| 1. Onboarding | Identity & Access, Project Lifecycle, D-MRV Ingestion | Notification |
| 2. VVB Audit | Project Lifecycle | Notification, D-MRV Ingestion (read) |
| 3. Issuance | Registry Sync, Credit Ledger | Notification |
| 4. Trading | Marketplace & Trading, Credit Ledger | OpenSearch, Redis, Notification |
| 5. Retirement | Credit Ledger, Registry Sync | Notification (certificate generation) |
| 6. Payment | Marketplace & Trading, Credit Ledger | Notification, ⚠ Payment Service (TBD) |

---

*End of Phase 4 Marketplace Workflows Document.*
