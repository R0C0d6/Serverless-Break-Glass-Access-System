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
  - [Phase 1 - Foundation](#phase-1--foundation-dynamodb--permission-boundary)
  - [Phase 2 - Identity](#phase-2--identity-the-two-iam-roles)
  - [Phase 3 - Logic](#phase-3--logic-the-lambda-functions)
  - [Phase 4 - Orchestration](#phase-4--orchestration-sns-api-gateway-step-functions)
  - [Phase 5 - Audit and Failsafe](#phase-5--audit-and-failsafe)
- [End-to-End Test](#end-to-end-test)
- [Lessons Learned](#lessons-learned)
- [Known Limitations](#known-limitations)
- [Roadmap](#roadmap)
- [Repository Layout](#repository-layout)

---

## The Problem

*This project started as an exam question.*

Preparing for the AWS Certified Security - Specialty exam, we hit a scenario that stuck with us. A
company needed a break-glass identity for when something went badly wrong - and had to reach its
workloads with **no inbound ports open**. Nothing on 22, no bastion, no SSH keys in circulation.
The intended answer was Session Manager, and picking it took five seconds. Imagine our surprise when we had it wrong. The better answer to the question was what it takes to actually build it.

     Because the real-world version is usually a root password in a sealed envelope, or an
     'AdministratorAccess' role created during an outage two years ago that nobody dares delete. 
     
They share one defect: the access is **standing**. It exists before the emergency and it is still there long after. Nothing expires, nobody approves its use, and the audit trail records that "the
break-glass account" did something without ever establishing who was holding it - which is exactly
what an attacker prefers.

So we built it properly: access that does not exist until it is requested, needs a second person
to approve it, is capped at SSM sessions on tagged instances by a permission boundary, expires on
a timer, and leaves a per-grant record of who asked, who approved, why, and for how long.

No open ports. No standing privilege. No anonymous emergency account.

---

## What This System Does

Our system provides a secure way to access critical AWS resources during emergencies without relying on permanently privileged accounts. Instead of leaving administrator access available all the time, it grants temporary, closely monitored access only when it is genuinely needed. 

Every request must be approved, every action is logged, and access is automatically revoked after a set period(30 minutes in our initial configuration). This reduces security risks while ensuring administrators can still respond quickly when unexpected issues arise.

Key properties:

- **No standing privilege**  <!-- TODO: one line -->
- **Two-person rule**  <!-- TODO: one line -->
- **Hard permission ceiling**  <!-- TODO: one line -->
- **Time-boxed by construction**  <!-- TODO: one line -->
- **Fully audited**  <!-- TODO: one line -->

---

## Architecture

**Region:** `us-east-1`

**Scenario implemented:** Scenario A - SSM Session Manager access to a tagged EC2 instance.

![Architecture — Serverless Break Glass System, Scenario A](screenshots/architecture/architecture-diagram.png)



> The process starts when an engineer needs emergency access to an EC2 instance. Instead of having permanent administrator privileges, the engineer submits a request for temporary elevated access. From the very beginning, a permission boundary is attached to the engineer's role, making sure the requested access can never exceed a predefined limit. The request is then handled by the **Request Handler Lambda**, which stores the request in **Amazon DynamoDB** and sends an approval notification through **Amazon SNS** to the designated approver.

> Once the approver reviews and approves the request, the **Approval Handler Lambda** uses **AWS Security Token Service (STS)** to temporarily assume the break-glass role and grant the required permissions. The engineer can then securely connect to the EC2 instance using **AWS Systems Manager (SSM) Session Manager**, without the need for SSH keys or opening network ports. Every step of the process is recorded in **AWS CloudTrail**, while **Security Hub** keeps an eye out for suspicious activity. Finally, **AWS Step Functions** keeps track of how long the emergency access should remain active. When the approved time expires, it automatically triggers the **Auto-Revoke Lambda**, removing the elevated permissions and restoring the environment to its normal, secure state.


---

## How Access Flows

**1 · Request.**
The engineer requests emergency access by submitting the reason for access and the EC2 instance they need to work on. The **Request Handler Lambda** validates the request, stores it in **Amazon DynamoDB**, and marks its status as **`pending`** while it waits for approval.

**2 · Notify.**
As soon as the request is created, **Amazon SNS** sends an email to the designated approver. The email includes details such as who requested access, the target EC2 instance, the reason for the request, and links to approve or reject it.

**3 · Approve.**
The approver reviews the request and, if everything looks legitimate, approves it. The **Approval Handler Lambda** performs the necessary checks, uses **AWS STS** to create temporary credentials, grants access by assuming the break-glass role, and updates the request status to **`active`**.

**4 · Use.**
The engineer can now open a secure **AWS Systems Manager (SSM) Session Manager** session to the EC2 instance. They can perform only the actions allowed by the temporary role and its permission boundary, nothing more. Every action is logged for auditing.

**5 · Expire.**
When the approved access period ends, **AWS Step Functions** automatically triggers the **Auto-Revoke Lambda**. The temporary credentials become unusable, the elevated permissions are removed, and the request status changes to **`expired`**.

**6 · Failsafe.**
As an extra layer of protection, a scheduled check runs every five minutes to look for any expired requests that may not have been cleaned up. If it finds one, it revokes any remaining access, updates the request status if needed, and ensures no temporary privileges are left behind.

---

## The Security Model

Four independent controls. 
### 1 · The permission boundary is a hard ceiling

A **permission boundary** acts like a safety ceiling for an IAM role. It defines the maximum permissions that the role can ever have, regardless of any other permissions attached to it later.

Think of it as a permanent guardrail. You can attach new policies to the role, but those policies can only grant permissions that already exist inside the boundary. Anything outside the boundary is automatically blocked.

This is why the permission boundary is the centrepiece of the design: it gives you confidence that even if the role is modified in the future, it cannot suddenly gain more power than you originally allowed. The boundary stays in place and continues to limit the role throughout its lifetime.


The policy we attached:

![Permission boundary JSON](screenshots/phase_one/permission_boundary_json.png)

This IAM policy follows the principle of least privilege by allowing only AWS Systems Manager (SSM) Session Manager actions on EC2 instances that are explicitly tagged as `BreakGlassEligible=true`. 

This helps ensure that administrators can securely access only approved emergency ("break-glass") instances while preventing unauthorized access to other resources.

![Permission boundary created](screenshots/phase_one/break_glass_permission_boundary_details.png)

### 2 · Access is opt-in per instance

The `BreakGlassEligible=true` condition adds an extra layer of security by requiring an EC2 instance to be explicitly tagged before the break-glass role can access it through AWS Systems Manager Session Manager. 

Until an administrator deliberately applies this tag, the instance remains inaccessible to the break-glass role, reducing the risk of unintended or unauthorized emergency access.


![Instance tagged BreakGlassEligible](screenshots/phase_one/break_glass_eligible_tagging.png)

### 3 · Only Lambda can assume the elevated role

The **trust policy** is the first security checkpoint for this role. It defines exactly **who is allowed to assume it** - and in this case, the answer is only one specific identity: **`BreakGlass-LambdaExecutionRole`**.

This means no human user, administrator, or someone with valid AWS credentials can directly assume this role. Having powerful permissions is not enough; the identity must also be explicitly trusted by the role’s trust policy. If you are not listed as a trusted principal, AWS will deny the assumption request.

The inclusion of **`sts:TagSession`** adds another layer of control. It allows the trusted Lambda role to pass session tags when assuming the role. These tags provide additional context about the session, such as the request source, ticket reference, or approval details, making it easier to track, audit, and enforce security decisions during the temporary access period.

In short, the trust policy ensures that this sensitive role cannot be casually accessed. Only the approved automated workflow can request the elevated permissions, and every session can carry traceable context for auditing.

![Break-glass role trust policy](screenshots/phase_two/break_glass_iamRole_json.png)

### 4 · Separation of duties

The solution includes a separation-of-duties check to help prevent the same person from both requesting and approving emergency access. In the current implementation, this is enforced by the Approval Handler through a simple string comparison between the requester's identity and the approver's identity. 
While this provides a basic safeguard against self-approval, it should not be considered a foolproof security control for production. A more robust implementation would use identity-aware authorization mechanisms and stronger validation to enforce separation of duties in production environments.


### Least privilege on the Lambda execution role

The Lambda execution role is the highest-value identity in the break-glass solution because it is responsible for granting temporary elevated access by assuming the **BreakGlass-ElevatedAccess** role. Since compromising this role could allow an attacker to issue privileged credentials, it was intentionally restricted using the principle of least privilege.

As shown in the policy, the role can assume only the designated break-glass IAM role, perform only the required operations on the specific DynamoDB grants table, publish approval and notification messages, and write execution logs to CloudWatch. By limiting both the allowed actions and the resources they apply to, the design reduces the attack surface and helps ensure the Lambda function can perform only its intended responsibilities, nothing more.

![Lambda execution role inline policy](screenshots/phase_two/lambda_execution_role_inline_policy.png)

---

## Build Walkthrough

### Phase 1 - Foundation: DynamoDB + permission boundary

*Phase 1* establishes the security foundation of the break-glass solution by creating the DynamoDB table used to securely track access requests and approvals, while also implementing a permission boundary to limit the maximum privileges that can ever be granted. 
Building these controls first ensures that every subsequent component operates within predefined security limits and that all privileged access can be recorded and monitored from the outset. This security-first approach reduces the risk of privilege escalation and provides a trusted foundation for the rest of the system.


#### Step 1 · Create the grants table

We created a DynamoDB table named `breakglass-grants` with `grant_id` (String) as the partition
key, and left every other setting at its default.

 The table stores a single record for each break-glass request, making `grant_id` a unique identifier that allows each request to be retrieved directly without requiring a sort key. Since the application primarily performs point lookups and updates using this unique identifier, adding a sort key would have introduced unnecessary complexity without providing additional security or performance benefits.

The table was also configured to use **on-demand capacity**, allowing DynamoDB to automatically scale with unpredictable workloads. Break-glass events are expected to occur infrequently but may happen in bursts during an incident, so on-demand capacity eliminates the need for manual capacity planning while ensuring the system remains responsive when emergency access is required. This approach supports both operational reliability and security by ensuring access requests can be processed without delays during critical situations.


![DynamoDB table configuration](screenshots/phase_one/create_dynamodb_table_details.png)

**What we observed:** After creating the table, DynamoDB completed the provisioning process and the table status changed to **Active** within a short period (typically around 30 seconds). Opening the **Explore items** tab showed an empty table, which was the expected result because no break-glass requests had been submitted yet. 




#### Step 2 · Write the permission boundary

We then created and attached a **permission boundary** that defines the maximum permissions any break-glass session can ever receive. The policy allows only the minimum actions required for emergency administration, such as starting AWS Systems Manager (SSM) Session Manager connections to approved EC2 instances and viewing instance details, while restricting access through conditions such as the `BreakGlassEligible=true` resource tag.

This is the most critical step in the entire build because the permission boundary acts as a security guardrail. Even if an IAM role or policy is accidentally configured with broader permissions in the future, the boundary prevents those permissions from exceeding the limits defined in this policy. As a result, emergency access remains tightly controlled, restricted to explicitly approved resources, and aligned with the principle of least privilege, significantly reducing the risk of privilege escalation or unauthorized access during a break-glass event.

![Permission boundary JSON](screenshots/phase_one/permission_boundary_json.png)

**What we observed:**

![Policy created successfully](screenshots/phase_one/permission_boundary_success.png)

#### Step 3 · Prepare the target EC2 instance

The instance needs an instance profile with SSM permissions before Session Manager can reach it.

![EC2 SSM role permissions](screenshots/phase_one/ec2_ssm_role_permissions.png)

![Instance profile attached](screenshots/phase_one/ec2_attacched_ssmrole_details.png)

**What we observed:** After attaching the IAM instance profile with the required AWS Systems Manager (SSM) permissions, the EC2 instance took a few minutes (around 3 minutes) to register with AWS Systems Manager. Once the SSM Agent established communication with the service, the instance appeared in **Fleet Manager** with a **Managed** status. This confirmed that the instance could now be securely accessed through Session Manager without exposing SSH (port 22) to the internet, improving the security posture by enabling authenticated, audited, and encrypted administrative access.


![Instance visible in Fleet Manager](screenshots/phase_one/ec2_fleet_manager_check.png)

#### Step 4 · Tag the instance as eligible

We tagged the EC2 instance with **`BreakGlassEligible=true`**, ensuring that only explicitly approved instances satisfy the permission boundary and are eligible for emergency access.

![BreakGlassEligible tag applied](screenshots/phase_one/break_glass_eligible_tagging.png)

---

### Phase 2 - Identity: the two IAM roles

We established the identities that make the break-glass workflow secure by separating day-to-day operations from temporary privileged access. Two IAM roles are created: one for requesting emergency access and another that provides the elevated permissions only after the required approval process.

One implementation challenge was a chicken-and-egg dependency: the elevated role's trust policy needed to reference the requester role before that role existed. To resolve this, the roles were created in stages-first creating the roles with a temporary trust configuration, then updating the trust policy once both roles were available. This ensured the trust relationship was established correctly while maintaining a secure and controlled deployment process.

#### Step 1 · Create the break-glass role with a custom trust policy

Think of a Trust Policy as a highly specific bouncer standing guard over your most sensitive credentials.

    The Rule (Allow / sts:AssumeRole): It permits the entity to "assume" or temporarily adopt the permissions of this break-glass role.

    The Only Exception (Principal: lambda.amazonaws.com): It restricts this ability exclusively to the AWS Lambda service.

Why this matters for our security posture:
Instead of trusting a human user account-which could be compromised, phished, or used unpredictably-this policy ensures the break-glass role can only be triggered by our automated Lambda functions. This guarantees that emergency interventions are executed programmatically, predictably, and leave a flawless, tamper-proof audit trail.

![Trust policy](screenshots/phase_two/break_glass_iamRole_json.png)

![Role created](screenshots/phase_one/break_glassElevatedAccess_details.png)

<!-- NOTE: this screenshot lives in screenshots/phase_one/ but is a Phase 2 artifact.
     Consider moving it to screenshots/phase_two/ and updating this path. -->

**What we observed:** The IAM role was created with its custom trust policy, but currently has no attached permission policies or permissions boundaries.

#### Step 2 · Attach the permission boundary

The moment the permissions boundary is attached, an absolute safety ceiling is locked onto the role. No matter what broad policies are assigned to it later, the break-glass role can never execute actions beyond this hard guardrail.

![Boundary attached to role](screenshots/phase_two/permission_boundary_attach.png)

#### Step 3 · Attach the inline access policy

Our AWS permissions operate on an intersection model: an action is granted only if both the permissions boundary and the inline policy explicitly allow it- like requiring two distinct keys to open a vault door.

This ensures zero drift between what the role is granted to do and what it is bounded to do.

![Inline policy on elevated access role](screenshots/phase_two/inline_policy_for_elevatedaccessrole.png)

#### Step 4 · Create and scope the Lambda execution role

This role grants the Lambda function strictly enough privilege. Limiting it to the exact actions required enforces least privilege, ensuring a compromised function cannot be weaponized to access any other AWS resources.

![Lambda execution role](screenshots/phase_two/breakglass_lambdaexecutionRole_details.png)

![Execution role inline policy](screenshots/phase_two/lambda_execution_role_inline_policy.png)

### Phase 3 - Logic: the Lambda functions

#### Step 1 · Request Handler

Accepts the access request, writes a `pending` record, publishes to SNS, and starts the
state machine.

![Request Handler function](screenshots/phase_three/lambda_request_handler_details00.png)

![Request Handler code](screenshots/phase_three/request_handler_code.png)

We confirmed the function runs under `BreakGlass-LambdaExecutionRole`, not an auto-generated one:

![Execution role confirmed](screenshots/phase_three/request_handler_execution_role.png)
 
![Grant record written](screenshots/phase_one/dynamo_table_items.png)

#### Step 2 · Approval Handler

Before minting any emergency credentials, the Approval Handler enforces strict four-eyes security. It verifies that a valid grant request exists, confirms it is still pending, and guarantees that the approver is not the requester, blocking self-approval attacks at the door.

![Approval Handler function](screenshots/phase_three/lambda_approval_handler_details.png)
 
#### Step 3 · Auto-Revoke

The Auto-Revoke function acts as an automated fail-safe, invalidating temporary access grants and updating the session state as soon as the emergency window expires.

Crucially, it only blocks new API calls made after revocation. It does not forcibly sever active network connections, kill in-flight processes, or terminate live sessions already established during the access window

![Auto-Revoke function](screenshots/phase_three/lambda_auto_revoke_details.png)
 
---

### Phase 4 - Orchestration: SNS, API Gateway, Step Functions

API Gateway exposes secure, authenticated entry points for break-glass requests, while Step Functions orchestrates the precise state machine governing approvals, timeouts, and automated teardowns. 
SNS ties the human element into the loop, dispatching real-time alerts to security teams at every critical lifecycle event. Together, they transform isolated Lambda functions into a deterministic, fully audited emergency workflow with zero room for manual bypass.

#### Step 1 · Create the SNS topic and confirm the subscription

<!-- TODO: what you created; note that an unconfirmed subscription fails silently. -->

<!-- TODO: ![SNS topic](screenshots/phase_four/...) -->

**What we observed:** <!-- TODO -->

#### Step 2 · Replace the SNS placeholders in all three functions

<!-- TODO: how many places needed editing — this is the pain point that argues for
     environment variables. See Known Limitations #8 -->

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
