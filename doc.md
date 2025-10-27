# Investigation: Migration of Self-Hosted Azure DevOps Agents from Virtual Machines to Azure Container Apps

---

## **Background / Current Setup**

Currently, each Azure subscription (~40 in total) hosts a **dedicated virtual machine** that runs a **self-hosted Azure DevOps agent**.

This setup was chosen because:
- Each subscription has **isolated networking** (different VNets and NSGs).  
- All deployments are **private** (no public endpoints), requiring local network presence in each subscription.

### **Current Challenges**
- Manual VM lifecycle management (patching, scaling, image maintenance).  
- Fixed compute cost — VMs are always running.  
- Difficult to standardize agent updates and configurations.  
- Limited elasticity — agents cannot scale dynamically based on workloads.

To address these challenges, we are exploring **Azure Container Apps (ACA)** as a **serverless, container-based platform** for running Azure DevOps self-hosted agents.  
This aims to improve flexibility, scalability, and cost-efficiency.

---

## **Objective**

Investigate and document the **requirements, design considerations, and migration plan** for hosting Azure DevOps self-hosted agents in **Azure Container Apps**, while maintaining:
- Full private network connectivity to each subscription’s resources.
- Security and compliance parity with the current VM-based setup.
- Improved scalability and reduced operational overhead.

---

## **Scope of Investigation**

### 1. **Architecture Design**
- Evaluate ACA environments per subscription with **VNet integration**.
- Identify **centralized vs per-subscription** ACA deployment models.
- Define subnet planning, private DNS, and private endpoint design.

### 2. **Image and Runtime**
- Create a **custom Docker image** for the Azure DevOps agent using Microsoft’s official container guidance.  
  👉 [Run Azure Pipelines agents in Docker containers](https://learn.microsoft.com/azure/devops/pipelines/agents/docker?view=azure-devops)
- Include build tools such as Terraform, Azure CLI, PowerShell, etc.
- Implement startup scripts for **automatic agent registration** using a **PAT** securely stored in **Azure Key Vault**.

### 3. **Security and Identity**
- Use **Managed Identity** for Azure resource access.
- Store and inject secrets from **Azure Key Vault**.
- Restrict communication to **private endpoints**.
- Ensure the container image pulls only from **private ACR**.

### 4. **Scalability and Resilience**
- Configure ACA to **scale to zero** and scale out based on load or queue metrics.
- Test **multi-replica deployments** for parallel pipelines.
- Validate restart and recovery behavior.

### 5. **Observability**
- Forward logs and metrics to **Log Analytics Workspace**.
- Implement alerts for agent or container failures.

### 6. **Infrastructure as Code**
- Use **Terraform** to provision ACA environments, container apps, networking, and identities consistently across subscriptions.

### 7. **Cost and Performance Analysis**
- Compare ACA compute cost vs VM cost.
- Evaluate cold start latency and build cache strategies.

---

## **Technical Requirements**

| Component | Description |
|------------|-------------|
| **Azure Container Apps Environment** | Must be integrated into a private VNet (dedicated subnet, recommended `/23`). |
| **Azure Container Registry (ACR)** | Host custom agent images. Use a Private Endpoint for secure access. |
| **Azure Key Vault** | Store PAT tokens, registry credentials, and other secrets. |
| **Managed Identity** | Used by Container App to authenticate with Azure resources. |
| **Log Analytics Workspace** | Centralized logging and monitoring for ACA diagnostics and agent logs. |
| **Terraform** | Infrastructure as Code for ACA and related resources. |
| **Azure DevOps PAT** | Used for agent registration; stored in Key Vault and rotated regularly. |

---

## **Official References**

- [Azure Container Apps networking and VNet integration](https://learn.microsoft.com/azure/container-apps/vnet-custom-internal?tabs=azure-cli)  
- [Run Azure Pipelines agents in Docker containers](https://learn.microsoft.com/azure/devops/pipelines/agents/docker?view=azure-devops)  
- [Use Managed Identities in Azure Container Apps](https://learn.microsoft.com/azure/container-apps/managed-identity?tabs=portal)  
- [Secure secrets in Container Apps using Azure Key Vault](https://learn.microsoft.com/azure/container-apps/secure-secrets-managed-identity?tabs=portal)  

---

## **Key Considerations**

- Each ACA environment requires a **dedicated subnet** — subnet size cannot be changed later.  
- ACA containers are **ephemeral** — build caches must use shared storage (Azure Files / Blob).  
- Agent registration must be **idempotent** to handle container restarts.  
- **Private DNS** zones must correctly resolve all private endpoints (ACR, Key Vault, etc.).  
- Managed Identities must be assigned correct roles at the **subscription scope**.

---

## **Risks and Mitigation**

| Risk | Mitigation |
|------|-------------|
| Cold start delays may affect first job runtime | Keep one container replica warm or pre-pull images |
| Private DNS misconfiguration could block startup | Validate DNS resolution via ACA diagnostics |
| PAT expiration halts agent registration | Automate PAT rotation via pipeline or script |
| Cross-subscription access failures | Assign Managed Identity roles at subscription level |
| Subnet IP exhaustion | Reserve a larger subnet (`/23` or larger) |

---

## **Deliverables**

1. Architecture diagram for ACA-based DevOps agents.  
2. Terraform templates for ACA environment, container app, and supporting resources.  
3. Custom Dockerfile for Azure DevOps agent container.  
4. PoC deployment and test report.  
5. Final recommendation report (cost, scalability, feasibility).

---

## **Next Steps (PoC Plan)**

### **Objective**
Perform a small-scale **Proof of Concept (PoC)** to validate containerized self-hosted agents on Azure Container Apps.

### **Scope**
- Deploy one **ACA environment** in a test subscription (private subnet).  
- Deploy a **single container app** running one Azure DevOps agent.  
- Connect to private Azure resources (e.g., Storage Account, Key Vault).  
- Run a sample pipeline that:
  - Pulls code from Azure Repos.  
  - Executes a Terraform deployment.  
  - Logs outputs to Log Analytics.  

### **Success Criteria**
- Agent registers and executes a pipeline successfully.  
- All communication remains private (no public egress).  
- Managed Identity successfully authenticates to Azure resources.  
- Logs and metrics visible in Log Analytics.  
- Cost and performance metrics collected for analysis.

---

## **Expected Outcome**

- Validated design for running Azure DevOps agents on Azure Container Apps.  
- Terraform-based repeatable deployment model.  
- Documented pros/cons and migration readiness.  
- Reduced VM management overhead and cost.  
- Improved elasticity, observability, and maintainability.

---

## **Future Scope**
- Automate deployment across all subscriptions.  
- Integrate agent scaling with pipeline queue metrics.  
- Evaluate centralised agent pools vs per-subscription models.

---
