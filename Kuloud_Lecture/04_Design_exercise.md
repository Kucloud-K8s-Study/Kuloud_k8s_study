# Exercise 4.1: Planning the Deployment

- CNI 플러그인을 사용하고 있는지 확인(프로세스 메시지)
  - ifconfig 명령을 통해서도 해당하는 CNI를 확인할 수 있다.

![1](https://user-images.githubusercontent.com/66216102/107108196-86fd2280-6879-11eb-9eb2-09721290ef33.PNG)

다양한 CNI 공급자가 존재하고, 요구 사항에 더 잘 맞는 플로그인이 있을 수 있다.

- Project Calico
- Calico with Canal
- Weave Works
- Flannel
- Romana
- Kube Router
- Kopeio

> CNI (Container Network Interface)
> 보통 Pod는 여러 노드에 걸쳐서 배포되는데 pod는 서로 하나의 네트워크에 있는 것처럼 통신이 가능.
> 이러한 기능을 지원하는 것이 CNI
>
> 우리는 CNI로 `Calico`를 사용중

### Default

- Pod는 기본적으로 Docker의 네트워크를 그대로 사용한다.
  - 도커 컨테이너가 시작할 때 만들어지는 `veth0`라는 가상 네트워크를 이용
  - 쿠버테니스는 Pod를 생성할 때 내부에 있는 컨테이너끼리 `veth0` 가상 인터페이스를 이용해 서로 공유할 수 있도록 함
- Pod 내의 Container들은 같은 IP주소를 사용하고 있다.
- 따라서 Kubernetes의 Pod 내부의 Container 간 네트워크 통신은 localhost로 가능하다. 물론 포트는 다르게 설정해 통신해야 한다.

> 자세한 내용은 쿠버네티스 `pause` 컨테이너 참고

![image-20210206122411826](https://user-images.githubusercontent.com/66216102/107108311-510c6e00-687a-11eb-992e-44be5e372bb9.png)

### CNI

- 하지만 여러 노드를 가지는 쿠버네티스의 특성상 위의 방식을 따르면 동일한 네트워크가 존재하게 됨
- 이 문제를 해결하기 위해서 **오버레이 네트워크(overlay network)** 방식을 통해 Pod가 서로 통신할 수 있는 구성을 만들 수 있다.
- **즉, 네트워크 인터페이스 (= kubenet 혹은 CNI)를 통하여 파드에 할당되어 있는 고유한 IP주소로 통신할 수 있다.**
  - 기본적으로 쿠버네티스는 `kubenet` 네트워크 플로그인을 제공해 주지만 **크로스 노드 네트워킹이나 네트워크 정책 설정** 같은 기능은 구현되어 있지 않다.

> 오버레이 네트워크
>
> 실제 노드 간의 네트워크 위에 별도 Flat한 네트워크를 구성하는 것을 의미한다.

![2](https://user-images.githubusercontent.com/66216102/107108197-8795b900-6879-11eb-9211-3c1a0df24ce2.PNG)

# Exercise 4.2: 기간별 애플리케이션 설계: Create a Job

- Job : 특정 횟수 혹은 어떤 정의된 내용만을 수행한 후 종료 함
- CronJob : 정기적으로 시간을 정해서 실행 함

### 1. 3초간 대기했다가 중지하는 작업 생성

![3](https://user-images.githubusercontent.com/66216102/107108198-8795b900-6879-11eb-8a3b-676f074f4cc6.PNG)

작업 실행 방식은 3가지가 존재

- backoffLimit: 백오프 정책이라고 하며 잡에 실패할 경우 다시 실행시킬 재시도 횟수를 지정 한다.
- completions: 지정한 개수만큼 Job를 처리
- parallelism: 하나의 잡에 여러개의 파드를 실행시켜 작업하는것을 말한다.

### 2. 5개의 Job를 처리 - completions

![image-20210206053735997](https://user-images.githubusercontent.com/66216102/107108306-4eaa1400-687a-11eb-9cd9-21ec54f47342.png)

- `kubectl get pods` 명령어를 쳐보면 경과 시간에 따라 Pod들이 순차적으로 실행된 것을 확인할 수 있다.

![4](https://user-images.githubusercontent.com/66216102/107108199-882e4f80-6879-11eb-944d-7789d42e4f33.PNG)

- 작업 완료 확인 및 Job 종료

### 3. 5개의 Job를 2개씩 병렬로 처리 - completions, parallelism

![5](https://user-images.githubusercontent.com/66216102/107108179-82d10500-6879-11eb-8dfc-8576292a2a4e.PNG)

- Pod가 두개씩 처리 되는 것을 확인할 수 있다.

### 4. 특정 시간 후에 작업을 중지하는 Job - activeDeadlineSecondes

![6](https://user-images.githubusercontent.com/66216102/107108182-84023200-6879-11eb-9071-91b1e05475d8.PNG)

- 해당 시간 이후에 Job이 종료됨

### 5. 성공/실패 상세항목 확인

![7](https://user-images.githubusercontent.com/66216102/107108183-84023200-6879-11eb-8d88-1ed2f580ab5a.PNG)

# Exercise 4.3: 기간별 애플리케이션 설계: Create a CronJob

### 1. 크론잡 실행 및 확인

![8](https://user-images.githubusercontent.com/66216102/107108184-849ac880-6879-11eb-93fe-41fb389fc8a6.PNG)

### 2. .yaml에서 설정한 것처럼 2분 후 실행되는 것을 확인

![9](https://user-images.githubusercontent.com/66216102/107108187-849ac880-6879-11eb-94ed-27335fd4a3c4.PNG)

### 3. 30초 동안 실행하도록 설정하고, 10초 뒤에 컨테이너가 종료되는지 확인

![image-20210206060220928](https://user-images.githubusercontent.com/66216102/107108307-4f42aa80-687a-11eb-99c9-e3732d3f6a22.png)

# Exercise 4.4: 라벨 사용하기

### 1. 디플로이먼트 생성 & 셀렉터 확인 & `-l` 옵션을 이용한 디플로이먼트 안의 파드 확인 (특정 selector만 검출)

![image-20210206061018738](C:\Users\Nick_주성우\AppData\Roaming\Typora\typora-user-images\image-20210206061018738.png)

### 2. pod에 대한 정보를 `--selector` 옵션을 이용해 yaml 형식으로 확인

![image-20210206061545391](https://user-images.githubusercontent.com/66216102/107108309-5073d780-687a-11eb-81e4-644ff0e2584c.png)

### 3. 라벨을 다르게 지정하고 design2 이름을 가진 파드의 개수 확인

![11](https://user-images.githubusercontent.com/66216102/107108190-85335f00-6879-11eb-8c3d-aa3b15954fbb.PNG)

![12](https://user-images.githubusercontent.com/66216102/107108193-85cbf580-6879-11eb-91b5-e85497787d1b.PNG)

![13](https://user-images.githubusercontent.com/66216102/107108194-86648c00-6879-11eb-8f38-6ab14d38950b.PNG)

- design2 이름을 가진 파드의 개수가 두개가 있는 것을 확인
  - 정의한 상태에 부합하기 위해 **복제본**을 하나 더 만든 것임

# Exercise 4.5: 포드 리소스 제한 및 요구 사항 설정

### stress.yaml 실행 후 파드를 조회했을 때 상태 확인

![14](https://user-images.githubusercontent.com/66216102/107108195-86648c00-6879-11eb-95d8-779eef6a8627.PNG)

- OOMKilled 상태와 재실행 횟수 확인
  - **OOMKilled** : 메모리가 부족해 종료된 상태
- 오래 기다리면 `CrashLoopBackOff` 를 볼 수 있다. 번번히 실패하면 이 상태를 표시
  - 즉, `CrashLoopBackOff` 상태란 파드(pod)가 시작과 비정상 종료를 연속해서 반복하는 상태를 말한다.

# Question 4.1

Which of the following are helper container types? Choose all answers that apply.

- **.** Ambassador
- **B.** Adapter
- **C.** ProbeHelper
- **D.** Sidecar

a, b, d

# Question 4.2

If a Pod uses more CPU than allowed, it will be evicted. True or False?

- **A.** True
- **B.** False

B

# Question 4.3

If a Pod tries to use more memory than the node has, it will be evicted. True or False?

- **A.** True
- **B.** False

A

# Question 4.4

If a Pod uses more than **spec.containers[].resources.limits.ephemeral-storage**, it will \***\*\_\*\***.

- **A.** Be restarted
- **B.** Be migrated
- **C.** Be evicted
- **D.** Remain the same

C

# Reference

[https://velog.io/@seunghyeon/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EA%B5%AC%EC%84%B1%EB%8F%84](https://velog.io/@seunghyeon/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EA%B5%AC%EC%84%B1%EB%8F%84)

[https://velog.io/@seunghyeon/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EA%B5%AC%EC%84%B1%EB%8F%84](https://velog.io/@seunghyeon/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EA%B5%AC%EC%84%B1%EB%8F%84)

[https://engineering.linecorp.com/ko/blog/prometheus-container-kubernetes-cluster/](https://engineering.linecorp.com/ko/blog/prometheus-container-kubernetes-cluster/)

[https://box0830.tistory.com/293](https://box0830.tistory.com/293)
