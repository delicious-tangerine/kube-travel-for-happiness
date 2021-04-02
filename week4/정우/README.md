# `Service`

## 목차

- [Why we needs `Service`?](#Why-we-needs-`Service`?)
- [What's `Service`?](#What's-`Service`?)
  - [Create `Service`](#Create-`Service`)
  - [DO NOT PING TO `SERVICE`](#DO-NOT-PING-TO-`SERVICE`)
- [클러스터 외부에 있는 것들과의 연결](#클러스터-외부에-있는-것들과의-연결)
  - [`Service Endpoint`](#`Service-Endpoint`)
  - [서비스 엔드포인트 수동 구성](#서비스-엔드포인트-수동-구성)
- [외부 클라이언트에 서비스 노출](#외부-클라이언트에-서비스-노출)
  - [NodePort Service](#NodePort-Service)
  - [외부 LB(LoadBalancer)로 서비스 노출](#외부-LB(LoadBalancer)로-서비스-노출)
- [`Ingress`](#`Ingress`)
  - [Create `Ingress`](#Create-`Ingress`)
  - [하나의 Ingress로 여러 서비스 expose](#하나의-Ingress로-여러-서비스-expose)
  - [HTTPS(TLS) For `Ingress`](#HTTPS(TLS)-For-`Ingress`)
- [Readiness Probe](#Readiness-Probe)
- [Headless Service](#Headless-Service)
  - [DNS로 파드 찾기](#DNS로-파드-찾기)
- [Service에서 생기는 Problem checklist](#Service에서-생기는-Problem-checklist)

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

<br>

---
---

<br>

## What's `Service`?

동일한 기능을 제공하는 파드 그룹에 지속적인 단일 접점을 만들기 위한 리소스이다.

각 서비스는 서비스가 존재하는 동안 **절대** 바뀌지 않는 IP 주소와 포트가 있다.

클라이언트는 해당 IP와 포트로 접속한 다음 해당 기능을 지원하는 파드 중 하나로 연결된다.

<br>

<img src="https://user-images.githubusercontent.com/37579681/113383303-9ce11c80-93be-11eb-874a-639b87c00391.jpeg" width=500 height=350>

<br>

---

<br>

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
<img src="https://user-images.githubusercontent.com/37579681/113383307-9f437680-93be-11eb-97e0-48654db0b6b1.jpeg" width=500 height=350>

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

<br>

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

<br>

---

<br>

### DO NOT PING TO `SERVICE`

서비스로 curl은 동작하지만 핑은 응답이 없다. 이는 서비스의 클러스터 IP가 가상 IP 이므로 서비스 포트와 결합된 경우에만 의미가 있기 때문이다.

<br>

--- 
---

<br>

## 클러스터 외부에 있는 것들과의 연결

서비스가 클러스터 내에 있는 파드로 연결을 전달하는 것이 아니라, 외부 IP와 포트로 연결을 전달하려는 경우가 있을 수 있다.

그러니깐 아래와 같은 이미지의 경우다.

<img src="https://user-images.githubusercontent.com/37579681/113383309-a074a380-93be-11eb-80c8-e7b00e821f2e.jpeg" width=500 height=350>

<br>

이 경우에 대해 설명하기 전에 알고 가야 하는 것들이 있다. 아래에서 알아보도록 하자.

<br>

---

<br>

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

<br>

---

<br>

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

<br>

<img src="https://user-images.githubusercontent.com/37579681/113383309-a074a380-93be-11eb-80c8-e7b00e821f2e.jpeg" width=500 height=350>

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

<br>

---
---

<br>

## 외부 클라이언트에 서비스 노출

위의 그림처럼 특정 서비스를 외부에 노출해 외부 클라이언트가 액세스할 수 있게 하고 싶을 수도 있다.

이런 경우, 몇 가지 방법이 있다.

- 노드포트(`NodePort`)로 서비스 유형 설정: `NodePort` 서비스의 경우 각 클러스터 노드는 노드 자체에서 포트를 열고 해당 포트로 수신된 트래픽을 서비스로 전달한다. 이 서비스는 내부 클러스터 IP와 포트로 액세스할 수 있을 뿐 아니라 모든 노드의 전용 포트로도 액세스할 수 있다. ***(주의 : 여기서 Node는 Node.js가 아니다!)***
- 서비스 유형을 노드포트 유형의 확장인 로드밸런서로 설정 : k8s가 실행중인 클라우드 인프라에서 프로비저닝된 LoadBalancer로 서비스에 액세스할 수 있다. 로드밸런서는 트래픽을 모든 노드의 노드포트로 전달한다.
- 단일 IP 주소로 여러 서비스를 노출하는 인그레스 리소스 만들기 : HTTP 레벨에서 작동한다. 나중에 Ingress에서 살펴보도록 하자.

<br>

---

<br>

### NodePort Service
NodePort Service를 만들면 쿠버네티스는 모든 노드에 특정 포트를 할당하고(모든 노드에서 동일 포트가 사용된다.) 서비스를 구성하는 파드로 들어오는 연결을 전달한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bluayer-nodeport
spec:
  # 서비스 유형을 노드포트로 설정
  type: NodePort
  ports:
  # 서비스 내부 ClusterIP의 Port
  - port: 80
    # 서비스 대상 파드의 포트
    targetPort: 8080
    # 각 클러스터 노드의 포트 30123으로 서비스에 액세스 가능
    # 지정하지 않으면 쿠버네티스가 임의의 포트를 선택한다.
    nodePort: 30123
  selector:
    app: bluayer
```

표현하자면 이런 그림이 될 것이다.

<br>

<img src="https://user-images.githubusercontent.com/37579681/113383312-a074a380-93be-11eb-8cc7-1e6f9b28dcd1.jpeg" width=500 height=350>

<br>

첫 번째 노드의 포트 30123에서 수신된 연결은 첫 번째 노드에서 실행 중인 파드 또는 두 번째 노드에서 실행 중인 파드로 전달될 수 있다.

참고로 NodePort로 서비스에 액세스하려면 해당 NodePort에 대한 외부 연결을 허용하도록 GCP 방화벽을 구성해야 한다.

```bash
$ gcloud compute firewall-rules create bluayer-svc-rule --allow=tcp:30123
```

<br>

---

<br>

### 외부 LB(LoadBalancer)로 서비스 노출

k8s 클러스터는 일반적으로 클라우드 인프라에서 로드밸런서를 자동으로 프로비저닝하는 기능을 제공한다.

로드밸런서는 공개적으로 액세스 가능한 고유한 IP 주소를 가지며 모든 연결을 서비스로 전달한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bluayer-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    # NodePort를 지정할 수 있지만 k8s가 포트를 선택하게 한다.
  selector:
    app: bluayer
```

표현하자면 이런 그림이 될 것이다.

<br>

<img src="https://user-images.githubusercontent.com/37579681/113383315-a10d3a00-93be-11eb-8fcf-dc4b40529f7f.jpeg" width=500 height=350>

<br>

로드밸런서를 사용하는 경우에는 노드포트 서비스와는 달리 방화벽을 설정할 필요가 없다고 한다.

<br>

***참고***

외부 클라이언트가 노드 포트로 서비스에 접속할 경우, 파드에 도달하려면 추가적인 네트워크 홉(hop)이 필요할 수 있다.

외부의 연결을 수신한 노드에서 실행 중인 파드로만 트래픽을 전달하도록 서비스를 구성할 수 있는데,

서비스의 `spec` 섹션에 다음과 같이 적어주면 된다.

```yaml
spec:
  externalTrafficPolicy: Local
```

또한 원래 노드 포트로 연결을 수신하면 SNAT(Source Network Address Translation)이 수행되기 때문에 패킷의 소스 IP가 변경되고, 따라서 클라이언트의 IP가 보존되지 않는다.

그러나 위와 같이 Local External Traffic Policy는 연결 수신 노드와 파드를 호스팅하는 노드 사이에 추가 홉이 없기 때문에 클라이언트 IP 보존이 가능할 수 있다.(SNAT이 실행되지 않기 때문에)

<br>

---
---

<br>

## `Ingress`

`Ingress` : 들어가거나 들어가는 행위, 들어갈 권리, 진입로.

`Ingress`는 한 IP 주소로 수십 개의 서비스에 접근이 가능하도록 지원해준다.

<br>

<img src="https://user-images.githubusercontent.com/37579681/113383318-a1a5d080-93be-11eb-87d0-a0dc0f08ae90.jpeg" width=500 height=350>

<br>

`Ingress`를 작동시키려면 Ingress Controller가 필요하다.

그런데 k8s 환경 일부는 Ingress 기본 컨트롤러를 제공하지 않기 때문에 확인할 필요가 있다.

<br>

***참고***

Minikube에서 활성화 하는 방법은 다음과 같다.

```bash
$ minikue addons list
# ingress가 disabled인지 확인하자.

# ingress controller 활성화
$ minikue addons enable ingress

# 컨트롤러 파드 존재 유무 확인
$ kubectl get po --all-namespaces
```

<br>

---

<br>

### Create `Ingress`

bluayer.example.com으로 요청되는 모든 HTTP 요청(Ingress Controller에 수신된)을 Port 80의 bluayer-nodeport 서비스로 전송하는 인그레스 규칙을 정의했다.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: bluayer
spec:
  rules:
  # Ingress는 blauyer.example.com이라는 도메인 이름을
  # 서비스에 매핑한다.
  - host: bluayer.exmpale.com
    http:
    paths:
    # 모든 요청은 bluayer-nodeport 서비스의 Port 80으로 전달된다.
    - path: /
      backend:
        serviceName: blauyer-nodeport
        servicePort: 80
```

```bash
# Ingress 목록 확인
$ kubectl get ingresses

# IP를 확인했다면,
# bluayer.example.com을 해당 IP로 확인하도록 DNS 서버를 구성하거나
# /etc/hosts에 아래와 같이 추가한다.
IP주소    도메인이름

# 모든 것이 설정되었다!
# Ingress로 파드 액세스를 해보자.
$ curl http://bluayer.example.com
```

참고로 인그레스 컨트롤러는 클라이언트로부터 요청을 받아 **서비스로 전달하는 것이 아니라 파드에 전송한다.**

**인그레스 - 서비스 - 엔드포인트는 파드를 선택하는 데만** 사용한다!!

<br>

---

<br>

### 하나의 Ingress로 여러 서비스 expose

- 동일한 호스트의 다른 경로로 여러 서비스 매핑
```yaml
...
  - host: bluayer.example.com
    http:
      paths:
      - path: /bluayer
        backend:
          serviceName: bluayer
          servicePort: 80
      - path: /bar
        backend:
          serviceName: bar
          servicePort: 80
```

- 서로 다른 호스트로 서로 다른 서비스 매핑
```yaml
spec:
  rules:
  - host: foo.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: foo
          servicePort: 80
  - host: bar.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: bar
          servicePort: 80
```

<br>

---

<br>

### HTTPS(TLS) For `Ingress`

```bash
# TLS 개인 키 만들기
$ openssl genrsa -out tls.key 2048

# 위에서 만들어진 개인키를 바탕으로
# 인증서 만들기
$ openssl req -new -x509 -key tls.key -out tls.cert -days 360 -subj

# 나중에 다루겠지만
# secret이라는 쿠버네티스 리소스에 저장한다.
# tls-secret이라는 이름의 secret resource에 개인키와 인증서가 저장된다.
$ kubectl create secret tls tls-secret --cert=tls.cert --key=tls.key
```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: bluayer
spec:
  tls:
  - host:
    - bluayer.example.com
    # 위에ㅐ서 만든 tls-secret
    secretName: tls-secret
  rules:
  # Ingress는 blauyer.example.com이라는 도메인 이름을
  # 서비스에 매핑한다.
  - host: bluayer.exmpale.com
    http:
    paths:
    # 모든 요청은 bluayer-nodeport 서비스의 Port 80으로 전달된다.
    - path: /
      backend:
        serviceName: blauyer-nodeport
        servicePort: 80
```

이렇게만 만들면 HTTPS로 인그레스를 통해 서비스에 액세스할 수 있다.

<br>

---
---

<br>

## Readiness Probe

레디니스 프로브는 주기적으로 호출되며 특정 파드가 클라이언트 요청을 수신할 수 있는지를 결정한다.

즉, 컨테이너의 레디니스 프로브가 성공을 반환하면 컨테이너가 요청을 수락할 준비가 됐다는 신호다.

더 구체적으로 이야기하자면 파드가 준비되어 있지 않으면 해당 파드는 서비스에서 제거되었다가, 파드가 다시 준비되면 서비스에 다시 추가된다.(실제로는 엔드포인트에 추가되었다가 제거되거나 한다.)

유형은 세 가지가 있다.

- Process를 실행하는 Exec Probe
- HTTP GET Probe
- TCP Socket Probe, 소켓이 연결되면 준비된 것으로 간주한다.

아래의 yaml은 파일이 있는지 확인해볼 수 있는 ls를 이용한 레드니스 프로브이다.

```yaml
apiVersion: v1
kind: ReplicationController
...
spec:
  ...
  template:
  ...
    spec:
      containers:
      - name: bluayer
        image: jungwoo/blauyer
        # 파드의 각 컨테이너에 레디니스 프로브를 정의할 수 있다.
        readinessProbe:
          exec:
            command:
            - ls
            - /var/ready
```

<br>

***중요한 점***
1. 레디니스 프로브를 항상 정의하라 : 파드에 레디니스 프로브를 추가하지 않으면 파드가 시작하는 즉시 서비스 엔드포인트가 된다.
2. 레디니스 프로브에 파드의 종료 코드를 포함하지 마라

<br>

---
---

<br>

## Headless Service

클라이언트가 모든 파드에 연결해야 하는 경우에 어떻게 해야할까?

각 파드의 IP를 알아아 할텐데, DNS 서버를 통해 하나의 DNS A Record 대신 서비스에 대한 여러 개의 A record를 반환한다.

각 레코드는 서비스를 지원하는 개별 파드의 IP를 가리키며, 클라이언트는 DNS A record 조회를 통해 서비스에 포함된 모든 파드의 IP를 얻어 연결할 수 있다.

```yaml
# bluayer-svc-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: bluayer-headless
spec:
  clusterIP: None
  ports:
    - port: 80
      targetPort: 8080
    selector:
      app: bluayer
```

DNS 관련 작업을 수행하려면 nslookup할 수 있는 컨테이너 이미지를 이용해 Pod를 올리고... 많은 작업이 필요하지만,

좀 더 빠른 작업을 아래에서 소개하고자 한다.

<br>

---

<br>

### DNS로 파드 찾기

```bash
# DNS 조회가 가능한 파드 만들기
$ kubectl run dnsutils --image=tutum/dnsutils --generator=run-pod/v1 --command -- sleep infinity

# 새 파드로 DNS 조회 하기
$ kubectl exec dnsutils nslookup bluayer-headless
# 파드들의 IP를 보여준다.
```

<br>

---
---

<br>

## Service에서 생기는 Problem checklist

사실 Service를 구축하다 보면 엄청나게 많은 문제를 마주하고 시간을 허비하게 된다.

따라서 아래의 체크리스트를 한 번 확인해보자.

- [ ] 먼저 외부가 아닌 클러스터 내에서 서비스의 클러스터 IP에 연결되는지 확인한다.
- [ ] 서비스에 액세스할 수 있는지 확인하려고 서비스 IP로 핑을 할 필요 없다.(서비스의 ClusterIP는 가상 IP이므로 Ping 되지 않는다.)
- [ ] 레디니스 프로브를 정의했다면 성공했는지 확인하라.
- [ ] 파드가 서비스의 일부인지 확인하려면 `kubectl get endpoints를 사용해 해당 엔드포인트 오브젝트를 확인한다.
- [ ] FQDN이나 그 일부로 서비스에 액세스하려고 하는데 작동하지 않는 경우, FQDN 대신 클러스터 IP를 사용해 액세스할 수 있는지 확인한다.
- [ ] 대상 포트가 아닌 서비스로 노출된 포트에 연결하고 있는지 확인한다.
- [ ] 파드 IP에 직접 연결해 파드가 올바른 포트에 연결돼 있는지 확인한다.
- [ ] 파드 IP로 애플리케이션에 액세스할 수 없는 경우 애플리케이션이 로컬호스트에만 바인딩하고 있는지 확인한다.
