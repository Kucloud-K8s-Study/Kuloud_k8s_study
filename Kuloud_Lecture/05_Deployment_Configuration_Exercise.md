### 학습 목표

- persistent volumes의 이해 및 생성
- persistent volume claims 설정
- volume 액세스 모드를 관리
- persistent volumes에 대한 액세스 권한이있는 애플리케이션을 배포
- storage의 동적 프로비저닝
- secrets 및 ConfigMap을 구성
- 배포 된 애플리케이션을 업데이트
- 이전 버전으로 롤백



> ⚠ 예제 코드는 LFD259 강의의 코드와 달라요..ㅠㅠ



## ConfigMap, Secret

- 환경에 따라 변하는 값이 있을 경우 각 환경마다 컨테이너 이미지를 가지고 있으면 부담이 됨
- 이런 환경에 따라 변할 필요가 있는 값들을 별도로 외부에서 결정할 수 있게 해줌
- ConfigMap: 상수 값
- Secret: 보안 Key와 같이 보안적인 관리가 필요한 값

- Key-Value 쌍으로 이루어져 있음
- Secret은 값을 넣을 때 Base64로 인코딩
  - Pod에 설정될 때는 Decoding



**Secret의 보안적 요소**

- Memory에 저장
- 1Mb 까지 생성할 수 있음



### Key-value 상수 값을 환경변수로 정의

**ConfigMap**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-dev
data:
  SSH: 'false'    # Key-Value는 모두 String이기 때문에 ''으로 감싸야함, 안하면 에러
  User: dev
```



**Secret**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sec-dev
data:
  Key: MTIzNA==    # base64 인코딩, 안하면 에러
```



**Pod**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-file
spec:
  containers:
  - name: container
    image: kubetm/init
    env:
    - name: file-c
      valueFrom:
        configMapKeyRef:
          name: cm-file
          key: file-c.txt
    - name: file-s
      valueFrom:
        secretKeyRef:
          name: sec-file
          key: file-s.txt
```





### 파일을 환경변수로 정의

- 파일명이 Key, 파일 내용이 Value



**ConfigMap**

`echo "Content" >> file-c.txt`

`kubectl create configmap cm-file --from-file=./file-c.txt`



**Secret**

`echo "Content" >> file-s.txt`

`kubectl create secret generic sec-file --from-file=./file-s.txt`



**Pod**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-file
spec:
  containers:
  - name: container
    image: kubetm/init
    env:
    - name: file-c    # Pod 내에서는 name을 별도로 지정해 줌
      valueFrom:
        configMapKeyRef:
          name: cm-file
          key: file-c.txt
    - name: file-s
      valueFrom:
        secretKeyRef:
          name: sec-file
          key: file-s.txt
```





### 파일을 마운팅하는 방법

**ConfigMap**

`echo "Content" >> file-c.txt`

`kubectl create configmap cm-file --from-file=./file-c.txt`



**Secret**

`echo "Content" >> file-s.txt`

`kubectl create secret generic sec-file --from-file=./file-s.txt`



**Pod**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-mount
spec:
  containers:
  - name: container
    image: kubetm/init
    volumeMounts:
    - name: file-volume
      mountPath: /mount
  volumes:
  - name: file-volume
    configMap:    # volume 내부에 ConfigMap 설정
      name: cm-file
```





### 환경변수 설정 방식과 Volume Mount 방식의 차이점

- 이미 Pod에 설정된 상태에서 ConfigMap/Secret 값이 바뀌게 된다면?

  - **환경변수 설정 방식은 한번 주입을 하면 끝이기 때문에 Pod의 환경변수 값은 영향이 없음**

    - **Pod가 재생성 된다면 영향을 받음**

    

  - **Volume Mount 방식은 Pod 내부의 설정 값도 바뀜**





## Volume

- 데이터를 안정적으로 관리하기 위해 사용
- K8s Cluster와 분리되어 관리



### emptyDir

- **컨테이너들끼리** 데이터를 공유하기 위해 볼륨을 사용
- 최초 볼륨은 비어 있음
- Pod 내부에 생성되기 때문에 Pod가 삭제되면 같이 삭제
- `mountPath`가 달라도 `name`이 같으면 같은 볼륨을 공유

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: pod-volume-1 
spec:
    containers:
    - name: container1
        image: tmkube/init
        volumeMounts:
        - name:empty-dir
        mountPath:/mount1
    - name: container2
        image: tmkube/init
        volmeMounts:
        - name: empty-dir
        mountPath:/mount2
    volumes:
    - name: empty-dir
    	emptyDir:{}
```



### hostPath

- Pod들이 올려져있는 Node의 Path를 볼륨으로 사용
- Pod가 죽어도 Node에 있는 데이터는 유지
- 만약 Pod가 죽어서 재생성 될 떄 같은 노드에 다시 생성될거라는 보장이 없음
  - 노드마다 같은 경로의 볼륨을 생성해서 각 노드의 볼륨들 끼리 마운팅해주면 해결
    - 관리자가 직접 구성해줘야 함 (자동화 X)
- 특정 노드에서만 필요로 되어지는 데이터(시스템 파일, 설정 파일 등)가 있을 경우 
- `type: Directory`: hostPath에 필요한 노드의 경로는 Pod 생선 전에 미리 만들어 줘야함 
  - 없을 경우 에러
- `type: DirectoryOrCreate`: 해당 경로가 없으면 디렉토리 생성

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: pod-volume-2
spec:
    containers:
    - name: container
        image: tmkube/init
        volumeMounts:
        - name: host-path
            mountPath: /mount1
    volumes:
    - name: host-path
        hostPath:
            path: /node-v
            type: Directory 
```



### PV(Persistent Volme) / PVC(Persistent Volme Claim)

|                              PV                              |                             PVC                              |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| :point_right: Admin(K8s 운영자)이 관리<br />:point_right: 로컬 볼륨 / 외부 볼륨이 있음 | :point_right: ​User(Pod에 서비스를 만들고 배포하는 담당자)가 관리<br />:point_right: ​Pod는 PV에 바로 연결되는 것이 아니라 PVC를 통해 연결 |

- 한 PVC와 연결된 PV는 다른 PVC와 연결 할 수 없음



#### PVC 마운팅 과정

1. **Admin이 PV 정의 생성**

   - Local Volume 사용 시

   ```yaml
   apiVersion: v1
   kind Persistent Volume
   metadata:
       name: pv-01
   spec:
       capacity:
           storage: 1G
       accessModes:
           - ReadWriteOnce
       local:
           path: /node-v
           #hostpath처럼 호스트에 있는 local path사용
       nodeAffinity:
       required:
           nodeSelectorTerms:
           - matchExpressions:
               # 연결된 Pod는 항상 node1에서만 생성
               -{key:node,operator:ln, values:[node1]}
   ```

2. **User가 PVC 생성**

   ```yaml
   apiVersion: v1
   kind : PersistentVolumeClaim
   metadata:
       name: pvc-01
   spec:
       accessModes:
           - ReadWriteOnce
       resources:
           requests:
               storage:1G
       storageClassName:""
   ```

3. **K8s가 PVC와 PV 연결**

   - PV에서 설정한 spec(capacity, accessMode)에 맞춰 자동으로 연결

4. **User가 Pod 생성 시 PVC 마운팅**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
       name: pod-volume-3
   spec:
       containers:
       - name: container
           image: tmkube/init
           volumeMounts:
           - name: pvc-pv
               mountPath: /volume
   volumes:
       - name : pvc-pv
           persistentVolumeClaim:
           claimName: pvc-01
   ```

   

### AccessMode

- **RWO**: ReadWriteOnce
- **ROM**: ReadOnlyMany
- **RWM**: ReadWriteMany

- 각각의 볼륨들 마다 지원해주는 모드가 다름



### Dynamic Provisioning

- User가 PVC를 생성하면 그거에 맞춰 PV를 생성
- 사전에 미리 Dynamic Provisioning을 지원해주는 Storage Solution 설치가 필요
  - StorageClass Object가 생성



**StorageClass**

- StorageClass를 활용해서 동적으로 PV를 만들 수 있음

- 추가로 더 만들거나 Default 값을 설정 할 수  있음

- PVC를 만들 때 StorageClassName을 설정하면 해당 Storage Solution과 연결된 PV와 볼륨이 생성

  - StorageClassName을 `""` 이렇게 설정하면 k8s가 알아서 알맞은 PV를 찾아 연결되고 볼륨이 먼저 만들어지지않음

    - 볼륨은 Pod와 연결되면 만들어짐 

    



### Status

- **Available**: 최초 PV가 생성 될 때

- **Bound**: PVC와 연결됨
- **Released**: PVC가 삭제될 때
- **Failed**: PV와 데이터간 연결에 문제가 생겼을 경우



### ReclaimPolicy

- **Retain**

  - 데이터는 보존됨
  - 다른 PVC에 다시 연결 할 수는 없음
  - Default: 별도로 설정하지 않았을 경우 기본 값

  

- **Delete**

  - PVC가 삭제되면 PV 삭제
  - Volume의 종류의 따라 데이터 보존은 다름
  - 재사용 불가
  - StorageClass를 사용하여 생성하였을 경우 Default 값

  

- **Recycle**

  - PV의 상태가 Available이 되고 PVC와 다시 연결 가능
  - 데이터가 삭제됨

