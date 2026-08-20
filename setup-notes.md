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

# Part 3: SSH Server Hardening — `sshd_config`

## Goal

Harden the SSH server by:

* Disabling password-based authentication
* Disabling root SSH login
* Limiting authentication attempts
* Allowing SSH access only for approved users

> **Safety:** Keep the current SSH terminal open until all fresh-login tests succeed.

---

## Step 1 — Back Up the Existing Configuration

```bash
sudo cp /etc/ssh/sshd_config /root/sshd_config.backup
```

Creates a backup of the original SSH configuration before making changes.

* `cp SOURCE DESTINATION` = copy a file
* `/root/sshd_config.backup` = backup location

**Expected:** No output.

---

## Step 2 — Check That SSH Supports Drop-in Configuration

```bash
head -n 3 /etc/ssh/sshd_config
```

Look for:

```text
Include /etc/ssh/sshd_config.d/*.conf
```

This tells SSH to also read `.conf` files from:

```text
/etc/ssh/sshd_config.d/
```

If the `Include` line is not present, **stop here before continuing**.

---

## Step 3 — Make Sure the Drop-in Directory Exists

```bash
sudo mkdir -p /etc/ssh/sshd_config.d
```
---

## Step 4 — Create the SSH Hardening Configuration

Open a new configuration file:

```bash
sudo nano /etc/ssh/sshd_config.d/99-hardening.conf
```

Add:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
MaxAuthTries 3
AllowUsers ubuntu deploy
```

### What these settings do

| Setting                           | Purpose                                                      |
| --------------------------------- | ------------------------------------------------------------ |
| `PermitRootLogin no`              | Prevents direct SSH login as `root`                          |
| `PasswordAuthentication no`       | Disables password-based SSH authentication                   |
| `KbdInteractiveAuthentication no` | Disables keyboard-interactive authentication                 |
| `MaxAuthTries 3`                  | Allows a maximum of 3 authentication attempts per connection |
| `AllowUsers ubuntu deploy`        | Only allows `ubuntu` and `deploy` to SSH                     |

> **Important:** `AllowUsers` does not create these users. They must already exist.

Save in nano:

```text
Ctrl + O
Enter
Ctrl + X
```

The new file should now be:

```text
/etc/ssh/sshd_config.d/99-hardening.conf
```

---

## Step 5 — Validate the Configuration

Before applying anything:

```bash
sudo sshd -t
```

`-t` tests the SSH configuration for errors.

**Expected:** No output.

If an error appears, **do not reload SSH**. Fix the configuration and run the command again.

---

## Step 6 — Verify the Effective Configuration

Check what SSH will actually use.

### Root login

```bash
sudo sshd -T | grep permitrootlogin
```

Expected:

```text
permitrootlogin no
```

### Password authentication

```bash
sudo sshd -T | grep passwordauthentication
```

Expected:

```text
passwordauthentication no
```

### Keyboard-interactive authentication

```bash
sudo sshd -T | grep kbdinteractiveauthentication
```

Expected:

```text
kbdinteractiveauthentication no
```

### Authentication attempts

```bash
sudo sshd -T | grep maxauthtries
```

Expected:

```text
maxauthtries 3
```

### Allowed users

```bash
sudo sshd -T | grep allowusers
```

Expected:

```text
allowusers ubuntu deploy
```

### Why `sshd -T`?

```bash
sudo sshd -T
```

prints the **effective SSH configuration** after SSH processes its configuration files.

This lets us verify that our intended values are actually being used.

---

## Step 7 — Reload SSH

Once `sshd -t` succeeds and all five settings show the expected values:

```bash
sudo systemctl reload ssh
```

`reload` tells the running SSH service to reread its configuration without intentionally terminating existing SSH sessions.

> Prefer `reload` over `restart` for configuration changes.

---

## Step 8 — Test Fresh SSH Connections

**Keep the current SSH session open.**

Open a second terminal on the local machine and test both accounts.

Test `deploy`:

```bash
ssh server-deploy
```

Both must successfully establish a new SSH session.

Only after both tests succeed should the original SSH session be closed.
