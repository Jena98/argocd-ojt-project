## eks cluster create

``` bash
eksctl create cluster -f eks-ops.yaml
eksctl create cluster -f eks-svc.yaml
```

## ServiceAccount 생성

> aws-load-balancer-controller, external-dns-eks-ops 설치에 필요한 sa 를 생성해야 합니다. 
> policy 와 role 을 아래 url 을 참고해서 생성한 다음 eksctl create iamserviceaccount 를 해야합니다. 

* https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html
* https://www.stacksimplify.com/aws-eks/aws-alb-ingress/install-externaldns-on-aws-eks/

``` bash
# eks-ops aws-load-balancer-controller sa
eksctl create iamserviceaccount \
    --cluster=eks-ops \
    --namespace=my-addons \
    --name=aws-load-balancer-controller \
    --role-name aws-load-balancer-controller-eks-ops \
    --attach-policy-arn=arn:aws:iam::{ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve

# eks-svc aws-load-balancer-controller sa
eksctl create iamserviceaccount \
    --cluster=eks-svc \
    --namespace=my-addons \
    --name=aws-load-balancer-controller \
    --role-name aws-load-balancer-controller-eks-svc \
    --attach-policy-arn=arn:aws:iam::{ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve

# eks-ops external-dns sa
eksctl create iamserviceaccount \
    --name external-dns \
    --role-name external-dns-eks-ops \
    --namespace my-addons \
    --cluster eks-ops \
    --attach-policy-arn arn:aws:iam::{ACCOUNT_ID}:policy/AllowExternalDNSUpdates \
    --approve \
    --override-existing-serviceaccounts

# eks-svc external-dns sa
eksctl create iamserviceaccount \
    --name external-dns \
    --role-name external-dns-eks-svc \
    --namespace my-addons \
    --cluster eks-svc \
    --attach-policy-arn arn:aws:iam::{ACCOUNT_ID}:policy/AllowExternalDNSUpdates \
    --approve \
    --override-existing-serviceaccounts    
```

