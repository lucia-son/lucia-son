## XE-CORE 설치된 컨테이너 생성

그림 참고: [autodraw](https://www.autodraw.com/)



### Overview

>**index.html** : 
>  /var/www/html/index.html 로 접속 시 현재 디렉토리 아래의 xe-core/index.php 파일로 REDIRECT 설정

> **Dockerfile**: 
> - ubuntu 설치, apache 설치, php 설치, xe-core를 다운로드
> - 컨테이너 배포시 apache실행되도록 설정하기 

> **docker-compose.yml** : 컨테이너 생성


### **1. index.html (redirection 용)**

```html
<html>
 <head> 
        <title>TEST PAGE</title>
        <meta http-equiv="refresh" content="0; url=xe-core/index.php" />
//0; 지연 0초 후=지연없이 
</head>
</html>
```

### 2. Dockerfile

```bash
FROM ubuntu:18.04
ENV DEBIAN_FRONTEND=noninteractive
"설치할 때 지역같은 거 물어보지 마세요. 모든 것을 기본값으로 설정하여 설치" 
RUN apt-get update
RUN apt-get -y install apache2
RUN apt-get install -y git software-properties-common
RUN add-apt-repository ppa:ondrej/php#
RUN apt-get update 
RUN apt-get -y install php5.6 php5.6-common php5.6-mysql php5.6-gd php5.6-fpm php5.6-xml libapache2-mod-php5.6
WORKDIR /var/www/html
RUN git clone https://github.com/xpressengine/xe-core.git
ADD index.html /var/www/html
"현재 내 로컬 컴퓨터에 있는 index.html을 생성될 이미지의 
/var/www/html로 붙이겠다"
"index.html<index> -> /var/www/html/xe-core/index.php"
RUN chmod 777 -R /var/www/html/xe-core
EXPOSE 80
CMD ["/usr/sbin/apachectl", "-D", "FOREGROUND"] 
"FOREGROUND로 실행될 것을 background(-D)로 실행하겠다"
```

> **Question. 왜  apachectl 이죠?** 
> **ubuntu에서는 apache2 , Centos에서는 httpd 를 사용한다.** 


### 3. docker-compose.yml

```bash
version: "3"

services:
  xe3:
    image: rapa/xe:1.0
    ports:
      - "8001:80"
    links:
      - "db3"
    depends_on:
      - "db3"

  db3:
    image: mariadb
    volumes:
      - ./mariadb:/var/lib/mysql
/*mariadb가 없는 관계로 자동으로 생성되면서 볼륨 연결 완료*/
    environment:
      - MYSQL_ROOT_PASSWORD=xe
      - MYSQL_DATABASE=xe
      - MYSQL_USER=xe
      - MYSQL_PASSWORD=xe
    expose:
      - "3306"
```

##### *docker-compose.yml 작성시 형식 주의*
```bash
"Hash 형식"
   MYSQL_ROOT_PASSWORD: xe
"배열 형식"
   - MYSQL_ROOT_PASSWORD=xe
```
