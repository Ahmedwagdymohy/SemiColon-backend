
## 🐍 DevOps Setup for SemiColon Registration System - Backend

This section describes the DevOps processes used for the SemiColon Registration System, detailing Docker configurations, Kubernetes deployments, and testing environments.

### ⚙️  Docker Setup ⚙️

The backend project is containerized using Docker, with separate Dockerfiles and Docker Compose configurations for development, testing, and production environments.

#### 🐳 Docker Compose

1. **docker-compose.yml** (Production)
    - **Services**:
      - `mongo`: A MongoDB instance using the `bitnami/mongodb:5.0` image.
      - `backend`: The main backend application running in a Node.js container.
    - **Health Checks**: Configured for MongoDB using `mongosh` to ensure the service is running properly.
    - **Backend Features**:
      - Ports exposed: `3000`
      - Command: `npm run deploy-linux`
      - Depends on the MongoDB service to be healthy.
    - **Environment Variables**:
      - `DEV_DB_URL`, `PROD_DB_URL`, `TEST_DB_URL`: Database URLs for different environments.
      - `SESSION_SECRET` and `PROD_SESSION_SECRET`: Secrets used for session management.

2. **docker-compose-testing.yml** (Testing)
    - **Services**:
      - `mongo`: Similar to production but focused on testing.
      - `test`: Executes the unit and integration tests using Jest, leveraging the `semicolon-backend:v1-base` Docker image.
    - **Command**: `npm run test`
    - **Environment Variables**: Similar to the production setup but pointed to the test database.

#### 🐳 Dockerfiles

1. **Dockerfile**:
    - **Stages**:
      - **Build Stage**: Uses `node:22-alpine` as the base image, installs dependencies, and builds the application.
      - **Production Stage**: Copies over the build artifacts and installs only the production dependencies.
    - **Health Check**: Configured to verify the health of the app via an HTTP request to `localhost:$PORT`.
    - **Exposed Port**: `3000`
    - **Final Command**: Runs the app using `npm run deploy-linux`.

---

### ⚙️ Kubernetes Setup ⚙️ 

The Kubernetes configuration includes deployments, ConfigMaps, and Ingress rules for managing both the backend application and MongoDB.

1. **🤔 configmap.yml**:
    - Stores environment-specific variables, such as the database URLs and application port.
    - Example:
      ```yaml
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: semicolon-backend-config
      data:
        PORT: '3000'
        NODE_ENV: 'production'
        DEV_DB_URL: 'mongodb://semicolon-backend-db-svc:27017/semicolon-dev?directConnection=true'
        TEST_DB_URL: 'mongodb://semicolon-backend-db-svc:27017/semicolon-test?directConnection=true'
        PROD_DB_URL: 'mongodb://semicolon-backend-db-svc:27017/semicolon-prod?directConnection=true'
      ```

2. **🤔 app-deployment.yml** (Backend Application):
    - Defines the deployment for the backend app.
    - **Replicas**: 1 instance for now, can be scaled.
    - **Init Containers**: Used to run tests before the main application starts.
    - **Environment Variables**: Pulled from the `ConfigMap` and `Secret`.

3. **🤔 app-deployment.yml** (MongoDB):
    - Defines the deployment for MongoDB using the `bitnami/mongodb:5.0` image.
    - Configured with a primary replica and emptyDir volumes for persistence.

4. **🤔 app-ingress.yml**:
    - Configures an NGINX Ingress for exposing the backend API to the outside world.
    - **Annotations**: SSL redirect is disabled for this setup.
    - **Rules**: Maps the host `semicolon-backend.com` to the service.

---

## 🔧 CI/CD Pipeline 🔧


The Jenkins pipeline automates the testing, building, and provisioning of infrastructure for the SemiColon Registration System. The stages include running tests, **building** Docker images, provisioning infrastructure with **Terraform**, and deploying configurations using **Ansible**.



### Jenkinsfile Overview

1. **❗ Preparation**:
   - Clones the repository from GitHub (`main` branch) to the Jenkins workspace.

2. **❗ Test**:
   - Runs tests using Docker Compose with the `docker-compose-testing.yml` file.
   - Tears down any existing containers and brings up new ones to ensure a fresh test environment.
   
   ```groovy
   sh "docker compose -f docker-compose-testing.yml down --remove-orphans"
   sh "docker compose -f docker-compose-testing.yml up -d --build"
   ```

3. **❗ Build**:
   - Builds a Docker image for the backend.
   - Uses Jenkins credentials to log into Docker Hub and pushes the built image tagged with the build number.
   
   ```groovy
   def imageName = "hassanbahnasy/semi-colon:${BUILD_NUMBER}"
   sh "docker build . -t ${imageName}"
   sh 'echo $PASSWORD | docker login -u $USERNAME --password-stdin'
   sh "docker push ${imageName}"
   ```

4. **❗ Provision Infrastructure**:
   - Uses Terraform to provision infrastructure on Azure.
   - Jenkins credentials are used for Azure authentication.
   - SSH agent is used for secure remote execution of Terraform commands.
   
   ```groovy
   withEnv(["TF_VAR_client_id=${AZURE_CLIENT_ID}", "TF_VAR_client_secret=${AZURE_CLIENT_SECRET}", "TF_VAR_tenant_id=${AZURE_TENANT_ID}", "TF_VAR_subscription_id=${AZURE_SUBSCRIPTION_ID}"]) {
       sshagent(['bahnasy']) { 
           sh 'cd terraform && terraform init'
           sh 'cd terraform && terraform apply -auto-approve'
       }
   }
   ```

5. **❗ Get Public IP**:
   - After provisioning, the pipeline fetches the public IP address of the deployed infrastructure from the Terraform output.
   - This IP is stored as an environment variable for later stages.

   ```groovy
   def publicIP = sh(script: 'cd terraform && terraform output -json public_ip_address', returnStdout: true).trim()
   ```

6. **❗ Run Ansible Playbook**:
   - Executes an Ansible playbook to configure the deployed infrastructure, using the public IP obtained from the Terraform stage.
   - Jenkins securely provides SSH credentials for connecting to the remote machine.

   ```groovy
   withCredentials([sshUserPrivateKey(credentialsId: 'ansible', keyFileVariable: 'SSH_KEY')]) {
       sh "ansible-playbook -i ${env.PUBLIC_IP}, semi-colon.yml --extra-vars 'target_host=${env.PUBLIC_IP}' --user azureuser --private-key $SSH_KEY -e \"ansible_ssh_common_args='-o StrictHostKeyChecking=no'\""
   }
   ```

### 📢 Post-Build Actions

After the pipeline completes, notifications are sent to a Slack channel based on the success or failure of the job:

- **Success**: Sends a green notification with a success message.
- **Failure**: Sends a red notification with a failure message.

```groovy
post {
    success {
        slackSend(channel: "depi", color: '#00FF00', message: "Succeeded: Job '${env.JOB_NAME} ${env.BUILD_NUMBER}'")
    }
    failure {
        slackSend(channel: "depi", color: '#FF0000', message: "Failed: Job '${env.JOB_NAME} ${env.BUILD_NUMBER}'")
    }
}
```

