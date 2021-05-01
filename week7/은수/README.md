# Week 07 - Kubernetes Deployment

쿠버네티스에서 실행되는 기본 애플리케이션 구조는 외부 클라이언트가 서비스를 통해 파드에 요청을 보내고, 파드는 레플리케이션 컨트롤러 또는 레플리카 셋에 의해 제어됩니다. 이 때 애플리케이션의 새로운 버전을 배포하려고 한다면 어떻게 해야 할까요? 쿠버네티스에서 선언적으로 애플리케이션은 업데이트하는 리소스, 디플로이먼트(Deployment)를 알아보도록 합시다

# 파드에서 실행중인 애플리케이션 업데이트

먼저 위 예시에서 파드를 업데이트 하는 방법은 다음과 같은 두 가지가 있습니다.

1. 기존 파드를 모두 삭제한 다음 새 파드를 시작합니다

2. 새로운 파드를 시작하고, 기동하면 기존 파드를 삭제합니다. 새 파드를 모두 추가한 다음 한꺼번에 기존 파드를 삭제하거나 순차적으로 새 파드를 추가하고 기존 파드를 점진적으로 제거해 이 작업을 수행할 수 있습니다

위 두 가지 방법을 모두 수동으로 처리할 수 있습니다. 1의 경우 기존의 파드를 제어하는 레플리카셋을 제거하고 업데이트된 애플리케이션 이미지를 사용하는 파드 템플릿이 포함된 레플리카셋을 생성하면 됩니다.

2의 경우 새로운 레플리카셋을 먼저 만들고, 레플리카셋이 스핀업한 파드가 모두 기동되면 서비스가 새로 기동된 파드를 가리키도록 셀렉터를 변경한 뒤 기존 레플리카셋을 삭제하면 됩니다. 또 점진적으로 파드를 교체해나가는 롤링 업데이트(rolling update)도 가능하지만 이를 수동으로 하려면 여러 단계를 거쳐 명령을 실행해야 합니다.

다행히 쿠버네티스에서는 kubectl 의 `rolling-update` 명령으로 롤링 업데이트를 간단하게 수행할 수 있습니다.

```sh
$ kubectl rolling-update OLD_CONTROLLER_NAME NEW_CONTROLLER_NAME --image=NEW_CONTAINER_IMAGE
```

위 처럼 기존 컨트롤러 이름과 새롭게 생성할 컨트롤러의 이름을 지정한 다음, 원래 이미지를 교체할 새 이미지를 명령줄 인자로 제공하면 됩니다. 이렇게 하면 쿠버네티스는 새로운 컨트롤러를 생성하고 롤링 업데이트에 필요한 모든 단계의 명령을 자동으로 수행합니다. 이 때 기존의 컨트롤러에 의해 새로 생성된 파드가 제어되지 않도록 두 컨트롤러에 `deployment` 라벨 셀렉터를 추가하고, 이 라벨 셀렉터에 의해 새로 생성된 파드가 기존의 컨트롤러에게 제어되지 않도록 구분합니다.

## kubectl rolling-update를 더 이상 사용하지 않는 이유

`rolling-update` 명령 단 한번으로 롤링 업데이트를 수행할 수 있다는 점은 아주 편리한 것 같습니다. 그러나, `rolling-update` 로 애플리케이션을 업데이트하는 것은 더 이상 사용하지 않습니다.

그 이유는 첫 번째로 롤링 업데이트의 모든 단계를 kubectl 클라이언트가 쿠버네티스 API를 호출해서 실행한다는 것 입니다. 만약 kubectl 이 업데이트를 수행하는 동안 네트워크 연결이 끊어지게 된다면, 업데이트 프로세스는 중간에 중단되게 됩니다.

또 다른 이유는 이러한 동작은 선언적(declarative)이지 않고 명령적(imperative)이란 것입니다. 쿠버네티스에서는 의도하는 시스템의 상태를 선언(declare)하고 쿠버네티스가 그것을 달성할 수 있는 가장 좋은 방법을 찾아냄으로써 스스로 그 상태를 달성하도록 하는 것을 추구합니다. 즉, 쿠버네티스에 파드를 추가하거나 초과된 파드를 직접 제거하라고 지시하지 말고 그저 원하는 레플리카 수를 선언함으로써 애플리케이션을 업데이트 할 수 있어야 합니다.

마찬가지로 파드 스펙에서 원하는 이미지 태그를 변경하고 쿠버네티스가 파드를 새 이미지로 실행하는 새로운 파드로 교체할 것입니다. 바로 이것이 현재 쿠버네티스에서 애플리케이션을 배포하는 가장 좋은 방법인 디플로이먼트(Deployment)라는 새로운 리소스를 도입하게 된 이유입니다.

# 애플리케이션의 선언적 업데이트: 디플로이먼트

디플로이먼트는 low-level 개념으로 간주되는 레플리케이션 컨트롤러 또는 레플리카셋을 통해 수행하는 대신 애플리케이션을 배포하고 선언적으로 업데이트하기 위한 high-level의 리소스입니다.

디플로이먼트를 생성하면 레플리카셋 리소스가 그 아래에 생성되고, 이 레플리카셋에 의해 파드가 생성되고 관리됩니다.

## 디플로이먼트 생성

디플로이먼트는 다음과 같이 생성할 수 있습니다.

```yaml
# kubia-deployment-v1.yaml
apiVersion: apps/v1
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
  selector:
    matchLabels:
      app: kubia
```

```sh
$ kubectl create -f kubia-deployment-v1.yaml -record
```

**Note**: create할 때 반드시 `--record` 옵션을 포함해야 합니다. 이는 명령어를 개정 이력(revision history)에 기록해 나중에 유용하게 사용할 수 있습니다.

이렇게 디플로이먼트를 생성한 뒤 디플로이먼트의 상태를 확인하기 위해 다음과 같이 `rollout` 이란 특별한 명령을 사용할 수 있습니다.

```sh
$ kubectl rollout status deployment kubia
deployment kubia successfully rolled out
```

디플로이먼트의 롤아웃이 성공적으로 수행됬다고 나오니, 디플로이먼트와 덩달아 생성되는 파드와 레플리카셋의 이름을 살펴보면 이름에 파드 템플릿의 해시값이 포함되는 것을 확인할 수 있습니다(ex. pod 이름: `kubia-1506449474-otnnh`, replicaset 이름: `kubia-1506449474`). 이는 디플로이먼트가 파드 템플릿의 각 버전마다 하나씩 여러 개의 레플리카 셋을 만들기 때문입니다.

## 디플로이먼트 업데이트

디플로이먼트는 파드 템플릿을 수정하기만 하면 쿠버네티스가 실제 시스템 상태를 리소스에 정의된 상태로 만드는 데 필요한 모든 단계를 수행합니다. 이 때 새로운 상태를 달성하는 방법은 디플로이먼트에 구성된 디플로이먼트 전략에 의해 결정됩니다.

### 사용 가능한 디플로이먼트 전략

디플로이먼트는 기본적으로 RollingUpdate라는 전략을 갖는데, 이전 파드를 하나씩 제거하고 동시에 새 파드를 추가해 전체 프로세스에서 애플리케이션을 계속 사용할 수 있도록 하고 서비스 다운타임이 없도록 합니다. 단 애플리케이션이 이전 버전과 새 버전을 동시에 실행할 수 있는 경우에만 이 전략을 사용해야 합니다.

다른 방법으로 Recreate라는 전략을 사용하여 새 파드를 만들기 전 이전 파드를 모두 삭제하는 방법을 적용할 수 있습니다. 이 전략은 RollingUpdate처럼 중단없이 애플리케이션을 업데이트 하지 않기 때문에 짧은 다운타임이 발생합니다.

### 롤링 업데이트 시작

롤아웃을 시작하려면 다음과 같이 파드 컨테이너에 사용되는 이미지를 변경하면 됩니다.

```sh
# 기존 kubia:v1 이미지를 kubia:v2 로 변경합니다
$ kubectl set image deployment kubia nodejs=luksa/kubia:v2
```

위 명령을 실행하면 새로운 레플리카셋이 생성되고 그 후 천천히 파드가 교체됩니다. 롤링 업데이트 후 레플리카셋을 조회하면 기존 레플리카셋과 새 레플리카셋을 나란히 볼 수 있습니다.

```sh
$ kubectl get rs
NAME              DESIRED   CURRENT   AGE
kubia-1506449474  0         0         24m
kubia-1581357123  3         3         23m
```

`rolling-update` 명령은 롤링 업데이트 후 기존 컨트롤러를 삭제하는 반면, 디플로이먼트는 롤링 업데이트가 끝나도 이전 레플리카셋을 삭제하지 않았는데, 그 이유는 롤백을 가능하게 하기 위함입니다.

## 디플로이먼트 롤백

디플로이먼트는 다음과 같이 `rollout undo` 명령으로 롤백할 수 있습니다.

```sh
$ kubectl rollout undo deployment kubia
```

위 명령을 수행하면 디플로이먼트는 이전 버전으로 롤백합니다. 이게 가능한 이유는 디플로이먼트가 revision hisotry 를 유지하기 때문입니다. 이러한 이력은 기본 레플리카셋에 저장되는데, 롤아웃이 완료되면 이전 레플리카셋은 삭제되지 않으므로 이전 버전뿐만 아니라 모든 버전으로 롤백할 수 있습니다.

```sh
# 특정 revision 으로 롤백하려면 --to-revision 옵션을 부여햡니다
$ kubectl rollout undo deployment kubia --to-revision=1
```

Revision history 는 `rollout history` 명령으로 조회할 수 있습니다

```sh
$ kubectl rollout history deployment kubia
deployments "kubia":
REVISION    CHANGE-CAUSE
2           kubectl set image deployment kubia nodejs=luksa/kubia:v2
3           kubectl set image deployment kubia nodejs=luksa/kubia:v3
```

**Note**: apps/v1 버전에서 디플로이먼트의 기본 revision history limit 은 10입니다.

# References

- Kubernetes in Action
