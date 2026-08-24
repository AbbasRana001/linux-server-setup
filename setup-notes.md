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

---

# Part 7: Protect the Docker Compose Configuration

## Goal

Ensure that the privileged Docker Compose configuration cannot be modified by the `deploy` user.

This is important because the `deploy` user is authorized to execute the Compose configuration through `sudo`.

If `deploy` could modify that configuration, it could potentially alter the Docker deployment definition and use the approved command to perform unintended privileged operations.

Therefore:

- `root` owns the Compose configuration.
- `deploy` can read the configuration.
- `deploy` cannot modify the configuration.
- `deploy` can invoke only the explicitly approved Compose commands.

## Step 1 — Create the Deployment Configuration

The application deployment directory is:

```text
/opt/task-manager/
```

The Docker Compose configuration was created at:

```text
/opt/task-manager/docker-compose.yml
```

The configuration uses the already-built Docker image from Docker Hub rather than building the application on the EC2 host.

```yaml
services:
  api:
    image: abbasrana01/fastapi-task-manager:f9fd28377070fd60b8830f904bc5464727eabbbd
    ports:
      - "8000:8000"
    restart: unless-stopped
```

### Why image is used instead of build

The FastAPI application is already containerized and published to Docker Hub.

Therefore, the EC2 host does not need:

- FastAPI source code
- The application's Dockerfile
- Build dependencies
- A local Docker image build process

The server only needs to pull and run the published image.

## Step 2 — Use a Specific Image Tag

The deployment uses:

```text
abbasrana01/fastapi-task-manager:f9fd28377070fd60b8830f904bc5464727eabbbd
```

rather than:

```text
abbasrana01/fastapi-task-manager:latest
```

Using a specific tag makes the deployment more reproducible because the server is explicitly configured to run the intended image version rather than whatever image happens to be associated with `latest` in the future.

## Step 3 — Configure the Application Port

The Compose configuration contains:

```yaml
ports:
  - "8000:8000"
```

This maps:

```text
EC2 host port 8000
        |
        v
Container port 8000
        |
        v
FastAPI / Uvicorn
```

The application is therefore directly reachable through port 8000 for the current deployment.

HTTPS and reverse-proxy configuration using Nginx are intentionally deferred to a later hardening phase.

## Step 4 — Configure Container Restart Behavior

The Compose configuration contains:

```yaml
restart: unless-stopped
```

This instructs Docker to restart the container when appropriate, including after an unexpected container failure or Docker daemon/server restart, unless the container was explicitly stopped.

This provides container-level resilience.

A systemd service will be configured separately to provide host-level service management and boot-time orchestration.

## Step 5 — Protect the Compose File

The Compose file was assigned to `root`:

```bash
sudo chown root:root /opt/task-manager/docker-compose.yml
```

Permissions were set to:

```bash
sudo chmod 644 /opt/task-manager/docker-compose.yml
```

The resulting permissions are conceptually:

```text
root   -> read/write
others -> read
```

The file does not need execute permissions because a YAML Compose file is configuration data, not an executable program.

Docker Compose reads the YAML configuration when the Compose command is executed.

For example:

```bash
sudo docker compose -f /opt/task-manager/docker-compose.yml up -d
```

Here:

```text
docker compose
     |
     | reads
     v
docker-compose.yml
     |
     | defines
     v
Docker resources
```

The YAML file itself is never executed as a Linux executable.

---

# Part 8: Deploy the Task Manager Application

## Goal

Pull the published Task Manager Docker image from Docker Hub and start the application on the EC2 instance.

## Step 1 — Pull the Docker Image

As the `deploy` user:

```bash
sudo docker compose -f /opt/task-manager/docker-compose.yml pull
```

The image was successfully downloaded:

```text
[+] pull 9/9
 ✔ Image abbasrana01/fastapi-task-manager:f9fd28377070fd60b8830f904bc5464727eabbbd Pulled
```

This confirmed that:

- The scoped `sudo` rule allowed the operation.
- Docker Compose successfully read the deployment configuration.
- The EC2 instance could communicate with Docker Hub.
- The requested application image was successfully pulled.

## Step 2 — Start the Application

The application was started with:

```bash
sudo docker compose -f /opt/task-manager/docker-compose.yml up -d
```

Docker created the Compose network and application container:

```text
[+] up 2/2
 ✔ Network task-manager_default Created
 ✔ Container task-manager-api-1 Started
```

The `-d` option runs the containers in detached mode, allowing the SSH session to continue without attaching to the application's output.

## Step 3 — Verify Container Status

The running Compose stack was checked:

```bash
sudo docker compose -f /opt/task-manager/docker-compose.yml ps
```

The container was reported as:

```text
task-manager-api-1
```

with status:

```text
Up
```

The port mapping was:

```text
0.0.0.0:8000->8000/tcp
[::]:8000->8000/tcp
```

This confirms that Docker published port 8000 from the container to the EC2 host.

## Step 4 — Verify Application Startup Logs

Application logs were checked using:

```bash
sudo docker compose -f /opt/task-manager/docker-compose.yml logs
```

The logs confirmed successful FastAPI startup:

```text
Started server process [1]
Waiting for application startup.
Application startup complete.
Uvicorn running on http://0.0.0.0:8000
```

This confirmed that:

- Uvicorn started successfully.
- FastAPI completed application startup.
- The application is listening on container port 8000.

No missing environment-variable or secret configuration error prevented startup.

## Step 5 — Test the Application Locally

The application was tested from the EC2 host:

```bash
curl http://localhost:8000
```

Response:

```text
{"detail":"Not Found"}
```

This is expected because the API does not define a route for `/`.

The response nevertheless confirms that an HTTP request reached the FastAPI application.

## Step 6 — Test FastAPI Swagger Documentation

The Swagger UI endpoint was tested:

```bash
curl http://localhost:8000/docs
```

The response returned the FastAPI Swagger UI HTML page.

The page identifies the application as:

```text
Mini Task Manager API - Swagger UI
```

This confirms that the FastAPI application is serving its documentation endpoint successfully.

## Result

The Dockerized Task Manager application is successfully deployed and running on the EC2 instance.

Current application flow:

```text
Docker Hub
    |
    v
EC2 Docker Engine
    |
    v
task-manager-api-1
    |
    v
FastAPI / Uvicorn
    |
    v
EC2 port 8000
```

---

# Part 9: Configure UFW

## Goal

Enable the host firewall and allow only the inbound traffic currently required by the deployment.

UFW was not configured until after the application had been successfully started and tested.

## Step 1 — Verify the Initial UFW State

The firewall was checked with:

```bash
sudo ufw status verbose
```

Initial result:

```text
Status: inactive
```

No UFW rules were active at that point.

## Step 2 — Confirm SSH Port

The effective SSH configuration was checked.

SSH is running on:

```text
22/tcp
```

This port must be allowed before enabling UFW to avoid locking out the administrative SSH session.

## Step 3 — Allow SSH

The SSH port was allowed:

```bash
sudo ufw allow 22/tcp
```

This permits inbound SSH administration.

## Step 4 — Allow the Application Port

Because the current deployment directly exposes FastAPI on port 8000, the application port was allowed:

```bash
sudo ufw allow 8000/tcp
```

HTTP and HTTPS ports were intentionally not opened at this stage because Nginx and TLS/HTTPS are deferred to a later phase.

## Step 5 — Configure UFW Defaults

Incoming traffic was configured to be denied by default:

```bash
sudo ufw default deny incoming
```

Outgoing traffic was allowed by default:

```bash
sudo ufw default allow outgoing
```

This follows a default-deny approach for unsolicited inbound traffic.

## Step 6 — Enable UFW

After confirming that SSH port 22 had been allowed, UFW was enabled:

```bash
sudo ufw enable
```

## Step 7 — Verify the Firewall

The final firewall configuration was verified:

```bash
sudo ufw status verbose
```

Result:

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip


To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
8000/tcp                   ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
8000/tcp                   ALLOW IN    Anywhere (v6)
```

## Result

UFW is active with a default-deny inbound policy.

Currently permitted inbound traffic:

```text
22/tcp    → SSH administration
8000/tcp  → FastAPI application
```

IPv4 and IPv6 rules are both configured.

Nginx, HTTP, and HTTPS have not yet been configured.

## Current Deployment State

At the end of this phase, the EC2 server has the following configuration:

```text
EC2 Instance
│
├── SSH
│   └── Port 22
│
├── UFW
│   ├── Default incoming: DENY
│   ├── SSH 22/tcp: ALLOW
│   └── FastAPI 8000/tcp: ALLOW
│
├── Docker Engine
│
├── Docker Compose
│
└── /opt/task-manager/
    └── docker-compose.yml
        └── root:root
            └── Task Manager image
                └── task-manager-api-1
                    └── FastAPI :8000
```

The application has been successfully pulled, started, and tested.

---

# Part 10: Create a systemd Service

## Goal

Configure `systemd` to manage the Docker Compose Task Manager application as a system service.

The Docker Compose application was already running successfully. The purpose of this stage is to integrate the Compose stack with the host's systemd service manager so that the application can be started automatically during system boot and managed using standard `systemctl` commands.

The systemd service is:

```text
/etc/systemd/system/task-manager.service
```

## Step 1 — Create the systemd Unit

Create the service file:

```bash
sudo nano /etc/systemd/system/task-manager.service
```

The following configuration was used:

```ini
[Unit]
Description=Task Manager Application
Requires=docker.service
After=docker.service network-online.target
Wants=network-online.target


[Service]
Type=oneshot
WorkingDirectory=/opt/task-manager
ExecStart=/usr/bin/docker compose -f /opt/task-manager/docker-compose.yml up -d
ExecStop=/usr/bin/docker compose -f /opt/task-manager/docker-compose.yml down
RemainAfterExit=yes
TimeoutStartSec=0


[Install]
WantedBy=multi-user.target
```

## Step 3 — Reload systemd

After creating the unit file, reload systemd:

```bash
sudo systemctl daemon-reload
```

This causes systemd to re-read its unit configuration and recognize the newly created service.

## Step 4 — Verify the Unit

Check the service:

```bash
sudo systemctl status task-manager.service
```

Initial status:

```text
○ task-manager.service - Task Manager Application
     Loaded: loaded (/etc/systemd/system/task-manager.service; disabled; preset: enabled)
     Active: inactive (dead)
```

This was expected because the service had been created and loaded but had not yet been started.

---

# Part 11: Enable and Start the systemd Service

**Goal**

Enable the Task Manager systemd service so that it starts automatically during system boot, and verify that systemd can successfully start the Docker Compose application.

## Step 1 — Enable the Service

Enable the service:

```bash
sudo systemctl enable task-manager.service
```

This configures systemd to start the service automatically when the system reaches the appropriate boot target.

## Step 2 — Start the Service

Start the application through systemd:

```bash
sudo systemctl start task-manager.service
```

## Step 3 — Verify Service Status

Check:

```bash
sudo systemctl status task-manager.service
```

The service reported:

```
● task-manager.service - Task Manager Application
     Loaded: loaded (/etc/systemd/system/task-manager.service; enabled; preset: enabled)
     Active: active (exited)
```

The `active (exited)` state is expected for the configured `Type=oneshot` service with:

```
RemainAfterExit=yes
```

The important result was:

```
ExecStart=/usr/bin/docker compose -f /opt/task-manager/docker-compose.yml up -d
(code=exited, status=0/SUCCESS)
```

This confirms that the systemd `ExecStart` operation completed successfully.

The service output also confirmed:

```
Container task-manager-api-1 Running
```

Therefore, systemd successfully started the Docker Compose application.

---

# Part 13: Verify Reboot Persistence

**Goal**

Verify that the Task Manager application automatically returns after an EC2 instance reboot.

This is an important final test because enabling a systemd service is only useful if the application actually starts correctly during the server's boot process.

## Step 1 — Reboot the EC2 Instance

Reboot the server:

```bash
sudo reboot
```

The SSH connection will terminate as the EC2 instance restarts.

Reconnect after the server becomes available.

## Step 2 — Verify the systemd Service After Reboot

Check:

```bash
sudo systemctl status task-manager.service --no-pager
```

The service should be active.

## Step 3 — Verify the Docker Container After Reboot

Run:

```bash
sudo docker compose -f /opt/task-manager/docker-compose.yml ps
```

The Task Manager container should be running.

## Step 4 — Verify the Application After Reboot

Test:

```bash
curl http://localhost:8000/docs
```

The FastAPI Swagger UI endpoint should respond successfully.

This confirms that the application was automatically restored after the EC2 instance reboot.

---
# Part 14: Configure a Stable Public IP with an Elastic IP

## Goal

Assign a stable public IP address to the EC2 instance.

An EC2 instance's automatically assigned public IPv4 address can change when the instance is stopped and started. This would make a domain-based deployment unreliable because the DNS record would point to an address that could later become invalid.

An AWS Elastic IP provides a persistent public IPv4 address that can remain associated with the EC2 instance across stop/start cycles.

## Step 1 — Allocate an Elastic IP

From the AWS EC2 console:

1. Open **EC2**.
2. Navigate to **Network & Security → Elastic IPs**.
3. Select **Allocate Elastic IP address**.
4. Leave the default settings.
5. Select **Allocate**.

This creates a dedicated Elastic IP address in the AWS account.

## Step 2 — Associate the Elastic IP with the EC2 Instance

Select the newly allocated Elastic IP and choose **Actions → Associate Elastic IP address**.

Configure the following:

- **Resource type:** Instance
- **Instance:** Select the Task Manager EC2 instance

Then choose **Associate**.

The Elastic IP is now associated with the EC2 instance.

### Important Note — Public IP Change

Associating the Elastic IP changes the instance's public IPv4 address.

Any existing SSH connection using the previous public IP may therefore disconnect or stop responding.

This is expected behavior.

After the association, the new Elastic IP should be obtained from the EC2 instance details and used for subsequent connections.

For example:

```bash
ssh -i MyServer.pem ubuntu@<elastic-ip>
```

### Cost Consideration

AWS charges for public IPv4 addresses.

Therefore, an Elastic IP should not be treated as completely free infrastructure.

The address is required here because the deployment now needs a stable public endpoint for DNS and HTTPS.

# Part 15: Configure a Free DuckDNS Domain

## Goal

Create a domain name for the Task Manager server so that users can access the application through a hostname instead of directly using the Elastic IP address.

The domain used for this deployment is:

```
serverab.duckdns.org
```

## Step 1 — Create the DuckDNS Subdomain

1. Open the DuckDNS website and authenticate using an available authentication provider.
2. Create a subdomain under DuckDNS.

For this deployment:

```
serverab.duckdns.org
```

was created.

## Step 2 — Point the Domain to the Elastic IP

In the DuckDNS domain configuration, update the current IP address to the EC2 instance's Elastic IP.

The DNS relationship is therefore:

```
serverab.duckdns.org
        |
        v
Elastic IP
        |
        v
EC2 Instance
```

Because the EC2 instance now has a stable Elastic IP, the DNS record does not need to be changed after normal EC2 stop/start operations.

## Step 3 — Verify DNS Resolution

DNS resolution was checked from the local computer rather than from the EC2 server:

```bash
nslookup serverab.duckdns.org
```

The result should resolve the DuckDNS hostname to the configured Elastic IP.

This confirms that the hostname can be translated into the public IP address of the EC2 instance.

---

# Part 16: Install Nginx as a Reverse Proxy

## Goal

Install Nginx and place it in front of the Dockerized FastAPI application.

Before this change, the public request path was:

```
Client
   |
   v
EC2 :8000
   |
   v
Docker Container
   |
   v
FastAPI
```

Nginx changes the architecture to:

```
Client
   |
   v
Nginx :80
   |
   v
localhost:8000
   |
   v
Docker Container
   |
   v
FastAPI
```

Nginx therefore becomes the public-facing web server and reverse proxy.


## Step 1 — Update APT

```bash
sudo apt update
```

This refreshes the available package information.


## Step 2 — Install Nginx

```bash
sudo apt install -y nginx
```

Ubuntu's Nginx package automatically starts the Nginx service after installation.

Verify the service:

```bash
sudo systemctl status nginx
```

Expected:

```
Active: active (running)
```


## Step 3 — Verify Nginx Before Custom Configuration

Before modifying the Nginx configuration, the default Nginx page was tested using the EC2 instance's public IP.

Example:

```
http://<elastic-ip>
```

The default:

```
Welcome to nginx!
```

page confirms that:

- Nginx is running.
- Port 80 is reachable.
- The AWS networking configuration permits HTTP traffic.
- Nginx can serve requests successfully.

This provides a known-good baseline before introducing the reverse-proxy configuration.


## Step 4 — Create the Nginx Reverse Proxy Configuration

Create:

```bash
sudo nano /etc/nginx/sites-available/api-server
```

Configuration:

```nginx
server {
    listen 80;
    server_name serverab.duckdns.org;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Step 5 — Enable the Nginx Site

Create a symbolic link:

```bash
sudo ln -s /etc/nginx/sites-available/api-server /etc/nginx/sites-enabled/
```

Nginx uses the configuration files linked from:

```
/etc/nginx/sites-enabled/
```

to determine which sites should be active.


## Step 6 — Disable the Default Nginx Site

Remove the default configuration link:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

This prevents the default Nginx site from taking precedence over the Task Manager application configuration.


## Step 7 — Validate the Nginx Configuration

Before applying the configuration:

```bash
sudo nginx -t
```

Expected output includes:

```
syntax is ok
test is successful
```

This validation should always be performed before reloading Nginx.

It prevents an invalid configuration from being applied to the running service.


## Step 8 — Reload Nginx

Apply the configuration:

```bash
sudo systemctl reload nginx
```

A reload was used instead of a restart because Nginx can apply configuration changes without unnecessarily terminating existing connections.


## Step 9 — Test the Reverse Proxy

The application was tested through the DuckDNS domain:

```
http://serverab.duckdns.org/docs
```

The FastAPI Swagger UI should be returned.

The request path is now:

```
Browser
   |
   v
serverab.duckdns.org
   |
   v
Nginx :80
   |
   v
localhost:8000
   |
   v
Docker
   |
   v
FastAPI
```

---

# Part 17: Configure HTTPS with Let's Encrypt

## Goal

Secure the Task Manager API using HTTPS and a trusted TLS certificate issued by Let's Encrypt.

The HTTP reverse proxy was established first because Let's Encrypt needs to verify control of the domain before issuing the certificate.

## Step 1 — Install Certbot

Install Certbot and the Nginx integration:

```bash
sudo apt install -y certbot python3-certbot-nginx
```

The Nginx plugin allows Certbot to configure the Nginx HTTPS configuration automatically.

## Step 2 — Request and Install the Certificate

Run:

```bash
sudo certbot --nginx -d serverab.duckdns.org
```

Certbot requests the certificate for:

```
serverab.duckdns.org
```

During the process, Certbot requests:

- An email address for certificate-related notifications.
- Agreement to the Let's Encrypt Terms of Service.
- Whether HTTP traffic should be redirected to HTTPS.

The HTTP-to-HTTPS redirect option was selected.

### How Domain Validation Works

Let's Encrypt must verify that the server controls the requested domain.

For the HTTP validation process, Certbot temporarily creates a validation resource under:

```
/.well-known/acme-challenge/
```

Let's Encrypt's validation infrastructure then requests that resource through the public domain.

The successful validation establishes control of:

```
serverab.duckdns.org
```

and allows Let's Encrypt to issue the certificate.

This is why the following components needed to be working first:

```
DuckDNS
   |
   v
Elastic IP
   |
   v
EC2
   |
   v
Nginx :80
```

## Step 3 — Verify the Generated Nginx Configuration

Certbot automatically modifies the Nginx configuration.

The resulting configuration can be inspected with:

```bash
sudo cat /etc/nginx/sites-available/api-server
```

The configuration should now include HTTPS/TLS settings and the HTTP-to-HTTPS redirect.

## Step 4 — Verify HTTPS

Open:

```
https://serverab.duckdns.org/docs
```

The FastAPI Swagger UI should load over HTTPS.

The browser should also indicate that the connection is secured using a trusted Let's Encrypt certificate.

The final request path becomes:

```
Browser
   |
   | HTTPS :443
   v
Nginx
   |
   | HTTP internally
   v
localhost:8000
   |
   v
Docker Container
   |
   v
FastAPI
```

TLS terminates at Nginx.

The FastAPI application continues listening on port 8000 inside the host/Docker networking path.
