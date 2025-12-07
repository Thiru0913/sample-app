# Jenkins CI/CD Pipeline for Spring Boot Microservices

Complete automated CI/CD pipeline using Jenkins Shared Libraries for building, testing, scanning, and deploying Java Spring Boot applications to Kubernetes with Helm.

## Overview

This pipeline provides:
- **Git Clone** - Source code checkout
- **Build** - Maven compilation and packaging
- **Unit Test** - Automated testing with coverage reports
- **Code Quality** - SonarQube analysis with quality gates
- **Security Scan** - Nexus IQ vulnerability scanning
- **Artifact Upload** - JFrog Artifactory integration
- **Docker Build** - Container image creation
- **Container Security** - Trivy vulnerability scanning
- **Registry Push** - Push to JFrog container registry
- **Secret Management** - HashiCorp Vault integration
- **Kubernetes Deploy** - Helm-based deployment with environment-specific configurations

## Architecture

```
Source Repository (GitHub)
    ↓
Jenkins Shared Library (vars/, src/)
    ↓
Pipeline Stages (11 stages)
    ├─ Checkout → Build → Test → SonarQube → Nexus IQ
    ├─ Artifact Upload → Docker Build → Trivy Scan
    ├─ Registry Push → Secret Management → Helm Deploy
    ↓
Kubernetes Cluster (dev/uat/prod)
    ├─ Deployment
    ├─ Service
    ├─ ConfigMap / Secrets
    ├─ HPA (autoscaling)
    └─ Ingress
```

## Repository Structure

```
jenkins-shared-library/
├── vars/
│   └── buildPipeline.groovy          # Main pipeline entry point
├── src/com/company/
│   ├── Build.groovy                  # Build and test steps
│   ├── CodeQuality.groovy            # SonarQube and Nexus IQ
│   ├── ArtifactManager.groovy        # Artifactory uploads
│   ├── Docker.groovy                 # Docker and Trivy
│   ├── SecretManager.groovy          # Vault + K8s secrets
│   └── HelmDeployer.groovy           # Helm deployment

Jenkinsfile                            # Template Jenkinsfile for repos
config.yaml.example                    # Configuration template

deployment/
├── Chart.yaml                         # Helm chart metadata
├── values.yaml                        # Default values
├── values-dev.yaml                    # Dev environment
├── values-uat.yaml                    # UAT environment
├── values-prod.yaml                   # Production environment
├── templates/
│   ├── deployment.yaml                # Kubernetes Deployment
│   ├── service.yaml                   # Kubernetes Service
│   ├── configmap.yaml                 # ConfigMap
│   ├── hpa.yaml                       # HorizontalPodAutoscaler
│   ├── pvc.yaml                       # PersistentVolumeClaim
│   ├── ingress.yaml                   # Ingress
│   ├── serviceaccount.yaml            # ServiceAccount
│   └── _helpers.tpl                   # Helm helpers
```

## Setup Instructions

### 1. Configure Jenkins

#### Install Required Plugins
- Pipeline
- Kubernetes
- Docker
- SonarQube Scanner
- JFrog Artifactory
- Git
- Email Extension

#### Create Jenkins Credentials

```bash
# GitHub credentials
Username: github_user
Password: github_token
ID: github-credentials

# Artifactory credentials
Username: artifactory_user
Password: artifactory_password
ID: artifactory-credentials

# Docker credentials
Username: docker_user
Password: docker_password
ID: jfrog-docker-credentials

# SonarQube token
Secret text: sonarqube_token
ID: sonarqube-token

# Nexus IQ token
Secret text: nexus-iq-token
ID: nexus-iq-token

# Vault token
Secret text: vault_token
ID: vault-token

# Kubeconfig
File: /path/to/kubeconfig
ID: kubeconfig
```

#### Configure Global Variables

```groovy
Jenkins → Manage Jenkins → Configure System → Global properties

ARTIFACTORY_URL = https://artifactory.example.com
SONARQUBE_URL = https://sonarqube.example.com
NEXUS_IQ_URL = https://nexus-iq.example.com
DOCKER_REGISTRY_URL = https://jfrog-docker-registry.example.com
VAULT_ADDR = https://vault.example.com
```

### 2. Configure Shared Library in Jenkins

```
Jenkins → Manage Jenkins → Configure System → Global Pipeline Libraries

Name: jenkins-shared-library
Default version: main
Modern SCM: 
  - Repository URL: https://github.com/your-org/jenkins-shared-library.git
  - Credentials: github-credentials
```

### 3. Set up Microservice Repository

#### Clone Template Files

```bash
git clone https://github.com/your-org/jenkins-shared-library.git
cp Jenkinsfile your-microservice-repo/
cp config.yaml.example your-microservice-repo/config.yaml
cp -r deployment/ your-microservice-repo/
```

#### Customize config.yaml

```yaml
# In your-microservice-repo/config.yaml

app:
  name: "my-service"
  version: "1.0.0"

git:
  repository: "https://github.com/your-org/my-service.git"

kubernetes:
  namespace: "my-service-dev"
  cluster_url: "https://k8s-cluster.example.com"

docker:
  registry_url: "https://jfrog-docker-registry.example.com"
```

#### Update Jenkinsfile Parameters

```groovy
// In Jenkinsfile

parameters {
    string(name: 'GIT_REPO', 
           defaultValue: 'https://github.com/your-org/my-service.git')
    choice(name: 'ENVIRONMENT', 
           choices: ['dev', 'uat', 'prod'])
}
```

### 4. Create Jenkins Pipeline Job

```
Jenkins → New Item → Pipeline

Name: my-service-cicd
Definition: Pipeline script from SCM
SCM: Git
Repository URL: https://github.com/your-org/my-service.git
Script Path: Jenkinsfile
```

## Usage

### Running the Pipeline

```bash
# Via Jenkins UI
Jenkins → my-service-cicd → Build with Parameters
  - GIT_REPO: https://github.com/your-org/my-service.git
  - GIT_BRANCH: main
  - ENVIRONMENT: dev
  - PROJECT_VERSION: 1.0.0

# Via CLI
curl -X POST http://jenkins.example.com/job/my-service-cicd/buildWithParameters \
  -u jenkins_user:jenkins_token \
  -F GIT_REPO=https://github.com/your-org/my-service.git \
  -F GIT_BRANCH=main \
  -F ENVIRONMENT=dev \
  -F PROJECT_VERSION=1.0.0
```

### Configuration Examples

#### Minimal config.yaml

```yaml
app:
  name: "my-service"

git:
  repository: "https://github.com/your-org/my-service.git"

kubernetes:
  namespace: "default"
  cluster_url: "https://k8s.example.com"
```

#### Full config.yaml with Secret Management

```yaml
app:
  name: "my-service"

vault:
  enabled: true
  address: "https://vault.example.com"
  credentials_id: "vault-token"

secret_config:
  create_secret: true
  secret_name: "my-service-secret"
  vault_path: "secret/data/my-service"
  mount_path: "/etc/secrets"

kubernetes:
  namespace: "my-service-prod"
  context: "prod-cluster"
```

### Environment-Specific Deployments

#### Dev Deployment

```bash
# Minimal resources, debug logging
ENVIRONMENT: dev
Replicas: 1
CPU: 100m, Memory: 128Mi
Log Level: DEBUG
```

#### UAT Deployment

```bash
# Medium resources, info logging, autoscaling
ENVIRONMENT: uat
Replicas: 2-4 (HPA enabled)
CPU: 250m, Memory: 256Mi
Log Level: INFO
```

#### Production Deployment

```bash
# High resources, optimized logging, autoscaling
ENVIRONMENT: prod
Replicas: 3-10 (HPA enabled)
CPU: 500m, Memory: 512Mi
Log Level: WARN
Load Balancer with Ingress
```

## Secret Management

### Using HashiCorp Vault

Enable secret management in `config.yaml`:

```yaml
vault:
  enabled: true
  address: "https://vault.example.com"
  credentials_id: "vault-token"

secret_config:
  create_secret: true
  secret_name: "my-service-secret"
  vault_path: "secret/data/my-service"
  mount_path: "/etc/secrets"
```

### Vault Secret Format

```json
{
  "data": {
    "DB_PASSWORD": "password123",
    "API_KEY": "key123",
    "SSL_CERT": "cert_content"
  }
}
```

### Manual Secret Creation

```bash
# Create Kubernetes secret manually
kubectl create secret generic my-service-secret \
  --from-literal=DB_PASSWORD=password123 \
  --from-literal=API_KEY=key123 \
  -n my-service-dev

# Reference in Helm values
secrets:
  enabled: true
  data:
    secretName: "my-service-secret"
```

## Helm Chart Deployment

### Standard Chart Structure

The `standard-app-chart` includes:
- Deployment with multiple replicas
- Service (ClusterIP, LoadBalancer)
- ConfigMap for application config
- Secrets for sensitive data
- PersistentVolumeClaim for data
- HorizontalPodAutoscaler
- Ingress for external access
- ServiceAccount and RBAC

### Override Values per Environment

```bash
# Dev
helm install my-service ./deployment/chart \
  -f deployment/values-dev.yaml \
  -n my-service-dev

# UAT
helm install my-service ./deployment/chart \
  -f deployment/values-uat.yaml \
  -n my-service-uat

# Production
helm install my-service ./deployment/chart \
  -f deployment/values-prod.yaml \
  -n my-service-prod
```

### Helm Values Override

```yaml
# deployment/values-custom.yaml
replicaCount: 5
image:
  tag: "1.2.0"
resources:
  limits:
    cpu: 2000m
    memory: 2Gi
```

## CI/CD Flow Details

### Stage 1: Checkout
- Clones source repository
- Loads `config.yaml`
- Validates prerequisites

### Stage 2: Build
- Runs Maven clean compile
- Packages JAR artifact

### Stage 3: Unit Test
- Executes `mvn test`
- Generates JaCoCo coverage reports
- Archives test results

### Stage 4: SonarQube
- Runs code quality analysis
- Checks quality gate
- Fails if quality gate fails

### Stage 5: Nexus IQ
- Scans dependencies for vulnerabilities
- Generates IQ report
- Can fail on policy violations

### Stage 6: Artifact Upload
- Uploads JAR to JFrog Artifactory
- Tags with version and build number
- Stores artifact path for later use

### Stage 7: Docker Build
- Creates Docker image
- Tags with registry, project name, version
- Builds from Dockerfile in source repo

### Stage 8: Trivy Scan
- Scans Docker image for vulnerabilities
- Fails on CRITICAL or HIGH severity
- Generates JSON report

### Stage 9: Registry Push
- Authenticates to Docker registry
- Pushes image with version tag
- Pushes latest tag

### Stage 10: Secret Management (Optional)
- Connects to HashiCorp Vault
- Downloads secrets as JSON
- Creates/updates Kubernetes Secret

### Stage 11: Helm Deploy
- Downloads Helm chart from JFrog
- Prepares environment-specific values
- Deploys with Helm upgrade --install
- Waits for pods to be ready
- Verifies deployment status

## Troubleshooting

### Build Failures

```bash
# Check Maven build
mvn clean compile -X

# Verify Dockerfile
docker build -t test-image .

# Check kubeconfig
kubectl cluster-info
```

### SonarQube Quality Gate Failure

```bash
# Check quality gate status
curl -s http://sonarqube:9000/api/qualitygates/project_status \
  -H "Authorization: Bearer $TOKEN"
```

### Helm Deployment Issues

```bash
# Check Helm release status
helm status my-service -n my-service-dev

# View Helm values
helm values my-service -n my-service-dev

# Check pod status
kubectl get pods -n my-service-dev
kubectl describe pod <pod-name> -n my-service-dev
```

### Container Registry Authentication

```bash
# Test Docker login
docker login -u $USER -p $PASSWORD $REGISTRY_URL

# Verify image push
docker push $REGISTRY_URL/my-service:latest
```

## Performance Optimization

### Build Optimization
- Use Docker layer caching
- Skip tests with `skipTests=true` if needed
- Use Maven offline mode with `-o`

### Deployment Optimization
- Use image pull policies (IfNotPresent)
- Implement pod disruption budgets
- Use node affinity rules

### Monitoring
- Enable Prometheus metrics
- Configure Grafana dashboards
- Set up alert rules

## Security Best Practices

1. **Secrets Management**
   - Use HashiCorp Vault for centralized secrets
   - Never store secrets in config.yaml
   - Rotate secrets regularly

2. **Container Security**
   - Run Trivy scans on all images
   - Use minimal base images
   - Apply resource limits

3. **Kubernetes Security**
   - Enable network policies
   - Use RBAC with service accounts
   - Run pods as non-root user

4. **Pipeline Security**
   - Use credential managers
   - Implement code review gates
   - Log all deployments

## Advanced Configuration

### Custom Health Checks

```yaml
# config.yaml
healthCheck:
  path: /api/health
  port: 8080
  initialDelaySeconds: 30
```

### Auto-scaling Configuration

```yaml
# deployment/values-prod.yaml
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

### Custom Network Policies

```yaml
# deployment/values-prod.yaml
networkPolicy:
  enabled: true
  policyTypes:
    - Ingress
    - Egress
```

## Support and Troubleshooting

For issues:
1. Check Jenkins logs: `Jenkins → Manage Jenkins → System Log`
2. Review pipeline build logs
3. Validate `config.yaml` syntax
4. Check Kubernetes events: `kubectl describe pod <name>`
5. Verify all credentials are configured

## Contributing

To extend this pipeline:
1. Add new Groovy class in `src/com/company/`
2. Import and call from `vars/buildPipeline.groovy`
3. Update documentation
4. Test with sample repository

## License

This CI/CD pipeline is provided as-is for internal organizational use.
