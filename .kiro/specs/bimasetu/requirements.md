# Requirements Document: BimaSetu

## Introduction

BimaSetu is a WhatsApp-based AI assistant that transforms how Indians interact with insurance across the entire insurance lifecycle. The system addresses critical information asymmetry in the Indian insurance market by providing four specialized modes: Hidden Insurance Discovery, BimaSalah (Insurance Counsel), BimaGuru (Policy Decoder), and BimaRakshak (Emergency Guardian). The platform operates entirely within WhatsApp, leveraging AI agents to provide personalized insurance guidance, emergency response, and policy management in vernacular languages.

## Glossary

- **BimaSetu_System**: The complete WhatsApp-based AI assistant platform
- **Orchestrator_Agent**: AI agent that routes user requests to specialized agents
- **Policy_Intelligence_Agent**: AI agent that parses and analyzes insurance policy documents
- **Card_Discovery_Agent**: AI agent that identifies hidden insurance benefits in credit/debit cards
- **Triage_Agent**: AI agent that performs medical emergency assessment
- **Hospital_Matching_Agent**: AI agent that matches patients to appropriate hospitals
- **Coverage_Reasoning_Agent**: AI agent that determines insurance coverage applicability
- **Claims_Assistant_Agent**: AI agent that helps users navigate claim processes
- **User**: Individual interacting with BimaSetu through WhatsApp
- **Policy_Document**: Insurance policy PDF uploaded by user
- **Coverage_Summary**: Plain-language explanation of insurance policy coverage
- **Card_Insurance**: Hidden insurance benefits embedded in credit/debit cards
- **Cashless_Hospital**: Hospital that accepts direct insurance claim settlement
- **Emergency_Mode**: BimaRakshak's 90-second agentic emergency response system
- **Family_Vault**: Secure storage for family members' health and insurance information
- **Vernacular_Language**: Regional Indian languages including Hindi, Tamil, Telugu, Bengali, etc.
- **IRDAI**: Insurance Regulatory and Development Authority of India
- **Claim_Rejection**: Insurance company's denial of a claim request
- **Waiting_Period**: Time period after policy purchase before certain benefits become active
- **Coverage_Gap**: Situation or condition not covered by current insurance policy

## Requirements

### Requirement 1: WhatsApp Integration

**User Story:** As a user, I want to interact with BimaSetu through WhatsApp, so that I can access insurance assistance without installing additional apps.

#### Acceptance Criteria

1. WHEN a user sends a message to the BimaSetu WhatsApp number, THE BimaSetu_System SHALL receive and process the message within 3 seconds
2. WHEN a user sends a voice message in Hindi or regional languages, THE BimaSetu_System SHALL transcribe the audio to text
3. WHEN the system generates a response, THE BimaSetu_System SHALL send the response through WhatsApp within 5 seconds
4. WHEN a user uploads a PDF document, THE BimaSetu_System SHALL accept PDF files up to 25MB in size
5. THE BimaSetu_System SHALL maintain conversation context across multiple message exchanges within a session

### Requirement 2: Hidden Insurance Discovery Mode

**User Story:** As a credit or debit card holder, I want to discover insurance benefits embedded in my cards, so that I can utilize coverage I didn't know existed.

#### Acceptance Criteria

1. WHEN a user provides card details, THE Card_Discovery_Agent SHALL identify the card issuer and card type
2. WHEN card information is validated, THE Card_Discovery_Agent SHALL retrieve all insurance benefits associated with that card
3. WHEN insurance benefits are found, THE BimaSetu_System SHALL present coverage details including sum insured, coverage type, and claim process
4. WHEN no insurance benefits are found, THE BimaSetu_System SHALL inform the user and suggest alternative insurance options
5. THE BimaSetu_System SHALL store discovered card insurance information in the user's profile for future reference

### Requirement 3: BimaSalah Insurance Counsel Mode

**User Story:** As an uninsured individual, I want personalized insurance recommendations based on my profile, so that I can make informed decisions about purchasing insurance.

#### Acceptance Criteria

1. WHEN a user requests insurance guidance, THE Orchestrator_Agent SHALL collect user profile information including age, family size, income, and health conditions
2. WHEN user profile is complete, THE BimaSetu_System SHALL perform risk assessment based on provided information
3. WHEN risk assessment is complete, THE BimaSetu_System SHALL generate personalized insurance recommendations with at least 3 plan options
4. WHEN displaying plan options, THE BimaSetu_System SHALL present comparison including premium, coverage amount, key benefits, and exclusions
5. WHEN a user asks questions in vernacular languages, THE BimaSetu_System SHALL respond in the same language
6. THE BimaSetu_System SHALL explain insurance terminology in simple, plain language

### Requirement 4: BimaGuru Policy Decoder Mode

**User Story:** As a policy holder, I want to understand my insurance policy in plain language, so that I know exactly what is covered and what is not.

#### Acceptance Criteria

1. WHEN a user uploads a policy PDF, THE Policy_Intelligence_Agent SHALL extract text content from the document within 10 seconds
2. WHEN policy text is extracted, THE Policy_Intelligence_Agent SHALL parse the document structure including sections, clauses, and coverage details
3. WHEN parsing is complete, THE BimaSetu_System SHALL generate a Coverage_Summary in plain language
4. WHEN displaying the Coverage_Summary, THE BimaSetu_System SHALL include covered conditions, exclusions, sum insured, waiting periods, and claim process
5. WHEN a user asks questions about their policy, THE Policy_Intelligence_Agent SHALL provide answers with specific references to policy clauses
6. THE BimaSetu_System SHALL identify and highlight coverage gaps in the user's current policy
7. THE BimaSetu_System SHALL store parsed policy information for future queries

### Requirement 5: Policy Question Answering

**User Story:** As a policy holder, I want to ask specific questions about my policy, so that I can get immediate clarification without reading the entire document.

#### Acceptance Criteria

1. WHEN a user asks a policy-related question, THE Policy_Intelligence_Agent SHALL retrieve relevant policy sections
2. WHEN relevant sections are found, THE Policy_Intelligence_Agent SHALL generate an answer based on policy content
3. WHEN providing answers, THE BimaSetu_System SHALL cite specific policy clause numbers and page references
4. WHEN policy content is ambiguous, THE BimaSetu_System SHALL present multiple interpretations and recommend contacting the insurer
5. WHEN a question cannot be answered from the policy, THE BimaSetu_System SHALL inform the user and suggest alternative information sources

### Requirement 6: Proactive Policy Alerts

**User Story:** As a policy holder, I want to receive timely alerts about my policy, so that I don't miss important dates or opportunities to claim.

#### Acceptance Criteria

1. WHEN a policy renewal date is within 30 days, THE BimaSetu_System SHALL send a renewal reminder to the user
2. WHEN a waiting period is about to end, THE BimaSetu_System SHALL notify the user that coverage is becoming active
3. WHEN the system detects a claimable event based on user messages, THE BimaSetu_System SHALL proactively suggest filing a claim
4. WHEN premium payment is due within 7 days, THE BimaSetu_System SHALL send a payment reminder
5. THE BimaSetu_System SHALL allow users to configure alert preferences and frequency

### Requirement 7: BimaRakshak Emergency Guardian Mode

**User Story:** As someone experiencing a medical emergency, I want immediate guidance on hospital selection and coverage, so that I can get appropriate care without worrying about payment.

#### Acceptance Criteria

1. WHEN a user indicates a medical emergency, THE Orchestrator_Agent SHALL activate Emergency_Mode within 2 seconds
2. WHEN Emergency_Mode is activated, THE Triage_Agent SHALL perform voice-based medical triage in Hindi or the user's preferred language
3. WHEN triage is complete, THE Triage_Agent SHALL assess emergency severity and required medical specialty
4. WHEN emergency assessment is complete, THE Hospital_Matching_Agent SHALL identify suitable hospitals within 10 seconds
5. WHEN matching hospitals, THE Hospital_Matching_Agent SHALL consider proximity, cashless acceptance, required specialty, and bed availability
6. WHEN hospitals are identified, THE BimaSetu_System SHALL present top 3 hospital options with distance, estimated time, and cashless status
7. WHEN a hospital is selected, THE BimaSetu_System SHALL provide ambulance dispatch options
8. WHEN ambulance is dispatched, THE BimaSetu_System SHALL send pre-alert to the selected hospital with patient information and insurance details
9. THE Emergency_Mode SHALL complete the entire workflow from triage to hospital pre-alert within 90 seconds

### Requirement 8: Hospital Matching and Selection

**User Story:** As a patient needing hospitalization, I want to find hospitals that accept my insurance cashlessly and have the required specialty, so that I can receive treatment without upfront payment.

#### Acceptance Criteria

1. WHEN matching hospitals, THE Hospital_Matching_Agent SHALL retrieve the user's active insurance policies
2. WHEN policies are retrieved, THE Hospital_Matching_Agent SHALL identify all Cashless_Hospitals in the network
3. WHEN filtering hospitals, THE Hospital_Matching_Agent SHALL match required medical specialty with hospital capabilities
4. WHEN calculating proximity, THE Hospital_Matching_Agent SHALL use the user's current location or specified location
5. WHEN presenting hospital options, THE BimaSetu_System SHALL display hospital name, distance, specialty availability, cashless status, and estimated wait time
6. THE Hospital_Matching_Agent SHALL prioritize hospitals based on a weighted score of proximity, specialty match, and cashless acceptance

### Requirement 9: Coverage Reasoning and Verification

**User Story:** As a patient about to undergo treatment, I want to know if my insurance covers the procedure, so that I can make informed financial decisions.

#### Acceptance Criteria

1. WHEN a user describes a medical procedure or condition, THE Coverage_Reasoning_Agent SHALL identify the relevant insurance policy
2. WHEN the policy is identified, THE Coverage_Reasoning_Agent SHALL determine if the procedure is covered
3. WHEN coverage is confirmed, THE BimaSetu_System SHALL provide coverage amount, co-payment requirements, and claim process
4. WHEN coverage is denied, THE BimaSetu_System SHALL explain the reason with policy clause references
5. WHEN waiting periods apply, THE BimaSetu_System SHALL inform the user of the waiting period end date
6. THE Coverage_Reasoning_Agent SHALL check for coverage exclusions and pre-authorization requirements

### Requirement 10: Claims Assistance

**User Story:** As a policy holder filing a claim, I want guidance through the claims process, so that I can maximize my chances of claim approval.

#### Acceptance Criteria

1. WHEN a user initiates a claim, THE Claims_Assistant_Agent SHALL collect required claim information including treatment details, bills, and policy number
2. WHEN claim information is collected, THE Claims_Assistant_Agent SHALL verify that all required documents are present
3. WHEN documents are incomplete, THE BimaSetu_System SHALL specify which documents are missing and how to obtain them
4. WHEN documents are complete, THE BimaSetu_System SHALL provide step-by-step claim filing instructions
5. WHEN a claim is rejected, THE Claims_Assistant_Agent SHALL analyze the rejection reason
6. WHEN analyzing rejections, THE Claims_Assistant_Agent SHALL cite relevant IRDAI regulations that support the user's case
7. WHEN IRDAI violations are found, THE BimaSetu_System SHALL generate an appeal template with regulation citations

### Requirement 11: Family Health Vault

**User Story:** As a family member managing insurance for my household, I want to store and access health information for all family members, so that I can manage everyone's insurance in one place.

#### Acceptance Criteria

1. WHEN a user creates a family profile, THE BimaSetu_System SHALL allow adding up to 10 family members
2. WHEN adding a family member, THE BimaSetu_System SHALL collect name, age, relationship, and health conditions
3. WHEN storing family information, THE BimaSetu_System SHALL encrypt all health data at rest
4. WHEN a user requests family member information, THE BimaSetu_System SHALL retrieve and display the requested profile
5. WHEN uploading policy documents, THE BimaSetu_System SHALL allow associating policies with specific family members
6. THE BimaSetu_System SHALL allow users to update or delete family member information

### Requirement 12: Vernacular Language Support

**User Story:** As a non-English speaker, I want to interact with BimaSetu in my native language, so that I can fully understand insurance information.

#### Acceptance Criteria

1. WHEN a user sends a message in Hindi, Tamil, Telugu, Bengali, Marathi, Gujarati, Kannada, or Malayalam, THE BimaSetu_System SHALL detect the language
2. WHEN the language is detected, THE BimaSetu_System SHALL process the message and respond in the same language
3. WHEN transcribing voice messages, THE BimaSetu_System SHALL support Hindi and regional languages
4. WHEN translating insurance terminology, THE BimaSetu_System SHALL use culturally appropriate explanations
5. THE BimaSetu_System SHALL maintain language consistency throughout a conversation session

### Requirement 13: Policy Document Parsing

**User Story:** As a developer, I want the system to accurately parse insurance policy PDFs, so that users receive correct policy information.

#### Acceptance Criteria

1. WHEN parsing a policy PDF, THE Policy_Intelligence_Agent SHALL extract text with at least 95% accuracy
2. WHEN extracting text, THE Policy_Intelligence_Agent SHALL preserve document structure including headings, sections, and tables
3. WHEN encountering scanned PDFs, THE Policy_Intelligence_Agent SHALL perform OCR to extract text
4. WHEN parsing tables, THE Policy_Intelligence_Agent SHALL maintain row and column relationships
5. WHEN parsing fails, THE BimaSetu_System SHALL inform the user and request a clearer document
6. THE Policy_Intelligence_Agent SHALL identify key policy sections including coverage, exclusions, claim process, and terms

### Requirement 14: Data Security and Privacy

**User Story:** As a user, I want my personal and health information to be secure, so that my privacy is protected.

#### Acceptance Criteria

1. THE BimaSetu_System SHALL encrypt all user data in transit using TLS 1.3 or higher
2. THE BimaSetu_System SHALL encrypt all user data at rest using AES-256 encryption
3. WHEN storing policy documents, THE BimaSetu_System SHALL store them in secure, access-controlled storage
4. WHEN accessing user data, THE BimaSetu_System SHALL authenticate the user via WhatsApp phone number verification
5. THE BimaSetu_System SHALL not share user data with third parties without explicit user consent
6. WHEN a user requests data deletion, THE BimaSetu_System SHALL permanently delete all user data within 30 days

### Requirement 15: Agent Orchestration

**User Story:** As a system architect, I want intelligent request routing, so that user queries are handled by the most appropriate specialized agent.

#### Acceptance Criteria

1. WHEN a user sends a message, THE Orchestrator_Agent SHALL analyze the message intent
2. WHEN intent is determined, THE Orchestrator_Agent SHALL route the request to the appropriate specialized agent
3. WHEN multiple agents are needed, THE Orchestrator_Agent SHALL coordinate agent interactions
4. WHEN an agent cannot handle a request, THE Orchestrator_Agent SHALL route to an alternative agent or request clarification
5. THE Orchestrator_Agent SHALL maintain conversation context across agent handoffs
6. WHEN agent processing fails, THE Orchestrator_Agent SHALL provide a graceful error message and recovery options

### Requirement 16: Response Time and Performance

**User Story:** As a user, I want fast responses from BimaSetu, so that I can get timely assistance especially during emergencies.

#### Acceptance Criteria

1. THE BimaSetu_System SHALL respond to text messages within 5 seconds for 95% of requests
2. THE BimaSetu_System SHALL complete policy PDF parsing within 10 seconds for documents under 50 pages
3. THE Emergency_Mode SHALL complete hospital matching within 10 seconds
4. THE BimaSetu_System SHALL transcribe voice messages within 5 seconds for messages under 60 seconds
5. WHEN system load is high, THE BimaSetu_System SHALL queue requests and inform users of expected wait time

### Requirement 17: Error Handling and Recovery

**User Story:** As a user, I want clear error messages when something goes wrong, so that I know how to proceed.

#### Acceptance Criteria

1. WHEN an error occurs, THE BimaSetu_System SHALL provide a user-friendly error message in the user's language
2. WHEN a service is temporarily unavailable, THE BimaSetu_System SHALL inform the user and provide an estimated recovery time
3. WHEN user input is invalid, THE BimaSetu_System SHALL explain what is wrong and how to correct it
4. WHEN a critical error occurs during Emergency_Mode, THE BimaSetu_System SHALL provide emergency contact numbers
5. THE BimaSetu_System SHALL log all errors for system monitoring and debugging

### Requirement 18: Card Information Security

**User Story:** As a card holder, I want my card information to be handled securely, so that I'm protected from fraud.

#### Acceptance Criteria

1. WHEN collecting card details, THE BimaSetu_System SHALL only request card type, issuer, and last 4 digits
2. THE BimaSetu_System SHALL not store full card numbers or CVV codes
3. WHEN identifying card insurance, THE Card_Discovery_Agent SHALL use card metadata without requiring sensitive information
4. THE BimaSetu_System SHALL comply with PCI-DSS standards for card data handling
5. WHEN card information is no longer needed, THE BimaSetu_System SHALL delete it from temporary storage

### Requirement 19: Conversation Context Management

**User Story:** As a user having a multi-turn conversation, I want the system to remember previous messages, so that I don't have to repeat information.

#### Acceptance Criteria

1. THE BimaSetu_System SHALL maintain conversation history for the duration of a session
2. WHEN a user refers to previous messages using pronouns or context, THE BimaSetu_System SHALL resolve references correctly
3. WHEN a session is inactive for 30 minutes, THE BimaSetu_System SHALL end the session and clear context
4. WHEN starting a new session, THE BimaSetu_System SHALL retrieve relevant user profile and policy information
5. THE BimaSetu_System SHALL allow users to explicitly start a new conversation topic

### Requirement 20: Ambulance Dispatch Integration

**User Story:** As someone in a medical emergency, I want the system to help me get an ambulance quickly, so that I can reach the hospital fast.

#### Acceptance Criteria

1. WHEN a user requests ambulance service, THE BimaSetu_System SHALL provide ambulance service contact numbers
2. WHEN location is available, THE BimaSetu_System SHALL share the user's location with ambulance services
3. WHEN ambulance is dispatched, THE BimaSetu_System SHALL provide estimated arrival time
4. THE BimaSetu_System SHALL track ambulance status and update the user
5. WHEN ambulance services are unavailable, THE BimaSetu_System SHALL provide alternative emergency transport options
