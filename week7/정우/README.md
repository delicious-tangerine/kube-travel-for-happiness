# `Deployment`

## 목차

<br>

---
---

<br>

## 서론

결국 애플리케이션은 변경되게 될 것이다.

파드를 만든 후에는 기존 파드의 이미지를 변경할 수 없으므로 기존 파드를 제거하고 새 이미지를 실행하는 새 파드로 교체해야 한다.

모든 파드를 업데이트하는 방법은 다음과 같다.

- 기존 파드를 모두 삭제한 다음 새 파드를 시작한다.
- 새로운 파드를 시작하고, 가동하면 기존 파드를 삭제한다. 새 파드를 모두 추가한 다음 한꺼번에 기존 파드를 삭제하거나 순차적으로 새 파드를 추가하고 기존 파드를 점진적으로 제거해 이 작업을 수행한다.

첫 번째는 짧은 시간 동안 애플리케이션을 사용할 수 없다.

두 번째는 애플리케이션이 동시에 두 가지 버전을 실행해야 한다.

k8s에서는 이 두 가지 방법을 어떻게 사용하는지 알아보도록 하자.

<br>

---
---

<br>

### 오래된 파드를 삭제하고 새 파드로 교체

모든 파드 인스턴스를 새 버전의 파드로 교체하기 위해 우리는 레플리케이션 컨트롤러를 사용했었다.

버전 v1 파드 세트를 관리하는 레플리케이션컨트롤러가 있는 경우 이미지의 버전 v2를 참조하도록 파드 템플릿을 수정한 다음, 이전 파드 인스턴스를 삭제하면 쉽게 교체할 수 있다.

<img src=1>

다만 이 방식은 다운타임을 동반한다.

<br>

---

<br>

### 새 파드 시작 및 이전 파드 삭제

다운타임이 발생하지 않고 한 번에 여러 버전의 애플리케이션을 실행하도록 해보자.

**한 번에 이전 버전에서 새 버전으로 전환한다면,**

서비스를 이용해서 새 파드가 모두 실행되면 서비스의 레이블 셀렉터를 변경하고 서비스를 새 파드로 전환한다.

(이런 방식을 blue-green deployment라고 한다.)

<br>

---
---

<br>

## Rolling Update Automatically

**Rolling Update**

새 파드가 모두 실행된 이후 이전 파드를 한 번에 삭제하는 방법 대신 파드를 단계별로 교체하는 방법을 사용할 수도 있다.

레플리케이션 컨트롤러를 통해 수동으로 하는 대신, kubectl을 통해 수행할 수 있다.

(다만 이 방법도 낡은 방법이다.)

<br>

---

<br>

## Rolling Update through `kubectl`

먼저 애플리케이션을 만들자.

```js
// v1/app.js

const http = require('http');
const os = require('os');
console.log("Kubia server starting");
var handler = function(request, response) {
    console.log("Received request from " + request.connection.remoteAddress);
    response.writeHead(200);
    response.end("This is v1");
};
var www = http.createServer(handler);
www.listen(8080);
```

애플리케이션을 노출해보자.

```yaml
# kubia-rc-and-service-v1.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia-v1
spec:
  replicas: 3
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v1
        name: nodejs
---
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  type: LoadBalancer
  selector:
    app: kubia
  ports:
  - port: 80
    targetPort: 8080
```

위의 yaml을 이용해서 RC와 Service를 만들었는가?

그렇다면 버전 2에 해당하는 v2 애플리케이션을 만들어보자.

```js
...
response.end("This is v2");
...
```

이미지를 업데이트 하자.

이제 모든 준비가 끝났다! 롤링 업데이트를 해보자.

```bash
# Rolliing update!
$ kubectl rolling-update kubia-v1 kubia-v2 --image=luksa/kubia:v2
```

명령어를 실행하면 kubia-v2라는 새 레플리케이션컨트롤러가 즉시 만들어진다.

이제 kubectl은 새 컨트롤러를 scale up해서 파드를 하나씩 교체하기 시작한다.

마지막에는 결국 kubia-v2 RC와 v2 파드 세 개만 남게 된다.

이 과정에서 다운타임은 없다!

<br>

---

<br>

### No more `kubectl rolling-update`!

kubectl rolling-update의 단점은 다음과 같다.

1. 생성된 오브젝트를 k8s가 수정하는 것은 변경 기록이 제대로 남지 않을 가능성이 높다. 변경한 사람을 찾기가 어렵다!

2.롤링 업데이트의 모든 단계를 수행하는 것이 kubectl client다. 즉 네트워크 연결이 끊어지면 업데이트가 중간에 중단될 수도 있다!

3. 쿠버네티스는 시스템의 상태 선언을 통해 스스로 상태를 달성하도록 하는 것이 핵심인데, 실제 명령을 사용하게 되었다. k8s 디자인 이념과 상반된다.

따라서 배포를 위한 `Deployment`라는 리소스를 도입하게 되었다.

<br>

---
---

<br>

## `Deployment`

`Deployment`는 낮은 수준의 개념으로 간주되는 RC 또는 RS(Replica Set)을 통해 수행하는 대신 애플리케이션을 배포하고 선언적(declarative)으로 업데이트하기 위한 높은 수준(high-level)의 리소스다.

디플로이먼트를 생성하면 레플리카셋 리소스가 그 아래에 생성되고 RS가 파드를 관리하게 된다.

롤링 업데이트 예제에서 봤듯, 애플리케이션을 업데이트할 때는 기존 RS 뿐 아니라 새로운 RS를 만들고 잘 조화되도록 조정해야 한다.

따라서 `Deployment`는 이런 관리를 수행한다.

 <br>

---

<br>

### Create `Deployment`

```yaml
# kubia-deployment.yaml
# 책에서는 v1beta1이나, 이제 v1 버전으로 올라왔다.
apiVersion: v1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v1
        name: nodejs
```

```bash
# 먼저 RC를 삭제하자
$ kubectl delete rc --all

# create를 사용할 때는 --record 옵션을 포함해야 한다.
# 계정 이력에 명령어를 기록하기 때문에 유용하다!
$ kubectl create -f kubia-deployment-v1.yaml --record

# Rollout 상태 확인
$ kubectl rollout status deployment kubia
```

***참고***

rollout을 실행하면 알 수 없는 숫자를 볼 수 있는데, 이는 해시값으로 레플리카셋에 할당된 해시 값이다.

<br>

---

<br>

### Update `Deployment`

디플로이먼트 전략은 두 가지 정도가 있다.
- `RollingUpdate` : 이전 파드를 하나씩 제거하고 동시에 새 파드를 추가해 전체 프로세스에서 애플레케이션 계속 사용 가능 (단, 이전 버전과 새 버전을 동시에 실행할 수 있는 경우에만 사용가능하다.)
- `Recreate` : 새 파드를 만들기 전에 이전 파드 모두 삭제 (다운 타임 발생)

RollingUpdate 방식은 시간이 좀 걸리기 때문에 준비되기 위한 시간이 필요하다.

```bash
$ kubectl patch deployment kubia -p '{"spec": {"minReadySeconds": 10}}'

# curl 명령어를 지속적으로 서비스에 보내자
$ while true; do curl http://xxx.xxx.xxx.xxx; done

# Deployment를 수정해보자
$ kubectl set image deployment kubia nodejs=luksa/kubia:v2
```

curl 명령어를 지속적으로 보내게 되면 v1 파드만 요청을 받고 있다가, v2도 요청을 받게 되고, 마지막에는 v2만 남는 것을 알 수 있다.

`Deployment` 리소스에서 파드 템플릿을 변경하는 것만으로 애플리케이션을 업데이트하게 되었다!

<br>

---

<br>

### Rollback `Deployment`

롤아웃 프로세스를 하다 Rollback을 해야하는 경우를 우리는 흔하게 겪는다.

만약 배포한 새 프로세스가 에러를 뱉기 시작한다면?

우리는 신속하게 대처를 해서 롤백을 해야 한다!

```bash
# 수동으로 이를 막아보자!
# 이 명령어를 실행하면 디플로이먼트가 이전 버전으로 롤백한다.
# 롤아웃 프로세스가 진행 중인 동안에도 취소 가능하다.
# 이미 생성된 파드는 제거되고 이전 파드로 다시 교체된다!!
$ kubectl rollout undo deployment kubia

# 이전 버전 뿐 아니라, 모든 버전으로 롤백 가능하다.
# 이력을 확인해보자.
$ kubectl rollout history deployment kubia

# 이런 식으로 버전을 지정할 수 있다.
$ kubectl rollout undo deployment kuiba --to-revision=1
```

<br>

---

<br>

### Controll Rollout Speed

롤링 업데이트 전략의 추가 속성을 통해 속도를 조절할 수 있다.

- maxSurge : 디플로이먼트가 의도하는 레플리카 수보다 얼마나 많은 파드 인스턴스 수를 허용할 수 있는지를 결정한다. 기본값은 25%로 설정되며, 의도한 개수보다 최대 25% 더 많은 파드 인스턴스가 있을 수 있다. (의도하는 레플리카 수가 4라면 업데이트 중에 동시에 5개 이상이 실행되지 않는다.) => 사실상 최고 개수를 위한 설정

- maxUnavailable : 업데이트 중에 의도하는 레플리카 수를 기준으로 사용할 수 없는 파드 인스턴스 수를 결정한다. 기본적으로 25%이며, 사용 가능한 파드 인스턴스 수는 의도하는 레플리카 수의 75% 이하로 떨어지지 않아야 한다. => 사실상 최저 개수를 위한 설정

<img src=2>

<br>

---

<br>

### Pause Rollout

롤아웃을 진행하다 중단하고 상태를 보고 싶을 수 있다.

이런 경우 어떻게 해야하는지 알아보자.

```bash
# 새로운 배포 시도
$ kubectl set image deployment kubia nodejs=luksa/kubia:v4

# 일시 정지
# 새 파드를 하나 생성했지만 모든 원본 파드도 계쏙 실행 중이어야 한다.
# Canary Release가 이런 방식이다.
$ kubectl rollout pause deployment kubia

# Rollout을 진행해보자
$ kubectl rollout resume deployment kubia
```

<br>

---

<br>

### 잘못된 버전의 롤아웃 방지

위에서 `minReadySeconds`를 기억하는가?

사실 이 속성은 롤아웃 속도를 늦추는 것이 주요 기능이 아니라 오작동 버전의 배포를 방지하는 것이다!

`minReadySeconds`가 지나기 전에 새 파드가 제대로 작동하지 않고 레디니스 프로브가 실패하기 시작하면 새 버전의 롤아웃이 효과적으로 차단된다. (따라서 보통은 10초보다는 길게 설정한다.)

버그가 있는 버전이 나타나더라도, 애플리케이션이 큰 혼란을 일으키지 않도록 하는 에어백과 같다.

레디니스 프로브와 함꼐 `Deployment`를 만들어보자.

```yaml
# kubia-deployment-v3-with-readinessheck.yaml
apiVersion: v1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  minReadySeconds: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v3
        name: nodejs
        readinessProbe:
          periodSeconds: 1
            httpGet:
              path: /
              port: 8080
```

```bash
$ kubectl apply -f kubia-deployment-v3-with-readinessheck.yaml
```

실제로 실행해보면 파드가 준비되지 않은 상태로 남아 있게 될 것이다.

(참고로 v3는 제대로 작동하지 않는 버전이라 가정하자.)

새 파드가 시작되자마자 레디니스 프로브가 매초마다 시작된다. 

새 파드에서 요청이 실패하기 때문에 클라이언트는 제대로 작동하지 않는 파드에 접속하지 않도록 된다.

즉, 새 파드를 사용할 수 없기 때문에 롤아웃 프로세스가 계속되지 않고, 10초 이상 준비되어 있지 않기 때문에 사용가능하지 않은 것으로 간주된다.


