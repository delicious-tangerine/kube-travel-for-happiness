# `고급 스케줄링`

## 목차

<br>

---
---

<br>

## 서론

쿠버네티스는 파드가 스케줄링될 위치에 영향을 미칠 수 있게 한다.

노드 셀렉터 이외의 방법으로도!

<br>

---
---

<br>

## 테인트와 톨러레이션을 사용해 특정 노드에서 파드 실행 제한

테인트와 톨러레이션은 어떤 파드가 특정 노드를 사용할 수 있는지를 제한하고자 할 때 사용된다.

테인트는 기존의 파드를 수정하지 않고, 노드에 테인트를 추가하는 것만으로도 파드가 특정 노드에 배포되지 않도록 한다.


**정리**
- 노드 어피니티 : 파드에 정보를 추가해 특정 노드에 스케줄링 혹은 스케줄링 X => 파드가 노드 결정
- 테인트 : 노드 어피니티와 반대로 노드가 파드를 제외
- 톨러레이션 : 파드에 적용되며 파드를 일치하는 테인트가 있는 노드에 스케줄 되게 함.

보통 테인트와 톨러레이션은 함께 작동하여 파드가 부적절한 노드에 스케줄되지 않도록 함.

<br>

---

<br>

### 테인트와 톨러레이션 소개

***노드의 테인트(Taint)***

마스터 노드는 테인트돼 있어 컨트롤 플레인 파드만 배치할 수 있다.

```bash
# Taints를 확인해보자
$ kubectl describe node master.k8s
```

테인트는 키(key), 값(value), 효과(effect)가 있고 <key>=<value>:<effect> 형태로 표시된다.

실제로 확인해 본 테인트는 파드가 이 테인트를 허용하지 않는 한 마스터 노드에 스케줄링 되지 못하게 막는 테인트다.

<br>

***파드의 톨러레이션(Toleration) 표시하기***

kubeadm으로 설치한 클러스터에는 마스터 노드를 포함해 모든 노드에서 kube-proxy 클러스터 컴포넌트가 파드로 실행된다.

파드로 실행되는 마스터 노드 컴포넌트도 쿠버네티스 서비스에 접근해야 할 수도 있기 때문에 kube-proxy가 마스터 노드에서도 실행된다.

kube-proxy 파드도 마스터 노드에서 실행되게 하려고 적절한 톨러레이션을 포함하고 있다.

```bash
# Toleration을 확인하자
$ kubectl describe po kube-proxy-80wqm -n kube-system
```

<br>

***테인트 효과***

첫 번째 톨러레이션은 마스터 노드의 테인트와 동일하다.

kube-proxy 파드의 다른 두 가지 톨러레이션은 준비되지 않았거나 도달할 수 없는 노드에서 파드를 얼마나 오래 실행할 수 있는지를 정의한다.

테인트는 세 가지 효과(Effect)가 가능하다.

- NoSchedule: 파드가 테인트를 허용하지 않는 경우 파드가 노드에 스케줄링 되지 않는다.
- PreferNoSchedule: NoSchedule의 소프트 버전으로, 스케줄러가 파드를 노드에 스케줄링하지 않으려 하지만 다른 곳에 스케줄링할 수 없으면 해당 노드에 스케줄링된다.
- NoExecute : 노드에서 이미 실행 중인 파드에도 영향을 주며, 이미 노드에서 실행 중인 파드도 제거할 수 있다.

<br>

---

<br>

### 노드에 사용자 정의 테인트 추가하기

프로덕션 노드에서 프로덕션이 아닌 파드가 실행되지 않도록 해보자.

```bash
$ kubectl taint node node1.k8s node-type=production:NoSchedule
```

실수로라도 파드를 프로덕션 노드에 배포하는 일은 없게 된다.

<br>

---

<br>

### 파드에 톨러레이션 추가

프로덕션 파드를 프로덕션 노드에 배포하려면 노드에 추가한 테인트의 톨러레이션이 필요하다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod
spec:
  replicas: 5
  template:
    spec:
      ...
      tolerations:
      - key: node-type
        # 책에 적혀있는 것처럼 대문자 O가 아니다.
        operator: Equal
        value: production
        effect: NoSchedule
```

<br>

---

<br>

### 테인트와 톨러레이션의 활용 방안 이해

노드는 하나 이상의 테인트를 가질 수 있으며, 파드는 하나 이상의 톨러레이션을 가질 수 있다.

지금까지 봤듯이, 테인트는 키(key)와 효과(effect)만 갖고 있고, 값(value)을 꼭 필요로 하지는 않는다.

톨러레이션은 Equal 연산자를 지정해 특정한 값을 허용하거나, Exists 연산자를 사용해 특정 테인트 키에 여러 값을 허용할 수 있다.

<br>

***스케줄링에 테인트와 톨러레이션 활용하기***

일부 노드에 특수한 하드웨어를 제공해 파드 중 일부만 이 하드웨어를 사용하게 하는 경우에도 테인트와 톨러레이션을 사용할 수 있다.

<br>

***노드 실패 후 파드를 재스케줄링하기까지의 시간 설정***

파드를 실행 중인 노드가 준비되지 않거나 도달할 수 없는 경우 톨러레이션을 사용해 쿠버네티스가 다른 노드로 파드를 다시 스케줄링하기 전에 대기해야 하는 시간을 지정할 수도 있다.

```bash
$ kubectl get po prod-XXXXXXX -o yaml
```

실제로 default 시간은 notReady, unreachable 상태에 관해 300초로 설정되어 있다.

<br>

---
---

<br>

## 노드 어피니티를 사용해 파드를 특정 노드로 유인하기

테인트는 파드를 특정 노드에서 떨어뜨려 놓는 데 사용된다.

쿠버네티스가 특정 노드 집합에만 파드를 스케줄링하도록 지시해보자.

***Node Affinity vs Node Selector***

노드 셀렉터와 유사하게 각 파드는 노드 어피티니를 정의할 수 있다.

꼭 지켜야 하는 필수 요구 사항이나 선호도를 지정할 수 있다.

<br>

***디폴트 노드 레이블 검사***

노드 어피니티는 노드 셀렉터와 같은 방식으로 레이블을 기반으로 노드를 선택한다.

노드에 여러 레이블이 있지만 GKE에서 살펴보면 이런 중요한 레이블들이 있다.

- failure-domain.beta.kubernetes.io/region : 노드가 위치한 지리적 리전 지정
- failure-domain.beta.kubernetes.io/zone : 노드가 있는 가용 영역을 지정한다.
- kubernetes.io/hostname : 노드의 호스트 이름

<br>

---

<br>

### 하드 노드 어피티니 규칙 지정

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: gpu
                operator: In
                values:
                  - "true"
```

훨씬 복잡해 보이지만 표현이 풍부하다.

- requiredDuringScheduling : 이 필드 아래 정의된 규칙은 파드가 노드로 스케줄링되고자 하는 레이블을 지정한다.
- IgnoredDuringExecution : 이 필드 아래에 정의된 규칙은 노드에서 이미 실행 중인 파드에는 영향을 미치지 않는다.

어피니티가 파드 스케줄링에만 영향을 미치며, 파드가 노드에서 제거되지 않음을 기억하자.

<br>

***nodeSelectorTerm***

nodeSelectorTerms와 matchExpressions 필드는 파드를 노드에 스케줄링하고자 노드의 레이블이 일치하도록 표현식을 정의하는 데 사용된다.

<img src="1">

<br>

---

<br>

### 파드의 스케줄링 시점에 노드 우선순위 지정

preferredDuringSchedulingIgnoredDuringExecution 필드를 통해 선호할 노드를 지정할 수 있다.

예시로 특정 가용 영역에 스케줄링 원하는 파드를 배포한다고 하자.

***노드 레이블링***

먼저 노드에 적절한 레이블을 지정해야 한다.

각 노드에는 노드가 속한 가용 영역을 지정하는 레이블과 이를 전용 또는 공유 노드로 표시하는 레이블이 있어야 한다.

```yaml
$ kubectl lable node node1.k8s availability-zone=zone1

$ kubectl lable node node1.k8s share-type=dedicated

$ kubectl lable node node2.k8s availability-zone=zone2

$ kubectl lable node node2.k8s share-type=shared
```

<br>

***선호하는 노드 어피니티 규칙 지정***

```yaml
# zone1의 dedicated 노드를 선호하는 디플로이먼트를 생성하자.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pref
spec:
  template:
    ...
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 80
              preference:
                matchExpressions:
                  - key: availability-zone
                    operator: In
                    values:
                      - zone1
            - weight: 20
              preference:
                matchExpressions:
                  - key: share-type
                    operator: In
                    values:
                      - dedicated
...
```

<br>

***노드 선호도 작동 방법 이해하기***

Availability-zone과 share-type 레이블이 파드의 노드 어피니티와 일치하는 노드에 가장 높은 순위가 매겨진다.

파드가 노드 어피니티 규칙에 설정된 가중치에 따라 zone1의 dedicated 노드가 최상위 우선순위를 갖게 된다.

<br>

***노드가 두 개인 클러스터에 파드 배포하기***

노드 우선순위가 높더라도, 여러 개의 파드를 두 개의 노드가 있는 클러스터에 배포하게 되면 하나는 다른 노드에 배포될 수 있다.

Selector-SpreadPriority 기능 때문인데 파드를 여러 노드에 분산시켜 노드 장애로 인해 전체 서비스가 중단되지 않도록 하는 기능이다.

<br>

---
---

<br>

## 파드 어피니티와 안티-어피니티를 이용해 파드 함께 배치하기

때때로 파드 간의 어피니티를 지정할 필요가 있는 경우가 있다.

한 노드에 프론트와 백엔드 파드가 함께 배포되면 더 좋지 않을까? 와 같은 경우이다.

<br>

---

<br>

### 파드 간 어피니티를 사용해 같은 노드에 파드 배포하기

```bash
$ kubectl run backend -l app=backend --image busybox -- sleep 999999
```

```yaml
# frontend를 배포해보자
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 5
  template:
  ...
    spec:
      affinity:
        podAffinity:
          # 필수 요구 사항 정의
          requiredDuringSchedulingIgnoredDuringExecution:
            - topologyKey: kubernetes.io/hostname
              labelSelector:
                matchLabels:
                  app: backend
```

<img src="2">

<br>

***파드 어피니티를 갖는 파드 배포***

```bash
$ kubectl create -f frontend-podaffinity-host.yaml

$ kubectl get po -o wide
```

모든 프론트엔드 파드는 실제로 백엔드 파드와 동일한 노드로 스케줄링 됐다!

프론트엔드 파드를 스케줄링할 때 스케줄러는 먼저 프론트엔드 파드의 podAffinity 설정에 정의된 labelSelector와 일치하는 모든 파드를 찾은 다음 프론트엔드 파드를 동일한 노드에 스케줄링했다.

<br>

***스케줄러가 파드 어피니티 규칙을 사용하는 방법 이해***

만약 백엔드 파드가 실수로 삭제돼서 다른 노드로 다시 스케줄링된다면,

프론트엔드 파드의 어피니티 규칙이 깨지기 떄문에 원래 노드에 스케줄링된다.

<br>

---

<br>

### 동일한 랙, 가용 영역 또는 리전에 파드 배포

***동일 가용 영역에 파드 배포하기***

노드가 서로 다른 가용 영역에 있는 경우 백엔드 파드와 동일한 가용 영역에서 프론트엔드 파트를 실행하려면

topologyKey 속성을 failure-domain.beta.kuberenetes.io/zone으로 변경하면 된다.

<br>

***같은 리전에 파드를 함께 배포하기***

동일한 리전에 배포하려면, topologyKey를 failure-domain.beta.kubernetes.io/region으로 설정해야 한다.

<br>

***topologyKey 작동 방법 이해***

topologyKey가 작동하는 방법은 간단하다.

podAffinity에 지정된 topologyKey 필드와 일치하는 키를 갖는 노드 레이블을 찾는다.

그런 다음 레이블이 이전에 찾은 파드의 값과 일치하는 모든 노드를 선택한다.

<br>

---

<br>

### 필수 요구 사항 대신 파드 어피니티 선호도 표현하기

podAffinity는 nodeAffinity랑 동일하다.

동일한 노드에 스케줄링되고 싶지만 여의치 않으면 다른 곳에 스케줄링돼도 무방하다고 알려준다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 5
  template:
    ...
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector:
                  matchLabels:
                    app:backend       
              weight: 80
containers: ...
```

사실상 nodeAffinity랑 똑같다.

이 파드를 배포하면 백엔드 파드와 동일한 노드에 네 개의 파드를 배포하고 다른 노드에 하나를 배포하게 된다.

<br>

---

<br>

### 파드 안티-어피니티를 사용해 파드들이 서로 떨어지게 스케줄링하기

파드를 서로 멀리 떨어뜨려 놓고 싶을 수 있다.

이것을 파드 안티 어피니티라고 한다.

podAntiAffinity 속성을 사용한다는 점을 제외하고는 파드 어피니티 설정과 동일한 방식으로 지정한다.

<br>

---
---

<br>

## 사용자 정의 API 오브젝트

쿠버네티스가 제공하는 API 오브젝트 만을 살펴봤다.

현재 생태계가 발달함에 따라 다양한 오브젝트가 많이 나타나고 있다.

이런 사용자 정의 리소스를 추가하는 방법을 알아보자.

<br>

---

<br>

### CustomResourceDefinition 소개

CRD 오브젝트를 쿠버네티스 API 서버에 게시하면 된다.

각 CRD에는 일반적으로 관련 컨트롤러를 갖는다.

<br>

***CRD 예제***

쿠버네티스 클러스터에서 최대한 쉽게 정적 웹사이트를 실행할 수 있게 한다고 하자.

사용자가 Website 리소스의 인스턴스를 만들면 쿠버네티스가 웹 서버 파드를 기동하고 서비스를 통해 노출하도록 하는 것이다.

```yaml
# kubia-website.yaml
# 사용자 정의 오브젝트 종류
kind: Website
metadata:
  name: kubia
spec:
  # 웹사이트의 파일을 저장하는 깃 레포지터리
  gitRepo: https://github.com/~~
```

하지만 딱 보기에도 정보가 많이 부족해보인다.

CRD 오브젝트를 만들어보러 가자.

<br>

***CRD 오브젝트 생성***

```yaml
#  website-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.extensions.example.com
spec:
  scope: Namespaced
  group: extensions.example.com
  version: v1
  names:
    kind: Website
    singular: website
    plural: websites
```

```bash
# CRD를 생성하자.
$ kubectl create -f website-crd.yaml
```

참고로 group 항목은 아래의 website 리소스의 apiVersion에 들어가게 된다.

```yaml
# kubia-website.yaml
# 사용자 정의 오브젝트 종류
apiVersion: extensions.example.com/v1
kind: Website
metadata:
  name: kubia
spec:
  # 웹사이트의 파일을 저장하는 깃 레포지터리
  gitRepo: https://github.com/~~
```

```bash
$ kubectl create -f kubia-website.yaml

$ kubectl get websites
```

이제 Website 오브젝트를 승인하고 저장했으며 생성도 된다!

당연히 삭제도 된다.

<br>

---

<br>

### 사용자 정의 컨트롤러로 사용자 정의 리소스 자동화

Website 오브젝트가 서비스로 노출된 웹서버 파드를 실행하려면 Website Controller를 생성하고 배포해야 한다.

Website Controller는 Website 오브젝트 생성을 위해 API 서버를 감시한 다음 서비스와 웹서버 파드를 생성한다.

파드가 관리되고 노드 장애에서 살아 남도록 하기 위해 컨트롤러는 관리되지 않는 파드 대신 Deployment 리소스를 생성한다.

해당 내용은 실제 코드를 작성해서 띄워봐야 하기 때문에 생략한다.