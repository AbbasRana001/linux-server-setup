# Part 1: Non-Root Deployment User

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

# Part 2: SSH Key Setup & User Configuration Guide

## Phase 1: Create the SSH Key Pair

Run these commands on your local machine (linux/WSL).

1. **Generate the key pair.** Using the modern, secure `ed25519` algorithm. The `-C` flag adds a human-readable label.

   ```bash
   ssh-keygen -t ed25519 -C "deploy-key"
   ```

   - **Prompt 1 (Location):** Press Enter to save to the default location (`~/.ssh/id_ed25519`).
   - **Prompt 2 (Passphrase):** Type a secure passphrase and press Enter (you will not see the characters as you type).

2. **View and copy the public key.**

   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```

   - Highlight and copy the entire output line (it starts with `ssh-ed25519` and ends with `deploy-key`).

## Phase 2: Prepare the Server Environment

Run these commands on your AWS EC2 server while logged in as the default `ubuntu` user.

1. **Create the `.ssh` directory for the new user.**

   ```bash
   sudo mkdir /home/deploy/.ssh
   ```

2. **Assign ownership to the deploy user.**

   ```bash
   sudo chown deploy:deploy /home/deploy/.ssh
   ```

3. **Restrict directory permissions.** Mode `700` ensures only the owner can read, write, or enter the directory.

   ```bash
   sudo chmod 700 /home/deploy/.ssh
   ```

## Phase 3: Install the Public Key

Run these commands on your AWS EC2 server as the `ubuntu` user.

1. **Open the `authorized_keys` file in a text editor.**

   ```bash
   sudo nano /home/deploy/.ssh/authorized_keys
   ```

2. **Paste your key.** Right-click to paste the public key you copied in Phase 1. Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

3. **Assign ownership of the file to the deploy user.**

   ```bash
   sudo chown deploy:deploy /home/deploy/.ssh/authorized_keys
   ```

4. **Restrict file permissions.** Mode `600` ensures only the owner can read and write to this file.

   ```bash
   sudo chmod 600 /home/deploy/.ssh/authorized_keys
   ```

## Phase 4: Test the Connection

Run this command on your local machine (WSL).

1. **Connect as the deploy user.**

   ```bash
   ssh -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes deploy@<YOUR-EC2-PUBLIC-IP>
   ```

2. **Verify identity (first time only).** When asked if you want to continue connecting, type `yes`. This saves the server's fingerprint to your `known_hosts` file.

3. **Verify success.** Once logged in, run `whoami` to confirm you are the `deploy` user, then type `exit` to return to your local machine.

## Phase 5: Create Local SSH Shortcuts

Run these commands on your local machine (WSL) to avoid typing long connection strings.

1. **Open your local SSH config file.**

   ```bash
   nano ~/.ssh/config
   ```

2. **Add your server aliases.** Paste the following configuration, replacing `<YOUR-EC2-PUBLIC-IP>` with your actual server IP address:

   ```
   Host ec2-deploy
       HostName <YOUR-EC2-PUBLIC-IP>
       User deploy
       IdentityFile ~/.ssh/id_ed25519
       IdentitiesOnly yes
   ```

3. **Save and exit** (`Ctrl+O`, `Enter`, `Ctrl+X`).

4. **Secure the config file.**

   ```bash
   chmod 600 ~/.ssh/config
   ```

## Usage

You can now connect to your server simply by typing `ssh ec2-admin` or `ssh ec2-deploy` in your WSL terminal.
