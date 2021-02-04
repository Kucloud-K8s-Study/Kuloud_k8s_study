# 03 Build

### K8s 기본 구조 리뷰

<img src="https://lh4.googleusercontent.com/hCfk3E8n1pRBMugPOvTunPYk4od_jN-BCgrkTCENDvpTM3DxgLibvCORmmaKuqZYqqvfWdWckToIU8484oiiKySqTrs9OIwHjGpRxvU-Dbmh39OPwI89npvq3Tz2Q8pwAWSpcrK5">

#### 기본 구성 요소

**container --> pod --> node --> cluster**

- **Container (MS word)**: 하나의 프로그램이 실행되기 위한 정보 코드, dependency등이 포함된 패키지

- **Pod (Windows)**: 컨테이너의 실행 방법을 정의하고 컨테이너가 실행될 수 있는 환경을 제공해주는, 하나의 pod에 하나의 컨테이너가 가장 일반적인 구조, 배포 가능한 가장 작은 단위의 객채

- **Node (내 컴퓨터)**: Pod가 실행될 수 있는 가상이나 물리적인 하드웨어

- **Cluster (컴퓨터들의 집합 / 서버)**: Node의 집합 (사실상 쿠버네티스를 사용하는 이유 --> Docker가 애플리케이션 하나 단위를 관리한다면 쿠버네티스는 여러 노드의 집합을 종합적으로 관리하는데 유용하기 때문)



### Container의 종류와 옵션

컨테이너 생성의 목적으로는 **Docker가 가장 일반적**으로 쓰이지만 다른 컨테이너 엔진들이 등장하면서 Kubernetes도 다양한 vendor를 지원할 수 있게 변화하고 있음

**Container runtime**: 컨테이너를 실행시키는 애플리케이션

- Docker Engine (Default)
- CRI-O

**Containerized Image**: 컨테이너를 생성할때 쓰이는 이미지

- Docker
- Open Container Initiative (OCI) - 자유도가 높고 다양한 운영체제에서 지원
- runC



### Docker

그래도 아직까지 가장 일반적으로 쓰이는 애플리케이션

**주요 기능**

- Containerizing (컨테이너 생성 (컨테이너화))
- Deploying (배포)
- Docker Hub를 통한 이미지 공유 및 협업
- Swarm (컨테이너 orchestration(관리)를 위한 툴)
  - Kubernetes에 비해 container 배포 속도가 훨씬 빠름
  - Kubernetes와 비슷한 기능을 하는데 대용량 cluster 관리에는 kubernetes에 비해 부적합

### Container 관리를 위한 툴

**Container Runtime Interface**: kubelet이 다양한 container runtime에서 애플리케이션을 실행할 수 있도록 도와주는 플러그인 

- 컨테이너의 생성, 삭제등 생명 주기를 관리
- 다른 애플리케이션과 충돌하지 않도록 protocol을 설정
- 다른 비슷한 스펙으로는 OCI(Open Container Initiative)가 있는데, OCI 스펙에 맞추어 구현된 컨테이너 런타임은 CRI를 통해서도 관리가 가능

**rkt**: container를 실행시키기 위한 CLI, 도커와 비슷한 역할을 수행

**CRI-O**: 컨테이너 런타임의 한 종류로, 현재는 다양한 운영체제, 벤더와의 확장성으로 주목받는중

**containerd**: Low-level 컨테이너 관리를 위한 런타임 애플리케이션 (일반 유저보다는 백엔드에서 관리를 위한)



### Application을 Kubernetes 환경에서 Build 하는 과정

<img src="https://miro.medium.com/max/2520/1*p8k1b2DZTQEW_yf0hYniXw.png">

**1. 기존의 애플리케이션을 컨테이너화(Containerize) 하기**: 환경에 구애받지 않고(stateless & transient) 하나의 빌드로 구성될 수 있도록 기존의 서비스를 마이크로화 하기

**2. Dockerfile 만들기**: image를 구성할때 필요한 파일들을 한 디렉토리에 구성하고 실행 명령어들을 담을 Dockerfile을 작성

- `ADD`, `RUN`, `CMD` 등을 활용해 실행 파일을 작성

**3. Container image 생성**: `sudo docker build -t simpleapp`

**4. Image 확인 및 컨테이너 실행:** `sudo docker images`

​													`sudo docker run simpleapp`

**5. Repository에 push:** sudo docker push (여기까지가 큰 범주에서 봤을때 도커의 역할)

<hr>

<img src="https://cdn.thenewstack.io/media/2018/07/d00605db-c1cdpiplin.png">

**6. Kubectl을 활용해 pod의 형태로 배포: **`kubectl create` 명령어를 활용해 배포하고 `kubectl get pods` 명령어를 통해 현재 실행중인 pod의 상태 체크 가능

**7.  Pod안에서 특정 컨테이너를 실행하거나 관리**

- **Multi-Container Pod:** 하나의 pod안에 한 container가 가장 일반적인 구조지만, 필요에 따라서 여러 컨테이너가 하나의 pod에서 실행될 수도 있음
  - 여러개의 container라도 하나의 IP주소, 네임스페이스를 공유
  - 공평하게 저장소를 나누어 사용 가능
  - 여러개의 컨테이너를 사용할시 서로 overriding 되는 일이 없도록 규칙이나 설정을 변경해야한다
    - 각자의 목적이 분명해야되고 통신시의 규칙이 있어야된다
  - **Ambassador**: cluster밖에 있는 리소스들과 통신하기 위한 컨테이너
  - **Adapter**: primary container에서 만들어진 데이터를 가공하기 위한 컨테이너
  - **Sidecar**: primary container에서 제공되지 않는 서비스를 제공하는 컨테이너

**8. Pod를 활성화 시키기**: 새로운 pod가 활성화되고 완전히 사용되기 전의 준비과정

- **readinessProbe**: pod가 배포되고 나서 완전히 healthy한 state를 띄우기까지 traffic을 받지 않는 것
- **livenessProbe**: 지속적으로 container가 정상적으로 작동되고 있는지 체크하는 것

**9. 테스트 하기:** 실제 pod에 있는 container들이 잘 동작하고 있는지 `kubectl describe pod <name>` 명령어를 통해 확인 가능 (어떤 액션들이 실제로 동작했는지 확인 가능)

**10. 필요에 따라 pod안에 container를 추가하거나 업데이트**

**11. 새로운 pod로 교체 하며 주기적으로 pod를 관리**

