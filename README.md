# Scenario: codebuild_secrets_exfil

**Size:** Small

**Difficulty:** Easy

**Command:** `$ ./cloudgoat.py create codebuild_secrets_exfil`

## Scenario Resources

- 2 IAM Users
- 1 IAM Role (CodeBuild service role)
- 1 IAM Policy (attached to service role)
- 1 CodeBuild Project
- 1 Secrets Manager Secret
- 1 S3 Bucket (CodeBuild artifact store)

## Scenario Start(s)

1. IAM User "Bob" — access key and secret key provided in `start.txt`

## Scenario Goal(s)

Retrieve the value of the secret stored in AWS Secrets Manager by abusing CodeBuild's inline buildspec override and the project's privileged service role.

## Summary

Starting as the low-privileged IAM user Bob, the attacker discovers they hold just enough permissions to list and start CodeBuild projects. The existing CodeBuild project runs with a service role that has broad Secrets Manager access — a permission Bob himself does not have.

By starting a new build and overriding the buildspec with inline commands, the attacker hijacks the build execution context. The injected commands run as the CodeBuild service role, which can freely call `secretsmanager:GetSecretValue`. The secret value is then exfiltrated by POSTing it to an attacker-controlled HTTP listener, completing the privilege escalation.

## Exploitation Route(s)

![Description of image](./SCR-20260704-pmuk.png)

## Route Walkthrough — IAM User "Bob"

1. As the IAM user Bob, the attacker begins by enumerating available permissions. Direct calls to Secrets Manager or IAM are denied.

2. The attacker lists CodeBuild projects using `codebuild:ListProjects` and discovers a project named `cg-vulnerable-project-<cgid>`.

3. Inspecting the project reveals its service role ARN. The attacker cannot assume the role directly, but can trigger it indirectly by starting a build.

4. The attacker starts a new build on the discovered project using `codebuild:StartBuild`. Crucially, they supply a `buildspecOverride` parameter containing inline shell commands instead of the repository's original buildspec.

5. The injected buildspec first queries Secrets Manager for the target secret using the build environment's inherited role credentials, then POSTs the secret value as JSON to an attacker-controlled HTTP listener (e.g. a `python3 -m http.server` or a `requestbin` endpoint).

6. The CodeBuild project assumes its service role for the duration of the build. Because the service role has `secretsmanager:GetSecretValue` on `*`, the injected command succeeds.

7. The attacker observes the secret value arrive at their listener, completing the scenario.

A cheat sheet for this route is available [here](./cheat_sheet.md).
