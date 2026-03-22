# Assignment 5 — DevSecOps Pipeline (GitHub + Jenkins + SonarQube)

This assignment walks through building a complete DevSecOps CI pipeline using two AWS EC2 instances, Jenkins as the automation server, and SonarQube as the static code analysis tool. The application code is written in FastAPI (Python).

---

## Architecture Overview

```
Developer (Local)
      |
      | git push
      v
   GitHub Repo
      |
      | webhook / manual trigger
      v
  Jenkins EC2 (port 8080)
      |
      | sonar-scanner
      v
 SonarQube EC2 (port 9000)
      |
      | quality gate result
      v
  Jenkins marks build PASS / FAIL
```

Two EC2 instances are used to isolate concerns — Jenkins handles CI orchestration while SonarQube handles code quality analysis independently.

---

## Project Files

| File | Purpose |
|------|---------|
| `main.py` | FastAPI application (3 endpoints) |
| `requirements.txt` | Python dependencies |
| `Dockerfile` | Containerizes the FastAPI app |
| `sonar-project.properties` | Tells SonarQube how to scan this project |
| `Jenkinsfile` | Declarative pipeline definition |

---

## Task 1 — Create 2 EC2 Instances with Your Own Keypair

### Steps

**1. Create a Key Pair**

1. Go to **AWS Console → EC2 → Key Pairs → Create key pair**
2. Fill in:
   - Name: `devops-a5-key`
   - Key pair type: `RSA`
   - Private key file format: `.pem`
3. Click **Create key pair** — the `.pem` file downloads automatically
4. Move it somewhere safe (e.g., `C:\Users\<you>\.ssh\devops-a5-key.pem`)

**2. Create EC2 Instance #1 — Jenkins Server**

1. Go to **EC2 → Instances → Launch instances**
2. Fill in:
   - Name: `Jenkins-Server`
   - AMI: `Ubuntu Server 22.04 LTS (HVM), SSD Volume Type`
   - Instance type: `t2.medium`
   - Key pair: select `devops-a5-key`
3. Under **Network settings → Create security group**, add these inbound rules:

   | Type       | Port | Source    | Reason                    |
   |------------|------|-----------|---------------------------|
   | SSH        | 22   | My IP     | Secure SSH access         |
   | Custom TCP | 8080 | 0.0.0.0/0 | Jenkins web UI access     |

4. Click **Launch instance**

**3. Create EC2 Instance #2 — SonarQube Server**

1. Go to **EC2 → Instances → Launch instances**
2. Fill in:
   - Name: `SonarQube-Server`
   - AMI: `Ubuntu Server 22.04 LTS (HVM), SSD Volume Type`
   - Instance type: `t2.medium`
   - Key pair: select `devops-a5-key`
3. Under **Network settings → Create security group**, add these inbound rules:

   | Type       | Port | Source    | Reason                         |
   |------------|------|-----------|--------------------------------|
   | SSH        | 22   | My IP     | Secure SSH access              |
   | Custom TCP | 9000 | 0.0.0.0/0 | SonarQube web UI access        |

4. Click **Launch instance**

**4. Screenshot**

Go to **EC2 → Instances**, and screenshot the **Public IPv4 address** column for both instances.

### Reasoning

Two separate instances are used so that Jenkins and SonarQube do not compete for the same CPU and memory. SonarQube (via its embedded Elasticsearch) is memory-intensive, and mixing it with Jenkins on the same machine often causes out-of-memory crashes. `t2.medium` (2 vCPU, 4GB RAM) is the minimum practical size for each. Using your own key pair rather than AWS-managed keys gives you direct SSH control over both instances.

---

## Task 2 — Connect to Jenkins EC2 via SSH

### Steps

**Option A — Windows Terminal / PowerShell**

```bash
ssh -i "C:\Users\<you>\.ssh\devops-a5-key.pem" ubuntu@<JENKINS_PUBLIC_IP>
```

If you get a permissions error on Windows, right-click the `.pem` file → Properties → Security → Advanced → disable inheritance → add only your user with Read permission.

**Option B — PuTTY**

1. Open **PuTTYgen** → Load → select `devops-a5-key.pem`
2. Click **Save private key** → save as `devops-a5-key.ppk`
3. Open **PuTTY**:
   - Host Name: `ubuntu@<JENKINS_PUBLIC_IP>`
   - Port: `22`
   - Go to **Connection → SSH → Auth → Credentials**
   - Browse and select `devops-a5-key.ppk`
4. Click **Open**

### Reasoning

AWS EC2 Ubuntu instances use key-based authentication only — there is no password login by default. PuTTY requires `.ppk` format (PuTTY's own format), which is why the conversion step from `.pem` is necessary on Windows. The default username for Ubuntu AMIs is always `ubuntu`.

---

## Task 3 — Update and Upgrade Ubuntu OS

Run this on **both** EC2 instances (SSH into each one):

### Steps

```bash
sudo apt update && sudo apt upgrade -y
```

After upgrading, if prompted about a kernel restart or service restart, press **Enter** to accept defaults.

### Reasoning

A fresh EC2 instance may have outdated package indexes and installed packages with known vulnerabilities. Running `apt update` refreshes the package index so you install the latest versions. Running `apt upgrade` patches existing packages. Doing this before installing any software prevents dependency conflicts and ensures you start from a secure, up-to-date baseline — which is a fundamental DevSecOps practice.

---

## Task 4 — Install Jenkins and Configure Ports

Run the following commands on the **Jenkins EC2** only.

### Steps

**1. Install Java 17 (Jenkins requires Java 17+)**

```bash
sudo apt install -y fontconfig openjdk-17-jre
java -version
```

**2. Add Jenkins Repository and Install**

```bash
# Download and store the Jenkins GPG signing key
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

# Add Jenkins stable repo to apt sources
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
sudo apt update
sudo apt install jenkins -y
```

**3. Start Jenkins**

```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
```

The status output should show `active (running)`.

**4. Get the Initial Admin Password**

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy this password.

**5. Complete Setup in Browser**

1. Open `http://<JENKINS_PUBLIC_IP>:8080` in your browser
2. Paste the admin password
3. Click **Install suggested plugins** and wait for it to finish
4. Create your admin account
5. Keep the default Jenkins URL and click **Save and Finish**

### Reasoning

Jenkins is a Java application, so Java must be installed first. The GPG key step ensures `apt` can verify the authenticity of downloaded Jenkins packages (prevents supply chain tampering). `systemctl enable` makes Jenkins start automatically on every reboot — important for a CI server that should always be available. Port 8080 was already opened in the security group in Task 1, so no firewall change is needed here.

---

## Task 5 — Install SonarQube Plugin in Jenkins

### Steps

1. Open Jenkins at `http://<JENKINS_PUBLIC_IP>:8080`
2. Go to **Manage Jenkins → Plugins → Available plugins**
3. In the search box, type `SonarQube Scanner`
4. Check the box next to **SonarQube Scanner**
5. Click **Install**
6. After installation completes, click **Restart Jenkins when no jobs are running**
   - Or navigate to `http://<JENKINS_PUBLIC_IP>:8080/restart` and confirm

**Screenshot:** Go to **Manage Jenkins → Plugins → Installed plugins**, search for `SonarQube`, and screenshot the result showing it installed.

### Reasoning

The SonarQube Scanner plugin adds two key capabilities to Jenkins: (1) it allows Jenkins to manage SonarQube server connections through its configuration UI, and (2) it exposes the `withSonarQubeEnv()` and `waitForQualityGate` pipeline steps used in the Jenkinsfile. Without this plugin, the pipeline script will fail at the SonarQube Analysis stage with an unrecognized step error.

---

## Task 6 — Install Docker on the SonarQube EC2

SSH into the **SonarQube EC2** and run:

### Steps

**1. Remove any old Docker versions**

```bash
sudo apt remove -y docker docker-engine docker.io containerd runc
```

**2. Install prerequisite packages**

```bash
sudo apt install -y ca-certificates curl gnupg lsb-release
```

**3. Add Docker's official GPG key**

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

**4. Add the Docker repository**

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**5. Install Docker Engine**

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

**6. Start Docker and add permissions**

```bash
sudo systemctl start docker
sudo systemctl enable docker

# Allow ubuntu user to run docker without sudo
sudo usermod -aG docker ubuntu

# Apply the group change in the current session
newgrp docker
```

**7. Verify**

```bash
docker --version
docker run hello-world
```

### Reasoning

Docker is required to run the SonarQube container in Task 7. The GPG key step verifies package integrity. Adding `ubuntu` to the `docker` group means you can run `docker` commands without `sudo` in subsequent steps — this is also required for Jenkins to invoke Docker without elevated privileges. `systemctl enable` makes Docker start on reboot.

---

## Task 7 — Pull and Run SonarQube Docker Image

Still on the **SonarQube EC2**:

### Steps

**1. Set the required kernel parameter**

SonarQube uses Elasticsearch internally, which requires a higher virtual memory limit:

```bash
sudo sysctl -w vm.max_map_count=262144

# Make the change permanent across reboots
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

**2. Pull the SonarQube image**

```bash
docker pull sonarqube:lts-community
```

**3. Run SonarQube**

```bash
docker run -d \
  --name sonarqube \
  --restart unless-stopped \
  -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  sonarqube:lts-community
```

**4. Verify the container is running**

```bash
docker ps
```

You should see `sonarqube` listed with status `Up`.

**5. Wait for SonarQube to be ready**

SonarQube takes 60–90 seconds to initialize. Check readiness:

```bash
docker logs sonarqube --follow
```

Wait until you see a line containing `SonarQube is operational`. Press `Ctrl+C` to exit the log stream.

**Screenshot:** The output of `docker ps` showing the running SonarQube container.

### Reasoning

`sonarqube:lts-community` is the free, long-term-support community edition. The `vm.max_map_count` setting is mandatory — without it, the embedded Elasticsearch inside SonarQube will crash immediately on startup. `SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true` disables additional Elasticsearch production-readiness checks that would also fail in a basic EC2 environment. `--restart unless-stopped` ensures SonarQube restarts automatically if the container crashes or the EC2 instance reboots.

---

## Task 8 — Login to SonarQube and Change Password

### Steps

1. Open `http://<SONARQUBE_PUBLIC_IP>:9000` in your browser
2. Login with the default credentials:
   - Username: `admin`
   - Password: `admin`
3. SonarQube immediately redirects you to a **Change Password** screen
4. Enter and confirm a new strong password
5. Click **Update**

### Reasoning

The default `admin/admin` credentials are publicly known and must be changed immediately. Leaving them unchanged exposes your SonarQube instance (which is publicly accessible on port 9000) to unauthorized access — anyone could view your code analysis results or generate tokens. This is a basic security hardening step required in any real environment.

---

## Task 9 — Create a SonarQube Webhook to Communicate with Jenkins

### Steps

1. In SonarQube, go to **Administration → Configuration → Webhooks**
2. Click **Create**
3. Fill in:
   - **Name:** `Jenkins`
   - **URL:** `http://<JENKINS_PUBLIC_IP>:8080/sonarqube-webhook/`
   - **Secret:** (leave blank for now)
4. Click **Create**

**Screenshot:** The Webhooks page showing the newly created `Jenkins` webhook with its URL.

### Reasoning

After SonarQube finishes analyzing code, Jenkins needs to know the quality gate result (pass or fail). SonarQube cannot push results to Jenkins unless it knows where to send them — this webhook URL is that destination. The path `/sonarqube-webhook/` is the fixed endpoint exposed by the SonarQube Scanner Jenkins plugin. Without this webhook, the `waitForQualityGate` step in the Jenkinsfile would hang until it times out.

---

## Task 10 — Configure Jenkins to Communicate with SonarQube

This requires two things: a token from SonarQube, and configuration inside Jenkins.

### Steps

**Step 1 — Generate a Token in SonarQube**

1. In SonarQube, click your avatar (top-right) → **My Account → Security**
2. Under **Generate Tokens**:
   - Token Name: `jenkins-token`
   - Type: `Global Analysis Token`
   - Expiry: `No expiration` (or set a date)
3. Click **Generate**
4. **Copy the token immediately** — SonarQube will never show it again

**Step 2 — Add the Token to Jenkins Credentials**

1. In Jenkins, go to **Manage Jenkins → Credentials → System → Global credentials (unrestricted)**
2. Click **Add Credentials**
3. Fill in:
   - Kind: `Secret text`
   - Scope: `Global`
   - Secret: paste the SonarQube token
   - ID: `sonar-token`
   - Description: `SonarQube Token`
4. Click **Create**

**Step 3 — Add SonarQube Server in Jenkins System Config**

1. Go to **Manage Jenkins → System**
2. Scroll down to the **SonarQube servers** section
3. Check **Environment variables**
4. Click **Add SonarQube**:
   - Name: `SonarQube`
   - Server URL: `http://<SONARQUBE_PUBLIC_IP>:9000`
   - Server authentication token: select `sonar-token` from the dropdown
5. Click **Save**

**Screenshots needed:**
- The token generation screen in SonarQube
- The Jenkins credentials page showing `sonar-token`
- The Jenkins System config showing the SonarQube server entry

### Reasoning

Jenkins needs to authenticate with SonarQube to submit analysis results. The token serves as an API key — it proves to SonarQube that the request is coming from an authorized Jenkins instance. Storing it as a Jenkins `Secret text` credential (rather than hardcoding it in the Jenkinsfile) keeps it out of version control. The name `SonarQube` given to the server entry must exactly match the string used in `withSonarQubeEnv('SonarQube')` in the Jenkinsfile, and the credential ID `sonar-token` must match the `credentials('sonar-token')` reference in the pipeline environment block.

---

## Task 11 — Configure SonarQube Scanner Tool Inside Jenkins

### Steps

1. Go to **Manage Jenkins → Tools**
2. Scroll to the **SonarQube Scanner installations** section
3. Click **Add SonarQube Scanner**:
   - Name: `sonar-scanner`
   - Check **Install automatically**
   - Select the latest version from the dropdown
4. Click **Save**

**Screenshot:** The SonarQube Scanner tool entry in the Tools configuration page.

### Reasoning

The SonarQube Scanner is a standalone CLI tool that performs the actual code analysis. Jenkins does not bundle it by default — it must be installed separately as a tool. Selecting **Install automatically** means Jenkins will download and cache the scanner binary on the build agent before the first pipeline run, so you do not need to manually install it on the Jenkins EC2. The name `sonar-scanner` here must match what is referenced if you use `tool 'sonar-scanner'` in a scripted pipeline (in our declarative pipeline, `withSonarQubeEnv` handles this automatically through the server configuration).

---

## Task 12 — Push Project Code to GitHub

### Steps

**1. Create a new repository on GitHub**

1. Go to `https://github.com/new`
2. Repository name: `devops-assignment-5`
3. Visibility: Public (or Private)
4. Do NOT initialize with a README (your local repo already has files)
5. Click **Create repository**

**2. Push your local code**

Open a terminal in your project directory and run:

```bash
git add .
git commit -m "Initial FastAPI project with Jenkinsfile and SonarQube config"
git branch -M main
git remote add origin https://github.com/<YOUR_USERNAME>/devops-assignment-5.git
git push -u origin main
```

Replace `<YOUR_USERNAME>` with your actual GitHub username.

**Screenshot:** Your GitHub repository page showing all project files:
- `main.py`
- `requirements.txt`
- `Dockerfile`
- `sonar-project.properties`
- `Jenkinsfile`

### Reasoning

Jenkins pulls source code from GitHub to run the pipeline. The code must be in a remote repository that Jenkins can access over the internet. The `Jenkinsfile` at the root of the repository is detected automatically by Jenkins when the pipeline is configured with **Pipeline script from SCM**. Pushing all configuration files together in one commit establishes a clean, reproducible project baseline.

---

## Task 13 & 14 — Create Declarative Pipeline and Write the Script

### Steps

**Create the Pipeline Job in Jenkins**

1. Jenkins → **New Item**
2. Enter name: `devops-assignment-5`
3. Select **Pipeline** → Click **OK**
4. In the job configuration:
   - Under **Pipeline → Definition**: select `Pipeline script from SCM`
   - SCM: `Git`
   - Repository URL: `https://github.com/<YOUR_USERNAME>/devops-assignment-5.git`
   - Credentials: add if the repo is private (leave blank for public)
   - Branch Specifier: `*/main`
   - Script Path: `Jenkinsfile`
5. Click **Save**

**The Jenkinsfile (already in your repo):**

```groovy
pipeline {
    agent any

    environment {
        SONAR_TOKEN = credentials('sonar-token')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=devops-assignment-5 \
                          -Dsonar.projectName="DevOps Assignment 5 - FastAPI" \
                          -Dsonar.projectVersion=1.0 \
                          -Dsonar.sources=. \
                          -Dsonar.exclusions=**/__pycache__/**,**/*.pyc,**/venv/**,.git/** \
                          -Dsonar.language=py \
                          -Dsonar.python.version=3 \
                          -Dsonar.token=${SONAR_TOKEN}
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

**Pipeline Stage Breakdown:**

| Stage | What it does |
|-------|-------------|
| `Checkout` | Clones the GitHub repo onto the Jenkins build agent |
| `SonarQube Analysis` | Runs `sonar-scanner` against the source code and sends results to SonarQube |
| `Quality Gate` | Waits for SonarQube to evaluate the analysis against quality rules and returns pass/fail |

**Screenshot:** The full Jenkinsfile content (either from GitHub or from the Jenkins pipeline config).

### Reasoning

Declarative pipelines (`pipeline { ... }`) are the modern Jenkins standard. They are human-readable, version-controlled alongside code, and easier to maintain than freestyle jobs. `withSonarQubeEnv('SonarQube')` injects the SonarQube server URL and token into the environment automatically, so they do not need to be hardcoded. `credentials('sonar-token')` retrieves the secret from Jenkins credential store at runtime, keeping it out of the source code. `checkout scm` automatically uses the SCM configuration from the Jenkins job itself, making the Jenkinsfile reusable.

---

## Task 15 — Manually Build and Verify

### Steps

1. Go to your Jenkins job: `http://<JENKINS_PUBLIC_IP>:8080/job/devops-assignment-5/`
2. Click **Build Now** in the left sidebar
3. A build appears in **Build History** (bottom-left) — click the build number (e.g., `#1`)
4. Click **Console Output** to watch the live log
5. Alternatively, click **Stage View** on the job page to see a visual pipeline diagram

A successful build will show all three stages in green and end with:
```
Finished: SUCCESS
```

If a stage fails, the console output will show the exact error. Common issues:
- `sonar-scanner: command not found` → The SonarQube Scanner tool was not saved in Task 11
- `Could not find credentials` → The credential ID in Jenkinsfile does not match what was created in Task 10
- `Connection refused` → The SonarQube server URL in Jenkins System config is wrong

**Screenshot:** The stage view or console output showing all three stages passing.

### Reasoning

A manual build verifies that every integration point configured in Tasks 4–12 is working correctly end-to-end: Jenkins can reach GitHub, sonar-scanner can authenticate with SonarQube, and the webhook can deliver the quality gate result back to Jenkins. Running it manually first (before any automation) isolates setup issues from trigger issues.

---

## Task 16 — View SonarQube Report

### Steps

**Option A — Direct link from Console Output**

In the Jenkins Console Output, find the line:
```
INFO: ANALYSIS SUCCESSFUL, you can browse http://<SONARQUBE_PUBLIC_IP>:9000/dashboard?id=devops-assignment-5
```

Open that URL.

**Option B — Via SonarQube UI**

1. Go to `http://<SONARQUBE_PUBLIC_IP>:9000`
2. Click **Projects**
3. Click **devops-assignment-5**

The report shows:
- **Bugs** — likely defects in logic
- **Vulnerabilities** — security issues
- **Code Smells** — maintainability issues
- **Coverage** — percentage of code covered by tests
- **Duplications** — copy-pasted code blocks

**Screenshot:** The SonarQube project dashboard showing the analysis results.

### Reasoning

The SonarQube report is the core output of the DevSecOps pipeline. It provides objective, automated code quality and security feedback on every build. The quality gate (configured in SonarQube's default profile) defines the minimum acceptable thresholds — if the code does not meet them, the Jenkins build fails, enforcing quality as a gate condition rather than an afterthought.

---

## Task 17 — Add Quality Gate Timeout

The `Jenkinsfile` already contains the quality gate timeout. Here is the relevant block:

```groovy
stage('Quality Gate') {
    steps {
        timeout(time: 2, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}
```

No changes to the file are needed. The two parameters work together:

| Parameter | Value | Effect |
|-----------|-------|--------|
| `timeout(time: 2, unit: 'MINUTES')` | 2 minutes | Jenkins aborts the stage if SonarQube does not respond within 2 minutes |
| `waitForQualityGate abortPipeline: true` | true | If the quality gate **fails**, the entire pipeline is marked as failed and aborted |

**Screenshot:** The `Quality Gate` stage block in your Jenkinsfile (from GitHub or Jenkins script editor).

### Reasoning

`waitForQualityGate` is a blocking call — it pauses the Jenkins build and waits for SonarQube to send back the quality gate result via the webhook configured in Task 9. Without a `timeout`, if the webhook is misconfigured or SonarQube is slow, the build will hang indefinitely, blocking the Jenkins executor. The 2-minute timeout is a safety net that guarantees the build either resolves or is abandoned within a bounded time. `abortPipeline: true` enforces the DevSecOps principle that failing code quality should block deployment, not just generate a warning.

---

## Task 18 — Final Build Verification

### Steps

1. Go to your Jenkins job: `http://<JENKINS_PUBLIC_IP>:8080/job/devops-assignment-5/`
2. Click **Build Now**
3. Monitor the **Stage View** — confirm all three stages complete in green:

   ```
   ✅ Checkout  →  ✅ SonarQube Analysis  →  ✅ Quality Gate
   ```

4. Click the build number → **Console Output** and confirm:
   ```
   INFO: Quality Gate status: OK
   Finished: SUCCESS
   ```

5. Return to SonarQube (`http://<SONARQUBE_PUBLIC_IP>:9000`) and confirm a new analysis appears with a **Passed** quality gate badge.

**Screenshot:** The Jenkins stage view showing all stages green, and the SonarQube dashboard showing the quality gate as **Passed**.

### Reasoning

This final build confirms that the complete DevSecOps pipeline is working from end to end — including the quality gate timeout added in Task 17. A passing quality gate means the code meets the defined standards for bugs, vulnerabilities, and code smells. In a production pipeline, this stage would gate a deployment: code that fails the quality gate would never reach the deployment stage, enforcing continuous code quality at the CI level.

---

## Port Reference

| Service | EC2 | Port | Security Group Rule |
|---------|-----|------|---------------------|
| SSH | Both | 22 | My IP only |
| Jenkins UI | Jenkins EC2 | 8080 | 0.0.0.0/0 |
| SonarQube UI | SonarQube EC2 | 9000 | 0.0.0.0/0 |

## Credential Reference

| ID in Jenkins | What it stores | Used in |
|---------------|---------------|---------|
| `sonar-token` | SonarQube Global Analysis Token | Jenkinsfile `environment` block |

## SonarQube Configuration Reference

| Field | Value |
|-------|-------|
| Server name in Jenkins | `SonarQube` |
| Project key | `devops-assignment-5` |
| Webhook URL | `http://<JENKINS_PUBLIC_IP>:8080/sonarqube-webhook/` |
