## 개요

### 6.1

보안컨텍스트를 이용한 컨테이너 및 파드의 권한 설정

### 6.2

시크릿 생성하고 연결하기

### 6.3

서비스 어카운트를 생성하고 역할을 부여한후 파드를 서비스 어카운트에 연결하기

### 6.4

네트워크정책을 통해 트래픽 차단, 허용하기 테스트전 통신 테스트 해보기

### 6.5

네트워크정책을 통해 트래픽 차단, 허용 해보기



## yaml 파일들

exercise 6.1 1~6 **second.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secondapp
spec:
  containers:
  - name: busy
    image: busybox
    command:
      - sleep
      - "3600"
    securityContext:
      runAsUser: 2000
      allowPrivilegeEscalation: false                    
```

exercise 6.1 7~9 **second.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secondapp
spec:
  containers:
  - name: busy
    image: busybox
    command:
      - sleep
      - "3600"
    securityContext:
      runAsUser: 2000
      allowPrivilegeEscalation: false
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"] 
```

exercise 6.2 1~3 **secret.yaml**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: lfsecret
data:
  password: TEZUckAxbgo=
```

exercise 6.2 4~6 **second.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secondapp
spec:
  containers:
  - name: busy
    image: busybox
    command:
      - sleep
      - "3600"
    securityContext:
      runAsUser: 2000
      allowPrivilegeEscalation: false
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]
    volumeMounts:
    - name: mysql
      mountPath: /mysqlpassword
  volumes:
  - name: mysql
    secret:
      secretName: lfsecret
```

exercise 6.3 1~2 **serviceaccount.yaml**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secret-access-sa
```

exercise 6.3 3~6 **clusterrole.yaml**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-access-cr
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - get
  - list
```

exercise 6.3 7~8 **rolebinding.yaml**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-rb
subjects:
- kind: ServiceAccount
  name: secret-access-sa
roleRef:
  kind: ClusterRole
  name: secret-access-cr
  apiGroup: rbac.authorization.k8s.io
```

exercise 6.3 9~11 **second.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secondapp
spec:
  serviceAccountName: secret-access-sa
  securityContext:
    runAsUser: 1000
  containers:
  - name: webserver
    image: nginx
  - name: busy
    image: busybox
    command:
      - sleep
      - "3600"
    securityContext:
      runAsUser: 2000
      allowPrivilegeEscalation: false
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]
    volumeMounts:
    - name: mysql
      mountPath: /mysqlpassword
  volumes:
  - name: mysql
    secret:
      secretName: lfsecret
```

exercise 6.4 1 **allclosed.yaml**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

exercise 6.4 2~5 **second.yaml** (webserver 컨테이너 오류 나는것)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secondapp
spec:
  serviceAccountName: secret-access-sa
  securityContext:
    runAsUser: 1000
  containers:
  - name: webserver
    image: nginx
  - name: busy
    image: busybox
    command:
      - sleep
      - "3600"
    securityContext:
      runAsUser: 2000
      allowPrivilegeEscalation: false
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]
    volumeMounts:
    - name: mysql
      mountPath: /mysqlpassword
  volumes:
  - name: mysql
    secret:
      secretName: lfsecret
```

exercise 6.4 6~7 second.yaml (webserver 실행 유저 수정)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secondapp
  labels:
    example: second
spec:
  serviceAccountName: secret-access-sa
  #  securityContext:
  #  runAsUser: 1000
  containers:
  - name: webserver
    image: nginx
  - name: busy
    image: busybox
    command:
      - sleep
      - "3600"
    securityContext:
      runAsUser: 2000
      allowPrivilegeEscalation: false
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]
    volumeMounts:
    - name: mysql
      mountPath: /mysqlpassword
  volumes:
  - name: mysql
    secret:
      secretName: lfsecret
```



exercise 6.5 5 **allclosed.yaml** (컨테이너에서 나가는 트래픽은 허용)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
#  - Egress
```



exercise 6.5 7 **allclosed.yaml** (192.168.0.0./16 대역의 ip는 허용 <- 컨테이너 환경마다 다를 수 있음)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 192.168.0.0/16
#  - Egress
```



exercise 6.5 10 **allclosed.yaml** (특정 포트만 허용)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 192.168.0.0/16
    ports:
    - port: 80
      protocol: TCP
#  - Egress
```

