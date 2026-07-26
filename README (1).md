# wordsmith-cicd-gitops
Automated CI/CD pipeline with GitOps-based deployment (ArgoCD) on Azure Kubernetes Service, built on Docker Samples' Wordsmith microservices app. Wordsmith is a microservices-based demo application consisting of three containers: a Java API service, a Go web application, and a PostgreSQL database

---
## 📖 Project Overview

This project demonstrates how to automate the build, containerization, and deployment of the **Wordsmith** sample application using modern DevOps practices on Microsoft Azure.

The application is containerized with Docker, built through Azure DevOps Pipelines, pushed to Azure Container Registry (ACR), and automatically deployed to Azure Kubernetes Service (AKS) using **ArgoCD**.

Instead of deploying directly from the CI pipeline, this project follows the **GitOps** approach where Kubernetes manifests stored in Git become the single source of truth for the cluster.

---

# 🏗️ Solution Architecture

```text
                    Developer
                        │
                        ▼
                 Azure Repos (Source Code)
                        │
                        ▼
              Azure DevOps CI Pipeline
                        │
        ┌───────────────┴────────────────┐
        │                                │
        ▼                                ▼
 Build Docker Images              Run Pipeline Tasks
        │
        ▼
Push Images to Azure Container Registry
        │
        ▼
Update Kubernetes Manifest Repository
        │
        ▼
Azure Repos (Manifest Repository)
        │
        ▼
      ArgoCD
        │
        ▼
Azure Kubernetes Service (AKS)
        │
        ▼
 Wordsmith Application
```

---

# 🛠️ Technologies Used

| Category | Technology |
|----------|------------|
| Source Control | Azure Repos |
| CI/CD | Azure DevOps Pipelines |
| Containerization | Docker |
| Container Registry | Azure Container Registry (ACR) |
| Container Orchestration | Azure Kubernetes Service (AKS) |
| GitOps | ArgoCD |
| Scripting | Bash |
| Operating System | Ubuntu Build Agent |
| Infrastructure | Kubernetes |

---

# 🎯 Project Objectives

- Automate Docker image builds
- Push images to Azure Container Registry
- Store Kubernetes manifests in Git
- Automatically update deployment manifests
- Commit updated manifests back to Git
- Automatically synchronize Kubernetes using ArgoCD
- Demonstrate a GitOps deployment workflow

---

# ☁️ Azure Resources Created

- Resource Group
- Azure Container Registry (ACR)
- Azure Kubernetes Service (AKS)
- Azure DevOps Project
- Azure Repositories
- Azure Pipelines
- Azure Service Connections

---

# 📂 Project Structure

```text
wordsmith/
│
├── api/
├── web/
├── db/
├── scripts/
│     └── updateK8sManifests.sh
│
├── k8s-specifications/
│     ├── api.yaml
│     ├── web.yaml
│     ├── db.yaml
│     └── ingress.yaml
│
├── azure-pipelines.yml
│
└── README.md
```

---

# ⚙️ CI/CD Workflow

### 1️⃣ Developer Pushes Code

```bash
git add .
git commit -m "Feature update"
git push origin main
```

---

### 2️⃣ Azure DevOps Pipeline Starts

The pipeline is automatically triggered after every push to the **main** branch.

Pipeline stages include:

- Checkout Source Code
- Build Docker Images
- Push Images to ACR
- Update Kubernetes Manifest
- Commit Changes
- Push Manifest Repository

---

### 3️⃣ Build Docker Images

Example:

```bash
docker build -t wordsmith.azurecr.io/web:25 .
```

---

### 4️⃣ Push Images to Azure Container Registry

```bash
docker push wordsmith.azurecr.io/web:25
```

---

### 5️⃣ Update Kubernetes Deployment Manifest

The pipeline executes a Bash script to update the image tag inside the Kubernetes deployment manifest.

**Before**

```yaml
image: wordsmith.azurecr.io/web:24
```

**After**

```yaml
image: wordsmith.azurecr.io/web:25
```

---

### 6️⃣ Commit Updated Manifest

The pipeline commits the updated manifest back to the Git repository.

Example commit message:

```text
Update Kubernetes manifest
```

---

### 7️⃣ ArgoCD Detects Git Changes

ArgoCD continuously watches the manifest repository.

When a new commit is detected, ArgoCD:

- Pulls the latest manifests
- Compares the desired state with the cluster
- Automatically synchronizes the cluster

No manual deployment commands are required.

---

### 8️⃣ Deploy to Azure Kubernetes Service

ArgoCD performs a rolling update, ensuring zero-downtime deployment of the latest container image.

---

### 9️⃣ Verify Deployment

```bash
kubectl get pods

kubectl get deployments

kubectl get svc

kubectl get nodes
```

---

---

## Part 1 — Continuous Integration (CI)

### Step 1: Create an Azure DevOps Project and Import the Repo

1. Create a new (private) Azure DevOps project
2. Go to **Repos → Import a repository**, choose **Git**, and paste the GitHub source URL — this imports the full history into Azure Repos
3. Azure Repos sometimes defaults to the wrong branch (alphabetical). Go to **Branches**, find `main`, and **Set as default branch**


<img width="959" height="506" alt="image" src="https://github.com/user-attachments/assets/03ed0b02-864f-476c-ba24-ba301c15d787" />

### Step 2: Create a Resource Group and ACR

```bash
# Via Azure Portal:
# Create a resource group, e.g. "wordsmith"
# Then create a Container Registry inside it, e.g. "wordsmithregistry", Standard tier
```

<img width="1919" height="576" alt="image" src="https://github.com/user-attachments/assets/00c05777-2f48-4c35-bf8c-12bb752eb1aa" />


### Step 3: Create One CI Pipeline Per Microservice

For each service (`web`, 'api'), create a separate pipeline using Azure DevOps' **"Build and push an image to Azure Container Registry"** template, then customize:

**Key concepts in every pipeline:**
- **Trigger** — path-based, so only changes under that service's folder trigger its pipeline
- **Pool** — set to Microsoft hosted agent
- **Stages** — `Build` and `Push` (a `Test` stage can be added later if the project has unit tests)

**Example — `azure-pipelines-web.yml`:**
```yaml
# Docker
# Build and push an image to Azure Container Registry
# https://docs.microsoft.com/azure/devops/pipelines/languages/docker

trigger:
 paths:
   include:
     - web/**

resources:
- repo: self

variables:
  # Container registry service connection established during pipeline creation
  dockerRegistryServiceConnection: 'wordsmith_acr'
  imageRepository: 'web'
  containerRegistry: 'wordsmith.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/web/Dockerfile'
  tag: '$(Build.BuildId)'

  # Agent VM image name
  vmImageName: 'ubuntu-latest'

stages:
- stage: Build
  displayName: Build and push stage
  jobs:
  - job: Build
    displayName: Build
    pool:
      vmImage: $(vmImageName)
    steps:
    - task: Docker@2
      displayName: Build and push an image to container registry
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: $(dockerfilePath)
        containerRegistry: $(dockerRegistryServiceConnection)
        tags: |
          $(tag)
```

Repeat the same structure for `api`, changing the `paths.include`, `imageRepository`, and `dockerfilePath` accordingly.

### Step 4: Verify Path-Based Triggers

Make a change only inside `web/` and commit — only the `web service` pipeline should trigger. Make a change inside `api/` — only that pipeline should trigger. This confirms independent CI per microservice.

---

## Part 2 — Continuous Delivery (CD) with GitOps

### Step 7: Create the AKS Cluster

1. Azure Portal → **Create a resource → Kubernetes Service**
2. Same resource group as before
3. Cluster name: e.g. `wordsmith-cluster`
4. Preset configuration: **Dev/Test**
5. Region: any with available quota
6. Node pool:
   - Scale method: **Autoscale**
   - Min nodes: `1`, Max nodes: `2`
7. **Review + Create → Create** (takes ~10–15 minutes)

<img width="959" height="523" alt="Screenshot 2026-07-25 170030" src="https://github.com/user-attachments/assets/4076c39f-ebc7-48b3-82c5-f9a8f6dd4950" />

### Step 8: Connect to the Cluster

```bash
az aks get-credentials --name <cluster-name> --resource-group <resource-group> --overwrite-existing
kubectl get pods
```

<img width="955" height="156" alt="Screenshot 2026-07-25 170315" src="https://github.com/user-attachments/assets/e2fb4676-25a8-4b35-a965-ab0247ad27f6" />

### Step 9: Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
```

<img width="959" height="326" alt="Screenshot 2026-07-25 170539" src="https://github.com/user-attachments/assets/ce215960-a227-42ee-989f-5ff7d49e04d2" />


<img width="852" height="175" alt="Screenshot 2026-07-25 170704" src="https://github.com/user-attachments/assets/2f3a0fb7-2006-4825-a86b-e73c5e66947b" />

### Step 10: Access the ArgoCD UI

**Get the admin password:**
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
echo <encoded-password> | base64 -d
```

<img width="959" height="209" alt="Screenshot 2026-07-25 171008" src="https://github.com/user-attachments/assets/7d0d50a8-0b00-4085-a1de-8f77a468c5d3" />

**Expose ArgoCD via NodePort:**
```bash
kubectl edit svc argocd-server -n argocd
```
Change `type: ClusterIP` → `type: NodePort`, then:
```bash
kubectl get svc -n argocd
kubectl get nodes -o wide
```

<img width="959" height="269" alt="Screenshot 2026-07-25 171216" src="https://github.com/user-attachments/assets/e69ae670-dff8-4cfa-b4eb-2c4c793a7e75" />

<img width="826" height="508" alt="Screenshot 2026-07-25 171132" src="https://github.com/user-attachments/assets/06f53b4c-5bf5-4895-970d-bbef82f54a06" />

<img width="959" height="222" alt="Screenshot 2026-07-25 171315" src="https://github.com/user-attachments/assets/c294c693-5db0-48b4-b479-ef298a94c3de" />

<img width="959" height="200" alt="Screenshot 2026-07-25 171746" src="https://github.com/user-attachments/assets/b65e3e88-1525-4e43-ae19-489d9789c665" />


**Open the port in Azure:**
Go to the cluster's **VMSS → Node pool → Instances → Networking → Add inbound port rule** (source: Any, destination port: the NodePort you noted)


<img width="959" height="538" alt="Screenshot 2026-07-25 172024" src="https://github.com/user-attachments/assets/a54f1930-ddc9-41d3-8f42-5ddc21a8d67e" />

**Log in:**
`http://<node-ip>:<nodeport>` → username `admin`, password from above.


<img width="957" height="543" alt="Screenshot 2026-07-25 172242" src="https://github.com/user-attachments/assets/b531a80f-15c2-4d8c-a2ae-267c2e4041b9" />


### Step 11: Connect ArgoCD to the Git Repository

1. Azure DevOps → **User Settings → Personal Access Tokens → New Token** (read access is enough)
2. ArgoCD → **Settings → Repositories → Connect Repo via HTTPS**
   - Repository URL: your Azure Repo clone URL
   - Username: your org name
   - Password: the PAT
3. Click **Connect** → should show "Successful"

<img width="959" height="536" alt="Screenshot 2026-07-25 172803" src="https://github.com/user-attachments/assets/e3d1643f-2aa7-445c-ab58-bde00f836f50" />

<img width="959" height="323" alt="Screenshot 2026-07-25 172825" src="https://github.com/user-attachments/assets/1e662ae6-5ea6-4143-a830-27d809726a9e" />


### Step 12: Create the ArgoCD Application

ArgoCD → **Create New Application**
- Name: `wordsmith-app` (create one per service, or one covering the whole `k8s-manifests` folder)
- Project: `default`
- Sync Policy: **Automatic**
- Repository URL: auto-filled
- Path: `k8s-manifests` 
- Cluster: your cluster
- Namespace: `default`

Click **Create** — ArgoCD immediately syncs whatever is in that path.


<img width="959" height="539" alt="Screenshot 2026-07-25 180529" src="https://github.com/user-attachments/assets/97e7417d-399b-4339-9cb0-c3b3eaa1f3a7" />

<img width="827" height="515" alt="Screenshot 2026-07-25 180814" src="https://github.com/user-attachments/assets/59d47b70-ac17-42f2-bdef-38ae99cf7250" />


**Optional — faster sync detection (default is 3 minutes):**
```bash
kubectl edit cm argocd-cm -n argocd
```
Add:
```yaml
timeout.reconciliation: 10s
```

<img width="926" height="320" alt="Screenshot 2026-07-25 192056" src="https://github.com/user-attachments/assets/70809439-6412-4dd2-8867-b774bae83b1a" />

<img width="603" height="68" alt="Screenshot 2026-07-25 192158" src="https://github.com/user-attachments/assets/5a4e2186-d71c-4211-8143-b9cd33178ab8" />


### Step 13: Write the Image-Tag Update Script

`scripts/updateK8sManifest.sh`:
```bash
#!/bin/bash

set -x

# Set the repository URL
REPO_URL="https://${PAT_TOKEN}@dev.azure.com/developerwajhi/wordsmith/_git/wordsmith"

# Clone the git repository into the /tmp directory
git clone "$REPO_URL" /tmp/temp_repo

# Navigate into the cloned repository directory
cd /tmp/temp_repo

# Make changes to the Kubernetes manifest file(s)
# For example, let's say you want to change the image tag in a deployment.yaml file
sed -i "s|image:.*|image: wordsmithregistry.azurecr.io/$2:$3|g" k8s-manifests/$1.yaml

git config user.name "Azure Devops"
git config user.email "developerwajhi@gmail.com"

# Add the modified files
git add .

# Commit the changes
git commit -m "Update Kubernetes manifest"

# Push the changes back to the repository
git push

# Cleanup: remove the temporary directory
rm -rf /tmp/temp_repo
```
> Security note: for production, don't hardcode the PAT in the script — pass it as a pipeline variable/environment variable instead.

<img width="959" height="539" alt="Screenshot 2026-07-25 175114" src="https://github.com/user-attachments/assets/e1b79daa-c263-4503-8b6d-65bf9d765954" />

> $2 $3 and $1 args will be passed through pipeline which are image repository, tag and deployment yaml file name respectively.
 

### Step 14: Add the Update Stage to Each Pipeline

Append to each service's pipeline YAML:
```yaml
- stage: update
  displayName: update manifest
  jobs:
  - job: update
    displayName: update manifest
    steps:
     - task: ShellScript@2
       inputs:
         scriptPath: 'scripts/updateK8sManifests.sh'
         args: 'web $(imageRepository) $(tag)'

       env:
         PAT_TOKEN: $(PAT_TOKEN)
```
(Change `web` to `api`  in their respective pipelines.)

### Step 15: Configure Private Registry Access (imagePullSecrets)

**Get ACR credentials:** Azure Portal → Container Registry → **Access keys** → enable Admin user.

**Create the Kubernetes secret** (correct namespace matters — this was a real gotcha):
```bash
kubectl create secret docker-registry acr-secret \
  --docker-server=<your-acr-name>.azurecr.io \
  --docker-username=<acr-username> \
  --docker-password=<acr-password> \
  --namespace=default
```

<img width="959" height="452" alt="Screenshot 2026-07-25 190423" src="https://github.com/user-attachments/assets/0c58c899-773c-41f8-b9eb-7a17847beaae" />

<img width="958" height="113" alt="Screenshot 2026-07-25 190753" src="https://github.com/user-attachments/assets/b6fde9f0-f392-430e-8f9b-53c637e5e12c" />


**Reference it in each deployment manifest:**
```yaml
imagePullSecrets:
  - name: acr-secret
```

<img width="959" height="550" alt="Screenshot 2026-07-25 190900" src="https://github.com/user-attachments/assets/d36d7e10-e36a-4fab-9762-53128b40f298" />

### Step 16: Verify End-to-End

1. Make a code change and commit
2. Pipeline runs: **Build & Push → Update**
3. New image tag appears in ACR
4. Deployment YAML in Git is updated automatically
5. ArgoCD detects the change and syncs → status shows "Synced" / "Healthy"
6. Confirm on the cluster:
```bash
kubectl get pods
kubectl get deploy <service>-deployment -o yaml
```
<img width="954" height="514" alt="image" src="https://github.com/user-attachments/assets/695467c4-04f7-47d7-9891-a07afe770ef6" />

<img width="959" height="510" alt="image" src="https://github.com/user-attachments/assets/02ded026-f1fe-4ec3-ac03-c8161d46efaf" />


<img width="959" height="539" alt="image" src="https://github.com/user-attachments/assets/30071157-da6b-4b69-ae79-9f1329c46243" />

<img width="959" height="526" alt="image" src="https://github.com/user-attachments/assets/4ff98869-c837-4b99-b526-6fd5d94253c4" />


<img width="768" height="155" alt="image" src="https://github.com/user-attachments/assets/502a04c6-c8da-440f-8dbb-0683ee118c8a" />


7. To view the app in a browser, expose the service and open its NodePort (same process as Step 10):
```bash
kubectl get svc
kubectl get nodes -o wide
```
<img width="1555" height="214" alt="image" src="https://github.com/user-attachments/assets/a63dbe81-3063-4428-9ed6-4a774e76eb5a" />

<img width="959" height="109" alt="image" src="https://github.com/user-attachments/assets/b1adae9b-b916-467d-98f3-5ce0f30b152b" />


<img width="1919" height="1026" alt="Screenshot 2026-07-25 192553" src="https://github.com/user-attachments/assets/49ef7d7d-7ba0-4d06-b1d6-a7d4e36e8917" />

<img width="959" height="512" alt="image" src="https://github.com/user-attachments/assets/0c6523a6-5686-4f90-a718-bc4f6c1100dd" />

---

## ⚠️ Common Errors & Fixes

### **Azure CLI not recognized (`az` command not found)**
Install the Azure CLI and restart the terminal before running Azure commands.

---

### **Azure CLI login failed due to Multi-Factor Authentication (MFA)**
Complete the MFA verification process and ensure you are logging into the correct Azure tenant and subscription.

---

### **No Azure subscriptions found**
Verify that your Azure account has an active subscription and switch to the correct subscription if multiple subscriptions are available.

```bash
az account list -o table
az account set --subscription "<subscription-name>"
```

---

### **AKS deployment failed due to vCPU quota limits**
The selected VM size exceeded the available subscription quota. Choose a smaller supported VM size or request a quota increase.

---

### **Selected AKS node size not supported**
Some VM sizes are unavailable in certain regions. Select a supported node size such as `Standard_D2s_v3`.

---

### **Unable to connect to the AKS cluster**
Retrieve the Kubernetes credentials for the cluster before using `kubectl`.

```bash
az aks get-credentials \
  --resource-group <resource-group> \
  --name <aks-cluster-name>
```

---

### **ArgoCD admin password could not be decoded on Windows**
Use PowerShell to decode the Base64-encoded Kubernetes secret instead of the Linux `base64` command.

---

### **`$'\r': command not found` in Bash script**
The shell script was saved with Windows (CRLF) line endings. Convert the file to Linux (LF) format before committing it.

---

### **Git commit failed with `Author identity unknown`**
Configure the Git username and email before committing changes from the pipeline.

```bash
git config user.name "Azure DevOps Pipeline"
git config user.email "azuredevops@pipeline.local"
```

---

### **Repository clone failed in the pipeline**
Verify that the Azure Repos URL and authentication method (PAT or System.AccessToken) are configured correctly.

---

### **Pipeline variables not available inside the Bash script**
Pass variables through the pipeline `args` or `env` section and reference them correctly inside the script.

---

### **Kubernetes manifest was not updated**
Verify the manifest file path, file name, and the image update command used in the Bash script.

---

### **Azure Container Registry authentication failed**
Verify the Azure DevOps service connection and ensure the pipeline has permission to push images to ACR.

---

### **`ImagePullBackOff` while deploying to AKS**
Ensure the Kubernetes image pull secret is created correctly and the deployment references the correct secret.

```bash
kubectl get secrets
kubectl describe pod <pod-name>
```

---

### **ArgoCD failed to access the Git repository**
Configure repository authentication using a Personal Access Token (PAT) with the required repository permissions.

---

### **ArgoCD did not deploy the latest image**
Verify that the pipeline successfully updated the Kubernetes manifest with the latest image tag and that Auto Sync is enabled.

---

### **Unable to delete Azure DevOps Service Connection**
Remove all pipeline or resource dependencies that reference the service connection before attempting to delete it.

---

### **Failed to remove Azure permissions for ACR**
This occurred because the referenced Azure Container Registry no longer existed. Verify the ACR name and recreate or reattach it if necessary.

---

### **Docker image push failed**
Ensure Docker is logged into Azure Container Registry and the pipeline is using the correct registry service connection.

```bash
az acr login --name <acr-name>
docker push <acr-name>.azurecr.io/<image>:<tag>
```

---

### **Bash script failed to update Kubernetes manifests**
Verify the repository path, manifest location, image name, and image tag variables before committing the changes back to Azure Repos.

```bash
git status
cat k8s-manifests/web.yaml
```

# 🔮 Future Improvements

- Replace the custom Bash manifest update script with **ArgoCD Image Updater**
- Package Kubernetes manifests using **Helm**
- Use **Azure Key Vault** with **Azure Workload Identity**
- Add container vulnerability scanning
- Integrate Azure Monitor and Application Insights
- Implement automated rollback strategies

---

# 📚 Learning Outcomes

Through this project I gained practical experience in:

- Designing and implementing CI/CD pipelines
- Deploying containerized applications to AKS
- Implementing GitOps with ArgoCD
- Managing Docker images in Azure Container Registry
- Automating Kubernetes deployments
- Troubleshooting Azure DevOps, Kubernetes, Git, and Bash scripting issues
- Applying Infrastructure as Code and GitOps best practices

---

## 👩‍💻 Author

**Salwa Paracha**

Cloud & DevOps Engineer

- Azure DevOps
- Kubernetes
- Docker
- GitOps
- Microsoft Azure

---

⭐ **If you found this project interesting, feel free to fork the repository or connect with me on LinkedIn!**
