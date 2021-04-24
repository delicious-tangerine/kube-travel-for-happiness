# `ConfigMap & Secret`

## 목차
0. [서론](#서론)

1. [컨테이너에 명령줄 인자 전달](#컨테이너에-명령줄-인자-전달)

  1.1. [k8s에서 명령과 인자 재정의](#k8s에서-명령과-인자-재정의)

2. [컨테이너의 환경변수 설정](#컨테이너의-환경변수-설정)

3. [ConfigMap](#ConfigMap)

  3.1. [Create ConfigMap](#Create-ConfigMap)

  3.2. [컨피그맵 항목을 환경변수로 사용하기](#컨피그맵-항목을-환경변수로-사용하기)

  3.3. [컨피그맵 볼륨](#컨피그맵-볼륨)

  3.4. [애플리케이션을 재시작하지 않고 애플리케이션 설정 업데이트](#애플리케이션을-재시작하지-않고-애플리케이션-설정-업데이트)

4. [Secret](#Secret)

  4.1. [Create Secret](#Create-Secret)
  
  4.2. [Pod에서 Secret 사용하기](#Pod에서-Secret-사용하기)

<br>

---
---

<br>

## 서론

대부분의 애플리케이션은 빌드된 애플리케이션 자체에 포함되지 않아야 하는 설정이 필요하다.

일반적으로 컨테이너화된 애플리케이션에서는 명령줄 인수로 애플리케이션에 설정을 넘겨주거나, 환경변수를 이용한다.

도커 컨테이너 내부에 있는 설정 파일을 이용할 수도 있지만 이는 까다롭다.

따라서 컨테이너화된 애플리케이션에서는 세 가지 방법 정도를 사용할 수 있다.

- 컨테이너에 명령줄 인수 전달
- 각 컨테이너를 위한 사용자 정의 환경변수 지정
- 특수한 유형의 볼륨을 통해 설정 파일을 컨테이너에 마운트

<br>

---
---

<br>

## 컨테이너에 명령줄 인자 전달

**ENTRYPOINT & CMD**

- ENTRYPOINT는 컨테이너가 시작될 때 호출되는 명령어
- CMD는 ENTRYPOINT에 전달되는 인자를 정의한다.

보통 Dockerfile에서 위의 두 명령어를 흔하게 볼 수 있다.

**shell & exec**

- shell 형식 - ENTRYPOINT node app.js (명령을 shell로 호출한다.)
- exec 형식 - ENTRYPOINT ["node", "app,js] ***(프로세스를 셸 내부가 아닌, 직접 실행한다.)***

```Dockerfile
FROM ubuntu:latest
RUN apt-get update; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
ENTRYPOINT ["/bin/fortuneloop.sh"]
CMD ["10"]
```

<br>

---

<br>

### k8s에서 명령과 인자 재정의

```yaml
kind: Pod
spec:
  containers:
  - image: some/image
    command: ["/bin/command"]
    args: ["arg1", "arg2", "arg3"]
```

대부분 사용자 정의 인자만 지정하고 명령을 재정의하는 경우는 거의 없다.

***참고로 command와 args는 파드 생성 이후에 업데이트 할 수 없다.***

<br>

---
---

<br>

## 컨테이너의 환경변수 설정

k8s에서는 파드의 각 컨테이너를 위한 환경변수 리스트를 지정할 수 있다.

파드 레벨이 아니라 컨테이너 정의 안에 설정할 수 있고 애플리케이션에서 사용할 수 있다(process.env.INTERVAL, System.getenv("INTERVAL"))

```yaml
kind: Pod
spec:
  containers:
  - image: luksa/fortune:env
    env:
    - name: INTERVAL
      value: "30"
    name: html-generator
...
```

<br>

고정 값 이외에도 이미 정의된 환경변수나 기타 기존 변수를 참조할 수도 있다.

<br>

```yaml
env:
- name: FIRST_VAR
  value: "foo"
- name: SECOND_VAR
  # foobar가 된다.
  value: "$(FIRST_VAR)bar"
```

<br>

파드 정의에 하드코딩된 값을 가져오는 것은 효율적이지만, 프로덕션과 개발을 위해 서로 분리된 파드 정의가 필요하다는 것을 뜻한다.

따라서 뒤에서 배우는 ConfigMap으로 설정을 분리해보자.

<br>

---
---

<br>

## ConfigMap

k8s에서는 설정 옵션을 컨피그맵이라 부르는 별도 오브젝트로 분리할 수 있다.

컨피그맵은 짧은 문자열에서 전체 설정 파일에 이르는 값을 가지는 키/값 쌍으로 구성된 맵이다.

애플리케이션은 필요한 경우 k8s REST API를 통해 컨피그맵의 내용을 직접 읽을 수 있지만, 반드시 필요한 경우가 아니라면 애플리케이션은 쿠버네티스와는 무관하도록 유지해야 한다.

<br>

---

<br>

### Create ConfigMap

``` bash
# 간단한 configmap 생성
$ kubectl create configmap fortune-config --from-literal=sleep-interval=25

# 여러 개의 --from-literal 인자를 추가한다.
$ kubectl create configmap my-config
=> --from-literal=foo=bar --from-literal=bar=baz --from-literal=one=two

# configmap을 살펴보자
$ kubectl get configmap fortune-config -o yaml

# file을 읽어와 저장한다.
$ kubectl create configmap my-config --from-file=config-file.conf

$ kubectl create configmap my-config --from-file=customkey=config-file.conf
```

다양한 옵션을 사용하면 아래처럼 만들 수도 있다.

<img src="https://user-images.githubusercontent.com/37579681/115952169-7e82c280-a51f-11eb-9240-e5444693a6bd.jpeg">

<br>

---

<br>

### 컨피그맵 항목을 환경변수로 사용하기

***환경변수를 컨피그맵에서 가져오는 파드 선언***

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-env-from-configmap
spec:
  containers:
  - image: luksa/fortune:env
    # 환경변수 설정
    env:
    - name: INTERVAL
      # 고정 값 대신 컨피그맵에서 값을 가져와 초기화
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
...
```

***컨피그맵의 모든 항목을 환경변수로 전달***

CONFIG_FOO와 CONFIG_BAR라는 두 개의 환경변수가 컨테이너 안에 존재하게 된다.

참고로, 대시가 있으면 올바른 환경변수 이름이 아니기 때문에 건너뛸 수 있으니 주의하자.

```yaml
spec:
  containers:
  - image: some-image
    envFrom:
    # 환경변수 앞에 붙을 접두사 지정
    - prefix: CONFIG_
      configMapRef:
        # my-config-map이라는 컨피그맵 참조
        name: my-config-map
...
```

***컨피그맵 항목을 명령줄 인자로 전달***

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-args-from-configmap
spec:
  containers:
  - image: luksa/fortune:args
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    args: ["$(INTERVAL)"]
```

<br>

---

<br>

### 컨피그맵 볼륨

환경변수 또는 명령줄 인자로 옵션을 전달하는 것은 일반적으로 짧은 변숫값에 대해서 사용된다.

컨피그맵은 모든 설정 파일을 포함할 수 있고, 이런 파일들을 컨테이너에 노출시키려면 컨피그맵 볼륨을 사용하면 된다.

컨피그맵 볼륨은 파일로 컨피그맵의 각 항목을 노출하고 컨테이너에서 실행 중인 프로세스는 이 파일 내용을 읽어 각 항목의 값을 얻을 수 있다.

```conf
# my-nginx-config.conf
server {
  listen          80;
  server_name     www.kubia-example.com;

  gzip on;
  gzip_types text/plain application/xml;

  location / {
    root  /usr/share/nginx/html;
    index index.html index.htm;
  }
}
```

이런 형식으로 디렉토리를 구성해보자

<img src="https://user-images.githubusercontent.com/37579681/115952173-80e51c80-a51f-11eb-8613-fcec0ab65ad8.jpeg">

```bash
$ kubectl create configmap fortune-config --from-file=configmap-files

$ kubectl get configmap fortune-config -o yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-configmap-volume
spec:
  containers:
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    ...
    - name: config
      # 컨피그맵 볼륨을 마운트하는 위치
      mountPath: /etc/nginx/conf.d
      readOnly: true
    ...
  volumes: 
  - name: config
    configMap:
      # 이 볼륨은 fortune-config 컨피그맵을 참조한다.
      name: fortune-config
```

아마 이런 구조가 될 것이다.

<br>

<img src="https://user-images.githubusercontent.com/37579681/115952174-82164980-a51f-11eb-96ff-8ea24239999b.jpeg">

<br>

하지만 이런 구조라면 원하지 않는 파일도 볼륨에 노출된다.

볼륨에 특정 컨피그맵 항목을 노출하고 싶다면?

아래와 같이 볼륨의 `items` 속성을 사용하자.

```yaml
volumes:
- name: config
  configMap:
    name: fortune-config
    items:
    # 볼륨에 포함할 항목을 조회해 선택
    - key: my-nginx-config.conf
      path: gzip.conf
```

생각해보니, 볼륨을 마운트할 때 원 이미지에 있던 디렉토리 내의 파일들에 접근할 수 없게 된다.

기존 파일들을 지키면서 파일을 마운트하려면 어떻게 해야할까?

정답은 전체 볼륨을 마운트하는 것이 아닌, `subPath` 속성을 히용하여 파일이나 디렉터리 하나를 볼륨에 마운트하는 것이다.

```yaml
spec:
  containers:
  - image: some/image
    volumeMounts:
    - name: myvolume
      # 디렉토리가 아니라 파일 마운트
      mountPath: /etc/someconfig.conf
      # 전체 볼륨을 마운트하는 대신 myconfig.conf 항목만 마운트
      subPath: myconfig.conf
```

참고로 컨피그맵 볼륨의 파일 권한도 설정할 수 있다.(기본은 644)

```yaml
volumes:
- name: config
  configMap:
    name: fortune-config
    defaultMode: "6600"
```

<br>

---

<br>

### 애플리케이션을 재시작하지 않고 애플리케이션 설정 업데이트

환경변수 또는 명령줄 인수를 이용할 때는 프로세스가 실행되고 있는 동안에 업데이트 할 수 없지만, 컨피그맵을 사용해 볼륨으로 노출하면 파드를 다시 만들거나 컨테이너를 다시 시작할 필요 없이 업데이트 할 수 있게 된다.

k8s가 컨피그맵 볼륨에 있는 모든 파일은 동시에 업데이트 된다.

컨피그맵이 업데이트되면 k8s는 새 디렉터리를 생성하고 모든 파일을 해당 디렉터리에 쓴 다음, 심볼릭 링크가 새 디렉터리를 가리키도록 해, 모든 파일을 한 번에 효과적으로 변경한다.

<br>

---
---

<br>

## Secret

보안이 유지돼야 하는 자격 증명과 개인 암호화 키와 같은 민감한 정보가 포함되어 있는 경우 시크릿이라는 오브젝트를 사용해야 한다.

- 환경변수로 시크릿 항목을 컨테이너에 전달
- 시크릿 항목을 볼륨 파일로 노출

쿠버네티스는 시크릿에 접근해야 하는 파드가 실행되고 있는 노드에만 개별 시크릿을 배포한다.

또한 물리 저장소에 기록되지 않도록 한다.

모든 파드에는 secret 볼륨이 자동으로 연결되어 있다.

참고로 secret에는 세 종류가 있다.
- 도커 레지스트리를 사용하기 위한 docker-registry
- TLS 통신을 위한 tls, generic

<br>

---

<br>

### Create Secret

fortune-https 이름을 가진 generic 시크릿을 생성해보자.

```bash
$ kubectl create secret generic fortune-https --from-file=https.keys
=> --from-file=https.cert --from-file=foo
```

참고로 컨피그맵은 일반 텍스트로 표시되고, 시크릿 항목은 Base64로 인코딩된다.

모든 민감한 데이터가 바이너리 형태가 아니기 떄문에, 쿠버네티스는 시크릿의 값을 StringData로 필드로 설정할 수 있게 해준다.

```yaml
kind: Secret
apiVersion: v1
stringData:
  foo: plain text
data:
  https.cert: LS~~~~
  https.key: LS~~~
```

stringData 필드는 쓰기 전용이다. 따라서 가져올 떄 표시되지는 않는다.

secret 볼륨을 통해 시크릿을 컨테이너에 노출하면, 디코딩할 필요 없이 사용할 수 있다.

<br>

---

<br>

### Pod에서 Secret 사용하기
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-https
spec:
  containers:
  - image: luksa/fortune:env
    name: html-generator
    env:
    - name: INTERVAL
      valueFrom:
      configMapKeyRef:
        name: fortune-config
        key: sleep-interval
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
    - name: certs
      mountPath: /etc/nginx/certs/
      readOnly: true
    ports:
    - containerPort: 80
    - containerPort: 443
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: fortune-config
      items:
      - key: my-nginx-config.conf
        path: https.conf
  - name: certs
    secret:
      secretName: fortune-https
```

결론적으로 이런 그림이 된다.

<img src="https://user-images.githubusercontent.com/37579681/115952176-83477680-a51f-11eb-88c1-051de410e4a4.jpeg">
