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

---
