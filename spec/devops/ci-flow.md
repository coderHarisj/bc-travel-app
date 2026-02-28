## Required Output Sections

### 1. Pipeline Stage Definitions
Define each stage with trigger, runner, and success criteria:

1. source code push
2. build
3. test
4. security scan
5. artifact publish
6. whatever step fails pipeline need to be failed and send email out

### 1. Automated Build Process
- Dockerfile / build script location and structure
- Build artifact naming convention: {{APP_NAME}}-{{VERSION}}-{{SHORT_SHA}}
- Based on the env branch the build process should start automatically
- dev, qa, uat, prod - direct pr merge & build
- for uat & prod automatic tag creation

### 2. Automated Testing Integration
- follow the rules in the @devops-context.md for testing

### 3. Artifact Management
- follow the rules in the @devops-context.md

### 4. Approval Gates
- dev, qa - no approval
- uat, prod - manual approval only for deployment

## Output Format
Produce a spec document AND mermaid code  
