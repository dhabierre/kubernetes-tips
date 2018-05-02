Kubernetes on AWS with Kops
# Install Kubernetes on AWS with KOPS

## Inputs

- [Kops - Kubernetes Operations](https://github.com/kubernetes/kops)
- [Installing Kubernetes on AWS with Kops](https://kubernetes.io/docs/getting-started-guides/kops/)
- [AWS Workshop for Kubernetes](https://github.com/aws-samples/aws-workshop-for-kubernetes)
- [Install the Windows Subsystem for Linux
](https://docs.microsoft.com/en-us/windows/wsl/install-win10) (if using Windows 10)

## Kubectl Installation

```sh
apt-get update && apt-get install -y apt-transport-https

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

apt-get update
apt-get install kubectl
```

## Kops Installation

```sh
# get latest release here: https://github.com/kubernetes/kops/releases
wget https://github.com/kubernetes/kops/releases/download/1.9.0/kops-linux-amd64

chmod +x kops-linux-amd64

mv kops-linux-amd64 /usr/local/bin/kops
```

## AWS CLI Installation & Configuration

### Installation

```sh
pip -V || sudo apt-get install python-pip

aws --version || pip install awscli --upgrade --user

export PATH=~/.local/bin:$PATH

complete -C '$(which aws_completer)' aws
```

### Configuration

```sh
aws configure
AWS Access Key ID [None]: xxxx
AWS Secret Access Key [None]: xxxx
Default region name [None]: eu-west-1
Default output format [None]: json

export AWS_PROFILE=default
```

## Route53 domain

Here we prefere to use [Gossip Protocol](https://en.wikipedia.org/wiki/Gossip_protocol).

Report to [this page](https://github.com/aws-samples/aws-workshop-for-kubernetes/tree/master/01-path-basics/102-your-first-cluster) to know more about DNS-based.

## S3 bucket

The S3 bucket is used to store the cluster configuration.

```sh
# configure Kops AZ

export AWS_AVAILABILITY_ZONES="$(aws ec2 describe-availability-zones --query 'AvailabilityZones[].ZoneName' --output text | awk -v OFS="," '$1=$1')"

echo $AWS_AVAILABILITY_ZONES

# create the S3 bucket
# the bucket name must be unique otherwise you will encounter an error on deployment.
# we will use an example bucket name of kops-state-store- and add a randomly generated string to the end.

export S3_BUCKET=kops-state-store-$(cat /dev/urandom | LC_ALL=C tr -dc "[:alpha:]" | tr '[:upper:]' '[:lower:]' | head -c 32)
export KOPS_STATE_STORE=s3://${S3_BUCKET}

echo $S3_BUCKET
echo $KOPS_STATE_STORE

aws s3 mb $KOPS_STATE_STORE
aws s3api put-bucket-versioning --bucket $S3_BUCKET --versioning-configuration Status=Enabled
```

## Build Cluster Configuration

Kops 1.6.2 adds an experimental support for [gossip-based discovery](https://github.com/aws-samples/aws-workshop-for-kubernetes/tree/master/01-path-basics/102-your-first-cluster) of nodes. This makes the process of setting up Kubernetes cluster using Kops DNS-free, and much more simplified.

```sh
ssh -V || apt-get install -y ssh

# create default ssh key (press ENTER for each choice)
ssh-keygen -t rsa -b 4096 -C "your@email.com"

# create the cluster in private VPC
kops create cluster \
  --name dha.cluster.k8s.local \
  --master-count 3 \
  --node-count 5 \
  --zones $AWS_AVAILABILITY_ZONES \
  --node-size t2.medium \
  --master-size t2.medium \
  --topology private \
  --networking-cidr 10.20.0.0/16
  --networking kube-router

kops get cluster

kops edit cluster dha.cluster.k8s.local

kops edit ig --name=dha.cluster.k8s.local nodes

# edit the configuration of a giver master
kops edit ig --name=dha.cluster.k8s.local master-eu-west-1a
```

## Create Cluster

```sh
kops update cluster dha.cluster.k8s.local --yes
kops validate cluster

# get the DNS of the Kubernetes API
aws elb describe-load-balancers --query 'LoadBalancerDescriptions[*].DNSName'
```

## Kubernetes Dashboard Installation

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

Get the admin password to connect to the Dashboard using the **token** method.


```sh
kops get secrets admin --type secret -oplaintext
```

## Install Nginx Ingress Controller

[More information](https://github.com/aws-samples/aws-workshop-for-kubernetes/tree/master/04-path-security-and-networking/405-ingress-controllers)

```sh
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml | kubectl apply -f -
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/default-backend.yaml | kubectl apply -f -
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/configmap.yaml | kubectl apply -f -
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/tcp-services-configmap.yaml | kubectl apply -f -
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/udp-services-configmap.yaml | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/rbac.yaml | kubectl apply -f -
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/with-rbac.yaml | kubectl apply -f -

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/aws/service-l4.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/aws/patch-configmap-l4.yaml
```

## Delete Cluster Resources

```sh
kops delete cluster dha.cluster.k8s.local --yes
```