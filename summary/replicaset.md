## Replication Controller (레플리케이션 컨트롤러)

> 지정한 숫자만큼의 파드가 항상 클러스터 안에서 실행되도록 관리

### Example

파드 2개를 명시해둔 레플리케이션 컨트롤러가 있으면 장애나 다른 이유로 파드가 2개보다 적을 경우,
새로운 파드를 실행해서 파드 개수를 2개로 맞춘다. 반대로 파드가 더 많아지면 삭제해서 2개로 유지한다.

> 💡
>
> 다만 요즘은 비슷한 역할을 하는 **레플리카세트**를 사용한다.

---

## Replicaset (레플리카 세트)

> 레플리케이션 컨트롤러 + **집합 기반의 셀럭터**를 지원

### Example

레플리케이션 컨트롤러는 셀럭터가 등호 기반이므로 레이블을 선택할 때 `=`, `!=` 를 사용한다. <br>
다만, 집합 기반의 셀렉터는 `in`, `notin`, `exists` 같은 연산자를 지원한다.

### Usage

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
spec:
  template: # 레플리카세트가 어떤 파드를 실행할지에 관한 정보 설정
    metadata:
      name: nginx-replicaset  # 파드 이름
      labels:
        app: nginx-replicaset # 오브젝트 식별 레이블
    spec:
      containers: # 컨테이너 구체적인 명세 설정
        - name: nginx-replicaset
          image: nginx
          ports:
            - containerPort: 80
  replicas: 3                 # 파드 유지 개수 (기본값 : 1)
  selector: # 어떤 레이블의 파드를 선택해서 관리할지를 설정
    matchLabels: # spec.template.metadata.labels 와 필드 설정이 동일해야 함
      app: nginx-replicaset
```

```shell
$ kubectl apply -f nginx-replicaset.yaml
replicaset.apps/nginx-replicaset created

$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-replicaset-b4n45   1/1     Running   0          52s
nginx-replicaset-cn82j   1/1     Running   0          52s
nginx-replicaset-v5bj6   1/1     Running   0          52s
```

replicas를 3으로 설정했기 때문에 3개의 nginx 파드가 생성됨을 알 수 있다. <br>
만약 파드를 1개 제거하면 어떻게 될까?

```shell
$ kubectl delete pod nginx-replicaset-v5bj6
pod "nginx-replicaset-v5bj6" deleted

$ kubectl get pods                          
NAME                     READY   STATUS    RESTARTS   AGE
nginx-replicaset-b4n45   1/1     Running   0          3m1s
nginx-replicaset-cn82j   1/1     Running   0          3m1s
nginx-replicaset-sd7kc   1/1     Running   0          7s
```

파드 개수를 3개로 유지하기 위해 `nginx-replicaset-sd7kc`라는 파드 1개를 추가로 실행하는 것을 볼 수 있다.

### 레플리카세트와 파드의 연관 관계

레플리카세트와 파드를 한꺼번에 삭제할 때는 아래 명령어를 사용한다.

```shell
$ kubectl delete replicaset 컨테이너이름
```

`--cascade=false` 옵션을 사용하면 레플리카세트가 관리하는 파드에 영향을 끼치지 않고 레플리카세트만 삭제할 수 있다.

```shell
$ kubectl delete replicaset nginx-replicaset --cascade=false
warning: --cascade=false is deprecated (boolean value) and can be replaced with --cascade=orphan.
replicaset.apps "nginx-replicaset" deleted
```

> 위 경고 메시지를 보면 `--cascade=false`가 지원 중단되었기 때문에 `--cascade=orphan`으로 대체한다.

| 이전 방식                  | 새 권장 방식                | 설명                            |
|------------------------|------------------------|-------------------------------|
| `--cascade=false`      | `--cascade=orphan`     | ReplicaSet 삭제 시, 관련된 Pod은 남겨둠 |
| `--cascade=true` (기본값) | `--cascade=background` | 관련된 리소스도 함께 삭제됨               |


