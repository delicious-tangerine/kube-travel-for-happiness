# `Controller`

## 목차
- [`Pod`를 안정적으로 유지하기](#`Pod`를-안정적으로-유지하기)
  
  - [Liveness Probe?](#Liveness-Probe?)

  - [Make and Run `Liveness Probe`](#Make-and-Run-`Liveness-Probe`)

  - [Liveness Probe Attributes](#Liveness-Probe-Attributes)

  - [실제로 Liveness Probe를 사용할 때 고려해야 할 점들](#실제로-Liveness-Probe를-사용할-때-고려해야-할-점들)

- [Replication Controller](#Replication-Controller)

  - [How `ReplicationController` works?](#How-`ReplicationController`-works?)

  - [Make and Run `ReplicationController`](#Make-and-Run-`ReplicationController`)

  - [Check ReplicationController](#Check-ReplicationController)

  - [`Pod` and `ReplicationController`](#`Pod`-and-`ReplicationController`)

  - [Pod Template 변경](#Pod-Template-변경)

  - [Pod Scale out](#Pod-Scale-out)

  - [Delete ReplicationController](#Delete-ReplicationController)

- [`ReplicaSet`](#ReplicaSet)

- [`DaemonSet`](#`DaemonSet`)

  - [Make `DaemonSet`](#Make-`DaemonSet`)

- [`Job`](#`Job`)

  - [`Job`에서 여러 파드 인스턴스 실행하기](#`Job`에서-여러-파드-인스턴스-실행하기)

  - [주기적인 `Job` 실행(`CronJob`)](#주기적인-`Job`-실행(`CronJob`))

---
---

<br>

## `Pod`를 안정적으로 유지하기
`Pod`가 노드에 스케줄링되는 즉시, 해당 노드의 `Kubelet`은 파드의 컨테이너를 실행하고 파드가 존재하는 한 컨테이너가 계속 실행되도록 할 것이다.

컨테이너의 주 프로세스에 크래시(crash)가 발생하면 `Kubelet`이 컨테이너를 다시 시작한다.(즉, Failover 혹은 자가치유가 가능하다.)

그러나 때때로 애플리케이션은 프로세스의 크래시 없이도 작동이 중단되는 경우가 있다.

예를 들어 애플리케이션이 무한 루프나 교착 상태(Deadlock)에 빠져서 응답을 하지 않는 상황이라면, 애플리케이션이 다시 시작되도록 애플리케이션 내부에 의존하지 않고 외부에서 애플리케이션 상태를 체크해야 한다.

***이런 걸 바로 `Liveness Probe`를 통해 할 수 있다.***

<br>


---

<br>


### Liveness Probe?
`Probe`는 '탐침, 살피다'라는 뜻을 가지고 있다.

쿠버네티스는 라이브니스 프로브(Liveness Probe)를 통해 컨테이너가 살아 있는지 확인할 수 있으며 파드의 spec에 각 컨테이너의 라이브니스 프로브를 지정할 수 있다.

이 프로브를 주기적으로 실행하고 프로브가 실패할 경우 컨테이너를 재시작한다.

쿠버네티스는 세 가지 메커니즘을 이용해 컨테이너에 프로브를 실행하는데,

- HTTP GET 프로브는 지정한 IP 주소, 포트, 경로에 HTTP GET 요청을 수행한다. 프로브가 응답을 수신하고 응답 코드가 오류를 나타내지 않는 경우(2xx 또는 3xx)에 프로브가 성공했다고 간주된다.
- TCP 소켓 프로브는 컨테이너의 지정된 포트에 TCP 연결을 시도한다. 연결에 성공하면 프로브가 성공한 것이다.
- Exec 프로브는 컨테이너 내의 임의의 명령을 실행하고 명령의 종료 상태 코드를 확인한다. 상태 코드가 0이면 프로브가 성공한 것이다. 모든 다른 코드는 실패로 간주된다.

<br>

***참고***

개인적으로 HTTP의 다른 메소드(Post, Put, Patch, Delete, Option)들은 liveness probe로 못 체크하는지 궁금해서 찾아봤는데,

따로 k8s 제공해주는 것은 없고 꼭 체크해야 한다면 **curl**을 이용해 확인하는 방법이 있다고 한다.

```yaml
...
  livenessProbe:
    exec:
      command:
        - curl
        - -X POST
        - http://localhost/~
...
```

<br>


---

<br>


### Make and Run `Liveness Probe`

참고로 `jungwoo/bluayer-unhealthy` 이미지는 500을 반환하는 이미지라고 가정하자.

```yaml
apiVersion: v1
kind: pod
metadata:
    name: bluayer-liveness
spec:
    containers:
    - image: jungwoo/bluayer-unhealthy
      name: bluayer
      livenessProbe:
      httpGet:
        path: /
        port: 8080
```

위의 yaml로 Pod를 생성하고 나면 다섯 번의 요청 후에 쿠버네티스는 프로브를 실패한 것으로 간주해 컨테이너를 다시 시작한다.

***참고***

```bash
// 이전 컨테이너가 종료된 이유를 파악하려는 경우,
// 이전 컨테이너의 로그를 보고 싶을 것이다.
// --previous 옵션을 쓰자

$ kubectl logs mypod --previous

// 컨테이너가 다시 시작된 이유를 확인하고 싶다면

$ kubectl describe po bluayer-liveness
...
// Last State 부분과 Events 부분을 통해 컨테이너의 이전 상황과 종료된 이유를 확인할 수 있다.
// 참고로 종료 코드는 128 + 리눅스 관련 종료 코드이다.
// ex1) 137 = 128 + 9(SIGKILL)
// ex2) 143 = 128 + 15(SIGTERM)
```

<br>


---

<br>


### Liveness Probe Attributes
```yaml
livenessProbe:
  httpGet:
    path: /
    port: 8080
  # 첫 번쨰 프로브 실행까지 딜레이 하는 시간
  # Min은 0s이기 때문에 프로세스 시작에 걸리는 시간을 꼭 고려해줘야 한다.
  initialDealySeconds: 15
  # 얼마나 자주 프로브를 실행할지에 대한 값
  # Defaults = 10s, Min = 1s
  periodSeconds: 10
  # 프로브 타임아웃 시간
  # Defaults = 1s, Min = 1s
  timeOutSeconds: 1
  # 연속적으로 성공해야 하는 최소 수
  # Defaults = 1, Min = 1
  successThreshold: 1
  # 한 번 실패했을 경우, 프로브가 포기하기 전까지 시도해야 하는 횟수
  # Defaults = 3, Min = 1
  failureThreshold: 3
```

---

<br>


### 실제로 Liveness Probe를 사용할 때 고려해야 할 점들
- **라이브니스 프로브가 확인해야 할 사항** : 애플리케이션의 내부만 체크하고, 외부 요인의 영향을 받지 않도록 해야 한다.
(예를 들어, DB와 연결할 수 없어서 웹 서버의 라이브니스 프로브가 실패하는 경우 근본적인 원인이 DB에 있다면 단순히 웹 서버 컨테이너 재시작으로 문제가 해결되지 않는다.)
- **프로브를 가볍게 유지하기** : 너무 많은 일을 하는 프로브는 컨테이너의 속도를 상당히 느려지게 만든다. 
- **프로브에 재시도 루프 구현하지 않기** : 실패 임계를 1로 설정해도 쿠버네티스는 실패를 한 번 했다고 간주하기 전에 프로브를 여러 번 재시도한다. 재시도 루프를 구현하는 것은 의미없다.

좋다. 우리는 컨테이너에 크래시가 발생하거나 라이브니스 프로브가 실패한 경우를 해결할 수 있게 되었다.

<br>

***그러나, 노드 자체에 크래시가 발생헀다면?***

kubelet은 노드에서 실행되기 때문에 **노드 자체가 고장나면 아무것도 할 수 없다.**

**이를 해결하기 위한 메커니즘을 이제 알아보자.**

<br>


---
---

<br>


## Replication Controller

어떤 이유에서든 파드가 사라지면, 쉽게 말해 클러스터에서 노드가 사라지거나 노드에서 파드가 제거된 경우,

***레플리케이션 컨트롤러는 사라진 파드를 감지해 교체 파드를 생성한다.***

<img src="https://user-images.githubusercontent.com/37579681/112585881-09e43780-8e3e-11eb-902c-d10da37b3a3f.jpeg" alt="ReplicationController" width=500 height=350>

<br>


---

<br>


### How `ReplicationController` works?

레플리케이션컨트롤러는 실행 중인 파드 목록을 지속적으로 모니터링하고, 특정 유형(type)의 실제 파드 수가 의도하는 수와 일치하는지 항상 확인한다.

어떻게 의도하는 수의 복제본보다 많은 복제본이 생길 수 있는가?

- 누군가 같은 유형의 파드를 수동으로 만든다.
- 누군가 기존 파드의 유형(type)을 변경한다.
- 누군가 의도하는 파드 수를 줄인다.

**이 맥락에서의 유형(type)** : 특정 레이블 셀렉터와 일치하는지 아닌지

<br>

레플리케이션컨트롤러의 ***파드 수 조정 알고리즘***은 다음과 같다.

<img src="https://user-images.githubusercontent.com/37579681/112585889-0cdf2800-8e3e-11eb-9aa6-5b59c36321b1.jpeg" alt="Pod number algo" width=500 height=350>

<br>

<br>

레플리케이션 컨트롤러에는 세 가지 ***필수 요소***가 있다.

- **레이블 셀렉터(label selector)** : 레플리케이션 컨트롤러의 범위에 있는 파드를 결정한다.
- **레플리카 수(replica count)** : 실행할 파드의 의도하는 수를 지정한다.
- **파드 템플릿(pod template)** : 새로운 파드 레플리카를 만들 때 사용된다.(일종의 설게도)

<img src="https://user-images.githubusercontent.com/37579681/112585890-0d77be80-8e3e-11eb-9059-1eb82f5d974a.jpeg" alt="Attr of RC" width=400 height=300>

<br>
<br>

레플리케이션 컨트롤러는 매우 단순하지만 아주 강력하다.

- 기존 파드가 사라지면 새 파드를 시작해 파드가 항상 실행되도록 한다.
- 클러스터 노드에 장애가 발생하면 장애가 발생한 노드에서 실행 중인 모든 파드에 관한 교체 복제본이 생성된다. 단, 레플리케이션 컨트롤러의 제어 하에 있는 파드만.
- 수동 또는 자동으로 파드를 쉽게 수평 확장(scale out)할 수 있게 한다. 

<br>

---

<br>

### Make and Run `ReplicationController`
```yaml
# bluayer-rc.yaml

apiVersion: v1
kind: ReplicationController
metadata:
  name: bluayer
spec:
  replicas: 3
  # Pod Selector로 레플리케이션컨트롤러가 관리하는 파드 선택
  selector:
    app: bluayer
# 새 파드에 사용할 파드 템플릿    
template:
  metadata:
    # template의 Pod label은 레플리케이션컨트롤러의 레이블 셀렉터(위의 spec.selector)와 완전히 일치해야 한다.
    # 그렇지 않으면 컨트롤러에서 새 파드를 무한정 생성할 수 있다.
    labels:
      app: bluayer
  spec:
    containers:
    - name: bluayer
      image: jungwoo/bluayer
      ports:
      - containerPort: 8080
```

```bash
$ kubectl create -f bluayer-rc.yaml
```

쿠버네티스는 레이블 셀렉터 app=bluayer와 일치하는 파드 인스턴스 3개를 유지하도록 하는 bluayer 레플리케이션컨트롤러를 생성한다.

<br>


---

<br>


### Check ReplicationController

```bash
$ kubectl get pods
# 3개의 pod가 작동 중인지 확인하라

# pod를 수동으로 삭제해보자
$ kubectl delete pod bluayer-53thy

$ kubectl get pods
# Pod가 네 개가 뜰 텐데, 한 개 파드는 Terminating 중이고
# 하나는 ContinaerCreating 혹은 Running일 것이다!

# ReplicationController에 대한 정보는
$ kubectl get rc
$ kubectl describe rc bluayer
```

<br>


---

<br>


### `Pod` and `ReplicationController`

레플리케이션컨트롤러는 레이블 셀렉터와 일치하는 파드만을 관리한다.

파드의 레이블을 변경하면 레플리케이션컨트롤러의 범위에서 제거되거나 추가될 수 있다.

```bash
# 관리되고 있는 파드의 레이블을 변경했다 하자
$ kubectl label pod bluayer-dddmd app=foo --overwrite

$ kubectl get pods -L app
# 네 개의 Pod를 볼 수 있다.
```

`bluayer-dddmd` 파드는 이제 레플리케이션컨트롤러에서 관리하지 않기 때문에 수동으로 삭제하기 전까지는 삭제되지 않으며 독자적인 파드가 되었다.

<img src="https://user-images.githubusercontent.com/37579681/112585892-0ea8eb80-8e3e-11eb-8fc7-e71e02b28470.jpeg" alt="Label modified" width=500 height=350>

<br>


---

<br>


### Pod Template 변경

레플리케이션컨트롤러의 파드 템플릿을 변경하면 변경 이후에 생성된 파드만 영향을 미치며 기존 파드는 영향을 받지 않는다.

<img src="https://user-images.githubusercontent.com/37579681/112585893-0f418200-8e3e-11eb-9001-8584b06df0ea.jpeg" alt="Pod template" width=500 height=350>

<br>


---

<br>


### Pod Scale out
```yaml
# bluayer-rc.yaml

apiVersion: v1
kind: ReplicationController
metadata:
  name: bluayer
spec:
  # replicas 수를 3 -> 10으로 변경하였다.
  replicas: 10
  selector:
    app: bluayer 
template:
  metadata:
    labels:
      app: bluayer
  spec:
    containers:
    - name: bluayer
      image: jungwoo/bluayer
      ports:
      - containerPort: 8080
```
놀랍게도 이게 전부이다.

`x개의 인스턴스가 실행되게 하고 싶다`라는 상태를 k8s에 지정할 뿐이다.

참고로 축소도 마찬가지로 하면 된다.

bash로는 다음과 같이 작업하면 된다.

```bash
# replica 수 10개로 증가
$ kubectl scale rc bluayer --replicas=10
```

<br>


---

<br>


### Delete ReplicationController

```bash
# ReplicationController와 하위 Pods 삭제
$ kubectl delete rc bluayer

# ReplicationController만 삭제
$ kubectl delete rc bluayer --cascade=false
```

<br>


---
---

<br>


## `ReplicaSet`

차세대 ReplicationController이며 좀 더 나은 기능들을 가지고 있다.

레플리케이션컨트롤러의 레이블 셀렉터는 특정 레이블이 있는 파드만을 매칭시킬 수 있는 반면, 레플리카 셋의 셀렉터는 특정 레이블이 없는 파드나 레이블의 값과 상관없이 특정 레이블의 키를 갖는 파드를 매칭시킬 수 있다.

또한 레플리케이션컨트롤러는 레이블이 env=production인 파드와 레이블이 env=devel인 파드를 동시에 매칭시킬 수 없으나 레플리카셋은 가능하다.

```yaml
apiVersion: v1
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    # matchLabel 셀렉터를 사용한다는 것을 기억하자
    matchLabels:
      app: bluayer
  template:
    metadata:
      labels:
        app: bluayer
    spec:
      containers:
      - name: bluayer
        image: jungwoo/bluayer
```

```bash
$ kubectl get rs

$ kubectl describe rs
```

<br>


***이 뿐 아니라, selector에 다양한 연산자도 사용할 수 있다***

- `IN` : 레이블의 값이 지정된 값 중 하나와 일치해야 한다.
- `NotIn` : 레이블의 값이 지정된 값과 일치하지 않아야 한다.
- `Exists` : 파드는 지정된 키를 가진 레이블이 포함되어야 한다(value는 중요하지 않음). 이 연산자를 사용할 때는 value를 지정하지 않아야 한다.
- `DoesNotExist` : 파드에 지정된 키를 가진 레이블이 포함돼 있지 않아야 한다. 이 연산자를 사용할 때는 value를 지정하지 않아야 한다.

```yaml
# 실제 예시는 이렇다
selector:
  matchExpressions:
    # 이 셀렉터는 파드의 키가 'app'인 레이블을 포함해야 한다.
    - key: app
      operator: In
      # Label의 값은 'bluayer'이어야 한다.
      values:
        - bluayer
```

<br>


***참고***

`matchLabels`와 `matchExpressions`를 모두 지정하면 모든 레이블이 일치하고, 모든 표현식이 `true`로 평가된 파드가 매칭된다.

<br>


---
---

<br>


## `DaemonSet`

***Daemon*** : 멀티태스킹 운영 체제에서 데몬은 사용자가 직접적으로 제어하지 않고, 백그라운드에서 돌면서 여러 작업을 하는 프로그램을 말한다. 

클러스터의 모든 노드에, 노드당 하나의 파드만 실행되길 원하는 경우, 즉 각 노드에서 하나의 파드 복제본만 실행해야 하는 경우에 `DaemonSet`을 사용한다.

예를 들어 모든 노드에서 로그 수집기와 리소스 모니터를 실행하는 예가 있다.

노드가 다운되면 데몬셋은 다른 곳에서 파드를 생성하지 않지만, 새 노드가 클러스터에 추가되면 데몬셋은 즉시 새 파드 인스턴스를 새 노드에 배포한다.

또한 누군가 실수로 파드 중 하나를 삭제해 노드에 데몬셋의 파드가 없는 경우에도 마찬가지다.

<img src="https://user-images.githubusercontent.com/37579681/112585895-0fda1880-8e3e-11eb-865c-f490145786d4.jpeg" alt="Daemon vs Replica" width=500 height=350>

<br>
<br>

***중요 참고*** : 데몬셋은 스케줄러와 무관하기 때문에, 스케줄링이 되지 않도록 설정한 노드에서도 데몬셋이 관리하는 파드는 실행되어야만 한다.

<br>


---

<br>


### Make `DaemonSet`

5초마다 표준 출력으로 "SSD OK"를 출력하는 모의 ssd-monitor 프로세스를 실행하는 데몬셋을 생성해보자.

```yaml
apiVersion: v1
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      nodeSelector:
        disk: ssd
      containers:
      - name: main
        image: luksa/ssd-monitor
```

```bash
# daemonSet 생성
$ kubectl create -f ssd-monitor-daemonset.yaml

# DaemonSet 살펴보기
$ kubectl get ds
# 어라 왜 만들어진 게 없는 거 같지?

$ kubectl get po
# 어라 파드가 없잖아?!
# 아... 노드에 레이블 추가하는 것을 잊었군!

# 노드 리스트를 보자.
$ kubectl get node

# [node-name]에는 그냥 string 형식의 노드 이름을 넣어주면 된다.
$ kubectl label node [node-name] disk=ssd

# 레이블을 추가했으니 데몬셋이 파드 하나를 생성했겠군.

$ kubectl get po
# 파드가 생성되었다!
```

<br>


---
---

<br>


## `Job`
완료 가능한 태스크(completable task)는 프로세스가 종료된 후 다시 시작되지 않는 태스크로, 쿠버네티스에서는 이런 기능을 `Job`으로 지원한다.

즉, `Job`은 파드의 컨테이너 내부에서 실행 중인 프로세스가 성공적으로 완료되면 컨테이너를 다시 시작하지 않는 파드를 실행할 수 있다.

잡에서 관리하는 파드는 성공적으로 끝날 때까지 다시 스케줄링 되며, 따라서 작업이 제대로 완료되는 것이 중요한 임시 작업에 유용하다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  # Pod Selector를 지정하지 않았다.(즉, 파드 템플릿의 레이블을 기반으로 만들어진다.)
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      # Job은 Default restart policy인 Always를 사용할 수 없다.
      restartPolicy: OnFailure
      containers:
      - name: main
        image: luksa/batch-job
```

```bash
# job에 대한 정보 보기
$ kubectl get jobs

# Completed된 pod도 포함해서 보기
# 파드가 완료될 떄 삭제되지 않은 이유는 해당 파드의 로그를 검사할 수 있게 하기 위해서다.
$ kubectl get po -a

# 수행된 파드의 로그를 볼 수도 있다.
$ kubectl logs batch-job-28qf4
```

<br>


---

<br>


### `Job`에서 여러 파드 인스턴스 실행하기
- 순차적으로(sequential) Job Pod 실행하기

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-batch-job
spec:
  # 차례로 다섯개의 파드를 실행하며
  # 파드 중 하나가 실패하면 잡이 새 파드를 생성한다.
  completion: 5
  template:
  ...
```

- 병렬로(parallel) Job Pod  실행하기

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-batch-job
spec:
  # 차례로 다섯개의 파드를 실행하며
  completion: 5
  # 파드 두 개를 생성하고 병렬로 실행한다.
  parallelism: 2
  template:
  ...
```

- `Job` 실행 중 스케일링
```bash
# parallelism을 3으로 증가시킨다.
# 따라서 3개의 파드가 동시에 실행된다.
$ kubectl scale job multi-completion-batch-jo --replicas 3
```

<br>


***참고***

- 파드 스펙에 `activeDeadlineSeconds` 속성을 설정해 파드의 실행시간을 제한할 수 있다. 파드가 이보다 오래 실행되면 시스템이 종료를 시도하고 잡을 실패한 것으로 표시한다.
- Job manifest에서 `spec.backoffLimit`을 지정해 Job 재시도 횟수를 설정할 수 있다. 기본 값은 6이다.

<br>


---

<br>


### 주기적인 `Job` 실행(`CronJob`)

`Job` 리소스를 생성하면 즉시 해당 파드를 실행하지만, 대부분의 배치 잡은 미래의 특정 시간 또는 지정된 간격으로 이를 반복 실행해야 한다.

이런 리소스를 `CronJob`이라 부르며 잘 알려진 크론 형식으로 지정하기 때문에 매우 간편하다.


```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: batch-job-every-15-min
spec:
  schedule: "0,15,30,45 * * * *"
  # 파드는 예정된 시간에서 적어도 15초 내에 시작해야 한다.
  startingDeadlineSeconds: 15
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: periodic-batch-job
        spec:
          restartPolicy: OnFailure
          containers:
          - name: main
            image: luksa/batch-job
````
