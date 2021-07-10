# `Stateful Set`

## 목차

<br>

---
---

<br>

## 서론

데이터베이스 파드를 복제하는 데 레플리카셋을 사용할 수 있을까?

정답은 아래에서 알아보도록 하자.

<br>

---
---

<br>

## 스테이트풀 파드 복제하기

파드 템플릿이 특정 PVC를 참조하는 볼륨을 포함하면 레플리카셋의 모든 레플리카는 정확히 동일한 PVC, PV를 사용한다.

각 인스턴스가 별도의 스토리지를 필요로 하는 분산 데이터 저장소를 실행하려면 레플리카셋을 사용할 수 없다.

적어도 단일 레플리카셋을 사용해서는 안 된다.

<br>

---

<br>

### 개별 스토리지를 갖는 레플리카 여러 개 실행하기

**수동으로 파드 생성하기**

수동으로 관리를 한다? 왜 k8s를 쓰는가?

<br>

**파드 인스턴스별로 하나의 레플리카셋 사용하기**

파드를 직접 생성하는 대신 여러 개의 레플리카셋을 생성할 수 있다.

그렇지만 단일 레플리카셋을 사용하는 것에 비해 훨씬 번거롭다.

파드를 스케일링 할 때 새로운 레플리카셋을 만들어야 한다.

이게 최선일까?

<br>

**동일 볼륨을 여러 개 디렉터리로 사용한다**

모든 파드가 동일한 PV를 사용하게 하되 각 파드의 볼륨 내부에서 별도의 파일 디렉터리를 갖게 한다.

그러나 단일 파드 템플릿으로부터 파드 레플리카를 다르게 설정할 수 없기 때문에 각 인스턴스에 어떤 디렉터리를 사용해야 하는지 전달할 수 없다.

물론 가능한 방법이 있긴 하지만 복잡하고 공유 스토리지 볼륨에 병목 현상이 발생할 수 있다.


<br>

---

<br>

### 각 파드에 안정적인 아이덴티티 제공하기

안정적인 네트워크 아이덴티티가 필요한 이유는 분산 스테이트풀 애플리케이션에서 이전 데이터를 가지고 시작할 때 생기는 문제를 막기 위해 필요하다.

k8s는 새 파드는 새로운 호스트 이름과 IP 주소를 할당받으므로 이를 해결하기 위해 다른 방법이 필요하다.

이 문제를 해결하는 방법은 각 개별 멤버에게 전용 쿠버네티스 서비스를 생성해 클러스터 멤버에 안정적인 네트워크 주소를 제공하는 것이다.

하지만 개별 파드는 자신이 어떤 서비스를 통해 노출되는지 알 수 없으므로 그 IP를 사용해 다른 파드에 자신을 등록할 수 없다.

**이런 문제들은 스테이트풀셋이 해결해준다**

<br>

---
---

<br>

## 스테이트풀셋 이해하기

스테이트풀셋은 **애플리케이션의 인스턴스가 각각 안정적인 이름과 상태를 가지며** 개별적으로 취급돼야 하는 애플리케이션에 알맞게 만들어졌다.

<br>

---
---

<br>

### 스테이트풀셋과 레플리카셋 비교하기

***스테이트리스 애플리케이션 == 가축***

인스턴스가 죽더라도 새로운 인스턴스를 만들 수 있고 사람들은 그 차이를 알아차리지 못할 것이기 때문에

인스턴스가 죽는 것은 아무런 문제가 되지 않는다.

**레플리카셋이나 레플리케이션컨트롤러로 관리되는 파드는 가축과 같다.**

<br>

***스테이트풀 애플리케이션 == 애완동물***

잃어버린 애완동물을 대체하려면 이전 애완동물과 생김새나 행동이 완전히 똑같은 새로운 애완동물을 찾아야 한다.

스테이트풀셋은 파드가 아이덴티티와 상태를 유지하면서 다시 스케줄링되게 한다.

<br>

---

<br>

### 안정적인 네트워크 아이덴티티 제공하기

스테이트풀셋으로 생성된 파드는 0부터 시작되는 인덱스가 할당되고 파드의 이름과 호스트 이름, 안정적인 스토리지를 붙이는 데 사용한다.

따라서 파드 이름을 예측할 수 있고 파드는 임의의 이름이 아닌 잘 정리된 이름을 갖는다.

**스테이트풀 파드는 각각 서로 다르므로 그룹의 특정 파드에서 동작하기를 원할 수 있다.**

따라서 스테이트풀셋은 **거버닝 헤드리스 서비스(Governing Headless Service)** 를 생성해서 각 파드에게 실제 네트워크 아이덴티티를 제공해야 한다.

이 서비스를 통해 각 파드는 자체 DNS 엔트리를 가지며 클러스터의 피어 혹은 클러스터의 다른 클라이언트가 호스트 이름을 통해 파드의 주소를 지정할 수 있다.

<br>

***예시***

default라는 네임스페이스에 속하는 foo라는 이름의 거버닝 서비스가 있고 파드의 이름이 A-0라면,

이 파드는 a-0.foo.default.svc.cluster.local이라는 FQDN을 통해 접근할 수 있다.

또한 foo.default.svc.cluster.local 도메인의 SRV 레코드를 조회해 모든 스테이트풀셋의 파드 이름을 찾는 목적으로 DNS를 사용할 수 있다.

<br>

***교체 관련***

스테이트풀셋으로 관리되는 파드 인스턴스가 사라지면 스테이트풀셋은 레플리카셋처럼 새로운 인스턴스로 교체한다.

단, 교체된 파드는 사라진 파드와 동일한 이름과 호스트 이름을 갖는다.

<br>

***Scaling***

스케일링하게 되면 사용하지 않는 다음 서수 인덱스를 갖는 새로운 파드 인스턴스를 생성한다.

인스턴스 두 개에서 세 개로 스케일업하면 새로운 인스턴스는 인덱스 2를 부여받는다.

스테이트풀셋의 스케일 다운의 좋은 점은 항상 어떤 파드가 제거될지 알 수 있다는 점이다.

스테이트풀셋의 스케일다운은 항상 가장 높은 서수 인덱스를 먼저 제거한다.

또한 해당 제거는 순차적으로 하나씩 제거된다.

(3개가 있다면 2번부터 제거하는 형식)

**따라서 스케일 다운의 영향을 예측할 수 있다.**

<br>

또한 스테이트풀셋은 인스턴스가 하나라도 비정상적인 경우 스케일 다운 작업을 허용하지 않는다.

만약 인스턴스 두 개 중 하나가 작동하지 않는 상황이라면, 제거하게 되면 두 개 모두 잃을 수 있기 때문이다.


---

<br>

### 각 스테이트풀 인스턴스에 안정적인 전용 스토리지 제공하기

각 스테이트풀 파드 인스턴스는 자체 스토리지를 사용할 필요가 있고 스테이트풀 파드가 다시 스케줄링되면,

새로운 인스턴스는 동일한 스토리지에 연결되어야 한다.

스테이트풀셋은 각 파드와 함꼐하는 PVC를 복제하는 하나 이상의 볼륨 클레임 템플릿을 가질 수 있다.

**따라서 볼륨 클레임 템플릿을 통해 PVC를 복제한다!!**

<img src="https://user-images.githubusercontent.com/37579681/125171799-1a53ce80-e1f1-11eb-9a8a-8cf1aa5f3f02.jpeg">

<br>

<br>

***PVC 생성과 삭제의 이해***

스케일 다운을 할 때는 파드만 삭제하고 클레임은 남겨둔다.

클레임이 삭제된다면 바인딩됐던 PV는 재활용되거나 삭제돼 콘텐츠가 손실되기 때문에 남겨둔다.

따라서 PV를 release하려면 PVC를 수동 삭제 해야한다.

<br>

***동일 파드의 새 인스턴스에 PVC 다시 붙이기***

스케일 다운 이후 스케일 업을 하면 PV에 바인딩된 동일한 클레임을 다시 연결할 수 있다.

새로운 파드에 그 콘텐츠가 연결된다는 것을 의미한다.

<img src="https://user-images.githubusercontent.com/37579681/125171801-1b84fb80-e1f1-11eb-9dba-801c4308b79a.jpeg">

<br>

<br>

---
---

<br>

### 스테이트풀셋 보장 이해하기

쿠버네티스는 두 개의 스테이트풀 파드 인스턴스가 절대 동일한 아이덴티티로 실행되지 않고

동일한 PVC에 바인딩되지 않도록 보장한다.

즉, 스테이트풀셋은 교체 파드를 생성하기 전에 파드가 더 이상 실행 중이지 않는다는 점을 절대적으로 확신해야 한다.

<br>

---
---

<br>

## Stateful Set 사용하기

```js
...
const dataFile = "/var/data/kubia.txt";
...
var handler = function (request, response) {
    if (request.method == 'POST') {
        var file = fs.createWriteStream(dataFile);
        file.on('open', function (fd) {
            request.pipe(file);
            console.log("New data has been received and stored.");
            response.writeHead(200);
            response.end("Data stored on pod " + os.hostname() + "\n");
        });
    } else {
        var data = fileExists(dataFile)
            ? fs.readFileSync(dataFile, 'utf8')
            : "No data posted yet";
        response.writeHead(200);
        response.write("You've hi " + os.hostname() + "\n");
        response.end("Data stored on this pod: " + data + "\n");
    }
};

var www = http.createServer(handler);
www.listen(8080);
```

```
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
```

<br>

---

<br>

### 스테이트풀셋을 통한 애플리케이션 배포하기

애플리케이션을 배포하려면 다른 유형의 오브젝트 두 가지(또는 세 가지)를 생성해야 한다.

- 데이터 파일을 저장하기 위한 PV
- 스테이트풀셋에 필요한 거버닝 서비스
- 스테이트풀셋 자체

각 파드 인스턴스를 위해 스테이트풀셋은 PV에 바인딩되는 PVC을 생성한다.

클러스터가 동적 프로비저닝을 지원하면 PV을 수동으로 생성할 필요가 없다.

하지만 그렇지 않다면 PV를 생성해야 한다. (이건 Skip)

<br>

**거버닝 서비스 생성하기**

```yaml
# kubia-service-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  # clusterIP 필드를 None으로 설정하면 헤드리스 서비스가 된다.
  # 이로써 파드 간 피어 디스커버리를 사용할 수 있다.
  clusterIP: None
selector:
  app: kubia
ports:
  - name: http
    port: 80
```

<br>


**스테이트풀셋 매니페스트 생성하기**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kubia
spec:
  serviceName: kubia
  replicas: 2
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
        - name: kubia
          image: bluayer/kubia-pet
          ports:
          - name: http
            containerPort: 8080
          volumeMounts:
            - mountPath: /var/data
              name: data
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    resources:
      requests:
      storage: 1Mi
      accessModes:
        - ReadWriteOnce
```

<br>

이제 만들어보자

```bash
# 스테이트풀셋 만들기
$ kubectl create -f kubia-statefulset.yaml

# 파드 조회
$ kubectl get po
```

스테이트풀셋이 두 개의 레플리카를 생성하도록 구성됐지만 하나의 파드만이 생성됐다.

하지만 잘못된 것이 없다. 단지 첫 번째 파드가 생성되고 준비가 완료돼야 두 번째 파드가 생성된다.

특정 클러스터된 스테이트풀 애플리케이션은 두 개 이상의 멤버가 동시에 생성되면 레이스 컨디션에 빠질 가능성이 있기 때문에 이렇게 동작한다.

<br>

---

<br>

### 파드 가지고 놀기

헤드리스 서비스를 생성했기 때문에 서비스를 통해 통신할 수 없다!

따라서 파드에 직접 연결해야 하는데, 파드 내부에서 curl을 실행하거나 포트포워딩 해야하는 법은 이전 포스팅에서 봤다.

이번에는 API 서버를 파드의 프록시로 사용해보자!

Kubia-0 파드에 요청을 보내고 싶다면 다음 URL을 호출해보자.

```bash

$ kubectl proxy


# URL은 이렇다 : <apiServiceHost>:<post>/api/v1/namespaces/default/pods/kubia-0/proxy/<path>
# 요청이 잘 받아지고 처리되었는가?
$ curl <URL>

# POST를 전송해서 기록하자!
$ curl -X POST -d "Hey there! This greeting was submitted to kubia-0." <URL>
```

<br>

**스테이트풀 파드를 삭제해 재스케줄링된 파드가 동일 스토리지에 연결되는지 확인하기**

```bash
$ kubectl delete po kubia-0

# 정상적으로 종료되자마자 스테이트풀셋으로 동일한 이름의 새 파드가 생성된다.
$ kubectl get po
```

새 파드는 클러스터의 어느 노드에나 스케줄링될 수 있으며 이전 파드가 스케줄링됐던 동일한 노드일 필요가 없다.

이전 파드의 모든 아이덴티티(이름, 호스트 이름, 스토리지)는 새 노드로 효과적으로 이동된다.

```bash
# 영구 데이터가 남았는지 보자!
$ curl <URL>
```

<img src="https://user-images.githubusercontent.com/37579681/125171802-1d4ebf00-e1f1-11eb-89d5-986a9e7745fa.jpeg">

<br>

<br>

**스테이트풀 파드를 헤드리스가 아닌 일반적인 서비스로 노출하기**

```yaml
# kubia-service-public.yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-public
spec:
  selector:
    app: kubia
  ports:
  - port: 80
    targetPort: 8080
```

헤드리스가 아닌 서비스를 파드 앞에 추가해본다.

외부에 서비스를 노출하지 않으므로 클러스터 내부에서만 접근할 수 있다.

서비스에 접근하려면 API 서버를 통해 클러스터 내부 서비스에 연결하자.

서비스에 프록시를 요청하는 URL은 다음과 같다.

```bash
# URL : <apiServiceHost>:<post>/api/v1/namespaces/<namespace>/services/<service name>/proxy/<path>
$ curl <URL>
```

<br>

---
---

<br>

## 스테이트풀셋의 피어 디스커버리

클러스터의 다른 멤버를 찾는 기능(피어 디스커버리)를 통해 스테이트풀셋의 각 멤버는 모든 다른 멤버를 쉽게 찾을 수 있어야 한다.

물론 API 서버와 통신해 찾을 수 있지만 애플리케이션을 완전히 쿠버네티스에 독립적으로 유지하며 기능을 노출하는 것이다.

따라서 애플리케이션이 쿠버네티스 API와 대화하는 것은 바람직하지 않다.

DNS 레코드 중 SRV 레코드를 이용해보자.

<br>

**SRV Record**

SRV 레코드는 특정 서비스를 제공하는 서버의 호스트 이름과 포트를 가리키는 데 사용된다.

<br>

새 임시 파드 내부에서 DNS lookup 도구인 dig를 실행해 스테이트풀 파드의 SRV 레코드를 조회할 수 있다.

```bash
$ kubectl run -it srvlookup --image=tutum/dnsutils --rm --restart=Never -- dig SRV kubia.default.svc.cluster.local

$ dig SRV kubia.default.svc.cluster.local
```

`srvlookup`이라 부르는 일회용 파드(--restart=Never)를 실행하고 콘솔에 연결되며, 종료되자마자 삭제된다.

<br>

---

<br>

### DNS를 통한 피어 디스커버리

kubia-public 서비스를 통해 데이터 저장소 클러스터로 연결된 클라이언트가 게시한 데이터는 임의의 클러스터 노드에 전달된다.

애플리케이션 서버에서 dns 관련 코드를 추가하고 SRV 레코드 룩업을 수행하고 GET 요청을 서비스를 뒷받침하느 ㄴ각 파드에 보낸다.

<br>

---

<br>

### 스테이트풀셋 업데이트

```bash
# 스테이트풀셋 업데이트 명령어
$ kubectl edit statefulset kubia

# spec.replicas을 3으로 수정하자.

$ kubectl get po

$ kubectl delete po kubia-0 kubia-1
```

<br>

---

<br>

### 클러스터된 데이터 저장소 사용하기

```bash
# 데이터를 쓰자!
$ curl -X POST -d "The sun is shining" <URL>

# 데이터를 읽어보자!
$ curl <URL>
```

클라이언트의 요청이 클러스터 노드는 중 하나에 도달하면 모든 피어를 디스커버리해 데이터를 수집한 다음 모든 데이터를 다시 클라이언트로 보낸다.

<br>

---
---

<br>

## 스테이트풀셋이 노드 실패를 처리하는 과정 이해하기

노드가 갑자기 실패하면 k8s는 노드나 그 안의 파드의 상태를 알 수 없다.

파드가 더 이상 실행 중이 아닌지, 또는 파드가 여전히 존재하고 도달할 수 있는지 알 수 없다.

스테이트풀셋은 노드가 실패한 경우 동일한 아이덴티티와 스토리지를 가진 두 개의 파드가 절대 실행되지 않는 것을 보장하므로,

스테이트풀셋은 파드가 더 이상 실행되지 않는다는 것을 확신할 때까지 대체 파드를 생성할 수 없고, 생성해서도 안 된다.

<br>

**만약 노드와 연결이 끊어진다면?**

컨트롤 플레인은 Node 내 파드들의 상태를 `Unknown`으로 바꾼다.

몇 분이 지나도 파드의 상태가 `Unknown`으로 남아 있다면 파드는 자동으로 노드에서 제거되고,

마스터(Control Plain)가 파드 리소스를 삭제해 파드를 제거한다.

kubelet이 파드가 deletion으로 표시된 것을 확인하면 파드 종료를 시작한다.

Pod들이 Unknown인 상태에서 파드들을 삭제해보자.

<br>

---

<br>

### 수동 파드 삭제

```bash
$ kubectl delete po kubia-0

$ kubectl get po
```

이상하다. 파드가 삭제되었고 kubectl도 삭제되었다고 말했다. 하지만 왜 동일한 파드(kubia-0) 남아있을까?

그 이유는 노드의 네트워크가 다운됐기 때문에 파드가 제거되지 않는 것이다.

따라서 다음처럼 파드를 강제로 삭제해보자.

```bash
# 스테이트풀 파드를 강제로 삭제하는 것은 위험하다는 것을 인지하자.
$ kubectl delete po kubia-0 --force --grace-period 0

$ kubectl get po
```

파드를 다시 조회해보면 비로소 kubia-0 파드가 생성됐음을 확인할 수 있다.

노드를 재시작하면 다시 정상으로 돌아온다.

**스테이트풀셋은 노드가 망가지게 되면 파드 삭제가 이뤄지지 않음을 기억하자!!**
