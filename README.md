# SonarQube CI/CD Integration with GitHub Actions

This project implements a Static Application Security Testing (SAST) pipeline using a self-hosted GitHub runner to analyze local code quality and security.

## 1. Fundamental Concepts

### What is SonarQube?

**SonarQube** acts as an automated code reviewer that inspects every line of code for quality and security.

* **Early Detection**: Catches bugs before compilation or deployment.
* **Security Guard**: Identifies vulnerabilities like SQL injection and XSS.
* **Maintainability**: Calculates "Technical Debt" to estimate cleanup time.

### Implementation Workflow (Local Loop)

1. **Push**: Developer pushes code to GitHub.
2. **Trigger**: GitHub Actions initiates the workflow defined in `.yml`.
3. **Execution**: A **Self-Hosted Runner** on your local machine picks up the job.
4. **Analysis**: The **Sonar Scanner** analyzes code and sends results to the local SonarQube server.
5. **Evaluation**: SonarQube checks results against a **Quality Gate**.

### Why Self-Hosted?

Since SonarQube is running on `localhost:10000`, GitHub’s cloud runners cannot reach it.

* **Connectivity**: The runner lives on your machine, allowing it to communicate with local Docker containers.
* **Cost Efficiency**: Uses your own hardware resources, saving GitHub Action minutes.

---

## 2. Step-by-Step Implementation

### Step 1: Prepare SonarQube Environment

Ensure your Docker container is active:

* **Command**: `docker run -d --name sonarqube -p 10000:9000 sonarqube:latest`
* **Access**: `http://localhost:10000` (Default: `admin/admin`)
* **Setup**: Create a project manually and set the Project Key to `main-project`.

### Step 2: Set up the GitHub Runner

1. Navigate to **Settings > Actions > Runners** in your GitHub repo.
2. Follow the OS-specific instructions to download and configure the runner.
3. Ensure the terminal shows `Listening for Jobs` before pushing code.

### Step 3: Configure Pipeline Secrets

Add the following under **Settings > Secrets and variables > Actions**:

* `SONAR_TOKEN`: Generated in SonarQube under **Account > Security**.
* `SONAR_HOST_URL`: Set to `http://localhost:9000` (internal container port).

### Step 4: Define the Workflow

Create `.github/workflows/sonar.yml`:

```yaml
name: SonarQube Analysis
on: [push]
jobs:
  sonar_scan:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Analyze Code
        uses: sonarsource/sonarqube-scan-action@master
        with:
          args: >
            -Dsonar.projectKey=main-project
            -Dsonar.projectName="Main Project"
            -Dsonar.sources=src
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

      - name: Quality Gate Check
        uses: sonarsource/sonarqube-quality-gate-action@master
        timeout-minutes: 5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

```

---

## 3. Quality Gates (Success & Failure)

### Automated Failure

The pipeline fails only if the `sonarqube-quality-gate-action` is included. This step forces GitHub to wait for the analysis result. If the result is "Failed," the GitHub check status turns red.

### Configuration

In the SonarQube UI, go to **Quality Gates** to set your "Fail" criteria:

* **Default ("Sonar way")**: Fails on new bugs or security risks.
* **Custom**: Set specific thresholds (e.g., Fail if `Vulnerabilities > 0` or `Coverage < 80)

