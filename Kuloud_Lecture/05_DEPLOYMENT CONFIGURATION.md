# 학습 목표

이 섹션이 끝나면 다음을 수행 할 수 있습니다.

- 영구 볼륨을 이해하고 생성합니다.
- 영구 볼륨 클레임을 구성합니다.
- 볼륨 액세스 모드를 관리합니다.
- 영구 저장소에 대한 액세스 권한이있는 애플리케이션을 배포합니다.
- 스토리지의 동적 프로비저닝에 대해 논의하십시오.
- 비밀 및 ConfigMap을 구성합니다.
- 배포 된 애플리케이션을 업데이트합니다.
- 이전 버전으로 롤백합니다.







<hr>

## Volume

pod spec에는 하나 이상의 volume과 사용가능 위치를 선언 할 수 있다.

pod spec에는 이름, 유형, 마운트 지점이 필요하다.

pod 내의 여러 컨테이너에서 동일한 볼륨을 사용할 수 있고 이를 통해 통신 할 수 있다.

또한, 하나의 볼륨을 여러 pod에서 사용 할 수있고 pod에는 accessModes 속성을 쓸 수 있다.

클러스터는 동일한 모드의 볼륨을 그룹화 한 다음 가장 작은것부터 큰것까지 크기별로 볼륨을 정렬한다.

충분한 크기의 볼륨이 일치 할 때까지 해당 액세스 모드 그룹의 각각에 대해 클레임을 확인한다.

accessModes의 종류

- RWO(Read Write Once) -단일노드에서 읽기 쓰기
- ROX(Read Only Many) - 여러 노드에서 읽기 전용
- RWX(Read Write Many)- 여러 노드에서 읽기 쓰기

![image-20210220100826719](C:\Users\LG\AppData\Roaming\Typora\typora-user-images\image-20210220100826719.png)

컨테이너에서 동일한 볼륨을 사용할 경우 /data/ 디렉토리를 사용한다.

MainApp은 볼륨을 읽고 쓰고 Logger는 볼륨을 읽기 전용으로 사용한다.

그리고 다른 볼륨의 /log/에 log에 기록한다.



#### Volume Spec 작성법

```
apiVersion: v1
kind: Pod
metadata:
    name: busybox
    namespace: default
spec:
    containers:
    - image: busybox
      name: busy
      command:
        - sleep
        - "3600"
      volumeMounts:
      - mountPath: /scratch
        name: scratch-volume
    volumes:
    - name: scratch-volume
            emptyDir: {}
```

볼륨 유형에서는 여러가지가 있는데,

#### emptyDir

컨테이너들끼리 데이터를 공유하기 위해서 볼륨을 사용하는것

최초 사용될때는 항상 이 볼륨이 비어 있기 때문에 emptyDir이라고 한다.



![image-20210220070318442](C:\Users\LG\AppData\Roaming\Typora\typora-user-images\image-20210220070318442.png)

만약에 container1 웹 역할   container2 백엔드 라고 했을때

웹 서버로 받은 파일을 마운트 된 볼륨에 저장을 해놓고 벡엔드에서 볼륨을 마운트 해두면 

두 서버가 저 볼륨하나를 로컬에 있는 파일처럼 사용하기때문에 두 서버가 파일을 주고받을 필요 없이 편하게 사용 할 수 있다

그리고 이 볼륨은 파드 안에 생성되기 때문에 파드에 문제가 발생을 해서 재 생성이 되면 데이터가 사라짐 그러므로 저 볼륨에는 일시적인 데이터를 저장할 필요가 있다. 

##### Share Volume 예시

```empty-dir.yaml
apiVersion: v1
kind: Pod
metadata:
	name: pod-volume-1
spec:
    containers:
    -	name: container1
        image: tmkube/init
        volumeMounts:
        -	name:empty-dir
        mountPath:/mount1
    -	name: container2
        image: tmkube/init
        volmeMounts:
        -	name: empty-dir
        mountPath:/mount2
    volumes:
    -	name: empty-dir
    	emptyDir:{}
```

컨테이너 두개가 있고 각각 empty-dir이라는 볼륨을 다른 경로로 마운트 해준 야물파일.

volumes에서 empty-dir이름의 볼륨을 emptyDir로 설정해주었다.



#### hostPath

이름 그대로 한 호스트, 파드들이 올라가있는 node의 path를 볼륨으로 사용하는것.

emptyDir과 다른점은 node의 path를 각각의 pod들이 마운트 하여 공유하기 때문에 

pod들이 죽어도 노드에 있는 데이터들은 사라지지 않는다.

이런 점에서 좋을 수 있지만

pod입장에서 한가지 큰 문제가 있다.

![image-20210220082105600](C:\Users\LG\AppData\Roaming\Typora\typora-user-images\image-20210220082105600.png)

만약 pod2가 죽어서 재 생성이 될때 꼭 해당 node1에 재 생성되지 않을 수도 있다

재생성 될때 스케쥴러가 자원 상황을 보고 다른 node에 pod를 재 생성하거나,

pod1에 장애가 생겨서 pod2를 다른 노드에 재생성 할 수도 있다.

node1에 장애가 생겨서 다른 노드에 재생성 될 수도 있다.

만약 pod2가 다른 노드로 옮겨졌을 때 기존 node1에 있는 볼륨에 마운트를 할 수 없게 된다.

hostPath기 때문에 자신이 올라가 있는 Node의 볼륨만 사용 할 수 있다.

하지만 굳이 방법을 찾는 다면,

새로운 노드가 추가가 될 때마다

직접 각 노드에있는 볼륨끼리 직접 마운트 하는 방법이 있다.

하지만 이 방법은 쿠버네티스가 해주는 작업은 아니다.

운영자가 직접 새로 만들어질때마다 연결을 해줘야한다.

어렵진 않지만 자동화를 시키는데 사람의 작업이 들어간다는 자체가 좋지 않다.

그럼 hostPath는 언제 사용하는가?

각각의 node에는 기본적으로 각 node자신을 위해  사용되는 파일이 있다.

시스템 파일이나 설정 파일 pod자신이 할당되어있는 host의 데이터를 읽거나 써야 할 때 사용한다.

**"pod의 데이터를 저장하기 위한 용도가 아니라 node에 있는 데이터를 pod에서 쓰기 위한 용도"**





```host.yml
apiVersion: v1
kind: Pod
metadata:
	name: pod-volume-2
spec:
	containers:
	-	name: container
		image: tmkube/init
		volumeMounts:
		-	name: host-path
			mountPath: /mount1
	volumes:
	-	name: host-path
		hostPath:
            path: /node-v
            type: Directory 
```

**hostPath의 hostPath속성의 path는 pod가 만들어지기 전에 사전에 미리 만들어져 있어야지 pod 생성 시 에러가 나지 않는다.**



#### PV(Persistent Volme) / PVC( Persistent Volume Claim)

 파드의 영속성있는 volume을 제공하기위한 개념

스토리지 수명을 포드와 구분하려면 영구 볼륨을 사용할 수 있습니다. 

![image-20210220084627162](C:\Users\LG\AppData\Roaming\Typora\typora-user-images\image-20210220084627162.png)

실제 volume들의 형태는 다양하다.

local volume도 있지만 외부에 원격으로 사용하는 volume도 있다.

그림처럼 아마존이나 깃에 연결하거나 nfs를 써서 다른 서버랑 연결 할 수도있어요

그리고 storage os같이 볼륨을 직접 만들과 관리 할 수도 있다.

이런것들을 각각  PV를 정의를 하고 연결을 한다.

근데 pod는 PV에 직접 연결하지 않고 PVC를 통해서 PV와 연결을 한다.

왜 PV에 직접 연결하지 않는가?

k8s는 볼륨 사용에 있어서 User 영역과 Admin영역으로 나눴다.

Admin은 k8s를 담당하는 k8s운영자

User는 pod에 서비스를 만들고 배포를 관리하는 서비스 담당자



##### pv/pvc목록 출력 하는  command

`$ kubectl get pv`
`$ kubectl get pvc`

```
#PV 생성 yml

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
			-{key:node,operator:ln, values:[node1]}
```

-{key:node,operator:ln, values:[node1]} 

PV에 연결된 pod들은  node1 부분처럼 라벨링 되어 있는 노드 위에만 무조건 만들어진다.

각각의 볼륨에 따라 속성이 다 다르다

그래서 이것들을 관리하는 admin이 PV를 만들어 놓으면

user는 이걸 사용하기 위해서 PVC 생성

accessModes의 종류

- RWO(Read Write Once) -단일노드에서 읽기 쓰기
- ROX(Read Only Many) - 여러 노드에서 읽기 전용
- RWX(Read Write Many)- 여러 노드에서 읽기 쓰기



```
#PVC 생성 yml

apiVersion: v1
kind : PersistentVolumeClaim
metadata:
	name: pvc-01
spec:
	accessModes:
		-	ReadWriteOnce
	resources:
		requests:
			storage:1G
	storageClassName:""
```

읽기 쓰기 권한이 있고 용량이 1G인 PV를 할당해 달라

""는 현재 만들어진 PV들 중에서 선택이 된다. 

클러스터는 동일한 accessModes의 볼륨을 그룹화 한 다음 가장 작은것부터 큰것까지 크기별로 볼륨을 정렬한다.

충분한 크기의 볼륨이 일치 할 때까지 해당 액세스 모드 그룹의 각각에 대해 클레임을 확인한다.

```pod.yml
apiVersion: v1
kind: Pod
metadata:
	name: pod-volume-3
spec:
	containers:
	-	name: container
		image: tmkube/init
		volumeMounts:
		-	name: pvc-pv
			mountPath: /volume
volumes:
	-	name : pvc-pv
		persistentVolumeClaim:
		claimName: pvc-01
```



# Configmap, Secret

Configmap, Secret을  사용해야하는 상황

![image-20210220104213389](C:\Users\LG\AppData\Roaming\Typora\typora-user-images\image-20210220104213389.png)

개발 환경과 상용환경이 있다.

- DEV(개발환경) Production(상용환경)

  A서비스는 일반 접근과 보안 접근을 지원하고 있다.

  그래서 개발환경에서는  보안접근을 해제할 수 있고, 접근 유저와 키를 세팅 할 수도 있다.

  근데 개발환경에서 상용환경으로 접근을 해야한다면,  값들이 바뀌어야한다.

  근데 이 값은 컨테이너 안에있는 서비스 이미지에 들어있는 값이라서 이걸 바꾼다는 거기 때문에  이 말은 즉, 개발환경과 상용환경의 컨테이너 이미지를 직접 관리하겠다는 의미다.

  근데 이 값 몇개때문에 큰 용량의 이미지를 별도로 관리하는건 너무 부담스러운 일이다.

  그래서 보통 이렇게 환경에 따라 변하는 값들은 외부에서 결정을 할 수 있게 하는데, 그걸 돕기 위한게 configMap과 Secret라는 오브젝트이다.

  내가 분리해야하는 일반적인 상수들을 모아서 configmap을 만들고, key와 같이 보안적인 관리가 필요할 때 configmap을 만든다.

  그리고 파드 생성 시에 이 두 오브젝트를 연결 할 수 있는데 , 이 컨테이너의 환경변수에 값들이 들어가게 된다.

  A서비스는 이 환경변수의 값을 읽어서 로직을 처리하게 되면, 개발환경이 원하는 역할을 할 수 있고, 그래서 저 이미지를 하나 만들어 놓으면 , 상용과 개발환경 둘다 사용할 수 있다.

  상용환경에서는 configmap과 Secret으로 값만 바꾸면 원하는 이미지로 사용가능



##### 사용방법

![image-20210220105448246](C:\Users\LG\AppData\Roaming\Typora\typora-user-images\image-20210220105448246.png)

configmap과  secret을 만들 때, 데이터로 상수를 넣을 수 있고, 파일을 넣을 수 있다.

파일을 넣을 땐 환경 변수로 세팅하는게 아니라 볼륨을 마운트 해서 사용할 수 있다.



먼저, 상수를 넣는법을 보면,

configmap은 key와 value로 구성이 되어 있다.

그래서 이렇게 필요한 상수들을 정의를 해 놓으면, 파드를 생성할 때 이 configmap을 가져와서 container안에 환경변수를 가져와서 실행을 할 수 있다.

그리고 Secret은 이름처럼 보안과 관련된 것들을 담는데, 주로 인증 키와 비번같은 것들을 담는다.

그리고 configmap과 다른점은 base64인코딩을 해서 만들어야한다는 건데,  이것은 secret의 value를 만들어야한다는 규칙이고 이게 파드로 주입이 될때는 자동으로 디코딩이 되서  환경변수에서 원래 값이 보이게 된다.

일반적인 오브젝트들은 k8s DB에 저장이 되는데, secert 은 메모리에 저장이 됩니다. 보안에 좋기 떄문에요! 그리고 컨피그맵은 무한정 데이터를 넣을 수 있지만 secret은 1메가 바이트만큼 넣을 수 있습니다. 그래서 시크릿을 너무 많이 만들면 시스템 자원에 영향을 끼치게 된다.

![image-20210220111136874](C:\Users\LG\AppData\Roaming\Typora\typora-user-images\image-20210220111136874.png)

##### configmap만들기

```
apiVersion: v1
kind: ConfigMap
metadata:
	name: cm-dev
data:
    SSH: False
    User: dev
```

이름 지정 후 데이터에 key-value 형태로 만든다

##### secert 만들기 

```
apiVersion: v1
kind: Secret
metadata:
	name: sec-dev
data:
    Key: MTIzNA==
```

값을 base64형태로 변환해서 넣으면 된다.

##### pod만들기

```
apiVersion: v1
kind: Pod
metadata:
	name: pod-1
spec:
	containers:
		- name: container
		  image: tmkube/init
	   	  envFrom:
		  - configMapRef:
			name: cm-dev
		  - secretRef:
			name: sec-dev
```

![image-20210220111210415](C:\Users\LG\AppData\Roaming\Typora\typora-user-images\image-20210220111210415.png)

##### 파일을 환경변수로 넣는 방법

이렇게 파일을 통으로 configmap에 담을 수 있는데 파일이름이 key가 되고,  파일 내용이 value가 된다

이때 key를 새로 정의를 해서 content만 넣어야한다.

`kubectl create configmap cm-file --from-file=./file.txt`

`kubectl create secret generic sec-file --from-file=./file.txt`

이렇게 만든다!! 시크릿을 통해서 파일 텍스트 안의 내용이 저 명령여로 base64로 자동인코딩이 되기 떄문에 두번 되지 않도록 주의!!!

```
apiVersion: v1
kind: Pod
metadata:
	name: file
spec:
	containers:
	- name: container
	  image: tmkube/init
	  env:
		- name: file
		  valueFrom:
		  	configMapKeyRef:
			  name: cm-file
			  key: file.txt
```

컨피그 맵의 이름은 cm-file이구 cm-file의 key에 대한 value를 넣게 됩니다.

#### 볼륨으로 마운트 하기

![image-20210220111900366](C:\Users\LG\AppData\Roaming\Typora\typora-user-images\image-20210220111900366.png)



파드를 만들때 컨테이너 안에 마운트 패스를 정의 하고 이 패스 안에 파일을 마운팅 할 수 있다

```
apiVersion: v1
kind: Pod
metadata:
	name: mount
spec:
	containers:
		- name: container
		  image: tmkube/init
		  volumeMounts:
		  - name: file-volume
			 mountPath: /mount
	volumes:
	- name: file-volume
	  configMap:
		name: cm-file
```

볼륨안에 configmap에 이름은 cm파일!!



**파일과 볼륨마운트 방식의 차이**

**만약 configmap의 contetnt를 바꾸게되면?**

파일 방식은 configmap을 변경해도 한번 보내면 끝이라 파드의 content는 변함이 없다.

파드가 죽어서 재 생성이 돼야  새 값이 들어간다



마운트 방식은 원본파일과 마운트 된거기 때문에 내용이 변하면 파드의 마운팅된 내용도 변한다.