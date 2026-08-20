# Jenkins & CI

## What Is CI? — Continuous Integration

**CI** is the practice of *continuously integrating* every developer's code into a tested, deployable **artifact** — automatically. The moment a commit lands in git, a pipeline is triggered that **clones → compiles → scans → unit tests → builds** the code into an artifact. It's the automation layer that sits on top of git (see [[git]]): git stores the code, CI turns each change into something you can ship.

The whole point is to catch problems *early*, when they're cheap. Errors show up at three levels, and CI is designed to fail at the earliest one:

| Error type | What broke | Where CI catches it |
|------------|-----------|---------------------|
| **Functionality error** | A code defect — the logic is wrong | Unit tests |
| **Build error** | Config error — it won't compile/package | Build stage |
| **Deployment error** | Fails when deployed across environments | Deploy stage |

### Shift-left

**Shift-left** means moving scans and tests **before** deployment, not after. You fail fast on a broken commit instead of discovering it in production where it's expensive to fix. Every gate — clone, build, scan, test — runs *before* an artifact is ever promoted:

```
clone → install deps → authenticate → build → scan → unit test → (artifact) → deploy
```

## Jenkins — A Web Server Made of Plugins

**Jenkins is a plain web server.** On its own it does almost nothing — **plugins add everything else**. This is the single most important thing to understand about it: you install a base Jenkins, then bolt on exactly the capabilities you need.

| Plugin category | Examples | Adds the ability to… |
|-----------------|----------|----------------------|
| **SCM** | Git | Pull source code |
| **Build** | Maven, Gradle, npm | Compile/package apps |
| **Notification** | Slack, Email | Report build results |
| **Cloud** | AWS, Azure | Authenticate & deploy to cloud |
| **Orchestration** | Kubernetes | Run builds in K8s pods |
| **Credentials** | Credentials management | Store secrets safely |

You add or remove plugins whenever you want — the server grows to fit the job.

### Plugins the RoboShop Pipelines Actually Use

Every non-trivial line in `nodejsEKSMain.groovy` / `nodejsEKSPipeline.groovy` traces back to one of these:

| Plugin | Adds | Used for |
|--------|------|----------|
| **Pipeline: AWS Steps** | `withAWS { }` | Every AWS/EKS/ECR block — injects credentials into the step instead of exporting them as shell env vars |
| **Generic Webhook Trigger** | `triggers { GenericTrigger(...) }` | Lets Jira Automation start a build over HTTP with a token — no GitHub push involved |
| **JIRA Steps** (`jira-steps-plugin`) | `jiraNewIssue`, `jiraGetIssue`, `jiraGetIssueTransitions`, `jiraTransitionIssue`, `jiraGetFields` | Every Jira call in `utils.groovy` |
| **Slack Notification** | `slackSend(...)` | `post { success/failure }` build alerts |
| **SonarQube Scanner** | `withSonarQubeEnv`, `waitForQualityGate()` | Static analysis + quality gate (see below) |
| **Pipeline Utility Steps** | `readJSON file: 'package.json'` | Reading the app version straight out of the repo |
| **Multibranch Scan Webhook Trigger** | A webhook endpoint on the Multibranch Pipeline job itself | Re-triggers the branch/PR scan on a push, without relying on GitHub's own webhook integration (see below) |

### Credentials RoboShop Needs

| Credential ID | Kind | Used by | Purpose |
|---------------|------|---------|---------|
| `aws-creds` | AWS Credentials | every `withAWS` block | EKS/ECR access |
| `github-token` | Secret text | `utils.groovy` (commit status, tag, PR), `library-scan` | GitHub API calls — see generation steps below |
| `slack-token` | Secret text | `post` blocks' `slackSend` | Slack bot token — see Slack app steps below |
| `jira-secret` | Secret text | `triggers { GenericTrigger(...) }` | Validates the `?token=...` on the incoming Jira webhook URL |
| `ssh-auth` | SSH Username with private key | Jenkins controller → agent node | Lets the master SSH in as `ec2-user` to launch the `ROBOSHOP`-labelled agent — see Master–Agent Architecture below |
| `sonar-creds` | Secret text | SonarQube server config | The Sonar authentication token — selected (not pasted raw) under **Manage Jenkins → System → SonarQube servers** |
| `jira-creds` | Username with password | JIRA Steps site config, via the `JIRA_SITE` env var | **Username** = Jira account email, **Password** = Jira API token (not the account password) — Jira rejects ticket creation without this exact shape. Selected once under **Manage Jenkins → System → JIRA Steps**, naming the site (e.g. `roboshop-jira`) and its URL |

Installation is a short script (RHEL-family): add the Jenkins repo, install Java (Jenkins runs on the JVM) and Jenkins itself, then start and enable the service.

```bash
# from session-67 notes — install, don't memorise
curl -o /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/rpm-stable/jenkins.repo
yum install fontconfig java-21-openjdk -y     # Jenkins needs a JDK
yum install jenkins -y
systemctl enable jenkins --now
```

### The three phases of any job

Every Jenkins job, however it's built, has the same shape:

- **Pre-build** — set up: environment variables, options, parameters.
- **Build** — the actual work: how you build/test/deploy your application.
- **Post-build** — what to do afterwards, usually **notifications** (success/failure).

## Master–Agent Architecture

Jenkins runs as a **master (controller)** with one or more **agent nodes**.

- The **master never runs builds itself.** It receives the trigger, decides *which agent* should run the build, hands it off, then **monitors the node and collects the logs**.
- **Agent nodes** do the real work. You keep **separate nodes for different stacks** — a node configured for Java, another for NodeJS, another for iOS/Android — each with the right tools installed.

Connecting an agent node (**Manage Jenkins → Nodes → New Node**, launch method **Launch agents via SSH**) needs an SSH credential: **Kind** `SSH Username with private key`, **Username** `ec2-user` (the default login user on Amazon Linux/RHEL EC2 instances), stored as the `ssh-auth` credential. The controller uses this to SSH into the agent box and start the agent process — no manual login required once it's configured.

```
        commit → trigger
                   │
              ┌────▼────┐
              │ MASTER  │   assigns work, watches nodes, gathers logs
              └──┬───┬──┘
                 │   │
          ┌──────▼┐ ┌▼──────┐
          │ Java  │ │NodeJS │   agents actually build
          │ node  │ │ node  │
          └───────┘ └───────┘
```

Agents are targeted by **label**. In the roboshop pipelines the agent is pinned to a `ROBOSHOP` label:

```groovy
// jenkins-shared-library/vars/nodejsEKSPipeline.groovy
agent {
    node { label 'ROBOSHOP' }   // run only on nodes tagged ROBOSHOP
}
```

## Freestyle vs Pipeline Jobs

Jenkins offers two kinds of jobs. **Freestyle** is the old click-through style; **Pipeline** is code. Real teams use Pipeline.

| | **Freestyle** | **Pipeline** |
|---|---------------|--------------|
| Defined in | Jenkins **UI** (clicks) | A **`Jenkinsfile`** in the repo |
| Version control | None | **Versioned in git** |
| Review changes | Can't | Via **Pull Request** |
| Restore / rollback | Can't | Yes — it's just code |
| Reuse | No | Yes (shared libraries) |
| Complex logic / visualisation | Poor | Full control + stage view |
| Track who changed what | Hard | Git history |

The Freestyle problem in one line: **everything lives in the UI** — no version control, no easy restore, no audit trail, no reuse. Pipeline fixes all of that by treating the build definition as **code that lives beside the app**.

## Multibranch Pipelines & the Scan Webhook

A **Multibranch Pipeline** job doesn't point at one branch — it scans a whole repo and auto-creates a sub-job per branch/PR that has a `Jenkinsfile`, and removes the job again when the branch disappears. That's what produces job paths like `ROBOSHOP/catalogue/main`, and what makes `env.BRANCH_NAME` meaningful in a `Jenkinsfile` (used further down to split `main` from feature branches).

1. Jenkins → **New Item** → **Multibranch Pipeline**, named e.g. `catalogue` inside a `ROBOSHOP` folder.
2. **Branch Sources → Git**: repository URL (e.g. `https://github.com/90s-org/catalogue.git`), credentials, **Discover branches**.
3. **Build Configuration**: by Jenkinsfile, path `Jenkinsfile` — the one-liner that calls `nodejsEKSPipeline`/`nodejsEKSMain`.
4. **Scan Multibranch Pipeline Triggers** → check the trigger the **Multibranch Scan Webhook Trigger** plugin adds, and set a token. That gives the job its own webhook endpoint — `http://jenkins.daws90s.shop:8080/multibranch-webhook-trigger/invoke?token=<token>` — that GitHub (or anything) can POST to, to re-trigger the branch/PR scan on demand instead of waiting on a poll interval. This is the generic-SCM equivalent of GitHub's own push-triggered rescan, without needing a GitHub-specific branch source plugin.

**Why the branch jobs are read-only:** because a branch-level job under a Multibranch Pipeline is entirely *computed* from the repo scan, its Configure page in the UI is locked — you can't tick "Trigger builds remotely" by hand, even as an admin. This matters again later: it's exactly why the release pipeline's Jira trigger has to be declared inside the `Jenkinsfile` itself (`triggers { GenericTrigger(...) }`) rather than clicked in through the UI.

## Declarative vs Scripted Pipelines

A Pipeline can be written in two syntaxes, both Groovy-based.

| | **Scripted** | **Declarative** |
|---|--------------|-----------------|
| Age | Old way | New way |
| Syntax | Full Groovy (Java-like) | Structured, opinionated blocks |
| Error checking | Compiled **at run time** — errors surface mid-build | **Validated before execution** — fails fast on syntax |
| Control | Maximum flexibility | Easier, safer, enough for most jobs |

**Default to declarative.** You reach for scripted only when you genuinely need arbitrary Groovy control flow.

## Jenkinsfile Syntax — The Declarative Skeleton

A declarative `Jenkinsfile` is a `pipeline { }` block with a fixed set of sections. This is the roboshop teaching pipeline, section by section:

```groovy
// jenkins/Jenkinsfile
pipeline {
    agent { node { label 'ROBOSHOP' } }     // WHERE it runs

    environment {                            // variables for every stage
        COURSE = "Jenkins"
    }

    options {                                // pipeline-wide behaviour
        disableConcurrentBuilds()            // no two builds of this job at once
        timeout(time: 15, unit: 'MINUTES')   // kill if it hangs
    }

    parameters {                             // inputs asked at build time
        string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: '...')
        booleanParam(name: 'DEPLOY', defaultValue: true, description: '...')
        choice(name: 'CHOICE', choices: ['One','Two','Three'], description: '...')
        password(name: 'PASSWORD', defaultValue: 'SECRET', description: '...')
    }

    stages {                                 // the actual work, in order
        stage('Build')  { steps { sh 'echo Building' } }
        stage('Test')   { steps { sh 'echo Testing'  } }
        stage('Deploy') {
            when { expression { "${params.DEPLOY}" == "true" } }  // conditional stage
            steps { sh 'echo Deploying' }
        }
    }

    post {                                   // runs after all stages
        always  { echo 'runs no matter what' }
        success { echo 'runs on success' }
        failure { echo 'runs on failure' }   // e.g. notify Slack here
    }
}
```

| Section | Purpose |
|---------|---------|
| `agent` | Which node/label runs the pipeline |
| `environment` | Env vars available to all stages |
| `options` | Pipeline-wide settings (timeout, no concurrent builds) |
| `parameters` | Runtime inputs (`string`, `boolean`, `choice`, `password`, `text`) |
| `stages` / `stage` / `steps` | The ordered units of work |
| `when` | Run a stage only if a condition holds |
| `post` | `always` / `success` / `failure` blocks — usually notifications |

## The Catalogue Pipeline — Building & Pushing to ECR

The `catalogue` component's pipeline is the real end-to-end example. It reads the app version, installs, tests, scans, builds a Docker image, and pushes it to **ECR** — each stage a gate.

### Read the version from `package.json`

The image tag is the **app version**, read straight out of the code so the build and the artifact always agree:

```groovy
// jenkins-shared-library/vars/nodejsEKSPipeline.groovy
stage('Read version') {
    steps { script {
        def packageJson = readJSON file: 'package.json'
        appVersion = packageJson.version        // e.g. 1.0.0
    }}
}
```

### Authenticate to AWS, build, push

AWS credentials are injected by the credentials plugin (`withAWS`), never hard-coded. Log in to ECR, build, push:

```groovy
stage('Docker Build') {
    steps { script {
        withAWS(credentials: 'aws-creds', region: 'us-east-1') {
            sh """
                aws ecr get-login-password --region us-east-1 \
                  | docker login --username AWS --password-stdin ${acc_id}.dkr.ecr.us-east-1.amazonaws.com
                docker build -t ${acc_id}.dkr.ecr.us-east-1.amazonaws.com/${project}/${component}:${appVersion} .
            """
        }
    }}
}
```

**Credentials** in Jenkins (SSH keys, AWS creds, GitHub tokens) are stored centrally and referenced by ID (`aws-creds`, `github-token`) — the secret never appears in the `Jenkinsfile`.

## SonarQube — Static Code Analysis & Quality Gates

**SonarQube** scans your **source code without running it** — two things at once:

- **SAST (Static Application Security Testing)** — security loopholes in the code.
- **Static code analysis** — coding-standard and maintainability issues.

It runs on its own server (web UI on **port 9000**); Jenkins runs a **scanner** that ships the code to it and waits for the verdict.

### The three things Sonar finds

| Finding | Meaning | Consequence |
|---------|---------|-------------|
| **Bug** | Code that sometimes works, sometimes doesn't — unexpected behaviour | May break in production |
| **Vulnerability** | A security loophole an attacker could exploit | Security incident |
| **Code smell** | Maintainability issue — works today, hard to understand/change | Rots over time |

Sonar rolls these up into **ratings** — Security, Reliability, Maintainability — each graded **A** (best) down. Plus **Code Coverage**, which comes from unit testing (below).

### New code vs overall code

Sonar reports on two scopes, and the distinction drives everything:

```
Commits:  C1  C2  C3  C4
Overall code = C1 + C2 + C3 + C4   (the whole codebase)
New code     = C4 − C3             (only what this change added)
```

You almost always **gate on new code first** — you can't force a huge legacy codebase to A-grade overnight, but you *can* insist every **new** commit is clean.

### Quality gates

A **quality gate** is a pass/fail threshold. The Jenkins scanner submits the analysis, then **waits for the gate**; if it fails, the pipeline fails:

```groovy
// grounded in the commented SonarQube block, nodejsEKSPipeline.groovy
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('sonar-server') {
            sh "${tool 'sonar-8'}/bin/sonar-scanner"
        }
    }
}
stage('SonarQube Quality Gate') {
    steps { timeout(time: 10, unit: 'MINUTES') { script {
        def qg = waitForQualityGate()          // pause until Sonar answers
        if (qg.status != 'OK') {
            error "Pipeline aborted: ${qg.status}"   // fail the build
        }
    }}}
}
```

The scanner is configured by a `sonar-project.properties` file in the repo — what to scan, what to exclude, where the coverage report lives:

```properties
# catalogue/sonar-project.properties
sonar.projectKey=roboshop-catalogue
sonar.sources=.
sonar.exclusions=node_modules/**,Jenkinsfile,Dockerfile,coverage/**,test/**
sonar.javascript.lcov.reportPaths=coverage/lcov.info   # coverage feeds the gate
sonar.junit.reportPaths=junit.xml
```

### Wiring the Sonar Server Into Jenkins

Sonar and Jenkins talk **both ways**: Jenkins pushes the analysis *to* Sonar, and Sonar reports the quality-gate result *back* to Jenkins via a webhook — `waitForQualityGate()` blocks waiting for that callback rather than polling, so the callback has to actually be configured or the stage just hangs until its own timeout.

1. Run the SonarQube server itself (own VM/container), reachable at e.g. `http://sonar.daws90s.shop:9000`.
2. In Sonar: **My Account → Security → Generate Token** — this is what Jenkins authenticates with.
3. In Jenkins: **Manage Jenkins → Credentials** → add the token from step 2 as a **Secret text** credential, ID `sonar-creds` — don't paste it raw into the server config, store it as a credential like everything else.
4. **Manage Jenkins → System → SonarQube servers** → add one. **Name** must match the pipeline's `withSonarQubeEnv('sonar-server')` exactly; **Server URL**, and **Server authentication token**: select the `sonar-creds` credential.
5. **Manage Jenkins → Tools → SonarQube Scanner installations** → add one named `sonar-8` (matches `tool 'sonar-8'` in the pipeline), pick a version, let Jenkins auto-install it.
6. In Sonar: **Administration → Configuration → Webhooks** → add one pointing back at Jenkins: `http://jenkins.daws90s.shop:8080/sonarqube-webhook/`. Without this step, `waitForQualityGate()` never hears back and the stage just times out after 10 minutes instead of getting a real answer.

### Rolling Sonar out without a revolt — the phased approach

You don't switch on strict gates on day one; you'd block every team. The real-world rollout is **phased**, easing quality up so developers can keep pace:

| Phase | What you do | Pipeline fails? |
|-------|-------------|-----------------|
| **1 (≈2 months)** | Onboard all projects — create Sonar projects, give devs access. Escalate laggards to TLs/managers. | No |
| **2** | Turn on gates for **new code** (issues/bugs/vulns/smells = 0, coverage ≥ 40%) — visible but non-blocking, then start failing at phase end. | Then yes |
| **3** | Extend gates to **overall code**, then ratchet coverage **50 → 60 → 70 → 80%**. | Yes |

## Unit Testing & Code Coverage

The **basic block of programming is a function** — so the smallest test is a **unit test** of a single function.

**Unit tests are written by developers.** They test **one function in isolation** and **mock every dependency** (the DB, other services) as succeeding — they don't care whether Mongo is up. There's **no deployment**: you're testing raw code.

```
login(username, password) {
    connectToDb()      ← mocked as success
    checkUsername()    ← the logic under test
    checkPassword()
}
```

**Code coverage** = the % of code lines exercised by unit tests. It's the number Sonar reads to enforce the coverage gate:

```
10 functions, 0 tested  → 0%
10 functions, 1 tested  → 10%
10 functions, all tested → 100%
```

In the pipeline it's a single stage — the `npm test` run also emits the coverage and JUnit reports Sonar later consumes:

```groovy
// nodejsEKSPipeline.groovy — npm test produces coverage/lcov.info + junit.xml
stage('Unit tests') { steps { script { sh "npm test" } } }
```

```json
// catalogue/package.json — the test script that generates coverage
"scripts": { "test": "jest --testPathPattern=test/ --coverage --forceExit" }
```

### Levels of testing

| Level | What it checks |
|-------|----------------|
| **Unit testing** | A single function, dependencies mocked |
| **API / component testing** | One whole component (e.g. just `catalogue`) |
| **Integration testing** | Multiple components together (e.g. `cart` calling `catalogue`) |

## Dependency (Library) Scanning

Your app is mostly **other people's code** — npm/pip/maven libraries. A vulnerability in a dependency is *your* vulnerability. **Dependency scanning** checks your libraries against known-vulnerability databases.

- The data comes from the **NVD (National Vulnerability Database)** — a non-profit effort backed by the US Gov, universities, and big tech, fed by **bounty** researchers who find and report new loopholes.
- Tools: **GitHub Dependabot** (used here), **Nexus IQ**, **Blackduck**.

Two terms Sonar/scanners speak in:

| Term | Meaning |
|------|---------|
| **CVE** (Common Vulnerabilities and Exposures) | The unique ID for one specific vulnerability |
| **CVSS** (Common Vulnerability Scoring System) | The scoring system that rates its severity |

Severities: **Low · Medium · High · Critical**. The rule the pipeline enforces: **check for High/Critical dependency alerts; if any exist, break the pipeline.**

In the catalogue pipeline this is a stage that hits the **GitHub Dependabot API** with a stored token, counts High/Critical alerts, and fails on any:

```groovy
// nodejsEKSPipeline.groovy — Check Dependabot Alerts (condensed)
withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
    sh '''
        curl -s -H "Authorization: Bearer ${GH_TOKEN}" \
          "https://api.github.com/repos/${org}/${component}/dependabot/alerts?state=open" -o alerts.json

        HIGH_CRITICAL=$(jq '[.[] | select(.security_vulnerability.severity=="high" or =="critical")] | length' alerts.json)
        [ "$HIGH_CRITICAL" -gt 0 ] && { echo "Found $HIGH_CRITICAL High/Critical alerts"; exit 1; }
    '''
}
```

(A real alert from the repo: the `redis` npm package flagged `CVE-2021-29469`, severity **high**, fixed in `3.1.1`.)

## Image Scanning with Trivy

Dependabot scans your *libraries*; **Trivy** scans your *container* — one simple CLI covering three surfaces:

1. **`package.json`** — app dependencies
2. **Dockerfile** — misconfigurations
3. **Image OS packages** — vulns in the base image

The rule is the same as Sonar/Dependabot: **HIGH/CRITICAL ⇒ fail.** `--exit-code 1` makes Trivy return non-zero so the pipeline stops:

```bash
# session-70 notes
trivy config --exit-code 1 --severity HIGH,CRITICAL --format table ./Dockerfile
trivy image  --scanners vuln --pkg-types os --exit-code 1 --severity HIGH,CRITICAL \
    --format table 160885265516.dkr.ecr.us-east-1.amazonaws.com/roboshop/catalogue:1.0.0
```

In the pipeline both scans run, and *either* failing fails the stage:

```groovy
// nodejsEKSPipeline.groovy — returnStatus captures the exit code without aborting immediately
def dockerfileScan = sh(script: "trivy config --exit-code 1 ...", returnStatus: true)
def imageScan      = sh(script: "trivy image  --exit-code 1 ...", returnStatus: true)
if (dockerfileScan != 0 || imageScan != 0) {
    error "Trivy found HIGH/CRITICAL issues. Failing pipeline."
}
```

## Jenkins Shared Library — Don't Repeat Your Pipelines

If every one of dozens of components copy-pastes the same 200-line `Jenkinsfile`, one change means editing them all. A **Jenkins Shared Library** centralises the pipeline:

- **Centralised, reusable pipelines** — the logic lives in *one* place.
- **DRY (Don't Repeat Yourself)** — components carry a tiny `Jenkinsfile`, not the whole thing.
- **Change once, reflect everywhere** — fix the central pipeline and every component picks it up.

Why one library isn't enough for a big org: the "right" pipeline depends on the **combination** of language + build tool + deploy target. NodeJS+npm+EKS is a different pipeline from Java+maven+VM. You end up with a **matrix** of reusable pipelines:

| Language | Build tool | Deploy target |
|----------|-----------|---------------|
| NodeJS | npm | EKS / VM |
| Java | maven / gradle / ant | EKS / VM |
| Python | pip | EKS / VM |

The library exposes each pipeline as a function under `vars/`. A component's whole `Jenkinsfile` shrinks to: import the library, pass a config map, call the right pipeline:

```groovy
// catalogue/Jenkinsfile — the ENTIRE file
@Library('jenkins-shared-library') _

def configMap = [ project: "roboshop", component: "catalogue" ]

if (env.BRANCH_NAME.equalsIgnoreCase('main')) {
    echo "We will deal later"          // main branch handled separately
} else {
    nodejsEKSPipeline(configMap)       // feature branch → run the shared pipeline
}
```

```groovy
// jenkins-shared-library/vars/nodejsEKSPipeline.groovy — the shared pipeline
def call(Map configMap) {
    pipeline { /* agent, stages: read version → install → test → sonar
                  → dependabot → docker build → trivy → ECR push → deploy */ }
}
```

## Feature-Branch Pipeline & Image Promotion

Feature branches and `main` don't run the same pipeline. A **feature branch** runs the full CI (build, scan, test, push a candidate image); **`main`** is handled separately (the deploy/promote path) — exactly the `if (BRANCH_NAME == 'main')` split above.

Two principles tie CI back to git:

- **Commit ID = identity of the code.** If the commit ID hasn't changed, the code hasn't changed; if it has, treat the code as changed. The commit is the reference.
- **Build once, run anywhere.** You build the image **one time**, then **promote** the *same* image through environments — never rebuild per environment.

**Image promotion** is retagging, not rebuilding:

```
1. pull the image from ECR
2. retag it with the commit id:
     catalogue:1.0.0   →   catalogue:t357ad1
   (the commit id is the reference that follows it through DEV → UAT → PROD)
```

This is the CI mirror of git's "one codebase, many environments" (see [[git]]): the same artifact moves forward, only configuration changes at each stop.

---

# GitHub Token & Slack Setup for Jenkins

Covers the two credentials the RoboShop pipelines need:

- `github-token` — used by `utils.groovy` for commit statuses, PR creation, and the
  `library-scan` stage's Dependabot alerts check.
- `slack-token` — used by `nodejsEKSPipeline.groovy`'s `post { success/failure }` Slack
  notifications.

## 1. Create the GitHub fine-grained token

1. GitHub → your profile photo → **Settings** → **Developer settings** →
   **Personal access tokens** → **Fine-grained tokens** → **Generate new token**.
2. Fill in:
   - **Token name**: e.g. `jenkins-roboshop-ci`
   - **Expiration**: pick a value your org allows (90 days is a safe default — set a
     calendar reminder to rotate it).
   - **Resource owner**: the `90s-org` organization.
   - **Repository access**: **Only select repositories** → pick `catalogue` (and any
     other repos the pipeline runs against).
3. Under **Permissions → Repository permissions**, set:
   - **Dependabot alerts**: `Read-only`
   - **Commit statuses**: `Read and write`
   - **Pull requests**: `Read and write` — also required, since `utils.groovy`'s
     `createPullRequest()` uses this same token for the `raise-pr` stage.
4. Click **Generate token** and **copy it immediately** — GitHub only shows it once.
5. If the org enforces fine-grained token approval, an org owner must approve the
   request before it becomes active (**Settings → Developer settings → Personal access
   tokens** on the org side).

## 2. Add the GitHub token to Jenkins

1. Jenkins → **Manage Jenkins** → **Credentials** → **System** → **Global credentials
   (unrestricted)** → **Add Credentials**.
2. Set:
   - **Kind**: `Secret text`
   - **Secret**: paste the token from step 1
   - **ID**: `github-token` (must match exactly — this is the `credentialsId` referenced
     in `utils.groovy` and the pipeline's `library-scan` stage)
   - **Description**: `GitHub fine-grained PAT for RoboShop CI`
3. **Create**.

## 3. Create the Slack app

1. Go to `https://api.slack.com/apps` → **Create New App** → **From scratch**.
2. Name it (e.g. `Jenkins CI`) and pick your workspace.
3. In the app's left sidebar, go to **OAuth & Permissions**.
4. Under **Scopes → Bot Token Scopes**, add:
   - `chat:write` — lets the bot post messages
   - `chat:write.public` — optional, lets it post to public channels without being
     invited first
5. Scroll up and click **Install to Workspace** (or **Install App**), then **Allow**.
6. Copy the **Bot User OAuth Token** (starts with `xoxb-`) shown on that page.
7. In Slack, invite the bot to the target channel: open `#test-ci` and run
   `/invite @Jenkins CI` (skip this if you added `chat:write.public`).

## 4. Add the Slack token to Jenkins

1. Jenkins → **Manage Jenkins** → **Plugins** → confirm **Slack Notification** plugin
   is installed.
2. **Manage Jenkins** → **Credentials** → **System** → **Global credentials** →
   **Add Credentials**:
   - **Kind**: `Secret text`
   - **Secret**: paste the `xoxb-...` bot token
   - **ID**: `slack-token` (must match the `tokenCredentialId` used in
     `nodejsEKSPipeline.groovy`'s `slackSend` calls)
3. **Manage Jenkins** → **System** → scroll to the **Slack** section:
   - **Workspace**: your workspace subdomain (e.g. `90s-org` for
     `https://90s-org.slack.com`)
   - **Credential**: select the `slack-token` credential
   - **Default channel**: `#test-ci` (optional — the pipeline already passes `channel:`
     explicitly)
4. Click **Test Connection**, then **Save**.

## 5. Verify

- Trigger a build and confirm:
  - `library-scan` stage successfully fetches Dependabot alerts (no 403/404).
  - Commit status checks appear on the GitHub commit/PR.
  - `raise-pr` stage opens a PR (on non-`main` branches).
  - A Slack message lands in `#test-ci` on success/failure.

---

# The RoboShop Release Pipeline — DEV → SIT → UAT → PROD

Everything above is the **CI pipeline** — one component, one branch, build/scan/test/push. This section covers the second, bigger pipeline (`nodejsEKSMain.groovy`) that takes an image that already passed CI and **promotes** it through four real environments, gated by GitHub and tracked by one Jira ticket per release.

## Generic Webhook Trigger — Letting Jira Start a Build

Normally something pushes to GitHub and Jenkins reacts. Here, Jira *also* needs to start builds (SIT/UAT/PROD) with no code push and no GitHub event involved. The **Generic Webhook Trigger** plugin turns any authenticated HTTP POST into a build trigger, and can lift fields straight out of the JSON body into `env.*`:

```groovy
// jenkins-shared-library/vars/nodejsEKSMain.groovy
triggers {
    GenericTrigger(
        genericVariables: [
            [key: 'ENVIRONMENT', value: '$.ENVIRONMENT'],   // JSONPath into the POST body
            [key: 'COMMIT_ID',   value: '$.COMMIT_ID'],
            // ...
        ],
        tokenCredentialId: 'jira-secret',                   // the whole authentication story
        causeString: 'Triggered by Jira Automation'
    )
}
```

The URL Jira's Automation "Send web request" action calls:

```
http://jenkins.daws90s.shop:8080/generic-webhook-trigger/invoke?token=<jira-secret value>
```

Because this path populates `env.*` and a manual "Build with Parameters" run only ever populates `params.*`, the pipeline can't just read one or the other — `resolve-inputs` (env wins when present, falls back to params) is what lets one build handle both trigger paths.

## `utils.groovy` — What Each Helper Actually Does

| Function | Does | Notable design choice |
|----------|------|------------------------|
| `updateCommitStatus(state, description, context)` | POSTs a GitHub commit status | Lowercases `state` — GitHub's API returns a 422 on anything but `error/failure/pending/success` |
| `validateCommitStatus(commitSha, requiredContexts)` | Fails the build unless every named context is `success` on that commit | The real promotion gate — see below |
| `getPodIP(namespace, component)` | `kubectl get pod ... -o jsonpath='{.items[0].status.podIP}'` | The Jenkins agent can't resolve `*.svc.cluster.local`, but shares a VPC with EKS — routes to pods by IP instead of cluster DNS |
| `tagCommit(commitSha, tag)` | Creates a `refs/tags/<tag>` GitHub ref on the commit | A real, permanent GitHub release marker — not a Jenkins-only concept |
| `createJiraTicket(projectKey, commitId, version)` | Looks up the `Commit ID`/`Version` custom field **IDs by name**, then `jiraNewIssue` | Looked up at runtime instead of hardcoding `customfield_XXXXX` — survives the Jira site being rebuilt |
| `transitionJiraIssue(issueKey, targetStatus)` | Finds a transition whose **destination status name** matches, fires it | Matches by where the transition *goes*, not what it's *labelled* — labels on the workflow diagram aren't guaranteed to mean anything |
| `safeTransitionJiraIssue(issueKey, targetStatus)` | Same, wrapped so a failure only logs a warning | Jira being down must never fail an otherwise-good deploy, overwrite a correct GitHub status, or hide the real exception |
| `createPullRequest(base, title, body)` | Opens `branch → base` if one isn't already open | Used by the CI pipeline's `raise-pr` stage, not the release pipeline |

## Validating at Every Gate — Why Nothing Can Skip a Step

In a naive setup, anyone with Jenkins access could run "Build with Parameters", pick `ENVIRONMENT=prod`, and skip SIT/UAT entirely. The Jira ticket *saying* "UAT Done" proves nothing if nobody checks it. So the pipeline doesn't trust the ticket — it trusts GitHub.

Each stage posts its own named commit status. Promoting to the next environment requires every status the earlier environments would have posted, re-checked fresh against the real commit:

| Target env | Required contexts before it'll deploy |
|------------|----------------------------------------|
| sit | `dev-deploy`, `api-tests` |
| uat | + `sit-deploy`, `sit-integration-tests` |
| prod | + `uat-deploy`, `uat-regression-tests` |

A commit genuinely cannot reach PROD without having passed real SIT and UAT runs — even a manually-triggered build with the "right" parameters fails at `validate-commit-status` if those contexts aren't present. GitHub commit statuses are the actual gate; the Jira ticket is just the human-facing tracker riding on top.

## The CR (Change Request) Gate

In real companies, a **Change Request (CR)** is the paperwork trail for anything touching production: what's changing, why, when, who approved it, how to undo it. Larger orgs route changes through a **CAB (Change Advisory Board)** — a review before a deploy is allowed into a defined change window.

- **Standard** change — pre-approved, low-risk, repeatable (this kind of routine app deploy would normally be one).
- **Normal** change — needs case-by-case approval.
- **Emergency** change — skips the normal cycle for an active incident, approved after the fact.

The pipeline's `change-request-check` models the *shape* of this, without a real CR/ITSM system behind it yet:

```groovy
// nodejsEKSMain.groovy — change-request-check (condensed)
if (!env_CR_NUMBER?.trim()) { error("CR_NUMBER is required for a prod deploy") }
if (!env_VERSION?.trim())   { error("VERSION is required for a prod deploy") }

/* Dummy deployment-window check — blocks weekends. Placeholder for a real CR lookup. */
def dayOfWeek = sh(script: 'date +%u', returnStdout: true).trim().toInteger()
if (dayOfWeek >= 6) { error("outside the approved deployment window") }

input message: "Approve prod deploy ...?", ok: 'Approve'   // the CAB stand-in
```

**CR Number** is a required field, filled in on the Jira ticket via a transition screen when moving to `Trigger PROD` — the same idea as a real CR ticket number being attached before a change proceeds. The `input` step is the human approval; it runs with its **own 4-hour timeout**, separate from the pipeline's overall 15-minute budget, so waiting on a person doesn't get killed by an unrelated timeout.

## Testing Team Roles & Responsibilities

Who owns which layer of testing, grounded in the actual file/job split:

| Test layer | Owned by | Where it lives | Mocked? |
|------------|----------|-----------------|---------|
| Unit tests | Developers | `catalogue/test/` | Yes — DB and other services mocked |
| API tests (dev gate) | Dev/SDET | Jenkins job `catalogue-api-tests` | No — real component, real dev namespace |
| Integration tests (SIT gate) | SDET | `roboshop-integration-tests/` | No — live HTTP, cross-service |
| Regression tests (UAT gate) | SDET | `roboshop-regression-tests/` | No — live HTTP, cross-service |
| Smoke tests (prod) | SDET | `catalogue/smoke/` | No — live HTTP against the just-deployed prod release |

The pattern: tests get **less isolated and more end-to-end** the closer you get to production. Unit tests trust nothing is real; smoke tests trust *everything* is real — they're the last check that the actual prod deployment (not just the image) genuinely works, run in-process in the same pipeline build rather than as a separate Jenkins job.

## Rollback on a Bad Deploy

```groovy
// nodejsEKSMain.groovy — prod-deploy (condensed)
def releaseExists = sh(script: "helm status ${component} -n roboshop-prod > /dev/null 2>&1", returnStatus: true) == 0
try {
    sh "helm upgrade --install ... --wait --timeout 5m"
}
catch (Exception e) {
    if (releaseExists) {
        sh "helm rollback ${component} 0 -n roboshop-prod --wait --timeout 5m"   // back to the previous revision
    }
    // else: first-ever deploy — nothing to roll back to, just fails
    throw e
}
```

`helm rollback <release> 0` means "the immediately preceding revision" — Helm keeps its own version history per release. Rollback only makes sense if a previous good release exists, which is why `helm status` is checked *before* even attempting the deploy.

**Known gap worth flagging to students:** `smoke-tests` (which runs *after* `prod-deploy` succeeds) does **not** trigger a rollback if it fails — the bad release stays live until a human acts. A deploy-time failure rolls back automatically; a failure only detected *after* deploy currently doesn't.

## Build Once, Run Anywhere — In Full

Earlier in this doc, image promotion was shown retagging with a short id. The release pipeline goes further: it never truncates the SHA at all, anywhere:

```groovy
// nodejsEKSMain.groovy — promote-image
docker pull    .../catalogue:${appVersion}
docker tag     .../catalogue:${appVersion} .../catalogue:${env.GIT_COMMIT}   // full 40-char SHA
docker push    .../catalogue:${env.GIT_COMMIT}
```

That full SHA is then the *only* identity the image carries through SIT/UAT/PROD — it's what's stored as the Jira ticket's `Commit ID` field, what `validate-commit-status` checks GitHub statuses against, and what every `helm upgrade --set deployment.imageVersion=...` deploys:

```
DEV:  build catalogue:1.0.0 (app version) → promote → catalogue:e551cedf...53fe (full commit SHA)
SIT:  helm upgrade --set deployment.imageVersion=e551cedf...53fe
UAT:  helm upgrade --set deployment.imageVersion=e551cedf...53fe
PROD: helm upgrade --set deployment.imageVersion=e551cedf...53fe
```

One image, one identity, deployed unchanged four times — the whole point of "build once, run anywhere."

## Quick Reference

| Concept | One-liner |
|---------|-----------|
| **CI** | Continuously integrate every commit into a tested artifact: clone → build → scan → test |
| CI stages | `clone → install deps → authenticate → build → scan → unit test → artifact → deploy` |
| **Shift-left** | Run scans/tests **before** deploy — fail fast, cheap to fix |
| 3 error types | Functionality (code defect) · Build (config) · Deployment (multi-env) |
| **Jenkins** | A plain web server; **plugins add all capability** (SCM, build, cloud, notify…) |
| Job phases | Pre-build (env/options) → Build → Post-build (notify) |
| **Master node** | Never builds; triggers, assigns agents, monitors, collects logs |
| **Agent node** | Does the build; separate labelled nodes per stack (Java/NodeJS/…) |
| `agent { label }` | Pin a pipeline to nodes with a given label (e.g. `ROBOSHOP`) |
| **Freestyle job** | UI-defined; no version control, no reuse, no audit — avoid |
| **Pipeline job** | `Jenkinsfile` in git — versioned, reviewable, reusable |
| **Declarative** | New syntax; validated **before** run — the default |
| **Scripted** | Old Groovy syntax; compiled at run time; max control |
| Jenkinsfile sections | `agent · environment · options · parameters · stages/steps · when · post` |
| `post` | `always` / `success` / `failure` — usually notifications |
| `parameters` | Runtime inputs: `string`, `booleanParam`, `choice`, `password`, `text` |
| **Credentials** | Secrets stored centrally, referenced by ID (`aws-creds`, `github-token`) |
| `withAWS` / `withCredentials` | Inject secrets into a stage without hard-coding them |
| **ECR push** | `ecr get-login-password | docker login` → `docker build` → `docker push` |
| Image tag = version | Read from `package.json` so build & artifact agree |
| **SonarQube** | Static analysis (SAST + coding standards); web UI on port 9000 |
| Bug / Vulnerability / Code smell | Breaks / exploitable / hard-to-maintain |
| Ratings | Security · Reliability · Maintainability, graded A–E |
| New vs overall code | New = `C4−C3` (this change); Overall = whole codebase — gate new first |
| **Quality gate** | Pass/fail threshold; `waitForQualityGate()` fails the pipeline |
| `sonar-project.properties` | Configures the scan — sources, exclusions, coverage report path |
| Sonar rollout | Phase 1 onboard → 2 gate new code → 3 gate overall, ratchet coverage to 80% |
| **Unit test** | Dev-written; one function, dependencies mocked; no deployment |
| **Code coverage** | % of code lines exercised by unit tests — feeds the Sonar gate |
| Testing levels | Unit → API/component → integration |
| **Dependency scan** | Check libraries vs NVD; Dependabot / Nexus IQ / Blackduck |
| **CVE / CVSS** | The vulnerability's ID / its severity score |
| Severities | Low · Medium · High · Critical — fail on High/Critical |
| **Trivy** | One CLI scanning Dockerfile, image OS packages, and deps; `--exit-code 1` to fail |
| **Shared library** | Centralised, DRY pipelines; change once, reflect everywhere (`@Library`) |
| Pipeline matrix | Reusable pipeline per language × build tool × deploy target |
| **Commit ID** | Identity of the code — unchanged id = unchanged code |
| **Build once, run anywhere** | Build the image once; **promote** (retag with commit id) across envs |
| Image promotion | Pull from ECR → retag `1.0.0 → t357ad1`; don't rebuild per env |
| **Multibranch Pipeline** | Scans a whole repo, auto-creates a sub-job per branch/PR with a Jenkinsfile |
| Branch job Configure page | Locked/read-only — computed from the scan; triggers must live in the Jenkinsfile |
| **Multibranch Scan Webhook Trigger** | Gives a Multibranch job its own webhook to re-trigger a rescan, SCM-agnostic |
| `ssh-auth` credential | SSH Username with private key, `ec2-user` — controller → agent connection |
| `sonar-creds` credential | Secret text holding the Sonar token — selected in SonarQube server config, never pasted raw |
| **Generic Webhook Trigger** | Any authenticated POST can start a build; JSONPath lifts fields into `env.*` |
| `resolve-inputs` pattern | `env.*` (webhook) wins, falls back to `params.*` (manual) — one build, two trigger paths |
| **`validate-commit-status`** | Re-checks required GitHub commit-status contexts before promoting — the real gate, not the Jira ticket |
| Accumulating contexts | sit needs dev's; uat needs sit's too; prod needs uat's too — can't skip a step |
| **CR (Change Request)** | Paperwork trail for a prod change: what/why/when/who/rollback; reviewed by a CAB |
| CR types | Standard (pre-approved) · Normal (case-by-case) · Emergency (after-the-fact) |
| `change-request-check` | CR_NUMBER + VERSION required, dummy window check, manual `input` = the CAB stand-in |
| Testing ownership | Unit (dev, mocked) → API/Integration/Regression (SDET, live) → Smoke (SDET, live prod) |
| **Rollback** | `helm rollback <release> 0` on a failed deploy over an *existing* release only |
| Rollback gap | Deploy-time failure auto-rolls-back; smoke-test failure (post-deploy) currently doesn't |
| **JIRA Steps plugin** | `jira*` pipeline steps; site auth configured once under Manage Jenkins → System |
| `utils.groovy` Jira helpers | `createJiraTicket` (fields by name) · `transitionJiraIssue` (by destination status) · `safeTransitionJiraIssue` (best-effort) |

---
