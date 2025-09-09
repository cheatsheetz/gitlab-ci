# GitLab CI/CD Cheat Sheet

## Table of Contents
- [Pipeline Configuration](#pipeline-configuration)
- [Runners](#runners)
- [Stages and Jobs](#stages-and-jobs)
- [Variables](#variables)
- [Docker Integration](#docker-integration)
- [Artifacts and Caching](#artifacts-and-caching)
- [Deployment Strategies](#deployment-strategies)
- [Best Practices and Security](#best-practices-and-security)

## Pipeline Configuration

### Basic .gitlab-ci.yml Structure
```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  NODE_VERSION: "16"
  DOCKER_DRIVER: overlay2

before_script:
  - echo "Starting CI/CD pipeline"
  - date

after_script:
  - echo "Pipeline completed"
  - df -h

build_job:
  stage: build
  image: node:${NODE_VERSION}
  script:
    - npm install
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour
  only:
    - main
    - develop

test_job:
  stage: test
  image: node:${NODE_VERSION}
  script:
    - npm install
    - npm test
  coverage: '/Statements\s*:\s*(\d+(?:\.\d+)?)%/'
  artifacts:
    reports:
      junit: reports/junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura.xml

deploy_job:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache curl
  script:
    - echo "Deploying application"
    - curl -X POST $DEPLOY_WEBHOOK_URL
  environment:
    name: production
    url: https://myapp.com
  when: manual
  only:
    - main
```

### Advanced Pipeline Features
```yaml
# Complex pipeline with multiple environments
stages:
  - build
  - test
  - security
  - deploy:staging
  - deploy:production

# Global variables
variables:
  POSTGRES_DB: test_db
  POSTGRES_USER: runner
  POSTGRES_PASSWORD: ""
  POSTGRES_HOST_AUTH_METHOD: trust

# Services for testing
services:
  - postgres:13
  - redis:6

# Include external configurations
include:
  - local: '/templates/.gitlab-ci-template.yml'
  - project: 'group/ci-templates'
    file: '/templates/security.yml'
  - remote: 'https://gitlab.com/awesome-project/raw/main/.gitlab-ci-template.yml'

# Conditional includes
include:
  - local: '/ci/build.yml'
    rules:
      - if: $CI_PIPELINE_SOURCE == "merge_request_event"

# Parallel jobs with matrix
test:
  stage: test
  parallel:
    matrix:
      - NODE_VERSION: ["14", "16", "18"]
        OS: ["ubuntu", "alpine"]
  image: node:$NODE_VERSION-$OS
  script:
    - npm test

# Job dependencies
deploy:staging:
  stage: deploy:staging
  needs:
    - job: build
      artifacts: true
    - job: test:unit
      artifacts: false
  script:
    - echo "Deploying to staging"

# Conditional execution
deploy:production:
  stage: deploy:production
  script:
    - echo "Deploying to production"
  rules:
    - if: $CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"
    - if: $CI_COMMIT_TAG
    - when: manual
      allow_failure: false
```

## Runners

### Runner Types and Configuration
```yaml
# Specific runner tags
build:docker:
  tags:
    - docker
    - linux
  script:
    - docker build -t myapp .

# Kubernetes executor
deploy:k8s:
  tags:
    - kubernetes
  image: kubectl:latest
  script:
    - kubectl apply -f deployment.yaml

# Shell executor
system:test:
  tags:
    - shell
    - privileged
  script:
    - sudo systemctl status nginx

# Windows runner
build:windows:
  tags:
    - windows
    - powershell
  script:
    - .\build.ps1
```

### Custom Runner Configuration
```toml
# config.toml for GitLab Runner
concurrent = 4
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "docker-runner"
  url = "https://gitlab.com/"
  token = "your-token"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
      ServerAddress = "s3.amazonaws.com"
      BucketName = "gitlab-runner-cache"
      BucketLocation = "us-east-1"
  [runners.docker]
    tls_verify = false
    image = "alpine:latest"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 0

# Kubernetes executor config
[[runners]]
  name = "k8s-runner"
  url = "https://gitlab.com/"
  token = "your-token"
  executor = "kubernetes"
  [runners.kubernetes]
    host = "https://kubernetes.default"
    namespace = "gitlab-runner"
    privileged = true
    image = "alpine:latest"
    [[runners.kubernetes.volumes.host_path]]
      name = "docker-sock"
      mount_path = "/var/run/docker.sock"
      host_path = "/var/run/docker.sock"
```

## Stages and Jobs

### Job Control and Flow
```yaml
# Sequential stages
stages:
  - prepare
  - build
  - test
  - security
  - package
  - deploy
  - cleanup

# Job inheritance
.base_job: &base
  image: alpine:latest
  before_script:
    - apk add --no-cache curl git
  after_script:
    - cleanup.sh

prepare:env:
  <<: *base
  stage: prepare
  script:
    - setup-environment.sh

# Hidden jobs (templates)
.deploy_template: &deploy_template
  stage: deploy
  script:
    - deploy.sh $ENVIRONMENT
  only:
    variables:
      - $DEPLOY_ENABLED == "true"

deploy:staging:
  <<: *deploy_template
  variables:
    ENVIRONMENT: staging
  environment:
    name: staging
    url: https://staging.myapp.com

deploy:production:
  <<: *deploy_template
  variables:
    ENVIRONMENT: production
  environment:
    name: production
    url: https://myapp.com
  when: manual

# DAG (Directed Acyclic Graph) jobs
build:frontend:
  stage: build
  script:
    - npm run build:frontend
  artifacts:
    paths:
      - frontend/dist/

build:backend:
  stage: build
  script:
    - go build -o backend ./cmd/server
  artifacts:
    paths:
      - backend

test:integration:
  stage: test
  needs:
    - build:frontend
    - build:backend
  script:
    - run-integration-tests.sh

# Retry and timeout
flaky:test:
  stage: test
  script:
    - run-flaky-tests.sh
  retry:
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure
  timeout: 30m

# Job with exit codes
custom:job:
  script:
    - exit_code=0
    - run-tests.sh || exit_code=$?
    - if [ $exit_code -eq 1 ]; then echo "Tests failed"; exit 1; fi
    - if [ $exit_code -eq 2 ]; then echo "Tests passed with warnings"; exit 0; fi
  allow_failure:
    exit_codes:
      - 137
      - 143
```

## Variables

### Variable Types and Scopes
```yaml
# Global variables
variables:
  GLOBAL_VAR: "global_value"
  DATABASE_URL: "postgres://user:pass@localhost/db"

# Job-specific variables
job:with:vars:
  variables:
    JOB_VAR: "job_value"
    DEBUG: "true"
  script:
    - echo "Global: $GLOBAL_VAR"
    - echo "Job: $JOB_VAR"

# Environment-specific variables
deploy:staging:
  variables:
    ENVIRONMENT: staging
    DEBUG: "true"
    REPLICAS: "2"
  environment:
    name: staging

deploy:production:
  variables:
    ENVIRONMENT: production
    DEBUG: "false"
    REPLICAS: "5"
  environment:
    name: production

# Conditional variables
variables:
  DEPLOY_STRATEGY: 
    value: "blue-green"
    description: "Deployment strategy"
  BUILD_TYPE:
    value: $([[ "$CI_COMMIT_BRANCH" == "main" ]] && echo "release" || echo "debug")

# File-based variables
load:env:
  script:
    - export $(cat .env | grep -v '^#' | xargs)
    - echo "Loaded environment variables"

# Protected variables (only available in protected branches/tags)
deploy:secure:
  script:
    - echo "Using protected variable: $SECURE_API_KEY"
  only:
    - main
    - /^v\d+\.\d+\.\d+$/
```

### Built-in Variables
```yaml
info:job:
  script:
    - echo "Project: $CI_PROJECT_NAME"
    - echo "Branch: $CI_COMMIT_BRANCH"
    - echo "Commit: $CI_COMMIT_SHA"
    - echo "Tag: $CI_COMMIT_TAG"
    - echo "Pipeline ID: $CI_PIPELINE_ID"
    - echo "Job ID: $CI_JOB_ID"
    - echo "Runner: $CI_RUNNER_DESCRIPTION"
    - echo "Build directory: $CI_BUILDS_DIR"
    - echo "Project directory: $CI_PROJECT_DIR"

# Using variables in rules
deploy:
  script:
    - deploy.sh
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG && $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      variables:
        ENVIRONMENT: "review"

# Variable interpolation
build:
  variables:
    VERSION: ${CI_COMMIT_TAG:-${CI_COMMIT_SHORT_SHA}}
    IMAGE_TAG: "${CI_PROJECT_NAME}:${VERSION}"
  script:
    - docker build -t $IMAGE_TAG .
    - echo "Built image: $IMAGE_TAG"
```

## Docker Integration

### Docker-in-Docker (DinD)
```yaml
# Docker-in-Docker configuration
variables:
  DOCKER_HOST: tcp://docker:2376
  DOCKER_TLS_CERTDIR: "/certs"
  DOCKER_TLS_VERIFY: 1
  DOCKER_CERT_PATH: "$DOCKER_TLS_CERTDIR/client"

services:
  - docker:20.10.16-dind

before_script:
  - docker info
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

build:docker:
  stage: build
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest

# Multi-stage Docker build
build:optimized:
  stage: build
  script:
    - docker build --target production -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker build --target development -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA-dev .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA-dev

# Docker Compose integration
test:integration:
  stage: test
  services:
    - docker:20.10.16-dind
  before_script:
    - apk add --no-cache docker-compose
  script:
    - docker-compose -f docker-compose.test.yml up -d
    - docker-compose -f docker-compose.test.yml exec -T app npm test
    - docker-compose -f docker-compose.test.yml down
```

### Container Registry
```yaml
# Using GitLab Container Registry
variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG
  LATEST_TAG: $CI_REGISTRY_IMAGE:latest

build:
  stage: build
  script:
    - docker build -t $IMAGE_TAG .
    - docker tag $IMAGE_TAG $LATEST_TAG
    - docker push $IMAGE_TAG
    - docker push $LATEST_TAG
  only:
    - main
    - develop

# Using external registries
build:aws:
  stage: build
  before_script:
    - apk add --no-cache aws-cli
    - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
  script:
    - docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/myapp:$CI_COMMIT_SHA .
    - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/myapp:$CI_COMMIT_SHA

# Vulnerability scanning
security:container:
  stage: security
  image: 
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image --exit-code 0 --format json --output container-scanning.json $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  artifacts:
    reports:
      container_scanning: container-scanning.json
```

## Artifacts and Caching

### Artifact Management
```yaml
# Basic artifacts
build:
  stage: build
  script:
    - make build
  artifacts:
    paths:
      - build/
      - dist/
    expire_in: 1 week
    when: on_success

# Conditional artifacts
test:
  stage: test
  script:
    - npm test
  artifacts:
    paths:
      - coverage/
    reports:
      junit: reports/junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura.xml
    expire_in: 30 days
    when: always

# Exclude patterns
build:full:
  script:
    - make build-all
  artifacts:
    paths:
      - build/
    exclude:
      - build/*.tmp
      - build/debug/

# Dynamic artifacts
generate:report:
  script:
    - ./generate-report.sh
    - echo "report-$(date +%Y%m%d).html" > artifact_name
  artifacts:
    name: "report-$CI_JOB_ID"
    paths:
      - "*.html"
    reports:
      junit: report.xml
```

### Caching Strategies
```yaml
# Global cache configuration
variables:
  CACHE_COMPRESSION_LEVEL: "1"
  CACHE_REQUEST_TIMEOUT: 5

# Node.js cache
build:node:
  image: node:16
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
      - .npm/
  before_script:
    - npm ci --cache .npm --prefer-offline
  script:
    - npm run build

# Multi-level caching
test:jest:
  cache:
    - key: node-modules-$CI_COMMIT_REF_SLUG
      paths:
        - node_modules/
      policy: pull
    - key: jest-cache-$CI_COMMIT_REF_SLUG
      paths:
        - .jest-cache/
      policy: pull-push
  script:
    - npm test -- --cacheDirectory=.jest-cache

# Conditional caching
build:conditional:
  cache:
    key: "$CI_JOB_NAME-$CI_COMMIT_REF_SLUG"
    paths:
      - vendor/
    policy: pull-push
    when: on_success
  script:
    - composer install --prefer-dist --no-progress --no-suggest --optimize-autoloader

# Cache with fallback keys
build:fallback:
  cache:
    key: 
      files:
        - composer.lock
      prefix: composer
    paths:
      - vendor/
    fallback_keys:
      - composer-$CI_COMMIT_REF_SLUG
      - composer-main
      - composer
  script:
    - composer install
```

## Deployment Strategies

### Environment Deployment
```yaml
# Review apps
review:
  stage: deploy
  script:
    - deploy-to-review.sh $CI_COMMIT_REF_SLUG
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_COMMIT_REF_SLUG.review.myapp.com
    on_stop: stop_review
  only:
    - merge_requests

stop_review:
  stage: deploy
  script:
    - cleanup-review.sh $CI_COMMIT_REF_SLUG
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  when: manual
  only:
    - merge_requests

# Staging deployment
deploy:staging:
  stage: deploy
  script:
    - kubectl apply -f k8s/staging/
    - kubectl rollout status deployment/myapp -n staging
  environment:
    name: staging
    url: https://staging.myapp.com
    kubernetes:
      namespace: staging
  only:
    - develop

# Production deployment with approval
deploy:production:
  stage: deploy
  script:
    - kubectl apply -f k8s/production/
    - kubectl rollout status deployment/myapp -n production
  environment:
    name: production
    url: https://myapp.com
    kubernetes:
      namespace: production
  when: manual
  only:
    - main
```

### Advanced Deployment Patterns
```yaml
# Blue-Green deployment
deploy:blue-green:
  stage: deploy
  script:
    - |
      if [ "$CURRENT_SLOT" = "blue" ]; then
        DEPLOY_SLOT="green"
      else
        DEPLOY_SLOT="blue"
      fi
    - deploy-to-slot.sh $DEPLOY_SLOT
    - run-smoke-tests.sh $DEPLOY_SLOT
    - switch-traffic.sh $DEPLOY_SLOT
  environment:
    name: production
    url: https://myapp.com
  when: manual

# Canary deployment
deploy:canary:
  stage: deploy
  script:
    - deploy-canary.sh 10  # 10% traffic
    - sleep 300
    - check-metrics.sh
    - deploy-canary.sh 50  # 50% traffic
    - sleep 300
    - check-metrics.sh
    - deploy-canary.sh 100 # 100% traffic
  environment:
    name: production
    url: https://myapp.com

# Database migrations
migrate:database:
  stage: deploy
  image: postgres:13
  before_script:
    - apt-get update && apt-get install -y postgresql-client
  script:
    - pg_dump $DATABASE_URL > backup.sql
    - psql $DATABASE_URL < migrations/migrate.sql
  artifacts:
    paths:
      - backup.sql
    expire_in: 1 day
  when: manual
  environment:
    name: production
```

## Best Practices and Security

### Security Best Practices
```yaml
# Secure variable handling
deploy:secure:
  script:
    - echo "Deploying with secure credentials"
    - deploy.sh
  variables:
    KUBECONFIG: $KUBE_CONFIG_BASE64
  only:
    variables:
      - $CI_COMMIT_REF_PROTECTED == "true"

# Image scanning
security:scan:
  stage: security
  image: 
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy fs --exit-code 1 --severity HIGH,CRITICAL .
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  allow_failure: false

# Dependency scanning
dependency:check:
  stage: security
  image: node:16
  script:
    - npm audit --audit-level moderate
  allow_failure: true
  only:
    - main
    - merge_requests

# SAST scanning
sast:
  stage: security
  image: registry.gitlab.com/gitlab-org/security-products/analyzers/semgrep:latest
  script:
    - semgrep --config=auto --json --output=sast-report.json .
  artifacts:
    reports:
      sast: sast-report.json
  only:
    - main
    - merge_requests

# Secret detection
secrets:
  stage: security
  image: registry.gitlab.com/gitlab-org/security-products/analyzers/secrets:latest
  script:
    - /analyzer run
  artifacts:
    reports:
      secret_detection: gl-secret-detection-report.json
  only:
    - main
    - merge_requests
```

### Performance and Optimization
```yaml
# Efficient pipeline structure
workflow:
  rules:
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
      when: never
    - if: $CI_COMMIT_BRANCH

# Fail fast
validate:
  stage: .pre
  script:
    - validate-syntax.sh
    - check-dependencies.sh
  rules:
    - changes:
      - "**/*.yml"
      - "**/*.yaml"
      - package*.json
      - requirements*.txt

# Parallel testing
test:
  stage: test
  parallel: 4
  script:
    - npm run test -- --ci --testNamePattern=".*${CI_NODE_INDEX}.*"

# Smart caching
build:optimized:
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - node_modules/
    policy: pull-push
  script:
    - npm ci
    - npm run build

# Resource limits
resource:intensive:
  tags:
    - high-memory
  variables:
    KUBERNETES_MEMORY_LIMIT: 4Gi
    KUBERNETES_CPU_LIMIT: 2
  script:
    - run-memory-intensive-task.sh
```

### Code Quality Integration
```yaml
# Code quality checks
code:quality:
  stage: test
  image: sonarsource/sonar-scanner-cli:latest
  script:
    - sonar-scanner
      -Dsonar.projectKey=$CI_PROJECT_NAME
      -Dsonar.sources=.
      -Dsonar.host.url=$SONAR_HOST_URL
      -Dsonar.login=$SONAR_TOKEN
  only:
    - main
    - merge_requests

# Test coverage
test:coverage:
  stage: test
  script:
    - npm run test:coverage
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura.xml

# Performance testing
test:performance:
  stage: test
  image: loadimpact/k6:latest
  script:
    - k6 run --out json=performance.json performance-tests.js
  artifacts:
    reports:
      performance: performance.json
  only:
    - main
```

## Official Documentation Links

- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [GitLab CI/CD YAML Reference](https://docs.gitlab.com/ee/ci/yaml/)
- [GitLab Runner Documentation](https://docs.gitlab.com/runner/)
- [GitLab Docker Integration](https://docs.gitlab.com/ee/ci/docker/)
- [GitLab Environment Deployments](https://docs.gitlab.com/ee/ci/environments/)
- [GitLab Security Scanning](https://docs.gitlab.com/ee/user/application_security/)
- [GitLab Pages](https://docs.gitlab.com/ee/user/project/pages/)
- [GitLab Container Registry](https://docs.gitlab.com/ee/user/packages/container_registry/)