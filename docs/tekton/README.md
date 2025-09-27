# Tekton Lernen (Learning Tekton with Jenkins)

This guide helps you learn Tekton, a Kubernetes-native CI/CD framework, and how to integrate it with Jenkins deployed via this Helm chart.

## Was ist Tekton? (What is Tekton?)

Tekton is an open-source framework for creating CI/CD systems on Kubernetes. It provides building blocks to create flexible, cloud-native CI/CD pipelines that run as Kubernetes resources.

### Key Concepts

- **Tasks**: Reusable, loosely coupled units of work that are executed as part of a Pipeline
- **Pipelines**: Collections of Tasks arranged in a specific execution order
- **PipelineRuns**: Instances of a Pipeline execution
- **TaskRuns**: Instances of a Task execution

## Getting Started with Tekton

🚀 **[Start Here: Hands-On Tutorial](getting-started.md)** - Complete step-by-step tutorial for beginners

### Prerequisites

Before you begin, ensure you have:
- A Kubernetes cluster (minikube, kind, k3d, or cloud provider)
- kubectl configured to access your cluster
- Jenkins deployed using this Helm chart
- Tekton Pipelines installed on your cluster

### Installing Tekton Pipelines

```bash
# Install Tekton Pipelines
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# Verify installation
kubectl get pods --namespace tekton-pipelines
```

### Installing Tekton Dashboard (Optional)

```bash
# Install Tekton Dashboard
kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml

# Access dashboard
kubectl --namespace tekton-pipelines port-forward svc/tekton-dashboard 9097:9097
```

## Tekton and Jenkins Integration Patterns

### Pattern 1: Jenkins Triggers Tekton Pipelines

Use Jenkins to orchestrate complex workflows that include Tekton pipeline executions.

### Pattern 2: Tekton Pipelines Call Jenkins Jobs

Use Tekton pipelines to trigger specific Jenkins jobs as part of a larger CI/CD workflow.

### Pattern 3: Parallel Execution

Run Jenkins jobs and Tekton pipelines in parallel for different aspects of your build/deployment process.

## Examples

- [Basic Tekton Pipeline](examples/basic-pipeline.yaml) - Simple pipeline example
- [Jenkins Integration](examples/jenkins-integration.yaml) - How to integrate with Jenkins
- [Multi-stage Pipeline](examples/multi-stage-pipeline.yaml) - Complex pipeline with multiple stages
- [Secret Management](examples/secret-management.yaml) - Handling secrets in Tekton pipelines

## Best Practices

1. **Use Workspaces**: Share data between tasks using Tekton workspaces
2. **Parameterize Pipelines**: Make your pipelines reusable with parameters
3. **Handle Secrets Securely**: Use Kubernetes secrets for sensitive data
4. **Monitor Pipeline Execution**: Use the Tekton dashboard or kubectl to monitor runs
5. **Version Control**: Store your Tekton YAML files in version control

## Troubleshooting

### Common Issues

1. **Pod Security Policies**: Ensure your cluster allows the required security contexts
2. **Resource Limits**: Set appropriate resource requests and limits
3. **Image Pull Secrets**: Configure image pull secrets for private registries
4. **Workspace Permissions**: Ensure proper file permissions in shared workspaces

### Debugging Commands

```bash
# List all pipelines
kubectl get pipelines

# List pipeline runs
kubectl get pipelineruns

# Describe a pipeline run
kubectl describe pipelinerun <pipeline-run-name>

# Get logs from a specific task
kubectl logs <pod-name> -c step-<step-name>
```

## Advanced Topics

- [Custom Tasks](advanced/custom-tasks.md)
- [Pipeline Triggers](advanced/triggers.md)
- [GitOps with Tekton](advanced/gitops.md)

## Resources

- [Official Tekton Documentation](https://tekton.dev/docs/)
- [Tekton Hub](https://hub.tekton.dev/) - Reusable tasks and pipelines
- [Tekton Community](https://github.com/tektoncd/community)
- [Jenkins Kubernetes Plugin](https://plugins.jenkins.io/kubernetes/)