# kubectl

쿠버네티스 클러스터를 관리하는 동작 대부분은 `kubectl` 커맨드라인 인터페이스(CLI)로 실행할 수 있다.

## 사용법

> kubectl [command] [type] [name] [flags]

- `command` : 자원에 실행할 동작
- `type` :  자원 타입 (ex. pod, service, ingress)
- `name` : 자원 이름
- `flag` : 부가 옵션

## 생성

### 1. 명령어로 파드 생성

```bash
$ kubectl run nginx-server --image=nginx --restart=Never --port=80
pod/nginx-server created
```

- `—restart=Never` : 이 옵션을 넣어 Pod만 생성하도록 한다. (없을 경우, Deployment 생성)

### 2. YAML (템플릿)로 생성

```yaml
# nginx-pod.yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-server

spec:
  containers:
    - name: nginx-server
      image: nginx

      ports:
        - containerPort: 80
```

```bash
$ kubectl apply -f nginx-pod.yaml
pod/nginx-server created
```

## 조회 (`get`)
### 파드 조회
```bash
$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
nginx-server   1/1     Running   0          5s
```

### 서비스 조회
```bash
$ kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   46d
```

## 포트포워딩 (`port-forward`)

Pod 내에 nginx에 접속하려면 포트포워딩이 필요하다.

```bash
$ kubectl port-forward nginx-server 12345:80
Forwarding from 127.0.0.1:12345 -> 80
Forwarding from [::1]:12345 -> 80
...
```

<img src="https://github.com/user-attachments/assets/35a6281a-c68b-4023-9855-9c8118d25d66">

## 로그 (`logs -f`)

```bash
$ kubectl logs -f nginx-server
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
...
```

## 삭제 (`delete`)

```bash
$ kubectl delete pod nginx-server
pod "nginx-server" deleted
```
