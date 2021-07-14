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
Kubespray is a composition of Ansible playbooks, inventory, provisioning tools, and domain knowledge for generic Kubernetes cluster configuration management tasks. Kubespray provides:
* A highly available cluster
* Composable attributes(Choice of the network plugin for instance)
* Support for most popular Linux distributions
* Can be run on bare metal and most cloud

1. **Install [python/python3](https://linuxize.com/post/how-to-install-python-3-7-on-ubuntu-18-04/) and [pip/pip3](https://linuxize.com/post/how-to-install-pip-on-ubuntu-18.04/) in your Ansible control machine.**

2. **CLone the Kubespray git repository into the control machine.**
    ```
    sudo apt-get install git -y
    git clone https://github.com/kubernetes-sigs/kubespray.git
    git checkout tags/v2.16.0  #For installing Kubernetes 1.20 required by Elasticsearch installation
    ```
3. **Go to the Kubespray directory and install all dependency packages from "requirements.txt"(ansible, jinja2, netadd, pbr, hvac, jmespath, ruamel,yaml, etc.).**
   ```
   sudo pip install -r requirements.txt
   ```
4. **Update the Ansible inventory file using the info of VMs created above**
   Vagrant will generate the inventory automatically while launching the VMs in `.vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory`, check this inventory and confirm the details of each nodes, update it if necessary.
5. **Review and modify parameters under "inventory/sample/group_vars" according to your environment and requirements**
   ```
   vim inventory/sample/group_vars/all/all.yml
   vim inventory/sample/group_vars/k8s-cluster/k8s-cluster.yml
   ```
6. **Deploy Kubespray with Ansible Playbook**
   ```
   ansible-playbook -vvv -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory cluster.yml
   ```
   Or use the Ansible provisioner in the Vagrantfile to launch the Kubernetes after creating all VMs successfully by uncommnet the last part of the Vagrantfile and run `Vagrant up`.
   ```
      # # Only execute the Ansible provisioner once, when all the machines are up and ready.
      # if i == $num_instances
      #   node.vm.provision "ansible" do |ansible|
      #     ansible.playbook = $playbook
      #     $ansible_inventory_path = File.join( $inventory, "hosts.ini")
      #     if File.exist?($ansible_inventory_path)
      #       ansible.inventory_path = $ansible_inventory_path
      #     end
      #     ansible.become = true
      #     ansible.limit = "all,localhost"
      #     ansible.host_key_checking = false
      #     ansible.raw_arguments = ["--forks=#{$num_instances}", "--flush-cache", "-e ansible_become_pass=vagrant"]
      #     ansible.host_vars = host_vars
      #     #ansible.tags = ['download']
      #     ansible.groups = {
      #       "etcd" => ["#{$instance_name_prefix}-[1:#{$etcd_instances}]"],
      #       "kube_control_plane" => ["#{$instance_name_prefix}-[1:#{$kube_master_instances}]"],
      #       "kube_node" => ["#{$instance_name_prefix}-[1:#{$kube_node_instances}]"],
      #       "k8s_cluster:children" => ["kube_control_plane", "kube_node"],
      #     }
      #   end
      # end
   ```
7. **Access k8s cluster viakubectl from the console by copy the /etc/kubernetes/admin.conf file to $HOME/.kube/config in the master node.**  
    
    ```
    vagrant ssh k8s-1
    mkdir -p $HOME/.kube; sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
    Verify kubectl is working by:  
    ```
    kubectl get nodes
    kubectl cluster-info
    ```
    To access the k8s cluster from your local machine, copy the contents of the /etc/kubernetes/admin.conf file into $HOME/.kube/config file in your machine.
    By default, kubectl looks for a file named config in the $HOME/.kube directory. You can specify other kubeconfig files by setting the KUBECONFIG environment variable or by setting the --kubeconfig=/path/to/your-config flag. If you are managing multiple Kubernetes cluster in your local environment, use contexts to control the access of multiple Kubernetes cluster.
    ```
    export KUBECONFIG=$KUBECONFIG:$HOME/.kube/picnic-config
    kubectl config get-contexts
    kubectl config use-context picnic-kubernetes-admin@picnic-cluster
    ```

## Kubernetes dashboard
A Kubernetes dashboard will be deployed into the cluster first to provide a web-based UI for this Kubernetes clusters. It allows users to manage applications running in the cluster and troubleshoot them, as well as manage the cluster itself.  
To install the Dashboard, execute following command:
```
kubectl apply -f k8s-dashboard/k8s-dashboard.yaml
kubectl apply -f dashboard-adminuser.yaml
```

To access the Dashboard locally, Create a secure channel to the Kubernetes cluster.
```
kubectl proxy
```
Now access Dashboard at `http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`. Find the token we can use to log in by running following command:
```
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
```

## Launch High available Elasticsearch in the Kubernetes cluster
To create a high available Elasticsearch cluster in the Kuberentes cluster, we will deploy a ES StatefulSet with 3 replicas. Each replicas(ES instance) will be scheduled into different Kubernetes node to prevent singel node failure. All the ES instances have [node roles](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html) of "master", "data", "ingest" and "ml".

1. **Create persistent volume for each ES instance**

An example of pv is shown below:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-pv0
  labels:
    volume: elastic-volume
spec:
  storageClassName: local
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  claimRef:
    namespace: default
    name: elasticsearch-data-elasticsearch-demo-es-default-0
  hostPath:
    path: /opt/elasticsearch
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-1
```

* "nodeAffinity" to create pv on a specific node.
* "persistentVolumeReclaimPolicy: Retain" to still keep the pv while pvc is deleted.
* "claimRef" to reserve the volume for a pvc.

Execute following commands to create PersistentVolumes for 3 ES instance:
```
kubectl apply -f elk7.12/persistent-volume-0.yaml
kubectl apply -f elk7.12/persistent-volume-1.yaml
kubectl apply -f elk7.12/persistent-volume-1.yaml
```

2. **Launch 3 replicas ES StatefulSet**

We will use the official Elastic Cloud on Kubernetes(ECK) to deploy Elasticsearch cluster and Kibana. ECK extends the basic Kubernetes orchestration capabilities to support the setup and management of Elasticsearch, Kibana, APM Server, Enterprise Search, and Beats on Kubernetes. With Elastic Cloud on Kubernetes, we can perform critical operations, such as:
* Managing and monitoring multiple clusters
* Scaling cluster capacity and storage
* Performing safe configuration changes through rolling upgrades
* Securing clusters with TLS certificates
* Setting up hot-warm-cold architectures with availability zone awareness

First, install the extensions of the Kubernetes API [custom resource definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) and the operator with its RBAC rules:
```
kubectl apply -f elk7.12/all-crds.yaml
kubectl -n elastic-system get all # To check the status of elastic-operator
```

Second, deploy the ES cluster by executing following command:
```
kubectl apply -f elk7.12/elasticsearch.yaml
```
The elastic-operator automatically creates and manages Kubernetes resources to achieve the desired state of the Elasticsearch cluster. It may take up to a few minutes until all the resources are created and the cluster is ready for use.  
To get an overview of the current Elasticsearch cluster in the Kuberentes cluster, including health, version and number of ndoes:
```
kubectl get elasticsearch
```
```
NAME                 HEALTH   NODES   VERSION   PHASE   AGE
elasticsearch-demo   green    3       7.10.1    Ready   19h
```
To access the logs for the ES pod:
```
kubectl logs -f name-of-ES-pod
```
## Launch Kibana to visualize and manage the Elasticsearch cluster

Kibana is an free and open frontend application that sits on top of the Elastic Stack, providing search and data visualization capabilities for data indexed in Elasticsearch. Commonly known as the charting tool for the Elastic Stack. Kibana also acts as the user interface for monitoring, managing, and securing an Elastic Stack cluster

To deploy an Kibana instance for our ES cluster:
```
kubectl apply -f elk7.12/kibana.yaml
```
I change the default service type from "ClusterIP" to "NodePort" to expose the Kibana service outside the cluster. Otherwise, you need to use port-forward to be able to visit the kibana instance from your local browser via `https://localhost:5601`.
```
kubectl port-forward service/kibana-demo-kb-http 5601
```
Login as the elastic user. The password can be obtained with the following command:
```
kubectl get secret elasticsearch-demo-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo
```

## High Availability consideration

The ES cluster we deployed above meet the requirements of being a resilient cluster and can keep working enen if some of the nodes are unavailable or disconnected:
* At least three master-eligible nodes
* At least two nodes of each role
* At least two copies of each shard

Usually we deploy instances of ES cluster in the same datacenter considering the network latency between Data Centers. This means we could loss the accss to this ES cluster if the whole Data Center is down for some reasons. To prevent one Data Center outage, we can use [cross-cluster replication](https://www.elastic.co/guide/en/elasticsearch/reference/current/xpack-ccr.html) to replicate data to a remote follower ES cluster in another Data Center. Then, we can failover to this remote ES cluster if current Data Center is down.

Besides, we can take [regular snapshots](https://www.elastic.co/guide/en/elasticsearch/reference/current/backup-cluster.html) of our ES cluster so that we can restore a completely fresh copy of it elsewhere if needed.

## Monitoring of the ES cluster
Elasticsearch provides plenty of metrics that can help us detect signs of trouble and take actions. A few key areas to monitor are:
1. Search and indexing performance
2. Memory and garbage collection
3. Host-level system and network metrics
4. Cluster health and node availability
5. Resource saturation and errors

Some of the basic metrics are directly available in the Kibana Cluster overview. For metrics not inclueded in Kibana, you can alway fetch them via [Elasticsearch's RESTful API](https://www.elastic.co/guide/en/elasticsearch/reference/current/rest-apis.html)

## Reference
* https://github.com/kubernetes-sigs/kubespray
* https://github.com/kubernetes-sigs/kubespray/blob/master/docs/vagrant.md
* https://github.com/elastic/cloud-on-k8s
* https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-quickstart.html
* https://www.datadoghq.com/blog/monitor-elasticsearch-performance-metrics/