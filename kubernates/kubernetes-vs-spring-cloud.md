# Kubernetes vs Spring Cloud

### 개요

기존 인프라를 Kubernetes로 변환하면서

Kubernetes의 객체들이 MSA를 위해 사용하던 Spring Cloud 서비스들의 역할을 대체할 수 있게 되었다.

물론 둘다 함께 사용할 수는 있겠지만 동일한 역할을 하는 요소들이 여러개 있게 되므로

* 동일한 작업을 두번 할 수 있다.
* 구조가 복잡해진다.
* 디버깅이 어려워진다.
* 불필요한 자원 사용량이 증가할 수 있다.

<figure><img src="../.gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

따라서 우리 팀도 어떤 기술을 사용해야 할지 결정하는 것이 좋을 것 같다.

### 내용

필요 기술과 그에 대한 스펙은 Spring Cloud와 Kubernetes에서 아래와 같이 구분된다.

<figure><img src="../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

[https://velog.io/@mdev97/Project-Spring-Cloud-Kubernetes](https://velog.io/@mdev97/Project-Spring-Cloud-Kubernetes)

현재 우리는 Spring Cloud Gateway, Eureka, OpenFeign을 사용하고 있다.

Spring Cloud Gateway의 경우 Ingress가 대체할 수 있다.

두 스펙 모두 L7에서 요청을 라우팅하는 기능을 제공한다.

다만, SCG에서 우리는 추가적인 Filter를 사용하여 인증 절차를 추가하였는데, Ingress에서 이를 구현하기 위해서는 Lua 코드를 작성해야 한다.

우리 팀은 Lua에 익숙하지 않으므로 인증 기능의 접목이 어려울 수 있다.

Spring Cloud Eureka의 경우 Service가 대체할 수 있다.

기존에 Eureka를 통해 각 컨테이너의 주소를 확인하고 로드밸런싱을 할 수 있었다면

Kubernetes의 Service는 Kubernetes의 내부 DNS를 통해 요청을 받고 해당 Service에 할당되어있는 Pod들로 요청을 분산할 수 있다.

내부 통신에 사용되는 OpenFeign은 서비스 레지스트리로부터 주소 정보를 받아 요청해주는 도구이므로 Kubernetes에서 동일하게 사용할 수 있다.

하지만 Eureka를 사용할 것인지, Service를 사용할 것인지에 따라 코드는 변경된다.

우리는 Spring Cloud Config 대신 docker compose의 .env 파일을 사용했었고

이는 ConfigMap으로 대체할 수 있었다.

***

우리 팀의 현재 상황에서 이런 분기에 몇 가지 고민되는 것이 있다.

1. 이제부터 설치하는 모든 제품은 Kubernetes 인프라로 제공되는가?
   1. 만약 아니라면 Kubernetes 환경에서 실행되는 코드와 Spring Cloud 환경에서 실행되는 코드가 둘다 관리되어야 한다.
   2. 서비스 코드의 property yaml 파일, OpenFeign을 사용하는 부분
   3. 이는 Spring 프로파일을 통해 처리할 수 있을 것 같긴 하다.
2. 만약 Ingress를 사용한다면 인증 처리는?
   1. [https://kubernetes.github.io/ingress-nginx/examples/auth/external-auth/](https://kubernetes.github.io/ingress-nginx/examples/auth/external-auth/)
3. Spring Cloud Gateway를 유지한다면 프론트엔드도 SCG를 통해 접근하게 되는가?

### 참고

NHN에서 기존 Spring Cloud 환경에서 Kubernetes로 전향하면서 코드 변경을 최소화 하기 위한 내용을 담은 영상

[https://www.youtube.com/watch?v=otss\_\_0kf-g](https://www.youtube.com/watch?v=otss__0kf-g)
