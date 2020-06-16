## Swarm 구성 

---

#### 초기 설정 

- 우분투 호스트 네임 manager로 바꾼 다음, 중지 후 메모리 수정
- manage > clone 기능을 통해 manager서버에서 3개의 worker 1~3을 생성하고 각 ip주소와 호스트 네임을 설정 
<div>
<img src="https://user-images.githubusercontent.com/66865899/84675764-a8667280-af67-11ea-944f-2229501a60ad.png" width="40%"></img>
<img src="https://user-images.githubusercontent.com/66865899/84675779-ad2b2680-af67-11ea-8673-d293681960a2.png" width="40%"></img>
</div>

#### 실습 환경 구성

---


실제로는 manager에 해당하는 서버를 이중화하여 준비한다. 

##### **Clustering & Check**

```bash
$ docker node ls
"도커 swarm에 클러스터되어 있는 호스트의 리스트 상태 확인"
```

### 1. TOKEN 발행(@manager node)
---

swarm 구성하기 위해서  manager는 token을 발행한다.

```bash
user1@manager:~$ docker swarm init --advertise-addr 211.183.3.100 
Swarm initialized: current node (w1en9xiuvs7haeyozfu8ta3b7) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2abzt2vlvxtzd7nsvxtcxmmdumhjcooobhrwsywhdhm3wawb6o-761gnz6t0qo6fhatfeo0bdz4g 211.183.3.100:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

### 2. SWARM JOIN (@worker 1~3 node)
---

manager에서 token으로 발행된 'docker swarm join ~' 코드를 그대로 복사하여 입력한다. 

```bash
$ docker swarm join --token SWMTKN-1-2abzt2vlvxtzd7nsvxtcxmmdumhjcooobhrwsywhdhm3wawb6o-761gnz6t0qo6fhatfeo0bdz4g 211.183.3.100:2377
**This node joined a swarm as a worker.**
```

**command line**

```bash
$ docker swarm leave "각 노드에서 스스로 노드에서 빠짐"
$ docker node rm [id or HOSTNAME] "노드에서 탈퇴시키기"
$ docker node promote/demote [id or HOSTNAME]: 특정 노드의 노드 계급을 변경시키기
```

### 3. SWARM-JOINing node list check(@ manager node)
---

```bash
$ docker node ls
```

<img src="https://user-images.githubusercontent.com/66865899/84676531-96390400-af68-11ea-9c2c-437a17e00684.png" width="50%"></img>


### 4. Distribute Containers into CLUSTER (@ manager node)
---

**컨테이너 복제 생성 후 배포** 

```bash
$ docker service create --replicas 2 -p 8080:80 --name web nginx 
```

**배포된 위치 확인**

```bash
$ docker service ls "어떤 서비스가 생성되었는지 확인" 
$ docker service ps web "web이라는 이름의 서버가 어디에 위치하는지 확인"
```

**잠깐! docker_worker1과 manager에만 web이 배포되었다고 했는데
왜 docker2와 docker3에서도 nginx가 뜨는 걸까요?** 
실제 컨테이너는 2개가 생성되어있고, *서버 2개*에서 트래픽을 처리하고 있음: 들어가는 주소는 내가 요청한 주소로 들어가지만, 
해당 노드들이 모두 Cluster가 되어있기 때문에 자동으로 해당 서비스가 있는 곳으로 접속이 된다. 


**트래픽 증가하는 경우 컨테이너 scale out** 

```bash
$ docker service scale web=4 
$ docekr service ps web

/*줄이기*/
$ docker service scale web=1
```

**Overlay network 구성 및 확인**

```bash
$ docker network create --driver=overlay --attachable web
$ docker network ls
```

## Docker Swarm, Haproxy, Nginx를 활용한 웹서비스와 로드밸런싱

### 1. 스택 파일 생성
스택을 생성하기 위한 스택 YAML 파일을 작성한다. 

```bash
$ mkdir swarm
$ touch web.yml 

version: "3"

services:
  nginx:
    image: nginx
    deploy:
      replicas: 3
      placement:
        constraints: [node.role != manager]
      restart_policy:
        condition: on-failure
        max_attempts: 3
    environment:
      SERVICE_PORTS: 80
    networks:
      - web

  proxy:
    image: dockercloud/haproxy
    depends_on:
      - nginx
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - 8001:80
    networks:
      - web
    deploy:
      mode: global
      placement:
        constraints: [node.role == manager]

networks:
  web:
    external: true
```

> **Options *for what* ** | 
> 
> - constraints: WORKER 서버 3대에만 nginx기반 컨테이너를 배포하고, manager서버에는 proxy만 배포
> - restart policy: 재부팅을 시도하라는 명령어
> - volumes: 각 worker 서버 잘 실행되는지 감시하기 위해 
> *"/var/run/docker.sock"* : Docker CLI가 데몬에게 명령어를 전달할 수 있는 인터페이스 
> /var/run/docker.sock:/var/run/docker.sock
> - global: 호스트별로 각 1개씩 나눠주기

### 2. Stack Deploy

```bash
$ docker stack deploy --compose-file=web.yml web
```
