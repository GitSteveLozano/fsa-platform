# FSA Platform Intelligence: A Technical PM Vision for $120B in Federal Aid

## Overview

This is a portfolio project demonstrating how a Technical PM approaches a broken government platform serving 17 million students annually.

**The Context:**
- Federal Student Aid (FSA) processes $120 billion in federal aid per year
- December 2023 deployment of FAFSA Processing System (FPS) resulted in 30% of forms having processing errors
- GAO documented that 9 of 25 contractual requirements remain undeployed as of September 2025
- Current architecture is batch-driven, opaque, and creates 4–12 week delays for students

**The Role:**
- USDS embedded Technical PM for FSA, reporting to Deputy Director
- No direct reports; influence spans contractors, federal agencies (IRS/SSA/DHS), FSA leadership, and engineering teams
- Responsibility: platform infrastructure roadmap, API contract design, prioritization frameworks, and architecture decisions

**This Project:**
A presentation + README that demonstrates the three skills a Technical PM must have:

1. **Problem Decomposition** — Understanding why the system fails at a structural level
2. **Stakeholder Translation** — Explaining technical problems in terms of student impact and business value
3. **Architecture Thinking** — Envisioning how to replace batch processing with real-time, API-first infrastructure

---

## The Problem: Why Batch Processing Breaks a Government Platform

### Current Flow (Documented via GAO Reports)

```
Student submits FAFSA
    ↓ (overnight batch)
FPS processes form
    ↓ (async, days)
IRS FTI (income verification) request
    ↓ (async, 3-day wait)
SSA identity verification
    ↓ (batch, formula errors)
SAI calculation (with bugs in asset field handling)
    ↓ (batch, delayed)
ISIR sent to college FAA
    ↓ (weeks of manual work)
Financial aid package offered to student

Timeline: 4–12 weeks
Result: 3.5M students without confirmation for weeks, 30% with errors
```

### Why This Matters (By Persona)

**Maya (17-year-old, first-gen student)**
- *JTBD:* "Help me figure out how much aid I can get before I commit to a school"
- *Pain:* Waits weeks for confirmation. SSA verification takes 3 days alone. Doesn't know if her form was received, lost, or errored. Misses enrollment deadline.

**David (Financial Aid Administrator)**
- *JTBD:* "Get ISIRs from FSA so I can build aid packages on my deadline"
- *Pain:* ISIRs delayed by months. Can't correct errors (9 FPS requirements undeployed). Can't tell students anything specific when FSA fails.

**Jordan (Platform Engineer, contractor)**
- *JTBD:* "Ship code that works without causing an incident"
- *Pain:* Test cases have no traceability to requirements (GAO finding). Unclear which FPS requirements are actually deployed. No DORA metrics to know if deploys are getting faster or slower.

**Technical PM (the role)**
- *JTBD:* "Make prioritization decisions that improve outcomes for millions of students"
- *Pain:* No observability into platform health. Can't distinguish between operational incidents and architectural problems. No clear governance for deciding what gets fixed when.

---

## Root Causes (From GAO Analysis)

| Problem | GAO Finding | Student Impact | Root Cause |
|---------|-------------|-----------------|-----------|
| **30% of forms had processing errors** | FPS deployed with incomplete functionality (12/2023) | 5M students received incorrect aid offers | SAI formula missing asset data fields. No formula validation in code. |
| **3.5M students unconfirmed for weeks** | No status notification system | Students can't plan enrollment or financial prep | Batch processing with no feedback loop. No real-time progress tracking. |
| **9 of 25 FPS requirements undeployed** | Contractor failed to deliver; FSA lost tracking (9/2025) | 3M forms require manual correction by FAAs | No requirement traceability. Contractor accountability undefined. Test coverage unknown. |
| **IRS/SSA/DHS integrations opaque** | "Availability issues, recurring errors, long wait times" (GAO) | Unknown (failures detected only when students report) | Black-box integrations. No error handling spec. No fallback when federal systems fail. |
| **Slow deployment cadence** | No DORA metrics published | Every bug fix takes weeks instead of days | Change control process designed for annual waterfall, not iterative delivery. |

---

## The Vision: From Weeks to Same-Day via Agent-Orchestrated APIs

### Reimagined Flow

```
Student initiates FAFSA
    ↓ (event triggers)
Orchestrator Agent dispatches parallel verification:
    ├─→ IRS Income Agent (FTI lookup, API call, <2s)
    ├─→ SSA Identity Agent (verification, API call, <1s)
    ├─→ DHS Eligibility Agent (residency check, API call, <3s)
    └─→ All agents run in parallel
    ↓ (results assembled)
Eligibility Calculation Agent runs SAI formula
    ├─ Input: validated IRS/SSA/DHS data
    ├─ Output: SAI + Pell eligibility
    └─ Audit trail logged for compliance
    ↓ (ISIR ready)
FAA Notification Agent alerts college in real-time
    ↓ (human in the loop)
FAA reviews and approves aid package
    ↓ (audit logged)
Student receives decision

Timeline: Same day
Result: Real-time status, zero delays, full auditability
```

### Key Architectural Changes

**1. API-First Integration Layer**

Replace batch integrations with MCP-connected agents that expose structured API contracts:

```
IRS FTI Agent
  Input:  {ssn, tax_year, consent_token}
  Output: {agi, tax_status, asset_value, confidence, audit_id}
  SLA:    <2s, with fallback to manual_review

SSA Verification Agent
  Input:  {name, dob, ssn}
  Output: {identity_confidence_0_to_1, audit_id}
  SLA:    <1s, threshold ≥0.85 auto-approves

DHS Eligibility Agent
  Input:  {citizenship_status, residency}
  Output: {aid_eligible, disqualifying_reason_optional, audit_id}
  SLA:    <3s, edge cases routed to manual_review
```

The Technical PM owns these API contracts. Engineers implement them. Policy teams review compliance implications. This is the spec work that replaces ambiguous requirements with deterministic systems.

**2. Event-Driven Orchestration**

FAFSA submission becomes an event that triggers parallel agent execution, not a batch job that waits overnight:

```yaml
Event: FASFASubmitted
  ├─ Trigger: submission_validated
  ├─ Parallel Agents:
  │   ├─ IRSAgent.verify_income()
  │   ├─ SSAAgent.verify_identity()
  │   └─ DHSAgent.check_eligibility()
  ├─ Collect Results: wait_for_all(timeout=5s)
  ├─ Calculate: SaiAgent.calculate(results)
  ├─ Audit: AuditAgent.log(all_steps)
  └─ Notify: NotificationAgent.alert_faa(isir)
```

This architecture requires event-driven infrastructure, not batch scheduling. The Technical PM designs the state machine and timeout handling.

**3. Requirement Traceability (What Was Missing)**

The GAO found "test cases with no traceability to requirements." This framework prevents that:

```
User Need: "Aid offer must be correct before enrollment deadline"
  ├─ Requirement: "SAI calculation must include asset data for dependent students"
  │   ├─ API Spec: "asset_value field in IRS output"
  │   ├─ Test Case: "TC-SAI-002: Verify asset_value is included in calculation"
  │   ├─ Deployed: YES (v1.4.2, 2026-03-15)
  │   └─ Status: ✓ Verified in production

User Need: "Students know where they are in the process"
  ├─ Requirement: "Real-time status notifications to student"
  │   ├─ API Spec: "POST /v1/notifications/student-status"
  │   ├─ Test Case: "TC-NOTIF-001: Student receives status update within 2s of ISIR generation"
  │   ├─ Deployed: NO (scheduled for Month 2 roadmap)
  │   └─ Status: ⚠ In development
```

Every shipped feature maps to a user need. Every user need has tests. GAO auditors can see the traceability. Contractors know exactly what "done" means.

**4. Student Impact Quantification**

Prioritization isn't based on engineering effort alone — it's based on how many students are affected:

| Priority | Tech Debt Item | Students Affected | Risk | Effort | Rationale |
|----------|---|---|---|---|---|
| **P0** | SAI formula — asset data gap | 5M dependent students | Incorrect aid offers | Medium | Largest student population affected by documented bug |
| **P0** | SSA verification latency (3 days → <1s) | 17M all applicants | Enrollment deadline risk | High | Every student is blocked by this. Real-time is possible with API-first architecture. |
| **P1** | Deploy 9 undeployed FPS requirements | 3M forms needing correction | FAAs can't help students | High | Unblocks human-scale problem-solving for edge cases |
| **P1** | Requirement traceability matrix | All teams (risk of silent failures) | Undetectable regressions | Low | Prevents next "9 undeployed requirements" situation |
| **P2** | IRS integration error handling spec | ~500K edge cases/year | Manual review queue backlog | Medium | Lowers manual work, improves SLA, necessary for agent architecture |
| **P2** | DORA metrics instrumentation | All teams (team velocity) | Can't measure improvement | Low | Foundation for continuous deployment optimization |

This table is what a Technical PM presents to FSA leadership. It says: "Here's why we fix the asset data gap before DORA metrics. Here's the student impact equation."

---

## The Technical PM's 90-Day Plan

### Days 1–30: Listen, Map, Don't Break Anything

**Goals:**
- Shadow every engineering team. Understand how code gets from laptop to production.
- Read every GAO finding. Map which documented problems are fixed and which remain.
- Build relationships with FSA leadership, contractor PMs, and integration owners.

**Deliverables:**
- Architecture diagram of current FPS + integration landscape
- Org chart of decision authority (who approves what)
- Risk assessment of the 9 undeployed requirements (which ones are critical)
- DORA metrics baseline (current deploy frequency, lead time for changes, change failure rate)

**Key Decisions:**
- Don't propose changes yet. Listen first.
- Earn standing by asking smart questions, not by having all the answers.

### Days 31–60: Instrument and Prioritize

**Goals:**
- Stand up requirement traceability matrix for all in-flight work
- Build prioritization framework with student impact quantification
- Get alignment with FSA leadership on P0 items

**Deliverables:**
- Requirement traceability spreadsheet (user need → spec → test → deployed)
- Tech debt backlog with student impact and effort estimates
- Proposal for P0 item: "SAI formula asset data gap fix" (spec, test cases, rollback plan)
- DORA metrics dashboard visible to all teams

**Key Decisions:**
- What's the one thing that, if fixed today, would help the most students?
- That's P0. Everything else stacks behind it.

### Days 61–90: Ship One Thing That Matters

**Goals:**
- Own the API contract spec for one integration (IRS FTI or SSA)
- Coordinate with engineers, legal, and policy
- Get it deployed and verified
- Prove that the traceability framework works

**Deliverables:**
- API contract spec for chosen integration (with input/output examples, SLA, error handling)
- Full test coverage with traceability to spec
- Deployment runbook with rollback plan
- Post-deployment verification showing spec compliance and SLA achievement
- Blog post or documentation explaining the architectural change

**Key Decisions:**
- Pick something achievable in 30 days that still moves the needle for students
- Make it a proof of concept for the larger agent architecture vision

### Months 4–12: Architecture for the Agentic Future

**Goals:**
- Propose the MCP-connected agent architecture for all federal integrations
- Build internal alignment across FSA, contractors, and agencies
- Pilot one full integration (IRS → SSA → SAI calculation) as end-to-end proof of concept
- Demonstrate the path from batch to real-time, from weeks to same-day

**Deliverables:**
- Agent architecture design document
- MCP protocol specifications for each federal system
- Pilot implementation code (orchestrator agent, IRS agent, SSA agent)
- Performance comparison: batch vs. agent-driven (latency, error rates, student outcome improvement)
- Business case for full rollout (cost savings, SLA improvement, student outcomes)

---

## Technical Specifications

### API Contract Example: IRS FTI Verification

```json
{
  "agent": "IRSFTIAgent",
  "endpoint": "POST /v1/agents/irs-fti-verify",
  "description": "Retrieve Federal Tax Information from IRS for income verification",
  "authentication": {
    "type": "OAuth2",
    "scopes": ["irs:fti:read"],
    "consent_token": "required, expires 24h after FAFSA submission"
  },
  "input": {
    "ssn": {
      "type": "string",
      "description": "Social Security Number (encrypted in transit)",
      "pattern": "^\\d{9}$"
    },
    "tax_year": {
      "type": "integer",
      "description": "Tax year to retrieve",
      "example": 2025
    },
    "consent_token": {
      "type": "string",
      "description": "Cryptographic proof of student consent (FAFSA sign-off)"
    }
  },
  "output": {
    "agi": {
      "type": "integer",
      "description": "Adjusted Gross Income (cents)",
      "nullable": true,
      "example": 5234500
    },
    "tax_status": {
      "type": "enum",
      "values": ["filed", "unfiled", "nonfiler"],
      "example": "filed"
    },
    "filing_status": {
      "type": "enum",
      "values": ["single", "married_joint", "married_separate", "head_household"],
      "nullable": true
    },
    "asset_value": {
      "type": "integer",
      "description": "Total reportable assets (cents). CRITICAL: Must be included for accurate SAI calculation",
      "nullable": true,
      "example": 25000
    },
    "confidence": {
      "type": "float",
      "range": [0.0, 1.0],
      "description": "IRS confidence in match (1.0 = exact SSN+name match, <0.8 = manual review recommended)"
    },
    "audit_id": {
      "type": "uuid",
      "description": "Immutable audit log reference for compliance review"
    }
  },
  "sla": {
    "p50_ms": 800,
    "p99_ms": 2000,
    "availability": "99.5%"
  },
  "error_handling": {
    "consent_expired": {
      "http_code": 401,
      "fallback": "route_to_manual_review",
      "description": "Student consent token expired or revoked"
    },
    "ssn_not_found": {
      "http_code": 404,
      "fallback": "route_to_manual_review",
      "description": "No matching tax record found"
    },
    "irs_unavailable": {
      "http_code": 503,
      "fallback": "queue_with_retry_backoff",
      "max_retries": 3,
      "retry_after_hours": 1,
      "description": "IRS system unavailable; agent retries, escalates to manual if timeout"
    }
  },
  "audit_requirements": {
    "all_calls_logged": true,
    "retention_years": 7,
    "ferpa_compliant": true,
    "log_fields": ["timestamp", "ssn_last4", "agi_returned", "confidence", "error_if_any", "manual_review_routed"]
  }
}
```

### Requirement Traceability Example

**User Need:** "Students can appeal if their aid offer is incorrect"

```
┌─ Requirement 1.2.3: "SAI calculation must use validated IRS data"
│   ├─ API Spec: IRSFTIAgent.output.asset_value must be included
│   ├─ Test Case: TC-SAI-Asset-001
│   │   ├─ Input: Dependent student with $25,000 assets
│   │   ├─ Expected: asset_value = 25000 in SAI calculation
│   │   └─ Status: ✓ PASSED in v1.4.2
│   ├─ Deployed: 2026-03-15
│   └─ Evidence: SAI formula includes asset_value * 5.64% (parent contribution)
│
├─ Requirement 1.2.4: "FAA can manually override SAI if student appeals"
│   ├─ API Spec: POST /v1/faa/sai-override
│   ├─ Test Case: TC-FAA-Override-001
│   │   ├─ Input: {student_id, new_sai, reason}
│   │   ├─ Expected: Override logged, audit trail shows reason
│   │   └─ Status: ✗ NOT COVERED (undeployed FPS requirement)
│   ├─ Deployed: NO
│   └─ ETA: Month 2, when FPS requirement #7 ships
│
└─ Verification: User can now submit FAFSA, get SAI with asset data, appeal if wrong
    Status: ✓ COMPLETE for happy path, ⚠ PARTIAL for appeal path
```

---

## Why This Approach Works

### 1. **It's Grounded in Real GAO Findings**

Every problem cited is documented by the U.S. Government Accountability Office. This isn't opinion — it's audit findings. When you walk into a USDS interview, you've done the homework.

### 2. **It Demonstrates PM Skills, Not Just Technical Knowledge**

- **Problem decomposition:** You understand why batch processing breaks. You can explain it to non-technical stakeholders.
- **Stakeholder translation:** You quantify impact in terms of students, not story points.
- **Prioritization framework:** You have a defensible method for "what to fix first."
- **Architecture thinking:** You're not just firefighting today's problems — you're envisioning the system for 2030.

### 3. **It Maps Directly to the Job Description**

| JD Requirement | This Project Demonstrates |
|---|---|
| "Platform infrastructure serving dozens of engineering teams" | Architecture diagram + DORA metrics framework + CI/CD roadmap |
| "API design connecting federal systems" | Detailed API contract specs for IRS/SSA/DHS + agent orchestration |
| "Technical debt prioritization balancing user needs and long-term health" | Student impact quantification + 90-day roadmap |
| "Architecture decisions that shape the system for a decade" | Agent-driven, event-based reimagining of the entire flow |

### 4. **It's Buildable (For the Agent Proof of Concept Phase)**

Once you pass Nick's initial screen, the next rounds will likely involve engineers and PMs who will want to see code. The agent architecture is mockable:

```python
# Proof-of-concept orchestrator
class FAFSAOrchestrator:
    def process_submission(self, submission_id):
        # Dispatch parallel agents
        irs_task = asyncio.create_task(
            self.irs_agent.verify_income(submission_id)
        )
        ssa_task = asyncio.create_task(
            self.ssa_agent.verify_identity(submission_id)
        )
        
        # Collect results
        irs_result, ssa_result = await asyncio.gather(
            irs_task, ssa_task, timeout=5
        )
        
        # Calculate SAI
        sai = self.eligibility_agent.calculate(
            irs_result, ssa_result
        )
        
        # Log audit trail
        self.audit_agent.log({
            "submission_id": submission_id,
            "irs_verified": irs_result.confidence,
            "ssa_verified": ssa_result.confidence,
            "sai_calculated": sai
        })
        
        return {"isir": sai, "audit_id": audit_id}
```

This isn't production code. It's a proof of concept that shows you understand event-driven architecture and how agents communicate. That's sufficient for round 2 interviews.

---

## Next Steps: Building the Agent Proof of Concept

Once you pass the initial screen with Nick, the next phase is the technical interview. That's where the agent code comes in.

**What we'd build:**
- Asyncio-based orchestrator that dispatches parallel agent calls
- Mock IRS/SSA/DHS agents (returning realistic test data)
- SAI calculation logic with validated formula
- Audit logging with immutable audit trail
- A simple CLI to submit a FAFSA and see the full process flow

**What it demonstrates:**
- You can architect systems for concurrency
- You understand state management in distributed systems
- You can write testable, auditable code
- You think about user outcomes, not just engineering aesthetics

**Timeline:**
- If you pass Nick's call (Friday), start building this Monday
- 1 week to have a working proof of concept
- That becomes the centerpiece of the technical rounds (week of April 10)

---

## References

- **GAO-23-106376:** "DEPARTMENT OF EDUCATION: Federal Student Aid System Modernization" (2023)
- **GAO-25-107396:** "DEPARTMENT OF EDUCATION: Gaps in Federal Student Aid Contract Oversight and System Testing Need Immediate Attention" (2025)
- **FSA Annual Report FY2024:** studentaid.gov/sites/default/files/fy2024-fsa-annual-report.pdf
- **FAFSA Simplification Timeline:** studentaid.gov — public documentation of rollout and issues

---

## Contact

**Portfolio:** github.com/GitSteveLozano  
**LinkedIn:** linkedin.com/in/stevelr  
**Email:** stephenlozanorivacoba@gmail.com

---

**Last Updated:** April 2, 2026  
**Next Review:** After initial USDS interview (April 3, 2026)
