# Coredns host unreachable

dnsutils에서 nslookup을 통해 coredns를 점검할 수 있다.

```
[root@k8s-mn e8ight]# kubectl exec dnsutils -n default -it -- nslookup ndxpro-eureka.ndxpro
Server:   10.96.0.10
Address:  10.96.0.10#53
​
Name: ndxpro-eureka.ndxpro.svc.cluster.local
Address: 10.103.249.194
```

나의 경우 host unreachable 문제가 간헐적으로 발생하고 있었는데,

```
ube-system            coredns-7cc7cd66d9-5g8wv                           1/1     Running                  0                9m10s   192.168.210.24    k8s-wn-2   <none>           <none>
kube-system            coredns-7cc7cd66d9-jvr8t                           1/1     Running                  0                10m     192.168.237.168   k8s-wn-1   <none>           <none>
```

coredns 파드가 생성되어있는 노드중 하나의 방화벽 문제인 것으로 확인되었다.

방화벽 설정이 잘 되어있는 coredns에서는 FQDN, IP를 잘 받아왔고 방화벽 설정이 이상하게 되어있는 coredns에서는 아래와 같이 host unreachable 에러가 발생했다.

```
;; communications error to 10.96.0.10#53: host unreachable
;; Got recursion not available from 10.96.0.10
;; Got recursion not available from 10.96.0.10
;; communications error to 10.96.0.10#53: host unreachable
;; communications error to 10.96.0.10#53: host unreachable
;; no servers could be reached
```

방화벽 설정 후 잘 작동 중.
