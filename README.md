# Azure-cluster
This document is intended to provide operational guidance for Azure Kubernetes Service management solution
This document describes the steps to upgrade Kubernetes (k8s) version of single node pool AKS clusters. 
NOTE: the document assumes that you: have cluster Admin privilege, installed and have basic understanding of az, kubectl, and jq

Pre-checks

•	Verify cluster is single node pool 
•	az aks show -g $cluster_rg -n $cluster_name | jq '{nodePoolCount:.agentPoolProfiles | length, nodeCount:.agentPoolProfiles[0].count, maxPodPerNode:.agentPoolProfiles[0].maxPods, vnetInfo:.agentPoolProfiles[0].vnetSubnetId}' 
	the query should return nodePoolCount is 1 
•	Make note of the maxPod and vnetInfo values 
•	Verify cluster subnet has enough IPs available for another node with max pods. 
•	In Azure Web Portal, go to the Vnet > Subnet that was queried in the previous step 
•	Make sure that available IP >= maxPodPerNode. 
•	Get PODs statuses and backup to file 
•	kubectl get pods -A -o wide --sort-by=.metadata.namespace > $cluster_name_status.bak 
 
Upgrade
	Getting the upgradable versions 
•	az aks get-upgrades -n $cluster_name -g $cluster_rg 

	Performing the upgrade 
•	az aks upgrade -g $cluster_rg -n $cluster_name --kubernetes-version $k8s_new_version 
•	change the value for $k8s_new_version as needed 

NOTE: It is recommended that we do not skip minor version when upgrading.

•	Example: if the current version is 1.16.0 where 1 is the major version and 16 is the minor version 
•	Goal: Upgrade to 1.18.10 . then the correct method of upgrade is this 
•	Upgrade to 1.17.x 
•	Then Upgrade again to 1.18.10 

Post-checks
•	Verify that node pool K8s version is the same as control plane K8s version 
•	az aks show -n $cluster_name -g $cluster_rg | jq -r '{orchestratorK8sVersion:.agentPoolProfiles[0].orchestratorVersion,nodePoolK8sVersion:.kubernetesVersion}' 
•	orchestratorK8sVersion should be the same as nodePoolK8sVersion 
•	Verify that all pods statuses are similiar to statuses in your backup 
•	kubectl get pods -A -o wide --sort-by=.metadata.namespace > $cluster_name_new_status.bak 
•	Compare the new file vs. old file 

Known Issues
1. Not enough IPs 
•	After confirming that the # of IPs available are not enough to support a new node. Use az CLI to do the following 
•	Get AKS cluster nodePool information 
•	az aks show -n $cluster_name -g $cluster_rg --query agentPoolProfiles 
•	Scale down from the current node count minus 1 
•	az aks scale -n $cluster_name -g $cluster_rg --node-count 2 --nodepool-name $node_pool_name 
•	Perform the upgrade again 

2. K8s docker images are not pulling from mcr.microsoft.com 
•	During troubleshooting, it is determined that k8s docker images are being pulled from the wrong server (for example: us.gcr.io) 
•	connect to a node that has the required image 
•	push required image to ACR 
. connect to the node that cannot pull the required image 
•	pull required image from ACR 

3. K8s docker images cannot be pulled from mcr.microsoft.com 
•	This is most likely an issue with Zscaler 
•	connect to a node that cannot pull required image 
•	attempt to pull required image 
•	verify error is server authentication issue 

