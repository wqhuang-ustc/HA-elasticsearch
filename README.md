# elasticsearch-assignment
This is a repo for codes required to deploy a highly-available Elasticsearch Cluster for an assignment.

## Introduction and goal
This assignment required to deploy a high-available Elasticsearch(ES) cluster using Docker and Kubernetes. Each ES instance in the ES cluster should be located on different k8s worker node for high availability consideration, also all ES instance should have the same ES roles(master, data, ingest, etc..). This solution can be deployed in cloud or on-premise k8s cluster.

## Kubernetes nodes provisioning
I prefer to provisioning Virtual Machines(VMs) using Infrastructure as Code(IaC). IaC manages infrastructures(virtual machine, networks, disks etc.) in a declative way with configuration file, it allow you to build, change and manage your infrastructure in a safe, consistent, repeatable way.

For this assignment, I will use Vagrant to create 3 Kubernetes nodes and configure them to satisfy the requirements of being a Kubernetes node. Vagant is an ideal tool to create VMs for development environment and you may use Terraform for provisioning VMs in a production environment. Check this [repo](https://github.com/wqhuang-ustc/terraform-kvm) for creating VMs using Terraform.

### Installation of Vagrant and VirtualBox
I use Vagrant version 2.2.16 and VirtualBox 6.1 on my MacOS. For Vagrant installation, check this [link](https://www.vagrantup.com/downloads). Use this [link](https://www.virtualbox.org/wiki/Downloads) to install VirtualBox in your environment.

### Configure VMs in the Vagrantfile and build
Modify this [Vagrantfile](kubespray/Vagrantfile) under kubespray directory to further customize the VMs.
1. $vm_memory (Increase to 4096M because Elasticsearch instance has minimum 2G memory requirement)
2. $subnet ||= "172.18.8" (Make sure this subnet is not already in use in your environment, change it if needed)
3. $os ||= "ubuntu1804" (Change this if you prefer other OS)

Run below commands to launch VMs for Kubernetes cluster:
```
cd elasticsearch-assignment/kubespray
vagrant up --provider=virtualbox
```

## Deploy the Kubernetes via Kubespray


## Kubernetes dashboard

## Launch High available Elasticsearch in the Kubernetes cluster

## Launch Kibana to visualize and manage the Elasticsearch cluster

