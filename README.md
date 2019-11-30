Kubernetes + Minikube


Install docker
https://docs.docker.com/install/linux/docker-ce/ubuntu/

Install kubectl 

curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl

Install minikube:
You can run minikube with many different vm drivers like virtualbox, vmware, kcm2, hyperkit and others.

In this tutorial we'll use virtualbox, if you don't already have virtualbox installed just run:
sudo apt-get install virtualbox

curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube

sudo mkdir -p /usr/local/bin/
sudo install minikube /usr/local/bin/

minikube config set vm-driver virtualbox
minikube start

Do not forget to delete the cluster after you're done with it by running 'minikube delete'.

eval $(minikube docker-env)

docker build -t kubernetes-minikube-tut:1.0.0 .

kubectl create deployment kubernetes-tut --image=kubernetes-minikube-tut:1.0.0

EXECUTE POD:
	Open 2º terminal: kubectl proxy

	export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')

	curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME:8080/proxy/

EXECUTE CONTAINER:

        kubectl exec $POD_NAME -- curl -s http://localhost:8080
        (since only one container in pod, command is executed on that container)


kubectl expose deployment/kubernetes-tut --type="NodePort" --port 8080 
(isto criar um serviço que expões o pod para o exterior)(Minikube não suporta LoadBalancer)

(sendo o que o minikube corre um cluster dentro duma VM, podemos aceder ao ip externo do serviço pelo IP do VM do minikube)
kubectl get svc
(apontar o port ao lado do 8080 ou) export SVC_PORT=$(kubectl get services/kubernetes-tut -o go-template='{{(index .spec.ports 0).nodePort}}')

curl $(minikube ip):$SVC_PORT

Para ver os pods, deployments e servicços: minikube dashboard (só um node presente como o minikube inicia um cluster com só um node)





Kubernetes + AWS

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


   		   
