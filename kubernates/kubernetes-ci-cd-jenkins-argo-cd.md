# Kubernetes CI/CD \[jenkins, argo cd]

## Jenkins & kustomize & Argo CD

### 개요

Docker compose를 사용했던 인프라에서 Kubernetes 인프라로 변경되면서 CI/CD 구조가 변경되었다.

CI/CD 구조로 가장 많이 검색되는 Jenkins-K8S CI/CD 방식을 검토하고 테스트하였다.

Jenkins - kustomize - Argo CD - Kubernetes

* source code repo와 manifest repo는 서로 다른 것임을 인지해야한다.
* manifest를 업데이트하는 방법으로 kustomize를 사용하였다.

> 컨테이너 이미지의 태그 버전을 기반으로 argo CD에서 이미지 변경을 추적하는 방법이 가장 많이 검색되었다. 백엔드 팀의 경우 이미지 tag 버전에 마이너 버전을 추가하는 방식으로 테스트하였다. 마이너 버전은 jenkins의 빌드 번호를 사용하여 빌드마다 tag가 겹치지 않도록 하였다. ex ) 172.16.28.217:12000/ndxpro-translator-manager:v1.3.5

***

### **Components**

#### **Jenkins**

* 젠킨스는 기존에 CI/CD를 모두 담당하고 있었다.
* 젠킨스 자체적으로는 git을 기반으로한 CD를 구성하기 어려우므로 CD는 argo CD에 인계한다.
* 고로 젠킨스에서는 CI. 즉, 코드 버전 관리, 테스트, Jar 빌드, 컨테이너 이미지 빌드 등을 수행한다.
* kustomize를 통해 새로운 컨테이터 이미지 tag를 argo CD로 전달한다.
* argo CD가 추적하고 있는 git repository의 manifest를 수정하고 merge하는 과정을 통해 argo CD에 작업을 인계한다.
* 별도의 플러그인을 추가하여 사용하지는 않았다.
* pipeline script 예시

```groovy
node {
    def gitBranch = env.gitlabSourceBranch ?: 'develop'
    stage ('git clone') {
				dir('code') {
    		    git branch: gitBranch,
                credentialsId: 'gitlab_jenkins_credentials',
                url: '<http://172.16.28.218/platform/ndxpro-ngsi-translator-manager.git>'    
		    }
    }
    stage ('junit test') {
        //set jdk
        jdk = tool name: 'azul_jdk_21'
        env.JAVA_HOME = "${jdk}"
        env.PATH = "${JAVA_HOME}:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        
        dir('code') {
            //test
            sh "./gradlew test"
            junit '**/build/test-results/test/*.xml'   
        }
    }
    stage ('build & upload docker image [gradle jib]') {
		    //set jdk
        jdk = tool name: 'azul_jdk_21'
        env.JAVA_HOME = "${jdk}"
        env.PATH = "${JAVA_HOME}:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            
    		//build docker image & push
    		dir('code') {
    		    sh './gradlew jib -DsendCredentialsOverHttp=true -Ptag=v1.3.${BUILD_NUMBER}'      
    		}
    }
    stage ('clone argo project') {
        dir('argo') {
       			git branch: 'kubernetes',
            		credentialsId: 'gitlab_jenkins_credentials',
            		url: '<http://172.16.28.218/platform/ndxpro-k8s.git>' 
        }
    }
    stage ('edit manifest by using kustomize') {
		    gitlabCommitStatus(connection: gitLabConnection(gitLabConnection: '플랫폼 개발팀 Gitlab 저장소 Connection', jobCredentialId: ''), name: 'build') {
                dir('argo/kustomization/base') {
                    sh 'kustomize edit set image 172.16.28.217:12000/ndxpro-translator-manager:v1.3.${BUILD_NUMBER}'
                    withCredentials([gitUsernamePassword(credentialsId: 'gitlab_jenkins_credentials')]) {
                        sh '''
                        git config user.name "jenkins"
                        git config user.email "jenkins@e8ight.co.kr"
                        git add kustomization.yaml
                        git commit -m "NDXPRO Translator image version to v1.3.${BUILD_NUMBER}"
                        git push --set-upstream origin kubernetes
                        '''
                    }
                }
        }
    }
}
```

* 이미지 빌드, 배포 단계는 이전과 유사하다.
* 이미지 빌드시 태그를 설정하는 내용만 추가되었다. ( -Ptag=v.1.3.${BUILD\_NUMBER} )
  * 백엔드에서는 위와 같이 jib의 파라미터로 이미지 tag를 입력하며
  * 프론트엔드에서는 docker build 명령어 파라미터로 이미지 tag를 입력할 수 있을 것 같다.
* docker가 실행되던 서버에서 docker를 재실행하는 명령을 보내는 대신 argo CD git repository를 clone하고 내용 수정, push 하는 절차로 변경되었다.

***

#### **Kustomize**

*   Kubernetes manifest를 선언형 명령으로 수정할 수 있는 도구이다.

    예를들어 Kustomize를 사용하지 않는 상황에서 아래와 같은 Deployment 선언 yaml 파일이 있을 때

    ```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ndxpro-translator-manager
      namespace: ndxpro
      labels:
        data: ndxpro-translator-manager
    spec:
      replicas: 1
      selector:
        matchLabels:
          data: ndxpro-translator-manager
      strategy: {}
      template:
        metadata:
            labels:
                data: ndxpro-translator-manager
        spec:
          volumes:
            - name: container-log-file
              persistentVolumeClaim:
                claimName: container-logs-pvc
            - name: translator-jars
              persistentVolumeClaim:
                claimName: translator-jars-pvc
          containers:
            - name: ndxpro-translator-manager
              image: 172.16.28.217:12000/ndxpro-translator-manager:v1.3
              imagePullPolicy: Always
              ports:
                - containerPort: 8080
              envFrom:
                - configMapRef:
                    name: ndxpro-configmap
              volumeMounts:
                - mountPath: /opt/logs
                  name: container-log-file
                  subPath: translator-manager-log-file
                - mountPath: /deploy/translatorJars
                  name: translator-jars
          restartPolicy: Always
    ```

    jenkins 파이프라인에서 파드의 컨테이너 이미지를 변경하려면 shell의 sed같은 명령어를 통해 파일을 수정해야 한다.

    `cat ndxpro-translator-manager.yml | sed 's/ndxpro-translator-manager:v1.3/ndxpro-translator-manager:v1.3.6/g'`

    이는 비교적 오류가 발생할 수 있고 사용하기 번거롭다.

    kustomize는 이런 문제를 쉽게 해결해준다.

    아래는 ndxpro-translator-manager 및 ndxpro-ingestion-agent의 이미지 tag를 변경하는 kustomization.yaml 파일 예시이다.

    ```
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
    - ndxpro-translator-manager.yml
    - ndxpro-ingestion-agent.yml
    images:
    - name: 172.16.28.217:12000/ndxpro-translator-manager
      newTag: v1.3.15
    - name: 172.16.28.217:12000/ndxpro-ingestion-agent
      newTag: v1.3.3
    ```

    Kustomization은 수정이 필요한 resource의 yaml 파일들에 대해서 (예시에서는 위의 ndxpro-translator-manager.yml 파일) 원본 manifest의 수정없이 Kustomization에 정의되어있는 수정 내용에 따라 Kubernetes 객체가 생성되도록 도와준다.

    위 예시에서는 Kustomization를 통해 ndxpro-translator-manager가 v.1.3.15 버전의 이미지를 사용하여 파드를 생성한다.

    또한 이 파일은 kustomize의 CLI도구를 통해 수정이 가능한데, sed등을 사용하는 것 보다 훨신 간단하고 명시적이게 수정할 수 있다.

    예를 들어 위의 kustomization.yaml을 수정할 때는 아래와 같은 명령어를 사용한다.

    `kustomize edit set image 172.16.28.217:12000/ndxpro-translator-manager:v1.3.16`

    이 명령어를 실행하면 kustomization.yaml 파일의 내용이 수정된다.

    ```
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
    - ndxpro-translator-manager.yml
    - ndxpro-ingestion-agent.yml
    images:
    - name: 172.16.28.217:12000/ndxpro-translator-manager
      newTag: v1.3.16   //이 부분이 변경됨
    - name: 172.16.28.217:12000/ndxpro-ingestion-agent
      newTag: v1.3.3
    ```
* 프로파일에 따라 배포 구조를 다르게 하는 기능도 있다. 프로파일 마다 별도의 디렉토리 관리가 필요하며, 원본 manifest 하나에서 프로파일에 따라 조금씩만 개정이 필요한 경우 용이하게 사용할 수 있다.
* kustomize 설치 방법 : [https://kubectl.docs.kubernetes.io/installation/kustomize/binaries/](https://kubectl.docs.kubernetes.io/installation/kustomize/binaries/)
* kustomize를 사용한 kubernetes 객체 관리 : [https://kubectl.docs.kubernetes.io/guides/config\_management/](https://kubectl.docs.kubernetes.io/guides/config_management/)
* jenkins 서버에는 CI 작업중 이미지 tag 수정을 위해 kustomize를 미리 설치해두었다.

***

#### **Argo CD**

* Argo CD는 git 기반의 Kubernetes CD를 담당한다.
* Git repository에 있는 manifest의 내용을 기반으로 파드, 서비스, 볼륨 등등 Kubernetes의 객체를 생성하고 노드에 할당해주는 것이다.
* Git repository의 manifest 내용이 수정되어 merge 되면 github 또는 gitlab의 webhook을 통해 argo CD에서 변경을 인지하고
* 변경 내용에 따라 Kubernetes 객체들을 재구성한다.
* Argo CD는 kustomize를 사용한 Kubernetes CD를 지원한다.

<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

***

### 구성 절차 요약

1. ndxpro-k8s repository의 kubernetes 브랜치에 kustomization.yaml 파일 구성
2. 기존 deploy yaml 파일은 변경할 필요 없음
3. 백엔드의 경우 project build.gradle에 tags 내용을 프로퍼티로 입력할 수 있도록 수정

```groovy
jib {
    from{
        image = "openjdk:21-jdk-slim"
    }
    to {
        image = "172.16.28.217:12000/ndxpro-translator-manager"
        tags = [project.hasProperty('tag') ? project.findProperty('tag'): 'latest'] as List<String>
        auth {
            username = "admin"
            password = "ndxpro123!"
        }
        container {
            mainClass ="kr.co.e8ight.ndxpro.translatormanager.TranslatorManagerApplication"
            ports = ["8080"]
            environment = ["TZ" : "Asia/Seoul"]
        }
    }
    setAllowInsecureRegistries(true)
}
```

1. 젠킨스 스크립트 수정

***

### CI/CD 진행 절차 요약

1. GitLab
   * Web hook : MR 내용을 젠킨스로
2. Jenkins
   * source code repo git clone
   * 테스트
   * 컨테이너 이미지 빌드
   * nexus에 신규 이미지 배포
   * kustomize를 통해 kustomization.yaml 파일의 이미지 태그 수정
   * argo CD가 추적하는 git branch에 merge
3. GitLab
   * Web hook : Merge 내용을 argo CD로
4. Argo CD
   * 변경 인지
   * kustomization.yaml 파일의 변경 내역을 기반으로 Kubernetes 객체 재배포

참고 :

kustomize

[https://kubectl.docs.kubernetes.io/guides/](https://kubectl.docs.kubernetes.io/guides/)

jenkins-argocd

[https://sdg9670.github.io/devops/kustomize-argocd/](https://sdg9670.github.io/devops/kustomize-argocd/)

[https://velog.io/@yellowsunn/Jenkins-ArgoCD-로-k8s-CICD-파이프라인-구축하기](https://velog.io/@yellowsunn/Jenkins-ArgoCD-%EB%A1%9C-k8s-CICD-%ED%8C%8C%EC%9D%B4%ED%94%84%EB%9D%BC%EC%9D%B8-%EA%B5%AC%EC%B6%95%ED%95%98%EA%B8%B0)

[https://cwal.tistory.com/22](https://cwal.tistory.com/22)
