# `Volume`

## 목차

0. [서론](#서론)

1. [Introduce about `Volume`](#Introduce-about-`Volume`)

  1.1. [Types of `Volume`](#Types-of-`Volume`)

2. [`Volume`을 사용한 컨테이너 간 데이터 공유](#`Volume`을-사용한-컨테이너-간-데이터-공유)

3. [`hostPath` Volume](#`hostPath`-Volume)

  3.1. [`hostPath` 볼륨 사용하는 거 보기](#`hostPath`-볼륨-사용하는-거-보기)

4. [Persistent Storage](#Persistent-Storage)

  4.1. [GCE Persistent Disk](#GCE-Persistent-Disk)

5. [기반 스토리지 기술과 파드 분리](#기반-스토리지-기술과-파드-분리)

  5.1. [Create `PV`](#Create-`PV`)

  5.2. [Create PVC](#Create-PVC)

  5.3. [Pod에서 PVC 사용하기](#Pod에서-PVC-사용하기)

  5.4. [PV 재사용](#PV-재사용)

6. [PV Dynamic Provisioning](#PV-Dynamic-Provisioning)

  6.1. [StorageClass](#StorageClass)

  6.2. [PVC에서 StorageClass 요청하기](#PVC에서-StorageClass-요청하기)

  6.3. [Dynamic Provisioning without StorageClass](#Dynamic-Provisioning-without-StorageClass)

  6.4. [PV Dynamic Provisioning Final](#PV-Dynamic-Provisioning-Final)

<br>

---
---

<br>

## 서론

파드 내부의 각 컨테이너는 고유하게 분리된 파일시스템을 가진다. 파일 시스템은 컨테이너 이미지에서 제공되기 때문이다.

새로 시작된 컨테이너가 이전에 실행되던 컨테이너와 같은 파드에 실행된다고 해도, 새로 시작한 컨테이너는 이전에 실행했던 컨테이너에 쓰여진 파일시스템의 어떤 것도 볼 수가 없다.

k8s는 파드와 동일한 사이클을 가진 스토리지 볼륨(Storage Volume)을 통해 한 파드 내에서 데이터를 보존하고 공유할 수 있게 한다.

<br>

---
---

<br>

## Introduce about `Volume`

`Volume`은 파드 스펙에서 정의되며, 독립적인 쿠버네티스 오브젝트가 아니기 때문에 자체적으로 생성, 삭제될 수 없다.

볼륨은 파드의 모든 컨테이너에서 사용 가능하지만, 접근하려는 컨테이너에서 ***각각 마운트 되어야 한다.***

볼륨이 없다면 아래와 같은 그림이 될 것이며, 서로의 데이터가 필요한데 공유되지 않으므로 사실상 아무 일도 하지 않는 컨테이너들이 될 것이다.

<br>

<img src="https://user-images.githubusercontent.com/37579681/113826216-de0d6e00-97bc-11eb-8607-9e125096a732.jpeg" width=350 height=500>

<br>

볼륨이 있다면, 이런 식의 데이터 공유로 인해 각 컨테이너가 제 역할을 할 것이다.

<br>

<img src="https://user-images.githubusercontent.com/37579681/113826220-df3e9b00-97bc-11eb-8ed3-56f0d5c93d54.jpeg" width=350 height=500>

<br>

---
---

<br>

### Types of `Volume`

참고로 저자도 볼륨 타입 중 절반도 모른다고 한다.

- emptyDir : 일시적인 데이터를 저장하는데 사용되는 간단한 빈 디렉터리
- hostPath : 워커 노드의 파일 시스템을 파드의 디렉토리로 마운트할 때 사용
- gitRepo : 깃 리포지터리의 콘텐츠를 체크아웃해 초기화한 볼륨
- nfs : NFS 공유를 파드에 마운트
- gcePersistentDisk || awsElasticBlockStore(EBS) || azureDisk : 클라우드 제공자의 전용 스토리지 마운트
- cinder, cephfs, iscsi, flocker, glusterfx, quobyte, rbd, flexVolume, vsphereVolume, photonPersistentDisk, scaleIO : 서로 다른 유형의 네트워크 스토리지 마운트
- configMap, secret, downwardAPI : k8s 리소스나 클러스터 정보를 파드에 노출하는데 사용하는 특별한 유형의 볼륨이다.
- persistentVolumeClaim : 사전에 혹은 동적으로 프로비저닝된 persistent storage를 사용하는 방법이다.

<br>

***이후의 내용들은 대부분 볼륨의 타입에 따른 사용 방법에 가깝다고 해도 과언이 아니다.***

<br>

---
---

<br>

## `Volume`을 사용한 컨테이너 간 데이터 공유

`emptyDir`볼륨은 동일 파드에서 실행 중인 컨테이너 간 파일을 공유할 때 유용하다.

또한 단일 컨테이너에서도 메모리에 넣기에 큰 데이터 세트의 정렬작업 등 임시 데이터를 디스크에 쓰는 목적인 경우에도 사용할 수 있다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
  - image: luksa/fortune
    name: html-generator
    volumeMounts:
    # html이라는 볼륨을 컨테이너의 /var/htdocs에 마운트
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    # html이라는 볼륨을 컨테이너의 /usr/share/nginx/html에 읽기 전용으로 마운트
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  # html이라는 단일 emptyDir 볼륨을 위의 컨테이너 두 개에 마운트
  volumes:
  - name: html
    emptyDir: {}
```

```bash
# Port forwarding
$ kubectl port-forward fortune 8080:80

# Nginx 서버 접근
$ curl http://localhost:8080
```

<br>

만약 디스크를 사용하는 볼륨이 아닌, 메모리를 사용하는 tmpfs 파일 시스템으로 볼륨을 만들고 싶으면 다음과 같이 지정한다.

```yaml
volumes:
  - name: html
    emptyDir:
      # 이 emptyDir의 파일들은 메모리에 저장될 것이다.
      medium: Memory
```

`gitRepo` 볼륨은 기본적으로 `emptyDir` 볼륨이고 파드가 시작되면 깃 레포(Repository)를 복제하고 특정 리비전을 체크아웃 해온다.

참고로 생성된 후에는 참조하는 레포와 동기화하지 않는다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gitrepo-volume-pod
spec:
  containers:
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    gitRepo:
      repository: https://github.com/luksa/kubia-website-example.git
      # 마스터 브랜치 체크아웃
      revision: master
      # 볼륨의 루트 디렉토리
      directory: .
```

<br>

***`SideCar` Container***

Git 동기화 프로세스가 Nginx 웹서버와 동일 컨테이너에서 실행되면 안 되며, 두 번째 컨테이너인 `Sidecar` 컨테이너에서 실행돼야 한다.

`SideCar` 컨테이너는 파드의 주 컨테이너의 동작을 보완하고 새로운 로직을 코드에 밀어 넣어 복잡성을 더하고 재사용성을 떨어뜨리는 일을 막는다.

<br>

---
---

<br>

## `hostPath` Volume

`hostPath` 볼륨은 노드 파일시스템의 특정 파일이나 디렉터리를 가리킨다.

동일 노드에 실행 중인 파드가 `hostPath` 볼륨의 동일 경로를 사용 중이면 동일한 파일이 표시된다.

이전의 `gitRepo`나 `emptyDir` 볼륨의 데이터는 파드가 종료되면 삭제되는 반면, `hostPath` 볼륨의 데이터는 삭제되지 않는다.

파드가 삭제되면 다음 파드가 호스트의 동일 경로를 가리키는 `hostPath` 볼륨을 사용하고, **이전 파드와 동일한 노드에 스케줄링된다는 조건에서** 새로운 파드는 이전 파드가 남긴 모든 항목을 볼 수 있다.

<br>

---

<br>

### `hostPath` 볼륨 사용하는 거 보기

``` bash
# 시스템 파드 조회 
$ kubectl get pod s --namespace kube-system

# 조회된 파드가 어떤 종류의 볼륨을 사용하는지 보기
# 위에서 찾은 파드를 넣자.
$ kubectl describe po f~~~ --namespace kube-system
```

노드의 로그파일이나 kubeconfig, CA 인증서를 접근하기 위해 대부분의 파드가 `hostPath` 볼륨을 사용한다.

<br>

---
---

<br>

## Persistent Storage

파드가 다른 노드에 재스케줄링된 경우에도 동일한 데이터를 사용해야 한다면?

이런 데이터는 어떤 클러스터 노드에서도 접근이 필요하기 때문에 NAS(Network-Attaced Storage) 유형에 저장되어야 한다.

NAS는 위에서 설명했던 볼륨 타입 중, gcePersistentDisk, awsElasticBlockStore, azureDisk, nfs 등이 있다.

하지만 모든 예제를 본 글에서 설명할 수 없기 때문에, GCE만 간단히 설명하도록 해보겠다.

<br>

***참고***

다른 방식이 더 궁금하신 분들은 **https://kubernetes.io/ko/docs/concepts/storage/volumes**를 참조하자.

<br>

---

<br>

### GCE Persistent Disk

```bash
# GCE 퍼시스턴트 디스크를 생성하자.(단, k8s Cluster가 있는 zone에 생성해야만 한다.)

# k8s 클러스터 조회
$ gcloud container clusters list
# 여기서는 Europe-west1-b 영역에 생성됐다고 가정하자

# mongodb 라는 이름을 가진 1GiB 크기의 디스크를 생성했다.
$ gcloud compute disks create --size=1GiB --zone=europe-west1-b mongodb
```

참고로, minikube를 사용하고 있다면 GCE Persistent Disk를 사용할 수 없기 때문에, hostPath Volume을 사용하는 yaml로 배포하자.

```yaml
apiVersion: v1
kind: Pod
metadata:
  # 위에서 생성한 mongodb와 상관없다는 거!
  name: mongodb
spec:
  volumes: 
  - name: mongodb-data
    gcePersistentDisk:
      # 위에서 명령어로 생성한 mongodb라는 이름과 같아야 한다.
      # 참고로 pd는 PersistentDisk를 뜻한다고 한다.
      pdName: mongodb
      fsType: ext4
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    # 위의 volumes와 같은 이름을 가져야 한다.
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
```

아마 이런 구조가 될 것이다.

<br>

<img src="https://user-images.githubusercontent.com/37579681/113826226-e1085e80-97bc-11eb-9008-62f0a1ada2c5.jpeg" width=500 height=250>

<br>

MongoDB 데이터베이스에 도큐먼트를 추가해서 데이터를 써보자.

```bash
$ kubectl exec -it mongodb mongo

> use mystore
> db.foo.insert({ name: 'foo' })

> db.foo.find()

# 파드를 삭제하고 다시 생성해보자.
$ kubectl delete pod mongodb

$ kubetl create -f mongodb-pod-gcepd.yaml

$ kubectl exec -it mongodb mongo

> use mystore
> db.foo.find()
# 새로운 파드가 이전 파드와 같이 동일한 GCE 디스크를 사용하고 있다!
```

<br>

---
---

<br>

## 기반 스토리지 기술과 파드 분리

사실 위와 같은 방식은 파드 개발자가 실제 NAS에 관한 지식을 가지고 있어야 한다.

이것은 k8s의 기본 철학인 **"인프라스트럭쳐의 세부 사항을 처리하지 않는다."**에 반한다.

따라서, 애플리케이션이 k8s 클러스터에 스토리지를 요청할 수 있도록 하기 위해

`PV, PersistentVolume`과 `PVC, PersistentVolumeClaim`이 도입되었다.

<br>

<img src="https://user-images.githubusercontent.com/37579681/113826232-e2398b80-97bc-11eb-89a5-ce7fa7044f94.jpeg" width=500 height=350>

<br>

개발자가 파드에 기술적인 세부 사항을 기재한 볼륨을 추가하는 대신 클러스터 관리자가 기반 스토리지를 설정하고 k8s API 서버로 PV 리소스를 생성해 k8s에 등록한다.

개발자가 파드에 Persistent Storage를 사용해야 하면 최소 크기 & 필요 접근 모드를 명시한 PVC manifest를 생성한 다음,

PVC manifest를 k8s API 서버에 게시하고 k8s는 적절한 PV를 찾아 볼륨에 바인딩한다.

그 다음 PVC는 파드 내부의 볼륨 중 하나로 사용되고 PVC 바인딩을 삭제해 릴리즈할 때까지 다른 사용자는 동일한 PV를 사용할 수 없다.

결론적으로는 이런 그림이 된다.

<br>

<img src="https://user-images.githubusercontent.com/37579681/113826239-e4034f00-97bc-11eb-9434-71e7d12c9b18.jpeg" width=500 height=350>

<br>

***참고***

PV와 PVC는 1:1 바인딩이다.

<br>

---

<br>

### Create `PV`

이전에 만든 GCE Persistent Disk를 다시 써서 PV를 만들어보자.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  # PV 사이즈
  capacity:
    storage: 1Gi
  # 단일 클라이언트의 읽기/쓰기 용이나
  # 여러 클라이언트를 위한 읽기 전용으로 마운트 된다.
  accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
  # PVC 해제 후 PV는 유지되어야 한다.
  persistentVolumeReclaimPolicy: Retain
  # 이전에 만든 GCE Persistent Disk를 기반으로 한다.
  gcePersistentDisk:
    pdName: mongodb
    fsType: ext4
```

kubectl create로 퍼시스턴트 볼륨을 생성하자.

```bash
$ kubectl get pv
# 아직 PVC를 생성하지 않았기 때문에 Available 상태이다.
```

<br>

***참고***

PV는 클러스터 level의 리소스이므로 특정 Namespace에 생성할 수 없다.

<br>

---

<br>

### Create PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  # 나중에 파드의 볼륨을 요청할 때 사용할 이름이다.
  name: mongodb-pvc
spec:
  resources:
    requests:
      stoorage: 1Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: ""
```

PVC가 생성되자마자 k8s는 적절한 PV를 찾고 binding 한다.

단, PV의 용량은 PVC의 요청을 수용할만큼 충분히 커야하고 또한 accessMode를 지원해야 한다.

```bash
$ kubectl get pvc
# mongodb-pv에 Bound 됐다고 나온다.
```

<br>

---

<br>

### Pod에서 PVC 사용하기
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  volumes: 
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: mongodb-pv
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    # 위의 volumes와 같은 이름을 가져야 한다.
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
```

아까 GCE Persistent Disk에 직접 연결한 것과 같은 결과를 낸다!

<br>

***참고: 이렇게 하면 뭐가 좋을까?***

<br>

<img src="https://user-images.githubusercontent.com/37579681/113826247-e5347c00-97bc-11eb-9938-2e7535ca7172.jpeg" width=500 height=350>

<br>

애플리케이션 개발자가 인프라스트럭쳐에서 스토리지를 가져오는 방식이 간접적으로 변했다!

이를 통해서 개발자는 기저에 사용된 실제 스토리지 기술을 알 수 없게 되었다.

또한 Pod와 PVC manifest는 의존성이 사라졌기 때문에 필요하다면 다른 k8s 클러스터에서도 사용할 수 있다.

<br>

---

<br>

### PV 재사용 

persistentVolumeClaimPolicy에는 3가지 정책이 있다.

이 3가지 정책에 따라 재사용하는 방식이 결정된다고 할 수 있다.

- Retain: Claim이 해제되더라도 볼륨과 data를 유지하도록 한다. 만약 재사용한다면 이전 파일을 어떻게 해야할지 결정해서 수동으로 처리해야 한다.

- Recycle : 볼륨의 데이터를 삭제하고 볼륨이 다시 claim 될 수 있도록 볼륨을 자동으로 사용가능하게 한다.

- Delete : 기반 스토리지를 삭제한다.(하지만 GCE PD에서는 사용불가능하다.)

<br>

---
---

<br>

## PV Dynamic Provisioning

클러스터 관리자는 여전히 실제 Storage를 미리 Provisioning 해야 한다.

이 작업을 자동으로 수행하기 위해서 Presistent Volume Provisioner를 배포하고 사용자가 선택한 PV의 타입을 하나 이상의 StorageClass Object로 정의할 수 있다.

사용자가 PVC에서 StorageClass를 참조하면 Provisioner가 Persistent Storage를 프로비저닝할 때 이를 처리한다.

즉, 관리자가 많은 PV를 프로비저닝하는 대신 하나 혹은 그 이상의 StorageClass를 정의하기만 하면 된다.

<br>

---

<br>

### StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
# PV 프로비저닝을 위해 사용되는 볼륨 플러그인
provisioner: kubernetes.io/gce-pd
# provisioner로 전달될 파라미터
parameters:
  type: pd-ssd
  zone: europe-west1-b
```

StorageClass 리소스는 PVC가 스토리지클래스에 요청할 때 어떤 프로비저너가 PV를 프로비저닝하는데 사용돼야 할지를 지정한다.

<br>

---

<br>

### PVC에서 StorageClass 요청하기

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  # PVC는 사용자 정의 StorageClass인 fast를 요청한다.
  storageClassName: fast
  resources:
    requests:
      storage: 100Mi
    accessModes:
      - ReadWriteOnce
```

PVC를 생성했다면 확인해보자.

```bash
# PVC 생성됐는지 확인
$ kubectl get pvc mongodb-pvc

# PV 자동으로 생성됐는지 확인
$ kubectl get pv

# GCE 퍼시스턴트 디스크가 생겼는지 확인
$ gcloud compute disks list
```

<br>

---

<br>

### Dynamic Provisioning without StorageClass

**기본 StorageClass**

아래의 명령어를 실행하면 standard(default) 라는 StorageClass를 볼 수 있다.

```bash
$ kubectl get sc
```

추가적인 정보를 더 보자

```bash
# GKE 클러스터의 standard StorageClass를 보자
$ kubectl get sc standard -o yaml
```

annotations를 살펴보면 이 스토리지 클래스는 기본 값이 된다는 것을 알 수 있다.

기본 스토리지 클래스는 PVC에서 명시적으로 어떤 스토리지 클래스를 사용할지 지정하지 않은 경우 PV를 동적 프로비저닝할 때 사용된다.

그러니깐 아래와 같이 pvc를 만들었을 때 말이다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc2
spec:
  resources:
    requests:
      storage: 100Mi
  accessModes:
  - ReadWriteOnce
```

위와 같은 PVC를 생성하면 기본 값으로 표시된 스토리지 클래스가 사용된다.

<br>

***참고***

만약 storageClassName 속성을 빈 문자열로 지정하지 않으면 미리 프로비저닝된 PV가 있더라도 동적 볼륨 프로비저너는 새로운 PV를 프로비저닝한다.

<br>

---

<br>

### PV Dynamic Provisioning Final

<br>

<img src="https://user-images.githubusercontent.com/37579681/113826252-e665a900-97bc-11eb-9215-f0f90b5d1a3a.jpeg" width=500 height=350>