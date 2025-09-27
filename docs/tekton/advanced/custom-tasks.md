# Custom Tasks in Tekton

Custom Tasks allow you to extend Tekton's functionality by creating reusable components that can be shared across pipelines and teams.

## Understanding Custom Tasks

Custom Tasks are Kubernetes Custom Resources that implement the Task interface but run outside the standard Tekton task execution model. They're useful for:

- Integrating with external systems
- Implementing complex logic that doesn't fit the standard step model
- Creating reusable components for specific use cases

## Creating a Custom Task

### 1. Basic Custom Task Structure

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: custom-notification-task
  labels:
    app.kubernetes.io/version: "0.1"
  annotations:
    tekton.dev/pipelines.minVersion: "0.17.0"
    tekton.dev/categories: Messaging
    tekton.dev/tags: notification, alert
    tekton.dev/displayName: "Custom Notification"
spec:
  description: >-
    Send custom notifications to multiple channels
  params:
    - name: message
      description: Message to send
      type: string
    - name: channels
      description: Notification channels (slack, email, teams)
      type: array
      default: ["slack"]
    - name: severity
      description: Message severity level
      type: string
      default: "info"
  results:
    - name: notification-id
      description: Unique ID of the sent notification
  steps:
    - name: send-notifications
      image: alpine:latest
      script: |
        #!/bin/sh
        set -e
        
        MESSAGE="$(params.message)"
        SEVERITY="$(params.severity)"
        NOTIFICATION_ID=$(date +%s%N | cut -b1-13)
        
        echo "Sending notification with ID: $NOTIFICATION_ID"
        echo "Message: $MESSAGE"
        echo "Severity: $SEVERITY"
        
        # Process each channel
        for channel in $(params.channels[*]); do
          echo "Processing channel: $channel"
          
          case $channel in
            "slack")
              echo "Sending to Slack..."
              # Implementation for Slack
              ;;
            "email")
              echo "Sending email..."
              # Implementation for email
              ;;
            "teams")
              echo "Sending to Microsoft Teams..."
              # Implementation for Teams
              ;;
            *)
              echo "Unknown channel: $channel"
              ;;
          esac
        done
        
        echo -n "$NOTIFICATION_ID" > $(results.notification-id.path)
```

### 2. Advanced Custom Task with Workspaces

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: advanced-analysis-task
spec:
  description: Advanced code analysis with multiple tools
  params:
    - name: analysis-tools
      description: Analysis tools to run
      type: array
      default: ["sonar", "checkmarx", "veracode"]
    - name: project-key
      description: Project identifier
      type: string
    - name: analysis-config
      description: Analysis configuration
      type: string
      default: "default"
  workspaces:
    - name: source
      description: Source code to analyze
    - name: reports
      description: Analysis reports output
      optional: true
    - name: config
      description: Analysis configuration files
      optional: true
  results:
    - name: analysis-summary
      description: Summary of analysis results
    - name: quality-gate
      description: Quality gate status (passed/failed)
  stepTemplate:
    env:
      - name: PROJECT_KEY
        value: $(params.project-key)
      - name: ANALYSIS_CONFIG
        value: $(params.analysis-config)
  steps:
    - name: prepare-analysis
      image: alpine:latest
      script: |
        #!/bin/sh
        echo "Preparing analysis for project: $PROJECT_KEY"
        echo "Configuration: $ANALYSIS_CONFIG"
        
        # Create reports directory if workspace is provided
        if [ -d "$(workspaces.reports.path)" ]; then
          mkdir -p $(workspaces.reports.path)/analysis-results
        fi
        
        # Load custom configuration if provided
        if [ -d "$(workspaces.config.path)" ]; then
          echo "Loading custom configuration..."
          ls -la $(workspaces.config.path)
        fi
    
    - name: run-analysis
      image: sonarsource/sonar-scanner-cli:latest
      workingDir: $(workspaces.source.path)
      script: |
        #!/bin/bash
        set -e
        
        TOOLS=($(params.analysis-tools[*]))
        ANALYSIS_RESULTS=""
        QUALITY_GATE="passed"
        
        for tool in "${TOOLS[@]}"; do
          echo "Running analysis with: $tool"
          
          case $tool in
            "sonar")
              echo "Running SonarQube analysis..."
              # SonarQube implementation
              sonar-scanner \
                -Dsonar.projectKey=$PROJECT_KEY \
                -Dsonar.sources=. || QUALITY_GATE="failed"
              ;;
            "checkmarx")
              echo "Running Checkmarx analysis..."
              # Checkmarx implementation
              ;;
            "veracode")
              echo "Running Veracode analysis..."
              # Veracode implementation
              ;;
          esac
          
          ANALYSIS_RESULTS="$ANALYSIS_RESULTS$tool:completed "
        done
        
        echo "Analysis completed: $ANALYSIS_RESULTS"
        echo -n "$ANALYSIS_RESULTS" > $(results.analysis-summary.path)
        echo -n "$QUALITY_GATE" > $(results.quality-gate.path)
```

## Best Practices for Custom Tasks

### 1. Parameter Validation

```yaml
steps:
  - name: validate-params
    image: alpine:latest
    script: |
      #!/bin/sh
      set -e
      
      # Validate required parameters
      if [ -z "$(params.project-key)" ]; then
        echo "Error: project-key parameter is required"
        exit 1
      fi
      
      # Validate parameter format
      if ! echo "$(params.project-key)" | grep -qE '^[a-zA-Z0-9_-]+$'; then
        echo "Error: project-key must contain only alphanumeric characters, hyphens, and underscores"
        exit 1
      fi
      
      echo "Parameter validation passed"
```

### 2. Error Handling

```yaml
steps:
  - name: robust-execution
    image: alpine:latest
    script: |
      #!/bin/bash
      set -e
      
      # Function for error handling
      handle_error() {
        local exit_code=$?
        echo "Error occurred in line $1. Exit code: $exit_code"
        
        # Cleanup operations
        cleanup_resources
        
        exit $exit_code
      }
      
      # Set error trap
      trap 'handle_error $LINENO' ERR
      
      cleanup_resources() {
        echo "Performing cleanup..."
        # Cleanup implementation
      }
      
      # Main execution logic
      echo "Starting robust execution..."
      # Your task logic here
```

### 3. Retry Logic

```yaml
steps:
  - name: retry-execution
    image: alpine:latest
    script: |
      #!/bin/bash
      
      retry_command() {
        local max_attempts=$1
        local delay=$2
        shift 2
        local command="$@"
        
        for ((i=1; i<=max_attempts; i++)); do
          echo "Attempt $i of $max_attempts: $command"
          
          if eval "$command"; then
            echo "Command succeeded on attempt $i"
            return 0
          else
            echo "Command failed on attempt $i"
            if [ $i -lt $max_attempts ]; then
              echo "Waiting $delay seconds before retry..."
              sleep $delay
            fi
          fi
        done
        
        echo "Command failed after $max_attempts attempts"
        return 1
      }
      
      # Example usage
      retry_command 3 5 "curl -f https://api.example.com/health"
```

## Publishing Custom Tasks

### 1. Creating a Task Bundle

```yaml
# Create a bundle with multiple related tasks
apiVersion: tekton.dev/v1beta1
kind: Bundle
metadata:
  name: custom-tasks-bundle
spec:
  tasks:
    - name: custom-notification-task
      version: "1.0.0"
    - name: advanced-analysis-task
      version: "1.0.0"
```

### 2. Task Documentation

Include comprehensive documentation with your custom tasks:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: documented-task
  annotations:
    tekton.dev/pipelines.minVersion: "0.17.0"
    tekton.dev/categories: Build Tools
    tekton.dev/tags: build, compile, maven
    tekton.dev/displayName: "Maven Build Task"
    tekton.dev/platforms: "linux/amd64"
spec:
  description: |
    This task compiles Java applications using Maven.
    
    ## Usage
    
    ```yaml
    - name: build
      taskRef:
        name: documented-task
      params:
        - name: goals
          value: ["clean", "compile", "test"]
    ```
    
    ## Parameters
    
    - `goals`: Maven goals to execute
    - `maven-image`: Maven Docker image to use
    
    ## Results
    
    - `build-status`: Build execution status
```

## Testing Custom Tasks

### 1. Unit Testing

```bash
# Create test pipeline for custom task
cat > test-custom-task.yaml << EOF
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: test-custom-task-pipeline
spec:
  tasks:
    - name: test-notification
      taskRef:
        name: custom-notification-task
      params:
        - name: message
          value: "Test message"
        - name: channels
          value: ["slack", "email"]
        - name: severity
          value: "info"
EOF

# Run test
kubectl apply -f test-custom-task.yaml
kubectl create -f - << EOF
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: test-custom-task-run
spec:
  pipelineRef:
    name: test-custom-task-pipeline
EOF
```

### 2. Integration Testing

```yaml
# Integration test pipeline
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: integration-test-pipeline
spec:
  workspaces:
    - name: test-data
  tasks:
    - name: setup-test-data
      taskRef:
        name: git-clone-task
      params:
        - name: url
          value: "https://github.com/tektoncd/pipeline.git"
      workspaces:
        - name: output
          workspace: test-data
    
    - name: test-custom-analysis
      runAfter: [setup-test-data]
      taskRef:
        name: advanced-analysis-task
      params:
        - name: project-key
          value: "test-project"
        - name: analysis-tools
          value: ["sonar"]
      workspaces:
        - name: source
          workspace: test-data
```

## Contributing Custom Tasks

When contributing custom tasks to the community:

1. **Follow naming conventions**: Use descriptive, kebab-case names
2. **Include comprehensive tests**: Unit and integration tests
3. **Document thoroughly**: Clear descriptions, examples, and parameter documentation
4. **Version properly**: Follow semantic versioning
5. **Consider backwards compatibility**: Don't break existing usage patterns

## Resources

- [Tekton Hub](https://hub.tekton.dev/) - Browse community tasks
- [Task Authoring Guidelines](https://tekton.dev/docs/pipelines/tasks/#task-authoring-recommendations)
- [Community Contributions](https://github.com/tektoncd/catalog)