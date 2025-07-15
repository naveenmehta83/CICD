As an expert in DevOps and CI/CD, I can provide you with detailed, production-level runbooks for a Continuous Integration (CI) pipeline for a microservices application. These examples reflect the industry standards widely adopted by leading companies.

The core principle of CI is to integrate code changes frequently into a shared repository, running automated builds and tests to detect integration errors early.

---

### **Industry Standard CI Pipeline Principles:**

Before diving into the examples, let's outline the universally accepted principles for a robust CI pipeline:

1.  **Automated Triggering:** Every code commit to a specific branch (e.g., `main`, `develop`, feature branch for PRs) automatically triggers the pipeline.
2.  **Fast Feedback Loop:** The pipeline should complete quickly to give developers rapid feedback on their changes.
3.  **Comprehensive Automated Testing:**
    *   **Unit Tests:** Verify individual components in isolation.
    *   **Integration Tests:** Verify interactions between components/services.
    *   **Code Quality Checks:** Linting, static analysis (SAST), code complexity.
4.  **Security Integration:**
    *   **Dependency Scanning (SCA):** Identify known vulnerabilities in third-party libraries.
    *   **Static Application Security Testing (SAST):** Analyze source code for security flaws.
5.  **Immutable Artifacts:** Produce self-contained, versioned artifacts (e.g., Docker images, JARs, binaries) that are identical across all environments.
6.  **Reproducibility:** Any build from a specific commit should yield the same artifact.
7.  **Artifact Storage:** Store artifacts in a secure, highly available, and versioned artifact repository.
8.  **Notifications & Visibility:** Provide clear status updates and notifications (e.g., Slack, email) on pipeline success or failure.
9.  **Scalability:** The CI system should be able to handle a high volume of builds concurrently.

---

### **Example 1: CI Pipeline with GitHub Actions (for a Java Spring Boot Microservice)**

This runbook details a CI workflow for a Java Spring Boot microservice, building a Docker image and pushing it to a container registry.

**Application:** `payment-service` (Java 17, Spring Boot, Maven)
**Repository:** `github.com/your-org/payment-service`
**Artifact:** Docker Image
**Container Registry:** GitHub Container Registry (GHCR)

---

#### **Runbook: CI Pipeline for `payment-service` (GitHub Actions)**

**1. Purpose:**
To automate the build, test, and containerization of the `payment-service` microservice upon every code push to ensure code quality, functional correctness, and security, producing a deployable Docker image.

**2. Pre-requisites:**
*   GitHub Repository: `your-org/payment-service`
*   GitHub Actions enabled for the repository.
*   Java 17 SDK (`.tool-versions` or `pom.xml` config).
*   Maven Wrapper (`mvnw`) included in the repository.
*   Docker build context and `Dockerfile` in the repository root.
*   **Secrets:**
    *   `GH_TOKEN` (automatically provided by GitHub Actions for `github.actor` or `GITHUB_TOKEN` for pushing images to GHCR)
    *   `DOCKER_USERNAME`: Username for Docker Hub (if using)
    *   `DOCKER_PASSWORD`: Password/PAT for Docker Hub (if using)
    *   `SONAR_TOKEN`: SonarCloud token (if using)
    *   `SNYK_TOKEN`: Snyk API token (if using)

**3. Trigger:**
This workflow is triggered automatically on:
*   Pushes to the `main` branch.
*   Pull requests targeting the `main` branch.
*   Manually via `workflow_dispatch`.

**4. Workflow Steps (Detailed `payment-service/.github/workflows/ci.yml`):**

```yaml
name: CI - Payment Service

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch: # Allows manual trigger

env:
  JAVA_VERSION: '17'
  DISTRIBUTION: 'temurin'
  DOCKER_IMAGE_NAME: ghcr.io/${{ github.repository }} # e.g., ghcr.io/your-org/payment-service

jobs:
  build_and_test:
    name: Build & Test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write # Required to push to GHCR
      security-events: write # Required for SAST/SCA results

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Required for SonarCloud

      - name: Set up JDK ${{ env.JAVA_VERSION }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: ${{ env.DISTRIBUTION }}
          cache: maven # Cache Maven dependencies

      - name: Cache Maven dependencies
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      - name: Run Maven Build, Unit Tests & Static Analysis
        run: ./mvnw clean verify
        # The 'verify' phase usually includes unit tests and potentially integration tests (if configured)
        # and static analysis plugins (e.g., Maven Enforcer, SpotBugs)

      - name: Run SonarCloud Analysis (Optional but Recommended for Quality Gate)
        # This step sends coverage and analysis results to SonarCloud
        # Requires 'SONAR_TOKEN' secret and SonarCloud configuration in pom.xml
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} # Required for SonarCloud integration with GH
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: ./mvnw org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=your-org_payment-service
        if: success() && github.event_name == 'push' # Only run on pushes to main for definitive analysis

      - name: Run Dependency Security Scan (e.g., Snyk)
        # Scans for known vulnerabilities in third-party dependencies
        uses: snyk/actions/maven@master # Or snyk/actions/docker@master for image scan later
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --all-projects --fail-on=all
        continue-on-error: true # Allow build to continue even if vulnerabilities found, but log them
        # Alternatively, use Trivy for open-source scanning:
        # - name: Run Trivy Dependency Scan
        #   uses: aquasecurity/trivy-action@master
        #   with:
        #     scan-type: 'fs'
        #     ignore-unfixed: true
        #     format: 'sarif'
        #     output: 'trivy-results.sarif'
        #     severity: 'CRITICAL,HIGH'
        # - name: Upload Trivy Scan Results to GitHub Security Tab
        #   uses: github/codeql-action/upload-sarif@v3
        #   with:
        #     sarif_file: 'trivy-results.sarif'
        #   if: always()

      - name: Build Docker Image
        run: docker build -t ${{ env.DOCKER_IMAGE_NAME }}:latest -t ${{ env.DOCKER_IMAGE_NAME }}:${{ github.sha }} .
        # Builds two tags: 'latest' (mutable, for dev/staging) and 'commit_sha' (immutable, for production traceability)

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }} # Use GITHUB_TOKEN for GHCR

      - name: Push Docker Image to GHCR
        run: |
          docker push ${{ env.DOCKER_IMAGE_NAME }}:latest
          docker push ${{ env.DOCKER_IMAGE_NAME }}:${{ github.sha }}

      - name: Update GitHub Commit Status
        # This is implicitly handled by GitHub Actions checks, but can be explicit for external tools.
        # Spinnaker or other CD tools can then watch for this commit status or the artifact directly.
        run: echo "Build successful for commit ${{ github.sha }}"

      - name: Notify on Slack (Optional)
        uses: rtCamp/action-slack-notify@v2
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
          SLACK_CHANNEL: '#devops-alerts'
          SLACK_MESSAGE: "CI Pipeline for `payment-service` **${{ job.status }}**! \nBranch: `${{ github.ref_name }}`\nCommit: `${{ github.sha }}`\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Pipeline>"
        if: always() # Run even if previous steps fail
```

**5. Error Handling & Rollback (CI specific):**
*   **Fail Fast:** If any step (build, test, security scan with `fail-on` flag) fails, the pipeline immediately stops.
*   **Notifications:** Automatic notifications (e.g., Slack) are sent on failure, directing developers to the failed run.
*   **No Rollback (CI):** The CI pipeline itself does not perform rollbacks. Its purpose is to validate the change. Rollbacks are handled by the CD pipeline or manual intervention post-deployment if issues arise in later environments.

**6. Monitoring:**
*   **GitHub Actions UI:** Provides detailed logs, step-by-step execution status, and historical runs.
*   **SonarCloud/SonarQube Dashboard:** For code quality metrics and security hotspots.
*   **Snyk/Trivy Reports:** For dependency vulnerabilities.
*   **Custom Notifications:** Integration with Slack, Teams, email for immediate alerts.

**7. Key Considerations:**
*   **Parallelization:** For larger microservice portfolios, consider monorepo setups with path-based triggers or separate workflows for each service to run in parallel.
*   **Test Data:** Ensure tests are isolated and don't rely on external services unless mocked or integrated as part of a dedicated integration test stage.
*   **Environment Variables & Secrets:** Use GitHub Secrets for sensitive information.
*   **Caching:** Leverage GitHub Actions caching to speed up dependency resolution.
*   **Docker Layer Caching:** Optimize your `Dockerfile` and build process to maximize Docker layer caching, significantly reducing build times.
*   **Artifact Naming Convention:** Consistent, traceable tagging (e.g., `image_name:git_sha`, `image_name:semantic_version`).

---

### **Example 2: CI Pipeline with Jenkins (for a Node.js Microservice)**

This runbook describes a Jenkins Pipeline for a Node.js microservice, including dependency management, testing, security scanning, and Docker image creation.

**Application:** `user-profile-service` (Node.js, Express, npm)
**Repository:** `your-gitlab-instance.com/devops-team/user-profile-service`
**Artifact:** Docker Image
**Container Registry:** JFrog Artifactory (or Docker Hub)

---

#### **Runbook: CI Pipeline for `user-profile-service` (Jenkins)**

**1. Purpose:**
To automate the build, test, and containerization of the `user-profile-service` microservice upon every code push, ensuring code quality, functional correctness, and security, producing a deployable Docker image.

**2. Pre-requisites:**
*   Jenkins server installed with necessary plugins (e.g., Git, Docker, Node.js, Pipeline Utility Steps, SonarQube Scanner, Artifactory/Docker Pipeline).
*   Node.js configured in Jenkins global tools or via `nvm`.
*   Docker daemon accessible from Jenkins agents.
*   GitLab/GitHub webhook configured to trigger Jenkins job.
*   `Dockerfile` in the repository root.
*   **Jenkins Credentials:**
    *   `gitlab-ssh-key`: SSH key to clone repository (if not public).
    *   `docker-registry-creds`: Username/Password for Artifactory Docker registry or Docker Hub.
    *   `sonar-token`: SonarQube/SonarCloud token.
    *   `owasp-dep-check-cli`: Path to OWASP Dependency-Check CLI (if using).

**3. Trigger:**
*   **SCM Poll:** (Less preferred) Periodically poll the SCM for changes.
*   **Webhook:** (Preferred) A webhook from GitLab/GitHub configured to trigger the Jenkins pipeline on `push` events to the `main` branch or on Merge Request/Pull Request creation/update.

**4. Workflow Steps (Detailed `Jenkinsfile`):**

```groovy
// Jenkinsfile for user-profile-service CI Pipeline

pipeline {
    agent {
        # Use a Docker agent with Node.js and Docker CLI pre-installed
        docker {
            image 'node:lts-slim-bullseye' # Base image with Node.js
            args '-v /var/run/docker.sock:/var/run/docker.sock' # Mount Docker socket for building images
        }
    }

    environment {
        SERVICE_NAME = 'user-profile-service'
        DOCKER_REGISTRY = 'your-artifactory-domain.com/docker-repo' // e.g., myregistry.jfrog.io/my-docker-local
        DOCKER_IMAGE_TAG = "${env.GIT_COMMIT}" // Immutable tag based on Git commit SHA
        # Or, if you prefer semantic versioning derived from package.json:
        # DOCKER_IMAGE_TAG = sh(returnStdout: true, script: "node -p \"require('./package.json').version\"").trim() + "-${env.GIT_COMMIT, 0, 7}"
    }

    options {
        # Set a timeout for the entire pipeline
        timeout(time: 30, unit: 'minutes')
        # Discard old builds to save space
        buildDiscarder(logRotator(numToKeepStr: '10'))
        # Enable timestamps in console output
        timestamps()
        # Allows for clean checkout and workspace each time
        skipDefaultCheckout()
    }

    stages {
        stage('Checkout Code') {
            steps {
                script {
                    # Clean workspace before checkout
                    deleteDir()
                    # Checkout the repository
                    git url: 'ssh://git@your-gitlab-instance.com:22/devops-team/user-profile-service.git',
                        credentialsId: 'gitlab-ssh-key',
                        branch: 'main'
                }
            }
        }

        stage('Install Dependencies & Cache') {
            steps {
                script {
                    # Check if node_modules exists and is valid from previous builds
                    # using checksums or smarter caching logic
                    if (fileExists('package-lock.json')) {
                        sh 'npm ci --prefer-offline --no-audit' // 'npm ci' for clean install based on lock file
                    } else {
                        sh 'npm install --no-audit' // If no lock file or first run
                    }
                    # Optional: Cache npm modules using Jenkins' built-in cache or persistent volumes
                    # cache(path: 'node_modules', key: "${env.JOB_NAME}-${env.BUILD_NUMBER}-${sh(script: 'npm config get cache', returnStdout: true).trim()}")
                }
            }
        }

        stage('Run Unit & Integration Tests') {
            steps {
                script {
                    sh 'npm test' // Assumes npm test runs unit and integration tests
                    # Optional: Generate test reports in JUnit XML format
                    # sh 'npm test -- --reporter mocha-junit-reporter --reporter-options mochaFile=./test-results/unit-tests.xml'
                    # junit 'test-results/*.xml' // Publish JUnit reports in Jenkins
                }
            }
        }

        stage('Code Quality & Linting') {
            steps {
                script {
                    sh 'npm run lint' // Assumes a 'lint' script in package.json
                }
            }
        }

        stage('Security Scans') {
            steps {
                script {
                    # OWASP Dependency-Check (SCA)
                    # Requires 'owasp-dep-check-cli' tool configured in Jenkins
                    tool name: 'owasp-dep-check-cli', type: 'owasp-dependency-check'
                    sh "dependency-check --scan . --format HTML --project \"${env.SERVICE_NAME}\" --out build/reports/dependency-check-report --failOnCVSS 7.0"
                    archiveArtifacts artifacts: 'build/reports/dependency-check-report.html', fingerprint: true
                    # SonarQube/SonarCloud (SAST & SCA)
                    withSonarQubeEnv(credentialsId: 'sonar-token') {
                        sh "npm install -g sonarqube-scanner" # Install scanner if not in agent
                        sh "sonar-scanner -Dsonar.projectKey=${env.SERVICE_NAME} -Dsonar.sources=."
                    }
                    # Optional: Add a quality gate check for SonarQube
                    timeout(time: 5, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    # Build the Docker image
                    sh "docker build -t ${env.DOCKER_REGISTRY}/${env.SERVICE_NAME}:${env.DOCKER_IMAGE_TAG} ."
                    sh "docker build -t ${env.DOCKER_REGISTRY}/${env.SERVICE_NAME}:latest ." # Optional: latest tag for dev/staging
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    # Login to Docker Registry (e.g., Artifactory)
                    withCredentials([usernamePassword(credentialsId: 'docker-registry-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo ${DOCKER_PASS} | docker login ${env.DOCKER_REGISTRY} --username ${DOCKER_USER} --password-stdin"
                    }
                    # Push the Docker image(s)
                    sh "docker push ${env.DOCKER_REGISTRY}/${env.SERVICE_NAME}:${env.DOCKER_IMAGE_TAG}"
                    sh "docker push ${env.DOCKER_REGISTRY}/${env.SERVICE_NAME}:latest"
                }
            }
        }
    }

    post {
        always {
            # Clean up workspace regardless of build success/failure
            cleanWs()
        }
        success {
            # Notify on success
            slackSend channel: '#devops-success', message: "CI Pipeline for `${SERVICE_NAME}` **SUCCESS**! Build: <${env.BUILD_URL}|${env.BUILD_NUMBER}>", color: '#00FF00'
        }
        failure {
            # Notify on failure
            slackSend channel: '#devops-failures', message: "CI Pipeline for `${SERVICE_NAME}` **FAILED**! Build: <${env.BUILD_URL}|${env.BUILD_NUMBER}>", color: '#FF0000'
        }
        # Add email notifications, JIRA updates, etc.
    }
}
```

**5. Error Handling & Rollback (CI specific):**
*   **Fail Fast:** Any non-zero exit code from a `sh` step or a failed test stage will mark the build as failed and stop execution.
*   **Notifications:** `post` section handles automatic notifications based on build status.
*   **No Rollback (CI):** Similar to GitHub Actions, the CI pipeline's role is to validate the change. Rollbacks are managed by the CD system (e.g., Spinnaker) after deployment.

**6. Monitoring:**
*   **Jenkins Build Console Output:** Provides real-time logs and detailed step execution.
*   **Jenkins Blue Ocean UI:** Offers a modern, visual representation of pipeline execution.
*   **Test Result Trends:** Jenkins can display trends for unit and integration tests.
*   **SonarQube Dashboard:** Comprehensive code quality and security metrics.
*   **OWASP Dependency-Check Reports:** HTML reports archived directly in Jenkins.
*   **Notifications:** Slack, email, JIRA integration for immediate alerts.

**7. Key Considerations:**
*   **Jenkins Agents:** Use ephemeral Docker agents or Kubernetes pods for isolated, reproducible, and scalable build environments.
*   **Plugin Management:** Keep Jenkins plugins updated for security and features.
*   **Credentials Management:** Use Jenkins Credentials Provider for secure storage of sensitive data.
*   **Artifact Retention:** Configure build discarders and artifact retention policies to manage storage.
*   **Security Scans:** Regularly update security scanner databases (e.g., OWASP Dependency-Check, Trivy).
*   **Pipeline as Code:** Always define pipelines using `Jenkinsfile` stored in SCM for version control and auditability.
*   **Parallelism:** For complex applications, Jenkins can run stages or steps in parallel to speed up the overall build time.

---

Both examples demonstrate a comprehensive CI process that goes beyond simple build-and-test, integrating essential steps like code quality analysis, security scanning, and robust artifact management, which are crucial for maintaining high standards in a production DevOps environment.