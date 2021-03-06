### 6.1 볼륨 소개
 * 볼륨은 파드의 구성 요소로 컨테이너와 동일하게 파드 스펙에서 정의된다.
 * 볼륨은 파드의 모든 컨테이너에서 사용 가능하지만 접근하려는 컨테이너에서 각각 마운트돼야 한다.
 * 같은 볼륨을 두 개의 컨테이너에 마운트하면 컨테이너는 동일한 파일로 동작할 수 있다.
 * 다양한 볼륨 유형들이 있으며, 실제 스토리지 기술에 특화된 것들도 존재한다.

### 6.2 볼륨을 사용한 컨테이너 간 데이터 공유
 * 가장 간단한 볼륨 유형은 emptyDir 볼륨으로 빈 디렉터리로 실행된다.
 * emptyDir 볼륨은 동일 파드에서 실행 중인 컨테이너 간 파일을 공유할 때 유용하다.
 * 깃 리포지토리를 통하여 볼륨을 공유할 수 있다.

### 6.3 워커 노드 파일시스템의 파일 접근, 6.4 퍼시스턴트 스토리지 사용
 * hostPath 볼륨은 노드 파일시스템의 특정 파일이나 디렉터리를 가리킨다.
 * hostPath 볼륨의 콘텐츠는 파드가 종료되어도 살아남아있다.

### 6.5 기반 스토리지 기술과 파드 분리
![image](https://user-images.githubusercontent.com/13315923/114256716-3da28e00-99f6-11eb-8848-f11ef282ac3f.png)

### 6.6 퍼시스턴트볼륨의 동적 프로비저닝
 * 클러스터 관리자가 퍼시스턴트 볼륨을 생성하는 대신 퍼시스턴트볼륨 프로비저너를 배포하고 사용자가 선택 가능한 퍼시스턴트볼륨의 타입을 하나 이상의 스토리지클래스 오브젝트로 정의할 수 있다.
 * 쿠버네티스는 대부분 인기 있는 클라우드 공급자의 프로비저너를 포함하므로 관리자가 항상 프로비저너를 배포하지 않아도 된다.
