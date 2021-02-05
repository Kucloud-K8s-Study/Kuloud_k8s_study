# 3. BUILD 실습



## 사전 준비사항

- GCP 나 AWS에 마스터 노드 한 개, 워커 노드 한 개 준비 되어있으면 실습하기 편함

- macOS의 경우에는 Termius를 App Store에서 다운 받아서 RSA 기반 키 생성하여 VM 인스턴스 생성하면 편리함.

- Chapter2의 Exercise 2.1 ~ Exercise 2.2 까지만 수행.

- 처음 wget https://training.linuxfoundation.org/cm/LFD259/LFD259_V2021-01-26_SOLUTIONS.tar.xz \ --user=LFtraining --password=Penguin2014  오류 시 파일명인 LFD259_V2021-01-26-SOLUTIONS.tar.xz 의 '\_' 문자열 잘 확인바람. pdf에서 바로 복사시 '\_' 가 생략되는듯.



  

## Exercise 3.1 : Deploy a New Application



### 간단설명

5초 마다 호스트이름과 현재시간을 출력하는 간단한 파이썬 애플리케이션을 도커라이징하고 실행해본다



1. python 설치

2. 애플리케이션 소스파일이 들어갈 디렉터리 생성

3. simple.py 이름으로 파일 생성(vim 편집기 이용)

4. simple.py에 실행권한 준 뒤 ``student@master: ̃/app1$ ./simple.py``  수행후 20~30초뒤 Ctrl+C 눌러 종료시킨 후 ``cat date.out`` 명령으로 애플리케이션이 잘 작동하는지 확인

5. Dockerfile 생성

   ```dockerfile
   FROM python:2
   ADD simple.py /
   ## RUN pip install pystrich
   CMD [ "python", "./simple.py" ]
   ```

6. simpleapp 이름으로 도커 이미지 빌드 수행

7. ``student@master: ̃$ sudo docker run simpleapp``  명령어 입력 후 위에서 테스트했던 방법과 마찬가지로 20~30 초 후 Ctrl+C로 종료

8. ``student@master: ̃/app1$ sudo find / -name date.out``

9. ```bash
   /home/student/app1/date.out   /var/lib/docker/overlay2/ee814320c900bd24fad0c5db4a258d3c2b78a19cde 629d7de7d27270d6a0c1f5/diff/date.out
   ```

   위와 같이 2개의 파일이 찾아짐. 아래 것이 컨테이너가 만든 date.out

   ```bash
   student@master: ̃/app1$ sudo tail \ /var/lib/docker/overlay2/ee814320c900bd24fad0c5db4a258d3c2b78a19cde
   629d7de7d27270d6a0c1f5/diff/date.out
   ```

   위 명령어를 입력하면 다음과 같이 출력됨

   ```bash
   2018-03-22 16:13:46
   53e1093e5d39
   2018-03-22 16:13:51
   53e1093e5d39
   2018-03-22 16:13:56
   53e1093e5d39
   ```



## Exercise 3.2: Configure A Local Docker Repo



### 간단설명

master 노드에 hub.docker.com 와 같이 세계와 공유되는 리포지토리가 아니라 Local Repository를 생성하여, 쿠버네티스에 속한 모든 노드들이 Local Repository에서 도커 이미지를 빌드하고 푸쉬하고 풀 할 수 있게한다. 



1. 홈디렉터리로 이동하여(cd) root 계정으로 로그인한다 (sudo -i)  <- 추가 라이브러리들을 설치하기 위함

2. docker-compose와 apache2-utils 설치

3. /localdocker/data 경로에 디렉터리 생성

4. /localdocker 로 이동하여 docker-compose.yaml을 작성한다. 이 파일은 nginx 와 registry 컨테이너를 포함함. 현재 단계에서는 테스트만 수행하고 **kompose** 라는 도구로 Kubernetes 파일로 변환할 거임

   ```dockerfile
   nginx:
     image: "nginx:1.17"
     ports:
       - 443:443
     links:
       - registry:registry
     volumes:
       - /localdocker/nginx/:/etc/nginx/conf.d
   registry:
     image: registry:2
     ports:
       - 127.0.0.1:5000:5000
     environment:
       REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /data
     volumes:
       - /localdocker/data:/data
   ```

5. `` root@master:/localdocker# docker-compose up`` 으로 실행 후 

6. **다른 터미널을 켜서** ``student@master: ̃/localdocker$ curl http://127.0.0.1:5000/v2/`` 를 했을 때 {} 가 잘 반환되는지 확인, 올바르게 동작하면 Ctrl+C로 종료

7. kompose 설치 후 student 계정으로 복귀

   ```bash
   root@master:/localdocker# curl -L https://bit.ly/2tN0bEa -o kompose
   root@master:/localdocker# chmod +x kompose
   root@master:/localdocker# mv ./kompose /usr/local/bin/kompose 
   root@master:/localdocker# exit
   ```

8. 2개의 Persistent Volume 작성(컨테이너가 종료되도 호스트 파일시스템에 남아 삭제되지 않는 볼륨)

   ```yaml
   # vol1.yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     labels:
       type: local
     name: task-pv-volume
   spec:
     accessModes:
     - ReadWriteOnce
     capacity:
       storage: 200Mi
     hostPath:
       path: /tmp/data
     persistentVolumeReclaimPolicy: Retain
   ```

   ```yaml
   # vol2.yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     labels:
       type: local
     name: registryvm
   spec:
     accessModes:
     - ReadWriteOnce
     capacity:
       storage: 200Mi
     hostPath:
       path: /tmp/nginx
     persistentVolumeReclaimPolicy: Retain
   ```

   

9. 리소스 생성

   ```bash
   student@master: ̃$ kubectl create -f vol1.yaml
   student@master: ̃$ kubectl create -f vol2.yaml
   ```

10. 리소스 확인 Status Available에 주목

    ```
    student@master: ̃$ kubectl get pv
    ```

    ![image-20210206060338640](https://user-images.githubusercontent.com/25981278/107094412-4df98980-684a-11eb-8cc3-32107f983943.png)

11. docker-compose.yaml 파일이 있는 docker-compose.yaml로 이동

    ```bash
    student@master: ̃$ cd /localdocker/ 
    student@master: ̃/localdocker$ ls
    ```

12. docker-compose.yaml 파일을 localregistry.yaml 로 변환

    ```bash
    student@master: ̃/localdocker$ sudo kompose convert -f docker-compose.yaml -o localregistry.yaml
    ```



## kubernetes 업데이트에 따른 localregistry.yaml 변경 필요성

```bash
student@master: ̃/localdocker$ kubectl create deployment drytry --image=nginx --dry-run=client -o yaml
```

--dry-run 모드로 실행하면 -o yaml 옵션을 주면 현재 클러스터에 맞는 yaml 파일 형태를 볼 수 있음.



결론적으로, 위의 명령어와 앞서 kompose로 생성한 localregistry.yaml을 비교 대조하여 41, 53, 93, 105 번째 라인을 바꾸어 주면됨(귀찮으면 제공파일의 edited-localregistry.yaml을 복사하여 사용)

![image-20210206061257509](https://user-images.githubusercontent.com/25981278/107094416-4f2ab680-684a-11eb-8056-93bb1afabb0d.png)

<  를 추가 혹은 변경

\> 는 삭제 혹은 변경

13. ``student@master: ̃/localdocker$ kubectl create -f localregistry.yaml`` 로컬레지스트리 생성 후
14. ``student@master: ̃/localdocker$ kubectl get pods,svc,pvc,pv,deploy`` 현재 리소스 상태 가져오면 다음과 같음

------

리소스 생성 후

![image-20210206061519420](https://user-images.githubusercontent.com/25981278/107094423-505be380-684a-11eb-95c5-082b15903fe2.png)



---------------------------------------------

리소스 생성 전

![image-20210206061600916](https://user-images.githubusercontent.com/25981278/107094425-50f47a00-684a-11eb-9862-1654e551ed48.png)

15. ``student@master: ̃/localdocker$ curl http://10.110.186.162:5000/v2/`` 앞서 docker-compose 처럼 kubernetes도 핑을 날려보면 {}을 리턴 받음. 10.110.186.162는 각자 컴퓨터에 따라 다름.

16. 로컬 레지스트리의 트래픽을 허용해주려면 docker 데몬 파일 수정후 도커 재실행 해야함.

    ```bash
    student@master: ̃$ sudo vim /etc/docker/daemon.json
    ```

    ```bash
    { "insecure-registries":["10.110.186.162:5000"] }
    ```

    10.110.186.162:5000 도 컴퓨터마다 다름.

    ```
    student@master: ̃$ sudo systemctl restart docker.service
    ```

17. 도커허브에서 ubuntu 이미지 받아온후 태그 붙여서 로컬레지스트리에 푸쉬하기

    ```bash
    student@master: ̃$ sudo docker pull ubuntu
    student@master: ̃$ sudo docker tag ubuntu:latest 10.110.186.162:5000/tagtest
    student@master: ̃$ sudo docker push 10.110.186.162:5000/tagtest
    student@master: ̃$ sudo docker image remove ubuntu:latest
    student@master: ̃$ sudo docker image remove 10.110.186.162:5000/tagtest
    student@master: ̃$ sudo docker pull 10.110.186.162:5000/tagtest
    ```

18. 앞서 배포한 파이썬 컨테이너를 태그한 후 로컬 레지스트리에 푸쉬하기

    ```bash
    student@master: ̃$ sudo docker tag simpleapp 10.110.186.162:5000/simpleapp 
    student@master: ̃$ sudo docker push 10.110.186.162:5000/simpleapp
    ```

19. 워커 노드에서 이미지 받아보기

    ```bash
    student@worker: ̃$ sudo vim /etc/docker/daemon.json
    ```

    ```
    { "insecure-registries":["10.110.186.162:5000"] }
    ```

    10.110.186.162:5000 도 컴퓨터마다 다름.

    ```bash
    student@worker: ̃$ sudo systemctl restart docker.service
    ```

    ```bash
    student@worker: ̃$ sudo docker pull 10.110.186.162:5000/simpleapp
    ```

![image-20210206062847041](https://user-images.githubusercontent.com/25981278/107094439-55209780-684a-11eb-8fe3-a0c613552f0e.png)

그럼 위와 같이 정상적으로 이미지를 pull함.

20. 마스터 노드로 돌아와서, simpleapp이미지를 가지고 try1 이름으로 디플로이먼트 리소스 생성하기. 복제본의 개수는 6개

    ```bash
    student@master: ̃$ kubectl create deployment try1 --image=10.110.186.162:5000/simpleapp
    student@master: ̃$ kubectl scale deployment try1 --replicas=6
    student@master: ̃$ kubectl get pods
    ```

    ![image-20210206063144328](https://user-images.githubusercontent.com/25981278/107094441-55b92e00-684a-11eb-8190-50cfb324af29.png)

21. 워커노드에서 ``sudo docker ps | grep simple`` 명령어를 입력하면 6개중 일부 simple 컨테이너가 동작하고 있을 것임.(랜덤이지만, 쿠버네티스에서 보통 적절히 노드에 따라 분배함)

22. 현재 6개 레플리카를 가지는 디플로이먼트를 yaml로 저장

    ```bash
    student@master: ̃/app1$ cd  ̃/app1/
    student@master: ̃/app1$ kubectl get deployment try1 -o yaml > simpleapp.yaml
    student@master: ̃$ kubectl delete deployment try1
    student@master: ̃/app1$ kubectl create -f simpleapp.yaml
    student@master: ̃/app1$ kubectl get deployment
    ```

    ![image-20210206063457157](https://user-images.githubusercontent.com/25981278/107094437-55209780-684a-11eb-893a-9370b8beefb7.png)

## Exercise 3.3: Configure Probes



### 간단설명

쿠버네티스의 상태 관리 방법으로써 크게 Liveness probe와 Readiness Probe로 나누어진다. 전자는 컨테이너 실행 중 지속적으로 실행되지만, 후자는 준비상태가 확인된 이후에는 실행되지 않는다.

구체적인 확인 방법으로는 exec, http get, tcpSocket 으로 컨테이너를 테스트한다.



1. simpleapp.yaml 파일 수정, simpleapp 컨테이너는 5초마다 exec 방법으로 준비되었는지 확인한다.

   ```yaml
   ....
       spec:
         containers:
         - image: 10.111.235.60:5000/simpleapp:latest
           imagePullPolicy: Always
           name: simpleapp
           readinessProbe:
             periodSeconds: 5
             exec:
               command:
               - cat
               - /tmp/healthy
           resources: {}
   ....
   ```

2. ReadinessProbe가 잘 작동하는지 확인하기 위해 리소스 삭제 후 재생성

   ```bash
   student@master: ̃/app1$ kubectl delete deployment try1
   student@master: ̃/app1$ kubectl create -f simpleapp.yaml
   student@master: ̃/app1$ kubectl get deployment
   student@master: ̃/app1$ kubectl get pods
   ```

   ![image-20210206065414271](https://user-images.githubusercontent.com/25981278/107094435-54880100-684a-11eb-9cce-c37a64c0e339.png)

3. 위와 같이 나오는 이유는 cat /tmp/healthy가 0을 리턴하지 않기 때문임. 해결책은 뒤에서 한 번에 보겠음.

4. 새로운 컨테이너에 livenessProbe 와 readinessProbe를 동시에 배치해보겠음.

   ```bash
   student@master: ̃/app1$ vim simpleapp.yaml
   ```

   ```yaml
   ....
   //    - image와 같은 레벨로 작성, 앞 공백 6칸임
         - name: goproxy
           image: k8s.gcr.io/goproxy:0.1
           ports:
           - containerPort: 8080
           readinessProbe:
             tcpSocket:
               port: 8080
             initialDelaySeconds: 5
             periodSeconds: 10
           livenessProbe:
             tcpSocket:
               port: 8080
             initialDelaySeconds: 15
             periodSeconds: 20
   ....
   ```

5. try1 디플로이먼트 리소스 삭제 후 재생성

   ```bash
   student@master: ̃$ kubectl delete deployment try1
   student@master: ̃$ kubectl create -f simpleapp.yaml
   student@master: ̃$ kubectl get pods
   ```

   ![image-20210206070014661](https://user-images.githubusercontent.com/25981278/107094433-54880100-684a-11eb-8f05-fa2ad5f5d907.png)

Running  상태의 컨테이너 보면, 2개중 1개만 Ready임. 새롭게 추가한 goproxy 컨테이만 준비되어있음.

6. simpleapp 컨테이너들도 준비된 상태로 만들기.

   ```bash
   student@master: ̃$ for name in $(kubectl get pod -l app=try1 -o name) > do
   > kubectl exec $name -c simpleapp -- touch /tmp/healthy
   > done
   ```

7. ``student@master: ̃$ kubectl get pods`` 로 다시 확인.

![image-20210206070258140](https://user-images.githubusercontent.com/25981278/107094429-5356d400-684a-11eb-8961-bbf97ef3d679.png)

8. ``student@master: ̃/app1$ kubectl describe pod try1-76cc5ffcc6-tx4dz | tail`` 로 세부사항 확인

![image-20210206070242289](https://user-images.githubusercontent.com/25981278/107094430-53ef6a80-684a-11eb-8a5e-0a6cfaac3038.png)



