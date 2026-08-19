### Part 1: Non-Root Deployment User

**Environment**
- Provider: AWS EC2
- OS: Ubuntu (Ubuntu 26.04 LTS)
- Administrative user: `ubuntu` (default AMI user, sudo-capable, retained as break-glass access)

**Actions performed**
1. Verified session identity (`whoami`).
2. Created dedicated deployment account with password login disabled:
   `sudo adduser --disabled-password --gecos "" deploy`
3. Verified account state:
   - `id deploy` — UID/GID assigned; user is NOT a member of the `sudo` group (least privilege).
   - `getent passwd deploy` — home `/home/deploy`, shell `/bin/bash` (required for SSH-based deploys).
   - `sudo getent shadow deploy` — password field locked (`!`), confirming key-based authentication only.
   - `ls -la /home/deploy` — home directory correctly owned and provisioned.

**Rationale**
- Principle of Least Privilege: deployments and application runtime are isolated from root/administrative identity, limiting blast radius on compromise or operator error.
- Separation of duties: human admin (`ubuntu`) vs. deployment identity (`deploy`).
- Default-deny posture: account has no credentials and cannot log in until SSH keys are explicitly provisioned (next step).
