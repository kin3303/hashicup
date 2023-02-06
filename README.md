
## Sample Consul Chart
 
- Hashicup Application

### Step 1. 배포에 필요한 도구 설치

- Helm 을 테스트 해보기 위해서는 미리 배포된 Kubernetes Cluster 가 필요하다.
- Cluster 와 연결할 클라이언트는 아래와 같이 Docker, kubectl, helm , helm push plugin, yamllint 를 설치해 놓아야 한다.
   - 아래는 ec2 아마존 리눅스 2 인스턴스에 이와 같은 도구를 설치하는다.
   - Helm으로 Harbor에 Chart를 Push하기 위해서는 helm push plugin을 설치해야 push 명령어를 사용할 수 있다.
- k8s 클러스터에는 Consul, Prometheus, Grafana, Jaeger 를 미리 설치해 놓아야 한다.

```console 
$  private_ip=$( curl -Ss -H "X-aws-ec2-metadata-token: $imds_token" 169.254.169.254/latest/meta-data/local-ipv4 )
$  sudo yum update -y
$  sudo yum -y install wget tar

######################################################################
# Install docker
######################################################################
$  sudo amazon-linux-extras install -y docker
$  sudo tee  /etc/docker/daemon.json > /dev/null <<EOF
{
"insecure-registries" : ["$private_ip","0.0.0.0"]
}
EOF
$  sudo service docker start
$  sudo usermod -a -G docker ec2-user
$  sudo chmod 666 /var/run/docker.sock

######################################################################
# Install docker compose
######################################################################
$  sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$  sudo chmod +x /usr/local/bin/docker-compose
$  sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

######################################################################
# Install Kubectl
######################################################################
$  sudo curl -L "https://dl.k8s.io/release/v1.23.6/bin/linux/amd64/kubectl" -o /usr/local/bin/kubectl
$  sudo chmod +x /usr/local/bin/kubectl
$  sudo ln -s /usr/local/bin/kubectl /usr/bin/kubectl

######################################################################
# Install Helm
###################################################################### 
$  curl -L https://git.io/get_helm.sh | bash -s -- --version v3.8.2

######################################################################
# Install helm push plugin
######################################################################
$  sudo yum -y install git
$  helm plugin install https://github.com/chartmuseum/helm-push

######################################################################
# Install yamlint 
######################################################################
$  sudo yum install -y python3-pip
$  sudo pip3 install yamllint
$  yamllint --version

######################################################################
# Download Sources
######################################################################
$  git clone https://github.com/kin3303/hashicup.git
```

### Step 2. Cluster 와 연결

```console
$  aws configure
$  aws eks --region ap-northeast-2 update-kubeconfig --name <CLUSTER_NAME>
$  chmod 600  ~/.kube/config
```

### Step 3. Chart 정적 테스트

- helm lint 명령을 통해 Chart 에 이상이 없는지 검사 

```console
$ ls
hashicup
$  helm lint hashicup 
==> Linting hashicup
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

- yaml lint 명령을 통해 Chart Yaml 에 이상이 없는지 검사

```console
######################################################################
# yamllint 설정
######################################################################
$ sudo tee  .yamllint > /dev/null <<EOF
extends: default
rules:
  line-length:
    max: 150
  trailing-spaces:
    level: warning
  key-duplicates:
    level: error
EOF

######################################################################
# yamllint 실행
######################################################################
$ helm template my-hashicup hashicup --namespace plateer  | yamllint -
  41:18     warning  trailing spaces  (trailing-spaces)
  271:1     warning  trailing spaces  (trailing-spaces)
  341:1     warning  trailing spaces  (trailing-spaces)
  450:1     warning  trailing spaces  (trailing-spaces)
  456:1     warning  trailing spaces  (trailing-spaces)
  468:1     warning  trailing spaces  (trailing-spaces)
  600:1     warning  trailing spaces  (trailing-spaces)
  606:1     warning  trailing spaces  (trailing-spaces)
  618:1     warning  trailing spaces  (trailing-spaces)
$ cat -n <(helm template my-hashicup hashicup --namespace plateer)
``` 

### Step 4. Chart 설치 및 테스트

```console
 
######################################################################
# 차트 설치
######################################################################
$ kubectl create ns plateer
$ helm install my-hashicup hashicup/ --version 0.1.0 --namespace plateer  --wait
$ kubectl get all -n plateer
$ kubectl port-forward service/nginx -n plateer  8080:80 --address 0.0.0.0
http://<호스트_IP>:8080 으로 접속하여 확인
 

######################################################################
# 리소스 정리
######################################################################
$ helm uninstall my-hashicup -n plateer
$ kubectl delete ns plateer
```
