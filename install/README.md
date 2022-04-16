# ArgoCD

* <https://argo-cd.readthedocs.io/en/stable/getting_started/>
* <https://argocd-applicationset.readthedocs.io/en/stable/Getting-Started/>

## Create eks cluster (Create sa)

* <https://github.com/jenana-devops/argocd-ojt-project/tree/master/eksctl>

## Prerequisites
- [awscli v2](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/install-cliv2.html)
- [kubectl](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/install-kubectl.html)
- [jq](https://stedolan.github.io/jq/download/)
- [helm v3](https://helm.sh/ko/docs/intro/install/)
- [htpasswd](https://command-not-found.com/htpasswd)

## Generate values.yaml

1. external-secret 으로 ssm 을 사용하기 위해서 파라미터 스토어에 시크릿을 저장합니다.
2. github oauth 를 사용합니다. ex) jenana-devops 조직의 psa 팀
3. argocd-notifications 를 사용하기 위해서 slack-token 을 발급 받습니다.
4. argocd.example.com 으로 acm 을 발급 받습니다.

```bash
#####################
# variables
#####################
export GITHUB_ORG="jenana-devops"
export GITHUB_TEAM="psa"

export ADMIN_PASSWORD="REPLACE_ME" # random string
export ARGOCD_PASSWORD="$(htpasswd -nbBC 10 "" ${ADMIN_PASSWORD} | tr -d ':\n' | sed 's/$2y/$2a/')" 
export ARGOCD_SERVER_SECRET="REPLACE_ME" # random string

export ARGOCD_NOTI_TOKEN="REPLACE_ME" # <https://api.slack.com/apps>

export ARGOCD_GITHUB_ID="REPLACE_ME" # github OAuth Apps <https://github.com/organizations/opspresso/settings/applications>
export ARGOCD_GITHUB_SECRET="REPLACE_ME" # github OAuth Apps

export ARGOCD_HOSTNAME="REPLACE_ME" # argocd.example.com
export AWS_ACM_CERT="$(aws acm list-certificates --query "CertificateSummaryList[].{CertificateArn:CertificateArn,DomainName:DomainName}[?contains(DomainName,'${ARGOCD_HOSTNAME}')] | [0].CertificateArn" | jq . -r)"

########################################
# put aws ssm parameter store
########################################
aws ssm put-parameter --name /k8s/common/admin-password --value "${ADMIN_PASSWORD}" --type SecureString --overwrite | jq .
aws ssm putparameter --name /k8s/common/argocd-password --value "${ARGOCD_PASSWORD}" --type SecureString --overwrite | jq .
aws ssm put-parameter --name /k8s/common/argocd-server-secret --value "${ARGOCD_SERVER_SECRET}" --type SecureString --overwrite | jq .
aws ssm put-parameter --name /k8s/common/argocd-noti-token --value "${ARGOCD_NOTI_TOKEN}" --type SecureString --overwrite | jq .
aws ssm put-parameter --name /k8s/${GITHUB_ORG}/argocd-github-id --value "${ARGOCD_GITHUB_ID}" --type SecureString --overwrite | jq .
aws ssm put-parameter --name /k8s/${GITHUB_ORG}/argocd-github-secret --value "${ARGOCD_GITHUB_SECRET}" --type SecureString --overwrite | jq .

#########################################
# get aws ssm parameter store
#########################################
export ADMIN_PASSWORD=$(aws ssm get-parameter --name /k8s/common/admin-password --with-decryption | jq .Parameter.Value -r)
export ARGOCD_PASSWORD=$(aws ssm get-parameter --name /k8s/common/argocd-password --with-decryption | jq .Parameter.Value -r)
export ARGOCD_SERVER_SECRET=$(aws ssm get-parameter --name /k8s/common/argocd-server-secret --with-decryption | jq .Parameter.Value -r)
export ARGOCD_GITHUB_ID=$(aws ssm get-parameter --name /k8s/${GITHUB_ORG}/argocd-github-id --with-decryption | jq .Parameter.Value -r)
export ARGOCD_GITHUB_SECRET=$(aws ssm get-parameter --name /k8s/${GITHUB_ORG}/argocd-github-secret --with-decryption | jq .Parameter.Value -r)
export ARGOCD_NOTI_TOKEN=$(aws ssm get-parameter --name /k8s/common/argocd-noti-token --with-decryption | jq .Parameter.Value -r)

#######################################
# replace values.yaml 
#######################################
cp values.yaml values.output.yaml
sed -i "s/AWS_ACM_CERT/${AWS_ACM_CERT}/g" values.output.yaml
sed -i "s/ARGOCD_HOSTNAME/${ARGOCD_HOSTNAME}/g" values.output.yaml
sed -i "s@ARGOCD_PASSWORD@${ARGOCD_PASSWORD}@g" values.output.yaml
sed -i "s/ARGOCD_SERVER_SECRET/${ARGOCD_SERVER_SECRET}/g" values.output.yaml
sed -i "s/ARGOCD_GITHUB_ID/${ARGOCD_GITHUB_ID}/g" values.output.yaml
sed -i "s/ARGOCD_GITHUB_SECRET/${ARGOCD_GITHUB_SECRET}/g" values.output.yaml
sed -i "s/GITHUB_ORG/${GITHUB_ORG}/g" values.output.yaml
sed -i "s/GITHUB_TEAM/${GITHUB_TEAM}/g" values.output.yaml
```

## Install ArgoCD

> argocd 설치 후 server 의 elb 가 생성되면 route53 의 argocd 도메인과 연결해주세요. 

```bash
helm repo add https://argoproj.github.io/argo-helm 
helm repo update
helm search repo argo-cd

helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace -f values.output.yaml
```
* Route 53 <https://us-east-1.console.aws.amazon.com/route53/v2/hostedzones#>

## Install ArgoCD CLI

```bash
sudo curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.0.4/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argocd
```

## argocd login 

> argocd cli 에 login 합니다.

```bash
argocd login {ARGOCD_HOSTNAME} --grpc-web --username admin --password {ADMIN_PASSWORD} --insecure
```

## argocd cluster add

> argocd 에 cluster 를 등록시킵니다. context 와 name 이 알맞는지 확인해 주세요.

```bash
CONTEXT_NAME=`kubectl config view -o jsonpath='{.contexts[0].name}'`
argocd cluster add $CONTEXT_NAME --name eks-ops

CONTEXT_NAME=`kubectl config view -o jsonpath='{.contexts[1].name}'`
argocd cluster add $CONTEXT_NAME --name eks-svc

argocd cluster list
```

## addons

> addons 를 설치합니다.

``` bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/jenana-devops/argocd-ojt-project/master/addons.yaml
```

## apps

> apps 를 설치합니다.

``` bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/jenana-devops/argocd-ojt-project/master/apps.yaml
```

