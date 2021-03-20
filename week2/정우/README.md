# 2주차 : `Pod`

## 목차
1. [`Pod` is](#`Pod`-is)
2. [Why we need `Pod`?](#Why-we-need-`Pod`?)
3. [Pod 이해하기](#Pod-이해하기)
4. [`Pod`에서 컨테이너를 어떻게 사용해야 하지?](#`Pod`에서-컨테이너를-어떻게-사용해야-하지?)
5. [How to make `Pod`?](#How-to-make-`Pod`?)
6. [Label](#Label)
7. [Annotation](#Annotation)
8. [K8S Namespace](#K8S-Namespace)
9. [파드 삭제](#파드-삭제)

---
---

## `Pod` is
`Pod`는 함께 배치된 컨테이너 그룹이며 쿠버네티스의 기본 빌딩 블록이다. 

컨테이너를 개별적으로 배포하기보다는 ***컨테이너를 가진 파드를 배포하고 운영***한다. 

**일반적으로 파드는 하나의 컨테이너만 포함한다.**

파드가 여러 컨테이너를 가지고 있을 경우에, 모든 컨테이너는 항상 하나의 워커 노드에서 실행된다.

---

## Why we need `Pod`?
***컨테이너는 단일 프로세스를 실행하는 것을 목적으로 설계했다.(프로세스가 자기 자신의 자식 프로세스를 생성하는 것을 제외하면).***


단일 컨테이너에서 관련 없는 다른 프로세스를 실행하는 경우 모든 프로세스를 실행하고 로그를 관리하는 것은 모두 **사용자 책임**이다.

---

## Pod 이해하기
여러 프로세스를 단일 컨테이너로 묶지 않기 때문에, 컨테이너를 함께 묶고 하나의 단위로 관리할 수 있는 또 다른 상위구조가 필요하다.

***이게 파드가 필요한 이유다.***

### 같은 파드에서 컨테이너 간 부분 격리
개별 컨테이너가 아닌 컨테이너 그룹을 분리할 수 있다.

한 파드 내의 모든 컨테이너는 동일한 네트워크 네임스페이스와 UTS(UNIX Timesharing System) 네임스페이스 안에서 실행되기 땜누에, 모든 컨테이너는 같은 호스트 이름과 네트워크 인터페이스를 `공유`한다. 기본적으로 파일 시스템은 컨테이너 간 공유되지 않고 공유하고 싶다면 k8s의 `볼륨`이라는 깨념이 필요하다.

### 컨테이너가 동일한 IP와 포트 공간을 공유하는 방법
파드 안의 컨테이너가 동일한 네트워크 네임스페이스에서 실행되기 때문에, 동일 파드 안 컨테이너에서 실행 중인 프로세스가 같은 포트 번호를 사용하지 않도록 주의해야 한다.(단, ***동일한 파드일 때만***)

파드 안에 있는 모든 컨테이너는 동일한 루프백 네트워크 인터페이스를 갖기 때문에 `localhost`를 통해 컨테이너들이 통신할 수 있다.

### `Pod` 간 통신(플랫 네트워크)
모든 파드는 다른 파드의 IP 주소를 사용해 접근하는 것이 가능하며, 이 과정에서 NAT도 사용하지 않는다. (`LAN`과 유사하다)

---

## `Pod`에서 컨테이너를 어떻게 사용해야 하지?
한 호스트에 모든 유형의 애플리케이션을 넣지 않고, 파드는 특정한 애플리케이션만을 호스팅한다.

파드는 상대적으로 가볍기 떄문에 오버헤드 없이 필요한 만큼 파드를 가질 수 있다.

즉, 모든 것을 파드 하나에 넣는 대신에 애플리케이션을 **여러 파드로 구성하고, 각 파드에는 밀접하게 관련 있는 구성 요소나 프로세스만 포함**해야 한다.

또한, 파드는 스케일링의 기본 단위이기 때문에 수평 확장(Scale out)을 위해 파드를 분리, 배포하는 것이 필요하다.

***다음은 컨테이너를 묶어 그룹으로 만들 때 던질 수 있는 질문이다***
- 컨테이너를 함께 실행해야 하는가, 혹은 서로 다른 호스트에서 실행할 수 있는가?
- 여러 컨테이너가 모여 하나의 구성 요소를 나타내는가, 혹은 개별적인 구성 요소인가?
- 컨테이너가 함께, 혹은 개별적으로 스케일링돼야 하는가?

---
---

## How to make `Pod`?
- k8s API Version
- Resource Type
- Metadata : 이름, 네임스페이스, 레이블 및 파드에 관한 기타 정보
- Spec : 파드 컨테이너, 볼륨, 기타 데이터 등 파드 자체에 대한 실제 명세
- Status : 파드 상태, 각 컨테이너 설명과 상태, 파드 내부 IP, 기타 기본 정보 등 현재 실행 중인 파드에 관한 정보

***Simple YAML***
```yaml
apiVersion: v1
kind: Pod
metadata:
  # Pod 이름
  name: bluayer-simple
spec:
  # 컨테이너를 만드는 컨테이너 이미지
  containers:
  # 이미지 이름
  - image: luksa/kubia
    # Container 이름
    name: bluayer
    ports:
    # 애플리케이션이 수신하는 포트
    - containerPort: 8080
      protocol: TCP
```

```bash
$ kubectl create -f bluayer-simple.yaml

# 파드의 전체 YAML 보기
$ kubectl get po bluayer-simple -o yaml

# 파드 조회
$ kubectl get pods

# 로그보기
$ kubectl logs bluayer-simple

# 컨테이너 지정 로그 보기
$ kubectl logs bluayer-simple -c bluayer

# 로컬 네트워크 포트를 파드의 포트로 포워딩
# 로컬 포트 8888을 bluayer-simple Pod의 8080 포트로 포워딩
$ kubectl from 127.0.0.1:8888 -> 8080
```

---
---

## Label

레이블 = 파드를 정리하는 메커니즘이자 어떤 파드인지 쉽게 알 수 있돌고 임의의 기준에 따라 작은 그룹으로 조직화하는 방법

***레이블을 통해 파드와 기타 다른 k8s 오브젝트의 조직화가 이뤄진다.***

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bluayer-simple-v2
  labels:
    # 두 개의 레이블을 파드에 붙임
    creation_method: manual
    env: prod
spec:
  containers:
  - image: luksa/kubia
    name: bluayer
    ports:
    - containerPort: 8080
      protocol: TCP
```

```bash
# --show-labels를 통해 레이블을 볼 수 있음
$ kubectl get po --show-labels

# 특정 레이블을 자체 열에 표시하도록 지정
$ kubectl get po -L creation_method, env

# 래아블 추가
$ kubectl label po bluayer-simple creation_method=manual

# 이미 있는 레이블 변경
$ kubectl label po bluayer-simple-v2 env=debug --overwrite

# 레이블 셀렉터로 파드 subset list up
$ kubectl get po -l creation_method=manual
$ kubectl get po -l env
$ kubectl get po -l '!env'
```

### 특정 워커 노드에 파드 올리기

해당 책에서 사용한 `스케줄링`의 뜻 : 어떤 Node에 어떤 Pod를 올릴지 결정하는 것.

```bash
# 노드들 list up
$ kubectl get nodes

# GPU가지고 있는 노드에 레이블 달기
$ kubectl label node gke-kubia gpu=true
```

```yaml
# bluayer-gpu.yaml
apiVersion: v1
kind: Pod
metadata:
  name: blauyer-gpu
spec:
  # nodeSelector는 k8s에 gpu=true 레이블을 포함한 노드에 해당 파드를 배포하게 한다.
  nodeSelector:
    gpu: "true"
  containers:
  - image: luksa/kubia
    name: bluayer
```

***참고 : 파드를 특정 노드로 스케줄링할 수 있지만, nodeSelector에 실제 호스트 이름을 지정할 경우에 해당 노드가 오프라인 상태인 경우 파드가 스케줄링되지 않을 수 있다.***

---
---

## Annotation

어노테이션은 파드나 다른 API 오브젝트에 설명을 추가해두는 것.

***즉 레이블은 Object 그룹화에 사용할 수 있지만, 어노테이션은 그렇지 않다.***

이러한 이유로 k8s에 새로운 기능이 도입될 때 흔히 어노테이션으로 새 필드를 추가하게 된다.

또한 레이블은 상대적으로 짧은 데이터를, 어노테이션에는 큰 데이터(최대 256KB)를 넣는다.

---
---

## K8S Namespace

***이는 Linux Namespace가 아니다.***

레이블을 통해서 겹쳐질 수 있는 오브젝트 그룹을 만들었다면,

k8s namespace를 통해서는 오브젝트를 겹치지 않는 그룹으로 분할할 수 있다.

이를 통해, **오브젝트를 별도 그룹으로 분리해 특정한 네임스페이스 안에 속한 리소스를 대상으로 작업할 수 있게 해주지만, 실행 중인 오브젝트에 대한 격리는 제공하지 않는다.**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: custom-namespace
```

```bash
$ kubectl create -f custom-namespace.yml
```

```bash
# 간단하게 네임스페이스 만들기
$ kubectl create namespace custom-namespace
```

```bash
# 파드에 네임스페이스 지정하기
$ kubectl create -f bluayer-manual.yaml -n custom-namespace
```

---
---

## 파드 삭제

이건 솔직히 정리하는 파일 용량이 아깝다. 찾아보도록 하자.
