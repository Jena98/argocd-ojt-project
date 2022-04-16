## 설치 순서 
1. eks cluster 2대 설치 (eks-ops, eks-svc) 후 sa 생성
    * eksctl 폴더 

2. argocd install 
    * install 폴더 

3. addons 설치 
    * addons.yaml 이 로컬에 없을땐 두번째 명령어를 실행

``` bash
kubectl apply -f addons.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/jenana-devops/argocd-ojt-project/master/addons.yaml
```

4. apps 설치
    * apps.yaml 이 로컬에 없을땐 두번째 명령어를 실행

``` bash
kubectl apply -f apps.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/jenana-devops/argocd-ojt-project/master/apps.yaml
```