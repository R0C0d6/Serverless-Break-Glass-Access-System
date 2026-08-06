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
  - [Phase 1 - Foundation](#phase-1---foundation-dynamodb--permission-boundary)
  - [Phase 2 - Identity](#phase-2---identity-the-two-iam-roles)
  - [Phase 3 - Logic](#phase-3---logic-the-lambda-functions)
  - [Phase 4 - Orchestration](#phase-4---orchestration-sns-api-gateway-step-functions)
  - [Phase 5 - Audit and Failsafe](#phase-5---audit-and-failsafe)
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

![Architecture: Serverless Break Glass System, Scenario A](screenshots/architecture/architecture-diagram.png)



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

We created a Standard topic named `breakglass-approvals`, the single channel every notification
in the system flows through: approval requests, credential handoffs, revocations and tripwire
alerts.

![SNS topic details](screenshots/phase_four/sns_topic_details_01.png)

An email subscription stays in *Pending confirmation* until the recipient clicks the link AWS
sends. Until then SNS accepts publishes and silently drops them: no error anywhere.

![Subscription confirmed](screenshots/phase_four/sns_confirmd_subscription.png)

**What we observed:** <!-- TODO: how long the confirmation email took to arrive -->

#### Step 2 · Replace the SNS placeholders in all three functions

<!-- TODO: note how many separate edits this took: this is the pain point that argues for
     environment variables. See Known Limitations #8. -->

![Lambdas updated with the real SNS topic ARN](screenshots/phase_four/updated_lambdas_with_sns_topic.png)

#### Step 3 · Build the approval endpoint in API Gateway

A REST API gives the approver something clickable. The `/approve` resource takes `grant_id` and
`approver` as query-string parameters and hands the whole request to the Approval Handler via
Lambda proxy integration.

![API Gateway configuration](screenshots/phase_four/api_gateway_details_001.png)

![GET method wired to the Approval Handler](screenshots/phase_four/method_for_approvalhandler.png)

![Method details](screenshots/phase_four/api_method_details.png)

Nothing is reachable until the API is deployed to a stage:

![Deploying to the prod stage](screenshots/phase_four/api_deployment.png)

![Invoke URL](screenshots/phase_four/api_method_invoke_url_details.png)

**What we observed:** hitting the invoke URL with a made-up `grant_id` returned
`{"error": "Grant not found"}`. That error is the success signal: it proves API Gateway reached
the Lambda and the Lambda ran its DynamoDB lookup. A wiring failure would have returned a 403 or
502 from the gateway instead.

![Expected failure: Grant not found](screenshots/phase_four/invoke_url_expected_failure.png)

#### Step 4 · Put the real approval link in the email

With a live invoke URL, the Request Handler's email body was updated to embed a clickable approve
link carrying the grant ID.

![Request Handler updated with the approval URL](screenshots/phase_four/modified_request_handler_with_url.png)

**What we observed:** the function ran clean end to end for the first time: `statusCode 200` with
a real `grant_id` and `status: pending`, no SNS placeholder error.

![Successful Request Handler test](screenshots/phase_four/success_test_for_request_handler.png)

<!-- NOTE: if this screenshot was actually taken after Step 6 rather than Step 4,
     move it down to Step 6; it works as evidence for either. -->

#### Step 5 · Build the Step Functions state machine

The state machine is the timer. It reads `wait_seconds` from its input, waits, then invokes
Auto-Revoke with the grant ID. Two states and no branching, deliberately small enough to audit at a
glance.

![State machine definition](screenshots/phase_four/state_machine_code.png)

**What we observed:** <!-- TODO: describe the 10-second test run -->

![Execution with a 10 second wait](screenshots/phase_four/state_machine_graph_10seconds.png)

![Execution detail](screenshots/phase_four/state_machine_execution.png)

![Execution graph](screenshots/phase_four/state_machine_execution_graph.png)

#### Step 6 · Wire the Request Handler to start the state machine

The Request Handler could write to DynamoDB and publish to SNS, but nothing started the timer. That
needed two changes. First, a new `states:StartExecution` permission on the execution role,
scoped to this one state machine:

![Execution role with StartExecution permission](screenshots/phase_four/wired_lambda_execution_role.png)

Second, the `start_execution` call itself, using the grant ID as the execution name so a retry
can never start two timers for the same grant.

![Request Handler starting the state machine](screenshots/phase_four/updated_request_handler-state-machine.png)

**What we observed:** <!-- TODO: all three things firing at once: the DynamoDB record, the
     approval email, and a waiting Step Functions execution -->

---

### Phase 5 - Audit and Failsafe

The first four phases built the machinery that grants access. This one builds the machinery that
watches it. Everything here assumes the rest of the system will eventually fail or be bypassed:
CloudTrail records what happened regardless of how it happened, Security Hub watches for the
controls themselves drifting, an EventBridge tripwire fires the moment the break-glass role is
assumed by anyone through any path, and a scheduled sweep catches grants that Step Functions
never got round to expiring.

#### Step 1 · Confirm CloudTrail is recording

CloudTrail is the immutable record underneath everything else. Every `AssumeRole` call on the
break-glass role lands here with the caller identity, timestamp, source IP, and every subsequent
API call made during that session, which is precisely the attribution a shared emergency account
can never give you.

![CloudTrail trail creation](screenshots/phase_five/cloudtrail_trail_creation_001.png)

Management events are the ones that matter for this system. Read and Write are both enabled so
that lookups as well as changes are captured.

![Management events enabled](screenshots/phase_five/cloudtrail_management_events.png)

**What we observed:** <!-- TODO: what the event history showed when you checked it -->

![Verified event history](screenshots/phase_five/verified_cloudtrail_event_history.png)

#### Step 2 · Enable Security Hub

CloudTrail tells you what happened. Security Hub tells you when the guardrails themselves have
moved, for instance if the permission boundary is ever detached from the break-glass role, or
credentials sit unused and unrotated.

![Enabling Security Hub](screenshots/phase_five/enabling_securityhub.png)

We enabled the **AWS Foundational Security Best Practices** standard, which covers the IAM and
logging checks most relevant to this build.

![Foundational Security Best Practices enabled](screenshots/phase_five/checked_aws_best_foundational_practices.png)

**What we observed:** <!-- TODO: findings take 15–30 min to populate; note what you saw and when -->

#### Step 3 · Build the AssumeRole tripwire

This is the control that does not care whether the approval flow was followed. It watches the
CloudTrail event stream for **any** `AssumeRole` call targeting `BreakGlass-ElevatedAccessRole`
and alerts immediately, legitimate approvals included. An alert on every use is the point: if one
arrives that nobody requested, that is the signal.

![EventBridge AssumeRole alert rule](screenshots/phase_five/eventbrigde_breakglass_assumealert.png)

The event pattern matches on `sts.amazonaws.com` as the source, `AssumeRole` as the event name, and
the specific role ARN as the target: narrow enough that routine role assumptions elsewhere in the
account stay quiet.

![Event pattern watching CloudTrail](screenshots/phase_five/rule_to_watch_cloudtrail_events.png)

The rule publishes to the same `breakglass-approvals` topic the rest of the system uses, so alerts
land in the approver's inbox alongside the approval requests.

![SNS topic selected as the alert target](screenshots/phase_five/selecting_snstopic_for_alert.png)

**What we observed:** <!-- TODO: the alert email, and how long after the assume it arrived -->

#### Step 4 · Build the scheduled failsafe

Step Functions handles expiry on the happy path. This step assumes it will not always. If an
execution fails, is stopped manually, or never starts, a grant can sit at `active` in DynamoDB
indefinitely: the credentials themselves have expired, but the record lies about the system's
state, which matters for auditing.

A fourth Lambda does the sweep: scan for anything still `active` past its `expires_at`, and hand
each one to Auto-Revoke.

![Failsafe Scanner Lambda code](screenshots/phase_five/failsafe_scanner_lambda_code.png)

Calling one Lambda from another needs an explicit permission the execution role did not have, and
it is scoped to Auto-Revoke alone rather than to Lambda in general.

![Invoke Lambda policy](screenshots/phase_five/breakglass_invoke_labda_policy_details.png)

An EventBridge schedule runs the scanner every five minutes: frequent enough that a zombie grant
is never stale for long, cheap enough to leave running permanently.

![Schedule rate](screenshots/phase_five/specifying_scheduled_rule_rate.png)

![Scheduled rule details](screenshots/phase_five/eventbrigde_scheduled_rulee_details.png)

**What we observed:** invoking the scanner manually returned `{"scanned": true, "revoked_count": 0}`,
which is the correct result on a healthy system: the sweep ran, found no grants still marked
`active` past their expiry, and revoked nothing. A non-zero `revoked_count` would mean Step
Functions had failed to expire something.

![Test result](screenshots/phase_five/clean_running_return.png)

---

## End-to-End Test

**Tested:** <!-- TODO: date --> · **Result:** <!-- TODO: passed / passed after fixes -->

<!-- TODO: This is the most credible section in the whole document: a real trace beats
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
<!-- TODO: this is the money shot: the actual SSM session on the instance -->

**8 · The tripwire fires.**
<!-- TODO: the EventBridge alert email -->

**9 · The grant expires.**
<!-- TODO: record flipping to expired -->

**10 · The credentials stop working.**
<!-- TODO: what error you got when you tried to reuse them -->

### Proving the boundary holds

A working session only shows that access was granted. What matters more is what that access
*cannot* do. With the break-glass credentials loaded and a live session open on the instance, we
deliberately tried actions outside the boundary.

Setting up the client first, including the Session Manager plugin the CLI needs to open a session
at all:

![Configuring credentials and installing the Session Manager plugin](screenshots/tests/installing_ssh_plugin.jpg)

Then the session itself, followed by the denials:

![Session opened, then non-SSM calls denied](screenshots/tests/successful_session_started.jpg)

Reading that session in order:

| Action | Result | What it proves |
|---|---|---|
| `aws sts get-caller-identity` | `assumed-role/BreakGlass-ElevatedAccessRole/breakglass-…` | The role really was assumed, and the session name carries the grant ID |
| `aws ec2 describe-instances --filters Name=tag:BreakGlassEligible,Values=true` | returned the tagged instance | The one permitted lookup works |
| `aws ssm start-session --target i-0c71…` | `Starting session with SessionId: breakglass-…` | The intended capability works after the boundary fix |
| `aws s3 ls` | `AccessDenied` on `s3:ListAllMyBuckets` | S3 is outside the boundary |
| `aws iam list-users` | `AccessDenied` on `iam:ListUsers` | IAM is outside the boundary, so the session cannot escalate itself |
| `whoami` | `ssm-user` | The shell runs as the unprivileged Session Manager user, not `root` |

The two `AccessDenied` responses are the whole point of the exercise. The session is live and the
credentials are valid, yet S3 and IAM are both refused. `iam:ListUsers` matters most: a session
that cannot read IAM cannot begin to map a privilege-escalation path, let alone attach itself a
broader policy.

<!-- TODO: the checks below are enforced in code and by the boundary but were not captured
     during this run. Worth running and screenshotting: they are quick, and each one is
     stronger evidence than a success. -->

Still to be captured:

- **Requester approves their own request** — expect `403` from the Approval Handler
- **Approving a grant that is already active** — expect `409`
- **Opening a session on an untagged instance** — expect `AccessDenied`, proving the tag condition
- **Reusing the credentials after expiry** — expect `ExpiredToken`

---

## Lessons Learned

### SSM Session Manager needs the session document in the boundary

**Symptom.** The approval flow worked end to end. Credentials were issued, `aws sts
get-caller-identity` confirmed the break-glass role had been assumed, and `ec2:DescribeInstances`
correctly returned the tagged instance. But the actual session refused to open:

```
An error occurred (AccessDeniedException) when calling the StartSession operation:
User: arn:aws:sts::<account>:assumed-role/BreakGlass-ElevatedAccessRole/breakglass-3bc6c9ef
is not authorized to perform: ssm:StartSession
on resource: arn:aws:ssm:us-east-1:<account>:document/SSM-SessionManagerRunShell
because no identity-based policy allows the ssm:StartSession action
```

![StartSession denied on the session document](screenshots/tests/policy_issue_error.jpg)

Everything upstream said the permissions were correct, which is what made this confusing.

**Diagnosis.** Read the resource in that error carefully: it is not the EC2 instance, it is
`document/SSM-SessionManagerRunShell`. `ssm:StartSession` authorises against *two* resources, not
one: the target instance **and** the Session Manager document that defines what the session may
do. Our boundary allowed only the instance, so the call was denied at the document.

The document also cannot be covered by the same statement, because the
`ssm:resourceTag/BreakGlassEligible` condition has nothing to match on a document ARN. Folding it
into the existing statement would silently deny it again.

**Fix.** A second statement in the permission boundary, allowing `ssm:StartSession` on the session
document ARNs and carrying no tag condition:

```json
{
  "Sid": "AllowSessionDocument",
  "Effect": "Allow",
  "Action": "ssm:StartSession",
  "Resource": [
    "arn:aws:ssm:*:*:document/SSM-SessionManagerRunShell",
    "arn:aws:ssm:*:*:document/AWS-StartSSHSession"
  ]
}
```

<!-- TODO: replace the block above with the exact statement you added, if it differs. -->

The instance-scoped statement stays exactly as it was, so the tag condition still governs *which
machine* can be reached. The new statement only unblocks the document that the session runs
through.

**Takeaway.** A permission boundary is only as good as your understanding of which resources an API
call actually touches, and a single API call can touch more than one. The denial message names the
exact resource that failed, so read it literally rather than assuming you know which resource was
meant.

### Temporary credentials break if a newline sneaks into the token

**Symptom.** Before the policy fix, an earlier attempt failed differently. Every AWS CLI call
returned `An HTTP Client raised an unhandled exception: Invalid header value` rather than any AWS
error at all.

![Invalid header value](screenshots/tests/error_due_to_policy_issues.jpg)

**Diagnosis.** This is not an AWS problem. A session token copied out of the notification email
carried a trailing newline into the `export` statement, and a newline is not a legal character in
an HTTP header. The request never left the machine, so AWS never saw it and never had a chance to
allow or deny it.

**Takeaway.** An error raised by the HTTP client rather than by AWS means the request was malformed
locally. Worth recognising quickly, because it looks alarming and has nothing to do with your
permissions. It is also a direct argument for
[Known Limitation 3](#known-limitations): credentials pasted by hand out of an email are fragile as
well as insecure.

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
They persist in mailboxes, backups and SNS delivery logs, and they reach the approver rather
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
*Planned fix:* Terraform (see [`terraform/`](terraform/)).

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
- [ ] <!-- TODO: second scenario: scoped S3 access -->
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
    ├── phase_three/           # Lambda functions
    ├── phase_four/            # SNS, API Gateway, Step Functions
    ├── phase_five/            # CloudTrail, Security Hub, EventBridge
    └── tests/                 # End-to-end session and boundary-denial evidence
```

---

## Cost Note

<!-- TODO: Short paragraph. Most of this stack is free-tier friendly, but Security Hub bills
     per check per account and is easy to leave running. Flag anything someone reproducing
     this should know before enabling. -->

---

## License

See [LICENSE](LICENSE).
