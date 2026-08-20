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

# Part 4: Configure Permissions

## Goal

Create a dedicated application/deployment directory for the Dockerized Task Manager application while maintaining the principle of least privilege.

The application will be deployed using Docker, so the FastAPI source code does not need to exist directly on the host. The host-side directory will be used for deployment-related files/configuration.

Application directory:

```text
/opt/task-manager/
```

---

## Step 1 — Inspect `/opt`

While logged in as the `deploy` user:

```bash
ls -ld /opt
```

```bash
ls -la /opt
```

`/opt` is owned by `root:root`, which is expected for a system-level application directory.

The `deploy` user should not have permission to create arbitrary directories directly under `/opt`.

---

## Step 2 — Create the Application Directory

Switch to the administrative `ubuntu` account:

```bash
exit
```

Then connect using the administrative SSH alias.

Verify:

```bash
whoami
```

Expected:

```text
ubuntu
```

Create the application directory:

```bash
sudo mkdir /opt/task-manager
```

The directory is initially owned by `root`.

---

## Step 3 — Assign Ownership to `deploy`

Give the deployment user ownership of the application directory:

```bash
sudo chown deploy:deploy /opt/task-manager
```

Verify:

```bash
ls -ld /opt/task-manager
```

Expected:

```text
drwxr-xr-x 2 deploy deploy ... /opt/task-manager
```

The directory remains underneath the root-owned `/opt` directory, but `deploy` now controls its own application directory.

---

## Step 4 — Verify `deploy` Can Manage Its Application Directory

Switch back to the deployment account:

```bash
exit
```

Then:

```bash
ssh server-deploy
```

Verify:

```bash
whoami
```

Expected:

```text
deploy
```

Enter the application directory:

```bash
cd /opt/task-manager
```

Verify the location:

```bash
pwd
```

Expected:

```text
/opt/task-manager
```

Test that `deploy` can create files without `sudo`:

```bash
touch permission-test.txt
```

Verify ownership:

```bash
ls -la
```

The test file should be owned by `deploy:deploy`.

Remove the test file:

```bash
rm permission-test.txt
```

Verify the directory is clean:

```bash
ls -la
```

### Result

The permission model has been verified:

* `/opt` remains owned by `root`
* `/opt/task-manager` is owned by `deploy`
* `deploy` can manage files inside `/opt/task-manager`
* `deploy` does not require `sudo` to manage its application files
* `deploy` has not been added to the `sudo` group

---

# Part 5: Install Required Packages

## Goal

Install Docker Engine and Docker Compose on the Ubuntu EC2 instance.

The FastAPI Task Manager application is already Dockerized and available on Docker Hub, so the EC2 host only needs the Docker runtime and deployment configuration.

Docker was installed from Docker's official APT repository rather than Ubuntu's `docker.io` package.

---

## Step 1 — Switch to the Administrative User

Docker installation requires administrative privileges.

If currently logged in as `deploy`:

```bash
exit
```

Connect using the administrative SSH alias.

Verify:

```bash
whoami
```

Expected:

```text
ubuntu
```

---

## Step 2 — Update APT Package Information

```bash
sudo apt update
```

This refreshes the package information from the configured APT repositories.

It does not install or upgrade packages by itself.

---

## Step 3 — Install Repository Prerequisites

```bash
sudo apt install ca-certificates curl
```

These packages were already installed and up to date on the server.

* `ca-certificates` provides trusted CA certificates for HTTPS connections.
* `curl` is used to retrieve Docker's repository signing key.

No packages needed to be installed or upgraded at this stage.

---

## Step 4 — Create the APT Keyring Directory

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

This creates the directory used to store repository signing keys.

---

## Step 5 — Add Docker's Repository Signing Key

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
```

The key allows APT to verify packages downloaded from Docker's official repository.

Make the key readable:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

---

## Step 6 — Add Docker's Official APT Repository

Create the repository configuration:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

This creates:

```text
/etc/apt/sources.list.d/docker.sources
```

The file tells APT to obtain Docker packages from Docker's official repository.

The command automatically determines:

* Ubuntu release codename
* CPU architecture

---

## Step 7 — Refresh APT

After adding the Docker repository:

```bash
sudo apt update
```

The output confirmed that Docker's repository was successfully recognized:

```text
https://download.docker.com/linux/ubuntu
```
---

## Step 8 — Install Docker Engine

Install Docker Engine and its required components:

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Installed components:

| Package                 | Purpose                                    |
| ----------------------- | ------------------------------------------ |
| `docker-ce`             | Docker Engine / daemon                     |
| `docker-ce-cli`         | `docker` command-line interface            |
| `containerd.io`         | Container runtime used by Docker           |
| `docker-buildx-plugin`  | Modern Docker image-building functionality |
| `docker-compose-plugin` | Provides the `docker compose` command      |

---

## Step 9 — Verify Docker Installation

Check the Docker CLI:

```bash
docker --version
```

Check Docker Compose:

```bash
docker compose version
```

---

## Step 10 — Verify the Docker Service

```bash
sudo systemctl status docker
```

The service should show:

```text
Active: active (running)
```

The Docker service was also shown as:

```text
enabled
```

This means Docker is configured to start automatically when the server boots.

---

## Step 11 — Test Docker End-to-End

Run Docker's test image:

```bash
sudo docker run --rm hello-world
```

Expected output includes:

```text
Hello from Docker!
```

This message shows that the installation appears to be working correctly.

This verifies the complete Docker workflow:

1. Docker CLI contacted the Docker daemon.
2. Docker pulled the `hello-world` image from Docker Hub.
3. Docker created a container.
4. The container executed successfully.
5. `--rm` automatically removed the test container after it exited.

### Result

Docker Engine and Docker Compose are installed and working correctly.

---

## Important Security Decision — Docker Access for `deploy`

At this stage, `deploy` has **not** been added to the `docker` group.

Do **not** run:

```bash
sudo usermod -aG docker deploy
```

yet.

Membership in the Docker group provides access to the Docker daemon and can effectively provide root-equivalent control over the host.

This is particularly important for this project because the `deploy` user was intentionally created as a restricted, non-root deployment identity.

We will decide how the deployment workflow should interact with Docker before granting `deploy` any additional privileges.
Our next part will therefore be **designing the Docker deployment permissions**, not blindly running `usermod -aG docker deploy`.

Once we settle that, we'll use your actual Docker Hub image to deploy the Task Manager.

---

# Part 6: Configure Restricted Docker Deployment Permissions

## Goal

Allow the non-root `deploy` user to manage the Task Manager Docker Compose application without granting unrestricted access to the Docker daemon.

The `deploy` user was intentionally created as a restricted deployment identity. Therefore, it should not be added to the `docker` group.

Instead, a scoped `sudo` policy will allow `deploy` to execute only the Docker Compose commands required to manage this specific application.

The deployment configuration will also be protected so that `deploy` cannot modify the privileged Compose file and then execute arbitrary Docker configuration through the approved command.

---

## Security Decision

The Docker group was deliberately **not** used.

Do not run:

```bash
sudo usermod -aG docker deploy
```

Membership in the Docker group provides access to the Docker daemon and can effectively provide root-equivalent control over the host.

Instead, the deployment uses scoped sudo permissions.

The intended model is:

```text
deploy
   |
   | sudo
   v
Approved Docker Compose commands
   |
   v
Root-owned Compose configuration
   |
   v
Docker daemon
   |
   v
Task Manager container
```

This provides a narrower privilege boundary than unrestricted Docker daemon access.

## Step 1 — Confirm the Docker Binary Path

As the administrative `ubuntu` user:

```bash
which docker
```

Output:

```text
/usr/bin/docker
```

The absolute path is used in the sudoers configuration rather than relying on the user's PATH.

This ensures that the sudo rule refers to the intended Docker executable.

## Step 2 — Create the Scoped Sudoers Policy

The sudoers configuration was created using `visudo`:

```bash
sudo visudo -f /etc/sudoers.d/deploy-docker
```

The following policy was configured:

```text
# Allow 'deploy' to manage the Task Manager Docker Compose stack without a
# password prompt, scoped strictly to this application's compose file.
# No general 'docker' access is granted — deploy cannot run arbitrary
# docker/docker compose commands, only the ones listed below.


Cmnd_Alias TASKMANAGER_DOCKER = /usr/bin/docker compose -f /opt/task-manager/docker-compose.yml pull, \
                                 /usr/bin/docker compose -f /opt/task-manager/docker-compose.yml up -d, \
                                 /usr/bin/docker compose -f /opt/task-manager/docker-compose.yml down, \
                                 /usr/bin/docker compose -f /opt/task-manager/docker-compose.yml restart, \
                                 /usr/bin/docker compose -f /opt/task-manager/docker-compose.yml ps, \
                                 /usr/bin/docker compose -f /opt/task-manager/docker-compose.yml logs


deploy ALL=(root) NOPASSWD: TASKMANAGER_DOCKER
```

Allowed operations

The `deploy` user can execute only these Compose operations for the Task Manager stack:

```text
pull
up -d
down
restart
ps
logs
```

General Docker commands are not permitted.

For example:

```bash
sudo docker ps
```

is intentionally denied.

## Step 3 — Validate the Sudoers Configuration

The sudoers configuration was validated with:

```bash
sudo visudo -c
```

The configuration parsed successfully.

## Step 4 — Verify Sudoers File Ownership and Permissions

The sudoers file was checked:

```bash
sudo ls -l /etc/sudoers.d/deploy-docker
```

The file is owned by:

```text
root:root
```

Its permissions were subsequently tightened to:

```text
0440
```

Resulting permissions:

```text
-r--r----- 1 root root ... /etc/sudoers.d/deploy-docker
```

This prevents the `deploy` user from modifying the sudo policy.

## Step 5 — Verify Effective Deployment Permissions

As the `deploy` user:

```bash
sudo -l
```

The output showed only the explicitly permitted Docker Compose commands.

An unrestricted Docker command was tested:

```bash
sudo docker ps
```

The command was denied:

```text
sudo: I'm sorry deploy. I'm afraid I can't do that
```

This confirmed that `deploy` does not have unrestricted Docker access.
