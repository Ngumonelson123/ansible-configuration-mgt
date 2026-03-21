# DEVOPS ORCHESTRATOR PROJECT — CI/CD Pipeline with Terraform, Jenkins, Ansible, Artifactory & SonarQube

## Stack
- **Jenkins** — CI/CD orchestration
- **Ansible** — Configuration management & deployments
- **Artifactory** — Binary artifact repository (JFrog Cloud)
- **SonarQube** — Code quality & security analysis
- **Nginx** — Reverse proxy for all tools
- **PHP TODO app** — Sample application pushed through the pipeline

---

## Architecture Overview

```
Developer pushes code to GitHub
            ↓
    Jenkins (php-todo pipeline)
            ↓
┌─────────────────────────────────────┐
│ 1. Initial Cleanup                  │
│ 2. Checkout SCM                     │
│ 3. Prepare Dependencies             │
│    - composer install               │
│    - php artisan migrate            │
│ 4. Execute Unit Tests (phpunit)     │
│ 5. Code Analysis (phploc)           │
│ 6. Package Artifact (zip)           │
│ 7. Upload to JFrog Artifactory      │
│ 8. Deploy to Dev ───────────────────┼──→ Triggers ansible-project pipeline
└─────────────────────────────────────┘
            ↓
    Jenkins (ansible-project pipeline)
            ↓
┌─────────────────────────────────────┐
│ 1. Initial Cleanup                  │
│ 2. Checkout SCM                     │
│ 3. SonarQube Analysis               │
│ 4. Quality Gate check               │
│ 5. Run Ansible Playbook             │
│ 6. Clean Workspace                  │
└─────────────────────────────────────┘
            ↓
    Ansible deploys to Dev servers:
    - tooling-dev-server  (Apache + PHP tooling app)
    - todo-dev-server     (Apache + Laravel todo app)
    - nginx-dev-server    (Nginx reverse proxy)
    - db-dev-server       (MySQL database)
```

---

## Quick Start

### 1. Provision infrastructure
```bash
cd terraform
terraform init
terraform apply -auto-approve
```

### 2. Populate inventory files from Terraform outputs
```bash
./scripts/update-inventory.sh
```

### 3. Test connectivity to all servers
```bash
./scripts/ping-all.sh
```

### 4. Configure CI environment (Jenkins + SonarQube + Artifactory + Nginx)
```bash
cd ansible
ansible-playbook playbooks/jenkins.yml     -i inventory/ci
ansible-playbook playbooks/sonarqube.yml   -i inventory/ci
ansible-playbook playbooks/artifactory.yml -i inventory/ci
ansible-playbook playbooks/nginx.yml       -i inventory/ci
```

### 5. Configure Dev environment (Tooling + TODO + DB + Nginx)
```bash
ansible-playbook playbooks/db.yml          -i inventory/dev
ansible-playbook playbooks/webservers.yml  -i inventory/dev
ansible-playbook playbooks/nginx.yml       -i inventory/dev
```

### 6. Or run everything at once
```bash
ansible-playbook playbooks/site.yml -i inventory/ci
ansible-playbook playbooks/site.yml -i inventory/dev
```

---

## Project Structure

```
project-14/
├── terraform/                        # AWS infrastructure
│   ├── versions.tf
│   ├── variables.tf
│   ├── main.tf
│   └── outputs.tf
├── ansible/
│   ├── ansible.cfg
│   ├── inventory/
│   │   ├── ci                        # CI environment hosts
│   │   ├── dev                       # Dev environment hosts
│   │   └── sit                       # SIT environment hosts
│   ├── playbooks/                    # Role-specific playbooks
│   ├── roles/
│   │   ├── common/                   # Shared tasks (apt, timezone, hosts)
│   │   ├── jenkins/                  # Jenkins installation
│   │   ├── sonarqube/                # SonarQube + PostgreSQL
│   │   ├── artifactory/              # JFrog Artifactory
│   │   ├── nginx/                    # Nginx reverse proxy
│   │   ├── webserver/                # Apache installation
│   │   ├── tooling/                  # Tooling PHP app deployment
│   │   ├── todo/                     # Laravel todo app deployment
│   │   └── mysql/                    # MySQL + database creation
│   └── deploy/
│       ├── Jenkinsfile               # ansible-project pipeline
│       └── sonar-project.properties  # SonarQube project config
└── scripts/
    ├── update-inventory.sh           # Auto-fill IPs after terraform apply
    └── ping-all.sh                   # Verify Ansible connectivity
```

---

## Step-by-Step Implementation

---

### STEP 1 — Provision AWS Infrastructure with Terraform

Terraform provisions all 8 EC2 instances needed for the project.

```bash
cd terraform
terraform init
terraform apply --auto-approve
```

**EC2 Instances created:**

| Server | Type | Role |
|---|---|---|
| jenkins-server | t3.medium | Jenkins CI/CD |
| sonarqube-server | t3.medium | Code quality analysis |
| artifactory-server | t3.medium | Artifact storage |
| nginx-ci-server | t2.micro | CI reverse proxy |
| nginx-dev-server | t2.micro | Dev reverse proxy |
| tooling-dev-server | t2.micro | Tooling PHP app |
| todo-dev-server | t2.micro | Laravel todo app |
| db-dev-server | t2.micro | MySQL database |

> **Screenshot:** AWS EC2 console showing all 8 instances in `Running` state with their assigned public IPs and availability zones — confirming successful Terraform provisioning.

---

### STEP 2 — Install and Configure Jenkins

Jenkins is installed on the `jenkins-server` via Ansible.

**Access Jenkins:**
```
http://<jenkins-public-ip>:8080
```

> **Screenshot:** Jenkins dashboard showing `Welcome to Jenkins!` — fresh installation on a new server at `44.192.100.90:8080`.

> **Screenshot:** Second Jenkins instance at `3.236.40.238:8080` used for the php-todo and ansible pipelines — Blue Ocean interface showing pipeline activity.

#### Jenkins Credentials Required:

```
Manage Jenkins → Credentials → System → Global → Add Credentials
```

| ID | Type | Purpose |
|---|---|---|
| `github` | Username + Token | Pull code from GitHub |
| `ansible-key` | SSH Private Key | Ansible SSH into managed nodes |
| `sonar-token` | Secret Text | SonarQube authentication |
| `jfrog-credentials` | Username/Password | Upload artifacts to JFrog |

> **Screenshot:** Jenkins Global Credentials page showing `sonar-token` (SonarQube token), `jfrog-credentials` (JFrog Artifactory credentials), and `github-token` (GitHub Credentials) — all three configured and ready.

---

### STEP 3 — Install and Configure SonarQube

SonarQube 7.9.3 is installed via Ansible on the `sonarqube-server`. The `sonarqube` role:
- Installs Java 11
- Installs PostgreSQL 14
- Creates `sonar` user and `sonarqube` database
- Downloads SonarQube to `/opt/sonarqube`
- Starts SonarQube as the `sonar` system user

```bash
ansible-playbook playbooks/sonarqube.yml -i inventory/ci
```

**Access SonarQube:**
```
http://<sonarqube-public-ip>:9000
Default credentials: admin / admin
```

> **Screenshot:** SonarQube Projects page showing `0 projects` — SonarQube running and accessible at `44.199.193.29:9000` before any scans have been pushed.

> **Screenshot:** SonarQube Projects page showing `php-todo` project with green **Passed** Quality Gate badge, last analyzed March 21, 2026 at 5:04 AM — confirming successful analysis from Jenkins.

#### Critical Fix — Remove Incompatible JavaScript Plugin

SonarQube 7.9.3's JavaScript plugin uses an old `cglib` version incompatible with Java 17 used by SonarScanner CLI 4.8+. This causes `ExceptionInInitializerError` and crashes the scan. Fix:

```bash
ssh ubuntu@<sonarqube-ip>

# Find and remove the plugin
sudo rm /opt/sonarqube/extensions/plugins/sonar-javascript-plugin-*.jar

# Clean stop and start
sudo -u sonar /opt/sonarqube/bin/linux-x86-64/sonar.sh stop
sleep 30
sudo -u sonar /opt/sonarqube/bin/linux-x86-64/sonar.sh start

# Verify
curl -s http://localhost:9000/api/system/status
# Expected: {"status":"UP"}
```

#### Configure SonarQube in Jenkins:

```
Manage Jenkins → Configure System → SonarQube servers
✓ Enable injection of SonarQube server configuration
Name: sonarqube
Server URL: http://<sonarqube-ip>:9000
Token: sonar-token
```

#### Install SonarQube Scanner Tool in Jenkins:

```
Manage Jenkins → Global Tool Configuration → SonarQube Scanner
Name: SonarQubeScanner
Version: 4.8.0.2856     ← compatible with SonarQube 7.9.3
Install automatically: ✓
```

#### Create Webhook for Instant Quality Gate Response:

```
SonarQube UI → Administration → Configuration → Webhooks → Create
Name: jenkins
URL: http://<jenkins-ip>:8080/sonarqube-webhook/
```

---

### STEP 4 — Install and Configure JFrog Artifactory

JFrog Artifactory Cloud (free 14-day trial) is used to store packaged build artifacts.

> **Screenshot:** JFrog landing page showing trial environment being created — hostname `triali5gtlv`, location `Amazon in N. Virginia`, provisioning at 48%.

> **Screenshot:** JFrog login page at `3.91.25.192:8082` — on-premise Artifactory instance login.

> **Screenshot:** JFrog user profile showing Identity Token generation page for API authentication.

#### Create Local Repository in Artifactory:

```
Artifactory → Repositories → Add Repository → Local
Package Type: Generic
Repository Key: php-todo-local
```

#### Configure Artifactory in Jenkins:

```
Manage Jenkins → Configure System → JFrog Platform Instances
Instance ID: artifactory-server
JFrog Platform URL: https://triali5gtlv.jfrog.io
Username: nelson@kewiscosacco.org
Password: ••••••••
→ Test Connection → "Found JFrog Artifactory 7.141.2"
```

> **Screenshot:** Jenkins Configure System showing JFrog instance details with URL `https://triali5gtlv.jfrog.io`, successful connection test showing "Found JFrog Artifactory 7.141.2".

---

### STEP 5 — Create php-todo Pipeline in Jenkins

#### Create Pipeline via Blue Ocean:

```
Jenkins → Open Blue Ocean → New Pipeline
→ Where do you store your code? GitHub
→ Connect: paste GitHub access token
→ Organization: Ngumonelson123
→ Repository: php-todo
→ Create Pipeline
```

> **Screenshot:** Blue Ocean "Create Pipeline" wizard — GitHub selected, organization `Ngumonelson123` confirmed, repository `php-todo` found and selected, ready to create.

> **Screenshot:** Blue Ocean showing php-todo pipeline with `Branch indexing` in progress — Jenkins scanning the repo for Jenkinsfile.

> **Screenshot:** Blue Ocean showing "You don't have any branches that contain a Jenkinsfile" — prompt to add Jenkinsfile to the php-todo repo.

#### php-todo Jenkinsfile:

```groovy
pipeline {
    agent any

    stages {
        stage('Initial Cleanup') {
            steps {
                dir("${WORKSPACE}") { deleteDir() }
            }
        }

        stage('Checkout SCM') {
            steps {
                git branch: 'main',
                    credentialsId: 'github',
                    url: 'https://github.com/Ngumonelson123/php-todo.git'
            }
        }

        stage('Prepare Dependencies') {
            steps {
                sh 'mv .env.sample .env'
                sh 'composer install'
                sh 'php artisan migrate'
                sh 'php artisan db:seed'
                sh 'php artisan key:generate'
            }
        }

        stage('Execute Unit Tests') {
            steps {
                sh './vendor/bin/phpunit'
            }
        }

        stage('Code Analysis') {
            steps {
                sh 'phploc app/ --log-csv build/logs/phploc.csv'
            }
        }

        stage('Package Artifact') {
            steps {
                sh 'zip -qr php-todo.zip ${WORKSPACE}/*'
            }
        }

        stage('Upload Artifact to Artifactory') {
            steps {
                script {
                    def server = Artifactory.server 'artifactory-server'
                    def uploadSpec = """{
                        "files": [{
                            "pattern": "php-todo.zip",
                            "target": "php-todo-local/php-todo",
                            "props": "type=zip;status=ready"
                        }]
                    }"""
                    server.upload spec: uploadSpec
                }
            }
        }

        stage('Deploy to Dev Environment') {
            steps {
                build job: 'ansible-project/main',
                      parameters: [[$class: 'StringParameterValue',
                                    name: 'env', value: 'dev']],
                      propagate: false, wait: true
            }
        }
    }
}
```

> **Screenshot:** php-todo Blue Ocean pipeline build #41 — ALL stages green: Initial Cleanup ✓, Checkout SCM ✓, Prepare Dependencies ✓, Execute Unit Tests ✓, Code Analysis ✓, Package Artifact ✓, Upload Artifact to Artifactory ✓, Deploy to Dev Environment ✓. Duration: 1m 1s. Shows "Triggered Builds: ansible-project #34 (42s)".

> **Screenshot:** php-todo activity page showing builds #27 through #41 all green — consistent successful runs after all fixes were applied.

---

### STEP 6 — Verify Artifact in JFrog Artifactory

After a successful pipeline run, the packaged artifact appears in Artifactory.

> **Screenshot:** JFrog Artifactory showing `php-todo-local/php-todo` artifact details — Name: `php-todo`, Size: 7.76 MB, Deployed By: `nelson@kewiscosacco.org`, Created: 20-03-26, SHA-256 checksum shown.

> **Screenshot:** Native browser view of `php-todo-local` repository at `triali5gtlv.jfrog.io/ui/native/php-todo-local/` showing `php-todo` file, timestamp `21-03-26 05:12:09`, size 7.8 MB with download link.

---

### STEP 7 — ansible-project Pipeline (SonarQube + Ansible Deploy)

This pipeline is triggered automatically by the php-todo pipeline's `Deploy to Dev Environment` stage.

#### ansible-project Jenkinsfile:

```groovy
pipeline {
    agent any

    parameters {
        string(name: 'env', defaultValue: 'dev', description: 'Deployment environment')
    }

    environment {
        ANSIBLE_CONFIG = "${WORKSPACE}/ansible.cfg"
        ANSIBLE_HOST_KEY_CHECKING = 'False'
    }

    stages {

        stage('Initial Cleanup') {
            steps {
                dir("${WORKSPACE}") { deleteDir() }
            }
        }

        stage('Checkout SCM') {
            steps {
                git branch: 'main',
                    credentialsId: 'github',
                    url: 'https://github.com/Ngumonelson123/ansible-configuration-mgt.git'
            }
        }

        stage('SonarQube Analysis') {
            environment {
                scannerHome = tool 'SonarQubeScanner'
            }
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh "${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=php-todo \
                        -Dsonar.projectName=php-todo \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=. \
                        -Dsonar.sourceEncoding=UTF-8 \
                        -Dsonar.php.exclusions=vendor/** \
                        -Dsonar.exclusions=**/*.js,**/*.ts,**/*.css \
                        -Dsonar.language=php"
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Run Ansible Playbook') {
            steps {
                script {
                    def inventory = params.env
                    ansiblePlaybook(
                        playbook: 'playbooks/site.yml',
                        inventory: "inventory/${inventory}",
                        credentialsId: 'ansible-key',
                        colorized: true,
                        installation: 'ansible'
                    )
                }
            }
        }

        stage('Clean Workspace after build') {
            steps {
                cleanWs(cleanWhenAborted: true, cleanWhenFailure: true,
                        cleanWhenNotBuilt: true, cleanWhenUnstable: true,
                        deleteDirs: true)
            }
        }
    }

    post {
        failure {
            echo 'Pipeline failed — check SonarQube Quality Gate or Ansible logs'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
    }
}
```

> **Screenshot:** ansible-project build #34 — ALL stages green: Initial Cleanup ✓, Checkout SCM ✓, SonarQube Analysis ✓, Quality Gate ✓, Run Ansible Playbook ✓, Clean Workspace ✓. Duration: 42s. Shows "Started by upstream pipeline php-todo/main build #41". Post step shows "Pipeline completed successfully!".

> **Screenshot:** ansible-project activity page showing build #21 (green) with Ansible playbook logs — tasks running across all dev servers: common role (apt cache, packages, timezone, hosts), MySQL role (install, start, create database, create user) — confirming full deployment to dev environment.

> **Screenshot:** ansible-project activity page showing the history of builds — runs #4 and #3 green (early working runs), then builds #5 through #20 red (during debugging), then build #21 green (fixed), all the way to builds #34+ consistently green.

---

### STEP 8 — Ansible Roles for Application Deployment

#### tooling role (`roles/tooling/tasks/main.yml`):

```yaml
---
- name: Remove existing tooling directory
  file:
    path: /var/www/html/tooling
    state: absent
  become: true

- name: Clone Tooling repository
  git:
    repo: "https://github.com/Ngumonelson123/tooling.git"
    dest: /var/www/html/tooling
    version: main
    force: yes
  become: true

- name: Set correct ownership for Apache
  file:
    path: /var/www/html/tooling
    owner: www-data
    group: www-data
    recurse: yes
    mode: "0755"
  become: true

- name: Deploy Apache virtual host config
  copy:
    dest: /etc/apache2/sites-available/tooling.conf
    content: |
      <VirtualHost *:80>
          DocumentRoot /var/www/html/tooling/html
          <Directory /var/www/html/tooling/html>
              Options Indexes FollowSymLinks
              AllowOverride All
              Require all granted
          </Directory>
      </VirtualHost>
  become: true
  notify: Restart Apache

- name: Enable tooling site
  command: a2ensite tooling.conf
  become: true
  notify: Restart Apache

- name: Disable default Apache site
  command: a2dissite 000-default.conf
  become: true
  notify: Restart Apache

- name: Enable Apache rewrite module
  command: a2enmod rewrite
  become: true
  notify: Restart Apache
```

> **Note:** `DocumentRoot` must be `/var/www/html/tooling/html` — the actual PHP files are in the `html/` subdirectory, not the repo root.
# DEVOPS ORCHESTRATOR PROJECT — CI/CD Pipeline with Terraform, Jenkins, Ansible, Artifactory & SonarQube

## Stack
- **Jenkins** — CI/CD orchestration
- **Ansible** — Configuration management & deployments
- **Artifactory** — Binary artifact repository (JFrog Cloud)
- **SonarQube** — Code quality & security analysis
- **Nginx** — Reverse proxy for all tools
- **PHP TODO app** — Sample application pushed through the pipeline

---

## Architecture Overview

```
Developer pushes code to GitHub
            ↓
    Jenkins (php-todo pipeline)
            ↓
┌─────────────────────────────────────┐
│ 1. Initial Cleanup                  │
│ 2. Checkout SCM                     │
│ 3. Prepare Dependencies             │
│    - composer install               │
│    - php artisan migrate            │
│ 4. Execute Unit Tests (phpunit)     │
│ 5. Code Analysis (phploc)           │
│ 6. Package Artifact (zip)           │
│ 7. Upload to JFrog Artifactory      │
│ 8. Deploy to Dev ───────────────────┼──→ Triggers ansible-project pipeline
└─────────────────────────────────────┘
            ↓
    Jenkins (ansible-project pipeline)
            ↓
┌─────────────────────────────────────┐
│ 1. Initial Cleanup                  │
│ 2. Checkout SCM                     │
│ 3. SonarQube Analysis               │
│ 4. Quality Gate check               │
│ 5. Run Ansible Playbook             │
│ 6. Clean Workspace                  │
└─────────────────────────────────────┘
            ↓
    Ansible deploys to Dev servers:
    - tooling-dev-server  (Apache + PHP tooling app)
    - todo-dev-server     (Apache + Laravel todo app)
    - nginx-dev-server    (Nginx reverse proxy)
    - db-dev-server       (MySQL database)
```

---

## Quick Start

### 1. Provision infrastructure
```bash
cd terraform
terraform init
terraform apply -auto-approve
```

### 2. Populate inventory files from Terraform outputs
```bash
./scripts/update-inventory.sh
```

### 3. Test connectivity to all servers
```bash
./scripts/ping-all.sh
```

### 4. Configure CI environment (Jenkins + SonarQube + Artifactory + Nginx)
```bash
cd ansible
ansible-playbook playbooks/jenkins.yml     -i inventory/ci
ansible-playbook playbooks/sonarqube.yml   -i inventory/ci
ansible-playbook playbooks/artifactory.yml -i inventory/ci
ansible-playbook playbooks/nginx.yml       -i inventory/ci
```

### 5. Configure Dev environment (Tooling + TODO + DB + Nginx)
```bash
ansible-playbook playbooks/db.yml          -i inventory/dev
ansible-playbook playbooks/webservers.yml  -i inventory/dev
ansible-playbook playbooks/nginx.yml       -i inventory/dev
```

### 6. Or run everything at once
```bash
ansible-playbook playbooks/site.yml -i inventory/ci
ansible-playbook playbooks/site.yml -i inventory/dev
```

---

## Project Structure

```
project-14/
├── terraform/                        # AWS infrastructure
│   ├── versions.tf
│   ├── variables.tf
│   ├── main.tf
│   └── outputs.tf
├── ansible/
│   ├── ansible.cfg
│   ├── inventory/
│   │   ├── ci                        # CI environment hosts
│   │   ├── dev                       # Dev environment hosts
│   │   └── sit                       # SIT environment hosts
│   ├── playbooks/                    # Role-specific playbooks
│   ├── roles/
│   │   ├── common/                   # Shared tasks (apt, timezone, hosts)
│   │   ├── jenkins/                  # Jenkins installation
│   │   ├── sonarqube/                # SonarQube + PostgreSQL
│   │   ├── artifactory/              # JFrog Artifactory
│   │   ├── nginx/                    # Nginx reverse proxy
│   │   ├── webserver/                # Apache installation
│   │   ├── tooling/                  # Tooling PHP app deployment
│   │   ├── todo/                     # Laravel todo app deployment
│   │   └── mysql/                    # MySQL + database creation
│   └── deploy/
│       ├── Jenkinsfile               # ansible-project pipeline
│       └── sonar-project.properties  # SonarQube project config
└── scripts/
    ├── update-inventory.sh           # Auto-fill IPs after terraform apply
    └── ping-all.sh                   # Verify Ansible connectivity
```

---

## Step-by-Step Implementation

---

### STEP 1 — Provision AWS Infrastructure with Terraform

Terraform provisions all 8 EC2 instances needed for the project.

```bash
cd terraform
terraform init
terraform apply --auto-approve
```

**EC2 Instances created:**

| Server | Type | Role |
|---|---|---|
| jenkins-server | t3.medium | Jenkins CI/CD |
| sonarqube-server | t3.medium | Code quality analysis |
| artifactory-server | t3.medium | Artifact storage |
| nginx-ci-server | t2.micro | CI reverse proxy |
| nginx-dev-server | t2.micro | Dev reverse proxy |
| tooling-dev-server | t2.micro | Tooling PHP app |
| todo-dev-server | t2.micro | Laravel todo app |
| db-dev-server | t2.micro | MySQL database |

> **Screenshot:** AWS EC2 console showing all 8 instances in `Running` state with their assigned public IPs and availability zones — confirming successful Terraform provisioning.

---

### STEP 2 — Install and Configure Jenkins

Jenkins is installed on the `jenkins-server` via Ansible.

**Access Jenkins:**
```
http://<jenkins-public-ip>:8080
```

> **Screenshot:** Jenkins dashboard showing `Welcome to Jenkins!` — fresh installation on a new server at `44.192.100.90:8080`.

> **Screenshot:** Second Jenkins instance at `3.236.40.238:8080` used for the php-todo and ansible pipelines — Blue Ocean interface showing pipeline activity.

#### Jenkins Credentials Required:

```
Manage Jenkins → Credentials → System → Global → Add Credentials
```

| ID | Type | Purpose |
|---|---|---|
| `github` | Username + Token | Pull code from GitHub |
| `ansible-key` | SSH Private Key | Ansible SSH into managed nodes |
| `sonar-token` | Secret Text | SonarQube authentication |
| `jfrog-credentials` | Username/Password | Upload artifacts to JFrog |

> **Screenshot:** Jenkins Global Credentials page showing `sonar-token` (SonarQube token), `jfrog-credentials` (JFrog Artifactory credentials), and `github-token` (GitHub Credentials) — all three configured and ready.

---

### STEP 3 — Install and Configure SonarQube

SonarQube 7.9.3 is installed via Ansible on the `sonarqube-server`. The `sonarqube` role:
- Installs Java 11
- Installs PostgreSQL 14
- Creates `sonar` user and `sonarqube` database
- Downloads SonarQube to `/opt/sonarqube`
- Starts SonarQube as the `sonar` system user

```bash
ansible-playbook playbooks/sonarqube.yml -i inventory/ci
```

**Access SonarQube:**
```
http://<sonarqube-public-ip>:9000
Default credentials: admin / admin
```

> **Screenshot:** SonarQube Projects page showing `0 projects` — SonarQube running and accessible at `44.199.193.29:9000` before any scans have been pushed.

> **Screenshot:** SonarQube Projects page showing `php-todo` project with green **Passed** Quality Gate badge, last analyzed March 21, 2026 at 5:04 AM — confirming successful analysis from Jenkins.

#### Critical Fix — Remove Incompatible JavaScript Plugin

SonarQube 7.9.3's JavaScript plugin uses an old `cglib` version incompatible with Java 17 used by SonarScanner CLI 4.8+. This causes `ExceptionInInitializerError` and crashes the scan. Fix:

```bash
ssh ubuntu@<sonarqube-ip>

# Find and remove the plugin
sudo rm /opt/sonarqube/extensions/plugins/sonar-javascript-plugin-*.jar

# Clean stop and start
sudo -u sonar /opt/sonarqube/bin/linux-x86-64/sonar.sh stop
sleep 30
sudo -u sonar /opt/sonarqube/bin/linux-x86-64/sonar.sh start

# Verify
curl -s http://localhost:9000/api/system/status
# Expected: {"status":"UP"}
```

#### Configure SonarQube in Jenkins:

```
Manage Jenkins → Configure System → SonarQube servers
✓ Enable injection of SonarQube server configuration
Name: sonarqube
Server URL: http://<sonarqube-ip>:9000
Token: sonar-token
```

#### Install SonarQube Scanner Tool in Jenkins:

```
Manage Jenkins → Global Tool Configuration → SonarQube Scanner
Name: SonarQubeScanner
Version: 4.8.0.2856     ← compatible with SonarQube 7.9.3
Install automatically: ✓
```

#### Create Webhook for Instant Quality Gate Response:

```
SonarQube UI → Administration → Configuration → Webhooks → Create
Name: jenkins
URL: http://<jenkins-ip>:8080/sonarqube-webhook/
```

---

### STEP 4 — Install and Configure JFrog Artifactory

JFrog Artifactory Cloud (free 14-day trial) is used to store packaged build artifacts.

> **Screenshot:** JFrog landing page showing trial environment being created — hostname `triali5gtlv`, location `Amazon in N. Virginia`, provisioning at 48%.

> **Screenshot:** JFrog login page at `3.91.25.192:8082` — on-premise Artifactory instance login.

> **Screenshot:** JFrog user profile showing Identity Token generation page for API authentication.

#### Create Local Repository in Artifactory:

```
Artifactory → Repositories → Add Repository → Local
Package Type: Generic
Repository Key: php-todo-local
```

#### Configure Artifactory in Jenkins:

```
Manage Jenkins → Configure System → JFrog Platform Instances
Instance ID: artifactory-server
JFrog Platform URL: https://triali5gtlv.jfrog.io
Username: nelson@kewiscosacco.org
Password: ••••••••
→ Test Connection → "Found JFrog Artifactory 7.141.2"
```

> **Screenshot:** Jenkins Configure System showing JFrog instance details with URL `https://triali5gtlv.jfrog.io`, successful connection test showing "Found JFrog Artifactory 7.141.2".

---

### STEP 5 — Create php-todo Pipeline in Jenkins

#### Create Pipeline via Blue Ocean:

```
Jenkins → Open Blue Ocean → New Pipeline
→ Where do you store your code? GitHub
→ Connect: paste GitHub access token
→ Organization: Ngumonelson123
→ Repository: php-todo
→ Create Pipeline
```

> **Screenshot:** Blue Ocean "Create Pipeline" wizard — GitHub selected, organization `Ngumonelson123` confirmed, repository `php-todo` found and selected, ready to create.

> **Screenshot:** Blue Ocean showing php-todo pipeline with `Branch indexing` in progress — Jenkins scanning the repo for Jenkinsfile.

> **Screenshot:** Blue Ocean showing "You don't have any branches that contain a Jenkinsfile" — prompt to add Jenkinsfile to the php-todo repo.

#### php-todo Jenkinsfile:

```groovy
pipeline {
    agent any

    stages {
        stage('Initial Cleanup') {
            steps {
                dir("${WORKSPACE}") { deleteDir() }
            }
        }

        stage('Checkout SCM') {
            steps {
                git branch: 'main',
                    credentialsId: 'github',
                    url: 'https://github.com/Ngumonelson123/php-todo.git'
            }
        }

        stage('Prepare Dependencies') {
            steps {
                sh 'mv .env.sample .env'
                sh 'composer install'
                sh 'php artisan migrate'
                sh 'php artisan db:seed'
                sh 'php artisan key:generate'
            }
        }

        stage('Execute Unit Tests') {
            steps {
                sh './vendor/bin/phpunit'
            }
        }

        stage('Code Analysis') {
            steps {
                sh 'phploc app/ --log-csv build/logs/phploc.csv'
            }
        }

        stage('Package Artifact') {
            steps {
                sh 'zip -qr php-todo.zip ${WORKSPACE}/*'
            }
        }

        stage('Upload Artifact to Artifactory') {
            steps {
                script {
                    def server = Artifactory.server 'artifactory-server'
                    def uploadSpec = """{
                        "files": [{
                            "pattern": "php-todo.zip",
                            "target": "php-todo-local/php-todo",
                            "props": "type=zip;status=ready"
                        }]
                    }"""
                    server.upload spec: uploadSpec
                }
            }
        }

        stage('Deploy to Dev Environment') {
            steps {
                build job: 'ansible-project/main',
                      parameters: [[$class: 'StringParameterValue',
                                    name: 'env', value: 'dev']],
                      propagate: false, wait: true
            }
        }
    }
}
```

> **Screenshot:** php-todo Blue Ocean pipeline build #41 — ALL stages green: Initial Cleanup ✓, Checkout SCM ✓, Prepare Dependencies ✓, Execute Unit Tests ✓, Code Analysis ✓, Package Artifact ✓, Upload Artifact to Artifactory ✓, Deploy to Dev Environment ✓. Duration: 1m 1s. Shows "Triggered Builds: ansible-project #34 (42s)".

> **Screenshot:** php-todo activity page showing builds #27 through #41 all green — consistent successful runs after all fixes were applied.

---

### STEP 6 — Verify Artifact in JFrog Artifactory

After a successful pipeline run, the packaged artifact appears in Artifactory.

> **Screenshot:** JFrog Artifactory showing `php-todo-local/php-todo` artifact details — Name: `php-todo`, Size: 7.76 MB, Deployed By: `nelson@kewiscosacco.org`, Created: 20-03-26, SHA-256 checksum shown.

> **Screenshot:** Native browser view of `php-todo-local` repository at `triali5gtlv.jfrog.io/ui/native/php-todo-local/` showing `php-todo` file, timestamp `21-03-26 05:12:09`, size 7.8 MB with download link.

---

### STEP 7 — ansible-project Pipeline (SonarQube + Ansible Deploy)

This pipeline is triggered automatically by the php-todo pipeline's `Deploy to Dev Environment` stage.

#### ansible-project Jenkinsfile:

```groovy
pipeline {
    agent any

    parameters {
        string(name: 'env', defaultValue: 'dev', description: 'Deployment environment')
    }

    environment {
        ANSIBLE_CONFIG = "${WORKSPACE}/ansible.cfg"
        ANSIBLE_HOST_KEY_CHECKING = 'False'
    }

    stages {

        stage('Initial Cleanup') {
            steps {
                dir("${WORKSPACE}") { deleteDir() }
            }
        }

        stage('Checkout SCM') {
            steps {
                git branch: 'main',
                    credentialsId: 'github',
                    url: 'https://github.com/Ngumonelson123/ansible-configuration-mgt.git'
            }
        }

        stage('SonarQube Analysis') {
            environment {
                scannerHome = tool 'SonarQubeScanner'
            }
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh "${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=php-todo \
                        -Dsonar.projectName=php-todo \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=. \
                        -Dsonar.sourceEncoding=UTF-8 \
                        -Dsonar.php.exclusions=vendor/** \
                        -Dsonar.exclusions=**/*.js,**/*.ts,**/*.css \
                        -Dsonar.language=php"
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Run Ansible Playbook') {
            steps {
                script {
                    def inventory = params.env
                    ansiblePlaybook(
                        playbook: 'playbooks/site.yml',
                        inventory: "inventory/${inventory}",
                        credentialsId: 'ansible-key',
                        colorized: true,
                        installation: 'ansible'
                    )
                }
            }
        }

        stage('Clean Workspace after build') {
            steps {
                cleanWs(cleanWhenAborted: true, cleanWhenFailure: true,
                        cleanWhenNotBuilt: true, cleanWhenUnstable: true,
                        deleteDirs: true)
            }
        }
    }

    post {
        failure {
            echo 'Pipeline failed — check SonarQube Quality Gate or Ansible logs'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
    }
}
```

> **Screenshot:** ansible-project build #34 — ALL stages green: Initial Cleanup ✓, Checkout SCM ✓, SonarQube Analysis ✓, Quality Gate ✓, Run Ansible Playbook ✓, Clean Workspace ✓. Duration: 42s. Shows "Started by upstream pipeline php-todo/main build #41". Post step shows "Pipeline completed successfully!".

> **Screenshot:** ansible-project activity page showing build #21 (green) with Ansible playbook logs — tasks running across all dev servers: common role (apt cache, packages, timezone, hosts), MySQL role (install, start, create database, create user) — confirming full deployment to dev environment.

> **Screenshot:** ansible-project activity page showing the history of builds — runs #4 and #3 green (early working runs), then builds #5 through #20 red (during debugging), then build #21 green (fixed), all the way to builds #34+ consistently green.

---

### STEP 8 — Ansible Roles for Application Deployment

#### tooling role (`roles/tooling/tasks/main.yml`):

```yaml
---
- name: Remove existing tooling directory
  file:
    path: /var/www/html/tooling
    state: absent
  become: true

- name: Clone Tooling repository
  git:
    repo: "https://github.com/Ngumonelson123/tooling.git"
    dest: /var/www/html/tooling
    version: main
    force: yes
  become: true

- name: Set correct ownership for Apache
  file:
    path: /var/www/html/tooling
    owner: www-data
    group: www-data
    recurse: yes
    mode: "0755"
  become: true

- name: Deploy Apache virtual host config
  copy:
    dest: /etc/apache2/sites-available/tooling.conf
    content: |
      <VirtualHost *:80>
          DocumentRoot /var/www/html/tooling/html
          <Directory /var/www/html/tooling/html>
              Options Indexes FollowSymLinks
              AllowOverride All
              Require all granted
          </Directory>
      </VirtualHost>
  become: true
  notify: Restart Apache

- name: Enable tooling site
  command: a2ensite tooling.conf
  become: true
  notify: Restart Apache

- name: Disable default Apache site
  command: a2dissite 000-default.conf
  become: true
  notify: Restart Apache

- name: Enable Apache rewrite module
  command: a2enmod rewrite
  become: true
  notify: Restart Apache
```

> **Note:** `DocumentRoot` must be `/var/www/html/tooling/html` — the actual PHP files are in the `html/` subdirectory, not the repo root.

#### todo role (`roles/todo/tasks/main.yml`):

```yaml
---
- name: Install Composer
  shell: |
    curl -sS https://getcomposer.org/installer | php
    mv composer.phar /usr/local/bin/composer
    chmod +x /usr/local/bin/composer
  args:
    creates: /usr/local/bin/composer
  become: true

- name: Clone php-todo repository
  git:
    repo: "https://github.com/Ngumonelson123/php-todo.git"
    dest: /var/www/html/php-todo
    version: main
    force: yes
  become: true

- name: Create .env from .env.sample
  copy:
    src: /var/www/html/php-todo/.env.sample
    dest: /var/www/html/php-todo/.env
    remote_src: yes
    force: no
  become: true

- name: Update .env DB settings
  lineinfile:
    path: /var/www/html/php-todo/.env
    regexp: "{{ item.regexp }}"
    line: "{{ item.line }}"
  loop:
    - { regexp: "^DB_HOST=", line: "DB_HOST=10.0.1.219" }
    - { regexp: "^DB_DATABASE=", line: "DB_DATABASE=homestead" }
    - { regexp: "^DB_USERNAME=", line: "DB_USERNAME=homestead" }
    - { regexp: "^DB_PASSWORD=", line: "DB_PASSWORD=sePret^i" }
  become: true

- name: Grant ubuntu ownership for composer install
  file:
    path: /var/www/html/php-todo
    owner: ubuntu
    group: ubuntu
    recurse: yes
  become: true

- name: Run composer install
  composer:
    command: install
    working_dir: /var/www/html/php-todo
    no_dev: false
  become: true
  become_user: ubuntu
  environment:
    HOME: /home/ubuntu
    COMPOSER_ALLOW_SUPERUSER: "0"

- name: Run artisan key generate
  command: php artisan key:generate
  args:
    chdir: /var/www/html/php-todo
  become: true

- name: Run artisan migrations
  command: php artisan migrate --force
  args:
    chdir: /var/www/html/php-todo
  become: true

- name: Set correct ownership for Apache
  file:
    path: /var/www/html/php-todo
    owner: www-data
    group: www-data
    recurse: yes
    mode: "0755"
  become: true
```

---

### STEP 9 — SonarQube Quality Gate Results

**sonar-project.properties** (in php-todo repo root):
```properties
sonar.projectKey=php-todo
sonar.projectName=php-todo
sonar.projectVersion=1.0
sonar.sources=.
sonar.sourceEncoding=UTF-8
sonar.php.exclusions=vendor/**
```

> **Screenshot:** SonarQube Projects page showing `php-todo` with green **Passed** Quality Gate badge. Metrics: Reliability A (0 bugs), Security A (0 vulnerabilities), Maintainability A (0 code smells). Last analysis: March 21, 2026 at 5:12 AM.

---

## Key Fixes and Lessons Learned

### Fix 1 — Git Clone: Permission Denied
**Error:** `fatal: could not create work tree dir '/var/www/html/tooling': Permission denied`  
**Cause:** Running git clone as `ubuntu` user who doesn't own `/var/www/html/`  
**Fix:** Run git clone as `root` (drop the `become_user` on git task), fix ownership after

### Fix 2 — Composer Must Not Run as Root
**Error:** `Do not run Composer as root/super user!`  
**Fix:** Use `become_user: ubuntu` with `HOME: /home/ubuntu` environment variable

### Fix 3 — Git Dubious Ownership
**Error:** `fatal: detected dubious ownership in repository`  
**Cause:** Directory previously owned by `www-data`, git runs as `root` — ownership mismatch  
**Fix:** Delete directory before every clone with `state: absent`

### Fix 4 — Composer Not Installed
**Error:** `Failed to find required executable composer in paths`  
**Fix:** Add composer install task to the role before the composer module task

### Fix 5 — composer.json Not Found
**Error:** `Composer could not find a composer.json file`  
**Discovery:** Tooling app is plain PHP — no composer needed. Remove composer tasks from tooling role entirely

### Fix 6 — SonarQube JavaScript Plugin Crash
**Error:** `java.lang.ExceptionInInitializerError` / cglib `InaccessibleObjectException`  
**Cause:** sonar-javascript-plugin 5.2.1 incompatible with Java 17 used by SonarScanner 4.8+  
**Fix:** Delete the plugin jar from `/opt/sonarqube/extensions/plugins/` and restart

### Fix 7 — Quality Gate Timeout
**Error:** `SonarQube task status is PENDING` → timeout after 1 minute  
**Fix:** Increase timeout to 5 minutes AND add SonarQube webhook pointing to Jenkins

### Fix 8 — Wrong Document Root
**Discovery:** Tooling app files are in `tooling/html/` not `tooling/`  
**Fix:** Set `DocumentRoot /var/www/html/tooling/html` in Apache virtual host

### Fix 9 — Nested Git Repository Warning
**Error:** `warning: adding embedded git repository: ansible`  
**Fix:** Add `ansible/` to `.gitignore` — the ansible folder has its own separate GitHub repo

### Fix 10 — Large Terraform Binary Pushed to GitHub
**Error:** `File terraform-provider-aws is 674.20 MB; this exceeds GitHub's file size limit`  
**Fix:**
```bash
git filter-branch --force --index-filter \
  "git rm -rf --cached --ignore-unmatch DevOps-Orchestrator/terraform/.terraform/" \
  --prune-empty --tag-name-filter cat -- --all
git push origin main --force
```
**Prevention:** Add `**/.terraform/`, `*.tfstate`, `*.tfstate.backup`, `*.pem` to `.gitignore`

---

## Access URLs

| Service | URL |
|---|---|
| Jenkins | `http://<jenkins-public-ip>:8080` |
| SonarQube | `http://<sonarqube-public-ip>:9000` |
| Artifactory | `https://<your-instance>.jfrog.io` |
| Tooling App | `http://<tooling-public-ip>` |
| TODO App | `http://<todo-public-ip>` |

---

## Default Credentials (change after first login)

| Tool | Username | Password |
|---|---|---|
| SonarQube | admin | admin |
| Artifactory | admin | password |
| Jenkins | admin | (set during setup wizard) |

---

## .gitignore (critical — never commit these)

```gitignore
**/.terraform/
*.tfstate
*.tfstate.backup
*.pem
ansible/
```

---

## Teardown

```bash
cd terraform
terraform destroy --auto-approve
```

> **Important:** Always destroy infrastructure when done to avoid unnecessary AWS charges.

#### todo role (`roles/todo/tasks/main.yml`):

```yaml
---
- name: Install Composer
  shell: |
    curl -sS https://getcomposer.org/installer | php
    mv composer.phar /usr/local/bin/composer
    chmod +x /usr/local/bin/composer
  args:
    creates: /usr/local/bin/composer
  become: true

- name: Clone php-todo repository
  git:
    repo: "https://github.com/Ngumonelson123/php-todo.git"
    dest: /var/www/html/php-todo
    version: main
    force: yes
  become: true

- name: Create .env from .env.sample
  copy:
    src: /var/www/html/php-todo/.env.sample
    dest: /var/www/html/php-todo/.env
    remote_src: yes
    force: no
  become: true

- name: Update .env DB settings
  lineinfile:
    path: /var/www/html/php-todo/.env
    regexp: "{{ item.regexp }}"
    line: "{{ item.line }}"
  loop:
    - { regexp: "^DB_HOST=", line: "DB_HOST=10.0.1.219" }
    - { regexp: "^DB_DATABASE=", line: "DB_DATABASE=homestead" }
    - { regexp: "^DB_USERNAME=", line: "DB_USERNAME=homestead" }
    - { regexp: "^DB_PASSWORD=", line: "DB_PASSWORD=sePret^i" }
  become: true

- name: Grant ubuntu ownership for composer install
  file:
    path: /var/www/html/php-todo
    owner: ubuntu
    group: ubuntu
    recurse: yes
  become: true

- name: Run composer install
  composer:
    command: install
    working_dir: /var/www/html/php-todo
    no_dev: false
  become: true
  become_user: ubuntu
  environment:
    HOME: /home/ubuntu
    COMPOSER_ALLOW_SUPERUSER: "0"

- name: Run artisan key generate
  command: php artisan key:generate
  args:
    chdir: /var/www/html/php-todo
  become: true

- name: Run artisan migrations
  command: php artisan migrate --force
  args:
    chdir: /var/www/html/php-todo
  become: true

- name: Set correct ownership for Apache
  file:
    path: /var/www/html/php-todo
    owner: www-data
    group: www-data
    recurse: yes
    mode: "0755"
  become: true
```

---

### STEP 9 — SonarQube Quality Gate Results

**sonar-project.properties** (in php-todo repo root):
```properties
sonar.projectKey=php-todo
sonar.projectName=php-todo
sonar.projectVersion=1.0
sonar.sources=.
sonar.sourceEncoding=UTF-8
sonar.php.exclusions=vendor/**
```

> **Screenshot:** SonarQube Projects page showing `php-todo` with green **Passed** Quality Gate badge. Metrics: Reliability A (0 bugs), Security A (0 vulnerabilities), Maintainability A (0 code smells). Last analysis: March 21, 2026 at 5:12 AM.

---

## Key Fixes and Lessons Learned

### Fix 1 — Git Clone: Permission Denied
**Error:** `fatal: could not create work tree dir '/var/www/html/tooling': Permission denied`  
**Cause:** Running git clone as `ubuntu` user who doesn't own `/var/www/html/`  
**Fix:** Run git clone as `root` (drop the `become_user` on git task), fix ownership after

### Fix 2 — Composer Must Not Run as Root
**Error:** `Do not run Composer as root/super user!`  
**Fix:** Use `become_user: ubuntu` with `HOME: /home/ubuntu` environment variable

### Fix 3 — Git Dubious Ownership
**Error:** `fatal: detected dubious ownership in repository`  
**Cause:** Directory previously owned by `www-data`, git runs as `root` — ownership mismatch  
**Fix:** Delete directory before every clone with `state: absent`

### Fix 4 — Composer Not Installed
**Error:** `Failed to find required executable composer in paths`  
**Fix:** Add composer install task to the role before the composer module task

### Fix 5 — composer.json Not Found
**Error:** `Composer could not find a composer.json file`  
**Discovery:** Tooling app is plain PHP — no composer needed. Remove composer tasks from tooling role entirely

### Fix 6 — SonarQube JavaScript Plugin Crash
**Error:** `java.lang.ExceptionInInitializerError` / cglib `InaccessibleObjectException`  
**Cause:** sonar-javascript-plugin 5.2.1 incompatible with Java 17 used by SonarScanner 4.8+  
**Fix:** Delete the plugin jar from `/opt/sonarqube/extensions/plugins/` and restart

### Fix 7 — Quality Gate Timeout
**Error:** `SonarQube task status is PENDING` → timeout after 1 minute  
**Fix:** Increase timeout to 5 minutes AND add SonarQube webhook pointing to Jenkins

### Fix 8 — Wrong Document Root
**Discovery:** Tooling app files are in `tooling/html/` not `tooling/`  
**Fix:** Set `DocumentRoot /var/www/html/tooling/html` in Apache virtual host

### Fix 9 — Nested Git Repository Warning
**Error:** `warning: adding embedded git repository: ansible`  
**Fix:** Add `ansible/` to `.gitignore` — the ansible folder has its own separate GitHub repo

### Fix 10 — Large Terraform Binary Pushed to GitHub
**Error:** `File terraform-provider-aws is 674.20 MB; this exceeds GitHub's file size limit`  
**Fix:**
```bash
git filter-branch --force --index-filter \
  "git rm -rf --cached --ignore-unmatch DevOps-Orchestrator/terraform/.terraform/" \
  --prune-empty --tag-name-filter cat -- --all
git push origin main --force
```
**Prevention:** Add `**/.terraform/`, `*.tfstate`, `*.tfstate.backup`, `*.pem` to `.gitignore`

---

## Access URLs

| Service | URL |
|---|---|
| Jenkins | `http://<jenkins-public-ip>:8080` |
| SonarQube | `http://<sonarqube-public-ip>:9000` |
| Artifactory | `https://<your-instance>.jfrog.io` |
| Tooling App | `http://<tooling-public-ip>` |
| TODO App | `http://<todo-public-ip>` |

---

## Default Credentials (change after first login)

| Tool | Username | Password |
|---|---|---|
| SonarQube | admin | admin |
| Artifactory | admin | password |
| Jenkins | admin | (set during setup wizard) |

---

## .gitignore (critical — never commit these)

```gitignore
**/.terraform/
*.tfstate
*.tfstate.backup
*.pem
ansible/
```

---

## Teardown

```bash
cd terraform
terraform destroy --auto-approve
```

> **Important:** Always destroy infrastructure when done to avoid unnecessary AWS charges.
