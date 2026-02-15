# BimaSetu (बीमा सेतु) — System Design Document

## 1. System Overview

BimaSetu is a WhatsApp-native AI assistant built on a multi-agent architecture with 7 specialized AI agents coordinated through a central orchestrator. The system provides end-to-end insurance support — from hidden card insurance discovery to real-time emergency hospital matching — entirely within WhatsApp.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER (WhatsApp)                          │
│              Text / Voice Notes / PDF Uploads / Images          │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   WhatsApp Business API                         │
│               (AWS API Gateway + Lambda)                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ORCHESTRATOR AGENT                            │
│              (Amazon Bedrock — Claude Sonnet)                   │
│                                                                  │
│   Intent Detection ─► Agent Routing ─► Context Management       │
│   Emergency Detection ─► Panic Mode Trigger                     │
└──┬──────┬──────┬──────┬──────┬──────┬──────┬────────────────────┘
   │      │      │      │      │      │      │
   ▼      ▼      ▼      ▼      ▼      ▼      ▼
┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐
│Card ││Policy││Triage││Hosp.││Cover.││Claims││BimaSa│
│Disc.││Intel.││Agent ││Match││Reason││Asst. ││lah   │
│Agent││Agent ││      ││Agent││Agent ││Agent ││Agent │
└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘
```

---

## 2. Agent Architecture

### 2.1 Orchestrator Agent

**Responsibility:** Central coordinator that routes user intent to the correct specialized agent. Manages conversation state, context window, and inter-agent communication. Triggers emergency mode on panic detection.

**AWS Stack:** Amazon Bedrock (Claude Sonnet) with agent routing

**Behavior:**
- Receives every incoming message from the WhatsApp integration layer
- Performs intent classification: card discovery, policy query, emergency, claims help, insurance advice, general Q&A
- Routes to the appropriate specialized agent(s) — may invoke multiple agents in sequence for complex flows
- Maintains session state in DynamoDB (conversation history, active mode, user profile reference)
- Detects emergency keywords/patterns (panic words, urgent tone indicators) and immediately escalates to Triage Agent with highest priority
- Manages graceful handoffs between modes (e.g., card discovery gap analysis -> BimaSalah advisory)

**Intent Detection Categories:**
| Intent | Trigger Examples | Routes To |
|--------|-----------------|-----------|
| Card Discovery | "HDFC Regalia", "my card insurance", "card benefits" | Card Discovery Agent |
| Policy Upload | PDF/image attachment, "upload policy", "my policy" | Policy Intelligence Agent |
| Coverage Query | "Is X covered?", "kya covered hai?", "sub-limits?" | Coverage Reasoning Agent |
| Emergency | "accident", "heart attack", "emergency", panicked voice | Triage Agent (P0 priority) |
| Claims Help | "claim rejected", "file claim", "documents needed" | Claims Assistant Agent |
| Insurance Advice | "no insurance", "which plan", "suggest insurance" | BimaSalah Agent |
| General | Greetings, help, settings, feedback | Orchestrator handles directly |

### 2.2 Card Discovery Agent

**Responsibility:** Maps credit/debit card types to their complete insurance benefit packages. Maintains a knowledge base of card-insurance mappings across all major Indian banks. Generates "Hidden Coverage Card" summaries.

**AWS Stack:** Amazon Bedrock Knowledge Bases + DynamoDB for card-benefit mappings

**Data Model:**
```
CardBenefitRecord {
  card_id: String          // e.g., "HDFC_REGALIA_2025"
  bank: String             // e.g., "HDFC Bank"
  card_name: String        // e.g., "HDFC Regalia"
  card_tier: String        // e.g., "Premium"
  annual_fee: Number
  benefits: [
    {
      type: String         // "air_accident", "personal_accident", "travel", etc.
      cover_amount: Number
      description: String
      claim_process: String
      conditions: String
      expiry_rules: String
    }
  ]
  last_verified: Date
  source_url: String
}
```

**Workflow:**
1. User sends card name (text)
2. Agent performs fuzzy matching against card knowledge base
3. If ambiguous (e.g., "HDFC card"), asks clarifying question with card options
4. Retrieves complete benefit package
5. Generates Hidden Coverage Card (visual summary via template rendering)
6. If user adds multiple cards — computes portfolio view with overlaps and combined totals
7. Runs gap analysis: identifies missing coverage categories (health, hospitalization, critical illness)
8. Bridges to BimaSalah if health insurance gap detected

### 2.3 Policy Intelligence Agent

**Responsibility:** Parses uploaded insurance PDFs using multi-modal document understanding. Extracts structured data: coverage limits, exclusions, waiting periods, sub-limits, network hospitals, riders. Creates the "My Coverage Card."

**AWS Stack:** Amazon Textract + Amazon Bedrock (Claude) for reasoning over extracted data

**Parsing Pipeline:**
```
PDF/Image Upload
      │
      ▼
┌─────────────────┐
│ Amazon Textract  │  ──► Raw text + table extraction + form extraction
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Bedrock Claude   │  ──► Semantic extraction into structured schema
│ (Document Parse) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Schema Mapping   │  ──► Map to standardized CoverageProfile schema
│ + Validation     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ DynamoDB Store   │  ──► Persist parsed policy; delete raw PDF
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Coverage Card    │  ──► Generate visual summary for WhatsApp
│ Generator        │
└─────────────────┘
```

**Extracted Policy Schema:**
```
CoverageProfile {
  policy_id: String
  insurer: String
  policy_type: String           // "Individual", "Family Floater", "Group"
  sum_insured: Number
  members_covered: [Member]

  coverage: {
    hospitalization: { limit, sub_limits, room_rent_cap, icu_cap }
    pre_post_hospitalization: { pre_days, post_days }
    daycare_procedures: [String]
    ambulance: { limit }
    organ_donor: { limit }
    domiciliary: { limit, conditions }
  }

  exclusions: {
    permanent: [String]
    waiting_period: [{ condition, period_months, start_date, end_date }]
    specific: [{ exclusion, clause_reference }]
  }

  sub_limits: [{ item, limit_type, limit_value, clause_reference }]
  copay: { percentage, applicable_conditions }

  riders: [{ name, cover, premium }]
  network_hospitals: { count, list_url }

  renewal_date: Date
  portability_eligible: Boolean
  restore_benefit: { available, conditions }

  confidence_scores: { overall, per_section: Map<String, Number> }
}
```

**Confidence Handling:**
- Overall confidence >= 85%: Present extracted data directly
- Section confidence < 85%: Flag that specific section with "We're not 100% sure about this — please verify: [clause text]"
- Overall confidence < 70%: Ask user to re-upload a clearer document

### 2.4 Triage Agent

**Responsibility:** Classifies emergency type from voice/text input in Hindi and regional languages. Extracts location data. Determines required hospital specialty.

**AWS Stack:** Amazon Transcribe (Hindi/regional) + Amazon Bedrock for classification

**Emergency Classification:**
```
EmergencyTriage {
  type: Enum [
    "CARDIAC",        // chest pain, heart attack
    "TRAUMA",         // accident, injury, bleeding
    "STROKE",         // paralysis, speech difficulty, sudden weakness
    "BURNS",          // fire, acid, electrical
    "RESPIRATORY",    // breathing difficulty, asthma attack, choking
    "PEDIATRIC",      // child emergency
    "OBSTETRIC",      // pregnancy complication
    "POISONING",      // ingestion, snake bite
    "OTHER"
  ]
  severity: Enum ["CRITICAL", "URGENT", "MODERATE"]
  location: {
    gps: { lat, lng }           // from WhatsApp location share
    text_description: String     // extracted from voice/text
    resolved_address: String     // geocoded
  }
  required_specialty: [String]   // ["trauma_center", "cath_lab", "neuro_icu"]
  patient_conscious: Boolean
  additional_context: String
}
```

**Voice Processing Pipeline:**
```
Voice Note (Hindi/Regional)
      │
      ▼
┌─────────────────────┐
│ Amazon Transcribe    │  ──► Text transcription (< 3 sec)
│ (Hindi/Regional ASR) │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Bedrock Claude       │  ──► Emergency classification + entity extraction
│ (Triage Classifier)  │      Handles: fragmented speech, background noise,
└────────┬────────────┘       crying, dialect variation, code-mixing
         │
         ▼
┌─────────────────────┐
│ Location Resolution  │  ──► GPS from WhatsApp OR geocode text description
│ (Amazon Location Svc)│
└────────┬────────────┘
         │
         ▼
  EmergencyTriage object ──► Hospital Matching Agent
```

**Key Design Decisions:**
- Maximum ONE follow-up question in emergency mode to minimize delay
- ZERO medical advice — classification only for hospital specialty matching
- Explicit disclaimers rendered in the conversation
- Optimistic classification: if uncertain, assume higher severity

### 2.5 Hospital Matching Agent

**Responsibility:** Maintains and queries hospital database with location, specialties, emergency facilities, and insurance network mappings. Performs multi-constraint matching.

**AWS Stack:** Amazon DynamoDB + Amazon Location Service + custom matching logic

**Hospital Data Model:**
```
Hospital {
  hospital_id: String
  name: String
  address: String
  location: { lat, lng }
  city: String
  state: String

  facilities: {
    emergency: Boolean
    trauma_center: { available, level }      // Level 1, 2, 3
    cath_lab: Boolean
    stroke_unit: Boolean
    neuro_icu: Boolean
    burn_unit: Boolean
    nicu: Boolean                             // Neonatal ICU
    pediatric_emergency: Boolean
    dialysis: Boolean
  }

  insurance_networks: [
    {
      insurer: String           // "Star Health", "HDFC Ergo", etc.
      tpa: String               // Third-party administrator
      cashless: Boolean
      network_tier: String
      empanelment_valid_until: Date
    }
  ]

  contact: { emergency_number, main_number }
  accreditation: String         // NABH, NABL, JCI
  bed_count: Number
  last_updated: Date
}
```

**Matching Algorithm:**
```
Input: EmergencyTriage + UserCoverageProfile

1. FILTER by required specialty
   - Match triage.required_specialty against hospital.facilities
   - Eliminate hospitals without required facility

2. FILTER by proximity
   - Calculate distance from triage.location to each hospital
   - Apply radius: CRITICAL=15km, URGENT=20km, MODERATE=30km

3. CLASSIFY by insurance status
   - For each remaining hospital, check if user's insurer + policy
     is in hospital.insurance_networks with cashless=true
   - Tag each: CASHLESS / REIMBURSEMENT_ONLY / NON_NETWORK

4. RANK results
   - Primary: CASHLESS hospitals first
   - Secondary: Distance (closer = better)
   - Tertiary: Facility match quality (Level 1 trauma > Level 2)
   - Quaternary: Accreditation (NABH preferred)

5. RETURN top 3 hospitals with:
   - Name, distance, cashless status, specialty match, contact
```

### 2.6 Coverage Reasoning Agent

**Responsibility:** Answers real-time coverage questions against the parsed policy. Handles complex queries involving sub-limits, co-pays, waiting period calculations, restoration benefits.

**AWS Stack:** Amazon Bedrock (Claude) with RAG over parsed policy data

**Query Processing:**
```
User Question (any language)
      │
      ▼
┌─────────────────────────┐
│ Question Understanding   │  ──► Extract: procedure/condition, cost context,
│ (Bedrock Claude)         │      time context, family member
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ Policy RAG Retrieval     │  ──► Retrieve relevant sections from parsed
│ (Bedrock Knowledge Base) │      CoverageProfile + original clause text
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ Coverage Reasoning       │  ──► Multi-step reasoning:
│ (Bedrock Claude)         │      1. Is procedure in exclusion list?
│                          │      2. Is waiting period active?
│                          │      3. What sub-limits apply?
│                          │      4. Calculate co-pay
│                          │      5. Check room rent impact
│                          │      6. Check restore benefit
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ Response Generation      │  ──► Plain language answer in user's language
│ with Clause References   │      with specific policy clause citations
└─────────────────────────┘
```

**Reasoning Examples:**
| Query | Reasoning Steps |
|-------|----------------|
| "Is knee replacement covered?" | Check exclusions -> Check waiting period (usually 2-4 years for joint replacement) -> Check sub-limit for joint procedures -> Calculate out-of-pocket |
| "ICU 3 din ka kitna lagega?" | Check ICU daily room rent sub-limit -> Compare with hospital's ICU rate -> Calculate daily difference -> Multiply by days -> Add to total OOP |
| "Papa ne 2L use kar liye, ab kitna bacha?" | Check sum insured -> Subtract utilized amount -> Check if restore benefit exists -> If yes, explain restoration conditions |

### 2.7 Claims Assistant Agent

**Responsibility:** Generates insurer-specific claim checklists, pre-fills forms, tracks claim status, and drafts grievance letters for rejected claims with IRDAI circular citations.

**AWS Stack:** Amazon Bedrock + AWS Lambda for automation workflows

**Claim Documentation Flow:**
```
┌─────────────────────────┐
│ 1. Claim Type Detection  │  ──► Cashless pre-auth / Reimbursement /
│                          │      Follow-up claim / Rejection appeal
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ 2. Insurer-Specific      │  ──► Each of 33+ insurers has different:
│    Checklist Generation  │      - Required documents
│                          │      - Form formats
│                          │      - Submission channels
│                          │      - SLA timelines
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ 3. Form Pre-fill         │  ──► Auto-populate from:
│                          │      - User profile (name, DOB, address)
│                          │      - Policy data (policy #, insurer, TPA)
│                          │      - Hospital data (name, ROHINI code)
│                          │      - Treatment data (from bedside queries)
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ 4. Submission Tracker    │  ──► Track claim status via:
│                          │      - Periodic user check-ins
│                          │      - Insurer portal scraping (where possible)
│                          │      - SLA countdown alerts
└────────┬────────────────┘
         │
         ▼  (if rejected)
┌─────────────────────────┐
│ 5. Rejection Analyzer    │  ──► Parse rejection letter
│                          │      Match reason to IRDAI guidelines
│                          │      Identify if contestable
│                          │      Draft grievance with circular citations
└─────────────────────────┘
```

**Rejection Analysis Logic:**
```
RejectionAnalysis {
  rejection_reason: String
  insurer_cited_clause: String

  analysis: {
    irdai_regulation_check: String     // Relevant IRDAI circular/guideline
    policy_clause_check: String        // What user's policy actually says
    is_contestable: Boolean
    contestability_confidence: Number  // 0-100
    grounds_for_appeal: [String]
  }

  grievance_letter: {
    addressed_to: String
    subject: String
    body: String                       // Formal letter with IRDAI references
    attachments_needed: [String]
    escalation_path: [
      "Insurer Grievance Cell (30 days)",
      "IRDAI IGMS Portal",
      "Insurance Ombudsman"
    ]
  }
}
```

---

## 3. AWS Service Architecture

### 3.1 Service Map

```
┌──────────────────────────────────────────────────────────────────────┐
│                          AWS CLOUD                                    │
│                                                                       │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────────────┐     │
│  │ API Gateway  │───►│ Lambda       │───►│ Amazon Bedrock       │     │
│  │ (WhatsApp    │    │ (Orchestrator│    │ ┌──────────────────┐ │     │
│  │  Webhook)    │    │  + Agents)   │    │ │ Claude Sonnet 4  │ │     │
│  └─────────────┘    └──────┬───────┘    │ │ (Reasoning)      │ │     │
│                            │            │ └──────────────────┘ │     │
│                            │            │ ┌──────────────────┐ │     │
│  ┌─────────────┐          │            │ │ Knowledge Bases  │ │     │
│  │ Amazon      │◄─────────┤            │ │ (RAG: IRDAI regs,│ │     │
│  │ Transcribe  │          │            │ │  card benefits,   │ │     │
│  │ (Voice→Text)│          │            │ │  hospital data)   │ │     │
│  └─────────────┘          │            │ └──────────────────┘ │     │
│                            │            │ ┌──────────────────┐ │     │
│  ┌─────────────┐          │            │ │ Agents for       │ │     │
│  │ Amazon      │◄─────────┤            │ │ Bedrock          │ │     │
│  │ Textract    │          │            │ │ (Multi-agent     │ │     │
│  │ (PDF Parse) │          │            │ │  orchestration)  │ │     │
│  └─────────────┘          │            │ └──────────────────┘ │     │
│                            │            └──────────────────────┘     │
│  ┌─────────────┐          │                                          │
│  │ DynamoDB    │◄─────────┤   ┌──────────────┐                      │
│  │ (All Data)  │          ├──►│ Amazon       │                      │
│  └─────────────┘          │   │ Location Svc │                      │
│                            │   └──────────────┘                      │
│  ┌─────────────┐          │                                          │
│  │ S3          │◄─────────┤   ┌──────────────┐                      │
│  │ (Temp PDF   │          ├──►│ SNS + SES    │                      │
│  │  storage)   │          │   │ (Alerts)     │                      │
│  └─────────────┘          │   └──────────────┘                      │
│                            │                                          │
│  ┌─────────────┐          │   ┌──────────────┐                      │
│  │ AWS KMS     │          ├──►│ CloudWatch   │                      │
│  │ (Encryption)│          │   │ + X-Ray      │                      │
│  └─────────────┘          │   │ (Monitoring) │                      │
│                            │   └──────────────┘                      │
│  ┌─────────────┐          │                                          │
│  │ OpenSearch   │◄─────────┘                                         │
│  │ Serverless   │                                                    │
│  │ (Vector DB)  │                                                    │
│  └──────────────┘                                                    │
└──────────────────────────────────────────────────────────────────────┘
```

### 3.2 Service Justification

| Function | AWS Service | Why This Service |
|----------|------------|-----------------|
| WhatsApp Integration | WhatsApp Business API via API Gateway + Lambda | Handles 500M+ potential users on their preferred messaging platform. Serverless = auto-scaling. |
| Voice Processing | Amazon Transcribe (Hindi, Tamil, Telugu, Bengali, Marathi, English) | Converts emergency voice notes to actionable text in < 3 seconds. Supports Indian languages natively. |
| Document Intelligence | Amazon Textract + Amazon Bedrock (multi-modal) | Parses insurance PDFs and photographed policy documents with table/form extraction. Bedrock adds semantic understanding. |
| AI Reasoning Engine | Amazon Bedrock (Claude Sonnet 4) with Agents for Bedrock | Multi-agent orchestration, policy reasoning, coverage analysis, and natural language generation. |
| Knowledge Base | Amazon Bedrock Knowledge Bases with OpenSearch Serverless | RAG over: IRDAI regulations, insurer networks, hospitals, drug formularies, card-insurance mappings. |
| Primary Database | Amazon DynamoDB | User profiles, parsed policies, card benefits, medical history (encrypted), conversation state. Single-digit ms reads. |
| Temporary Storage | Amazon S3 | Temporary PDF storage during parsing. Auto-delete after extraction. |
| Location Services | Amazon Location Service | Real-time proximity matching for emergency hospital selection. Geocoding for voice-described locations. |
| Ambulance Integration | AWS Lambda + API Gateway | Calling 108/private ambulance APIs. Automated dispatch with patient pre-brief. |
| Hospital Pre-Alert | Amazon SNS + SES + WhatsApp Business API | Multi-channel hospital notification with patient insurance details. |
| Security | AWS KMS + IAM + VPC + CloudTrail | DPDP Act 2023 compliant. All health data encrypted (AES-256 at rest, TLS 1.3 in transit). |
| Monitoring | Amazon CloudWatch + X-Ray | Latency tracking — critical for emergency mode where every second counts. Distributed tracing across agents. |

---

## 4. Data Architecture

### 4.1 DynamoDB Table Design

**Users Table**
```
PK: USER#{phone_hash}
SK: PROFILE

Attributes:
  name, age, city, family_members[], preferred_language,
  cards[], policies[], abha_id,
  created_at, updated_at, consent_timestamp
```

**Policies Table**
```
PK: USER#{phone_hash}
SK: POLICY#{policy_id}

Attributes:
  insurer, policy_type, sum_insured,
  coverage_profile (full CoverageProfile JSON),
  parsed_at, confidence_score,
  renewal_date,
  GSI: insurer-index (for aggregate analytics)
```

**Cards Table**
```
PK: CARD#{card_id}
SK: BENEFITS

Attributes:
  bank, card_name, card_tier,
  benefits[], total_cover_value,
  last_verified, source_url
```

**Hospitals Table**
```
PK: HOSPITAL#{hospital_id}
SK: INFO

Attributes:
  name, address, location{lat,lng},
  facilities{}, insurance_networks[],
  contact{}, accreditation, bed_count,
  last_updated

GSI: city-specialty-index (for filtered queries)
GSI: insurer-network-index (for cashless lookups)
```

**Conversations Table**
```
PK: USER#{phone_hash}
SK: CONV#{timestamp}

Attributes:
  mode, messages[], agent_context,
  active_emergency (if any),
  ttl (90-day auto-expiry)
```

**Medical History Table (Family Health Vault)**
```
PK: USER#{phone_hash}
SK: MEMBER#{member_id}#MEDICAL

Attributes:
  member_name, relation, dob,
  blood_group, allergies[],
  conditions[], medications[],
  prescriptions[], lab_reports[],
  abha_linked: Boolean

(All attributes encrypted with KMS)
```

### 4.2 Knowledge Base Structure (OpenSearch Serverless)

**Index: irdai-regulations**
- All IRDAI circulars, master guidelines, consumer protection regulations
- Chunked and embedded for RAG retrieval
- Used by: Claims Assistant Agent (rejection analysis), Coverage Reasoning Agent

**Index: card-insurance-benefits**
- Detailed benefit descriptions from bank websites and card T&C documents
- Structured per card variant
- Used by: Card Discovery Agent

**Index: hospital-network-lists**
- Insurer-published network hospital lists (IRDAI mandated public data)
- Cross-referenced with hospital geo-data
- Used by: Hospital Matching Agent

**Index: medical-cost-benchmarks**
- CGHS rate lists, NHA package rates, city-wise hospitalization cost data
- Used by: BimaSalah Agent (risk assessment), Coverage Reasoning Agent

---

## 5. Key Workflow Designs

### 5.1 Emergency Mode (BimaRakshak) — End-to-End Flow

```
TIME: 0s ─────────────────────────────────────────────────► 90s

[0-3s]   User sends voice note in Hindi
              │
[3-6s]   Amazon Transcribe ──► Text transcription
              │
[6-12s]  Triage Agent ──► Emergency classification
              │              + Location extraction
              │              + Specialty requirement
              │
[12-15s] (Optional) ONE follow-up question ──► User responds
              │
[15-30s] Hospital Matching Agent
              │  ├─ Query hospitals by specialty (DynamoDB)
              │  ├─ Filter by proximity (Location Service)
              │  ├─ Cross-ref insurance network (DynamoDB)
              │  └─ Rank and return top 3
              │
[30-35s] Present options to user via WhatsApp
              │
[35-45s] User confirms hospital selection
              │
[45-60s] PARALLEL EXECUTION:
              ├─ Ambulance dispatch (Lambda ──► 108 API)
              ├─ Hospital pre-alert (SNS ──► Hospital WhatsApp/SMS)
              │     └─ Contains: insurance details, policy #, TPA,
              │        medical history, emergency type, ETA
              └─ Pre-authorization initiation
                    └─ TPA notification with policy details
              │
[60-90s] Confirmation to user:
              ├─ "Ambulance dispatched. ETA: 8 minutes"
              ├─ "Max Hospital notified. Bed being prepared."
              └─ "Your Star Health policy is active. Cashless confirmed."
```

### 5.2 Policy Upload & Parse Flow

```
User uploads PDF via WhatsApp
        │
        ▼
Lambda receives file ──► Store in S3 (temporary, encrypted)
        │
        ▼
Amazon Textract ──► Extract raw text, tables, forms
        │
        ▼
Bedrock Claude ──► Semantic extraction:
        │           - Identify insurer and policy type
        │           - Extract coverage limits per category
        │           - Parse exclusion lists
        │           - Calculate waiting period dates
        │           - Identify sub-limits and co-pays
        │           - Extract network hospital info
        │           - Identify riders and add-ons
        │
        ▼
Schema Validation ──► Map to CoverageProfile schema
        │               Compute confidence scores per section
        │
        ▼
DynamoDB ──► Store structured CoverageProfile
        │
S3 ──► DELETE raw PDF
        │
        ▼
Generate Coverage Card ──► Visual summary image
        │
        ▼
Send to user via WhatsApp:
  "Here's your Coverage Card for [Insurer] [Plan Name]"
  [Coverage Card Image]
  "Sum Insured: ₹5,00,000 | Room Rent: ₹5,000/day cap"
  "Ask me anything about your policy!"
```

### 5.3 Card Discovery Flow

```
User: "HDFC Regalia"
        │
        ▼
Card Discovery Agent
        │
        ├─ Fuzzy match "HDFC Regalia" against card knowledge base
        │
        ├─ Retrieve: HDFC_REGALIA_2025 record
        │     ├─ Air Accident Cover: ₹1,00,00,000
        │     ├─ Personal Accident: ₹50,00,000
        │     ├─ Travel Insurance (International)
        │     │     ├─ Flight Delay (>3hrs): ₹10,000
        │     │     ├─ Baggage Loss: ₹60,000
        │     │     ├─ Emergency Medical: $25,000
        │     │     └─ Trip Cancellation: ₹1,00,000
        │     ├─ Purchase Protection: 90 days
        │     ├─ Lost Card Liability: ₹5,00,000
        │     └─ Total Hidden Value: ₹1,56,70,000
        │
        ├─ Generate Hidden Coverage Card (visual template)
        │
        └─ Send to user:
            "Your HDFC Regalia has ₹1.57 Crore in insurance
             you probably didn't know about!"
            [Hidden Coverage Card Image]
            "Want to add more cards to see your total portfolio?"
```

### 5.4 Claim Rejection Fighter Flow

```
User: "My claim was rejected" / uploads rejection letter
        │
        ▼
Claims Assistant Agent
        │
        ├─ Parse rejection letter (Textract + Bedrock)
        │     ├─ Rejection reason
        │     ├─ Insurer cited clause
        │     └─ Claim amount and details
        │
        ├─ Cross-reference with:
        │     ├─ User's parsed policy (CoverageProfile)
        │     ├─ IRDAI regulations knowledge base
        │     └─ Historical rejection patterns
        │
        ├─ Analysis:
        │     ├─ "Rejection reason: Pre-existing condition (Diabetes)"
        │     ├─ "Your policy waiting period for PED: 3 years"
        │     ├─ "Your policy start date: Jan 2022"
        │     ├─ "Current date: Feb 2026 → 4+ years elapsed"
        │     ├─ "IRDAI Circular IRDAI/HLT/REG/CIR/XXX states..."
        │     └─ "VERDICT: CONTESTABLE — waiting period has expired"
        │
        ├─ Generate grievance letter with:
        │     ├─ Formal structure addressed to insurer GRO
        │     ├─ Policy and claim reference numbers
        │     ├─ Specific IRDAI circular citations
        │     ├─ Waiting period calculation proof
        │     └─ Request for reconsideration
        │
        └─ Guide user through escalation:
            Step 1: Submit to insurer grievance cell (30-day window)
            Step 2: If unresolved → IRDAI IGMS portal
            Step 3: If still unresolved → Insurance Ombudsman
```

---

## 6. Security Architecture

### 6.1 Data Protection

```
┌─────────────────────────────────────────────────┐
│                DATA FLOW SECURITY                │
│                                                   │
│  User ──TLS 1.3──► API Gateway ──IAM──► Lambda  │
│                                           │       │
│                                    ┌──────┴─────┐│
│                                    │ AWS KMS    ││
│                                    │ AES-256    ││
│                                    └──────┬─────┘│
│                                           │       │
│                          ┌────────────────┤       │
│                          │                │       │
│                    ┌─────┴─────┐   ┌──────┴─────┐│
│                    │ DynamoDB  │   │ S3          ││
│                    │ (Encrypted│   │ (Encrypted  ││
│                    │  at rest) │   │  temp only) ││
│                    └───────────┘   └─────────────┘│
└─────────────────────────────────────────────────┘
```

### 6.2 Privacy Controls (DPDP Act 2023 Compliance)

| Principle | Implementation |
|-----------|---------------|
| **Explicit Consent** | User must explicitly opt-in before any data is stored. Consent recorded with timestamp. |
| **Purpose Limitation** | Data used ONLY for insurance assistance. Never shared with third parties without consent. |
| **Data Minimization** | Only card NAME stored (never card numbers). Raw PDFs deleted after parsing. Location data not persisted. |
| **Right to Erasure** | User can send "DELETE MY DATA" at any time. Complete purge within 24 hours. |
| **Data Encryption** | All PII and health data encrypted with KMS (AES-256). Phone numbers stored as hashes. |
| **Access Control** | IAM roles with least-privilege. No human access to user data without audit trail. |
| **Audit Trail** | CloudTrail logs all data access. Retained for compliance review. |

### 6.3 Medical Liability Protection

- System provides ZERO medical advice
- Triage Agent classifies emergency TYPE only — for hospital specialty matching
- All medical decisions remain with doctors
- Explicit disclaimers rendered in every emergency conversation:
  > "BimaSetu helps you find the right hospital and use your insurance. It does not provide medical advice. Always follow your doctor's guidance."

---

## 7. Monitoring & Observability

### 7.1 Critical Metrics

| Metric | Target | Alert Threshold |
|--------|--------|-----------------|
| Emergency mode end-to-end latency | < 90s | > 120s |
| Voice transcription latency | < 3s | > 5s |
| Hospital match query time | < 2s | > 4s |
| Policy parse time | < 30s | > 60s |
| Coverage Q&A response time | < 5s | > 10s |
| Agent error rate | < 1% | > 2% |
| WhatsApp message delivery rate | > 99% | < 97% |

### 7.2 Distributed Tracing (X-Ray)

Every request is traced end-to-end through:
```
WhatsApp → API Gateway → Lambda (Orchestrator) → Agent(s) →
  Bedrock/Textract/Transcribe → DynamoDB → Response → WhatsApp
```

Emergency mode traces are tagged with high priority for real-time dashboard visibility.

### 7.3 Operational Dashboards (CloudWatch)

- **Emergency Dashboard:** Real-time view of active emergencies, response times, hospital match success rate
- **Agent Health Dashboard:** Per-agent invocation count, latency, error rate, token usage
- **User Activity Dashboard:** Daily active users, mode distribution, policy uploads, card discoveries
- **Cost Dashboard:** Per-service AWS spend, cost per user interaction, cost per emergency response

---

## 8. Scalability Design

### 8.1 Serverless Auto-Scaling

| Component | Scaling Strategy |
|-----------|-----------------|
| API Gateway | Auto-scales to any request volume |
| Lambda Functions | Concurrent executions scale automatically. Reserved concurrency for emergency functions. |
| DynamoDB | On-demand capacity mode. Auto-scales reads/writes. |
| Bedrock | Managed service with provisioned throughput for emergency mode |
| Textract | Async processing for batch policy uploads |
| Transcribe | Streaming mode for real-time voice processing |

### 8.2 Pre-Computation Strategy

To meet the 90-second emergency SLA, the following data is pre-computed and cached:

- **Hospital-insurer network mappings:** Updated daily via batch job. Stored in DynamoDB with GSI for fast lookup.
- **Hospital geo-indexes:** Pre-computed geohash indexes for proximity queries.
- **Card benefit summaries:** Pre-rendered Hidden Coverage Card templates for top 200 cards.
- **IRDAI regulation embeddings:** Pre-embedded and indexed in OpenSearch for instant RAG retrieval.

### 8.3 Cold Start Mitigation

- Lambda functions for emergency mode use **Provisioned Concurrency** to eliminate cold starts
- Non-emergency functions use standard on-demand scaling
- Bedrock model endpoints are kept warm with periodic health checks

---

## 9. Integration Points

### 9.1 External APIs

| Integration | Purpose | Status |
|-------------|---------|--------|
| WhatsApp Business API | Primary user interface | Core — Required |
| 108 Ambulance API | Emergency ambulance dispatch | Phase 2 (simulated in MVP) |
| Private ambulance aggregators | Alternative ambulance dispatch | Phase 2 |
| Google Maps / Amazon Location | Geocoding and proximity calculations | Core — Required |
| PM-JAY eligibility API | Ayushman Bharat check | Phase 1 |
| ABHA APIs | Health record sync | Phase 4 |
| Insurer portals (where available) | Claim status tracking | Phase 3 |

### 9.2 Future Integrations

| Integration | Purpose | Phase |
|-------------|---------|-------|
| Bima Sugam API | Insurance marketplace connection | Phase 4 |
| WhatsApp Pay | Premium payments, claim disbursements | Phase 4 |
| Hospital information systems | Real-time bed availability | Phase 3 |
| Insurer claim submission APIs | Direct digital claim filing | Phase 3 |

---

## 10. Deployment Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Production Environment                  │
│                                                            │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────┐    │
│  │  VPC     │    │ Public   │    │ Private Subnets   │    │
│  │          │    │ Subnets  │    │                    │    │
│  │          │    │ ┌──────┐ │    │ ┌──────────────┐  │    │
│  │          │    │ │API GW│ │    │ │ Lambda Funcs │  │    │
│  │          │    │ └──────┘ │    │ │ (All Agents) │  │    │
│  │          │    │          │    │ └──────────────┘  │    │
│  │          │    │          │    │ ┌──────────────┐  │    │
│  │          │    │          │    │ │ DynamoDB     │  │    │
│  │          │    │          │    │ │ (VPC Endpt)  │  │    │
│  │          │    │          │    │ └──────────────┘  │    │
│  │          │    │          │    │ ┌──────────────┐  │    │
│  │          │    │          │    │ │ OpenSearch   │  │    │
│  │          │    │          │    │ │ Serverless   │  │    │
│  │          │    │          │    │ └──────────────┘  │    │
│  └──────────┘    └──────────┘    └──────────────────┘    │
│                                                            │
│  CloudTrail ──► Audit Logs                                │
│  CloudWatch ──► Metrics + Alarms                          │
│  X-Ray ──► Distributed Tracing                            │
│  KMS ──► All encryption keys                              │
└──────────────────────────────────────────────────────────┘
```

### Environments:
- **Dev:** Sandbox WhatsApp API, reduced Bedrock throughput, test data
- **Staging:** Full service integration, synthetic test users, load testing
- **Production:** Live WhatsApp API, provisioned Bedrock throughput, real hospital data, KMS encryption, full monitoring