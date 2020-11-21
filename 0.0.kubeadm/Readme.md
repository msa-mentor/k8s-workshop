# 0. Single Master version without cloud controller manager

# kubeadm을 사용해 AWS EC2 Instance에서 kubernetes 클러스터 생성하기

> 1 단계 : 반복적으로 클러스터를 생성할 수 있도록, 명령어를 정리한다.
> 2 단계 : Cloudformation으로 자동 생성하도록 한다.

## VPC 만들기 
```bash

# VPC 현황 파악
aws ec2 describe-vpcs

# VPC 생성
export VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.1.0.0/16 \
  --output text \
  --query 'Vpc.VpcId')

echo ${VPC_ID}
vpc-********

# VPC Tagging
aws ec2 create-tags \
  --resource ${VPC_ID} \
  --tags Key=Name,Value=k8s-by-kubeadm

# VPC dns enable
aws ec2 modify-vpc-attribute \
  --vpc-id ${VPC_ID} \
  --enable-dns-hostnames '{"Value": true}'
```

### Subnet 만들기
```bash

export SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --availability-zone ap-northeast-2c \
  --cidr-block 10.1.1.0/24 \
  --output text --query 'Subnet.SubnetId')

echo ${SUBNET_ID}
subnet-**********

aws ec2 create-tags \
  --resources ${SUBNET_ID} \
  --tags Key=Name,Value=k8s
```

### Internet GateWay 생성

```bash
export INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway \
  --output text \
  --query 'InternetGateway.InternetGatewayId')

echo ${INTERNET_GATEWAY_ID}
igw-********


aws ec2 create-tags \
  --resources ${INTERNET_GATEWAY_ID} \
  --tags Key=Name,Value=k8s


# 게이트웨이 연결
aws ec2 attach-internet-gateway \
  --internet-gateway-id ${INTERNET_GATEWAY_ID} \
  --vpc-id ${VPC_ID}


# 라우트 테이블 가져오기
export ROUTE_TABLE_ID=$(aws ec2 describe-route-tables \
  --filters Name=vpc-id,Values=${VPC_ID} \
  --output text \
  --query 'RouteTables[0].RouteTableId')

echo ${ROUTE_TABLE_ID}
rtb-*************

aws ec2 create-tags \
    --resources ${ROUTE_TABLE_ID} \
    --tags Key=Name,Value=k8s


aws ec2 associate-route-table \
  --route-table-id ${ROUTE_TABLE_ID} \
  --subnet-id ${SUBNET_ID}

aws ec2 create-route \
  --route-table-id ${ROUTE_TABLE_ID} \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id ${INTERNET_GATEWAY_ID}


# 보안그룹 설정
export SECURITY_GROUP_ID=$(aws ec2 describe-security-groups \
  --filters Name=vpc-id,Values=${VPC_ID} \
  --output text \
  --query 'SecurityGroups[0].GroupId')



echo ${SECURITY_GROUP_ID}
sg-************

aws ec2 create-tags \
  --resources ${SECURITY_GROUP_ID} \
  --tags Key=Name,Value=k8s

# SSH
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0;

# HTTPS
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port 6443 \
  --cidr 0.0.0.0/0
```

### Master Node 생성

```bash
#Ubuntu 18.04
export IMAGE_ID="ami-0e67aff698cb24c1d"

# ssh 용 key pair 생성
aws ec2 create-key-pair \
  --key-name k8s \
  --output text \
  --query 'KeyMaterial' > k8s.id_rsa

chmod 400 k8s.id_rsa

# Master용 EC2 생성
# c5.xlarge, hands on 용으로는  t3a.large(2vcpu), t3a.xlarge(4vcpu)
export MASTER_INSTANCE_ID=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name k8s \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type c5.xlarge \
    --private-ip-address 10.1.1.10 \
    --user-data "name=master" \
    --subnet-id ${SUBNET_ID} \
    --output text --query 'Instances[].InstanceId')


aws ec2 create-tags \
  --resources ${MASTER_INSTANCE_ID} \
  --tags "Key=Name,Value=k8s-master"



# Worker Node용 EC2 생성
export WORKER_INSTANCE_ID_1=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name k8s \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type c5.xlarge \
    --private-ip-address 10.1.1.20 \
    --user-data "name=worker1" \
    --subnet-id ${SUBNET_ID} \
    --output text --query 'Instances[].InstanceId') 

aws ec2 create-tags \
  --resources ${WORKER_INSTANCE_ID_1} \
  --tags "Key=Name,Value=k8s-worker01"

export WORKER_INSTANCE_ID_2=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name k8s \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type c5.xlarge \
    --private-ip-address 10.1.1.30 \
    --user-data "name=worker2" \
    --subnet-id ${SUBNET_ID} \
    --output text --query 'Instances[].InstanceId') 

aws ec2 create-tags \
  --resources ${WORKER_INSTANCE_ID_2} \
  --tags "Key=Name,Value=k8s-worker02"
```





# 1. Kubeadm 설치 (Master, Worker 공통)
- 참조: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
## 1.1 네트워크 설정 체크 
```bash
# MAC Address check & product_uuid
ip link
sudo cat /sys/class/dmi/id/product_uuid

#br_netfilter 모듈이 로딩된것 확인
$ lsmod | grep br_netfilter
br_netfilter           28672  0
bridge                176128  1 br_netfilter
#위의 명령에서 아무런 내용이 없으면 아래 명령어를 실행한다. 
sudo modprobe br_netfilter

# Node 의 iptable이 bridge traffice을 바라볼 수 있는지 확인
# net.bridge.bridge-nf-call-iptables 가 1로 설정되어야 함
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system


```

## 1.2 Install Docker

```bash
# install docker : https://docs.docker.com/engine/install/ubuntu/

## 1. Set Up the repository
sudo apt-get update

sudo apt-get install \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg-agent \
  software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo apt-key fingerprint 0EBFCD88

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

## 2. install docker
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io

## 3. add user to docker group
sudo usermod -aG docker $USER

## 4. remember to log out and back in for this to take effect
## 로그아웃 했다가 다시 접속한다. 

```

## 1.3 kubeadm, kubelet, kubectl 설치

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```



# 2. Mater Node 구성

## 2.1 kubeadm init으로 Master Node 구성
```bash
# --apiserver-advertise-address : The IP address the API Server will advertise it's listening on. If not set the default network interface will be used
# --pod-network-cidr : Cluster IP 대역, Flannel 은 10.244.0.0/16, Calico : 192.168.0.0/16
# --service-cidr : Use alternative range of IP address for service VIPs. (default "10.96.0.0/12")
# --apiserver-cert-extra-sans : TLS 인증서에 적용할 IP 또는 도메인으로 kubectl 로 접근하는 kube-apiserver 의 IP 를 추가
sudo kubeadm init \
    --pod-network-cidr=10.244.0.0/16 \
    --apiserver-cert-extra-sans=10.1.1.10,3.***.***.32

# Tips :  --v=5 라고 하면 log level 이 올라가서 쉽게 진행사항, debugging할 수 있다.
```

## 2.2 kubectl config 설정
```bash
# 설치에 성공하면 아래와 같은 성공 메세지가 나온다. 
Your Kubernetes control-plane has initialized successfully!


# 1) Master에  kubectl 설정
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 2) Master Node에 Network Addon 설치 : Flannel 설치
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# 3) 각 Worker Node에서 join 할때 사용한 명령어 
sudo kubeadm join 10.1.1.10:6443 --token wcgrxw.i5cr9x2ew87admvt \
    --discovery-token-ca-cert-hash sha256:1ad6ba264644314677431368c269489ef71ee3d409c284154470ed528ce2bd45 

```


# 3. Worker Node 구성
 - 위의 ***1. Kubeadm 설치 (Master, Worker 공통*** 를 따라서 설정
```bash
#접속
ssh -i k8s.id_rsa ubuntu@13.125.237.228

# 1. Kubeadm 설치 (Master, Worker 공통) 를 따라서 설정

# 위의 과정이 끝나면 아래 명령어로 클러스터에 조인한다. 
sudo kubeadm join 10.1.1.10:6443 --token wcgrxw.i5cr9x2ew87admvt \
    --discovery-token-ca-cert-hash sha256:1ad6ba264644314677431368c269489ef71ee3d409c284154470ed528ce2bd45 








```



### Firewall   


### Check required ports 

***Control-plane node(s)***
|Protocol|Direction|Port Range|Purpose|Used By|
|--------|---------|---------|-------|-------|
|TCP|Inbound|6443*|Kubernetes API server|All|
|TCP|Inbound|2379-2380|etcd server client API	|kube-apiserver, etcd|
|TCP|Inbound|	10250|	Kubelet API	|Self, Control plane|
|TCP|Inbound|	10251|	kube-scheduler	|Self|
|TCP|Inbound|	10252|	kube-controller-manager|	Self|


***Worker node(s)***
|Protocol|	Direction|	Port Range|	Purpose|	Used By|
|--------|-----------|------------|---------|---------|
|TCP	|Inbound	|10250	|Kubelet API	|Self, Control plane|
|TCP	|Inbound	|30000-32767	|NodePort Services†|	All|

# Ubuntu의 경우는 기본 firewall이 ufw(Uncomplicated Firewall) 이므로 
```bash
sudo ufw enable

sudo ufw allow proto tcp 6443,2379,2380,10250,10251,10252



```



## 참조 
 - [Installing kubeadm-official Doc](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
 - [Setup a Kubernetes Cluster on AWS EC2 Instance with Ubuntu using kubeadm](https://www.howtoforge.com/setup-a-kubernetes-cluster-on-aws-ec2-instance-ubuntu-using-kubeadm/)
 - [AWS EC2에서 kubeadm으로 쿠버네티스 클러스터 만들기 - (1) AWS 인프라 구축](https://jonnung.dev/kubernetes/2020/03/01/create-kubernetes-cluster-using-kubeadm-on-aws-ec2/)
