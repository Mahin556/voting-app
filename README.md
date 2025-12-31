### Docker Compose

```bash
git clone https://github.com/Mahin556/voting-app.git

cd voting-app

openssl rand -base64 756 > mongo-keyfile
chmod 400 mongo-keyfile
chown 999:999 mongo-keyfile

docker compose up -d mongo-0 mongo-1 mongo-2

docker exec -it root-mongo-0-1 mongosh \
  -u root \
  -p example \
  --authenticationDatabase admin

use admin

rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-0:27017" },
    { _id: 1, host: "mongo-1:27017" },
    { _id: 2, host: "mongo-2:27017" }
  ]
})

use langdb;

db.languages.insert({"name" : "csharp", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 5, "compiled" : false, "homepage" : "https://dotnet.microsoft.com/learn/csharp", "download" : "https://dotnet.microsoft.com/download/", "votes" : 0}});
db.languages.insert({"name" : "python", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 3, "script" : false, "homepage" : "https://www.python.org/", "download" : "https://www.python.org/downloads/", "votes" : 0}});
db.languages.insert({"name" : "javascript", "codedetail" : { "usecase" : "web, client-side", "rank" : 7, "script" : false, "homepage" : "https://en.wikipedia.org/wiki/JavaScript", "download" : "n/a", "votes" : 0}});
db.languages.insert({"name" : "go", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 12, "compiled" : true, "homepage" : "https://golang.org", "download" : "https://golang.org/dl/", "votes" : 0}});
db.languages.insert({"name" : "java", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 1, "compiled" : true, "homepage" : "https://www.java.com/en/", "download" : "https://www.java.com/en/download/", "votes" : 0}});
db.languages.insert({"name" : "nodejs", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 20, "script" : false, "homepage" : "https://nodejs.org/en/", "download" : "https://nodejs.org/en/download/", "votes" : 0}});

db.languages.find().pretty();

exit

docker compose up -d backend

docker compose up -d frontend
```

---

### Kubernetes

```bash
#Install Kubectl
#https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

---

```bash
#Install awscli
#https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

---

```bash
#Install eksctl
#https://github.com/eksctl-io/eksctl
#https://docs.aws.amazon.com/eks/latest/eksctl/installation.html
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl
```

---

```bash
aws configure
aws sts get-caller-identity
```

---

```bash
# Create cluster with managed node group
eksctl create cluster \
  --name my-eks-cluster \
  --region ap-south-1 \
  --nodegroup-name ng-1 \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --version 1.33 \
  --managed
```

```bash
# Update kubeconfig manually (if needed)
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name my-eks-cluster
```

```bash
# Enable IAM OIDC provider (required for IRSA)
eksctl utils associate-iam-oidc-provider \
  --cluster my-eks-cluster \
  --region ap-south-1 \
  --approve
```

---

```bash
kubectl get nodes
kubectl get pods -A

git clone https://github.com/Mahin556/voting-app.git

cd voting-app

kubectl create ns voting-app
kubectl config set-context --current --namespace voting-app
cd manifests
kubectl apply -f mongo-statefullset.yaml #To create Mongo statefulset with Persistent volumes.
kubectl apply -f mongo-service.yaml

#Create a temporary network utils pod. Enter into a bash session within it. In the terminal run the following command:
kubectl run --rm utils -it --image praqma/network-multitool -- bash
for i in {0..2}; do nslookup mongo-$i.db-service; done
#Note: This confirms that the DNS records have been created successfully and can be resolved within the cluster, 1 per MongoDB pod that exists behind the Headless Service - earlier created.
```

---

```bash
#On the mongo-0 pod, initialise the Mongo database Replica set. In the terminal run the following command:
kubectl exec -it mongo-0 -n voting-app -- mongosh

rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-0.db-service:27017" },
    { _id: 1, host: "mongo-1.db-service:27017" },
    { _id: 2, host: "mongo-2.db-service:27017" }
  ]
})

#Note: Wait until this command completes successfully, it typically takes 10-15 seconds to finish, and completes with the message: bye

#To confirm run this in the terminal:
kubectl exec -it mongo-0 -n voting-app -- mongosh --eval "rs.status()"

kubectl exec -it mongo-0 -n voting-app -- mongosh --eval "
rs.status().members.forEach(m => print(m.name + ' => ' + m.stateStr))
"


#Load the Data in the database by running this command:
cat << 'EOF' | kubectl exec -i mongo-0 -n voting-app -- mongosh
use langdb;

db.languages.insertOne({
  name: "csharp",
  codedetail: {
    usecase: "system, web, server-side",
    rank: 5,
    compiled: false,
    homepage: "https://dotnet.microsoft.com/learn/csharp",
    download: "https://dotnet.microsoft.com/download/",
    votes: 0
  }
});

db.languages.insertOne({
  name: "python",
  codedetail: {
    usecase: "system, web, server-side",
    rank: 3,
    script: false,
    homepage: "https://www.python.org/",
    download: "https://www.python.org/downloads/",
    votes: 0
  }
});

db.languages.insertOne({
  name: "javascript",
  codedetail: {
    usecase: "web, client-side",
    rank: 7,
    script: false,
    homepage: "https://en.wikipedia.org/wiki/JavaScript",
    download: "n/a",
    votes: 0
  }
});

db.languages.insertOne({
  name: "go",
  codedetail: {
    usecase: "system, web, server-side",
    rank: 12,
    compiled: true,
    homepage: "https://golang.org",
    download: "https://golang.org/dl/",
    votes: 0
  }
});

db.languages.insertOne({
  name: "java",
  codedetail: {        ← FIXED
    usecase: "system, web, server-side",
    rank: 1,
    compiled: true,
    homepage: "https://www.java.com/en/",
    download: "https://www.java.com/en/download/",
    votes: 0
  }
});

db.languages.insertOne({
  name: "nodejs",
  codedetail: {
    usecase: "system, web, server-side",
    rank: 20,
    script: false,
    homepage: "https://nodejs.org/en/",
    download: "https://nodejs.org/en/download/",
    votes: 0
  }
});

db.languages.find().pretty();
EOF


kubectl exec -it mongo-0 -n voting-app -- mongosh

use langdb

db.languages.find().pretty()

#Create Mongo secret:
kubectl apply -f mongo-secret.yaml

#Create GO API deployment by running the following command:
kubectl apply -f backend-deployment.yaml

#Expose API deployment through service using the following command:
kubectl expose deploy api \
 --name=api \
 --type=LoadBalancer \
 --port=80 \
 --target-port=8080

#or 

kubectl apply -f backend-service.yaml
```

```bash
API_ELB_PUBLIC_FQDN=$(kubectl get svc backend-service -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
until nslookup $API_ELB_PUBLIC_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
curl $API_ELB_PUBLIC_FQDN/ok
```

```bash
#Test and confirm that the API route URL /languages, and /languages/{name} endpoints can be called successfully. In the terminal run any of the following commands:
curl -s $API_ELB_PUBLIC_FQDN/languages | jq .
curl -s $API_ELB_PUBLIC_FQDN/languages/go | jq .
curl -s $API_ELB_PUBLIC_FQDN/languages/java | jq .
curl -s $API_ELB_PUBLIC_FQDN/languages/nodejs | jq .
```

```bash
kubectl apply -f frontend-deployment.yaml #Create the Frontend Deployment resource.

#Create a new Service resource of LoadBalancer type.
kubectl expose deploy frontend \
 --name=frontend \
 --type=LoadBalancer \
 --port=80 \
 --target-port=8080

#or 

kubectl apply -f frontend-service.yaml
```
```bash
FRONTEND_ELB_PUBLIC_FQDN=$(kubectl get svc frontend -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
until nslookup $FRONTEND_ELB_PUBLIC_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
curl -I $FRONTEND_ELB_PUBLIC_FQDN
```
```bash
#Query the MongoDB database directly to observe the updated vote data. 
kubectl exec -it mongo-0 -- mongo langdb --eval "db.languages.find().pretty()"
```
```bash
eksctl delete cluster \
  --name my-eks-cluster \
  --region ap-south-1
```

---

Policy for EC2 to access EKS Cluster:
```bash
{
	"Version": "2012-10-17",
	"Statement": [{
		"Effect": "Allow",
		"Action": [
			"eks:DescribeCluster",
			"eks:ListClusters",
			"eks:DescribeNodegroup",
			"eks:ListNodegroups",
			"eks:ListUpdates",
			"eks:AccessKubernetesApi"
		],
		"Resource": "*"
	}]
}
```