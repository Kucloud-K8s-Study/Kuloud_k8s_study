## 06 SECURITY



### 주제

- API 응답 흐름에 대한 설명
- 인증 규칙 구성
- 인증 정책 검토

- 네트워크 정책으로 네트워크 트래픽 제한



### 들어가기 전

- 쿠베같은 시스템에는 보안은 크고 복잡한 주제이기 때문에,
  쿠베의 맥락에서 보안을 다루는 몇가지 개념을 다룰 것
- **API 서버의 인증 측면으로 초점**을 맞추고, 
  kubeadm 으로  kubernetes 클러스트를 부트 스트랩 할 때
  기본 구성 **ABAC 및 RBAC** 와 같은 항목을 다룰 것
- 앞으로 수정가능하고 최종수락과 거절할 수 있는 
  **승인 제어 시스템**을 다룰것

- 본격적인 API 객체인 보안콘텐츠 그리고 보안 및 정책을
  사용하여 포드들을 더 타이트하게 보호하는 방법을 포함고
  추가적으로 다른 개념들을 다룰 것

- 최종적으로 네트워크 정책들을 봄
  허나 기본적으로 우린 다른 네임스페이스에서 모든 포드를 통한 
  트래픽을 허용하는 네트워크 정책을 사용하지 않는 경향이 있음
  실제로는 Ingress 규칙을 정의해 제한 가능함
  Flannel 혹은 Calico 는 네트워크 정책을 구현할지 정할 수 있음



*ABAC (속성 기반 접근 제어) 및 RBAC (역할 기반 접근 제어)

![image-20210220103711635](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210220103711635.png)

- 사용자, 접근되는 자원, 환경 조건의 **속성에 기반한** 접근 제어

![image-20210220103720802](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210220103720802.png)

- 사용자가 가지는 역할들, 그 역할을 맡은 사용자에게 허용되는 접근 **규칙기반** 접근제어





### 1. API에 접근하기

k8s 클러스터에 작업 수행을 위한 API 액세스 주요 3단계

- Authentication (인증)

- Authorization (ABAC or RBAC) (권한부여)
- Admission Control (승인제어)

![image-20210220104751487](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210220104751487.png)



### 2. 인증

중요한 포인트

- 인증은 인증서, 토큰 및 기본 인증을 간단하게 구성함
- 사용자는  API를 통해 생성되지 않지만, 외부 시스템에서 관리되어야 함
- 시스템 계정은 API에 액세스하는 프로세스로 사용됨



다음 두 가지이상 고급 인증 메커니즘

- 웹훅: 베어러 토큰 확인 및 외부 OpenID 제공자와의 연결에 사용됨

  

- kube-apiserver 
  선택한 인증 메커니즘에 따라 설정해야하는 구성 옵션 하위 집합의 4가지 예

  - **--basic-auth-file**
  - **--oidc-issuer-url**
  - **--token-auth-file**
  - **--authorization-webhook-config-file**



### 3. 권환부여

3가지 메인 인증 모듈

- ABAC
- RBAC
- Webhook



kube-apiserver 시작 옵션

- **--authorization-mode=ABAC**
- **--authorization-mode=RBAC**
- **--authorization-mode=Webhook**
- **--authorization-mode=AlwaysDeny**
- **--authorization-mode=AlwaysAllow**



### 4. ABAC, RBAC and Webhook Modes

- ABAC (속성기반 접근제어) - 최초의 권한부여 모델

  - 아래는 bob 사용자에게 foobar 네임 스페이스의 pods를 읽을 수 있는 권한 부여

  ```
  {
      "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
      "kind": "Policy",
      "spec": {
          "user": "bob",
          "namespace": "foobar",
          "resource": "pods",
          "readonly": true     
      }
  }
  ```

  

- RBAC (역할기반 접근제어) - 현재 기본 권한부여 모델

  RBAC 프로세스 요약

  - 네임스페이스 생성 또는 결정
  - 사용자에 대한 인증서 자격 증명 만들기
  - 컨텍스트를 사용한 사용자의 자격 증명을 네임스페이스로 설정
  - 예상되는 작업 세트에 대한 역할 만들기
  - 사용자 역할 바인딩
  - 사용자에게 제한된 엑세스 권한 유무 확인
    

- Webhook Modes

  - 웹훅은 HTTP 콜백으로, 어떤 일이 발생할때 발생하는 HTTP POST
  - HTTP POST 를 통한 간단한 알림
  - 웹훅을 구현하는 웹 앱은 특정 상황이 발생하면 URL에 메세지 게시



### 5. 인증 컨트롤러

- 요청에 의해 생성되는 개체의 콘텐츠에 액세스 할 수 있는 sw
- 콘텐츠 수정, 유효성 검사
- 잠재적으로 요청 거부 가능



- kube-apiserver 1.13.1 릴리스 부터 승인 컨트롤러 추가됨

- 승인 컨트롤러는 실행 중에 전달된 목록 대신 바이너리로 컴파일 됨

- 활ㆍ비활성화하려면 다음 옵션을 전달하여 활ㆍ비활성화하려는 플러그인 변경 가능

  ```
  --enable-admission-
  plugins=Initializers,NamespaceLifecycle,LimitRanger
  --disable-admission-plugins=PodNodeSelector
  ```

- 첫 컨트롤러는 API 요청의 동적 수정을 허용하는 이니셜 라이저로 뛰어난 유연성 제공

- 각 승인 컨트롤러 기능은 설명서에서 확인



### 6. 보안 콘텐츠

- pod 내의 pod과 컨테이너에는 컨테이너에서 실행되는 프로세스가 
  수행할 작업을 제한하기 위한 특정 보안 제약이 주어짐
- 예를 들어) 
  프로세스의 UID, Linux 기능 및 파일 시스템 그룹을 제한함

- 이 보안 제한을 보안 컨텍스트라 칭함
- 전체 포드 or 컨테이너별 정의할 수 있고, 리소스 매니페스트에서 추가 섹션을 표시함
- 차이점은 **Linux 기능이 컨테이너 수준에서 설정**된다는 점

- 예를 들자면) 
  컨테이너가 루트 사용자에게 프로세스를 실행할 수 없다는 정책을 시행하려는 경우 
  아래와 같은 포드 보안 컨텍스트를 추가 함

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - image: nginx
    name: nginx
```

- 다음 포드를 만들 때 컨테이너가 루트로 실행하려고하는데 허용되지 않는다고 경고 표시
  -> **포드는 실행 되지 않음**

```
$ kubectl get pods
NAME   READY  STATUS                                                 
RESTARTS  AGE
nginx  0/1    container has runAsNonRoot and image will run as 
root  0         10s
```



### 7. 팟 보안 정책

- PSP (PodSecruityPolicies): 보안 컨텍스트의 적용을 자동하기 위해 정의

- PSP 는  PSP API 스키마에 따라 표준 k8s 매니페스트를 통해 정의

- 예를 들어)
  클러스터의 컨테이너가 루트 사용자가 실행되는 것을 원하지 않을 시
  해당 효과에 대한 PSP 정의 가능

  또한,  컨테이너가 권한이 부여되지 않도록하거나 네트웍 네임스페이스 
  또는 호스트 PID 네임 스페이스를 사용 가능



- OPA (Open Policy Agent): PSP 이외의 방법, 

  - 통합된 도구 및 정책 프레임 워크 셋 제공
  - 위를 통해 **클라우드 배포에 대해 단일 구성지점**으로 사용 가능

  - K8s 내부에 허용 컨트롤러로 배포 가능하며, OPA는 이를 적용하거나 변경 가능
  - pod 보안 정책을 사용하려면 PSP를 포함한 컨트롤러관리자의 승인 컨트롤러를 구성해야 함
  - 위 정책은 클러스터의 RBAC 구성과 결합 될 때 그 의미가 부각됨
  - pod 보안 정책을 통해 사용자가 실행할 수 있는 항목과 컨테이너가 가질 수 있는 기능 
    및 낮은 수준의 권한을 미세 조정할 수 있음



### 8. 네트웍 보안 정책

- 기본적으로 pod은 모든 송ㆍ수신 트래픽 허용
- 네트워크 격리를 구성하고 pod에 대한 트래픽 차단 가능
- 최신 k8s 에서는 송신 트래픽도 차단 가능
- NetworkPolicy 를 구성하여야 위 기능이 가능



- 정책은 네임스페이스를 좁힐 수 있게 편리하게 해줌
- 추가 설정을 위해서는 podSelector 또는 라벨이 포함되어야 함
- 추가 송ㆍ수신 설정은 IP 주소 및 pod에서 송ㆍ수신되는 트래픽을 선언함
- NetworkPolicies를 지원하는 공급자 : Calico, Romana, Cilium, Kube-router 및 WeaveNe