# Tekton Triggers - Automating Pipeline Execution

Tekton Triggers enable automatic pipeline execution in response to external events like Git pushes, pull requests, or webhook calls.

## Overview

Tekton Triggers consist of several components:
- **EventListener**: Listens for incoming HTTP events
- **Trigger**: Defines what happens when an event is received
- **TriggerTemplate**: Template for creating resources when triggered
- **TriggerBinding**: Extracts data from events and passes it to templates

## Installing Tekton Triggers

```bash
# Install Tekton Triggers
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml

# Verify installation
kubectl get pods --namespace tekton-pipelines
```

## Basic GitHub Webhook Trigger

### 1. TriggerTemplate

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: github-push-template
spec:
  params:
    - name: gitrepositoryurl
      description: The git repository URL
    - name: gitrevision
      description: The git revision
    - name: gitrepositoryname
      description: The name of the repository
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: github-push-pipeline-run-
      spec:
        pipelineRef:
          name: build-deploy-pipeline
        params:
          - name: repo-url
            value: $(tt.params.gitrepositoryurl)
          - name: revision
            value: $(tt.params.gitrevision)
          - name: repo-name
            value: $(tt.params.gitrepositoryname)
        workspaces:
          - name: shared-data
            volumeClaimTemplate:
              spec:
                accessModes:
                  - ReadWriteOnce
                resources:
                  requests:
                    storage: 1Gi
```

### 2. TriggerBinding

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: github-push-binding
spec:
  params:
    - name: gitrepositoryurl
      value: $(body.repository.clone_url)
    - name: gitrevision
      value: $(body.head_commit.id)
    - name: gitrepositoryname
      value: $(body.repository.name)
```

### 3. Trigger

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: Trigger
metadata:
  name: github-push-trigger
spec:
  serviceAccountName: tekton-triggers-sa
  interceptors:
    - ref:
        name: "github"
      params:
        - name: "secretRef"
          value:
            secretName: github-secret
            secretKey: secretToken
        - name: "eventTypes"
          value: ["push"]
  bindings:
    - ref: github-push-binding
  template:
    ref: github-push-template
```

### 4. EventListener

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: github-listener
spec:
  serviceAccountName: tekton-triggers-sa
  triggers:
    - triggerRef: github-push-trigger
```

## Advanced Trigger Patterns

### 1. Pull Request Trigger

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: github-pr-binding
spec:
  params:
    - name: gitrepositoryurl
      value: $(body.pull_request.head.repo.clone_url)
    - name: gitrevision
      value: $(body.pull_request.head.sha)
    - name: pullrequestnumber
      value: $(body.number)
    - name: pullrequestbaseref
      value: $(body.pull_request.base.ref)
    - name: pullrequestaction
      value: $(body.action)

---
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: github-pr-template
spec:
  params:
    - name: gitrepositoryurl
    - name: gitrevision
    - name: pullrequestnumber
    - name: pullrequestbaseref
    - name: pullrequestaction
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: pr-pipeline-run-
        labels:
          tekton.dev/pr-number: $(tt.params.pullrequestnumber)
      spec:
        pipelineRef:
          name: pr-validation-pipeline
        params:
          - name: repo-url
            value: $(tt.params.gitrepositoryurl)
          - name: revision
            value: $(tt.params.gitrevision)
          - name: pr-number
            value: $(tt.params.pullrequestnumber)
          - name: base-ref
            value: $(tt.params.pullrequestbaseref)
          - name: pr-action
            value: $(tt.params.pullrequestaction)
        workspaces:
          - name: shared-data
            volumeClaimTemplate:
              spec:
                accessModes:
                  - ReadWriteOnce
                resources:
                  requests:
                    storage: 1Gi

---
apiVersion: triggers.tekton.dev/v1beta1
kind: Trigger
metadata:
  name: github-pr-trigger
spec:
  interceptors:
    - ref:
        name: "github"
      params:
        - name: "secretRef"
          value:
            secretName: github-secret
            secretKey: secretToken
        - name: "eventTypes"
          value: ["pull_request"]
    - ref:
        name: "cel"
      params:
        - name: "filter"
          value: "body.action in ['opened', 'synchronize', 'reopened']"
  bindings:
    - ref: github-pr-binding
  template:
    ref: github-pr-template
```

### 2. Multi-Repository Trigger

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: multi-repo-binding
spec:
  params:
    - name: gitrepositoryurl
      value: $(body.repository.clone_url)
    - name: gitrevision
      value: $(body.head_commit.id)
    - name: gitrepositoryname
      value: $(body.repository.name)
    - name: gitorganization
      value: $(body.repository.owner.login)

---
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: multi-repo-template
spec:
  params:
    - name: gitrepositoryurl
    - name: gitrevision
    - name: gitrepositoryname
    - name: gitorganization
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: $(tt.params.gitorganization)-$(tt.params.gitrepositoryname)-
      spec:
        pipelineRef:
          name: universal-build-pipeline
        params:
          - name: repo-url
            value: $(tt.params.gitrepositoryurl)
          - name: revision
            value: $(tt.params.gitrevision)
          - name: repo-name
            value: $(tt.params.gitrepositoryname)
          - name: organization
            value: $(tt.params.gitorganization)
        workspaces:
          - name: shared-data
            volumeClaimTemplate:
              spec:
                accessModes:
                  - ReadWriteOnce
                resources:
                  requests:
                    storage: 1Gi

---
apiVersion: triggers.tekton.dev/v1beta1
kind: Trigger
metadata:
  name: multi-repo-trigger
spec:
  interceptors:
    - ref:
        name: "github"
      params:
        - name: "secretRef"
          value:
            secretName: github-secret
            secretKey: secretToken
        - name: "eventTypes"
          value: ["push"]
    - ref:
        name: "cel"
      params:
        - name: "filter"
          value: "body.repository.owner.login in ['myorg', 'partnerorg']"
        - name: "overlays"
          value:
            - key: pipeline_name
              expression: "body.repository.name + '-pipeline'"
  bindings:
    - ref: multi-repo-binding
  template:
    ref: multi-repo-template
```

## Custom Interceptors

### 1. CEL Interceptor for Complex Filtering

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: Trigger
metadata:
  name: conditional-trigger
spec:
  interceptors:
    - ref:
        name: "cel"
      params:
        - name: "filter"
          value: |
            // Only trigger on pushes to main branch with specific file changes
            body.ref == 'refs/heads/main' &&
            body.commits.exists(c, c.modified.exists(f, f.startsWith('src/')) || 
                                   c.added.exists(f, f.startsWith('src/')))
        - name: "overlays"
          value:
            - key: changed_files
              expression: |
                body.commits.map(c, c.modified + c.added + c.removed).flatten().filter(f, f.startsWith('src/'))
            - key: commit_count
              expression: "size(body.commits)"
  bindings:
    - ref: github-push-binding
  template:
    ref: github-push-template
```

### 2. Custom Webhook Interceptor

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-interceptor-config
data:
  script.sh: |
    #!/bin/bash
    # Custom processing logic
    echo "Processing webhook payload..."
    
    # Extract relevant information
    REPO_NAME=$(echo $WEBHOOK_PAYLOAD | jq -r '.repository.name')
    BRANCH=$(echo $WEBHOOK_PAYLOAD | jq -r '.ref' | sed 's/refs\/heads\///')
    
    # Custom validation
    if [ "$BRANCH" != "main" ] && [ "$BRANCH" != "develop" ]; then
      echo "Ignoring push to branch: $BRANCH"
      exit 1
    fi
    
    # Pass through modified payload
    echo $WEBHOOK_PAYLOAD | jq '.custom_field = "processed"'

---
apiVersion: triggers.tekton.dev/v1beta1
kind: ClusterInterceptor
metadata:
  name: custom-webhook-interceptor
spec:
  clientConfig:
    service:
      name: custom-interceptor-service
      namespace: tekton-pipelines
      path: "/process"
```

## EventListener Configuration

### 1. Secure EventListener

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: secure-github-listener
spec:
  serviceAccountName: tekton-triggers-sa
  config:
    logging:
      level: info
    auth:
      git:
        secretName: github-auth-secret
  resources:
    kubernetesResource:
      spec:
        template:
          spec:
            serviceAccountName: tekton-triggers-sa
            containers:
              - name: event-listener
                resources:
                  requests:
                    memory: "64Mi"
                    cpu: "250m"
                  limits:
                    memory: "128Mi"
                    cpu: "500m"
  triggers:
    - name: github-push
      interceptors:
        - ref:
            name: "github"
          params:
            - name: "secretRef"
              value:
                secretName: github-webhook-secret
                secretKey: secretToken
            - name: "eventTypes"
              value: ["push", "pull_request"]
      bindings:
        - ref: github-push-binding
      template:
        ref: github-push-template
```

### 2. EventListener with Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tekton-triggers-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - tekton-webhook.example.com
      secretName: tekton-webhook-tls
  rules:
    - host: tekton-webhook.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: el-secure-github-listener
                port:
                  number: 8080
```

## GitLab Integration

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: gitlab-push-binding
spec:
  params:
    - name: gitrepositoryurl
      value: $(body.project.git_http_url)
    - name: gitrevision
      value: $(body.after)
    - name: gitrepositoryname
      value: $(body.project.name)
    - name: gitbranch
      value: $(body.ref)

---
apiVersion: triggers.tekton.dev/v1beta1
kind: Trigger
metadata:
  name: gitlab-push-trigger
spec:
  interceptors:
    - ref:
        name: "gitlab"
      params:
        - name: "secretRef"
          value:
            secretName: gitlab-secret
            secretKey: secretToken
        - name: "eventTypes"
          value: ["Push Hook"]
    - ref:
        name: "cel"
      params:
        - name: "filter"
          value: "body.ref.startsWith('refs/heads/')"
  bindings:
    - ref: gitlab-push-binding
  template:
    ref: github-push-template  # Reuse the same template
```

## Jenkins Integration with Triggers

### 1. Trigger Jenkins Job from Tekton

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: trigger-jenkins-from-webhook
spec:
  params:
    - name: jenkins-url
    - name: job-name
    - name: webhook-data
  steps:
    - name: process-and-trigger
      image: curlimages/curl:latest
      script: |
        #!/bin/sh
        echo "Processing webhook data and triggering Jenkins job"
        
        # Extract parameters from webhook data
        REPO_NAME=$(echo '$(params.webhook-data)' | jq -r '.repository.name')
        BRANCH=$(echo '$(params.webhook-data)' | jq -r '.ref' | sed 's/refs\/heads\///')
        COMMIT_SHA=$(echo '$(params.webhook-data)' | jq -r '.head_commit.id')
        
        # Trigger Jenkins job with parameters
        curl -X POST "$(params.jenkins-url)/job/$(params.job-name)/buildWithParameters" \
          --data-urlencode "REPO_NAME=$REPO_NAME" \
          --data-urlencode "BRANCH=$BRANCH" \
          --data-urlencode "COMMIT_SHA=$COMMIT_SHA"
```

## Monitoring and Troubleshooting

### 1. EventListener Logs

```bash
# Get EventListener pod logs
kubectl logs -l app.kubernetes.io/managed-by=EventListener -n tekton-pipelines

# Follow logs in real-time
kubectl logs -f deployment/el-github-listener -n tekton-pipelines
```

### 2. Debug Webhook Payloads

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: Trigger
metadata:
  name: debug-trigger
spec:
  interceptors:
    - ref:
        name: "cel"
      params:
        - name: "filter"
          value: "true"  # Accept all events
        - name: "overlays"
          value:
            - key: debug_info
              expression: |
                {
                  'headers': header,
                  'body_keys': body.map(k, k).map(k, string(k)),
                  'webhook_type': header['X-GitHub-Event'][0]
                }
  bindings:
    - ref: debug-binding
  template:
    ref: debug-template
```

## Best Practices

1. **Use appropriate interceptors**: Filter events early to avoid unnecessary pipeline runs
2. **Secure webhooks**: Always use secret tokens for webhook validation
3. **Resource limits**: Set appropriate resource limits for EventListeners
4. **Monitoring**: Monitor EventListener health and webhook delivery
5. **Error handling**: Implement proper error handling in TriggerTemplates
6. **Testing**: Test triggers with sample payloads before production use

## Resources

- [Tekton Triggers Documentation](https://tekton.dev/docs/triggers/)
- [GitHub Webhook Events](https://docs.github.com/en/developers/webhooks-and-events/webhooks/webhook-events-and-payloads)
- [GitLab Webhook Events](https://docs.gitlab.com/ee/user/project/integrations/webhooks.html#events)