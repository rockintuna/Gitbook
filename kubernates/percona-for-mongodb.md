# Percona for mongodb 구성하기

**Percona for mongodb**

MongoDB Community Edition 기반으로 구축되었으며 몇가지 엔터프라이즈급 향상 기능이 포함되어 있습니다.

* **암호화 WiredTiger 스토리지 엔진**
* 외부인증 및 권한 부여
* **사용자 또는 애플리케이션의 감사 로깅, 로그 편집 및 쿼리 데이터베이스 상호 작용**
* [\*\*MongoDB용 Percona Operator](https://www.percona.com/software/percona-kubernetes-operators) 와 호환\*\*
  * kubernates 환경에서 mongodb를 운영할 때 가장 많이 사용하는 operator 입니다.
  * kubernates 위에서 mongodb replica / sharding 구성을 쉽게 배포할 수 있습니다.

**Mongodb Sharded Cluster의 요소**

![](<../.gitbook/assets/image (2).png>)

Shard(replica set) : 데이터의 하위 집합(샤딩된 집합)을 포함하는

mongos : 외부 접근용 인터페이스 역할을 하는 라우터, 각 shard의 데이터로부터 쿼리해주는 역할

config server : 클러스터에 대한 메타데이터 구성 저장

Percona for mongodb 기본 구성

* mongos 파드 3개
* config server 파드 3개
* shard 파드 3개

**Percona for mongodb 설치**

Apply CRD

```
kubectl apply --server-side -f deploy/crd.yaml
```

Create namespace

```
kubectl create namespace percona-mongo
kubectl config set-context $(kubectl config current-context) --namespace=percona-mongo
```

Apply RBAC

```
kubectl apply -f deploy/rbac.yaml
```

Apply Operator

```
kubectl apply -f deploy/operator.yaml
```

Secret 수정 및 생성

```
vi deploy/secrets.yaml
kubectl create -f deploy/secrets.yaml
```

PV 생성

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: percona-mongo-cfg-pv-1
  namespace: percona-mongo
spec:
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: percona-mongo-cfg
  nfs:
    path: /percona_mongo/cfg_1
    server: 172.16.28.235
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: percona-mongo-cfg-pv-2
  namespace: percona-mongo
spec:
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: percona-mongo-cfg
  nfs:
    path: /percona_mongo/cfg_2
    server: 172.16.28.235
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: percona-mongo-cfg-pv-3
  namespace: percona-mongo
spec:
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: percona-mongo-cfg
  nfs:
    path: /percona_mongo/cfg_3
    server: 172.16.28.235
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: percona-mongo-rs-pv-1
  namespace: percona-mongo
spec:
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: percona-mongo-rs0
  nfs:
    path: /percona_mongo/rs_1
    server: 172.16.28.235
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: percona-mongo-rs-pv-2
  namespace: percona-mongo
spec:
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: percona-mongo-rs0
  nfs:
    path: /percona_mongo/rs_2
    server: 172.16.28.235
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: percona-mongo-rs-pv-3
  namespace: percona-mongo
spec:
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: percona-mongo-rs0
  nfs:
    path: /percona_mongo/rs_3
    server: 172.16.28.235
```

Storage Class 생성

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: percona-mongo-cfg
provisioner: kubernetes.io/nfs
allowVolumeExpansion: false
```

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: percona-mongo-rs0
provisioner: kubernetes.io/nfs
allowVolumeExpansion: false
```

deploy/cr.yaml 수정 \[각 pvc의 storageClassName 추가]

```
          resources:
            requests:
              storage: 3Gi
          storageClassName: percona-mongo-rs0

          resources:
            requests:
              storage: 3Gi
          storageClassName: percona-mongo-cfg
```

worker node가 2개라서 각 파드의 affinity를 수정

기본적으로는 worker node 별로 파드가 생성되도록 anti-affinity가 설정되어 있음

```
      affinity:
        antiAffinityTopologyKey: "none"
#        antiAffinityTopologyKey: "kubernetes.io/hostname"
```

Percona for Mongo Cluster 생성

```
k apply -f deploy/cr.yaml
```

테스트

```
kubectl run -i --rm --tty percona-client --image=percona/percona-server-mongodb:7.0.14-8 --restart=Never -- bash -il
mongosh "mongodb://my-cluster-name-mongos.percona-mongo.svc.cluster.local/admin?ssl=false"
```

node port 생성

```
apiVersion: v1
kind: Service
metadata:
  name: percona-mongo-svc
  namespace: percona-mongo
spec:
  ports:
    - name: "mongos-node-port"
      port: 27017
      targetPort: 27017
      nodePort: 37017
  selector:
    app.kubernetes.io/component: mongos
    app.kubernetes.io/instance: my-cluster-name
    app.kubernetes.io/managed-by: percona-server-mongodb-operator
    app.kubernetes.io/name: percona-server-mongodb
    app.kubernetes.io/part-of: percona-server-mongodb
  type: NodePort
status:
  loadBalancer: {}
```

참고 :

[https://docs.percona.com/percona-operator-for-mongodb/kubernetes.html](https://docs.percona.com/percona-operator-for-mongodb/kubernetes.html)

[https://docs.percona.com/percona-operator-for-mongodb/options.html](https://docs.percona.com/percona-operator-for-mongodb/options.html)
