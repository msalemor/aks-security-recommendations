# AKS Security Recommendations

## 1.0 Goals:  

- High-level breakdown of all aspects of security that should be considered when running an AKS cluster. 
- Leave behind for customers rather than an emailed dump of links. 
> ***Not covered here:*** specific application security for apps deployed to the cluster 
 

## 2.0 Cluster level concerns 

- These concerns should be considered before setting up the cluster (i.e. running: az aks create) 
- Couple of general articles 
  - https://docs.microsoft.com/en-us/azure/aks/concepts-security 
  - https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-security 
 

### 2.1 - Master  

- The cluster's master node(s) are managed by AKS. Locking down access to them is critical. 
- Use a private cluster stronger posture if possible. Does the customer need the API server exposed externally? 
  - https://docs.microsoft.com/en-us/azure/aks/private-clusters 

> **See "Egress Security" and properly setting the --outbound-type flag for private clusters 

- If a publicly exposed K8S API server, use authorized IP ranges to lock down what internal and external IP's can access the api. 

> VERY IMPORTANT! <br><br>https://docs.microsoft.com/en-us/azure/aks/api-server-authorized-ip-ranges 

### 2.2 - Integrate the cluster with AAD for user auth  

- Link AAD user auth with Kubernetes built-in RBAC 
> This is critical: Subsequent security features like connecting to ACR, running pods with a specific Managed Identity, and properly externalizing secrets. <br><br>https://docs.microsoft.com/en-us/azure/aks/managed-aad 
- Use a User Defined Mamaged Identity rather than System Assigned when creating the cluster so identity can be reused on cluster recreate 
  --assign-identity $IDENTITY 

### 2.3 Node Security 

- OS patches are applied nightly by AKS. Rebooting the VM doesn't happen automatically 
- Linux kernel updates require a reboot 
  > Kured daemonset is a solution for safe rebooting (cordoned / drained) 
  > Suggested.  Rather than leveraging Kured, recommended approach is to use VMSS Node Image Upgrade, now supported by AKS. This must be scripted by the customer as it's not automatically enforced as of yet (this on the roadmap).  This is considered an advantageous approach when compared to Kured, as Node Image Upgrade will bring a new OS image and will apply the image to all nodes in a cordoned/drained fashion. 

 Upgrade Azure Kubernetes Service (AKS) node images - Azure Kubernetes Service | Microsoft Docs 

 

Security hardening the AKS agent node host OS. AKS provides a security optimized host OS by default. There is no option to select an alternate operating system. No action need to be taken, but more info here.  

Security hardening in AKS virtual machine hosts - Azure Kubernetes Service | Microsoft Docs 

 

Upgrade Kubernetes version 

Regularly update K8S version (need to stay within n-2 of officially supported kuberntes versions) 

https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-security#regularly-update-to-the-latest-version-of-kubernetes 

 

Compute isolation (optional) 

Leverage isolated VM types if there's a concern about neighbors running on the same physical hardware. 

https://docs.microsoft.com/en-us/azure/virtual-machines/isolation 

 

Note: for product clusters, separating system and user node pools is a best practice for resiliency and scale reasons, but not necessarily security. 

 

Integrate the cluster with Azure Container Registry 

Don't use docker login/password. Connect to ACR through the cluster's AAD integration. See Image Management for more security tweaks around the ACR. 

https://docs.microsoft.com/en-us/azure/aks/cluster-container-registry-integration 

 

Deploy ACR with a private endpoint and connect to it from AKS privately. Peering may be needed if a private cluster. See below.  

 

Enable SSH (optional) 

Do this only if SSH'ing to agent nodes is deemed useful for troubleshooting. Safely store these keys. 

https://docs.microsoft.com/en-us/azure/aks/aks-ssh 

 

Enable Monitoring 

When creating the cluster: --enable-addons monitoring 

Integrate the cluster with Azure Monitoring. This could involve a discussion of Prometheus metrics scaping and the pros/cons of AKS Container Insights vs Prometheus/Grafana.  

Overview of Azure Monitor for containers - Azure Monitor | Microsoft Docs 

Configure Azure Monitor for containers Prometheus Integration - Azure Monitor | Microsoft Docs 

 

Enable Azure Defender for Kubernetes 

Provides real-time threat protection of the cluster and generates alerts for threats and malicious activity. 

Enabled through Azure Security Center for a fee. This is not enabled by default. 

Azure Defender for Kubernetes - the benefits and features | Microsoft Docs 

 

Azure Defender for Kubernetes provides protections at the cluster level. If you also deploy the Log Analytics agent of Azure Defender for Servers, you'll get the threat protection for your nodes that's provided with that plan.  

 

We recommend deploying both for the most protection. 

https://docs.microsoft.com/en-us/azure/security-center/defender-for-kubernetes-introduction#can-i-still-get-aks-protections-without-the-log-analytics-agent 

 

Azure Defender for Servers requires the Log Analytics Agent to be deployed. It is ok to run this side-by-side with the Azure Monitor agent (enabling container insights).  

https://docs.microsoft.com/en-us/azure/security-center/defender-for-kubernetes-introduction#if-my-cluster-is-already-running-an-azure-monitor-for-containers-agent-do-i-need-the-log-analytics-agent-too 

 

Separate apps across node pools (optional) 

Proper node pool design is critical for cluster reliability. However, it can also affect security if want to ensure certain application stay physically isolated from each other and will not run the same VM nodes. In this case node pool can provide this level of physical isolation. 

Use multiple node pools in Azure Kubernetes Service (AKS) - Azure Kubernetes Service | Microsoft Docs 

 

## 3.0 Network concerns 

- Things to consider from a networking perspective as it relates to the cluster. Proper LZ design is outside of the scope of this document. 
  - https://docs.microsoft.com/en-us/azure/aks/concepts-network

### 3.1 - Network Security 

- Do not add NSG's to the subnets hosting the cluster (not supported for AKS).   
- Use Kubernetes network policy to control flow (east-west) within the cluster between applications/pods.  

### 3.2 - Network Policy 

- Allow/Deny networking rules to pods inside of the cluster 
  - https://docs.microsoft.com/en-us/azure/aks/use-network-policies 
> Note: This is critical for applying who and what can access application pods. Network Policy enables east/west network traffic between pods inside the cluster. Example would be putting appA into namespace A and appB in namespace B, and leveraging Network Policy to not let pods in these application call each other. Enables isolation. 
- Azure and Calico are two different flavors of network policy. 
  - https://docs.microsoft.com/en-us/azure/aks/use-network-policies#differences-between-azure-and-calico-policies-and-their-capabilities 

### 3.3 - Avoid using public IPs to expose load balanced pods 

- Use an ingress controller to reverse proxy and aggregate Kubernetes services. Perhaps route external traffic through a WAF (AppGw, Front Door, etc) before hitting the ingress controller. Let the ingress controller route to services and then to pods. 
  - https://docs.microsoft.com/en-us/azure/aks/ingress-internal-ip 
> General rule: avoid using public IPs anywhere inside of the AKS cluster if not explicitly required. 

### 3.4 - Egress Security 

- Route and limit egress traffic leaving the cluster through a firewall. 
  - https://docs.microsoft.com/en-us/azure/aks/limit-egress-traffic 
- Avoid a public IP for egress with a private cluster + Standard LB 
> IMPORTANT for private clusters 

- Private AKS clusters means only that the API server gets a private IP. By default, AKS still uses a public IP for egress traffic from nodes/pods to outside world, even in private AKS instances. This is because the Standard LB requires the ability to egress, and will create a public IP to do so unless you manage egress traffic flow. You can set the outbound type to use a UDR to Azure Firewall or another NVA to disable the creation of the public IP when leveraging the Standard Load Balancer. 
  - https://docs.microsoft.com/en-us/azure/aks/egress-outboundtype
  - https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest#az_aks_create-examples 

## 4.0 Developer/Manifest/Configuration concerns 

- Things to consider from a developer and configuration perspective 

### 4.1 Limit root access 

  - Do not run containers as root 
  - Define a security context for privilege and access control settings so that the pod/container doesn't get root access. 
    - https://docs.microsoft.com/en-us/azure/aks/developer-best-practices-pod-security 
    - https://kubernetes.io/docs/tasks/configure-pod-container/security-context/ 
  - Best solution -- use this built-in Azure Policy to enforce this cluster wide 
    - https://portal.azure.com/#blade/Microsoft_Azure_Policy/PolicyDetailBlade/definitionId/%2Fproviders%2FMicrosoft.Authorization%2FpolicyDefinitions%2F95edb821-ddaf-4404-9732-666045e056b4 

### 4.2 Externalize Secrets 

- Externalize application secrets to KeyVault and connect to K8S secrets/pods/envVars. Not GA as of yet.  
  - https://github.com/Azure/secrets-store-csi-driver-provider-azure 
  - https://docs.microsoft.com/en-us/azure/key-vault/general/key-vault-integrate-kubernetes 

### 4.3 - Leverage Pod Identity 

- Instead of authorizing access to resources at the cluster level, do so for individual pods with Managed Identity. 
  - Pod Identity should be leveraged when externalizing secrets to KeyVault. 
    - https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-identity#use-pod-identities 
    
> NOTE: Pod Identity not supported with Windows node pools as of yet. Can instead access KV secrets from the cluster MI through CSI driver. 

### 4.4 - Use Namespaces to logically isolate deployments 

- Leverage namespaces along with Network Policies for application isolation. Namespace on their own provide no isolation and are not a security layer.  
  - https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/ 

## 5.0 Governance concerns / Azure Policy 

- Set of governance and compliance rules to enforce organizational standards and to assess compliance at-scale. 
  - https://docs.microsoft.com/en-us/azure/governance/policy/concepts/policy-for-kubernetes 
  - https://docs.microsoft.com/en-us/azure/aks/policy-reference 
- Azure Policy extends Gatekeeper v3, an admission controller webhook for Open Policy Agent (OPA), to apply at-scale enforcements and safeguards on your clusters in a centralized, consistent manner. Azure Policy makes it possible to manage and report on the compliance state of your Kubernetes clusters from one place. 
  - Example of a policy. Enforce that authorized IP ranges are defined on a publicly exposed AKS cluster. 
  - Examples of how an organization wants the platform to respond to a non-complaint resource include: 
    - Deny the resource change 
    - Log the change to the resource 
    - Alter the resource before the change 
    - Alter the resource after the change 
    - Deploy related compliant resources 
- Azure Advisor bubbles up recommendations both from Azure Policy and overall platform. 

## 6.0 Image Management concerns 

- Protect and secure aspects of container images and the AKS cluster 

### 6.1 Scan images 

- Scan container images to ensure they are free of vulnerabilities 
  - https://docs.microsoft.com/en-us/azure/security-center/defender-for-container-registries-introduction 

### 6.2 Lock down allowed container images 

- Ensure the AKS cluster restricts container images to trusted registries. 
  - https://docs.microsoft.com/en-us/azure/aks/policy-reference#microsoftcontainerservice 

### 6.3 Lock down ACR with RBAC 

- Only possible with AAD enabled clusters. Easiest to enable this when creating the cluster.  
  - https://docs.microsoft.com/en-us/azure/container-registry/container-registry-roles 


### 6.4 Network lockdown of ACR with Private Link 

- Expose ACR through private link removing external access to the registry. Note, vnet peering will be required if AKS deployed in private cluster.  
  - https://docs.microsoft.com/en-us/azure/container-registry/container-registry-private-link 
