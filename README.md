# Serverless Break Glass Access System
![AWS Lambda](https://img.shields.io/badge/AWS_Lambda-FF9900?style=for-the-badge&logo=awslambda&logoColor=white)
![AWS Step Functions](https://img.shields.io/badge/AWS_Step_Functions-FF9900?style=for-the-badge&logo=awsstepfunctions&logoColor=white)
![Amazon SNS](https://img.shields.io/badge/Amazon_SNS-FF9900?style=for-the-badge&logo=amazonsns&logoColor=white)
![Amazon API Gateway](https://img.shields.io/badge/Amazon_API_Gateway-FF9900?style=for-the-badge&logo=amazonapigateway&logoColor=white)
![Amazon DynamoDB](https://img.shields.io/badge/Amazon_DynamoDB-4053D6?style=for-the-badge&logo=amazondynamodb&logoColor=white)
![AWS IAM](https://img.shields.io/badge/AWS_IAM-DD344C?style=for-the-badge&logo=amazonaws&logoColor=white)
![AWS STS](https://img.shields.io/badge/AWS_STS-DD344C?style=for-the-badge&logo=amazonaws&logoColor=white)
![AWS CloudTrail](https://img.shields.io/badge/AWS_CloudTrail-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Amazon EventBridge](https://img.shields.io/badge/Amazon_EventBridge-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![AWS Security Hub](https://img.shields.io/badge/AWS_Security_Hub-DD344C?style=for-the-badge&logo=amazonaws&logoColor=white)
![Amazon CloudWatch](https://img.shields.io/badge/Amazon_CloudWatch-FF9900?style=for-the-badge&logo=amazoncloudwatch&logoColor=white)

A serverless solution for emergency access management.

### Authors
- Roland Awuku
- Habiba Adam Salisu

---

## Table of Contents

- [The Problem](#the-problem)
- [What This System Does](#what-this-system-does)
- [Architecture](#architecture)
- [How Access Flows](#how-access-flows)
- [The Security Model](#the-security-model)
- [Build Walkthrough](#build-walkthrough)
  - [Phase 1 — Foundation](#phase-1--foundation-dynamodb--permission-boundary)
  - [Phase 2 — Identity](#phase-2--identity-the-two-iam-roles)
  - [Phase 3 — Logic](#phase-3--logic-the-lambda-functions)
  - [Phase 4 — Orchestration](#phase-4--orchestration-sns-api-gateway-step-functions)
  - [Phase 5 — Audit and Failsafe](#phase-5--audit-and-failsafe)
- [End-to-End Test](#end-to-end-test)
- [Lessons Learned](#lessons-learned)
- [Known Limitations](#known-limitations)
- [Roadmap](#roadmap)
- [Repository Layout](#repository-layout)

---

## The Problem

<!-- TODO: 2–3 paragraphs. Why standing admin access is a liability; what teams do today
     (shared root creds, permanent admin roles, a break-glass password in a vault);
     what goes wrong with those. Keep it concrete — name the failure mode you're preventing. -->

---

## What This System Does

<!-- TODO: One paragraph, plain language. The elevator pitch. -->

**In one sentence:** <!-- TODO -->

Key properties:

- **No standing privilege** — <!-- TODO: one line -->
- **Two-person rule** — <!-- TODO: one line -->
- **Hard permission ceiling** — <!-- TODO: one line -->
- **Time-boxed by construction** — <!-- TODO: one line -->
- **Fully audited** — <!-- TODO: one line -->

---

## Architecture

**Region:** `us-east-1`
**Scenario implemented:** Scenario A — SSM Session Manager access to a tagged EC2 instance.

> **Note:** the diagram below is written in Mermaid. It renders as a picture on GitHub.
> In VS Code's built-in preview it will look like plain code unless you install the
> *Markdown Preview Mermaid Support* extension.

```mermaid
flowchart TD
    ENG([Engineer])
    APR([Approver])

    ENG -->|1 · submit request| RH[Lambda: RequestHandler]

    RH -->|2 · write status=pending| DDB[(DynamoDB<br/>breakglass-grants)]
    RH -->|3 · publish| SNS{{SNS<br/>breakglass-approvals}}
    RH -->|4 · start timer| SFN[Step Functions<br/>BreakGlass-StateMachine]

    SNS -->|5 · approval email| APR
    APR -->|6 · click approve link| APIGW[API Gateway<br/>GET /approve]
    APIGW --> AH[Lambda: ApprovalHandler<br/>enforces requester ≠ approver]

    AH -->|7 · AssumeRole + session tags| STS[STS]
    STS --> ROLE[IAM Role<br/>BreakGlass-ElevatedAccessRole]
    PB[Permission Boundary<br/>SSM only · tagged instances only] -.->|hard ceiling| ROLE
    ROLE -->|8 · ssm:StartSession| EC2[EC2 Instance<br/>tag BreakGlassEligible true]
    AH -->|status=active| DDB
    AH -->|credentials| SNS

    SFN -->|9 · after grant duration| AR[Lambda: AutoRevoke]
    AR -->|status=expired| DDB
    AR --> SNS

    SCHED[EventBridge<br/>rate 5 minutes] --> FS[Lambda: FailsafeScanner]
    FS -.->|zombie grants| AR

    ROLE -.->|AssumeRole event| CT[CloudTrail]
    CT --> EB[EventBridge<br/>AssumeRoleAlert]
    EB --> SNS
```

<!-- TODO: 1–2 paragraphs walking through the diagram in prose. -->

---

## How Access Flows

<!-- TODO: Walk through the lifecycle as a short narrative — six or seven sentences.
     The diagram above shows the wiring; this should explain the story. -->

**1 · Request.** <!-- TODO: what the engineer does, what gets written, record status becomes `pending` -->

**2 · Notify.** <!-- TODO: SNS email to the approver, what the email contains -->

**3 · Approve.** <!-- TODO: approver clicks, checks that run, credentials minted, status becomes `active` -->

**4 · Use.** <!-- TODO: engineer opens the SSM session, what they can and cannot do -->

**5 · Expire.** <!-- TODO: timer fires, status becomes `expired`, credentials die -->

**6 · Failsafe.** <!-- TODO: the 5-minute sweep and what it catches -->

---

## The Security Model

Four independent controls. Every one of them has to fail before privilege escalation is possible.

### 1 · The permission boundary is a hard ceiling

<!-- TODO: Explain what a permission boundary is and why it's the centrepiece.
     Key point to make: it caps the role permanently, no matter what policies
     get attached to it later. -->

The policy we attached:

![Permission boundary JSON](screenshots/phase_one/permission_boundary_json.png)

<!-- TODO: 1–2 sentences on what this JSON allows, in plain English. -->

![Permission boundary created](screenshots/phase_one/break_glass_permission_boundary_details.png)

### 2 · Access is opt-in per instance

<!-- TODO: Explain the BreakGlassEligible=true condition — an instance is invisible
     to the break-glass role until somebody deliberately tags it. -->

![Instance tagged BreakGlassEligible](screenshots/phase_one/break_glass_eligible_tagging.png)

### 3 · Only Lambda can assume the elevated role

<!-- TODO: Explain that the trust policy names exactly one principal —
     BreakGlass-LambdaExecutionRole. No human can assume this role directly, even
     with valid credentials. Mention why sts:TagSession is in there. -->

![Break-glass role trust policy](screenshots/phase_two/break_glass_iamRole_json.png)

### 4 · Separation of duties

<!-- TODO: The requester ≠ approver check. Be honest about how it's enforced today
     (a string comparison in the Approval Handler) and point at Known Limitations #1. -->

### Least privilege on the Lambda execution role

<!-- TODO: Note that this role is the highest-value identity in the system —
     it can mint elevated access — and what you scoped it down to. -->

![Lambda execution role inline policy](screenshots/phase_two/lambda_execution_role_inline_policy.png)

---

## Build Walkthrough

Everything below was built through the AWS Console. The full click-by-click guide is in
[`break-glass-phase1.md`](break-glass-phase1.md).

<!-- TODO: rename that file — it currently holds all five phases despite the name. -->

---

### Phase 1 — Foundation: DynamoDB + permission boundary

<!-- TODO: 2–3 sentences — what this phase establishes and why it has to come first. -->

#### Step 1 · Create the grants table

We created a DynamoDB table named `breakglass-grants` with `grant_id` (String) as the partition
key, and left every other setting at its default.

<!-- TODO: add a sentence on why no sort key and why on-demand capacity. -->

![DynamoDB table configuration](screenshots/phase_one/create_dynamodb_table_details.png)

**What we observed:** <!-- TODO: e.g. table went Active after ~40s; Explore items showed empty -->

#### Step 2 · Write the permission boundary

<!-- TODO: what you pasted and why this is the most important step in the build. -->

![Permission boundary JSON](screenshots/phase_one/permission_boundary_json.png)

**What we observed:**

![Policy created successfully](screenshots/phase_one/permission_boundary_success.png)

#### Step 3 · Prepare the target EC2 instance

The instance needs an instance profile with SSM permissions before Session Manager can reach it.

![EC2 SSM role permissions](screenshots/phase_one/ec2_ssm_role_permissions.png)

![Instance profile attached](screenshots/phase_one/ec2_attacched_ssmrole_details.png)

**What we observed:** <!-- TODO: how long until the instance appeared as Managed in Fleet Manager -->

![Instance visible in Fleet Manager](screenshots/phase_one/ec2_fleet_manager_check.png)

#### Step 4 · Tag the instance as eligible

<!-- TODO: one sentence — this tag is what the boundary condition keys on. -->

![BreakGlassEligible tag applied](screenshots/phase_one/break_glass_eligible_tagging.png)

---

### Phase 2 — Identity: the two IAM roles

<!-- TODO: 2–3 sentences. Worth calling out the chicken-and-egg problem — the trust policy
     references a role that doesn't exist yet — and how you worked around it. -->

#### Step 1 · Create the break-glass role with a custom trust policy

<!-- TODO: what you pasted, and what the trust policy means in plain English. -->

![Trust policy](screenshots/phase_two/break_glass_iamRole_json.png)

![Role created](screenshots/phase_one/break_glassElevatedAccess_details.png)

<!-- NOTE: this screenshot lives in screenshots/phase_one/ but is a Phase 2 artifact.
     Consider moving it to screenshots/phase_two/ and updating this path. -->

**What we observed:** <!-- TODO: e.g. role created with no policies and no boundary yet -->

#### Step 2 · Attach the permission boundary

<!-- TODO: one or two sentences on what changes the moment the boundary is set. -->

![Boundary attached to role](screenshots/phase_two/permission_boundary_attach.png)

#### Step 3 · Attach the inline access policy

<!-- TODO: explain the double-lock — boundary and policy must BOTH allow an action.
     Explain why they're deliberately identical here. -->

![Inline policy on elevated access role](screenshots/phase_two/inline_policy_for_elevatedaccessrole.png)

#### Step 4 · Create and scope the Lambda execution role

<!-- TODO: 1–2 sentences on what this role is allowed to do and why nothing more. -->

![Lambda execution role](screenshots/phase_two/breakglass_lambdaexecutionRole_details.png)

![Execution role inline policy](screenshots/phase_two/lambda_execution_role_inline_policy.png)

**What we observed:** <!-- TODO -->

---

### Phase 3 — Logic: the Lambda functions

<!-- TODO: 2–3 sentences. Three functions, one shared execution role, what each owns. -->

#### Step 1 · Request Handler

Accepts the access request, writes a `pending` record, publishes to SNS, and starts the
state machine.

![Request Handler function](screenshots/phase_three/lambda_request_handler_details00.png)

![Request Handler code](screenshots/phase_three/request_handler_code.png)

We confirmed the function runs under `BreakGlass-LambdaExecutionRole`, not an auto-generated one:

![Execution role confirmed](screenshots/phase_three/request_handler_execution_role.png)

**What we observed:** <!-- TODO: the first test failed at the SNS step because the ARN was still
     a placeholder, but the DynamoDB write succeeded — describe what you saw in the table -->

![Grant record written](screenshots/phase_one/dynamo_table_items.png)

#### Step 2 · Approval Handler

<!-- TODO: what it validates before minting credentials — grant exists, still pending,
     requester ≠ approver. -->

![Approval Handler function](screenshots/phase_three/lambda_approval_handler_details.png)

**What we observed:** <!-- TODO -->

#### Step 3 · Auto-Revoke

<!-- TODO: what it does and — importantly — what it does NOT do (see Known Limitations #5). -->

![Auto-Revoke function](screenshots/phase_three/lambda_auto_revoke_details.png)

**What we observed:** <!-- TODO -->

---

### Phase 4 — Orchestration: SNS, API Gateway, Step Functions

<!-- TODO: 2–3 sentences on what this phase ties together. -->

#### Step 1 · Create the SNS topic and confirm the subscription

<!-- TODO: what you created; note that an unconfirmed subscription fails silently. -->

<!-- TODO: ![SNS topic](screenshots/phase_four/...) -->

**What we observed:** <!-- TODO -->

#### Step 2 · Replace the SNS placeholders in all three functions

<!-- TODO: how many places needed editing — this is the pain point that argues for
     environment variables. See Known Limitations #8. -->

<!-- TODO: ![...](screenshots/phase_four/...) -->

#### Step 3 · Build the approval endpoint in API Gateway

<!-- TODO: REST API, /approve resource, GET method, Lambda proxy integration, prod stage. -->

<!-- TODO: ![...](screenshots/phase_four/...) -->

**What we observed:** <!-- TODO: hitting the URL with a fake grant_id returned "Grant not found",
     which proved the wiring worked -->

#### Step 4 · Put the real approval link in the email

<!-- TODO -->

<!-- TODO: ![Approval email received](screenshots/phase_four/...) -->

#### Step 5 · Build the Step Functions state machine

<!-- TODO: what the machine does — wait, then invoke Auto-Revoke. -->

<!-- TODO: ![State machine definition](screenshots/phase_four/...) -->

**What we observed:** <!-- TODO: the test execution with wait_seconds=10 -->

<!-- TODO: ![Execution graph](screenshots/phase_four/...) -->

#### Step 6 · Wire the Request Handler to start the state machine

<!-- TODO: the extra IAM permission needed, and the code change. -->

**What we observed:** <!-- TODO: all three things firing at once — DynamoDB record, email,
     and a waiting execution -->

---

### Phase 5 — Audit and Failsafe

<!-- TODO: 2–3 sentences. -->

#### Step 1 · Confirm CloudTrail is recording

<!-- TODO -->

<!-- TODO: ![CloudTrail](screenshots/phase_five/...) -->

#### Step 2 · Enable Security Hub

<!-- TODO -->

<!-- TODO: ![Security Hub](screenshots/phase_five/...) -->

#### Step 3 · Build the AssumeRole tripwire

<!-- TODO: the EventBridge pattern and why a real-time alert on role assumption matters. -->

<!-- TODO: ![EventBridge rule](screenshots/phase_five/...) -->

**What we observed:** <!-- TODO: the alert email and how long after the assume it arrived -->

#### Step 4 · Build the scheduled failsafe

<!-- TODO: the fourth Lambda plus the 5-minute schedule, and what failure it protects against. -->

<!-- TODO: ![Failsafe schedule](screenshots/phase_five/...) -->

**What we observed:** <!-- TODO: `{"scanned": true, "revoked_count": 0}` on a clean run -->

---

## End-to-End Test

**Tested:** <!-- TODO: date --> · **Result:** <!-- TODO: passed / passed after fixes -->

<!-- TODO: This is the most credible section in the whole document — a real trace beats
     any amount of description. Narrate the actual run, in order, with the evidence
     inline at each step. -->

**Scenario:** <!-- TODO: e.g. "An engineer needs shell access to a production instance
that has stopped responding to health checks." -->

**1 · The request goes in.**
<!-- TODO: what you sent, what came back -->

**2 · The grant lands as pending.**
<!-- TODO: what the DynamoDB record looked like -->

**3 · The approval email arrives.**
<!-- TODO: how long it took, what the email contained -->

**4 · The timer starts.**
<!-- TODO: state machine sitting in WaitForGrantDuration -->

**5 · The approver clicks approve.**
<!-- TODO: what the browser showed -->

**6 · Credentials are issued.**
<!-- TODO: the second email, record flipping to active -->

**7 · The session opens.**
<!-- TODO: this is the money shot — the actual SSM session on the instance -->

**8 · The tripwire fires.**
<!-- TODO: the EventBridge alert email -->

**9 · The grant expires.**
<!-- TODO: record flipping to expired -->

**10 · The credentials stop working.**
<!-- TODO: what error you got when you tried to reuse them -->

### Negative tests

<!-- TODO: These are what prove the controls actually bite. Each one is worth a
     screenshot of the denial — a denied action is stronger evidence than a
     successful one. -->

**Requester tries to approve their own request.**
Expected `403`. <!-- TODO: observed + screenshot -->

**Approving a grant that is already active.**
Expected `409`. <!-- TODO: observed + screenshot -->

**Opening a session on an untagged instance.**
Expected `AccessDenied`. <!-- TODO: observed + screenshot -->

**Calling a non-SSM API with the break-glass credentials.**
Expected `AccessDenied` — this is the boundary doing its job. <!-- TODO: observed + screenshot -->

**Reusing the credentials after expiry.**
Expected `ExpiredToken`. <!-- TODO: observed + screenshot -->

---

## Lessons Learned

### SSM Session Manager needs the session document in the boundary

**Symptom.** <!-- TODO: what the error actually said, verbatim if you have it. This is the
     problem that cost you the most time during testing — describe it properly. -->

**Diagnosis.** `ssm:StartSession` authorises against *two* resources, not one — the target EC2
instance **and** the Session Manager document (`SSM-SessionManagerRunShell`). The permission
boundary as originally written only allowed the instance, so the call was denied at the document.
The `ssm:resourceTag/BreakGlassEligible` condition also cannot apply to a document ARN, which is
why the document needs its own statement rather than being folded into the existing one.

**Fix.**

```json
<!-- TODO: paste the corrected boundary statement -->
```

<!-- TODO: ![Corrected boundary](screenshots/...) -->

**Takeaway.** <!-- TODO: 1–2 sentences. Something like: a permission boundary is only as good as
     your understanding of which resources an API call actually touches — and the docs don't
     always make that obvious. -->

### <!-- TODO: second lesson, if you have one -->

<!-- TODO -->

---

## Known Limitations

Naming these is deliberate. Each one is a real gap, and together they are the backlog.

**1 · The approval endpoint is unauthenticated.**
`approver` arrives as a query-string parameter supplied by whoever calls the URL, so anyone
holding the link can approve, and the requester ≠ approver check can be sidestepped by simply
typing a different email into the URL.
*Planned fix:* an HMAC-signed, single-use token, or an API Gateway authorizer.

**2 · Approval happens on a `GET`.**
Corporate mail scanners and link-preview services fetch URLs in emails automatically, which can
approve a request before a human ever reads it.
*Planned fix:* confirmation page on `GET`, state change on `POST`.

**3 · Credentials are emailed in plaintext.**
They persist in mailboxes, backups and SNS delivery logs — and they reach the approver rather
than the requester.
*Planned fix:* notify only; the requester retrieves credentials once from an authenticated endpoint.

**4 · Approval reads then writes without a condition.**
Two clicks in quick succession can both pass the "is it still pending?" check and issue two
separate sessions.
*Planned fix:* a `ConditionExpression` on the status update.

**5 · Auto-revoke does not actually revoke.**
It updates the DynamoDB record; the credentials themselves remain valid until natural expiry,
so a session cannot be cut short.
*Planned fix:* an `aws:TokenIssueTime` deny policy plus `ssm:TerminateSession`.

**6 · Pending requests never expire.**
A request raised last week is still approvable today.
*Planned fix:* an age check against `requested_at`.

**7 · The failsafe uses `Scan`.**
Cost and latency grow with the size of the table.
*Planned fix:* a GSI on `status`, queried instead of scanned.

**8 · ARNs are hardcoded in function source.**
Every environment means hand-editing several files, and it is easy to miss one.
*Planned fix:* Lambda environment variables.

**9 · The whole thing was built by console click-through.**
Not reproducible, and no review trail on infrastructure changes.
*Planned fix:* Terraform — see [`terraform/`](terraform/).

---

## Reference

### The grants table

| Attribute | Type | Notes |
|---|---|---|
| `grant_id` | String (PK) | <!-- TODO --> |
| `requester` | String | <!-- TODO --> |
| `instance_id` | String | <!-- TODO --> |
| `reason` | String | <!-- TODO --> |
| `duration_minutes` | Number | <!-- TODO --> |
| `status` | String | `pending` → `active` → `expired` |
| `requested_at` | String | ISO 8601, UTC |
| `approver` | String | <!-- TODO --> |
| `approved_at` | String | <!-- TODO --> |
| `expires_at` | String | <!-- TODO --> |
| `revoked_at` | String | <!-- TODO --> |

### Resources created

| Type | Name |
|---|---|
| DynamoDB table | `breakglass-grants` |
| IAM policy (boundary) | `BreakGlass-PermissionBoundary` |
| IAM role | `BreakGlass-ElevatedAccessRole` |
| IAM role | `BreakGlass-LambdaExecutionRole` |
| Lambda | `BreakGlass-RequestHandler` |
| Lambda | `BreakGlass-ApprovalHandler` |
| Lambda | `BreakGlass-AutoRevoke` |
| Lambda | `BreakGlass-FailsafeScanner` |
| SNS topic | `breakglass-approvals` |
| REST API | `BreakGlass-ApprovalAPI` |
| State machine | `BreakGlass-StateMachine` |
| EventBridge rule | `BreakGlass-AssumeRoleAlert` |
| EventBridge rule | `BreakGlass-FailsafeSchedule` |

---

## Roadmap

- [ ] Codify the whole stack in Terraform (`terraform/main.tf` is currently empty)
- [ ] <!-- TODO: signed, single-use approval tokens -->
- [ ] <!-- TODO: Slack interactive approve/deny buttons in place of email -->
- [ ] <!-- TODO: explicit deny path with an audit record -->
- [ ] <!-- TODO: second scenario — scoped S3 access -->
- [ ] <!-- TODO: CloudWatch dashboard -->

---

## Repository Layout

```
.
├── README.md                  # This document
├── break-glass-phase1.md      # Full console build guide, phases 1–5
├── terraform/                 # Infrastructure as code (in progress)
│   └── main.tf
└── screenshots/               # Build and test evidence
    ├── architecture/          # Diagram stills and walkthrough clip
    ├── phase_one/             # DynamoDB, permission boundary, EC2 prep
    ├── phase_two/             # IAM roles and policies
    └── phase_three/           # Lambda functions
```

---

## Cost Note

<!-- TODO: Short paragraph. Most of this stack is free-tier friendly, but Security Hub bills
     per check per account and is easy to leave running. Flag anything someone reproducing
     this should know before enabling. -->

---

## License

See [LICENSE](LICENSE).
