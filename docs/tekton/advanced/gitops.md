# GitOps with Tekton

GitOps is a way of implementing Continuous Deployment for cloud native applications. This guide shows how to implement GitOps patterns using Tekton pipelines with Jenkins integration.

## GitOps Principles

1. **Declarative**: The entire system is described declaratively
2. **Versioned and Immutable**: The canonical desired system state is versioned in Git
3. **Pulled Automatically**: Software agents automatically pull the desired state declarations from Git
4. **Continuously Reconciled**: Software agents continuously observe actual system state and attempt to apply the desired state

## GitOps Architecture with Tekton

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Source Code   │    │   Config Repo   │    │   Target Env    │
│   Repository    │    │   (GitOps)      │    │   (Kubernetes)  │
│                 │    │                 │    │                 │
│  ┌───────────┐  │    │  ┌───────────┐  │    │  ┌───────────┐  │
│  │Application│  │    │  │Kubernetes │  │    │  │Application│  │
│  │   Code    │  │    │  │Manifests  │  │    │  │ Running   │  │
│  └───────────┘  │    │  └───────────┘  │    │  └───────────┘  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         │ git push              │ git push              │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Tekton CI      │    │  Tekton CD      │    │  ArgoCD/Flux    │
│  Pipeline       │────│  Pipeline       │────│  GitOps Agent   │
│  (Build/Test)   │    │  (Deploy)       │    │  (Sync)         │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## Setting Up GitOps with Tekton

### 1. Source Repository Pipeline

```yaml
# CI Pipeline for source code changes
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: gitops-ci-pipeline
spec:
  params:
    - name: git-url
      description: Source repository URL
    - name: git-revision
      description: Git revision to build
    - name: image-name
      description: Container image name
    - name: config-repo-url
      description: GitOps configuration repository
  workspaces:
    - name: source-code
    - name: config-repo
  tasks:
    # Build and test application
    - name: clone-source
      taskRef:
        name: git-clone-task
      params:
        - name: url
          value: $(params.git-url)
        - name: revision
          value: $(params.git-revision)
      workspaces:
        - name: output
          workspace: source-code
    
    - name: run-tests
      runAfter: [clone-source]
      taskRef:
        name: run-tests
      workspaces:
        - name: source
          workspace: source-code
    
    - name: build-image
      runAfter: [run-tests]
      taskRef:
        name: kaniko-build
      params:
        - name: image-name
          value: $(params.image-name)
        - name: image-tag
          value: $(params.git-revision)
      workspaces:
        - name: source
          workspace: source-code
    
    # Update GitOps repository
    - name: update-config-repo
      runAfter: [build-image]
      taskRef:
        name: update-gitops-repo
      params:
        - name: config-repo-url
          value: $(params.config-repo-url)
        - name: image-name
          value: $(params.image-name)
        - name: image-tag
          value: $(params.git-revision)
        - name: app-name
          value: "myapp"
        - name: environment
          value: "staging"
      workspaces:
        - name: config-repo
          workspace: config-repo

---
# Task to update GitOps repository
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: update-gitops-repo
spec:
  params:
    - name: config-repo-url
      description: GitOps configuration repository URL
    - name: image-name
      description: Container image name
    - name: image-tag
      description: Container image tag
    - name: app-name
      description: Application name
    - name: environment
      description: Target environment
  workspaces:
    - name: config-repo
      description: GitOps configuration repository
  steps:
    - name: clone-config-repo
      image: alpine/git:latest
      workingDir: $(workspaces.config-repo.path)
      script: |
        #!/bin/sh
        set -e
        
        echo "Cloning GitOps configuration repository..."
        git clone $(params.config-repo-url) .
        
        echo "Current directory contents:"
        ls -la
    
    - name: update-manifests
      image: mikefarah/yq:latest
      workingDir: $(workspaces.config-repo.path)
      script: |
        #!/bin/sh
        set -e
        
        APP_NAME=$(params.app-name)
        ENVIRONMENT=$(params.environment)
        IMAGE_NAME=$(params.image-name)
        IMAGE_TAG=$(params.image-tag)
        
        # Path to the manifest file
        MANIFEST_PATH="environments/${ENVIRONMENT}/${APP_NAME}/deployment.yaml"
        
        echo "Updating manifest: $MANIFEST_PATH"
        echo "New image: ${IMAGE_NAME}:${IMAGE_TAG}"
        
        # Update the image in the deployment
        yq eval ".spec.template.spec.containers[0].image = \"${IMAGE_NAME}:${IMAGE_TAG}\"" -i $MANIFEST_PATH
        
        # Verify the change
        echo "Updated manifest:"
        cat $MANIFEST_PATH
    
    - name: commit-and-push
      image: alpine/git:latest
      workingDir: $(workspaces.config-repo.path)
      env:
        - name: GIT_AUTHOR_NAME
          value: "Tekton Pipeline"
        - name: GIT_AUTHOR_EMAIL
          value: "tekton@example.com"
        - name: GIT_COMMITTER_NAME
          value: "Tekton Pipeline"
        - name: GIT_COMMITTER_EMAIL
          value: "tekton@example.com"
      script: |
        #!/bin/sh
        set -e
        
        APP_NAME=$(params.app-name)
        ENVIRONMENT=$(params.environment)
        IMAGE_TAG=$(params.image-tag)
        
        # Configure git
        git config --global user.name "$GIT_AUTHOR_NAME"
        git config --global user.email "$GIT_AUTHOR_EMAIL"
        
        # Check for changes
        if git diff --quiet; then
          echo "No changes to commit"
          exit 0
        fi
        
        # Commit changes
        git add .
        git commit -m "Update ${APP_NAME} in ${ENVIRONMENT} to ${IMAGE_TAG}"
        
        # Push changes
        git push origin main
        
        echo "Successfully updated GitOps repository"
```

### 2. Deployment Pipeline (GitOps CD)

```yaml
# CD Pipeline triggered by GitOps repository changes
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: gitops-cd-pipeline
spec:
  params:
    - name: config-repo-url
      description: GitOps configuration repository URL
    - name: git-revision
      description: Git revision of config changes
    - name: environment
      description: Target environment
    - name: app-name
      description: Application name
  workspaces:
    - name: config-repo
  tasks:
    - name: clone-config
      taskRef:
        name: git-clone-task
      params:
        - name: url
          value: $(params.config-repo-url)
        - name: revision
          value: $(params.git-revision)
      workspaces:
        - name: output
          workspace: config-repo
    
    - name: validate-manifests
      runAfter: [clone-config]
      taskRef:
        name: validate-k8s-manifests
      params:
        - name: environment
          value: $(params.environment)
      workspaces:
        - name: source
          workspace: config-repo
    
    - name: deploy-to-env
      runAfter: [validate-manifests]
      taskRef:
        name: kubectl-deploy
      params:
        - name: environment
          value: $(params.environment)
        - name: app-name
          value: $(params.app-name)
      workspaces:
        - name: source
          workspace: config-repo
    
    - name: run-smoke-tests
      runAfter: [deploy-to-env]
      taskRef:
        name: smoke-tests
      params:
        - name: app-name
          value: $(params.app-name)
        - name: environment
          value: $(params.environment)
    
    - name: notify-deployment
      runAfter: [run-smoke-tests]
      taskRef:
        name: notification-task
      params:
        - name: message
          value: "Successfully deployed $(params.app-name) to $(params.environment)"
        - name: channels
          value: ["slack", "email"]

---
# Manifest validation task
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: validate-k8s-manifests
spec:
  params:
    - name: environment
      description: Environment to validate
  workspaces:
    - name: source
  steps:
    - name: kubeval-validation
      image: garethr/kubeval:latest
      workingDir: $(workspaces.source.path)
      script: |
        #!/bin/sh
        set -e
        
        ENVIRONMENT=$(params.environment)
        MANIFEST_DIR="environments/${ENVIRONMENT}"
        
        echo "Validating Kubernetes manifests in: $MANIFEST_DIR"
        
        if [ ! -d "$MANIFEST_DIR" ]; then
          echo "Error: Environment directory $MANIFEST_DIR does not exist"
          exit 1
        fi
        
        # Validate all YAML files
        find "$MANIFEST_DIR" -name "*.yaml" -o -name "*.yml" | while read -r file; do
          echo "Validating: $file"
          kubeval "$file"
        done
        
        echo "All manifests are valid"
    
    - name: security-scan
      image: aquasec/trivy:latest
      workingDir: $(workspaces.source.path)
      script: |
        #!/bin/sh
        set -e
        
        ENVIRONMENT=$(params.environment)
        MANIFEST_DIR="environments/${ENVIRONMENT}"
        
        echo "Running security scan on manifests..."
        
        # Scan Kubernetes manifests for security issues
        trivy config "$MANIFEST_DIR" --severity HIGH,CRITICAL
        
        echo "Security scan completed"

---
# Deployment task
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: kubectl-deploy
spec:
  params:
    - name: environment
      description: Target environment
    - name: app-name
      description: Application name
  workspaces:
    - name: source
  steps:
    - name: apply-manifests
      image: bitnami/kubectl:latest
      workingDir: $(workspaces.source.path)
      script: |
        #!/bin/bash
        set -e
        
        ENVIRONMENT=$(params.environment)
        APP_NAME=$(params.app-name)
        MANIFEST_DIR="environments/${ENVIRONMENT}/${APP_NAME}"
        
        echo "Deploying ${APP_NAME} to ${ENVIRONMENT} environment"
        
        # Create namespace if it doesn't exist
        kubectl create namespace "${ENVIRONMENT}" --dry-run=client -o yaml | kubectl apply -f -
        
        # Apply all manifests
        if [ -d "$MANIFEST_DIR" ]; then
          echo "Applying manifests from: $MANIFEST_DIR"
          kubectl apply -f "$MANIFEST_DIR" -n "${ENVIRONMENT}"
        else
          echo "Error: Manifest directory $MANIFEST_DIR not found"
          exit 1
        fi
        
        # Wait for deployment to be ready
        kubectl rollout status deployment/${APP_NAME} -n "${ENVIRONMENT}" --timeout=300s
        
        echo "Deployment completed successfully"
```

### 3. ArgoCD Integration

```yaml
# ArgoCD Application for GitOps
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-config
    targetRevision: HEAD
    path: environments/staging/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

---
# Tekton task to sync ArgoCD application
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: argocd-sync
spec:
  params:
    - name: app-name
      description: ArgoCD application name
    - name: argocd-server
      description: ArgoCD server URL
  steps:
    - name: sync-app
      image: argoproj/argocd:latest
      env:
        - name: ARGOCD_TOKEN
          valueFrom:
            secretKeyRef:
              name: argocd-token
              key: token
      script: |
        #!/bin/bash
        set -e
        
        APP_NAME=$(params.app-name)
        ARGOCD_SERVER=$(params.argocd-server)
        
        echo "Syncing ArgoCD application: $APP_NAME"
        
        # Login to ArgoCD
        argocd login $ARGOCD_SERVER --auth-token $ARGOCD_TOKEN --insecure
        
        # Sync application
        argocd app sync $APP_NAME --force
        
        # Wait for sync to complete
        argocd app wait $APP_NAME --timeout 300
        
        echo "Application sync completed"
```

### 4. Multi-Environment Promotion Pipeline

```yaml
# Pipeline for promoting changes across environments
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: promotion-pipeline
spec:
  params:
    - name: app-name
      description: Application name
    - name: source-env
      description: Source environment
    - name: target-env
      description: Target environment
    - name: config-repo-url
      description: GitOps configuration repository
  workspaces:
    - name: config-repo
  tasks:
    - name: clone-config
      taskRef:
        name: git-clone-task
      params:
        - name: url
          value: $(params.config-repo-url)
      workspaces:
        - name: output
          workspace: config-repo
    
    - name: extract-source-version
      runAfter: [clone-config]
      taskRef:
        name: get-deployed-version
      params:
        - name: app-name
          value: $(params.app-name)
        - name: environment
          value: $(params.source-env)
      workspaces:
        - name: source
          workspace: config-repo
    
    - name: promote-to-target
      runAfter: [extract-source-version]
      taskRef:
        name: update-gitops-repo
      params:
        - name: config-repo-url
          value: $(params.config-repo-url)
        - name: image-name
          value: "$(tasks.extract-source-version.results.image-name)"
        - name: image-tag
          value: "$(tasks.extract-source-version.results.image-tag)"
        - name: app-name
          value: $(params.app-name)
        - name: environment
          value: $(params.target-env)
      workspaces:
        - name: config-repo
          workspace: config-repo
    
    - name: run-integration-tests
      runAfter: [promote-to-target]
      when:
        - input: $(params.target-env)
          operator: in
          values: ["staging", "production"]
      taskRef:
        name: integration-tests
      params:
        - name: app-name
          value: $(params.app-name)
        - name: environment
          value: $(params.target-env)

---
# Task to get currently deployed version
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: get-deployed-version
spec:
  params:
    - name: app-name
      description: Application name
    - name: environment
      description: Environment to check
  workspaces:
    - name: source
  results:
    - name: image-name
      description: Current deployed image name
    - name: image-tag
      description: Current deployed image tag
  steps:
    - name: extract-version
      image: mikefarah/yq:latest
      workingDir: $(workspaces.source.path)
      script: |
        #!/bin/sh
        set -e
        
        APP_NAME=$(params.app-name)
        ENVIRONMENT=$(params.environment)
        MANIFEST_PATH="environments/${ENVIRONMENT}/${APP_NAME}/deployment.yaml"
        
        echo "Extracting version from: $MANIFEST_PATH"
        
        # Extract current image
        CURRENT_IMAGE=$(yq eval '.spec.template.spec.containers[0].image' "$MANIFEST_PATH")
        
        # Split image into name and tag
        IMAGE_NAME=$(echo "$CURRENT_IMAGE" | cut -d':' -f1)
        IMAGE_TAG=$(echo "$CURRENT_IMAGE" | cut -d':' -f2)
        
        echo "Current image: $CURRENT_IMAGE"
        echo "Image name: $IMAGE_NAME"
        echo "Image tag: $IMAGE_TAG"
        
        echo -n "$IMAGE_NAME" > $(results.image-name.path)
        echo -n "$IMAGE_TAG" > $(results.image-tag.path)
```

## Jenkins Integration with GitOps

### 1. Jenkins Job for Manual Approvals

```yaml
# Task to trigger Jenkins approval job
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: jenkins-approval-gate
spec:
  params:
    - name: jenkins-url
      description: Jenkins server URL
    - name: approval-job
      description: Jenkins approval job name
    - name: app-name
      description: Application name
    - name: source-env
      description: Source environment
    - name: target-env
      description: Target environment
  results:
    - name: approval-status
      description: Approval status (approved/rejected)
  steps:
    - name: request-approval
      image: curlimages/curl:latest
      env:
        - name: JENKINS_USER
          valueFrom:
            secretKeyRef:
              name: jenkins-credentials
              key: username
        - name: JENKINS_TOKEN
          valueFrom:
            secretKeyRef:
              name: jenkins-credentials
              key: token
      script: |
        #!/bin/sh
        set -e
        
        JENKINS_URL=$(params.jenkins-url)
        JOB_NAME=$(params.approval-job)
        APP_NAME=$(params.app-name)
        SOURCE_ENV=$(params.source-env)
        TARGET_ENV=$(params.target-env)
        
        echo "Requesting approval for promoting $APP_NAME from $SOURCE_ENV to $TARGET_ENV"
        
        # Trigger Jenkins approval job
        BUILD_NUMBER=$(curl -s -X POST \
          "$JENKINS_URL/job/$JOB_NAME/buildWithParameters" \
          --user "$JENKINS_USER:$JENKINS_TOKEN" \
          --data-urlencode "APP_NAME=$APP_NAME" \
          --data-urlencode "SOURCE_ENV=$SOURCE_ENV" \
          --data-urlencode "TARGET_ENV=$TARGET_ENV" \
          -D headers.txt | grep -i location | sed 's/.*\/\([0-9]*\)\/.*/\1/')
        
        echo "Jenkins job triggered with build number: $BUILD_NUMBER"
        
        # Wait for approval (polling)
        while true; do
          STATUS=$(curl -s "$JENKINS_URL/job/$JOB_NAME/$BUILD_NUMBER/api/json" \
            --user "$JENKINS_USER:$JENKINS_TOKEN" \
            | jq -r '.result')
          
          if [ "$STATUS" = "SUCCESS" ]; then
            echo "Approval granted"
            echo -n "approved" > $(results.approval-status.path)
            break
          elif [ "$STATUS" = "FAILURE" ]; then
            echo "Approval rejected"
            echo -n "rejected" > $(results.approval-status.path)
            exit 1
          else
            echo "Waiting for approval... (status: $STATUS)"
            sleep 30
          fi
        done
```

### 2. Production Promotion Pipeline

```yaml
# Production promotion with approval gate
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: production-promotion-pipeline
spec:
  params:
    - name: app-name
      description: Application name
    - name: config-repo-url
      description: GitOps configuration repository
    - name: jenkins-url
      description: Jenkins server URL
  workspaces:
    - name: config-repo
  tasks:
    - name: request-production-approval
      taskRef:
        name: jenkins-approval-gate
      params:
        - name: jenkins-url
          value: $(params.jenkins-url)
        - name: approval-job
          value: "production-approval"
        - name: app-name
          value: $(params.app-name)
        - name: source-env
          value: "staging"
        - name: target-env
          value: "production"
    
    - name: promote-to-production
      runAfter: [request-production-approval]
      when:
        - input: "$(tasks.request-production-approval.results.approval-status)"
          operator: in
          values: ["approved"]
      taskRef:
        name: promotion-pipeline
      params:
        - name: app-name
          value: $(params.app-name)
        - name: source-env
          value: "staging"
        - name: target-env
          value: "production"
        - name: config-repo-url
          value: $(params.config-repo-url)
      workspaces:
        - name: config-repo
          workspace: config-repo
```

## Best Practices

1. **Separate repositories**: Keep application code and configuration in separate repositories
2. **Environment parity**: Use identical configuration structure across environments
3. **Immutable artifacts**: Use immutable container images with unique tags
4. **Progressive deployment**: Implement canary or blue-green deployments
5. **Monitoring and observability**: Monitor both the GitOps process and deployed applications
6. **Security scanning**: Scan both code and configurations for security issues
7. **Rollback strategy**: Plan for quick rollbacks when issues occur
8. **Access control**: Implement proper RBAC for GitOps repositories

## Monitoring GitOps Pipelines

### 1. Pipeline Metrics

```yaml
# ServiceMonitor for Tekton metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: tekton-gitops-metrics
spec:
  selector:
    matchLabels:
      app: tekton-pipelines-controller
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

### 2. Alerting Rules

```yaml
# PrometheusRule for GitOps alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gitops-alerts
spec:
  groups:
    - name: gitops
      rules:
        - alert: GitOpsPipelineFailure
          expr: increase(tekton_pipeline_run_failed_total[5m]) > 0
          for: 0m
          labels:
            severity: critical
          annotations:
            summary: "GitOps pipeline failure detected"
            description: "Pipeline {{ $labels.pipeline }} has failed"
        
        - alert: GitOpsDeploymentStuck
          expr: time() - tekton_pipeline_run_start_time > 1800
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "GitOps deployment taking too long"
            description: "Pipeline {{ $labels.pipeline }} has been running for over 30 minutes"
```

## Resources

- [GitOps Principles](https://opengitops.dev/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Flux Documentation](https://fluxcd.io/docs/)
- [Tekton GitOps Examples](https://github.com/tektoncd/catalog/tree/main/task)