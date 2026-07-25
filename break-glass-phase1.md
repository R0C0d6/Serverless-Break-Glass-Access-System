# Break Glass System — Console Build Guide
## Phase 1 of 5: DynamoDB Table + IAM Permission Boundary

> **Scope note:** We're building Scenario A first — SSM Session Manager access to an EC2 instance. Approval will go out as an **email link** (via SNS) instead of Slack, to keep this MVP fully inside the AWS console. We'll add Slack later once the core works.

> **Before you start:** Make sure you're logged into the AWS Management Console, and check the **region dropdown** in the top-right corner. Pick one region (e.g. `us-east-1` or wherever you normally build) and **use that same region for every single step below**. If services stop showing things you created, the #1 cause is being in the wrong region.

---

## Step 1: Create the DynamoDB Table (tracks who has access)

This table is the "logbook" — every access grant, who requested it, who approved it, and when it expires, lives here.

1. In the search bar at the top of the AWS Console, type **DynamoDB** and click on it when it appears.
2. On the left sidebar, click **Tables**.
3. Click the orange **Create table** button.
4. Under **Table name**, type exactly: `breakglass-grants`
5. Under **Partition key**, type: `grant_id`
   - Leave the dropdown next to it as **String**.
6. Leave everything else on this page at its default settings — do **not** check any extra boxes, do **not** add a sort key.
7. Scroll to the bottom and click the orange **Create table** button.
8. Wait about 30–60 seconds. The table status will change from "Creating" to "Active" — you can watch this on the Tables list page. Don't move on until it says **Active**.

✅ **Checkpoint:** Click into the `breakglass-grants` table, click the **Explore table items** button — it should show an empty table with no items yet. That's correct, we haven't created any grants.

---

## Step 2: Create the IAM Permission Boundary Policy

This is the single most important security piece of the whole system. Think of it like a fence: no matter what permissions we accidentally attach to the break-glass role later, it can **never** step outside this fence. We're building it to only ever allow SSM Session Manager access — nothing else.

1. In the search bar, type **IAM** and click it.
2. On the left sidebar, click **Policies**.
3. Click the orange **Create policy** button.
4. You'll see two tabs: **Visual** and **JSON**. Click on **JSON**.
5. You'll see a text box with some default text already in it. Click inside the box, press **Ctrl+A** (or **Cmd+A** on Mac) to select everything, then press **Delete** to clear it completely.
6. Copy the entire block below and paste it into that now-empty box:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSSMSessionOnly",
      "Effect": "Allow",
      "Action": [
        "ssm:StartSession",
        "ssm:TerminateSession",
        "ssm:ResumeSession",
        "ssm:DescribeSessions",
        "ssm:GetConnectionStatus"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ssm:resourceTag/BreakGlassEligible": "true"
        }
      }
    },
    {
      "Sid": "AllowDescribeInstances",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

   **What this means in plain English:** "You may only start/stop/resume an SSM session, and only on an EC2 instance that has been specifically tagged with `BreakGlassEligible = true`. You may also look up instance info, but you cannot do anything else — no S3, no IAM, no deleting things, nothing."

7. Click the orange **Next** button (bottom right).
8. Under **Policy name**, type exactly: `BreakGlass-PermissionBoundary`
9. Under **Description**, type: `Maximum allowed scope for any break-glass elevated access role`
10. Scroll down and click the orange **Create policy** button.

✅ **Checkpoint:** Go back to **Policies** in the left sidebar, type `BreakGlass` in the search box at the top of the policy list. You should see `BreakGlass-PermissionBoundary` appear in the results. If it's there, you did it correctly.

---

## Step 3: Tag an EC2 Instance as "Break-Glass Eligible"

Since our permission boundary only allows access to tagged instances, we need at least one instance tagged correctly to test against later.

> If you don't have a test EC2 instance yet, that's okay — we'll come back to this step once you've launched one. You can skip ahead and return here later.

1. In the search bar, type **EC2** and click it.
2. On the left sidebar, click **Instances**.
3. Click the checkbox next to the instance you want to use for testing (it must have the **SSM Agent installed** — Amazon Linux 2/2023 and recent Ubuntu AMIs have this built in already).
4. Click the **Tags** tab near the top of the instance details panel (below the instance list).
5. Click **Manage tags**.
6. Click **Add new tag**.
7. Under **Key**, type: `BreakGlassEligible`
8. Under **Value**, type: `true`
9. Click **Save**.

✅ **Checkpoint:** On the Tags tab, you should now see a row that says `BreakGlassEligible` in the Key column and `true` in the Value column.

---

### What we've built so far
- A DynamoDB table to log every access grant
- A permission boundary that acts as a hard ceiling — SSM session access only, and only on tagged instances
- A tagged test instance to grant access to later

---

---

## Phase 2 of 5: Break-Glass IAM Role + Lambda Execution Role

> **Scope note:** We're creating two IAM roles in this phase. The first is the break-glass role itself — the one that gets temporarily assumed during an incident. The second is the Lambda execution role — the identity our Lambda functions will run as. These two roles are the security spine of the whole system, so we're building them carefully and tightly.

> **Before you start:** Have your AWS Account ID ready — you'll need it when writing resource ARNs. You can find it by clicking your account name in the top-right corner of the AWS Console.

---

## Step 1: Create the Break-Glass IAM Role

This is the role that gets temporarily assumed when an engineer needs elevated access. It has no standing permissions on its own — it can only be used when explicitly handed out by the approval flow. We'll lock down its trust policy so only Lambda can ever assume it.

1. In the search bar, type **IAM** and click it.
2. On the left sidebar, click **Roles**.
3. Click the orange **Create role** button.
4. Under **Trusted entity type**, select **Custom trust policy**.
5. A JSON editor box will appear. Clear everything in it, then copy and paste the trust policy below:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

   **What this means in plain English:** "Only AWS Lambda functions can assume this role — not individual users, not other services, not people logging into the console. We'll tighten this further in Phase 3 to only allow *our specific* Lambda execution role, not just any Lambda in the account."

6. Click the orange **Next** button.
7. On the **Add permissions** page, do **not** attach any policies here — skip it entirely. We'll add the permission boundary and an inline policy in the next two steps. Click **Next**.
8. Under **Role name**, type exactly: `BreakGlass-ElevatedAccessRole`
9. Under **Description**, type: `Temporarily assumed break-glass role for SSM access. Scoped by permission boundary — SSM on tagged instances only.`
10. Click the orange **Create role** button.

✅ **Checkpoint:** Go back to **Roles** in the left sidebar, type `BreakGlass` in the search box. You should see `BreakGlass-ElevatedAccessRole` appear. Click into it — it should show no attached policies, and the "Permissions boundary" section should say "None". That's expected — we're about to fix both in the next two steps.

---

## Step 2: Attach the Permission Boundary to the Break-Glass Role

Right now the role exists but has no ceiling and no permissions — if someone assumed it, they'd get nothing. We now nail the ceiling in place: the `BreakGlass-PermissionBoundary` from Phase 1. Once attached, no matter what policies we later add to this role, it can never do more than SSM on tagged instances.

1. You should still be on the `BreakGlass-ElevatedAccessRole` detail page. If not, go to **IAM → Roles** and click into it.
2. Click the **Permissions** tab.
3. Scroll down to the **Permissions boundary** section (it's below the policies list).
4. Click the **Set boundary** button.
5. Select **Use a permissions boundary to control the maximum role permissions**.
6. In the search box that appears, type `BreakGlass` — you should see `BreakGlass-PermissionBoundary` in the list.
7. Click the radio button next to `BreakGlass-PermissionBoundary` to select it.
8. Click the orange **Set boundary** button.

✅ **Checkpoint:** Back on the **Permissions** tab, scroll down to the "Permissions boundary" section. It should now display `BreakGlass-PermissionBoundary` with a lock icon. The role is now permanently capped — it can never do more than what the boundary allows, even if policies added later try to go further.

---

## Step 3: Attach the Inline Access Policy to the Break-Glass Role

The permission boundary defines the maximum. Now we attach the actual policy that grants the SSM permissions the role will use. The boundary and this policy must both allow an action for it to succeed — if they disagree, the most restrictive one wins.

1. Still on the `BreakGlass-ElevatedAccessRole` detail page, click the **Permissions** tab.
2. Click the **Add permissions** dropdown button, then choose **Create inline policy**.
3. Click the **JSON** tab.
4. Clear the existing text, then copy and paste the policy below:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSSMSessionOnTaggedInstances",
      "Effect": "Allow",
      "Action": [
        "ssm:StartSession",
        "ssm:TerminateSession",
        "ssm:ResumeSession",
        "ssm:DescribeSessions",
        "ssm:GetConnectionStatus"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ssm:resourceTag/BreakGlassEligible": "true"
        }
      }
    },
    {
      "Sid": "AllowDescribeInstances",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

   **What this means:** This inline policy grants SSM access on tagged instances. The permission boundary from Phase 1 says exactly the same thing. Both must agree for any action to succeed — this double-lock is intentional. If someone later attaches a broader managed policy to this role (e.g. `AmazonSSMFullAccess`), the boundary still wins and clamps it back down.

5. Click **Next**.
6. Under **Policy name**, type: `BreakGlass-ElevatedAccessInlinePolicy`
7. Click the orange **Create policy** button.

✅ **Checkpoint:** On the **Permissions** tab, under "Permissions policies", you should now see `BreakGlass-ElevatedAccessInlinePolicy` listed as an inline policy. The "Permissions boundary" section should still show `BreakGlass-PermissionBoundary`. Both are in place. The break-glass role is fully configured.

---

## Step 4: Create the Lambda Execution Role

This role is the identity our Lambda functions will run as. It needs just enough to do the job: write to DynamoDB, publish to SNS, call `sts:AssumeRole` on the break-glass role, and log to CloudWatch. Nothing more. This role is the highest-value target in the system — if someone compromises it, they can grant themselves elevated access. Scope it tightly.

1. In the left sidebar, click **Roles**.
2. Click the orange **Create role** button.
3. Under **Trusted entity type**, select **AWS service**.
4. Under **Use case**, select **Lambda** from the list.
5. Click **Next**.
6. On the **Add permissions** page, do **not** attach any managed policies — we'll use an inline policy instead. Click **Next**.
7. Under **Role name**, type exactly: `BreakGlass-LambdaExecutionRole`
8. Under **Description**, type: `Execution role for break-glass Lambda functions. Tightly scoped to DynamoDB, SNS, STS AssumeRole, and CloudWatch Logs only.`
9. Click the orange **Create role** button.

✅ **Checkpoint:** Go to **IAM → Roles** and search `BreakGlass`. You should now see both `BreakGlass-ElevatedAccessRole` and `BreakGlass-LambdaExecutionRole` in the list. Click into `BreakGlass-LambdaExecutionRole` — it should show no attached policies yet and the trust relationship should show `lambda.amazonaws.com` as the principal.

---

## Step 5: Attach the Lambda Execution Policy

Now we give the Lambda execution role its scoped permissions. You'll need two pieces of information before writing the policy: your AWS Account ID and your region.

1. Go to **IAM → Roles**, click into `BreakGlass-ElevatedAccessRole`.
2. At the top of the role summary, copy the full **ARN** — it looks like `arn:aws:iam::123456789012:role/BreakGlass-ElevatedAccessRole`. Save this to a notepad.
3. From that ARN, note your 12-digit **Account ID** (the number between `::` and `:role`). You'll need this below.
4. Note the **region** you're building in (e.g. `us-east-1`) — you picked this at the start of Phase 1.
5. Now go back to **IAM → Roles** and click into `BreakGlass-LambdaExecutionRole`.
6. Click the **Permissions** tab.
7. Click **Add permissions → Create inline policy**.
8. Click the **JSON** tab, clear the box, and paste the policy below. **Before saving, replace `YOUR_ACCOUNT_ID` with your real 12-digit account ID and `YOUR_REGION` with your region (e.g. `us-east-1`) everywhere they appear:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAssumeBreakGlassRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::YOUR_ACCOUNT_ID:role/BreakGlass-ElevatedAccessRole"
    },
    {
      "Sid": "AllowDynamoDBGrantsTable",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:UpdateItem",
        "dynamodb:Query",
        "dynamodb:Scan"
      ],
      "Resource": "arn:aws:dynamodb:YOUR_REGION:YOUR_ACCOUNT_ID:table/breakglass-grants"
    },
    {
      "Sid": "AllowSNSPublish",
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "*"
    },
    {
      "Sid": "AllowCloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:YOUR_REGION:YOUR_ACCOUNT_ID:log-group:/aws/lambda/*"
    }
  ]
}
```

   **What this means in plain English:** "(1) You may assume only the break-glass role — no other role in the account. (2) You may read and write to only the `breakglass-grants` DynamoDB table — not any other table. (3) You may publish SNS notifications. (4) You may write your own Lambda execution logs to CloudWatch. That is everything — nothing else."

9. Click **Next**.
10. Under **Policy name**, type: `BreakGlass-LambdaExecutionPolicy`
11. Click the orange **Create policy** button.

✅ **Checkpoint:** Click into `BreakGlass-LambdaExecutionRole` → **Permissions** tab. You should see `BreakGlass-LambdaExecutionPolicy` listed. Click the expand arrow next to it and verify the four permission blocks (STS, DynamoDB, SNS, CloudWatch Logs) are all present. Critically — confirm the ARNs show your actual account ID, not the placeholder text `YOUR_ACCOUNT_ID`. If placeholders are still there, click the policy name → **Edit** and fix them now before moving on.

---

### What we've built so far
- A break-glass IAM role that has no standing access, is permanently capped by the Phase 1 permission boundary, and can only be assumed by Lambda
- A Lambda execution role locked to the exact minimum: assume the break-glass role, read/write DynamoDB, publish SNS, write logs
- The two-role architecture is in place — one role grants elevated access, the other does the granting

---

## Phase 3 of 5: Lambda Functions

> **Scope note:** We're writing three Lambda functions in this phase. The Request Handler accepts an access request and writes it to DynamoDB. The Approval Handler fires when the approver clicks the email link and issues the STS credentials. The Auto-Revoke Handler fires when the timer expires and marks the grant as expired. All three run under the `BreakGlass-LambdaExecutionRole` from Phase 2.

> **Before you start:** The SNS topic ARN won't exist until Phase 4. You'll see a placeholder in the code below — leave it as-is for now. We'll come back and fill it in once the SNS topic is created.

---

## Step 1: Create the Request Handler Lambda

This is the entry point. It accepts a JSON payload describing the access request, writes a "pending" record to DynamoDB, and publishes an approval request notification to SNS. Later in Phase 4 we'll wire it to API Gateway and Step Functions.

1. In the search bar, type **Lambda** and click it.
2. Click the orange **Create function** button.
3. Select **Author from scratch**.
4. Under **Function name**, type exactly: `BreakGlass-RequestHandler`
5. Under **Runtime**, select **Python 3.12** from the dropdown.
6. Under **Architecture**, leave it as **x86_64**.
7. Expand the **Change default execution role** section.
8. Select **Use an existing role**.
9. In the role dropdown, find and select `BreakGlass-LambdaExecutionRole`.
10. Click the orange **Create function** button.
11. You're now on the function detail page. In the **Code** tab, you'll see a code editor with a file called `lambda_function.py`. Click inside the editor, press **Cmd+A** (Mac) or **Ctrl+A** (Windows) to select all the existing code, and delete it.
12. Copy the entire block below and paste it into the now-empty editor:

```python
import json
import boto3
import uuid
from datetime import datetime, timezone

dynamodb = boto3.resource('dynamodb')
sns = boto3.client('sns')

TABLE_NAME = 'breakglass-grants'
SNS_TOPIC_ARN = 'REPLACE_WITH_YOUR_SNS_TOPIC_ARN'  # Fill this in during Phase 4

def lambda_handler(event, context):
    body = event if isinstance(event, dict) else json.loads(event.get('body', '{}'))

    requester = body.get('requester')
    instance_id = body.get('instance_id')
    reason = body.get('reason')
    duration_minutes = int(body.get('duration_minutes', 60))

    if not all([requester, instance_id, reason]):
        return {
            'statusCode': 400,
            'body': json.dumps({'error': 'Missing required fields: requester, instance_id, reason'})
        }

    grant_id = str(uuid.uuid4())
    requested_at = datetime.now(timezone.utc).isoformat()

    table = dynamodb.Table(TABLE_NAME)
    table.put_item(Item={
        'grant_id': grant_id,
        'requester': requester,
        'instance_id': instance_id,
        'reason': reason,
        'duration_minutes': duration_minutes,
        'status': 'pending',
        'requested_at': requested_at,
        'approver': 'none',
        'approved_at': 'none',
        'expires_at': 'none'
    })

    approval_message = (
        f"Break-Glass Access Request\n\n"
        f"Requester: {requester}\n"
        f"Instance: {instance_id}\n"
        f"Reason: {reason}\n"
        f"Duration: {duration_minutes} minutes\n"
        f"Grant ID: {grant_id}\n\n"
        f"Approval and denial links will be added in Phase 4 once API Gateway is set up."
    )

    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject='ACTION REQUIRED: Break-Glass Access Request',
        Message=approval_message
    )

    return {
        'statusCode': 200,
        'body': json.dumps({'grant_id': grant_id, 'status': 'pending'})
    }
```

13. Click the orange **Deploy** button (near the top of the code editor). Wait for the green "Changes deployed" banner before moving on.

✅ **Checkpoint:** Click the **Test** tab. Click **Create new test event**. Under **Event name**, type `TestRequest`. Under **Event JSON**, clear the existing content and paste:
```json
{
  "requester": "engineer@example.com",
  "instance_id": "i-0123456789abcdef0",
  "reason": "Production instance unresponsive, need to investigate",
  "duration_minutes": 30
}
```
Click **Save**, then click the orange **Test** button. The function will fail with an SNS error (the ARN is still a placeholder — that's expected). The important thing to check: go to **DynamoDB → Tables → breakglass-grants → Explore table items**. You should see a new item with `status: pending` and the fields you just sent. If that item is there, the DynamoDB write worked — the function is doing its job up to the SNS step.

---

## Step 2: Create the Approval Handler Lambda

This function fires when the approver clicks the approval link in their email. It looks up the grant in DynamoDB, blocks self-approval, calls `sts:AssumeRole` to generate temporary credentials, updates the DynamoDB record to "active", and emails the credentials via SNS.

1. Go back to the Lambda console main page and click **Create function** again.
2. Select **Author from scratch**.
3. Under **Function name**, type exactly: `BreakGlass-ApprovalHandler`
4. Under **Runtime**, select **Python 3.12**.
5. Expand **Change default execution role**, select **Use an existing role**, and choose `BreakGlass-LambdaExecutionRole`.
6. Click the orange **Create function** button.
7. In the code editor, select all existing code and delete it. Paste the following:

```python
import json
import boto3
from datetime import datetime, timezone

dynamodb = boto3.resource('dynamodb')
sts = boto3.client('sts')
sns = boto3.client('sns')

TABLE_NAME = 'breakglass-grants'
BREAK_GLASS_ROLE_ARN = 'REPLACE_WITH_BREAK_GLASS_ROLE_ARN'  # BreakGlass-ElevatedAccessRole ARN from Phase 2
SNS_TOPIC_ARN = 'REPLACE_WITH_YOUR_SNS_TOPIC_ARN'           # Fill this in during Phase 4

def lambda_handler(event, context):
    params = event.get('queryStringParameters') or event
    grant_id = params.get('grant_id')
    approver = params.get('approver')

    if not grant_id or not approver:
        return {'statusCode': 400, 'body': json.dumps({'error': 'Missing grant_id or approver'})}

    table = dynamodb.Table(TABLE_NAME)
    response = table.get_item(Key={'grant_id': grant_id})
    item = response.get('Item')

    if not item:
        return {'statusCode': 404, 'body': json.dumps({'error': 'Grant not found'})}

    if item['status'] != 'pending':
        return {'statusCode': 409, 'body': json.dumps({'error': f"Grant is already {item['status']}"})}

    if item['requester'] == approver:
        return {'statusCode': 403, 'body': json.dumps({'error': 'Requester cannot approve their own request'})}

    duration_minutes = int(item.get('duration_minutes', 60))
    duration_seconds = min(duration_minutes * 60, 43200)  # STS max is 12 hours

    sts_response = sts.assume_role(
        RoleArn=BREAK_GLASS_ROLE_ARN,
        RoleSessionName=f"breakglass-{grant_id[:8]}",
        DurationSeconds=duration_seconds,
        Tags=[
            {'Key': 'GrantId', 'Value': grant_id},
            {'Key': 'Requester', 'Value': item['requester']},
            {'Key': 'Approver', 'Value': approver}
        ]
    )

    credentials = sts_response['Credentials']
    expires_at = credentials['Expiration'].isoformat()
    approved_at = datetime.now(timezone.utc).isoformat()

    table.update_item(
        Key={'grant_id': grant_id},
        UpdateExpression='SET #s = :s, approver = :a, approved_at = :aa, expires_at = :ea',
        ExpressionAttributeNames={'#s': 'status'},
        ExpressionAttributeValues={
            ':s': 'active',
            ':a': approver,
            ':aa': approved_at,
            ':ea': expires_at
        }
    )

    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject='Break-Glass Access APPROVED',
        Message=(
            f"Break-glass access has been approved.\n\n"
            f"Grant ID: {grant_id}\n"
            f"Requester: {item['requester']}\n"
            f"Instance: {item['instance_id']}\n"
            f"Approver: {approver}\n"
            f"Expires at: {expires_at}\n\n"
            f"Temporary credentials (treat as a secret — share only via secure channel):\n"
            f"AWS_ACCESS_KEY_ID={credentials['AccessKeyId']}\n"
            f"AWS_SECRET_ACCESS_KEY={credentials['SecretAccessKey']}\n"
            f"AWS_SESSION_TOKEN={credentials['SessionToken']}"
        )
    )

    return {
        'statusCode': 200,
        'body': json.dumps({'message': 'Access approved', 'grant_id': grant_id, 'expires_at': expires_at})
    }
```

   **What the self-approval check does:** Line `if item['requester'] == approver` compares the email used to make the request against the email passed in the approval URL. If they match, the function returns a 403 and stops. The approver has to be a different person than the requester — this is enforced in code, not just policy.

8. You'll notice two placeholder ARNs: `REPLACE_WITH_BREAK_GLASS_ROLE_ARN` and `REPLACE_WITH_YOUR_SNS_TOPIC_ARN`. Update `BREAK_GLASS_ROLE_ARN` right now — go to **IAM → Roles → BreakGlass-ElevatedAccessRole**, copy the ARN from the top of that page, and paste it in. Leave `SNS_TOPIC_ARN` as the placeholder for now — we'll fill it in Phase 4.
9. Click **Deploy**.

✅ **Checkpoint:** Confirm the function deployed (green "Changes deployed" banner). Check the **Configuration → Permissions** tab and confirm the execution role shows `BreakGlass-LambdaExecutionRole`. You don't need to run a full test yet — the SNS placeholder will cause a failure if you do. We'll test the full approval flow in Phase 4 once everything is wired together.

---

## Step 3: Create the Auto-Revoke Lambda

This function fires at the end of the grant timer (triggered by Step Functions in Phase 4). STS credentials expire on their own when their duration runs out — this function handles the bookkeeping side: marking the DynamoDB record as "expired" and sending a revocation notification.

1. In the Lambda console, click **Create function** again.
2. Select **Author from scratch**.
3. Under **Function name**, type exactly: `BreakGlass-AutoRevoke`
4. Under **Runtime**, select **Python 3.12**.
5. Expand **Change default execution role**, select **Use an existing role**, and choose `BreakGlass-LambdaExecutionRole`.
6. Click the orange **Create function** button.
7. In the code editor, select all and delete. Paste the following:

```python
import json
import boto3
from datetime import datetime, timezone

dynamodb = boto3.resource('dynamodb')
sns = boto3.client('sns')

TABLE_NAME = 'breakglass-grants'
SNS_TOPIC_ARN = 'REPLACE_WITH_YOUR_SNS_TOPIC_ARN'  # Fill this in during Phase 4

def lambda_handler(event, context):
    grant_id = event.get('grant_id')

    if not grant_id:
        return {'statusCode': 400, 'body': json.dumps({'error': 'Missing grant_id'})}

    table = dynamodb.Table(TABLE_NAME)
    response = table.get_item(Key={'grant_id': grant_id})
    item = response.get('Item')

    if not item:
        return {'statusCode': 404, 'body': json.dumps({'error': 'Grant not found'})}

    if item['status'] == 'expired':
        return {'statusCode': 200, 'body': json.dumps({'message': 'Grant already expired — no action needed'})}

    revoked_at = datetime.now(timezone.utc).isoformat()

    table.update_item(
        Key={'grant_id': grant_id},
        UpdateExpression='SET #s = :s, revoked_at = :r',
        ExpressionAttributeNames={'#s': 'status'},
        ExpressionAttributeValues={
            ':s': 'expired',
            ':r': revoked_at
        }
    )

    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject='Break-Glass Access REVOKED',
        Message=(
            f"Break-glass access has expired and been revoked.\n\n"
            f"Grant ID: {grant_id}\n"
            f"Requester: {item.get('requester', 'unknown')}\n"
            f"Instance: {item.get('instance_id', 'unknown')}\n"
            f"Approver: {item.get('approver', 'unknown')}\n"
            f"Revoked at: {revoked_at}\n\n"
            f"The temporary STS credentials for this session have expired naturally. "
            f"No further API calls using this session token will succeed."
        )
    )

    return {
        'statusCode': 200,
        'body': json.dumps({'message': 'Grant revoked', 'grant_id': grant_id})
    }
```

8. Leave `SNS_TOPIC_ARN` as the placeholder — same as the other functions, we'll fill it in Phase 4.
9. Click **Deploy**.

✅ **Checkpoint:** In the Lambda console, you should now see three functions in the list: `BreakGlass-RequestHandler`, `BreakGlass-ApprovalHandler`, and `BreakGlass-AutoRevoke`. Click into each one, go to **Configuration → Permissions**, and confirm all three show `BreakGlass-LambdaExecutionRole` as the execution role. If any of them shows a different role, click **Edit** on that function's configuration and update it.

---

### What we've built so far
- Three Lambda functions that cover the entire access lifecycle: request → approve → revoke
- All three run under the same tightly-scoped execution role from Phase 2
- The Approval Handler enforces the requester ≠ approver rule at the code level
- SNS ARN placeholders are still in two functions — we'll fill those in Phase 4 once the topic exists

---

## Phase 4 of 5: Step Functions + SNS (Orchestration + Approval)

> **Scope note:** This phase wires everything together. We'll create the SNS topic, get the approval email link working via API Gateway, build the Step Functions state machine that manages the timer, and connect the Request Handler Lambda to start the state machine on every new request. By the end of this phase, the full end-to-end flow will be testable.

---

## Step 1: Create the SNS Topic and Subscribe Your Approver's Email

SNS is the notification hub. Approval requests, grant-issued confirmations, and revocation notices all go through this single topic.

1. In the search bar, type **SNS** and click it (Simple Notification Service).
2. On the left sidebar, click **Topics**.
3. Click the orange **Create topic** button.
4. Under **Type**, select **Standard** (not FIFO).
5. Under **Name**, type exactly: `breakglass-approvals`
6. Leave all other settings at their defaults.
7. Scroll down and click the orange **Create topic** button.
8. You're now on the topic detail page. At the top you'll see the **ARN** — it looks like `arn:aws:sns:us-east-1:123456789012:breakglass-approvals`. **Copy this ARN to a notepad right now** — you'll need it in four separate places.
9. On the left sidebar, click **Subscriptions**, then click the orange **Create subscription** button.
10. Under **Topic ARN**, your new topic ARN should already be filled in.
11. Under **Protocol**, select **Email** from the dropdown.
12. Under **Endpoint**, type the email address where approval requests should be delivered (your approver's address).
13. Click the orange **Create subscription** button.
14. Open that email inbox now. You'll receive an "AWS Notification — Subscription Confirmation" email within about 1 minute. **Click the "Confirm subscription" link in that email** — without this step, no emails will ever arrive and the whole notification side of the system will silently fail.

✅ **Checkpoint:** Go back to **SNS → Subscriptions**. The subscription should show **Status: Confirmed**. It will say "Pending confirmation" until you click the link — do not move on until it says Confirmed.

---

## Step 2: Update All Three Lambda Functions with the Real SNS ARN

Now that you have a real SNS topic ARN, update every Lambda function that still has the placeholder.

1. Go to **Lambda**, click into `BreakGlass-RequestHandler`.
2. In the code editor, find the line that reads: `SNS_TOPIC_ARN = 'REPLACE_WITH_YOUR_SNS_TOPIC_ARN'`
3. Replace the placeholder string with your real ARN — it should look like: `SNS_TOPIC_ARN = 'arn:aws:sns:us-east-1:123456789012:breakglass-approvals'`
4. Click **Deploy** and wait for the green "Changes deployed" banner.
5. Repeat steps 1–4 for `BreakGlass-ApprovalHandler`. This function has two placeholders:
   - Update `SNS_TOPIC_ARN` with the SNS topic ARN (same as above).
   - Confirm `BREAK_GLASS_ROLE_ARN` is already filled in with the `BreakGlass-ElevatedAccessRole` ARN from Phase 3. If it still shows the placeholder, update it now.
   - Click **Deploy**.
6. Repeat steps 1–4 for `BreakGlass-AutoRevoke` (SNS ARN only). Click **Deploy**.

✅ **Checkpoint:** Open each of the three Lambda functions and scan the code for any remaining `REPLACE_WITH_` strings. There should be none left. All three should have real ARNs throughout.

---

## Step 3: Create the API Gateway Endpoint for the Approval Link

The approver needs a clickable URL in their email. We'll create a REST API in API Gateway that maps to the `BreakGlass-ApprovalHandler` Lambda. When the approver clicks the link, it passes `grant_id` and `approver` as query string parameters and the Lambda fires.

1. In the search bar, type **API Gateway** and click it.
2. Click the orange **Create API** button.
3. Under **Choose an API type**, find **REST API** (not "REST API Private") and click **Build**.
4. Select **New API**.
5. Under **API name**, type exactly: `BreakGlass-ApprovalAPI`
6. Under **API endpoint type**, select **Regional**.
7. Click the orange **Create API** button.
8. You're now inside the API editor. On the left panel, click **Resources**.
9. Select the root `/` resource in the tree.
10. Click the **Create resource** button.
11. Under **Resource name**, type: `approve`
12. The **Resource path** will auto-fill as `/approve` — leave it exactly as-is.
13. Click the orange **Create resource** button.
14. With `/approve` now selected in the resource tree, click the **Create method** button.
15. Under **Method type**, select **GET** from the dropdown.
16. Under **Integration type**, select **Lambda function**.
17. Make sure **Lambda proxy integration** is checked (this passes the full request object including query params to the Lambda).
18. Under **Lambda function**, start typing `BreakGlass-ApprovalHandler` and select it from the autocomplete.
19. Click the orange **Create method** button.
20. A dialog will appear asking "Add Permission to Lambda Function?" — click **OK**. This grants API Gateway permission to invoke the Lambda.
21. Click the **Deploy API** button at the top of the page.
22. Under **Stage**, select **New stage** from the dropdown.
23. Under **Stage name**, type: `prod`
24. Click the orange **Deploy** button.
25. After deployment, you'll land on the Stage editor. At the top you'll see the **Invoke URL** — it looks like `https://abc123xyz.execute-api.us-east-1.amazonaws.com/prod`. **Copy this URL to your notepad.**

✅ **Checkpoint:** Open a browser tab and go to: `YOUR_INVOKE_URL/approve?grant_id=test&approver=test@example.com`. You should get a JSON response back — something like `{"error": "Grant not found"}` is the correct response here (there's no real grant with ID "test"). Getting any JSON back at all confirms that API Gateway successfully invoked the Lambda. A grant-not-found error from inside the Lambda is a success signal for this wiring check.

---

## Step 4: Update the Approval Email to Include the Real Approval Link

Now that we have a real API Gateway URL, we need to update the Request Handler Lambda so the email it sends includes the actual clickable link.

1. Go to **Lambda → BreakGlass-RequestHandler**.
2. In the code editor, find the `approval_message` block and replace it with the version below. **Replace `YOUR_INVOKE_URL` with your API Gateway Invoke URL from Step 3:**

```python
    approval_message = (
        f"Break-Glass Access Request\n\n"
        f"Requester: {requester}\n"
        f"Instance: {instance_id}\n"
        f"Reason: {reason}\n"
        f"Duration: {duration_minutes} minutes\n"
        f"Grant ID: {grant_id}\n\n"
        f"To APPROVE this request, click:\n"
        f"YOUR_INVOKE_URL/approve?grant_id={grant_id}&approver=YOUR_APPROVER_EMAIL\n\n"
        f"If you did not expect this request or want to deny it, simply ignore this email. "
        f"The request will expire automatically after the approval window closes."
    )
```

   Replace `YOUR_APPROVER_EMAIL` with your approver's actual email address — this is what the Approval Handler uses for the requester ≠ approver check, so it must match exactly.

3. Click **Deploy**.

✅ **Checkpoint:** Re-run the test event from Phase 3 Step 1 (go to the **Test** tab, select `TestRequest`, click **Test**). This time the function should succeed end-to-end: check the email inbox — you should receive an approval request email with a real clickable link in it. The link won't work for the test grant_id (DynamoDB item won't exist from the test event — you'd need to trigger the Lambda via its actual invoke flow), but confirming the email arrives with a properly formatted link means the SNS wiring is working.

---

## Step 5: Build the Step Functions State Machine

The state machine is the orchestration backbone. It receives the grant ID, waits out the full grant duration, then triggers the Auto-Revoke Lambda to mark the grant expired. We're keeping the state machine lean for now — it handles the timer side of things.

1. In the search bar, type **Step Functions** and click it.
2. On the left sidebar, click **State machines**.
3. Click the orange **Create state machine** button.
4. Select **Write your workflow in code** (not the drag-and-drop Workflow Studio builder).
5. Under **Type**, select **Standard** (not Express — Standard supports the long-running wait states needed for 30–60 minute grants).
6. In the definition editor, clear the existing content and paste the state machine definition below. **Replace `YOUR_REGION` and `YOUR_ACCOUNT_ID` before saving:**

```json
{
  "Comment": "Break-glass access timer: waits for the grant duration, then triggers auto-revoke.",
  "StartAt": "WaitForGrantDuration",
  "States": {
    "WaitForGrantDuration": {
      "Type": "Wait",
      "SecondsPath": "$.wait_seconds",
      "Comment": "Waits for the number of seconds specified in the input. Set wait_seconds to 120 for testing.",
      "Next": "AutoRevoke"
    },
    "AutoRevoke": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "arn:aws:lambda:YOUR_REGION:YOUR_ACCOUNT_ID:function:BreakGlass-AutoRevoke",
        "Payload": {
          "grant_id.$": "$.grant_id"
        }
      },
      "ResultPath": null,
      "End": true
    }
  }
}
```

   **What this does:** The state machine reads `wait_seconds` from its input (so the Request Handler can set the timer based on the requested duration), waits that long, then invokes the Auto-Revoke Lambda with the `grant_id`. That's the entire machine — simple and auditable.

7. Click **Next**.
8. Under **State machine name**, type exactly: `BreakGlass-StateMachine`
9. Under **Execution role**, select **Create a new role** — AWS will auto-generate a role with the permissions to invoke Lambda. Leave the auto-generated name.
10. Leave all other settings at defaults.
11. Click the orange **Create state machine** button.
12. After creation, copy the state machine **ARN** from the top of its detail page — you'll need it in the next step.

✅ **Checkpoint:** On the state machine detail page, click **Start execution**. In the input box, paste:
```json
{
  "grant_id": "test-sm-check",
  "wait_seconds": 10
}
```
Click **Start execution**. Watch the execution graph — it should move into the **WaitForGrantDuration** state. After 10 seconds it will attempt to invoke `BreakGlass-AutoRevoke` with `grant_id: test-sm-check`. The Lambda will return a 404 (no such grant in DynamoDB — that's fine). What you're confirming is that the state machine transitions correctly and successfully invokes the Lambda. Check the execution detail — the AutoRevoke state should show as **Succeeded** (even though the Lambda returned a 404 internally, the Lambda itself ran and returned a 200-level response to Step Functions, which counts as success from the orchestration perspective).

---

## Step 6: Wire the Request Handler Lambda to Start the State Machine

The Request Handler currently writes to DynamoDB and sends the SNS notification, but doesn't start the state machine yet. We need to add that step — and give the Lambda execution role permission to do it.

**Part A: Add Step Functions permission to the Lambda execution role**

1. Go to **IAM → Roles → BreakGlass-LambdaExecutionRole**.
2. Click **Add permissions → Create inline policy**.
3. Click the **JSON** tab, clear the box, and paste the following. Replace `YOUR_REGION` and `YOUR_ACCOUNT_ID`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowStartStateMachine",
      "Effect": "Allow",
      "Action": "states:StartExecution",
      "Resource": "arn:aws:states:YOUR_REGION:YOUR_ACCOUNT_ID:stateMachine:BreakGlass-StateMachine"
    }
  ]
}
```

4. Click **Next**, name it `BreakGlass-StartStateMachinePolicy`, and click **Create policy**.

**Part B: Update the Request Handler Lambda code**

1. Go to **Lambda → BreakGlass-RequestHandler**.
2. In the code editor, find the line `import uuid` at the top and add `import math` after it (we'll use it to convert minutes to whole seconds).
3. Find the `return` statement at the bottom of the function (the last few lines). Add the following block immediately **before** the `return` statement, replacing `YOUR_REGION` and `YOUR_ACCOUNT_ID`:

```python
    sfn = boto3.client('stepfunctions')
    sfn.start_execution(
        stateMachineArn='arn:aws:states:YOUR_REGION:YOUR_ACCOUNT_ID:stateMachine:BreakGlass-StateMachine',
        name=grant_id,
        input=json.dumps({
            'grant_id': grant_id,
            'wait_seconds': duration_minutes * 60
        })
    )
```

4. Click **Deploy**.

✅ **Checkpoint:** Run the `TestRequest` test event again from the **Test** tab. This time, all three things should happen: (1) a new item appears in **DynamoDB → breakglass-grants** with `status: pending`; (2) an approval request email arrives in the approver's inbox with a real clickable link; (3) a new execution appears in **Step Functions → BreakGlass-StateMachine** sitting in the WaitForGrantDuration state. If you see all three, the full request flow is working end-to-end. Click **Stop execution** on the Step Functions execution to cancel the 30-minute wait.

---

### What we've built so far
- An SNS topic with a confirmed email subscription — notifications are live
- An API Gateway endpoint the approver can click to invoke the Approval Handler
- Approval emails that include the real clickable approve link
- A Step Functions state machine that manages the grant timer and fires auto-revoke
- The Request Handler Lambda now starts the state machine on every new request, passing the correct wait duration

---

## Phase 5 of 5: Audit + Failsafe

> **Scope note:** This final phase focuses on visibility and resilience. We're verifying CloudTrail is capturing everything, enabling Security Hub for continuous compliance checks, creating a real-time EventBridge alert that fires the moment the break-glass role is assumed, and building a scheduled failsafe that force-checks DynamoDB every 5 minutes for expired grants that Step Functions may have missed.

---

## Step 1: Verify CloudTrail Is Active

CloudTrail records every API call made in your account, including every `AssumeRole` event. It's usually on by default, but we need to confirm it and make sure management events are being captured.

1. In the search bar, type **CloudTrail** and click it.
2. On the left sidebar, click **Trails**.
3. Look at the list of trails:
   - If you see at least one trail with **Logging** set to **On**, you're good — note the trail name and move to the checkpoint below.
   - If the list is empty, click **Create trail** and configure it:
     - Under **Trail name**, type: `breakglass-audit-trail`
     - Under **Storage location**, select **Create a new S3 bucket** and accept the auto-generated bucket name.
     - Under **Management events**, ensure both **Read** and **Write** are checked.
     - Click **Next**, skip through the log file validation and encryption settings (leave defaults), click **Next** again, then click **Create trail**.
4. If a trail already exists, click into it and verify that **Management events** is capturing at least **Write** events (Read + Write is better). If it shows "None", click **Edit**, enable management events, and save.

✅ **Checkpoint:** Go to **CloudTrail → Event history** in the left sidebar. You should see a list of recent events — things like `ConsoleLogin`, `GetCallerIdentity`, or other activity from your session. If the list has events in it, CloudTrail is active and working. Any `AssumeRole` call on `BreakGlass-ElevatedAccessRole` will appear here with the full caller identity, timestamp, source IP, and every API call made during that session.

---

## Step 2: Enable Security Hub

Security Hub runs automated security checks against your account and surfaces findings when things drift from best practices — for example, if someone accidentally removes the permission boundary from the break-glass role, or leaves IAM access keys unused for 90 days. We want it on.

1. In the search bar, type **Security Hub** and click it.
2. If you see a **Go to Security Hub** button, Security Hub is already enabled — click it and skip to the checkpoint.
3. If you see an **Enable Security Hub** button:
   - Click it.
   - On the enable screen, confirm **AWS Foundational Security Best Practices** is checked. Leave the other standards (CIS AWS Foundations, PCI DSS) checked too if they appear — they add breadth without any extra cost.
   - Click the orange **Enable Security Hub** button.
4. After enabling, you'll land on the Security Hub **Summary** page. It takes 15–30 minutes to populate the first set of findings.
5. On the left sidebar, click **Security standards**. Confirm **AWS Foundational Security Best Practices** shows a green **Enabled** label.

✅ **Checkpoint:** On the Security Hub **Summary** page, confirm the service status shows "Enabled" at the top of the page. You may see a score of 0% or "No findings yet" — that's normal while the first scan runs. Come back in 30 minutes if you want to see actual results, but for now just confirming it's enabled is the goal.

---

## Step 3: Create the EventBridge Real-Time AssumeRole Alert Rule

This is your tripwire. The moment anyone assumes `BreakGlass-ElevatedAccessRole` — whether through the proper approval flow or not — this rule fires and delivers an SNS alert to the approver's inbox. It's a real-time signal that the break-glass role is in active use.

1. In the search bar, type **EventBridge** and click it (Amazon EventBridge).
2. On the left sidebar, click **Rules**.
3. Confirm the **Event bus** dropdown at the top shows **default** — leave it there.
4. Click the orange **Create rule** button.
5. Under **Name**, type exactly: `BreakGlass-AssumeRoleAlert`
6. Under **Description**, type: `Fires whenever BreakGlass-ElevatedAccessRole is assumed. Sends SNS alert in real time.`
7. Under **Event bus**, leave it as **default**.
8. Under **Rule type**, select **Rule with an event pattern**.
9. Click **Next**.
10. On the **Build event pattern** page:
    - Under **Event source**, select **AWS events or EventBridge partner events**.
    - Under **Method**, select **Custom pattern (JSON editor)**.
    - Clear the existing JSON and paste the event pattern below. **Replace `YOUR_ACCOUNT_ID` with your real account ID:**

```json
{
  "source": ["aws.sts"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["sts.amazonaws.com"],
    "eventName": ["AssumeRole"],
    "requestParameters": {
      "roleArn": ["arn:aws:iam::YOUR_ACCOUNT_ID:role/BreakGlass-ElevatedAccessRole"]
    }
  }
}
```

    **What this means:** "Watch the stream of CloudTrail events. When you see an `AssumeRole` API call where the target role is specifically `BreakGlass-ElevatedAccessRole`, fire this rule." It doesn't matter who called it or how — any assumption triggers the alert.

11. Click **Next**.
12. On the **Select targets** page:
    - Under **Target types**, select **AWS service**.
    - Under **Select a target**, choose **SNS topic** from the dropdown.
    - Under **Topic**, select `breakglass-approvals` from the list.
13. Click **Next**, skip the optional tags page, and click the orange **Create rule** button.

✅ **Checkpoint:** On the EventBridge **Rules** page, you should see `BreakGlass-AssumeRoleAlert` listed with **Status: Enabled**. To test it end-to-end: fire a real request through the Request Handler Lambda, then click the approval link in the email. If the Approval Handler runs successfully, you should receive two separate emails: one from the Approval Handler itself (the "Access APPROVED" message with credentials), and a second one a minute or two later triggered by this EventBridge rule (its content will be the raw CloudTrail event JSON). Getting both emails confirms the tripwire is working.

---

## Step 4: Create the EventBridge Scheduled Failsafe

Step Functions handles normal grant expiry. But if Step Functions itself has an issue, grants could stay "active" in DynamoDB indefinitely even after the STS credentials have expired. This rule runs every 5 minutes, scans DynamoDB for any grant whose `expires_at` is in the past but whose `status` is still "active", and force-revokes them.

This requires a fourth Lambda function first, then the EventBridge schedule.

**Part A: Create the Failsafe Scanner Lambda**

1. Go to **Lambda** and click **Create function**.
2. Select **Author from scratch**.
3. Under **Function name**, type exactly: `BreakGlass-FailsafeScanner`
4. Under **Runtime**, select **Python 3.12**.
5. Expand **Change default execution role**, select **Use an existing role**, and choose `BreakGlass-LambdaExecutionRole`.
6. Click the orange **Create function** button.
7. In the code editor, select all and delete. Paste the following. **Replace `REPLACE_WITH_YOUR_SNS_TOPIC_ARN` with your real SNS topic ARN:**

```python
import json
import boto3
from datetime import datetime, timezone

dynamodb = boto3.resource('dynamodb')
sns = boto3.client('sns')
lambda_client = boto3.client('lambda')

TABLE_NAME = 'breakglass-grants'
SNS_TOPIC_ARN = 'REPLACE_WITH_YOUR_SNS_TOPIC_ARN'
AUTO_REVOKE_FUNCTION_NAME = 'BreakGlass-AutoRevoke'

def lambda_handler(event, context):
    table = dynamodb.Table(TABLE_NAME)
    now = datetime.now(timezone.utc).isoformat()

    response = table.scan(
        FilterExpression='#s = :active AND expires_at < :now',
        ExpressionAttributeNames={'#s': 'status'},
        ExpressionAttributeValues={':active': 'active', ':now': now}
    )

    expired_grants = response.get('Items', [])
    revoked_count = 0

    for grant in expired_grants:
        grant_id = grant['grant_id']
        lambda_client.invoke(
            FunctionName=AUTO_REVOKE_FUNCTION_NAME,
            InvocationType='Event',
            Payload=json.dumps({'grant_id': grant_id})
        )
        revoked_count += 1

    if revoked_count > 0:
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject=f'Failsafe: {revoked_count} expired grant(s) force-revoked',
            Message=(
                f"The failsafe scanner found {revoked_count} grant(s) past their expiry "
                f"time but still marked active in DynamoDB. Auto-Revoke has been triggered for each.\n\n"
                f"Grant IDs: {', '.join(g['grant_id'] for g in expired_grants)}"
            )
        )

    return {
        'statusCode': 200,
        'body': json.dumps({'scanned': True, 'revoked_count': revoked_count})
    }
```

8. Click **Deploy**.

**Part B: Grant the Lambda execution role permission to invoke other Lambdas**

The Failsafe Scanner needs to call `BreakGlass-AutoRevoke` directly. The execution role doesn't have that permission yet.

1. Go to **IAM → Roles → BreakGlass-LambdaExecutionRole**.
2. Click **Add permissions → Create inline policy**.
3. Click the **JSON** tab, clear the box, and paste the following. Replace `YOUR_REGION` and `YOUR_ACCOUNT_ID`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowInvokeAutoRevoke",
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:YOUR_REGION:YOUR_ACCOUNT_ID:function:BreakGlass-AutoRevoke"
    }
  ]
}
```

4. Click **Next**, name it `BreakGlass-InvokeLambdaPolicy`, and click **Create policy**.

**Part C: Create the EventBridge Scheduled Rule**

1. Go back to **EventBridge → Rules**.
2. Click the orange **Create rule** button.
3. Under **Name**, type exactly: `BreakGlass-FailsafeSchedule`
4. Under **Description**, type: `Runs every 5 minutes. Scans DynamoDB for expired-but-still-active grants and force-revokes them.`
5. Under **Rule type**, select **Schedule**.
6. Click **Next**.
7. On the **Define schedule** page, select **A schedule that runs at a regular rate, such as every 10 minutes**.
8. Under **Rate expression**, set the value to `5` and the unit dropdown to **minutes**.
9. Click **Next**.
10. On the **Select targets** page:
    - Under **Target types**, select **AWS service**.
    - Under **Select a target**, choose **Lambda function** from the dropdown.
    - Under **Function**, select `BreakGlass-FailsafeScanner` from the list.
11. Click **Next**, skip the optional tags, and click the orange **Create rule** button.

✅ **Checkpoint:** Go to **EventBridge → Rules**. You should now see two rules: `BreakGlass-AssumeRoleAlert` (event-driven, status Enabled) and `BreakGlass-FailsafeSchedule` (rate-based, status Enabled). To confirm the scanner is wired correctly: go to **Lambda → BreakGlass-FailsafeScanner**, click the **Test** tab, create a test event with input `{}`, and run it. You should get back `{"scanned": true, "revoked_count": 0}`. That's correct — it ran, found no expired-but-active grants, and returned cleanly. The first time it finds anything to revoke, it will trigger Auto-Revoke automatically and send an SNS alert.

---

### What we've built so far
- CloudTrail confirmed active — immutable log of every API call in the account
- Security Hub enabled — continuous compliance monitoring against AWS security best practices
- A real-time EventBridge tripwire that fires the moment `BreakGlass-ElevatedAccessRole` is assumed
- A scheduled failsafe that runs every 5 minutes and force-revokes any zombie grants
- A fourth Lambda (`BreakGlass-FailsafeScanner`) that does the DynamoDB scan and delegates cleanup

---

## The Full System Is Now Built

Here's a complete summary of everything in place:

| Component | What you built |
|---|---|
| **DynamoDB table** | `breakglass-grants` — the logbook for every grant, past and present |
| **Permission boundary** | `BreakGlass-PermissionBoundary` — the hard ceiling on the break-glass role |
| **Break-glass IAM role** | `BreakGlass-ElevatedAccessRole` — assumed by Lambda only, capped by the boundary |
| **Lambda execution role** | `BreakGlass-LambdaExecutionRole` — tightly scoped, the highest-value IAM identity in the system |
| **Lambda: Request Handler** | Accepts requests, writes pending record to DynamoDB, emails approver, starts state machine |
| **Lambda: Approval Handler** | Validates approval, enforces requester ≠ approver, issues STS credentials, updates DynamoDB |
| **Lambda: Auto-Revoke** | Marks grants expired in DynamoDB, sends revocation notification |
| **Lambda: Failsafe Scanner** | Scans for zombie grants every 5 minutes, delegates cleanup to Auto-Revoke |
| **SNS topic** | `breakglass-approvals` — single notification channel for all system events |
| **API Gateway** | `BreakGlass-ApprovalAPI` — the clickable approval link embedded in request emails |
| **Step Functions** | `BreakGlass-StateMachine` — manages the grant duration timer and triggers auto-revoke |
| **EventBridge (alert)** | `BreakGlass-AssumeRoleAlert` — real-time tripwire on every break-glass role assumption |
| **EventBridge (failsafe)** | `BreakGlass-FailsafeSchedule` — every 5 minutes, cleans up any expired-but-active grants |
| **CloudTrail** | Immutable audit log of all management events and API calls |
| **Security Hub** | Continuous compliance monitoring against AWS Foundational Security Best Practices |

---

### Optional Extensions (for later, once the core system is stable)

1. **Swap email approval for Slack interactive buttons.** Replace the SNS email notification in the Request Handler with a Slack message containing Approve/Deny buttons, built using the Slack Bolt SDK running on Lambda + API Gateway. The Approval Handler logic is unchanged — only the delivery and trigger mechanism changes.
2. **Add a second access scenario (e.g. scoped S3 access).** Create a second permission boundary and break-glass role scoped to a specific S3 bucket or prefix. The orchestration layer — Step Functions, Lambda, DynamoDB, SNS — is entirely reusable. Add a `scenario` field to the request payload and route to the appropriate role at assume-time.
3. **Build a CloudWatch dashboard for the audit trail.** Create a CloudWatch dashboard that surfaces DynamoDB metrics (total grants issued, currently active, expired) and CloudTrail Insights (AssumeRole frequency, unusual activity patterns). This makes the system's activity visible at a glance for security reviews and incident retrospectives.
