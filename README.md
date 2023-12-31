# k3s Setup and Configuration on Amazon EC2 Instance
## Introduction
We will be using an Amazon EC2 instance with operating system _Ubuntu 22.04.3 LTS_, kernel _GNU/Linux 6.2.0-1012-aws_, architecture _x86-64_, and a single EBS volume of 64GB to setup and configure a Kubernetes cluster using the [k3s distribution](https://docs.k3s.io/). We'll go with a K3s single-server setup.        
![K3s Single Server Setup](/public/assets/images/k3s-architecture.png "K3s Single Server Setup")   

The ultimate goal for having this cluster up and running is to administer and orchestrate the [Gen3 stack](https://gen3.org/resources/operator/index.html), which consists of over a dozen containerised microservices.   

## Setting up a Custom Virtual Private Cloud (VPC)
Using the Amazon console, we'll create a custom VPC with the configuration values listed below.   

### Create VPC
Name: gen3-vpc
IPv4: CIDR Block: 10.0.0.0/16   

### Create Public Subnets
Name: public-1a   
Availability Zone: us-east-1a   
IPv4 CIDR Block: 10.0.1.0/24   

Name: public-1b   
Availability Zone: us-east-1b   
IPv4 CIDR Block: 10.0.2.0/24   

### Create Private Subnets
Name: private-1a   
Availability Zone: us-east-1a   
IPv4 CIDR Block: 10.0.3.0/24   

Name: private-1b   
Availability Zone: us-east-1b   
IPv4 CIDR Block: 10.0.4.0/24   

### Create Private Route Table
Name: private-rt   
VPC: gen3-vpc    
Subnet Associations: private-1a, private-1b    

### Create Internet Gateway
Name: gen3-internet-gateway      
VPC: gen3-vpc     

## Installation
### K3s
The installation script for K3s can be used as follows:
```bash
curl -sfL https://get.k3s.io | sh -
```
The installation includes additional utilities such as `kubectl`, `crictl`, `ctr`, `k3s-killall.sh`, and `k3s-uninstall.sh`. `kubectl` will automatically use the `kubeconfig` file that gets written to `/etc/rancher/k3s/k3s.yaml` after the installation. By default, the container runtime that K3s uses is `containerd`. Docker is not needed, but can be installed if desired.   

If we check the present working directory by running `pwd`, we should get 
```bash
/home/ubuntu
```
We might encounter some permissions issues when trying to use the `kubectl` command line tool. This can be resolved by running the following commands:
```bash
mkdir ~/.kube
sudo k3s kubectl config view --raw | tee ~/.kube/config
chmod 600 ~/.kube/config
export KUBECONFIG=~/.kube/config
```
For these updates to persist upon reboot, the `.profile` and `.bashrc` files should be updated with `export KUBECONFIG=~/.kube/config`.   

### Helm
[Helm](https://helm.sh/) is a package manager for Kubernetes that allows for the installation or deployment of applications onto a Kubernetes cluster. We can install it as follows:
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
To see if Helm has been installed, we can run a simple `helm` command like:
```bash
helm list
```
and we should get an empty table as our output,
| NAME          | NAMESPACE | REVISION  | UPDATED | STATUS  | CHART | APP VERSION |
| ------------- | --------- | --------- | ------- | ------- | ----- | ----------- |
|               |           |           |         |         |       |             |

### Longhorn
[Longhorn](https://longhorn.io/docs/1.5.1/) is an open-source distributed block storage system for Kubernetes, and is supported by K3s. To install Longhorn, we apply the `longhorn.yaml`:

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.5.1/deploy/longhorn.yaml
```

We need to create a persistent volume claim and a pod to make use of it:   

**longhorn-volv-pvc.yaml**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-volv-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 2Gi
```

**volume-test-pod.yaml**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
  namespace: default
spec:
  containers:
  - name: volume-test
    image: nginx:stable-alpine
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: volv
      mountPath: /data
    ports:
    - containerPort: 80
  volumes:
  - name: volv
    persistentVolumeClaim:
      claimName: longhorn-volv-pvc
```
To create the `longhorn-volv-pvc` and `volume-test-pod` defined above, we need to be apply the YAML:
```bash
kubectl create -f longhorn-volv-pvc.yaml
kubectl create -f volume-test-pod.yaml
```
The creation of the persistent volume and persistent volume claim can be confirmed by running the following command:
```bash
kubectl get pv
kubectl get pvc
```

### Installing Gen3 Microservices with Helm
The Helm charts for the Gen3 services can be found in the [uc-cdis/gen3-helm repository](https://github.com/uc-cdis/gen3-helm.git). We'd like to add the Gen3 Helm chart repository. To do this, we run:  

```bash
helm repo add gen3 http://helm.gen3.org
helm repo update
```
The Gen3 Helm chart repository contains the templates for all the microservices making up the Gen3 stack. For the `elastic-search-deployment` to run in a Linux host machine, we need to increase the max virtual memory areas by running:
```bash
sudo sysctl -w vm.max_map_count=262144
``` 
This setting will only last for the duration of the session. The host machine will be reset to the original value if it gets rebooted. For this change to be set permanently on the host machine, the `/etc/sysctl.conf` file needs to be edited with `vm.max_map_count=262144`. To see the current value, run `/sbin/sysctl vm.max_map_count`. More details can be found on the [official Elasticsearch website](https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html).   

Some of the microservices require the `uwsgi-plugin` for Python 3. To install it, run the following:
```bash
sudo apt update  
sudo apt install uwsgi-plugin-python3 
```

Before performing a `helm install`, we need to create a `values.yaml` file. This file should be inside the root and contain the contents of the `values.yaml` file that can be found in the root of this repository. Now the Helm installation can begin by running:
```bash
helm upgrade --install gen3-dev gen3/gen3 -f gen3/values.yaml 
```
In the above command, `gen3-dev` is the name of the release of the helm deployment. If the installation is successful, then a message similar to the following should be displayed in the terminal:
```bash
NAME: gen3-dev
LAST DEPLOYED: Tue Nov 14 13:27:49 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
```
If all went well, we should see the `revproxy-dev` deployment up and running with the following command:
```bash
kubectl get ingress
```
The output should look similar to this:
| NAME          | CLASS   | HOSTS           | ADDRESS    | PORTS   | AGE |
| ------------- | ------- | --------------- | ---------- | ------- | --- |
| revproxy-dev  | traefik | gen3local.co.za | 10.0.2.238 | 80, 443 | 34s |

The list of deployments can be seen by running:
```bash
kubectl get deployments
```    

![Gen3 services deployed](/public/assets/images/gen3-deployments.png "Gen3 services deployed")  

### Grafana OSS in Kubernetes (Optional)   
Grafana open source software (OSS) allows for the querying, visualising, alerting on, and exploring of metrics, logs, and traces wherever they are stored. Graphs and visualisations can be created from time-series database (TSDB) data with the tools that are provided by Grafana OSS. We'll be using the [Grafana documentation](https://grafana.com/docs/grafana/latest/setup-grafana/installation/kubernetes/) to guide us in installing Grafana in our k8s cluster.   

To create a namespace for Grafana, run the following command:
```bash
kubectl create namespace gen3-grafana
```
We'll create a `grafana.yaml` file which will contain the blueprint for a persistent volume claim (pvc), a service of type loadbalancer, and a deployment. This file can be found in the `gen3` directory of this repo. To create these resources, we need to apply the manifest as follows:
```bash
kubectl apply -f gen3/grafana.yaml --namespace=gen3-grafana
```
To get all information about the Grafana deplyment, run:
```bash
kubectl get all --namespace=gen3-grafana
```
![Grafana k8s Objects](/public/assets/images/grafana-k8s-objects.png "Grafana k8s Objects")  

The `grafana` service should have an **EXTERNAL-IP**. This IP can be used to access the Grafana sign-in page in the browser. If there is no **EXTERNAL_IP**, then port-forwarding can be performed like this:
```bash
kubectl port-forward service/grafana 3000:3000 --namespace=gen3-grafana
```
Then the Grafana sign-in page can be accessed on `http://localhost:3000`. Use `admin` for both the username and the password.   

Since we are working on an Amazon EC2 instance, we would need to make use of the _auto-assigned IP address_ of the EC2 instance instead of localhost.   
![Grafana Login Page](/public/assets/images/grafana-login-page.png "Grafana Login Page")   

After logging in, we are free to explore the various features that Grafana offers...
![Grafana Landing Page](/public/assets/images/grafana-landing-page.png "Grafana Landing Page")   

**NOTE:** For access from a browser, we need to ensure that the inbound rules of the security group assigned to this EC2 instance allows for traffic to access the port 3000.
![Security Group Inbound Rules](/public/assets/images/security-group-inbound-rules.png "Security Group Inbound Rules")  

### Docker (Optional)
Let us go through the steps to install [Docker](https://docs.docker.com/get-started/overview/). We'll begin by doing a general software dependency update:
```bash
apt-get update
```
Add the official Docker GPG key:
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
Add the Docker repository:
```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list &gt; /dev/null
```
Install the necessary Docker dependencies:
```bash
sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release -y
```
The following two commands will install the latest version of the Docker engine:
```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io -y
```
Add the user to the Docker group  with the following command:
```bash
sudo usermod -aG docker $USER
```
End the terminal session and then restart it. Docker should be installed.    

### Running a PostgreSQL Database inside a Docker Container (Optional)
A postgreSQL database can be created to run inside a Docker container. This should not be in the Kubernetes cluster. This is not necessary for testing, but would be required if a persistent database is required. The following commands can be copied into a script called `init-db.sh` for convenience, or they could be run independently, but sequentially, as follows:
```bash
echo "Start postgres docker container"
docker run --rm --name gen3-dev-db -e POSTGRES_PASSWORD=gen3-password -d -p 5432:5432 -v postgres_gen3_dev:/var/lib/postgresql/data postgres:14
echo "Database starting..."
sleep 10
echo "Create gen3 Database"
docker exec -it gen3-dev-db bash -c 'PGPASSWORD=gen3-password psql -U postgres -c "create database gen3_db"'
echo "Create gen3_schema Schema"
docker exec -it gen3-dev-db bash -c 'PGPASSWORD=gen3-password psql -U postgres -d gen3_db -c "create schema gen3_schema"'
```
If the script runs successfully, the output should look like:
![Gen3 PostgreSQL Database](/public/assets/images/gen3-db.png "Gen3 PostgreSQL Database")   
By default, the hostname of the database is the container id.   

### Installing the k9s Tool (Optional)
**k9s** is a useful tool that makes troubleshooting issues in a Kubernetes cluster easier. Installing it is highly recommended. It can be downloaded and installed as follows:
```bash
wget https://github.com/derailed/k9s/releases/download/v0.25.18/k9s_Linux_x86_64.tar.gz
tar -xzvf k9s_Linux_x86_64.tar.gz
chmod +x k9s
sudo mv k9s /usr/local/bin/
```
To open it, simply run:
```bash
k9s
```