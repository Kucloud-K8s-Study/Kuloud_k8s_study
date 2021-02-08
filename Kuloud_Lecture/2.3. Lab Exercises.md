## 2.3. Lab Exercises

### [Exercise 2.1: Overview and Preliminaries(개요 및 예비)]

- ubuntu18.04로 된 클러스터를 생성

```
student@ckad-1:˜$ wget https://training.linuxfoundation.org/cm/LFD259/LFD259 V2020-10-15 SOLUTIONS.tar.bz2 \ --user=LFtraining --password=Penguin2014
student@ckad-1:˜$ tar -xvf LFD259 V2020-10-15 SOLUTIONS.tar.bz2
```

- 테스트하기위해 만들어진 yaml파일을 다운로드 받는다.

### [Exercise 2.2: Deploy a New Cluster(새로운 클러스터 배포)]

```
student@ckad-1:˜$ find $HOME -name k8sMaster.sh
student@ckad-1:˜$ cp LFD259/SOLUTIONS/s_02/k8sMaster.sh .
student@ckad-1:˜$ bash k8sMaster.sh | tee $HOME/master.out

```

- Kubeadm을 사용하여 Master 노드를 deploy한다.
- k8sMaster.sh 파일을 홈 디렉터리에 복사한다.
- k8sSecond.sh 파일 실행, 스크립트의 결과를 저장하기 위해 tee 명령어 사용

```
student@ckad-2:˜$ wget https://training.linuxfoundation.org/cm/LFD259/LFD259 V2020-10-15 SOLUTIONS.tar.bz2 \ --user=LFtraining --password=Penguin2014
student@ckad-2:˜$ tar -xvf LFD259 V2020-10-15 SOLUTIONS.tar.bz2

student@ckad-2:˜$ find $HOME -name k8sSecond.sh
student@ckad-2:˜$ cp LFD259/SOLUTIONS/s_02/k8sSecond.sh .
student@ckad-2:˜$ bash k8sSecond.sh | tee $HOME/master.out
```

- 두번째 노드를 생성한다.
- wget을 통해 다운로드 받는 작업을 한다. (마스터 노드에서 했던 작업과 동일)
- k8sSecond.sh 파일을 홈 디렉터리로 복사한다. (마스터 노드에서 했던 작업과 동일)

```
student@ckad-2:˜$ bash k8sSecond.sh
```

- k8sSecond.sh를 실행
- 스크립트가 완료되면 미니언 노드가 클러스터에 참여할 준비가 된 것이다.

```
student@ckad-2:˜$ sudo kubeadm join --token 118c3e.83b49999dc5dc034 \
10.128.0.3:6443 --discovery-token-ca-cert-hash \
sha256:40aa946e3f53e38271bae24723866f56c86d77efb49aedeb8a70cc189bfe2e1d
```

- kubeadm 조인문은 마스터 노드의 output에 있는 마지막쯤 구문에서 찾을 수 있다.
- 두 번째 노드에서 설정이 완료되었으니 마스터 노드에 가서 설정을 해준다.

```
student@ckad-1:˜$ sudo apt-get install bash-completion vim -y
student@ckad-1:˜$ source <(kubectl completion bash)
student@ckad-1:˜$ echo "source <(kubectl completion bash)" >> $HOME/.bashrc
student@ckad-1:˜$ kubectl get node

```

- 마스터 노드에 돌아와 텍스트 편집기를 설치한다.
- 특수문자와 관련된 문제가 있을 수 있으므로 bash-completion을 설치한다.
- 두개의 노드가 클러스터의 부분인지 확인해본다.

![https://blog.kakaocdn.net/dn/bqKQbZ/btqUYhl7AhX/OAKxplFPWmPh1rDL4RtdkK/img.png](https://blog.kakaocdn.net/dn/bqKQbZ/btqUYhl7AhX/OAKxplFPWmPh1rDL4RtdkK/img.png)

```
student@ckad-1:˜$ kubectl --help
```

- kubectl을 통해 어떤 명령어를 쓸 수 있는지 확인 하려면 help를 사용한다.

### [taint]

### taint란

- 쿠버네티스 클러스터의 특정 노드에 taint를 지정할 수 있습니다. taint를 설정한 노드에는 포드들이 스케쥴링 되지 않는다.
- taint가 걸린 노드에 포드들을 스케쥴링 하려면 toleration을 이용해서 지정해 주어야한다.
- Taints는 특정 nodes에 일반적인 Pod가 배포되는 것을 막을 수 있다.
- 쿠버네티스 마스터 Node에는 관리를 위한 Pod만이 배포되어야 하기 때문에, 일반적인 Pod를 배포할 수 없도록 Taints가 이미 적용되어 있고, 마스터 Node에 Pod를 배포하기 위해서는 이에 맞는 Toleration을 가지고 있어야 한다.
- 가장 좋은 예로는 쿠버네티스의 마스터 Node에 적용된 Taints가 이에 해당한다.
- 이 외에도 운영용 Node로 특정 Node들을 적용해놓고, 개발이나 스테이징 환경용 Pod이 (실수로라도) 배포되지 못하게 한다는 것등에 사용할 수 있다.

![https://blog.kakaocdn.net/dn/cCfU6T/btqUZuyFnFg/B8wGg1kq2HHTk9mKaanrrk/img.png](https://blog.kakaocdn.net/dn/cCfU6T/btqUZuyFnFg/B8wGg1kq2HHTk9mKaanrrk/img.png)

![https://blog.kakaocdn.net/dn/boxqAR/btqUUrbQDZl/7VeBlbmMxvLGKLuadauFQ1/img.png](https://blog.kakaocdn.net/dn/boxqAR/btqUUrbQDZl/7VeBlbmMxvLGKLuadauFQ1/img.png)

### [Exercise 2.3: Create a Basic Pod]

### 1. pod를 만드는법

### pod란

- 파드(Pod) 는 쿠버네티스에서 생성하고 관리할 수 있는 배포 가능한 가장 작은 컴퓨팅 단위이다.
- 포드는 컨테이너 1개로 구성될수도 있고 여러개로 구성될수도 있다.
- 쿠버네티스는 컨테이너를 직접 관리하지 않고 포드를 관리한다.
- 도커 개념 측면에서, 파드는 공유 네임스페이스와 공유 파일시스템 볼륨이 있는 도커 컨테이너 그룹과 비슷하다.

- yaml파일 작성

![https://blog.kakaocdn.net/dn/ciLczO/btqUYfBXPTc/WRGEj46u0LJ4xpesFqaPNK/img.png](https://blog.kakaocdn.net/dn/ciLczO/btqUYfBXPTc/WRGEj46u0LJ4xpesFqaPNK/img.png)

- 생성 명령어

![https://blog.kakaocdn.net/dn/NoFHD/btqU0xomnOM/jaDhrskO0uolSkqtST3pM0/img.png](https://blog.kakaocdn.net/dn/NoFHD/btqU0xomnOM/jaDhrskO0uolSkqtST3pM0/img.png)

- 확인(조회) 명령어

![https://blog.kakaocdn.net/dn/cTmrSB/btqUSq5A05V/wQoAHJfk0RF7NVj10ANNU1/img.png](https://blog.kakaocdn.net/dn/cTmrSB/btqUSq5A05V/wQoAHJfk0RF7NVj10ANNU1/img.png)

- 삭제 명령어

![https://blog.kakaocdn.net/dn/3zwNE/btqUZvRTksY/1CfNG3KEpajDsqysqrvrp0/img.png](https://blog.kakaocdn.net/dn/3zwNE/btqUZvRTksY/1CfNG3KEpajDsqysqrvrp0/img.png)

- 포트 지정

![https://blog.kakaocdn.net/dn/zpQdw/btqU0xBTzOK/9EbBBdFYUcfoh4k4RBRj1k/img.png](https://blog.kakaocdn.net/dn/zpQdw/btqU0xBTzOK/9EbBBdFYUcfoh4k4RBRj1k/img.png)

- 생성 및 조회 -> 실행 -> 삭제

![https://blog.kakaocdn.net/dn/QyKAL/btqUYGsBLli/ZSJkK65YFMdICTVRfExqbK/img.png](https://blog.kakaocdn.net/dn/QyKAL/btqUYGsBLli/ZSJkK65YFMdICTVRfExqbK/img.png)

![https://blog.kakaocdn.net/dn/dpQnT9/btqUODdh7iL/8uMsqFHyuGjuztR7nivu50/img.png](https://blog.kakaocdn.net/dn/dpQnT9/btqUODdh7iL/8uMsqFHyuGjuztR7nivu50/img.png)

### 2. 서비스 생성

### 서비스란

- Pod의 경우에 지정되는 Ip가 랜덤하게 지정이 되고 리스타트 때마다 변하기 때문에 고정된 엔드포인트로 호출이 어렵다, 또한 여러 Pod에 같은 애플리케이션을 운용할 경우 이 Pod 간의 로드밸런싱을 지원해줘야 하는데, 서비스가 이러한 역할을 한다.
- 서비스는 지정된 IP로 생성이 가능하고, 여러 Pod를 묶어서 로드 밸런싱이 가능하며, 고유한 DNS 이름을 가질 수 있다.
- 서비스는 다음과 같이 구성이 가능하며, 라벨 셀렉터 (label selector)를 이용하여, 관리하고자 하는 Pod 들을 정의할 수 있다.

- yaml 파일 작성

![https://blog.kakaocdn.net/dn/bq32gU/btqUSqYRWVW/dDC1xjQwwqc4UcOjKtscSk/img.png](https://blog.kakaocdn.net/dn/bq32gU/btqUSqYRWVW/dDC1xjQwwqc4UcOjKtscSk/img.png)

- 서비스와 연동할 포드 yaml파일 작성

![https://blog.kakaocdn.net/dn/dWOpvC/btqUZvRUlj2/PodKjV2OSFPgLMOFMxoV5k/img.png](https://blog.kakaocdn.net/dn/dWOpvC/btqUZvRUlj2/PodKjV2OSFPgLMOFMxoV5k/img.png)

- yaml파일을 통해 서비스, 포드 생성

![https://blog.kakaocdn.net/dn/vm6T4/btqUSqdDqNA/4ty0NE6mdiMLvKYDCSzuXk/img.png](https://blog.kakaocdn.net/dn/vm6T4/btqUSqdDqNA/4ty0NE6mdiMLvKYDCSzuXk/img.png)

- 조회

![https://blog.kakaocdn.net/dn/wgHh4/btqUYF1wL5M/061VKMmKep5l16PsLRS0W1/img.png](https://blog.kakaocdn.net/dn/wgHh4/btqUYF1wL5M/061VKMmKep5l16PsLRS0W1/img.png)

- CRUSTER-IP를 통해 접속 테스트

![https://blog.kakaocdn.net/dn/XrxgZ/btqUZvElA96/lCMWKDzE9TVq0D6AsKrdhK/img.png](https://blog.kakaocdn.net/dn/XrxgZ/btqUZvElA96/lCMWKDzE9TVq0D6AsKrdhK/img.png)

- 서비스 삭제

![https://blog.kakaocdn.net/dn/J98JO/btqUSpySu47/nTgyugkX6KJfKdc3DkK4q1/img.png](https://blog.kakaocdn.net/dn/J98JO/btqUSpySu47/nTgyugkX6KJfKdc3DkK4q1/img.png)

### 3. 다른 타입의 서비스 생성

- yaml 파일 작성

![https://blog.kakaocdn.net/dn/bFQteV/btqU0xBUJrT/cbZGJ76WmVNKpwb8v2xy50/img.png](https://blog.kakaocdn.net/dn/bFQteV/btqU0xBUJrT/cbZGJ76WmVNKpwb8v2xy50/img.png)

- 생성 후 조회

![https://blog.kakaocdn.net/dn/l15e9/btqUWtN3TAb/iwfl3VsL8n56rLQMJltoKK/img.png](https://blog.kakaocdn.net/dn/l15e9/btqUWtN3TAb/iwfl3VsL8n56rLQMJltoKK/img.png)

- GCP나 AWS에서 받은 publicIP를 통해 테스트

![https://blog.kakaocdn.net/dn/cGjEJO/btqUOC6A2Ig/3GAI7kr28vQILSeMUSk4C0/img.png](https://blog.kakaocdn.net/dn/cGjEJO/btqUOC6A2Ig/3GAI7kr28vQILSeMUSk4C0/img.png)

### [Exercise 2.4: Multi-Container Pods]

### 1. 하나의 yaml파일에서 두 개의 pod 만들기

- yaml 파일 수정

![https://blog.kakaocdn.net/dn/RgOxF/btqUSo7OEBb/AHdobFLHkuMllDhWU3Vgd0/img.png](https://blog.kakaocdn.net/dn/RgOxF/btqUSo7OEBb/AHdobFLHkuMllDhWU3Vgd0/img.png)

- 기존 basicpod 삭제 후 다시 생성

![https://blog.kakaocdn.net/dn/dkd1Xz/btqUUqRLuWE/YhZ2fhqFpa4DEOSHsYGWEK/img.png](https://blog.kakaocdn.net/dn/dkd1Xz/btqUUqRLuWE/YhZ2fhqFpa4DEOSHsYGWEK/img.png)

- 조회

![https://blog.kakaocdn.net/dn/rbT3r/btqURvF6K4p/4Wp9tgbe7CdL0nslBUtX71/img.png](https://blog.kakaocdn.net/dn/rbT3r/btqURvF6K4p/4Wp9tgbe7CdL0nslBUtX71/img.png)

- 삭제

![https://blog.kakaocdn.net/dn/01IOQ/btqU0yOnHQm/LqpynWuSPURrLS2K0xZ1M0/img.png](https://blog.kakaocdn.net/dn/01IOQ/btqU0yOnHQm/LqpynWuSPURrLS2K0xZ1M0/img.png)

### [Exercise 2.5: Create a Simple Deployment]

### 1. deployment 생성, nginx 웹서버 실행

### deployment란

- 일단 쿠버네티스 클러스터를 구동시키면, 그 위에 컨테이너화된 애플리케이션을 배포할 수 있다.
- 그러기 위해서, 쿠버네티스 **디플로이먼트** 설정을 만들어야 한다.
- 디플로이먼트는 쿠버네티스가 애플리케이션의 인스턴스를 어떻게 생성하고 업데이트해야 하는지를 지시한다.
- 디플로이먼트가 만들어지면, 쿠버네티스 마스터가 해당 디플로이먼트에 포함된 애플리케이션 인스턴스가 클러스터의 개별 노드에서 실행되도록 스케줄한다.
- 애플리케이션 인스턴스가 생성되면, 쿠버네티스 디플로이먼트 컨트롤러는 지속적으로 이들 인스턴스를 모니터링한다.
- 인스턴스를 구동 중인 노드가 다운되거나 삭제되면, 디플로이먼트 컨트롤러가 인스턴스를 클러스터 내부의 다른 노드의 인스턴스로 교체시켜준다.
- **이렇게 머신의 장애나 정비에 대응할 수 있는 자동 복구(self-healing) 메커니즘을 제공한다.**

- 생성 명령어 입력

![https://blog.kakaocdn.net/dn/bFUkeZ/btqUZvK7L9L/brr8iyXdbGqrDmBilEfG00/img.png](https://blog.kakaocdn.net/dn/bFUkeZ/btqUZvK7L9L/brr8iyXdbGqrDmBilEfG00/img.png)

- 확인(두개 동시에 확인 가능)

![https://blog.kakaocdn.net/dn/bHGSZM/btqUUprFdfE/KxIP5WsSfUKU8i3nG3nAx0/img.png](https://blog.kakaocdn.net/dn/bHGSZM/btqUUprFdfE/KxIP5WsSfUKU8i3nG3nAx0/img.png)

- 상세 보기

![https://blog.kakaocdn.net/dn/cylWKl/btqUYf9SvEI/4mXrgC1EpmsfoDoNZSMio0/img.png](https://blog.kakaocdn.net/dn/cylWKl/btqUYf9SvEI/4mXrgC1EpmsfoDoNZSMio0/img.png)

![https://blog.kakaocdn.net/dn/RvNlu/btqUYGsDube/J6N6xJMAdcNgE88ueUqyN1/img.png](https://blog.kakaocdn.net/dn/RvNlu/btqUYGsDube/J6N6xJMAdcNgE88ueUqyN1/img.png)

![https://blog.kakaocdn.net/dn/bl9ERA/btqUOB0TzBG/IKcrhzPqGI8VFG5g68voUK/img.png](https://blog.kakaocdn.net/dn/bl9ERA/btqUOB0TzBG/IKcrhzPqGI8VFG5g68voUK/img.png)

- 사용가능한 네임스페이스 목록 조회

![https://blog.kakaocdn.net/dn/bHA01y/btqUYFNZKhU/AEl75Qhs95Kg0XILV6lK01/img.png](https://blog.kakaocdn.net/dn/bHA01y/btqUYFNZKhU/AEl75Qhs95Kg0XILV6lK01/img.png)

- 한번에 모든 네임스페이스 옵션 조회

![https://blog.kakaocdn.net/dn/tY8LH/btqUTmWeA7L/N2Ol35wsPViy8TGWMkFkg0/img.png](https://blog.kakaocdn.net/dn/tY8LH/btqUTmWeA7L/N2Ol35wsPViy8TGWMkFkg0/img.png)

- 한번에 여러 리소스 조회

![https://blog.kakaocdn.net/dn/GeVdb/btqUWsIqy48/1wFmo7u4ppKIgkNEXJOLU1/img.png](https://blog.kakaocdn.net/dn/GeVdb/btqUWsIqy48/1wFmo7u4ppKIgkNEXJOLU1/img.png)

## 2.4. Knowledge Check

# Question 2.1

---

Which of the following are part of a Pod?

- A. One or more containers
- B. Shared IP address
- C. One namespace
- D. All of the above

문제 해설 : Pod는 한개 또는 여러개의 컨테이너로 이루어져있다.

# Question 2.2

---

Which company developed Borg as an internal project?

- A. Amazon
- B. Google
- C. IBM
- D. Toyota

문제 해설 : 구글에서 만들었다.

# Question 2.3

---

In what database are the objects and the state of the cluster stored?

- A. ZooKeeper
- B. MySQL
- C. etcd
- D. Couchbase

문제 해설 : etcd는 모든 클러스터 데이터에 대한 Kubernetes의 백업 저장소로 사용되는 일관되고 가용성이 높은 키 값 저장소입니다.

# Question 2.4

---

Orchestration is managed through a series of watch-loops or controllers. Each controller interrogates the _________ for a particular object state.

- A. kube-apiserver
- B. etcd
- C. kubelet
- D. ntpd

문제 해설 : 오케스트레이션은 **운영자** 또는 **컨트롤러** 로 알려진 일련의 감시 루프를 통해 관리됩니다 .

각 컨트롤러 는 특정 객체 상태에 대해 **kube-apiserver** 를 조사하여 선언 된 상태가 현재 상태와 일치 할 때까지 객체를 수정합니다.

답 :

A, B, C, A
