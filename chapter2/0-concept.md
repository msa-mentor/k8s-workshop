# Kubernetes Concept

![](../chapter1/img/1.png)


## 1. Concept 개요

쿠버네티스를 이용한다는 것을, API 서버를 이용해 쿠버네티스가 관리하는 오브젝트의 desired state를 기술하는 것이다. 구체적으로 어떤 종류의 애플리케이션 워크로드를 사용할 것인지, 어떤 컨테이너 이미지를 쓸것인지, Replicas(복제)의 수는 몇 개인지, 어떤 네트워크와 디스크 자원을 쓸 수 있도록 할 것인지 등을 기술한다. 

주로 `kubectl` CLI를 이용해서 쿠버네티스 API 서버에 desired state를 기술하고, desired state를 설정하면, *쿠버네티스 컨트롤 플레인* 은 Pod Lifecycle Event Generator (PLEG) 를 통해 클러스터의 현재 상태를 바라는 상태와 일치시킨다. 그렇게 함으로써, 쿠버네티스가 컨테이너를 시작 또는 재시작하거나, 주어진 애플리케이션의 복제 수를 스케일링하는 등의 다양한 작업을 자동으로 수행한다.

## 2. 쿠버네티스 오브젝트

쿠버네티스는 시스템의 상태를 나타내는 추상화한 개념들 다수 가지고 있다. 컨테이너화되어 배포된 애플리케이션과 워크로드, 이와 연관된 네트워크와 디스크 자원, 그 밖에 클러스터가 무엇을 하고 있는지에 대한 정보가 이에 해당한다. 이런 추상 개념은 쿠버네티스 API 내 오브젝트로 표현된다. 

기초적인 쿠버네티스 오브젝트에는 다음과 같은 것들이 있다.

* [파드](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)
* [서비스](https://kubernetes.io/docs/concepts/services-networking/service/)
* [볼륨](https://kubernetes.io/docs/concepts/storage/volumes/)
* [네임스페이스](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

그리고, 쿠버네티스에는 컨트롤러라는 concept이 있는데 쿠버네티스의 basic object를 활용해 다양한 기능을 제공한다. 

* [레플리카 셋](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
* [디플로이먼트](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
* [스테이트풀 셋](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
* [데몬 셋](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
* [잡](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/)



## 3.  쿠버네티스 오브젝트 이해하기

*쿠버네티스 오브젝트* 는 쿠버네티스 시스템내에 저장되는 엔티티이다. (상태값이 ETCD에 저장). 쿠버네티스는 클러스터의 상태를 나타내기 위해 이 엔티티를 이용한다. 

* 어떤 컨테이너화된 애플리케이션이 동작 중인지 (그리고 어느 노드에서 동작 중인지)
* 그 애플리케이션이 이용할 수 있는 리소스
* 그 애플리케이션의 재구동 정책, 업그레이드, 그리고 장애에 대해서 어떻게 동작해야 하는지에 대한 정책등


### 3.1 오브젝트 스펙(spec)과 상태(status)

모든 쿠버네티스 오브젝트는 오브젝트의 구성을 결정해주는 두 개의 오브젝트 필드를 포함하는데 오브젝트 *spec* 과 오브젝트 *status* 가 그것이다. 필히 제공되어야만 하는 *spec* 은, 오브젝트가 가졌으면 하고 원하는 특징, 즉 desired state를 기술한다. *status* 는 오브젝트의 *actual state* 를 기술하고, 쿠버네티스 시스템에 의해 제공되고 업데이트 된다. 쿠버네티스 컨트롤 플레인은 오브젝트의 실제 상태(actual state)를 여러분이 제시한 의도한 상태(desired state)에 일치시키기 위해 능동적으로 관리한다.


예를 들어, 쿠버네티스 디플로이먼트는 클러스터에서 동작하는 애플리케이션을 표현해 줄 수 있는 오브젝트이다. 디플로이먼트를 생성할 때, 디플로이먼트 spec에 3개의 애플리케이션 레플리카가 동작되도록 설정할 수 있다. 쿠버네티스 시스템은 그 디플로이먼트 spec을 읽어 spec에 일치되도록 상태를 업데이트하여 3개의 의도한 애플리케이션 인스턴스를 구동시킨다.


### 3.2 쿠버네티스 오브젝트 기술하기

쿠버네티스에서 오브젝트를 생성할 때, 오브젝트의 이름과 같은 기본적인 정보와 Desired state를 오브젝트 spec에 기술한다. 

`kubectl`을 이용해 YAML로 쿠버네티스 API를 호출하면 `kubectl`가 YAML을 JSON형식으로 정보를 변환해준다. 



쿠버네티스 디플로이먼트를 위한 `.yaml` 파일 예시
```yaml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

`kubectl` 이용해 적용해 보면 

```shell
kubectl apply -f https://k8s.io/examples/application/deployment.yaml --record
deployment.apps/nginx-deployment created
```


### 3.3 Required Fields

생성하고자 하는 쿠버네티스 오브젝트에 대한 `.yaml` 파일 내, 다음 필드를 위한 값들을 설정해 줘야한다.

* `apiVersion` - 이 오브젝트를 생성하기 위해 사용하고 있는 쿠버네티스 API 버전이 어떤 것인지
* `kind` - 어떤 종류의 오브젝트를 생성하고자 하는지
* `metadata` - `name` 문자열, `UID`, 그리고 선택적인 `namespace` 를 포함하여 오브젝트를 유일하게 구분지어 줄 데이터

그리고 오브젝트 `spec` 필드를 제공해야 한다. 오브젝트 `spec`에 대한 정확한 포맷은 모든 쿠버네티스 오브젝트마다 다르고, 그 오브젝트 특유의 필드를 포함한다. [Kubernetes API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/) 는 쿠버네티스를 이용하여 생성할 수 있는 오브젝트에 대한 모든 spec 포맷을 살펴볼 수 있도록 해준다. 

예를 들어, `파드`에 대한 `spec` 포맷은 [여기](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#podspec-v1-core)에서 확인할 수 있고, `디플로이먼트`에 대한 `spec` 포맷은 [여기](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#deploymentspec-v1-apps)에서 확인할 수 있다.


### 4. Basic Object 
간단하게 설명 하자면 Pod는 컨테이너화된 애플리케이션, Volume은 디스크, Service는 로드밸런서 그리고 Namespace는 패키지명 정도로 생각하면 된다. 그러면 각각을 자세하게 살펴보도록 한다.


#### 4.1 Pod
Pod 는 쿠버네티스에서 가장 기본적인 배포 단위로, 컨테이너를 포함하는 단위이다.

쿠버네티스의 특징중의 하나는 컨테이너를 개별적으로 하나씩 배포하는 것이 아니라 Pod 라는 단위로 배포하는데, Pod는 하나 이상의 컨테이너를 포함한다.

Pod 안에 한개 이상의 컨테이너를 가지고 있을 수 있다고 했는데 왜 개별적으로 하나씩 컨테이너를 배포하지 않고 여러개의 컨테이너를 Pod 단위로 묶어서 배포하는 것인가?

Pod는 다음과 같이 매우 재미있는 특징을 갖는다.

 - Pod 내의 컨테이너는 IP와 Port를 공유한다. 
두 개의 컨테이너가 하나의 Pod를 통해서 배포되었을때, localhost를 통해서 통신이 가능하다.
예를 들어 컨테이너 A가 8080, 컨테이너 B가 7001로 배포가 되었을 때, B에서 A를 호출할때는 localhost:8080 으로 호출하면 되고, 반대로 A에서 B를 호출할때에넌 localhost:7001로 호출하면 된다. 

 - Pod 내에 배포된 컨테이너간에는 디스크 볼륨을 공유할 수 있다. 
근래 애플리케이션들은 실행할때 애플리케이션만 올라가는것이 아니라 Reverse proxy, 로그 수집기등 다양한 주변 솔루션이 같이 배포 되는 경우가 많고, 특히 로그 수집기의 경우에는 애플리케이션 로그 파일을 읽어서 수집한다. 애플리케이션 (Tomcat, node.js)와 로그 수집기를 다른 컨테이너로 배포할 경우, 일반적인 경우에는 컨테이너에 의해서 파일 시스템이 분리되기 때문에, 로그 수집기가 애플리케이션이 배포된 컨테이너의 로그파일을 읽는 것이 불가능 하지만, 쿠버네티스의 경우 하나의 Pod 내에서는 컨테이너들끼리 볼륨을 공유할 수 있기 때문에 다른 컨테이너의 파일을 읽어올 수 있다.

#### 4.2. Volume
Pod가 기동할때 디폴트로, 컨테이너마다 로컬 디스크를 생성해서 기동되는데, 이 로컬 디스크의 경우에는 영구적이지 못하다. 즉 컨테이너가 리스타트 되거나 새로 배포될때 마다 로컬 디스크는 Pod 설정에 따라서 새롭게 정의되서 배포되기 때문에, 디스크에 기록된 내용이 유실된다. 

데이타 베이스와 같이 영구적으로 파일을 저장해야 하는 경우에는 컨테이너 리스타트에 상관 없이 파일을 영속적으로 저장애햐 하는데, 이러한 형태의 스토리지를 볼륨이라고 한다. 

볼륨은 컨테이너의 외장 디스크로 생각하면 된다. Pod가 기동할때 컨테이너에 마운트해서 사용한다.

앞에서 언급한것과 같이 쿠버네티스의 볼륨은 Pod내의 컨테이너간의 공유가 가능하다.

#### 4.3. Service
Pod와 볼륨을 이용하여, 컨테이너들을 정의한 후에, Pod 를 서비스로 제공할때, 일반적인 분산환경에서는 하나의 Pod로 서비스 하는 경우는 드물고, 여러개의 Pod를 서비스하면서, 이를 로드밸런서를 이용해서 하나의 IP와 포트로 묶어서 서비스를 제공한다.

Pod의 경우에는 동적으로 생성이 되고, 장애가 생기면 자동으로 리스타트 되면서 그 IP가 바뀌기 때문에, 로드밸런서에서 Pod의 목록을 지정할 때는 IP주소를 이용하는 것은 어렵다. 또한 오토 스케일링으로 인하여 Pod 가 동적으로 추가 또는 삭제되기 때문에, 이렇게 추가/삭제된 Pod 목록을 로드밸런서가 유연하게 선택해 줘야 한다. 


### 4.4. Namespace
네임스페이스는 한 쿠버네티스 클러스터내의 논리적인 분리단위라고 보면 된다.

Pod,Service 등은 네임 스페이스 별로 생성이나 관리가 될 수 있고, 사용자의 권한 역시 이 네임 스페이스 별로 나눠서 부여할 수 있다.

즉 하나의 클러스터 내에, 개발/운영/테스트 환경이 있을때, 클러스터를 개발/운영/테스트 3개의 네임 스페이스로 나눠서 운영할 수 있다. 네임스페이스로 할 수 있는 것은

 - 사용자별로 네임스페이스별 접근 권한을 다르게 운영할 수 있다.
 - 네임스페이스별로 리소스의 쿼타 (할당량)을 지정할 수 있다. 개발계에는 CPU 100, 운영계에는 CPU 400과 GPU 100개 식으로, 사용 가능한 리소스의 수를 지정할 수 있다. 
 - 네임 스페이스별로 리소스를 나눠서 관리할 수 있다. (Pod, Service 등)



주의할점은 네임 스페이스는 논리적인 분리 단위이지 물리적이나 기타 장치를 통해서 환경을 분리(Isolation)한것이 아니다. 다른 네임 스페이스간의 pod 라도 통신은 가능하다. 

물론 네트워크 정책을 이용하여, 네임 스페이스간의 통신을 막을 수 있지만 높은 수준의 분리 정책을 원하는 경우에는 쿠버네티스 클러스터 자체를 분리하는 것을 권장한다.


### 5. 라벨
 라벨은 쿠버네티스의 리소스를 선택하는데 사용이 된다. 각 리소스는 라벨을 가질 수 있고, 라벨 검색 조건에 따라서 특정 라벨을 가지고 있는 리소스만을 선택할 수 있다.

이렇게 라벨을 선택하여 특정 리소스만 배포하거나 업데이트할 수 있고 또는 라벨로 선택된 리소스만 Service에 연결하거나 특정 라벨로 선택된 리소스에만 네트워크 접근 권한을 부여하는 등의 행위를 할 수 있다. 

라벨은 metadata 섹션에 키/값 쌍으로 정의가 가능하며, 하나의 리소스에는 하나의 라벨이 아니라 여러 라벨을 동시에 적용할 수 있다.

```yaml
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```


셀렉터를 사용하는 방법은 오브젝트 스펙에서 selector 라고 정의하고 라벨 조건을 적어 놓으면 된다. 


### 6. 컨트롤러
앞에서 소개한 4개의 기본 오브젝트로, 애플리케이션을 설정하고 배포하는 것이 가능한데 이를 조금 더 편리하게 관리하기 위해서 쿠버네티스는 컨트롤러라는 개념을 사용한다.

컨트롤러는 기본 오브젝트들을 생성하고 이를 관리하는 역할을 해준다. 컨트롤러는 Replication Controller (aka RC), Replication Set, DaemonSet, Job, StatefulSet, Deployment 들이 있다. 각자의 개념에 대해서 살펴보도록 하자.

#### 6.1 Replication Controller
Replication Controller는  Pod를 관리해주는 역할을 하는데, 지정된 숫자로 Pod를 기동 시키고, 관리하는 역할을 한다. 

Replication Controller (이하 RC)는 크게 3가지 파트로 구성되는데, Replica의 수, Pod Selector, Pod Template 3가지로 구성된다.

 - Selector : 먼저 Pod selector는 라벨을 기반으로 하여,  RC가 관리한 Pod를 가지고 오는데 사용한다.

 - Replica 수 :  RC에 의해서 관리되는 Pod의 수인데, 그 숫자만큼 Pod 의 수를 유지하도록 한다.예를 들어 replica 수가 3이면, 3개의 Pod만 띄우도록 하고, 이보다 Pod가 모자르면 새로운 Pod를 띄우고, 이보다 숫자가 많으면 남는 Pod를 삭제한다.

 - Pod를 추가로 기동할 때 그러면 어떻게 Pod를 만들지 Pod에 대한 정보 (도커 이미지, 포트,라벨등)에 대한 정보가 필요한데, 이는 Pod template이라는 부분에 정의 한다.

#### 6.2 ReplicaSet
ReplicaSet은 Replication Controller 의 새버전으로 생각하면 된다.

큰 차이는 없고 Replication Controller 는 Equality 기반 Selector를 이용하는데 반해, Replica Set은 Set 기반의 Selector를 이용한다. 

#### 6.3 Deployment
Deployment (이하 디플로이먼트) Replication controller와 Replica Set의 좀더 상위 추상화 개념이다. 실제 운영에서는 ReplicaSet 이나 Replication Controller를 바로 사용하는 것보다, 좀 더 추상화된 Deployment를 사용하게 된다.




## Chapter 2 Contents
 - ### [1. POD](1-pod.md)
 - ### [2. Controller](2-controller.md)
 - ### [3. Service](3-service.md)
 - ### [4. Volume](4.volume.md)

### Reference 
 - [Kubernetes Concept](https://kubernetes.io/ko/docs/concepts/)
 - [조대협의 블로그](https://bcho.tistory.com/1256?category=731548)
