# Deployment (디플로이먼트)

> 상태가 없는 앱을 배포할 때 사용하는 가장 기본적인 컨트롤러

- 디플로이먼트는 레플리카세트를 관리하면서 앱 배포를 더 세밀하게 관리
- 배포 기능을 세분화한 것
    - `롤링 업데이트`, `배포 도중 잠시 멈추기`, `재배포`, `이전 버전으로 롤백`

## Usage

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels: # selector.matchLabels의 하위 필드와 .metadata의 하위 필드가 같아야 한다.
      app: nginx-deployment
  template:
    metadata:
      name: nginx-deployment
      labels:
        app: nginx-deployment
    spec:
      containers:
        - name: nginx-deployment
          image: nginx
          ports:
            - containerPort: 80
```

```shell
$ kubectl apply -f nginx-deployment.yaml
deployment.apps/nginx-deployment created
$ kubectl get deploy,rs,pods
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   3/3     3            3           16s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-757fc7f4dd   3         3         3       16s

NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-757fc7f4dd-bc4mm   1/1     Running   0          16s
pod/nginx-deployment-757fc7f4dd-dnjp4   1/1     Running   0          16s
pod/nginx-deployment-757fc7f4dd-gdq6v   1/1     Running   0          16s
```

- **deploy** : 디플로이먼트
- **rs** : 레플리카세트
- **pods** : 파드

---

## nginx-deployment의 컨테이너 이미지 업데이트하기

1. `kubectl set` 명령으로 직접 컨테이너 이미지 지정

```shell
$ kubectl set image deployment/nginx-deployment nginx-deployment=nginx:1.9.1
deployment.apps/nginx-deployment image updated
```

```shell
$ kubectl get deploy,rs,pods                
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   3/3     3            3           27m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-645dcbd767   3         3         3       2m31s

NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-645dcbd767-bbx6g   1/1     Running   0          2m31s
pod/nginx-deployment-645dcbd767-gfpn6   1/1     Running   0          2m10s
pod/nginx-deployment-645dcbd767-hk5w2   1/1     Running   0          2m17s
```

컨테이너 이미지를 업데이트하면서 nginx-deployment가 관리하는 `nginx-deployment-645dcbd767` 새로운 레플리카세트가 생성된다.

```shell
$ kubectl get deploy nginx-deployment -o=jsonpath="{.spec.template.spec.containers[0].image}{'\n'}"
nginx:1.9.1
```

`kubectl set image` 명령 실행 후 버전이 변경된 것을 확인할 수 있다.

2. `kubectl edit` 명령으로 현재 파드의 설정 정보를 연 다음 컨테이너 이미지 정보를 수정

```shell
$ kubectl edit deploy nginx-deployment

# AS-IS
spec:
      containers:
      - image: nginx:1.9.1
        imagePullPolicy: Always
        name: nginx-deployment
        ports:
        - containerPort: 80
        
# TO-BE
spec:
      containers:
      - image: nginx:1.10.1
        imagePullPolicy: Always
        name: nginx-deployment
        ports:
        - containerPort: 80
        
deployment.apps/nginx-deployment edited
```

```shell
$ kubectl get deploy nginx-deployment -o=jsonpath="{.spec.template.spec.containers[0].image}{'\n'}"
nginx:1.10.1
```

위 명령을 통해 nginx 버전이 1.10.1로 바뀐 것을 확인할 수 있다.

3. 처음 적용했던 템플릿의 컨테이너 이미지 정보를 수정한 다음 `kubectl apply` 명령을 실행해서 변경

---

## 디플로이먼트 롤백하기

`kubectl rollout history deploy 디플로이먼트이름` 으로 컨테이너 이미지 변경 내역을 확인할 수 있다.

```shell
$ kubectl rollout history deploy nginx-deployment
deployment.apps/nginx-deployment 
REVISION  CHANGE-CAUSE
2         <none> # kubectl set image 를 통해 변경한 리비전 (nginx:1.9.1)
3         <none> # kubectl edit 을 통해 변경한 리비전 (nginx:1.10.1)
```

2, 3의 리비전을 확인할 수 있다. <br>
특정 리비전의 상세 내용을 확인하려면 `--revision=리비전숫자` 옵션을 사용한다.

```shell
$ kubectl rollout history deploy nginx-deployment --revision=3
deployment.apps/nginx-deployment with revision #3
Pod Template:
  Labels:       app=nginx-deployment
        pod-template-hash=849fbbdd67
  Containers:
   nginx-deployment:
    Image:      nginx:1.10.1
    Port:       80/TCP
    Host Port:  0/TCP
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
```

리비전 #2로 되돌리기 위해선 `kubectl rollout undo deploy 디플로이먼트이름` 을 실행한다.

```shell
$ kubectl rollout undo deploy nginx-deployment
deployment.apps/nginx-deployment rolled back
```

```shell
$ kubectl get deploy,rs,pods
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   3/3     3            3           10h

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-645dcbd767   3         3         3       9h
replicaset.apps/nginx-deployment-849fbbdd67   0         0         0       9h

NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-645dcbd767-hcg9g   1/1     Running   0          29s
pod/nginx-deployment-645dcbd767-pd25t   1/1     Running   0          23s
pod/nginx-deployment-645dcbd767-spsdw   1/1     Running   0          18s
```

#1 kubectl set image 를 통해 변경한 리비전 (nginx:1.9.1)의 레플리카세트와 그가 관리하는 파드들로 다시 되돌려진 것을 확인할 수 있다. <br>

그렇다면 리비전 숫자는 #2로 되돌려져 있을까?

```shell
$ kubectl rollout history deploy nginx-deployment
deployment.apps/nginx-deployment 
REVISION  CHANGE-CAUSE
3         <none>
4         <none>
```

리비전 #2는 #4로 변경된 것을 확인할 수 있다. <br>

> 리비전 숫자만으로는 해당 리비전에서 어떤 변경이 있었는지 파악하기 어렵다. <br>
> 이럴 때는 CHANGE-CAUSE에 설명을 추가해서 관리할 수 있다.

```yaml
metadata:
  name: nginx-deployment
  labels:
    app: nginx-deployment
  annotations:
    kubernetes.io/change-cause: version 1.10.1 # annotation에 'kubernetes.io/change-cause'을 추가한다. 
```

```shell
$ kubectl apply -f nginx-deployment.yaml
deployment.apps/nginx-deployment configured
```

```shell
$ kubectl rollout history deploy nginx-deployment
deployment.apps/nginx-deployment 
REVISION  CHANGE-CAUSE
3         <none>
4         <none>
5         version 1.10.1
```

위처럼 입력한 버전 숫자를 메시지로 출력할 수 있다.

---

## 파드 개수 조정하기

실행 중인 디플로이먼트의 파드 개수를 조정하려면 `kubectl scale` 명령 + `--replicas` 옵션에 파드 개수를 입력해서 조정한다.

```shell
$ kubectl scale deploy nginx-deployment --replicas=5
deployment.apps/nginx-deployment scaled

$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-757fc7f4dd-27cfj   1/1     Running   0          13m
nginx-deployment-757fc7f4dd-8dvb2   1/1     Running   0          24s
nginx-deployment-757fc7f4dd-8jkxw   1/1     Running   0          13m
nginx-deployment-757fc7f4dd-ctqpx   1/1     Running   0          24s
nginx-deployment-757fc7f4dd-dkk8t   1/1     Running   0          13m
```

---

## 배포 정지, 재개, 재시작하기

`kubectl rollout` 명령을 이용해서 진행 중인 배포를 잠시 멈췄다가 다시 시작할 수 있다.

### 배포 정지 : rollout pause

```shell
$ kubectl rollout pause deployment/nginx-deployment
deployment.apps/nginx-deployment paused
```

### 배포 재개 : rollout resume

```shell
$ kubectl rollout resume deploy/nginx-deployment
deployment.apps/nginx-deployment resumed
```

### 배포 재시작 : rollout restart

```shell
$ kubectl rollout restart deploy/nginx-deployment
deployment.apps/nginx-deployment restarted
```
