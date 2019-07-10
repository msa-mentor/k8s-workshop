### Pod


### Step 1. Pod YAML Description 살펴보기 

$ kubectl get pod <pod-name> -o yaml
page 62


### Step 2. YAML 파일 생성하기

각 영역 설명 추가 

### Step 3 explain을 이용한 pod 정보 보기 

$ kubectl explain pod.spec


### Step 4. Portforwd
$ kubectl port-forward kubia-manual 8888:8080


## Pods Label 

## NameSpace

## Stopping and Removing pod