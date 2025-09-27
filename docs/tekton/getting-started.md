# Getting Started with Tekton - Hands-On Tutorial

This tutorial will guide you through setting up Tekton and creating your first pipeline with Jenkins integration.

## Prerequisites

- Kubernetes cluster (minikube, kind, k3d, or cloud provider)
- kubectl configured to access your cluster
- Jenkins deployed using this Helm chart
- Basic knowledge of Kubernetes and Docker

## Step 1: Install Tekton

```bash
# Install Tekton Pipelines
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# Install Tekton Dashboard (Optional but recommended)
kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml

# Verify installation
kubectl get pods --namespace tekton-pipelines
```

Wait for all pods to be in `Running` state.

## Step 2: Access Tekton Dashboard

```bash
# Port forward to access the dashboard
kubectl --namespace tekton-pipelines port-forward svc/tekton-dashboard 9097:9097

# Open your browser to http://localhost:9097
```

## Step 3: Create Your First Task

Create a simple "Hello World" task:

```bash
cat > hello-task.yaml << EOF
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello-world
spec:
  params:
    - name: greeting
      description: The greeting to display
      default: "Hello World from Tekton!"
  steps:
    - name: echo
      image: ubuntu
      command: ["/bin/bash"]
      args: ["-c", "echo '\$(params.greeting)'"]
EOF

# Apply the task
kubectl apply -f hello-task.yaml
```

## Step 4: Run Your First Task

```bash
cat > hello-taskrun.yaml << EOF
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: hello-world-run
spec:
  taskRef:
    name: hello-world
  params:
    - name: greeting
      value: "Hello from my first Tekton task!"
EOF

# Apply and run the task
kubectl apply -f hello-taskrun.yaml

# Check the status
kubectl get taskrun hello-world-run

# View the logs
kubectl logs -f taskrun/hello-world-run
```

## Step 5: Create a Git Clone Task

```bash
cat > git-clone-task.yaml << EOF
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: git-clone
spec:
  params:
    - name: url
      description: Repository URL to clone
    - name: revision
      description: Git revision to checkout
      default: "main"
  workspaces:
    - name: output
      description: The git repo will be cloned here
  steps:
    - name: clone
      image: gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/git-init:v0.40.2
      command: ["/ko-app/git-init"]
      args:
        - "-url=\$(params.url)"
        - "-revision=\$(params.revision)"
        - "-path=\$(workspaces.output.path)"
EOF

kubectl apply -f git-clone-task.yaml
```

## Step 6: Create Your First Pipeline

```bash
cat > first-pipeline.yaml << EOF
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: my-first-pipeline
spec:
  params:
    - name: repo-url
      description: The git repository URL
    - name: greeting
      description: Greeting message
      default: "Hello from Pipeline!"
  workspaces:
    - name: shared-data
  tasks:
    - name: greet
      taskRef:
        name: hello-world
      params:
        - name: greeting
          value: \$(params.greeting)
    
    - name: fetch-source
      taskRef:
        name: git-clone
      params:
        - name: url
          value: \$(params.repo-url)
      workspaces:
        - name: output
          workspace: shared-data
    
    - name: list-files
      runAfter: [fetch-source]
      taskSpec:
        workspaces:
          - name: source
        steps:
          - name: list
            image: ubuntu
            workingDir: \$(workspaces.source.path)
            command: ["/bin/bash"]
            args: ["-c", "echo 'Files in repository:' && ls -la"]
      workspaces:
        - name: source
          workspace: shared-data
EOF

kubectl apply -f first-pipeline.yaml
```

## Step 7: Run Your Pipeline

```bash
cat > first-pipelinerun.yaml << EOF
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: my-first-pipeline-run
spec:
  pipelineRef:
    name: my-first-pipeline
  params:
    - name: repo-url
      value: "https://github.com/spring-projects/spring-petclinic.git"
    - name: greeting
      value: "Hello from my learning journey with Tekton!"
  workspaces:
    - name: shared-data
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
EOF

kubectl apply -f first-pipelinerun.yaml
```

## Step 8: Monitor Your Pipeline

```bash
# Check pipeline run status
kubectl get pipelinerun my-first-pipeline-run

# Get detailed information
kubectl describe pipelinerun my-first-pipeline-run

# View logs from the dashboard or CLI
kubectl logs -f pipelinerun/my-first-pipeline-run

# List all tasks in the pipeline run
kubectl get taskrun -l tekton.dev/pipelineRun=my-first-pipeline-run
```

## Step 9: Jenkins Integration Setup

### Install Jenkins with Tekton Support

Add these values to your Jenkins Helm chart:

```yaml
# jenkins-values.yaml
controller:
  plugins:
    - kubernetes:3937.vd7b_82db_e347b_
    - tekton:0.1.0
  
  JCasC:
    configScripts:
      tekton-config: |
        jenkins:
          clouds:
            - kubernetes:
                name: "kubernetes"
                serverUrl: "https://kubernetes.default"
                namespace: "default"
                tektonEnabled: true
                tektonNamespace: "tekton-pipelines"
```

### Create a Jenkins Job that Triggers Tekton

```groovy
// Jenkins Pipeline Script
pipeline {
    agent any
    
    parameters {
        string(name: 'REPO_URL', defaultValue: 'https://github.com/spring-projects/spring-petclinic.git', description: 'Repository URL')
        string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
    }
    
    stages {
        stage('Trigger Tekton Pipeline') {
            steps {
                script {
                    sh """
                        cat > jenkins-triggered-pipelinerun.yaml << EOF
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: jenkins-triggered-\${BUILD_NUMBER}
spec:
  pipelineRef:
    name: my-first-pipeline
  params:
    - name: repo-url
      value: "${params.REPO_URL}"
    - name: greeting
      value: "Hello from Jenkins build #\${BUILD_NUMBER}!"
  workspaces:
    - name: shared-data
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
EOF
                        
                        # Apply the PipelineRun
                        kubectl apply -f jenkins-triggered-pipelinerun.yaml
                        
                        # Wait for completion
                        kubectl wait --for=condition=Succeeded pipelinerun/jenkins-triggered-\${BUILD_NUMBER} --timeout=600s
                    """
                }
            }
        }
        
        stage('Get Tekton Results') {
            steps {
                script {
                    sh """
                        echo "Tekton Pipeline Results:"
                        kubectl describe pipelinerun jenkins-triggered-\${BUILD_NUMBER}
                    """
                }
            }
        }
    }
}
```

## Step 10: Create a Build and Deploy Pipeline

```bash
cat > build-deploy-pipeline.yaml << EOF
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: maven-build
spec:
  workspaces:
    - name: source
  steps:
    - name: build
      image: maven:3.8-openjdk-11
      workingDir: \$(workspaces.source.path)
      command: ["/bin/bash"]
      args: 
        - -c
        - |
          if [ -f "pom.xml" ]; then
            echo "Building Maven project..."
            mvn clean package -DskipTests
            echo "Build completed!"
          else
            echo "No pom.xml found, skipping Maven build"
          fi

---
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: build-deploy-pipeline
spec:
  params:
    - name: repo-url
      description: Repository URL
    - name: revision
      description: Git revision
      default: "main"
  workspaces:
    - name: shared-data
  tasks:
    - name: clone-repo
      taskRef:
        name: git-clone
      params:
        - name: url
          value: \$(params.repo-url)
        - name: revision
          value: \$(params.revision)
      workspaces:
        - name: output
          workspace: shared-data
    
    - name: build-app
      runAfter: [clone-repo]
      taskRef:
        name: maven-build
      workspaces:
        - name: source
          workspace: shared-data
    
    - name: run-tests
      runAfter: [build-app]
      taskSpec:
        workspaces:
          - name: source
        steps:
          - name: test
            image: maven:3.8-openjdk-11
            workingDir: \$(workspaces.source.path)
            command: ["/bin/bash"]
            args: 
              - -c
              - |
                if [ -f "pom.xml" ]; then
                  echo "Running tests..."
                  mvn test
                else
                  echo "No tests to run"
                fi
      workspaces:
        - name: source
          workspace: shared-data
EOF

kubectl apply -f build-deploy-pipeline.yaml
```

## Step 11: Test the Complete Pipeline

```bash
cat > complete-pipelinerun.yaml << EOF
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: complete-pipeline-run
spec:
  pipelineRef:
    name: build-deploy-pipeline
  params:
    - name: repo-url
      value: "https://github.com/spring-projects/spring-petclinic.git"
    - name: revision
      value: "main"
  workspaces:
    - name: shared-data
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 2Gi
EOF

kubectl apply -f complete-pipelinerun.yaml

# Monitor the pipeline
kubectl get pipelinerun complete-pipeline-run -w
```

## Step 12: Clean Up

```bash
# Delete pipeline runs
kubectl delete pipelinerun --all

# Delete tasks and pipelines
kubectl delete task hello-world git-clone maven-build
kubectl delete pipeline my-first-pipeline build-deploy-pipeline

# Clean up files
rm -f *.yaml
```

## Next Steps

Congratulations! You've successfully:
- ✅ Installed Tekton
- ✅ Created your first task and pipeline
- ✅ Integrated Tekton with Jenkins
- ✅ Built a complete CI/CD pipeline

### Continue Learning:

1. **[Explore Examples](examples/)** - Check out more complex pipeline examples
2. **[Advanced Topics](advanced/)** - Learn about custom tasks, triggers, and GitOps
3. **[Best Practices](README.md#best-practices)** - Follow recommended patterns
4. **[Troubleshooting](README.md#troubleshooting)** - Debug common issues

### Useful Commands:

```bash
# List all Tekton resources
kubectl get tasks,pipelines,pipelineruns,taskruns

# Get logs from a specific task
kubectl logs -l tekton.dev/task=<task-name>

# Debug a failed pipeline run
kubectl describe pipelinerun <pipeline-run-name>

# Access Tekton dashboard
kubectl port-forward -n tekton-pipelines svc/tekton-dashboard 9097:9097
```

## Troubleshooting Common Issues

### Pipeline Stuck in Pending State
```bash
# Check if there are sufficient resources
kubectl describe pipelinerun <name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Task Fails with Permission Issues
```bash
# Check service account permissions
kubectl get serviceaccount default -o yaml
kubectl describe clusterrolebinding <binding-name>
```

### Workspace Issues
```bash
# Check PVC status
kubectl get pvc -l tekton.dev/pipelineRun=<pipeline-run-name>
kubectl describe pvc <pvc-name>
```

Happy learning with Tekton! 🚀