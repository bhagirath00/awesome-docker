# Installing Docker on Windows

## Introduction

Windows is widely used by developers and DevOps engineers. Running Docker on Windows works well, but it requires understanding two important concepts: **WSL 2** and **Docker Desktop for Windows**.

Docker containers run Linux processes. Windows is not Linux. To bridge this gap, Docker on Windows uses the **Windows Subsystem for Linux 2 (WSL 2)** — a Microsoft technology that runs a real Linux kernel directly inside Windows. This is not a virtual machine in the traditional sense — it is a lightweight, deeply integrated Linux environment that starts instantly and performs nearly as fast as native Linux.

This file walks you through the complete setup: enabling WSL 2, installing Docker Desktop, and configuring your environment for real DevOps work.

---

## How Docker Works on Windows

Before installing, you should understand what is happening under the hood.

When you run `docker run nginx` in a Windows terminal:

1. The Docker CLI sends the command to the Docker daemon
2. The Docker daemon runs inside your WSL 2 Linux environment
3. The container is created as a Linux process inside WSL 2
4. Ports are automatically mapped to your Windows network
5. You access the container from your Windows browser via `localhost`

From your perspective, it works exactly the same as on Linux or macOS. The WSL 2 integration handles everything transparently.

---

## System Requirements

Docker Desktop on Windows requires:

- **Windows 11 64-bit** (Home, Pro, Enterprise, or Education) — recommended
- **Windows 10 64-bit** version 21H2 or higher
- At least **4 GB RAM** (8 GB recommended)
- Hardware virtualization enabled in BIOS/UEFI (check in Task Manager → Performance → CPU)
- WSL 2 backend (required — this is the default)

---

## Step 1: Enable WSL 2

WSL 2 must be installed and configured before Docker Desktop can use it.

### Enable WSL 2 with One Command (Windows 11 and recent Windows 10)

Open PowerShell as Administrator and run:

```powershell
wsl --install
```

This single command:
- Enables the required Windows features
- Downloads and installs the Linux kernel
- Sets WSL 2 as the default version
- Installs Ubuntu as the default Linux distribution

Restart your computer after this command completes.

### Verify WSL 2 Installation

After restarting, open PowerShell and check:

```powershell
# Check WSL version and installed distributions
wsl --list --verbose

# Output:
  NAME      STATE           VERSION
* Ubuntu    Running         2
```

The VERSION column should show `2`. If it shows `1`, upgrade:

```powershell
wsl --set-version Ubuntu 2
```

---

## Step 2: Install Docker Desktop

**Download Docker Desktop for Windows:**
`https://www.docker.com/products/docker-desktop/`

Click **Download for Windows**.

**Run the installer:**

1. Double-click the downloaded `.exe` file
2. Ensure **Use WSL 2 instead of Hyper-V** is checked (it should be by default)
3. Click **OK** and wait for installation to complete
4. Click **Close and restart** when prompted
5. After restart, Docker Desktop launches automatically

You will see the Docker whale icon in your system tray (bottom-right of taskbar). When the animation stops, Docker is ready.

---

## Step 3: Configure Docker Desktop for WSL 2

After installation, configure Docker to integrate with your WSL 2 distribution.

1. Open Docker Desktop
2. Go to **Settings** (gear icon in the top right)
3. Go to **Resources → WSL Integration**
4. Enable integration with your Ubuntu distribution (toggle it on)
5. Click **Apply & Restart**

This allows you to run Docker commands from inside your WSL 2 Ubuntu terminal.

---

## Step 4: Verify the Installation

Open a PowerShell or Command Prompt window and verify:

```powershell
# Check Docker version
docker version

# Check Docker Compose version
docker compose version

# Run a test container
docker run hello-world
```

Also verify from inside WSL 2 — open an Ubuntu terminal (search "Ubuntu" in Start menu):

```bash
# Both of these should work
docker version
docker run hello-world
```

If both work, your installation is complete and correctly integrated.

---

## Working with Docker on Windows

### Which Terminal to Use

You can run Docker commands from:

- **PowerShell** — good for Windows-native tasks and scripting
- **Command Prompt** — basic Docker commands work here
- **WSL 2 Ubuntu terminal** — recommended for DevOps work, behaves exactly like Linux

For DevOps work, the **WSL 2 Ubuntu terminal** is the best choice. Most DevOps tooling (kubectl, AWS CLI, Ansible, etc.) works best in a Linux environment. Running these tools inside WSL 2 gives you a proper Linux experience without leaving Windows.

```powershell
# Open WSL 2 Ubuntu from PowerShell
wsl

# Or open the Ubuntu app directly from Start menu
```

### Accessing Files Between Windows and WSL 2

You can access your Windows files from inside WSL 2:

```bash
# Windows C: drive is mounted at /mnt/c in WSL 2
ls /mnt/c/Users/YourName/

# Navigate to a Windows project folder
cd /mnt/c/Users/YourName/projects/myapp

# Run Docker from there
docker build -t myapp:1.0 .
```

You can also access WSL 2 files from Windows Explorer:

- Open File Explorer
- In the address bar, type `\\wsl$`
- You will see your Linux distributions — browse your files like a regular Windows folder

---

## Resource Configuration

Docker Desktop on Windows uses resources from your Windows machine. Configure limits in **Settings → Resources**.

Recommended settings:

| Resource | Minimum | Recommended |
|---|---|---|
| **CPUs** | 2 | 4 |
| **Memory** | 4 GB | 8 GB |
| **Disk image size** | 20 GB | 60 GB |
| **Swap** | 1 GB | 2 GB |

---

## Using Docker with AWS from Windows

### Install AWS CLI in WSL 2

For AWS work, install the AWS CLI inside your WSL 2 Ubuntu environment:

```bash
# Inside WSL 2 Ubuntu terminal

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version

# Configure credentials
aws configure
```

### Authenticate with AWS ECR from WSL 2

```bash
# Inside WSL 2 Ubuntu terminal

# Authenticate Docker with ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Build and push
docker build -t myapp:1.0 .
docker tag myapp:1.0 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
```

Running all of this inside WSL 2 means you use the exact same commands and scripts as your Linux CI/CD runners and Linux servers — no Windows-specific workarounds needed.

---

## Common Windows Docker Issues

### Issue: "WSL 2 installation is incomplete"

```
Docker Desktop requires a newer WSL kernel version.
```

Fix — update WSL kernel:

```powershell
wsl --update
wsl --shutdown
```

Then restart Docker Desktop.

### Issue: "Hardware assisted virtualization and data execution protection must be enabled"

This means virtualization is disabled in your BIOS.

Fix:
1. Restart your computer
2. Enter BIOS/UEFI settings (usually F2, F10, or Delete during boot)
3. Find the Virtualization option (Intel VT-x or AMD-V)
4. Enable it and save

### Issue: Docker is very slow on Windows

The most common cause is running Docker from Windows file paths (`C:\Users\...`). File system operations across the Windows-WSL boundary are slow.

Fix: Do all your Docker work from inside the WSL 2 filesystem, not from Windows paths.

```bash
# SLOW — mounting from Windows filesystem
docker run -v /mnt/c/Users/YourName/project:/app python:3.11 python app.py

# FAST — working within WSL 2 filesystem
cd ~/projects/myapp   # This is inside WSL 2
docker run -v $(pwd):/app python:3.11 python app.py
```

Move your projects to your WSL 2 home directory (`~/projects/`) for the best performance.

### Issue: Port conflict

```
Error starting userland proxy: listen tcp4 0.0.0.0:80: bind: address already in use
```

Find and stop what is using the port:

```powershell
# Find process using port 80
netstat -ano | findstr :80

# Kill the process (replace PID with the actual number)
taskkill /PID <PID> /F
```

---

## Windows-Specific Docker Considerations

### Line Endings

Windows uses `CRLF` line endings, while Linux uses `LF`. If your scripts or Dockerfiles are created on Windows, they may have incorrect line endings that cause errors inside containers.

Always save files with LF line endings when working with Docker. In Visual Studio Code, set line endings to LF in the bottom status bar (click where it says `CRLF` and change to `LF`).

In Git, configure it to not convert line endings:

```bash
# Inside WSL 2
git config --global core.autocrlf false
```

### Windows Containers

Docker Desktop also supports **Windows containers** — containers that run Windows processes instead of Linux. These are used for running .NET Framework applications and other Windows-specific workloads.

Switch between Linux containers (the default) and Windows containers from the Docker Desktop system tray menu.

For DevOps work in most organizations, you will always use **Linux containers**. Windows containers are a specialized use case.

---

## Summary

- Docker on Windows uses WSL 2 — a real Linux kernel running inside Windows — to run Linux containers
- Install WSL 2 first with `wsl --install`, then install Docker Desktop for Windows
- Configure Docker Desktop to integrate with your WSL 2 Ubuntu distribution in **Settings → Resources → WSL Integration**
- For DevOps work, use the WSL 2 Ubuntu terminal — it behaves exactly like a real Linux system
- Store your projects inside the WSL 2 filesystem (not in Windows paths) for best performance
- Install AWS CLI inside WSL 2 and authenticate with ECR the same way you would on Linux
- Watch out for CRLF line endings — always save Docker-related files with LF endings

In the next chapter, we explore the Docker CLI — the commands you will use every day to manage containers, images, and more.
