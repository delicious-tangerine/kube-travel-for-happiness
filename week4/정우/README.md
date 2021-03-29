# `Service`

## 목차

---
---

<br>

## Why we needs `Service`?

Pod가 다른 Pod에게 제공하는 서비스를 사용하려면 다른 파드를 찾는 방법이 필요하다.

기존 방식대로라면, 서버의 정확한 IP 주소나 호스트 이름을 지정해 구성할 수 있겠지만,

쿠버네티스에서 동일한 작업을 수행하면 이런 이유로 동작하지 않는다.

- 파드는 일시적이기 때문에, 언제든 사라지거나 이동될 수 있다.
- 쿠버네티스는 노드에 파드를 스케줄링한 후 파드가 시작되기 바로 전에 파드의 IP 주소를 할당한다. 따라서 동적이므로 미리 알 수 없다.
- 수평 스케일링(scale out)은 여러 파드가 동일한 서비스를 제공할 수 있음을 의미한다. 따라서 클라이언트는 어떤 파드에 요청하는지 알 수 없어야 한다. 그렇기 때문에 모든 파드는 단일 IP 주소로 액세스할 수 있어야 한다.

---
---

## What's `Service`?

동일한 기능을 제공하는 파드 그룹에 지속적인 단일 접점을 만들기 위한 리소스이다.

각 서비스는 서비스가 존재하는 동안 **절대** 바뀌지 않는 IP 주소와 포트가 있다.

클라이언트는 해당 IP와 포트로 접속한 다음 해당 기능을 지원하는 파드 중 하나로 연결된다.

<img src=1>

---

### Create `Service`

어떤 파드가 서비스의 일부분인지 정의하는 것은 레이블 셀렉터를 사용한다.

서비스를 만들 때에는 `kubectl expose`를 이용해서 만들 수도 있지만 여기서는 yaml을 이용해 만들어보겠다.


```yaml
apiVersion: v1
kind: Service
metadata:
  name: bluayer
spec:
  ports:
    # 서비스가 사용할 포트
  - port: 80
    # 서비스가 포워드할 포트
    targetPort: 8080
  selector:
    # app = bluayer 레이블이 있는 모든 파드가 이 서비스에 포함된다.
    app: bluayer
```

위의 yaml로 서비스를 생성하고 나서 다음 명령어를 실행시켜 보자.

```bash
# Service를 보여주는 명령어
$ kubectl get svc
# 클러스터 IP(클러스터 내부에서만 사용할 수 있는 IP)만이 있다.
# 외부로 서비스를 노출하는 방법은 뒤에서 설명한다.
```

<br>

***참고***

서비스 테스트는 여러 가지가 있다.
- 클러스터 IP로 요청을 보내고 응답을 로그로 남기는 파드를 만든다.
- 쿠버네티스 노드로 ssh 접속하고 curl 명령을 실행한다.
- Pod 하나를 선택해 `kubectl exec`으로 curl 명령어를 보낸다.
```bash
$ kubectl exec bluayer-123da -- curl -s http://xx.xxx.xxx.xxx
```
<img src>

<br>
<br>

***Session Affinity***

요청할 때마다 다른 파드가 선택되지만 특정 클라이언트의 모든 요청을 매번 같은 파드로 리디렉션하려면 서비스의 Session Affinity 속성을 ClientIP로 설정한다.

```yaml
apiVersion: v1
kind: Service
spec:
  # Default = None
  sessionAffinity: ClientIP
  ...
```

<br>

***여러 포트 노출***

하나의 서비스를 사용해 멀티 포트 서비스를 사용하면 단일 클러스터 IP로 모든 서비스 포트가 노출된다.

**단, 여러 포트가 있는 서비스를 만들 때는 각 포트의 이름을 지정해야 한다.**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bluayer
spec:
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  selector:
    app: bluayer
```

***지정 이름 포트 사용***

포트 이름을 지정하게 되면 Service의 spec을 바꾸지 않고도 포트 번호를 변경할 수 있다.

즉, Pod spec에서 포트 번호를 변경하기만 하면 되는 것이다.

```yaml
# Pod yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: bluayer
    ports:
    - name: http
      containerPort: 8080
    - name: https
      containerPort: 8443
```

```yaml
# Service yaml
apiVersion: v1
kind: Service
spec:
  ports:
  - name: http
    port: 80
    # 컨테이너 포트의 이름이 http(8080)인 것에 매핑
    targetPort: http
  - name: https
    port: 443
    # 컨테이너 포트의 이름이 https인 것(8443)에 매핑
    targetPort: https
```

---

### DO NOT PING TO `SERVICE``

서비스로 curl은 동작하지만 핑은 응답이 없다. 이는 서비스의 클러스터 IP가 가상 IP 이므로 서비스 포트와 결합된 경우에만 의미가 있기 때문이다.

--- 
---

## 클러스터 외부에 있는 것들과의 연결

서비스가 클러스터 내에 있는 파드로 연결을 전달하는 것이 아니라, 외부 IP와 포트로 연결을 전달하려는 경우가 있을 수 있다.

그러니깐 아래와 같은 이미지의 경우다.

<img>

<br>

이 경우에 대해 설명하기 전에 알고 가야 하는 것들이 있다.

---

### `Service Endpoint`

서비스는 파드에 직접 연결되지 않고 그 사이에 Endpoint Resource가 있다.

```bash
$ kubectl describe svc bluayer
Name:         bluayer
Namespace:    default
Labels:       <none>
# 서비스의 파드 셀렉터는 엔드포인트 목록을 만드는데 사용된다.
Selector:     app=bluayer
Type:         ClusterIP
IP:           10.xxx.xxx.xxx
Port:         <unset> 80/TCP
# 이 서비스의 엔드포인트를 나타내는 파드 IP와 포트
Endpoints:    10.108.1.4:8080,10.108.2.5:8080
Session Affinity: None
No events.
```

Endpoint도 기본적으로 resource기 때문에 기본 정보를 볼 수 있다.

```bash
$ kubectl get endpoints bluayer
```

---

### 서비스 엔드포인트 수동 구성

서비스의 엔드포인트를 서비스와 분리하면 엔드포인트를 수동으로 구성하고 업데이트 할 수 있다.

파드 셀렉터 없이 서비스를 만들면 쿠버네티스는 **자동으로** 엔드포인트 리소스를 만들지 못하기 때문에, 유저가 직접 엔드포인트 리소스를 만들어줘야 한다.

```yaml
# external-service.yaml
# Pod Selector가 없다!
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 80
```

```yaml
# external-service-endpoints.yaml
apiVersion: v1
kind: Endpoints
metadata:
  # 엔드포인트의 이름은 서비스 이름과 일치해야 한다!!!
  name: external-service
subsets:
  - addresses:
    # 서비스가 연결을 전달할 엔드포인트의 IP
    - ip: 11.11.11.11
    - ip: 22.22.22.22
    ports:
    # 엔드 포인트의 대상 포트
    - port: 80
```

결국 아까 나왔던 이런 그림의 인터넷 연결이 가능하게 된다.

<img>

<br>

***참고***

엔드포인트를 수동으로 구성하는 방법보다 더 간단한 방법이 있다. 바로 `FQDN(Fully Qualified Domain Name)`을 사용하는 것이다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  # 타입이 ExternalName으로 설정되어야 한다!
  type: ExternalName
  # FQDN(실제 서비스의 정규화된 도메인 이름)
  externalName: someapi.somecompnay.com
  ports:
  - port: 80
```

---
---

## 외부 클라이언트에 서비스 노출

위의 그림처럼 특정 서비스를 외부에 노출해 외부 클라이언트가 액세스할 수 있게 하고 싶을 수도 있다.

이런 경우, 몇 가지 방법이 있다.

- 노드포트(`NodePort`)로 서비스 유형 설정: `NodePort` 서비스의 경우 각 클러스터 노드는 노드 자체에서 포트를 열고 해당 포트로 수신된 트래픽을 서비스로 전달한다. 이 서비스는 내부 클러스터 IP와 포트로 액세스할 수 있을 뿐 아니라 모든 노드의 전용 포트로도 액세스할 수 있다. ***(주의 : 여기서 Node는 Node.js가 아니다!)***
- 