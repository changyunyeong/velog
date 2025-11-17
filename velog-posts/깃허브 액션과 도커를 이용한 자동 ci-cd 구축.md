<p>결론부터 말하자면 deploy yml은 아래처럼 작성하면 된다 </p>
<br /> 


<pre><code>name: {Project Name} CI/CD

on:
  push:
    branches: [ &quot;main&quot;, &quot;develop&quot; ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:

      #------------------- CI --------------------

      - uses: actions/checkout@v3
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'


      - name: Grant execute permission for gradlew
        run: chmod +x ./malhaedo/gradlew

      # Gradle Caching
      - name: gradle caching
        uses: actions/cache@v3
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-

      # application.yml 파일 생성
      - name: make application.yml
        run: |
          mkdir -p ./malhaedo/src/main/resources
          cd ./malhaedo/src/main/resources
          touch ./application.yml
          echo &quot;${{ secrets.APPLICATION_YML }}&quot; &gt; ./application.yml
        shell: bash

      # Spring Boot 어플리케이션 Build
      - name: Spring Boot Build
        run: |
          cd malhaedo
          ./gradlew clean build -x test

      # DockerHub Login
      - name: docker login
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      # Docker 이미지 Build/Push
      - name: Build/Push Docker Image
        run: |
          docker build -t ${{ secrets.DOCKER_USERNAME }}/${{ secrets.DOCKER_IMAGE }} .
          docker push ${{ secrets.DOCKER_USERNAME }}/${{ secrets.DOCKER_IMAGE }}

      #------------------- CD --------------------

      # EC2 접속 후 서버 배포
      - name: Deploy to Server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.EC2_HOST }}
          key: ${{ secrets.EC2_SSH_KEY }}
          username: ${{ secrets.EC2_USERNAME }}
          port: ${{ secrets.EC2_SSH_PORT }}
          script: |
            sudo docker stop ${{ secrets.DOCKER_IMAGE }}
            sudo docker rm ${{ secrets.DOCKER_IMAGE }}
            sudo docker pull ${{ secrets.DOCKER_USERNAME }}/${{ secrets.DOCKER_IMAGE }}
            sudo docker run -d -p 8080:8080 --name ${{ secrets.DOCKER_IMAGE }} ${{ secrets.DOCKER_USERNAME }}/${{ secrets.DOCKER_IMAGE }}
            sudo docker image prune -f</code></pre><br /> 


<p>yml 파일은 Actions에서 적당한 템플릿 클릭하면 프로젝트 구조에 맞게 알아서 최상위 폴더에 설정된다
Github Secret은 Settings -&gt; Secrets and Variables -&gt; Actions -&gt; Repository secrets 에서 설정</p>
<p><code>APPLICATION_YML</code> : application yml을 gitignore에 설정해놨기 때문에 따로 secret에 저장해두어야 한다
<code>DOCKER_USERNAME</code> , <code>DOCKER_PASSWORD</code> : 말 그대로 도커 username과 password
<code>DOCKER_IMAGE</code> : 도커 이미지 이름인데 아래에 더 자세한 설명이 있음
<code>EC2_HOST</code> : ec2에 연결된 고정 ip 주소 (도메인을 따로 설정했더라도 반드시 ip 주소여야 함)
<code>EC2_SSH_KEY</code> : ssh private key의 내용을 전부 넣으면 된다
<code>EC2_USERNAME</code> : EC2 접속할 사용자명, 내 경우에는 ubuntu
<code>EC2_SSH_PORT</code> : ssh 접속 포트 기본값은 22이다</p>
<p>이 중 하나라도 누락하면 action이 실패하므로 꼭 확인하자 (나같은 경우에는 username을 빼먹어서 두 시간 헛고생한...)</p>
<p>앞서 언급했던 도커에 대해 조금 더 자세히 설명해보겠다
감사하게도 <a href="https://velog.io/@leestana01/Docker-2-Create-and-Push-DockerFile">해당 벨로그</a>에서 많은 도움을 받았다!</p>
<p>프로젝트 파일 최상단에 Docker 파일을 작성해야 함! 인텔리제이에서 손쉽게 작성할 수 있다
docker에 로그인 후 repository를 만들면 사전 세팅 완료</p>
<br /> 



<pre><code>FROM openjdk:17-jdk-alpine

# 인자 설정 - Jar_File
ARG JAR_FILE=malhaedo/build/libs/*.jar

# jar 파일 복제
COPY ${JAR_FILE} app.jar

# 실행 명령어
ENTRYPOINT [&quot;java&quot;, &quot;-Dspring.profiles.active=docker&quot;, &quot;-jar&quot;, &quot;app.jar&quot;]</code></pre><br /> 

<p><code>docker build -t 도커허브계정명/프로젝트이름(레포지토리 이름) .</code> 로 image file을 만들 수 있다 
이후 터미널에
<code>docker login</code>을 입력하고 로그인 진행 후 <code>docker push 도커허브계정명/프로젝트이름</code>으로 push 하면 모든 준비는 끝났다</p>
<p>이제 main/develop 브랜치에 push 하면 자동으로 수정사항이 반영이 되며 배포가 진행된다!</p>
<p><a href="https://velog.io/@b1rdn2w/%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%ED%98%91%EC%97%85-Spring-Boot-%EC%84%9C%EB%B2%84-%EA%B5%AC%EC%B6%95-3-AWS-EC2-HTTPS-%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0#route-53%EC%97%90-domain-%EC%A0%81%EC%9A%A9-%ED%9B%84-ns-%EB%93%B1%EB%A1%9D">+ https 설정하는 법</a></p>