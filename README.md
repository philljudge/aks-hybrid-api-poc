# AKS-Hybrid & AKS Cloud api-deployment

Achieving Distributed High Availability for Critical National Infrastructure with AKS Hybrid

Azure Stack HCI (hyperconverged infrastructure) is a virtualisation host that runs on-premises on validated partner hardware. AKS Hybrid is an on-premise implementation of Azure Kubernetes Service (AKS) orchestrator which automates running containerised applications at scale and runs on Windows Server with Hyper-V, and the Stack HCI platform. Together they provide a solution for hosting highly available workloads on-premises.  Azure Arc is a cloud based control plane which can be used for managing on-premises AKS Hybrid instances. Flux is an open-sourced set of continuous delivery solutions for Kubernetes. AKS and AKS Hybrid natively support Flux through their GitOps capabilities.

The aim of this guide is to show the steps required to create the infrastructure that underpins a simulated supplier on-premises environment, and cloud environment.  See [here](https://github.com/ianlcurtis1/ha-aks-hybrid-poc) for a solution that uses this pattern.

Please note, this guide is work in progress and over the coming weeks and months I will be adding more detailed information & steps on how to setup the different parts of the deployment. 

To complete the deployment we created an environment that mirrored an on-premises server infrastructure. To do this you will need the following:

Prerequisites
1)	An azure tenant 
2)	A subscription in the tenant
3)	Relevant permissions on the subscription to build resources 
4)	Global Admin role on the Azure tenant


Deploy AKS Hybrid 
AKS Hybrid Host

To deploy the AKS hybrid environment, we followed the following evaluation guide. For the deployment we used a Standard ED16ds v4 (16 vCPUs, 128 Gib Memory) VM as our AKS hybrid host.

https://learn.microsoft.com/en-us/azure/aks/hybrid/aks-hci-evaluation-guide-1 


AKS and Arc

Once the AKS Hybrid host is deployed, the next step is to deploy AKS on to the server, create a Kubernetes target cluster, and Arc-enable the AKS cluster. To do this we followed the following guide:  

https://learn.microsoft.com/en-us/azure/aks/hybrid/aks-hci-evaluation-guide-2b 

At this point you should have a deployment that looks something like the diagram below:


![AKSHybrid ArcHLD](https://github.com/philljudge/aks-hybrid-api-poc/assets/131694192/0c0cea40-e625-4ad3-b326-89b0cb9720a5)



The following components should now be configured:	
1)	Installed a VM to act as the AKS host
2)	Deployed the AKS service on the host
3)	Arc enabled the AKS to cluster to allow remote management

Deploy NFS Server

The next step is deploy a Network File Share (NFS) server on your hyper v host. The NFS server acts as a persistent storage queue for the software running on the AKS cluster. It provides resilience should cloud connectivity be lost, or a pod fails.

Please note, we did the minimum to run the deployment and the settings we used should not be used in a production environment. For example, in production you would secure the file share using with Access controls or similar, use the exports file to specify which clients can mount the share, implement firewall rules or configure host based authentication.

For our server we used Ubuntu 22.04, the guide we followed can be found in the link below. 

https://linuxhint.com/install-and-configure-nfs-server-ubuntu-22-04/ 

Deploy the API

The AKS Kubernetes YAML deployment file references the ACR image, but to be able to pull the image the AKS Hybrid cluster we used Azure Container Registry to store the image API.  When you deploy the application to the Kubernetes cluster using a YAML file, the file references the azure container registry containing the application image. The AKS Hybrid cluster needs a secret, created using the command below.

kubectl create secret docker-registry my-registry-secret 

-- docker-username=DOCKER_USER 

-- docker-password=DOCKER_PASSWORD 

-- docker-email=DOCKER_EMAIL

Keep a record of the my-registry-secret (or whatever secret name you decide to use). This is needed later in the YAML deployment file.  The username and password for the file can be found in the Azure Container Registry. 

In our setup we decided to leverage the GitOps feature to deploy the YAML configurations, this feature is made available to the Hybrid AKS cluster by Arc enabling the AKS cluster which was covered earlier in the guide.  GitOps is a technique for implementing continuous deployment for the application.  Changes to the workload environment, such as an application update, happen via pull request to the Git repository, after which Flux, running in each cluster, automatically syncs the changes and applies them to the cluster.  To setup and configure Gitops for the AKS Hybrid use the below guide.

[Tutorial: Deploy applications using GitOps with Flux v2 - Azure Arc | Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/tutorial-use-gitops-flux2?tabs=azure-portal)

Deploy AKS

AKS in Azure

For the deployment we decided to have both an on-premises AKS hybrid cluster instance, and a cloud instance, each following the same architectural deployment stamp to give suppliers 2 deployment options.

The cloud AKS cluster could also acts as a failover in the event that an on-premises instance goes down. To deploy the AKS Cluster we followed the below guide.

https://learn.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster?tabs=azure-cli 

AKS integration with ACR

The AKS cluster needs to be integrated to the Azure Registry Container.  Once this integration is in place the AKS cluster is assigned the relevant permissions to access the ACR and the images can be deployed to the AKS cluster.  To attach the ACR to the AKS cluster we used the below guide. 

[Integrate Azure Container Registry with Azure Kubernetes Service (AKS) - Azure Kubernetes Service | Microsoft Learn](https://learn.microsoft.com/en-us/azure/aks/cluster-container-registry-integration?tabs=azure-cli)

Azure File Share

Just like the AKS Hybrid setup the AKS cloud cluster also requires persistent storage.  To achieve this in Azure we did not need to build any servers that need to be maintained.  In Azure we created an Azure File Share & this is where all the messages will be stored before the message processer transfers the files into Event Hub.  To create the file share we used the guide below.  The storage does need to be mounted to the AKS cluster for messages to be stored, this will be covered later when the guide is updated

https://learn.microsoft.com/en-us/azure/storage/files/storage-how-to-create-file-share?tabs=azure-cli

The following components should now be configured:	
1)	Installed a VM to act as the AKS host
2)	Deployed the AKS service on the host
3)	Arc enabled the AKS to cluster to allow remote management
4)	Local NFS Server for persistent Storage 
5)	An Azure Event hub to receive messages from supplier Environment
6)	Azure Container Registry to store the application

At this point you should have a deployment that looks similar to the diagram below:



![AKSHybrid Arc AKSCloud NFS-HLD](https://github.com/philljudge/aks-hybrid-api-poc/assets/131694192/a4b0a605-c429-4291-8716-2ebecc45384d)


Monitoring/Troubleshooting
Coming soon.

Security
Coming soon.

Now that all the steps are competed you have an infrastructure that can underpin an AKS Hybrid (Supplier environment) and AKS cloud environment.  In future articles I will be doing a breakdown of the full installation and going through each service step by step.  It will also cover the steps to install the application from the GitHub repository and how you can use Gitops to manage the state of the application running in the clusters. 
