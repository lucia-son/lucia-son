### Swarm 구성 

---

#### 초기 설정 

- 우분투 호스트 네임 manager로 바꾼 다음, 중지 후 메모리 수정


- manage > clone 기능을 통해 manager서버에서 3개의 worker 1~3을 생성하고 각 ip주소와 호스트 네임을 설정 

<img src="https:/g>/s3-us-west-2.amazonaws.com/secure.notion-static.com/cb52ad5d-35b1-48a6-b555-5953f45d147c/Untitled.png" width="40%"></img>

#### 실습 환경 구성

---

[//topology](//topology) 참조

실제로는 manager에 해당하는 서버를 이중화하여 준비한다. 

Clustering & Check 

```bash
$ docker node ls
"도커 swarm에 클러스터되어 있는 호스트의 리스트 상태 확인"
```

**쿠버네티스의 서버 구성:** 
data store & master & node
**node 내부에는 컨테이너 집합**(pod)이 있는데 각 집합은 하나의 서비스를 제공하기 위해 웹 컨테이너와 db 컨테이너 2개가 연결되어 있다.  
즉 ,**서비스 제공의 최소 단위  =  포드**(pod)  

//포드 구성 불가의 예시 

### @manager: TOKEN 발행

swarm 구성하기 위해서  manager는 token을 발행한다.

```bash
user1@manager:~$ docker swarm init --advertise-addr 211.183.3.100 
Swarm initialized: current node (w1en9xiuvs7haeyozfu8ta3b7) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2abzt2vlvxtzd7nsvxtcxmmdumhjcooobhrwsywhdhm3wawb6o-761gnz6t0qo6fhatfeo0bdz4g 211.183.3.100:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

### @worker 1~3 : SWARM JOIN 하기

manager에서 token으로 발행된 'docker swarm join ~' 코드를 그대로 복사하여 입력한다. 

```bash
$ docker swarm join --token SWMTKN-1-2abzt2vlvxtzd7nsvxtcxmmdumhjcooobhrwsywhdhm3wawb6o-761gnz6t0qo6fhatfeo0bdz4g 211.183.3.100:2377
**This node joined a swarm as a worker.**
```

추가 명령어 

```bash
$ docker swarm leave "각 노드에서 스스로 노드에서 빠짐"
$ docker node rm [id or HOSTNAME] "노드에서 탈퇴시키기"
$ docker node promote/demote [id or HOSTNAME]: 특정 노드의 노드 계급을 변경시키기
```

### @manager: SWARM에 JOIN한 노드 확인

```bash
$ docker node ls
```

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c80b9c46-7a8e-4001-8511-a233b801d2e4/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c80b9c46-7a8e-4001-8511-a233b801d2e4/Untitled.png)

* 는 자기 자신을 표시한다. 

### @manager: 클러스터 내부에 컨테이너 배포 및 확인

**컨테이너 복제 생성 후 배포** 

```bash
$ docker service create --replicas 2 -p 8080:80 --name web nginx 
```

**배포된 위치 확인**

```bash
$ docker service ls "어떤 서비스가 생성되었는지 확인" 
$ docker service ps web "web이라는 이름의 서버가 어디에 갔는지 확인"
```

**잠깐! docker_worker1과 manager에만 web이 배포되었다고 했는데
왜 docker2와 docker3에서도 nginx가 뜨는 걸까요?** 
실제 컨테이너는 2개가 생성되어있고, *서버 2개*에서 트래픽을 처리하고 있음: 들어가는 주소는 내가 요청한 주소로 들어가지만, Cluster가 되어있기 때문에 자동으로 해당 서비스가 있는 곳으로 접속이 된다. 

//연결되는 동작 그림 

**트래픽 증가하는 경우 컨테이너 scale out** 

```bash
$ docker service scale web=4 
$ docekr service ps web

/*줄이는 것도 가능*/
$ docker service scale web=1
```

Overlay network 구성 및 확인

```bash
$ docker network create --driver=overlay --attachable web
$ docker network ls
```

### Docker Swarm, Haproxy, Nginx를 활용한 웹서비스와 로드밸런싱

**스택 파일 생성**

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

**해석** | 

- constraints: WORKER 서버 3대에만 nginx기반 컨테이너를 배포하고, manager서버에는 proxy만 배포
- restart policy: 재부팅을 시도하라는 명령어
- volumes: 각 worker 서버 잘 실행되는지 감시하기 위해 
*"/var/run/docker.sock"* : Docker CLI가 데몬에게 명령어를 전달할 수 있는 인터페이스 
/var/run/docker.sock:/var/run/docker.sock
- global: 호스트별로 각 1개씩 나눠주기

### Stack Deploy

```bash
$ docker stack deploy --compose-file=web.yml web
```
