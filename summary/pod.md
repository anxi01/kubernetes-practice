# Pod (파드)

쿠버네티스는 실제로 파드라는 단위로 컨테이너를 묶어 관리하므로 보통 여러 개의 컨테이너로 구성된다. <br>
파드로 컨테이너 여러 개를 한꺼번에 관리할 때는 컨테이너마다 역할을 부여할 수 있다.

## 사용법

```yaml
# nginx-pod.yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-server      # 파드 이름 설정
  labels:
    app: nginx-server     # 오브젝트를 식별하는 레이블 설정 (앱 컨테이너, nginx-server로 식별)

spec:
  containers:
    - name: nginx-server  # 컨테이너 이름 설정
      image: nginx
      ports:
        - containerPort: 80
```

### 템플릿 실행

```shell
kubectl apply -f nginx-pod.yaml
```

---

## 생명 주기

- **Pending** : 파드를 생성하는 중
- **Running** : 파드 안 컨테이너 실행 중 (실행 중 or 시작 or 재시작)
- **Succeeded** : 정상 실행 종료 -> 재시작 X
- **Failed** : 정상적으로 실행 종료되지 않은 경우
- **Unknown** : 파드의 상태를 확인할 수 없는 경우

---

## kubelet으로 컨테이너 진단하기

> 컨테이너가 실행된 후, kubelet이 컨테이너를 주기적으로 진단한다.

### Probe

컨테이너에서 kubelet에 의해 주기적으로 수행되는 진단이다.

- **livenessProbe**
    - 컨테이너가 실행됐는지 확인
    - 진단 실패 시, kubelet이 컨테이너 종료 및 재시작
- **readinessProbe**
    - 컨테이너 실행 후 실제 서비스 요청에 응답할 수 있는지 진단
    - 진단 실패 시, 엔드포인트 컨트롤러는 해당 파드에 연결된 모든 서비스를 대상으로 엔드포인트 정보 제거
    - readinessProbe를 지원하는 컨테이너는 실행된 다음 실제 트래픽을 받을 준비가 되었을 때 트래픽을 받을 수 있다.
        - ex) 자바 애플리케이션 : 프로세스 시작 후 앱이 초기화될 때까지 시간이 걸릴 때 유용 ‼️

### 진단 Handler(핸들러) 및 진단 결과

컨테이너가 구현한 핸들러를 kubelet이 호출해서 실행한다.

#### Handler

- **ExecAction** : 컨테이너 안에 지정된 명령을 실행, 종료 코드가 0일 때 Success 진단
- **TCPSocketAction** : 컨테이너에 지정된 IP와 포트로 TCP 상태를 확인, 포트가 열려있으면 Success 진단
- **HTTPGetAction** : 컨테이너 안에 지정된 IP, 포트, 경로로 HTTP GET 요청 보냄, 응답 상태 코드가 200 ~ 400 사이면 Success 진단

#### 진단 결과

- **Success** : 컨테이너 진단 성공
- **Failure** : 컨테이너 진단 실패
- **Unknown** : 진단 자체 실패해서 컨테이너 상태를 알 수 없음

---

## 초기화 컨테이너 (Init Container)

> 초기화 컨테이너는 앱 컨테이너 실행 전 파드를 초기화한다.
>
> 보통 보안상 이유로 앱 컨테이너 이미지와 같이 두면 안되는 앱의 소스코드를 별도 관리할 때 유용 ‼️

```yaml
# nginx-pod.yaml
spec:
  initContainers: # 초기화 컨테이너 설정
    - name: init-nginx-server
      image: nginx
      command: [ 'sh', '-c', 'sleep 2; echo nginx on!' ]  # 2초 대기 후 "nginx on" 메시지 출력

  containers:
    - name: nginx-server
      image: nginx
      ports:
        - containerPort: 80
```

---

## CPU, 메모리 자원 할당하기

파드를 설정할 때 파드 내 각 컨테이너마다 리소스를 할당하기 위해 **limits, requests** 필드를 사용한다.

- **requests** : 최소 자원 요구량, 만약 요구하는 만큼의 여유 자원이 없다면 Pending 상태로 클러스터 내 자원 여유가 생길 때까지 대기
- **limits** : 자원 최대 제한량, 만약 트래픽이 몰릴 경우, 같은 노드에 실행된 다른 컨테이너가 모두 영향을 받아 모든 서비스에 영향을 끼칠 수 있다.

```yaml
# nginx-pod.yaml
containers:
  - name: nginx-server
    ...
    resources:
      requests:
        cpu: "0.1"
        memory: "200M" # 보통 바이트 단위인 E(Exa), P(Peta), T(Tera), G(Giga), M(Mega), K(Kilo) 

      limits:
        cpu: "0.5" # 코어 하나의 50%연산 능력만 사용하도록 제한
        memory: "1G"
```

CPU의 경우, 1을 코어 하나의 연산 능력을 온전히 사용할 수 있음을 나타낸다.
즉, CPU 코어 하나의 연산 능력을 기준으로 설정한다.

---

## 환경변수 설정하기

```yaml
# nginx-pod.yaml
env:
  - name: TESTENV
    value: "testvalue01"
  - name: HOSTNAME
    valueFrom:
      fieldRef:
        fieldPath: spec.nodeName
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
  - name: POD_IP
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
  - name: CPU_REQUEST
    valueFrom:
      resourceFieldRef:
        containerName: nginx-server
        resource: requests.cpu
  - name: CPU_LIMIT
    valueFrom:
      resourceFieldRef:
        containerName: nginx-server
        resource: limits.cpu
```

- **name** : 환경 변수 이름
- **value** : 문자열, 숫자 형식의 값 설정
- **valueFrom** : 다른 곳에서 값을 참조해 설정
    - **fieldRef** : 파드의 현재 설정 내용을 값으로 선언
        - **fieldPath** : `.fieldRef`에서 어디서 값을 가져올 건지를 지정
- **resourceFieldRef** : CPU, 메모리 사용 할당량 정보 참조
    - containerName : 환경변수 설정 가져올 컨테이너 이름 설정
    - resource : 어떤 자원의 정보를 가져올지 설정

---

## 파드 구성 패턴

### 사이드카 패턴

사이드카 패턴은 원래 사용하려던 기본 컨테이너의 기능을 **확장/강화하는 용도의 컨테이너**를 추가하는 패턴이다. <br>
기본 컨테이너는 원래 목적의 기능에만 충실하도록 구성하고, 나머지 공통 부가 기능들은 사이드카 컨테이너를 추가해서 사용한다.

<img src="https://github.com/user-attachments/assets/befa5b36-c056-442a-8a89-2abdd2b09d50">

### 앰배서더 패턴

앰버서더 패턴은 파드 안에서 **프록시 역할**을 하는 컨테이너를 추가하는 패턴이다.

<img src="https://github.com/user-attachments/assets/d2538af2-20a9-4ec4-93da-b356af95100e">

웹 서버 컨테이너는 캐시에 localhost로 접근하고 실제 외부 캐시 중 어디로 접근할지는 프록시 컨테이너가 처리한다.

### 어댑터 패턴

어댑터 패턴은 파드 외부로 노출되는 정보를 표준화하는 어댑터 컨테이너를 사용하는 패턴이다.

<img src="https://github.com/user-attachments/assets/5495013f-1d1a-45c5-939d-e9b816058505">
