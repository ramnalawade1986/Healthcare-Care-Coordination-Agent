# Healthcare Care Coordinator — Bedrock Agent + Step Functions (Demo)

> **Status:** Production-ready demo. Includes human-in-the-loop approvals (Discord), field-level KMS encryption, DynamoDB memory, SES email, and a live Flask dashboard.

## ✨ What this does

- Ingests patient JSON to S3 and kicks off a **Step Functions** workflow
- Uses a **Bedrock Agent (Claude Sonnet)** and deterministic **Lambda** tools to:
  - parse a report, verify basics, pull a guideline snippet
  - propose a care plan + follow-up
  - request **Discord** approval (with Approve/Reject links)
  - on approval, email the patient via **SES**
  - persist a longitudinal **Agent Memory** record in **DynamoDB**
- Demonstrates **privacy by design**:
  - S3 bucket encryption
  - DynamoDB **SSE-KMS**
  - **Field‑level KMS encryption** (e.g., `patient_email_enc`, `actions_enc`) with Encryption Context
- A **Flask dashboard** shows the live execution timeline, state I/O, Discord run logs, memory items, and a security panel

---

## 🏗 Architecture (high level)

1. **Flask Demo UI** uploads patient JSON → S3 (`raw_data/…`)
2. **EventBridge** triggers **DataProcessor Lambda**
3. **Step Functions** orchestrates Planner → Parser → FHIR Verify → Guideline → Propose
4. If high risk, **DiscordApproval Lambda** posts an approval card and **waits for task token**
5. Human clicks **Approve/Reject** → API Gateway → **ApprovalCallback Lambda** → Step Functions resumes
6. **ExecuteAction Lambda** continues workflow
7. **AppointmentMailer Lambda** sends SES email and mirrors a run log line to Discord; writes to **RunsTable**
8. **WriteMemory Lambda** persists latest memory in **AgentMemory**, storing **KMS‑encrypted** sensitive fields
9. **SummarizeRun Lambda** writes a compact run summary

---

## 📦 Repo layout (suggested)

```
infra/
  stack.yaml                 # CloudFormation (full stack)
lambdas/
  planner/
  report_parser/
  fhir_tool/
  guideline_search/
  propose_action/
  discord_approval/
  approval_callback/
  execute_action/
  appointment_mailer/
  write_memory/
  summarize_run/
  data_generator/
  data_processor/
  patient_gateway/
app/
  app.py                     # Flask dashboard (updated)
README.md
```

---

## 🚀 Deploy (one‑shot)

Prereqs: AWS CLI, Python 3.11, verified SES sender, a Discord webhook, and (optional but recommended) a **KMS CMK**.

```bash
aws cloudformation deploy \
  --template-file infra/stack.yaml \
  --stack-name HealthCareStack \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    Environment=dev \
    NameSuffix=red \
    SesFromEmail=<verified@sender> \
    DiscordWebhookUrl=<discord_webhook_url> \
  --region us-west-2
```

> **Note:** If you omit `DiscordWebhookUrl`, the Lambdas read the webhook from `Secrets Manager` (`DiscordWebhookSecret`).

---

## 🖥 Run the dashboard

```bash
cd app
pip install -r requirements.txt
export AWS_REGION=us-west-2
export STACK_NAME=HealthCareStack
export DISCORD_WEBHOOK_URL=<discord_webhook_url>
export KMS_KEY_ARN=<arn:aws:kms:...:key/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx>   # optional but recommended
python app.py
```

Open http://localhost:8000

---

## 🔐 Security notes

- **S3**: Bucket public access blocked + default SSE (SSE-S3 or SSE-KMS)
- **DynamoDB**: SSE-KMS enabled on both tables (Memory & Runs)
- **Field‑level KMS** (in `WriteMemoryFunction`):
  - Encrypts `patient_email` and `recommended_actions` into `{{"alg": "KMS", "ciphertext_b64": "…"}}`
  - Uses **Encryption Context**: `{{"table": "<MemoryTableName>", "patient_id": "<id>", "topic": "<caregap|…>"}}`
  - The Flask `/memory/decrypted` route demonstrates live decrypt via KMS

**KMS permissions** (principal = role that runs the dashboard / decrypts):
- `kms:Encrypt`, `kms:Decrypt`, `kms:GenerateDataKey*`, `kms:DescribeKey` on your CMK
- Add a Key Policy statement with `Principal`: that role’s ARN (or your user for local dev)

---

## 🧪 Sample patient JSON

```json
{
  "patient_id": "demo-123",
  "patient_name": "Hackathon Demo",
  "patient_email": "you@example.com",
  "last_lab_result": { "test_name": "HbA1c", "result_value": 12.2, "is_critical": true },
  "preferred_doctor": "Dr. A. Endo",
  "preferred_department": "Endocrinology"
}
```

---

## 🧭 Demo steps (3 minutes)

1. **Upload & Run**: pick the JSON above → see execution ARN & timeline stream
2. **Detailed Steps**: inspect inputs/outputs (risk = `high`, follow‑up suggestion)
3. **Discord Approval**: approve the card; workflow resumes
4. **Email & Logs**: view Discord run log & raw RunsTable
5. **Memory & KMS**: open **Memory (encrypted fields)** to see ciphertext + live decrypted values
6. **Security**: show S3/DDB SSE and KMS key in the Security panel

---

## 🛠 Troubleshooting

- **Email not sent**: SES sender must be **verified** in region; in Sandbox, recipient must be verified too
- **Discord 400**: Webhook URL invalid or bot permissions changed
- **`no patient_email`**: Ensure top‑level `patient_email` exists; dashboard normalizes `email` to `patient_email`
- **KMS AccessDenied**: Add your execution role/user to the **key policy** and IAM perms for `Encrypt/Decrypt`

---

## 📄 License

Apache-2.0 (example).
