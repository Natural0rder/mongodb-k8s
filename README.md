# mongodb-k8s
How to deploy a MongoDB cluster and Ops Manager on Kubernetes leveraging Minikube, MongoDB Kubernetes Operator, Helm and a single AWS EC2 instance.

- https://www.mongodb.com/docs/kubernetes-operator/master/kind-quick-start/
- https://www.mongodb.com/blog/post/running-mongodb-ops-manager-in-kubernetes
- https://www.mongodb.com/blog/post/tutorial-part-2-ops-manager-in-kubernetes

## Prerequisites

Launch a AWS EC2 t3.xlarge instance on Ubuntu Server 22.04 LTS with a 30GB root volume.
Edit the associated Security Group to allow all inbound traffic to ease the exercise.

### Connect to instance with SSH

    ssh -i "<pem-file>.pem" ubuntu@ec2-XX-XXX-XXX-XXX.eu-west-3.compute.amazonaws.com

### Install kubectl

    curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
    chmod +x ./kubectl
    sudo mv ./kubectl /usr/local/bin/kubectl

### Install docker

    sudo apt-get update && \
    sudo apt-get install docker.io -y
    
### Install Minikube

    curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
    minikube version
    
 ### Install conntrack
 
     sudo apt install conntrack
     
 ### Install cri-dockerd
 
    wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.2.0/cri-dockerd-v0.2.0-linux-amd64.tar.gz
    tar xvf cri-dockerd-v0.2.0-linux-amd64.tar.gz
    sudo mv ./cri-dockerd /usr/local/bin/ 
    cri-dockerd --help
    
    wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
    wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
    sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
    sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
    
    sudo systemctl daemon-reload
    sudo systemctl enable cri-docker.service
    sudo systemctl enable --now cri-docker.socket
    sudo systemctl status cri-docker.socket
    
    sudo sysctl fs.protected_regular=0

### Install crictl

    VERSION="v1.24.1"
    wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
    sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
    rm -f crictl-$VERSION-linux-amd64.tar.gz
    
### Start Minikube

    sudo -i
    minikube start --vm-driver=none
    minikube status

<img width="856" alt="Screenshot 2022-08-11 at 09 59 16" src="https://user-images.githubusercontent.com/102281652/184089502-bf5e544f-fcb8-4429-8c66-2e8dd8a6fcdc.png">

<img width="286" alt="Screenshot 2022-08-11 at 10 20 34" src="https://user-images.githubusercontent.com/102281652/184092936-77ee1bf9-d6a2-4035-879f-b21512b8e12d.png">

### Install HELM
 
    curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
    sudo apt-get install apt-transport-https --yes
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
    sudo apt-get update
    sudo apt-get install helm
    
### Install mongosh

    wget -qO - https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add -
    echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
    sudo apt-get update
    sudo apt-get install -y mongodb-mongosh

### Start Kubernetes Dashboard

    minikube dashboard --url
    
<img width="742" alt="Screenshot 2022-08-11 at 10 24 16" src="https://user-images.githubusercontent.com/102281652/184093755-526f55d2-f6a6-4124-97be-456bd666aa9b.png">

### Open another terminal an create SSH tunnel

    ssh -i "<pem-file>.pem" -L 8081:localhost:[remote port of minikube dashboard] ubuntu@[ec2 public ip]
    
### Browse from your local machine

    http://127.0.0.1:8081/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/#/error?namespace=_all
    
<img width="1708" alt="Screenshot 2022-08-11 at 10 26 30" src="https://user-images.githubusercontent.com/102281652/184093793-1cefb124-9f76-40a7-91d7-9bf17b45905c.png">
    
## Deploy MongoDB Kubernetes Enterprise Operator

### Install with HELM

    helm repo add mongodb https://mongodb.github.io/helm-charts
    helm install enterprise-operator mongodb/enterprise-operator --namespace mongodb --create-namespace
    kubectl config set-context $(kubectl config current-context) --namespace=mongodb
 
 ## Deploy Ops Manager

### Create Ops Manager Kubernetes Secret (to be used for sign-in to the Ops Manager web portal)

    kubectl create secret generic ops-manager-admin-secret --from-literal=Username="<email-for-login>"  --from-literal=Password="<complex-password>" --from-literal=FirstName="firstname" --from-literal=LastName="lastname" -n mongodb

### Create file ops-manager.yaml and copy content

```
apiVersion: mongodb.com/v1
kind: MongoDBOpsManager
metadata:
 name: ops-manager
 namespace: mongodb
spec:
 replicas: 1
 version: "5.0.0"
 adminCredentials: ops-manager-admin-secret
 externalConnectivity:
  type: NodePort
 applicationDatabase:
  members: 3
  version: "5.0.5-ent"
```

    touch ops-manager.yaml
    vi ops-manager.yaml
   
### Deploy Ops Manager Kubernetes Object

    kubectl apply -f ops-manager.yaml
    
### Monitor provisioning

    kubectl get om -n mongodb
    kubectl get om -o yaml -w
    
<img width="731" alt="Screenshot 2022-08-11 at 10 44 34" src="https://user-images.githubusercontent.com/102281652/184096683-519b16fd-ef8d-4bf9-9e95-e08d298308d6.png">
    
### Get and browse Ops Manager portal

    kubectl get svc

<img width="781" alt="Screenshot 2022-08-11 at 10 46 17" src="https://user-images.githubusercontent.com/102281652/184096971-2cb6f57c-c178-4d79-910b-477a49486110.png">
  
- Add the proper inbound traffic rule at Security Group level

<img width="1175" alt="Screenshot 2022-08-11 at 10 54 41" src="https://user-images.githubusercontent.com/102281652/184098962-2a0097c4-db63-49fd-bad0-5ea9ffbb5e6d.png">

- Report exposed port for ops-manager-svc-ext to access web portal (ex: http://[ec2 public ip]:31618)
    
<img width="458" alt="Screenshot 2022-08-11 at 10 55 14" src="https://user-images.githubusercontent.com/102281652/184099026-cd954e7e-e294-43e7-bad8-59bb914136e6.png">

- Fill the Ops Manager admin form and browse the ops-manager-db organization to check the AppDB

<img width="1712" alt="Screenshot 2022-08-11 at 11 03 04" src="https://user-images.githubusercontent.com/102281652/184099992-b7077092-84cf-4bcb-add4-9b12a6d9dbdf.png">

### Create a new Organization

<img width="699" alt="Screenshot 2022-08-11 at 11 05 40" src="https://user-images.githubusercontent.com/102281652/184100304-0e2de64e-8f2d-4b04-896c-5c286cb67f78.png">

### Create an API Key for the new Organization

- At Access Manager/Organization level, create a new API Key (role = Organization Owner)

<img width="423" alt="Screenshot 2022-08-11 at 11 07 19" src="https://user-images.githubusercontent.com/102281652/184100708-7a3d316e-a857-4afa-86bf-d07f677c4032.png">

- Save keys and add the Kubernetes Operator POD IP address in the Access List

<img width="812" alt="Screenshot 2022-08-11 at 11 09 46" src="https://user-images.githubusercontent.com/102281652/184100956-e3757bc3-c6b7-4e89-9c76-43dbe14959f8.png">

## Deploy a MongoDB Replica Set

### Create a Project for the Organization

<img width="694" alt="Screenshot 2022-08-11 at 11 12 49" src="https://user-images.githubusercontent.com/102281652/184102000-eee3a02c-b0bb-4fd7-a56a-b0a915560fb3.png">

### Generate and apply YAML (secret, configMap) from Ops Manager UI

- At Project/Deployment level, hit the Setup Kubernetes button and use the existing API Key

<img width="1149" alt="Screenshot 2022-08-11 at 11 20 50" src="https://user-images.githubusercontent.com/102281652/184102740-476c4b22-084a-4e52-835c-0dec5450f65c.png">

- Generate YAML

<img width="941" alt="Screenshot 2022-08-11 at 11 22 32" src="https://user-images.githubusercontent.com/102281652/184103056-20370deb-7d7c-4788-b010-5f881d1f9fdd.png">

```
apiVersion: v1
kind: Secret
metadata:
  name: organization-secret
  namespace: mongodb
stringData:
  user: <privateKey>
  publicApiKey: <publicKey>
```
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-project
  namespace: mongodb
data:
  baseUrl: http://ops-manager-svc.mongodb.svc.cluster.local:8080

  # Optional Parameters
  projectName: <projectName>

  orgId: <orgId>
```
    kubectl apply -f secret.yaml -f config-map.yaml

### Create replica-set.yaml and copy content

```
apiVersion: mongodb.com/v1
kind: MongoDB
metadata:
  name: my-cluster
  namespace: mongodb
spec:
  members: 3
  version: "5.0.5-ent"
  type: ReplicaSet
  opsManager:
    configMapRef:
      name: my-project
  credentials: organization-secret
```

    touch replica-set.yaml  
    vi replica-set.yaml

### Deploy the replica set

    kubectl apply -f replica-set.yaml -n mongodb
    kubectl get mdb -n mongodb -w
    kubectl get mdb -o yaml -w
    
- Check the provisioning

<img width="390" alt="Screenshot 2022-08-11 at 11 29 31" src="https://user-images.githubusercontent.com/102281652/184104385-756bca59-706b-44b1-ae53-ecaf26bf2ed1.png">

- Check at Ops Manager level

<img width="1467" alt="Screenshot 2022-08-11 at 11 30 11" src="https://user-images.githubusercontent.com/102281652/184104506-22f5db8a-3d0c-4f26-b42b-3ab09294d733.png">
    
## Apply an upgrade

### Update replica-set.yaml with a 5 nodes replica set

    members: 5
    
### Deploy the change

    kubectl apply -f replica-set.yaml -n mongodb

## Connect with mongosh

### Get the primary node POD IP

    mongosh "mongodb://<my-replica-set-0-IP>:27017"
