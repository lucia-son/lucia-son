
## Similar Commands on Docker 

자세히 보아야 다르다.

> **Dockerfile** 

- **CMD와 ENTRYPOINT**

CMD: RUN 명령어와 겹치는 경우, RUN의 명령어가 우선시

ENTRYPOINT: RUN명령어와 겹치는 경우 ENTRYPOINT 명령어 우선시  

- **ONBUILD ADD & COPY**

ONBUILD ADD: tar 파일의 경우 패키지가 풀린 상태로 붙여진다. 

ONBUILD COPY: tar 파일의 경우 패키지 파일 그대로 붙여진다. 

> **docker-compose.yml**

- **image와 build**

image: 이미 존재하는 이미지를 사용

build: 이미지를 빌드한 다음, 해당 이미지를 사용

- **expose와 ports**

expose: 웹에 공개하는 것이 아님, 내부 연결

ports: 웹에 공개한다는 것



