# 🚀 CI/CD Pipeline with Jenkins, SonarQube, Nexus & Tomcat on AWS EC2

A complete end-to-end **CI/CD pipeline** built on AWS EC2 using Jenkins (with a custom agent node), SonarQube for code quality analysis, Nexus as artifact repository, and Apache Tomcat for deployment. The project uses the **Spring Framework PetClinic** application as the sample project.

---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [Architecture](#-architecture)
- [Tools & Technologies](#-tools--technologies)
- [AWS Infrastructure Setup](#-aws-infrastructure-setup)
  - [EC2 Instances](#step-1-ec2-instances)
  - [Security Group Configuration](#step-2-security-group-configuration)
- [Jenkins Setup](#-jenkins-setup)
  - [Jenkins Dashboard](#step-3-jenkins-dashboard)
  - [Jenkins Agent Node](#step-4-jenkins-agent-node)
  - [Jenkins Credentials](#step-5-jenkins-credentials)
- [Pipeline Stages](#-pipeline-stages)
  - [Pipeline Job Status](#pipeline-job-status)
  - [Jenkins Workspace](#jenkins-workspace)
  - [Stage View Builds](#stage-view-builds)
- [Pipeline Results](#-pipeline-results)
  - [SonarQube Quality Gate](#sonarqube-quality-gate)
  - [Nexus Artifact Repository](#nexus-artifact-repository)
  - [Application Deployment](#application-deployment)
  - [Email Notifications](#email-notifications)
- [Sample Jenkinsfile](#-sample-jenkinsfile-structure)
- [Key Learnings](#-key-learnings)
- [Author](#-author)

---

## 📌 Project Overview

This project demonstrates a production-grade CI/CD pipeline that automates:

- **Source code cloning** from GitHub
- **Build** using Maven
- **Code quality analysis** with SonarQube
- **Artifact upload** to Nexus Repository Manager
- **Deployment** to Apache Tomcat
- **Email notifications** on build success/failure

The pipeline runs on a **custom Jenkins agent node** (`agent_kishore`) on a dedicated EC2 instance, keeping the Jenkins master node clean for orchestration only.

---

## 🏗️ Architecture

```
Developer → GitHub
                │
                ▼
        Jenkins Master (EC2: 3.109.124.29:8080)
                │
        agent_kishore (separate EC2)
                │
    ┌───────────┼─────────────┐
    ▼           ▼             ▼
SonarQube     Nexus        Tomcat
(43.204.221.52:9000) (13.201.4.220:8081) (65.2.121.19:8080)
                              │
                        PetClinic App
                   (http://65.2.121.19:8080/petclinic/)
```

---

## 🛠️ Tools & Technologies

| Tool | Version | Purpose | Port |
|------|---------|---------|------|
| Jenkins | 2.555.2 | CI/CD Orchestration (Master + Agent) | 8080 |
| SonarQube | Community Edition | Code Quality & Security Analysis | 9000 |
| Sonatype Nexus | Community Edition | Artifact Repository Manager | 8081 |
| Apache Tomcat | — | Application Deployment Server | 8080 |
| AWS EC2 | c7i.flex.large | Cloud Infrastructure (×4 instances) | — |
| Maven | 3.9.12-1 | Build Tool | — |
| Spring Framework PetClinic | v7.0.3 | Sample Java Web Application | — |
| Gmail (SMTP) | — | Build Notifications via Extended Email Plugin | 465 / 25 |

---

## ☁️ AWS Infrastructure Setup

### Step 1: EC2 Instances

Four EC2 instances were launched in the **ap-south-1b (Mumbai)** region, all of type `c7i.flex.large`, using key pair `key-mumbai`:

| Instance Name | Instance ID | Public IP | Role |
|--------------|-------------|-----------|------|
| JENKINS | i-0ea484fec30fba7fb | 3.109.124.29 | Jenkins Master (port 8080) |
| SONARQUBE | i-067888984db533809 | 43.204.221.52 | Code Quality Analysis (port 9000) |
| nexus | i-03d68df742fe3f081 | 13.201.4.220 | Artifact Repository (port 8081) |
| tomcat | i-017f5ab501b8e5717 | 65.2.121.19 | Application Deployment (port 8080) |

All 4 instances are **Running** with 3/3 status checks passed.

> **Note:** Jenkins and Tomcat both use port 8080, but they run on **separate EC2 instances** with different public IPs.

<img width="3839" height="2028" alt="Screenshot 2026-05-21 003848" src="https://github.com/user-attachments/assets/6d39d427-ade7-4c73-b964-80007629c91e" />


---

### Step 2: Security Group Configuration

A single security group **`launch-wizard-1`** (`sg-03c2c594e701feedc`) is shared across all EC2 instances. It has **8 inbound rules** covering all required service ports:

| Port | Type | Protocol | Source | Purpose |
|------|------|----------|--------|---------|
| 22 | SSH | TCP | 0.0.0.0/0 | Remote access to EC2 instances |
| 25 | SMTP | TCP | 0.0.0.0/0 | Email relay (outbound) |
| 80 | HTTP | TCP | 0.0.0.0/0 | General web traffic |
| 465 | SMTPS | TCP | 0.0.0.0/0 | Secure Gmail SMTP for email notifications |
| 8080 | Custom TCP | TCP | 0.0.0.0/0 | Jenkins UI & Tomcat application |
| 8081 | Custom TCP | TCP | 0.0.0.0/0 | Nexus Repository Manager |
| 8082 | Custom TCP | TCP | 0.0.0.0/0 | Additional custom service port |
| 9000 | Custom TCP | TCP | 0.0.0.0/0 | SonarQube |

> ⚠️ All rules allow `0.0.0.0/0` (open to the internet). This is acceptable for a **learning/demo environment** — always restrict access to known IPs in production.

The green banner at the top confirms: **"Inbound security group rules successfully modified on security group."**

<img width="3839" height="2025" alt="Screenshot 2026-05-21 003839" src="https://github.com/user-attachments/assets/962905db-0e1c-45c2-97a5-3cbb1cb54326" />


---

## 🔧 Jenkins Setup

### Step 3: Jenkins Dashboard

After logging into Jenkins at `http://3.109.124.29:8080`, the dashboard shows the single pipeline job `production_setup`:

| Field | Value |
|-------|-------|
| Last Success | Build #12 — 58 sec ago |
| Last Failure | Build #11 — 2 min 35 sec ago |
| Last Duration | 46 sec |

<img width="3833" height="2023" alt="Screenshot 2026-05-21 003919" src="https://github.com/user-attachments/assets/a89cebad-6875-49e6-b385-503e35f97d2b" />

---

### Step 4: Jenkins Agent Node

Instead of running builds on the Jenkins master (Built-In Node), a dedicated **agent node** named `agent_kishore` was configured on a separate EC2 instance. This is a **DevOps best practice** — the master handles orchestration only, while agents do the actual build work.

#### Nodes Overview

Both nodes are visible in **Manage Jenkins → Nodes**:

| Node | Architecture | Free Disk | Clock | Response Time |
|------|-------------|-----------|-------|---------------|
| agent_kishore | Linux (amd64) | 10.85 GiB | In sync | 15ms |
| Built-In Node | Linux (amd64) | 10.44 GiB | In sync | 0ms |

> ⚠️ Both nodes show **0B Free Swap Space** and **1.86 GiB Free Temp Space** warnings — normal for a demo environment.

<img width="3827" height="2023" alt="Screenshot 2026-05-21 010903" src="https://github.com/user-attachments/assets/eb0d168d-fc16-4147-9da6-4b6766820bbe" />


#### Configuring the Agent via EC2 Instance Connect

The agent EC2 instance (`i-07d5f10c7cc25bd0e`, Public IP: `13.204.42.92`) was accessed via AWS EC2 Instance Connect. Maven was installed on it and the agent home directory (`~/agent_kishore`) was set up, containing:

- `caches/` — Maven/build caches
- `remoting/` — Jenkins remoting files
- `remoting.jar` — Agent connection JAR
- `workspace/` — Pipeline build workspace

<img width="3839" height="2024" alt="Screenshot 2026-05-21 010911" src="https://github.com/user-attachments/assets/be6addad-6409-4167-bce1-d712f2378d9e" />


---

### Step 5: Jenkins Credentials

All secrets are stored securely via **Manage Jenkins → Credentials → System → Global credentials**:

| Credential ID | Icon Type | Username/ID | Purpose |
|--------------|-----------|-------------|---------|
| `sonar` | Secret/Token | — | SonarQube authentication token |
| `nexus` | Username/Password | admin/****** | Nexus repository upload access |
| `tomcat` | Username/Password | — | Tomcat manager deployment |
| `email` | Username/Password | jenkins/****** | Jenkins SMTP sender account |
| `EMAIL_KISHORE` | Username/Password | kishorhc2004@gmail.com/****** | Personal Gmail for build notifications |

<img width="3839" height="2023" alt="Screenshot 2026-05-21 004057" src="https://github.com/user-attachments/assets/6f9b093f-2350-4abe-9fe4-f64d21ca2268" />


---

## 🔄 Pipeline Stages

The Jenkins pipeline `production_setup` has the following stages in order:

```
Start → git-clone → build → sonar-scan → expose to internet → tomcat → Post Actions → End
```

| Stage | Typical Duration | What Happens |
|-------|-----------------|--------------|
| `git-clone` | ~0.67–0.72s | Clones Spring PetClinic source from GitHub |
| `build` | ~21–41s | Runs `mvn clean package` to produce `.war` |
| `sonar-scan` | ~18–25s | Sends code to SonarQube for static analysis |
| `expose to internet` | ~0.7–1s | Network/port exposure verification step |
| `tomcat` | ~1s | Deploys `.war` to Tomcat via Manager API; also uploads to Nexus |
| `Post Actions` | ~3s | Sends build result email via Extended Email Plugin |

---

### Pipeline Job Status

The pipeline job page (`production_setup`) shows the **SonarQube Quality Gate badge** embedded directly:

- **Spring Framework Petclinic Quality Gate:** ✅ `Passed`
- **Server-side processing:** ✅ `Success`

The build history on the left shows multiple runs — a mix of ✅ successes and ❌ failures that occurred during iterative development. **Build #12 is the last stable build.**

<img width="3839" height="2023" alt="Screenshot 2026-05-21 011045" src="https://github.com/user-attachments/assets/0dc8b479-d29d-4d69-8a98-2f6afe7ea15f" />


---

### Jenkins Workspace

After a successful build, the pipeline workspace on the agent node contains the complete Spring PetClinic project (this is build #12's workspace):

```
workspace/
├── .devcontainer/
├── .git/
├── .github/
├── .mvn/
│   └── wrapper/
├── src/
├── .editorconfig         (192 B)
├── .gitattributes        (30 B)
├── .gitignore            (127 B)
├── LICENSE.txt           (11.09 KiB)
├── mvnw                  (9.55 KiB)
├── mvnw.cmd              (6.73 KiB)
├── pom.xml               (26.87 KiB)
└── readme.md             (11.94 KiB)
```

All files were checked out on **May 20, 2026, 5:15:21 PM**. The compiled `target/` directory is generated locally during the `build` stage and is not shown here as it is excluded via `.gitignore`.

<img width="3839" height="2015" alt="Screenshot 2026-05-21 004038" src="https://github.com/user-attachments/assets/06209cc0-45d0-42e5-b16b-a860df10ca54" />


---

### Stage View Builds

#### Build #15 — All Stages Passed (Ran on Built-In Jenkins Node)

Build #15 completed in **46 seconds** with all 6 stages green. The **Post Actions** stage (highlighted by the blue arrow) ran the Extended Email step on the **Jenkins** built-in node, sending the notification to `kishorhc2004@gmail.com`.

<img width="3839" height="2023" alt="Screenshot 2026-05-21 011045" src="https://github.com/user-attachments/assets/382e14df-bed5-4e32-8fb5-4f611d070897" />


---

#### Build #17 — All Stages Passed (Ran on agent_kishore Node)

Build #17 completed in **1 minute 15 seconds** with all 6 stages green. The Post Actions stage confirms it ran on **`agent_kishore`** — the dedicated agent EC2 instance — sending the notification to `kishorhc2004@gmail.com`.

<img width="3832" height="2027" alt="Screenshot 2026-05-21 011030" src="https://github.com/user-attachments/assets/1ce5b61a-c54f-4699-ba08-eae6adc32626" />


> **Key difference:** Build #15's Post Actions ran on `Jenkins` (built-in node), while Build #17's Post Actions ran on `agent_kishore`. This demonstrates both node configurations working correctly.

---

## 📊 Pipeline Results

### SonarQube Quality Gate

Accessible at `http://43.204.221.52:9000`, the SonarQube dashboard for **Spring Framework Petclinic** shows:

| Metric | Value |
|--------|-------|
| Version | 7.0.3 |
| Lines of Code | 9.7k |
| Quality Gate | ✅ **Passed** |
| New Issues | 0 (Required = 0) |
| Accepted Issues | 0 |
| Coverage | Not computed |
| Duplications | Not computed |
| Security Hotspots | 0 (Grade A) |
| New Code Since | May 20, 2026 |
| Last Analysis | 2 minutes ago |

> ℹ️ The embedded database warning is expected — SonarQube Community Edition uses H2 for demo purposes.


<img width="3839" height="2022" alt="Screenshot 2026-05-21 004122" src="https://github.com/user-attachments/assets/7776381e-5e1a-4c0b-a4be-536d57d40be1" />


---

### Nexus Artifact Repository

Accessible at `http://13.201.4.220:8081`, the Sonatype Nexus repository `sonar_update` stores the built artifact. The full artifact path in Nexus is:

```
org/
└── springframework/
    └── samples/
        └── spring-framework-petclinic/
            └── SNAPSHOT-1.0/
                ├── spring-framework-petclinic-SNAPSHOT-1.0.war       ← Main deployable artifact
                ├── spring-framework-petclinic-SNAPSHOT-1.0.war.md5   ← MD5 checksum
                └── spring-framework-petclinic-SNAPSHOT-1.0.war.sha1  ← SHA1 checksum
```

<img width="3839" height="2012" alt="Screenshot 2026-05-21 004128" src="https://github.com/user-attachments/assets/dffe3493-9b1a-4e27-995e-436afa8aaf37" />

---

### Application Deployment

The **Spring Framework PetClinic** application is successfully deployed to the Tomcat EC2 instance and is live and accessible in the browser at:

```
http://65.2.121.19:8080/petclinic/
```

The deployed application shows the full Spring PetClinic UI with navigation tabs: **HOME**, **FIND OWNERS**, **VETERINARIANS**, and **ERROR**.

<img width="3839" height="2024" alt="Screenshot 2026-05-21 004111" src="https://github.com/user-attachments/assets/92bbdf1d-ae0a-44c8-86bb-e1c689a7b7d0" />


---

### Email Notifications

The pipeline uses the **Extended Email Plugin** configured with Gmail SMTP (port 465) to send automated build notifications to `kishorhc2004@gmail.com`.

The Gmail inbox confirms notifications were received for every build:

| Build # | Status | Received |
|---------|--------|----------|
| #12 | ✅ BUILD SUCCESS | 12:39 AM |
| #11 | ❌ BUILD FAILED | 12:37 AM |
| #10 | ✅ BUILD SUCCESS | 12:30 AM |
| #8 | ✅ BUILD SUCCESS | 12:27 AM |
| #5 | ✅ Build SUCCESS — Application Deployed | 12:23 AM |
| — | Test email #5 (Jenkins SMTP config test) | 12:16 AM |

> The `Test email #5` at the bottom was sent during initial SMTP configuration testing from Jenkins.

<img width="3839" height="2026" alt="Screenshot 2026-05-21 004142" src="https://github.com/user-attachments/assets/e846ad55-2b1f-40ab-a6b5-5dbce069e9fb" />


---

## 📁 Sample Jenkinsfile Structure

Below is the representative structure of the `Jenkinsfile` used in the `production_setup` pipeline:

```groovy
pipeline {
    agent { label 'agent_kishore' }

    stages {

        stage('git-clone') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/<your-username>/spring-petclinic.git'
            }
        }

        stage('build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('sonar-scan') {
            steps {
                withCredentials([string(credentialsId: 'sonar', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        mvn sonar:sonar \
                          -Dsonar.projectKey=spring-petclinic \
                          -Dsonar.host.url=http://43.204.221.52:9000 \
                          -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('expose to internet') {
            steps {
                echo 'Verifying application port exposure...'
            }
        }

        stage('tomcat') {
            steps {
                // Upload artifact to Nexus
                withCredentials([usernamePassword(credentialsId: 'nexus',
                                  usernameVariable: 'NEXUS_USER',
                                  passwordVariable: 'NEXUS_PASS')]) {
                    sh '''
                        mvn deploy -DskipTests \
                          -DaltDeploymentRepository=nexus::default::http://13.201.4.220:8081/repository/sonar_update/
                    '''
                }
                // Deploy WAR to Tomcat
                withCredentials([usernamePassword(credentialsId: 'tomcat',
                                  usernameVariable: 'TOMCAT_USER',
                                  passwordVariable: 'TOMCAT_PASS')]) {
                    sh '''
                        curl -u $TOMCAT_USER:$TOMCAT_PASS \
                          -T target/*.war \
                          "http://65.2.121.19:8080/manager/text/deploy?path=/petclinic&update=true"
                    '''
                }
            }
        }
    }

    post {
        always {
            emailext(
                to: 'kishorhc2004@gmail.com',
                subject: "${currentBuild.result} - ${env.JOB_NAME} - #${env.BUILD_NUMBER}",
                body: """
                    Build ${currentBuild.result}!
                    Project:      ${env.JOB_NAME}
                    Build Number: ${env.BUILD_NUMBER}
                    Build URL:    ${env.BUILD_URL}
                """
            )
        }
    }
}
```

---

## 🔑 Key Learnings

- Setting up a **multi-node Jenkins environment** with a master + dedicated agent architecture
- Integrating **SonarQube** for automated code quality gates directly inside a CI pipeline
- Uploading versioned build artifacts to **Sonatype Nexus Repository Manager** using Maven
- Deploying a **WAR file** to Apache Tomcat via the Tomcat Manager REST API using `curl`
- Configuring **Extended Email Plugin** with Gmail SMTP for build success/failure notifications
- Managing **Jenkins credentials** securely (tokens, username/password, certificates) for multiple external services
- Setting up **AWS Security Groups** with the correct inbound rules for all services across multiple EC2 instances
- Understanding the difference between running builds on the **built-in Jenkins node vs a custom agent node**

---

## 👤 Author

<table>
  <tr>
    <td align="center">
      <b>Kishore HC</b><br/>
      DevOps Engineer<br/><br/>
      <a href="https://github.com/kishorehc">🐙 GitHub — kishorehc</a><br/>
      <a href="https://www.linkedin.com/in/kishore-h-c-847775287/">💼 LinkedIn — Kishore H C</a><br/>
      <a href="mailto:kishorhc2004@gmail.com">📧 kishorhc2004@gmail.com</a>
    </td>
  </tr>
</table>

---

> ⭐ If you found this project useful, consider giving the repository a star on GitHub!

---

*This project was built for learning and demonstrating end-to-end DevOps practices on AWS.
Security group rules are intentionally open (`0.0.0.0/0`) for demo purposes — always restrict ingress to known IPs in a production environment.*
