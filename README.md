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
```
curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME:8080/proxy/
```
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
To find out the exposed port of the service we can run 'kubectl get svc' and see check port under PORT, or we can run the follwing command to get the port in the shell variable SVC_PORT:
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

To delete the pods you first have to delete the service and then the deployment, as since deployment has a set minimum of pods that should run, the deleted pods will just be replaced.
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

Amazon Elastic Kubernetes Service (Amazon EKS) is a managed service that makes it easy for you to run Kubernetes on AWS without needing to stand up or maintain your own Kubernetes control plane.

### Requirements

1) Install Python: sudo apt-get install python3
   Install pip3: curl -O https://bootstrap.pypa.io/get-pip.py
                 python3 get-pip.py --user
   
   Add binaries to PATH:
      Add 'export PATH=~/.local/bin:$PATH' to ~/.profile and run 'source ~/.profile'.
      
   Install AWS CLI: pip3 install awscli --upgrade --user

   Open https://console.aws.amazon.com/iam/
	   Go to Users
	   Add User
	   Enter a name and select 'Programmatic access' as the Access type.
	   Press 'Create Group'
	   Press 'Create policy'
	   
	   Add permissions for: (podesse ser mais especifico ao selectionar ações e recursos)
	      ec2
	      eks
	      cloudformation
	      iam
	   
	   Add name kubernetes-tut-policy

	   Select created policy
	   
	   Add name to group kubernetes-tut-group

	   Select group and press 
	   
	   Skil tags

	   Press Create User

	   Download .csv file and save somewhere safe (file wont be shown again)


   Type in terminal: aws configure
   Enter the 'access key id' and 'secret access key' stored on the downloaded .csv
   Enter as region: eu-west-2
   Enter as output format json

   Install eksctl: 
   	curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
   	sudo mv /tmp/eksctl /usr/local/bin
   	
### Basic Walkthrough

   Create cluster:
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

    (--managed) %Creates EKS-managed nodegroup

Check cluster is created on https://eu-west-2.console.aws.amazon.com/ecs
Check cloudformation is created on https://eu-west-2.console.aws.amazon.com/cloudformation

Enter into terminal: kubectl apply -f hello-world-replicaset.yaml
(file uses my public docker image of server)
Enter into terminal: kubectl apply -f hello-world-service.yaml

Enter into terminal: kubectl get pods
See all 10 replica pods running

Enter into terminal: kubectl get services -o wide
Enter into browser external IP of hello-world service plus the port :8080


DEPLOY METRICS SERVER

curl -o v0.3.6.tar.gz https://github.com/kubernetes-sigs/metrics-server/archive/v0.3.6.tar.gz

tar -xzf v0.3.6.tar.gz

kubectl apply -f metrics-server-0.3.6/deploy/1.8+/
	   
kubectl get deployment metrics-server -n kube-system

kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml

kubectl apply -f eks-admin-service-account.yaml

kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
copy the code after 'token:'

kubectl proxy

Go to: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#!/login
Enter previously copied token and sign in


   		   
