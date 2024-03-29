## Kubernetes Using Minikube

Minikube is used as an easy tool to run Kubernetes locally. Minikube runs a cluster, with a single node, inside a virtual machine on your computer, it's ideal for users trying out Kubernetes.

### Requirements

**Install docker**

Follow the tutorial: https://docs.docker.com/install/linux/docker-ce/ubuntu/

**Install kubectl**
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

**Install minikube**
```
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
sudo mkdir -p /usr/local/bin/
sudo install minikube /usr/local/bin/
```

You can run minikube with many different vm drivers like virtualbox, vmware, kcm2, hyperkit and others...

You can use virtualbox if you already have it installed.
If you don't already have virtualbox installed you can just run:
```
sudo apt-get install virtualbox
```

Set virtualbox as the default vm-driver which minikube will use to run its virtual machine.
```
minikube config set vm-driver virtualbox
```

### Basic Walkthrough

Create your local cluster using minikube start.
```
minikube start
```

To avoid having to upload your docker images to docker hub before testing them in your local cluster, you can build your images and deploy them locally using 'eval $(minikube docker-env)'.
Run the following commands in the root of this repository:
```
eval $(minikube docker-env)

docker build -t kubernetes-minikube-tut:1.0.0 ./minikube

kubectl create deployment kubernetes-tut --image=kubernetes-minikube-tut:1.0.0
```
You can check that the deployment was created by calling
```
kubectl get deployments
```
There will only be one pod created in this deployment by default but we could scale the number of replicas by calling 'kubectl scale'.
```
kubectl get pods
```

To execute commands inside your pods without exposing them yet with services, you can run a proxy to the kubernetes api server.
First open a terminal and run:
```
kubectl proxy
```
Now open a second terminal and run the following command to get the created pod's name. 
```
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
```
Now you'll be able to comunicate directly with the pod.
On the same terminal run the command:
```
curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME:8080/proxy/
```
Close the proxy by closing the terminal it is running on.

To execute commands inside the container of the pod we can use the 'kubectl exec' command. 
```
kubectl exec $POD_NAME -- curl -s http://localhost:8080
```
Since only one container is in the pod, the command is executed on that container.

To expose the application running on the pods to outside the cluster while load balancing between pod replicas we create a service.
Minikube does not support LoadBalancer type services so we'll use NodePort to expose the service on each node’s IP at a static port.
```
kubectl expose deployment/kubernetes-tut --type="NodePort" --port 8080 
```

Since minikube runs a cluster on a virtual machine, the VM is exposed to the host system via an IP address that can be obtained with the 'minikube ip' command.
To find out the exposed port of the service we can run 'kubectl get svc' and see the port under PORT, or we can run the following command to set the shell variable SVC_PORT to the port:
```
export SVC_PORT=$(kubectl get services/kubernetes-tut -o go-template='{{(index .spec.ports 0).nodePort}}')
```
Now we can use the service to communicate with the application running on the minikube VM's cluster.
```
curl $(minikube ip):$SVC_PORT
```

To check all running pods, deployments, services and other information on the cluster running on minikube's VM we can use minikube's dashboard. To open it run the command:
```
minikube dashboard
```

To delete the pods you first have to delete the service and then the deployment, since the deployment has a set minimum of pods that it runs, the deleted pods will just be replaced while the deployment is running.
```
kubectl delete svc kubernetes-tut
kubectl delete deployment kubernetes-tut
kubectl delete pod $POD_NAME
```
You can also delete these objects from the minikube dashboard.

Do not forget to delete the cluster after you're done with it at the end by running:
```
minikube delete
```


## Kubernetes Using AWS

We'll use the EKS service or Elastic Kubernetes Service to make it easy for us to run kubernetes on AWS without having to maintain our own kubernetes control plane.

### Requirements

**Install Python**
```
sudo apt-get install python3
```

**Install Pip3**
```
curl -O https://bootstrap.pypa.io/get-pip.py
python3 get-pip.py --user
```

Too add binaries to PATH add 'export PATH=~/.local/bin:$PATH' to ~/.profile and run 'source ~/.profile'.
      
**Install AWS CLI**
```
pip3 install awscli --upgrade --user
```   

**Create User Permissions**

Open https://console.aws.amazon.com/iam/

  1)  Go to Users;
  2)  Add User;
  3)  Enter a name and select 'Programmatic access' as the Access type;
  4)  Press 'Create Group';
  5)  Press 'Create policy';
  6)  Add permissions for the services: eks, ec2, cloudformation and iam;
  7)  Add name to policy kubernetes-tut-policy
  8)  Select created policy
  9)  Add name to group kubernetes-tut-group
 10)  Select group and press 
 11)  Skip tags
 12)  Press Create User
 13)  Download .csv file and save somewhere safe (file wont be shown again)
	  
Enter in terminal: 
```
aws configure
```
Enter the 'access key id' and 'secret access key' stored on the downloaded .csv
Enter as region: eu-west-2
Enter as output format json

**Install eksctl**
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```
   	
### Basic Walkthrough

Create the cluster:
```
eksctl create cluster \
--name kuber-tut-cluster \
--version 1.14 \
--region eu-west-2 \
--nodegroup-name kuber-tut-standard-workers \
--node-type t3.medium \
--nodes 3 \
--nodes-min 1 \
--nodes-max 4 \
--managed
```

Check that the cluster is created on: https://eu-west-2.console.aws.amazon.com/ecs
Check that the cloudformation is created on: https://eu-west-2.console.aws.amazon.com/cloudformation

Navigate the terminal to the root of this repository.
Enter into terminal: 
```
kubectl apply -f ./aws/hello-world-replicaset.yaml
```
(file uses public docker image of server)

Enter into terminal: 
```
kubectl apply -f ./aws/hello-world-service.yaml
```

Enter into terminal: 
```
kubectl get pods
```
All 10 replica pods should be running since the hello-world-replicaset.yaml file specifies replication of 10.

Enter into terminal: 
```
kubectl get services -o wide
```
Copy the external IP address, and add at the end of the addess the server port 8080. 
Enter into terminal: 
```
curl <EXTERNAL IP ADDRESS>:8080
```
If everything is working the above command should output "Hello World!"

### Deploy the metrics server

To view the statistics of our AWS cluster on a dashboard like we did with minikube, we have to deploy the metrics server.
Do do so execute the following commands:
```
curl -o v0.3.6.tar.gz https://github.com/kubernetes-sigs/metrics-server/archive/v0.3.6.tar.gz

tar -xzf v0.3.6.tar.gz

kubectl apply -f metrics-server-0.3.6/deploy/1.8+/
	   
kubectl get deployment metrics-server -n kube-system

kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml

kubectl apply -f ./aws/eks-admin-service-account.yaml

kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```
Copy the code after 'token:' outputted into the terminal.

Finally execute:
```
kubectl proxy
```
Enter into a browser the address:
```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#!/login
```
Select Token, enter the previosuly copied token into the textarea and Sign In.


   		   
