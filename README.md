# Azure AKS Portworx
Instructions and examples to setup Portworx on Azure AKS with a production ready configuration.

## Pre-requisites

1. [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
2. [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

## Azure AKS Portworx Architecture

<todo: add architecture diagram>

## Steps to Install Portworx on Azure AKS

1. Create and configure Azure AKS infrastructure [see section below](#create-azure-aks-infrastructure)
2. Generate Portworx spec for your AKS cluster [see section below](#spec-generator)
3. Install Portworx on your cluster [see section below](#install-portworx-on-aks)

### Azure Infrastructure Recommendations

#### Azure Managed Disks vs Azure Files

<todo: research Azure Disk, single pod access but premium storage, vs Azure Files with multiple pod access but standard storage only>

<todo: research dynamic vs static storage and make recommendation>

https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv
https://docs.microsoft.com/en-us/azure/aks/azure-disks-dynamic-pv

#### Azure Managed Disk count

<todo: work with Portworx to determine the recommended number of Azure managed disks relative to VMs and datastores>

<todo: is the datastore pods to disks a many to one relationship? need to clarify with Portworx team>

## Create Azure AKS Infrastructure

Before you can install Portworx on your Azure AKS cluster, you must first create and configure all of the Azure AKS infrustructure necessary to run Portworx.

Steps to create and prepare Azure infrastructure for AKS and Portworx. [createAzureInfrastructure.sh](deployment/createAzureInfrastructure.sh) is a bash shell scripts that provides all of the Azure CLI and kubectl commands necessary to complete the steps below.

1. Create an Azure Resource Group for the AKS cluster
2. Create an Azure Active Directory Service Principal for authentication in the AKS cluster and its infrastructure dependencies
3. Create an Azure AKS cluster running K8s
4. Create Azure Managed Disks via K8s Persistent Volume Claims, which will be used to persist the data used via Portworx
5. Attach the Azure Managed Disks to the Azure VMs that are created in the AKS nodepool

Once all of this infrastructure has been created and prepped in Azure, you are now ready to install Portworx.

## Portworx Configuration

On K8s, Portworx uses K8s persistent volumes to store data that lives beyond the lifecycle of consuming pods. K8s uses the Azure Storage Class as a driver to take advantage of Azure Managed Disks as the storage medium. Azure Managed Disks provide high availability, high performance, availability and ensures that your data is save as it is a managed storage layer provided by Microsoft in the Azure Cloud.

### Spec Generator

You will need to generate a Portworx spec with the configuration specific to your Portworx installation and apply it to your AKS cluster using kubectl. The best way to do this is to use the [Portworx spec generator.](https://docs.portworx.com/portworx-install-with-kubernetes/cloud/azure/aks/2-deploy-px/#generate-the-specs) The following configuration options are recommended:

#### Built-in ETCD
This option uses the Portworx provided KVDB for the key/value database that Portworx needs to run. Using KVDB is the recommended option in Production by Portworx.

<todo: add screenshot>

#### Consume Unused storage option.
This option will enable Portworx to find the Azure Managed Disks that are attached to the AKS nodepool VMs and use them for Portworx volumes.

<todo: add screenshot>

#### Azure Kubernetes Service
Finally, make sure you select the AKS option on the Customize tab to let Portworx know what orchestration cloud provider it is running on.

<todo: add screenshot>

An example Portworx Azure AKS spec can be found in [deployment/portworx-azure-spec.yaml](deployment/portworx-azure-spec.yaml)

### KVDB

I wanted to call out the Portworx provided ETCD solution, KVDB explicitly. There are a lot of options for providing Portworx with the ETCD database that it needs and [you can find their documentation here.](https://docs.portworx.com/reference/knowledge-base/etcd/#requirements)

## Install Portworx on AKS

Now that we have everything setup and ready to go, installing Portworx is pretty easy. Just run the following kubectl command after you've generated your Portworx AKS spec:

```
kubectl apply -f deployment/portworx-azure-spec.yaml
```

### Check the health of Portworx on AKS

Once the `kubectl apply` operation has completed successfully you can run the following commands to make sure that Portworx has installed successfully. It can take Portworx a few minutes to get everything up and running, so give it 5-10 minutes.

To check the health of the services and pods, run:

```
kubectl get pods,svc -n kube-system
```

You should see a number of Portworx pods and services, hopefully all going into the Ready state.

To check the health of the Portworx application and the storage pool, run:

```
PX_POD=$(kubectl get pods -l name=portworx -n kube-system -o jsonpath='{.items[0].metadata.name}')
  kubectl exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl status
```

## Troubleshooting

If you run into issues with your Portworx installation, the following debugging and troubleshooting tips may help.

**Super Important** If you need to make changes to your Portworx spec, Azure infrastructure, etc. make sure to run the following command to uninstall Portworx. `kubectl delete -f deployment/portworx-azure-spec.yaml` **does not** clean up all Portworx configuration resources from the initial installation attempt and can cause your troublshooing changes not to take effect.

Make sure you run the following to uninstall Portworx if you make spec or infrastructure changes. [More information can be found here.](https://docs.portworx.com/portworx-install-with-kubernetes/operate-and-maintain-on-kubernetes/uninstall/uninstall/#delete-wipe-px-cluster-configuration)

```
curl -fsL https://install.portworx.com/px-wipe | bash
```

### Portworx

<todo: cleanup this section>

- kubecel logs -n kube-system <pod-name>
- https://docs.portworx.com/portworx-install-with-kubernetes/operate-and-maintain-on-kubernetes/troubleshooting/troubleshoot-and-get-support/#useful-commands
- kubectl describe pod/px-lighthouse-d7949d765-9fln9 -n kube-system
- Shell into running pod:
- kubectl exec -it <pod-name> -- /bin/bash
- k exec -it px-lighthouse-6b66ddfd98-lg46b -c config-init -n kube-system /bin/bash
- kubectl describe pod/etcd-docker-for-desktop -n kube-system
- For Portworx pods, you need to specify a container with -c [px-lighthouse config-sync stork-connector] or one of the init containers: [config-init]
- kubectl logs -n kube-system px-lighthouse-6b66ddfd98-lg46b -c px-lighthouse

### Azure Infrastructure Creation

<todo: cleanup this section>

- Make sure that the appropriate storage class has been deployed to the AKS cluster for your Azure Disk type
- kubectl get sc
- *If the storage class does not exist on the AKS cluster*, edit azure-storage-class.yaml and apply the Azure storage class to the AKS cluster
- kubectl apply -f deployment/azure-storage-class.yaml
- To ssh into a VM, set the SSH keys, create a jumbox k8s pod, exec into the pod and SSH from there
https://docs.microsoft.com/en-us/azure/aks/ssh
```
az vm user update \
  --resource-group MC_aaros-aks-portworx_aarosAksCluster_westus2 \
  --name aks-nodepool1-17636774-0 \
  --username azureuser \
  --ssh-key-value ~/.ssh/id_rsa.pub
  az vm list-ip-addresses --resource-group MC_aaros-aks-portworx_aarosAksCluster_westus2 -o table
  kubectl run -it --rm aks-ssh --image=debian
  apt-get update && apt-get install openssh-client -y
  kubectl get pods
  kubectl cp ~/.ssh/id_rsa aks-ssh-589f4659c5-jdcqr:/id_rsa
  kubectl exec -it aks-ssh-589f4659c5-jdcqr /bin/bash
  chmod 0600 id_rsa
  ssh -i id_rsa azureuser@10.240.0.4
```