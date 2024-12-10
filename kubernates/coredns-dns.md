# Coredns에서 외부 DNS 서버 추가하기

K8s의 파드들은 기본적으로 coredns에서 각 파드 또는 서비스들의 도메인 주소를 통해 IP를 가져온다.



예를들어&#x20;

동일한 namespace 내 서비스에 대한 도메인주소는

{service name}.svc

다른 namespace의 서비스에 대한 도메인 주소는

{service name}.{namespace}.svc



만약, coredns에서 제공하는 클러스터 내 DNS 외에 다른 DNS 서버를 사용하고 싶다면 coredns config를 수정하여 해결할 수 있다.

```
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
    log
}

other.com:53 {
    forward . 172.16.10.222
    cache 30
}
```

coredns configmap에 위와 같이 설정해주면

`other.com`을 suffix로 들어오는 주소(ex. rockintuna.other.com)는 지정된 DNS 서버를 사용하여 IP 주소를  찾아오게 된다.

