---
layout: post
title: Docker & Kubenates Tip
excerpt: "개발하며 필요한 Docker, kubenates 명렁어나 설정등에 대한 tip 모음"
tags: [tip, queue]
modified: 2020-07-07
comments: true
category : etc
---

{% include _toc.html %}


Docker
==============================

도커 사설 레파지토리 로그인
~~~
docker login registry.private.com
docker logout 
~~~

도커이미지 빌드
~~~
docker build -t {tagName} .
~~~

도커이미지 태그
~~~
docker tag {imageId} {tagName}
~~~

도커이미지 remote 저장소 올리기 (login 필요)
~~~
docker push {registryPath}
~~~

도커이미지 중 <none>:<none> 상태 이미지 제거
~~~
docker image prune
~~~
  
도커이미지 컨테이너 모두 삭제
~~~
docker container rm $(docker container ls -aq)
~~~


### Dockerfile 샘플

~~~~
FROM openjdk:8-jdk-alpine
VOLUME /tmp
ARG JAR_FILE
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
~~~~~

~~~~~
FROM openjdk:11
VOLUME /tmp
COPY target/application-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
EXPOSE 8080
~~~~~

도커 이미지 확인
~~~
docker images
~~~

도커 이미지 삭제 ( -f는 강제 )
~~~
docker rmi {이미지ID} -f
~~~

도커 컨테이너
~~~
docker ps
~~~

도커 실행 ( 외부 8080 포트를 내부 expose된 8080포트로 연결 )
~~~
docker run -i -t -p 8080:8080 --name {name} {imageTag} /bin/ba
~~~


도커 컨테이너 삭제/정지
~~~
docker rm {컨테이너ID}
docker stop e9b7a01b4f04
~~~




Kubenates
==============================


쿠버네이트 이미지 실행 ( 파일없이 직접 )
~~~
kubectl run platform-api --image={imageTag} --replicas=1 --port 8080
~~~

서비스 생성 ( NodePort - 내부, LoadBalancer - 외부 )
~~~
kubectl expose deployment platform-api --target-port=8080 --type=NodePort
kubectl expose deployment platform-api --target-port=8080 --type=LoadBalancer
~~~


서비스 확인
~~~
kubectl get svc
~~~

서비스 제거
~~~
kubectl delete svc -l run={appName}
~~~

Pod확인
~~~
kubectl get pods
kubectl get pods -n {namespace}
~~~

Pod제거
~~~
kubectl delete deployment {podName}
~~~

Deploy확인
~~~
kubectl get deployments
~~~

ReplicaSet 확인
~~~
kubectl get rs
~~~

pod 생성 ( 파일로 생성 - 선언적 인프라스트럭쳐 )
~~~
kubectl create -f Pod.yaml
~~~

Pod.yml
~~~
apiVersion: v1
kind: Pod
metadata:
  name: {podName}
spec:
  containers:
  - name: {containerName}
    image: {imageName}:{imageVersion}
    command: ["/bin/sh"]
    args: ["-c", "sleep 10 && echo Sleep expired > /dev/termination-log"]
~~~


특정 Selector 조건에 걸리는 리소스 찾기
~~~
kubectl get pod,replicaset,deployment --selector=app.kubernetes.io/instance=test
~~~

설정확인
~~~
## 상태정보 같이 조회
kubectl get {pod|deploy|svc...} {resourceName} -o yaml
# 설정만 조회
kubectl get {pod|deploy|svc...} {resourceName} -o yaml --export
~~~

설정파일의 정합성 체크와 실행 예상결과 확인
(정합성체크 안하면 틀린문법은 무시되고 실행됨)
~~~
kubectl apply -f {fileName} --validate
kubectl apply -f {fileName} --dry-run
~~~

### 라벨과 어노테이션

라벨
: 팟, 디플로이먼트, 서비스 등 쿠버네티스 객체를 그룹화하고 구성하기 위해 사용


어노테이션
: 데이터 추가를 통해 특정 인스턴스에 대한 부가정보 제공

### 컨피그맵과 시크릿


컨피그맵
: env, 볼륨마운트 등의 방법으로 별도 key=value의 설정을 세팅할 수 있는 리소스
: 직접 설정 및 파일로 설정할 수 있다.

컨피그맵 생성
~~~
kubectl create configmap test-config --from-literal=log.level=err
~~~

~~~
kubectl get configmap
kubectl describe configmap test-config
~~~

설정매핑
~~~
envFrom:
  - configMapRef:
    name: example-config
~~~

설정매핑(특정 key)
~~~
valueFrom:
  - configMapKeyRef:
    name: example
~~~    

시크릿
: env, 볼륨마운트 등의 방법으로 세팅할 수 있지만 password, key값 등 보안을 요하는 정보를 kubenate자체에 보관 할 수 있도록 해주는 리소스
: 직접 설정 (파일은 가능할 순 있어도 노출 위험)

생성
~~~
kubectl create secret generic test-creds --from-file=username.txt --from-file=password.txt
~~~

조회
~~~
kubectl get secret
kubectl get secret test-creds -o yaml --export
~~~

설정매핑 (특정 key)
~~~
valueFrom:
  secretKeyRef:
    name: database-creds
    key: password.txt
~~~


### 롤링 업데이트 ( deployment 를 변경함)

추가버젼을 기록
~~~
kubectl --record deployment.apps/{컨테이너명} set image deployment.v1.apps/{컨테이너명} {컨테이너명}={docker이미지명}:{version}
~~~


롤아웃 실행
~~~
kubectl rollout status deployment.v1.apps/{컨테이너명}
~~~

1개인 경우 안내려가는 경우가 있는데, 아래 설정 추가 ( deployment )
~~~
  strategy:
    type: RollingUpdate
    rollingUpdate:
       maxSurge: 0
       maxUnavailable: 1
~~~

(꼭 버져닝을 해주고 레파지토리에 올리고, 반영해야 적용된다.
적용이 안되는 경우도 있는데, get pods 로 반영시간을 확인하면 됨)



리플리카셋 변경
~~~
kubectl scale --current-replicas=1 --replicas=2 deployment/{deploy명}
~~~


로그 확인
~~~
kubectl logs {POD명} -f
~~~


pod 접근
~~~
kubectl exec -it {POD명} -- /bin/bash
~~~

pod안에 container 접근
~~~
kubectl exec -it api-default-559c85dddc-jh46r -c api-default-nginx -- /bin/bash
~~~


리소스 제약설정
~~~
resources:
   limits:
      cpu: "1"
      memory: 1Gi
   requests:
      cpu: "0.1"
      memory: 100Mi
~~~


### 기타

#### jsonPath

jsonpath 문법을 통해서 특정 부분 데이터만 파싱해서 가져올 수 있다.

~~~
kubectl get pods -l app=test -o jsonpath='{.items[*].metadata.name}'
~~~

