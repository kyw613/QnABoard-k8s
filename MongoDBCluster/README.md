## ⭐ 필요 배경

현재 **Kubernetes 환경**에서 **3-Tier 애플리케이션**을 운영하고 있는 상황입니다.

- **Frontend**: NextJs
- **Backend**: Django
- **Database**: MongoDB




## **💡 문제점**

- `Frontend`와 `Backend`는 **Kubernetes Deployment를 활용**하여 `ReplicaSet`으로 배포되므로, Pod 장애 발생 시 자동 복구 가능
- 하지만 **Database는 단일 인스턴스로 운영할 경우, 장애 발생 시 전체 서비스 중단 위험**
    - 초기 테스트에서 **100명 이상 동시 접속 시 서비스가 다운되는 현상**이 반복
- 데이터베이스의 고가용성을 확보하지 않으면 **운영 중 데이터 유실 및 서비스 다운 가능성 존재**




## 🔍 구성 시 중요 고려 사항

1. **장애 복구 시 데이터 무결성 유지**
    - 장애 발생 후 재시작 시에도 데이터가 정상적으로 복구되는가?
2.  **Fail-over 기능을 통한 가용성 유지**
    - 만약 Primary DB가 다운될 경우, 자동으로 다른 노드가 Primary 역할을 수행할 수 있는가?
3.  **내·외부 연결 구조 최적화**
    - 내부에서는 **ClusterIP**를 사용하여 통신
    - 외부 접근이 필요한 경우, **Primary 노드만 NodePort를 사용하여 연동**
    - `Robo 3T` 등 외부 관리 도구를 활용하여 접근 가능하도록 구성
4. **운영 비용 및 확장성 고려**
    - 단순한 DB 인스턴스 증설이 아닌, **장기적인 확장성을 고려한 구조 설계**
    - 클러스터링 및 읽기/쓰기 분리를 통해 **추가 인프라 비용 최소화**




### **✅ 해결 방안**

**StatefulSet과 MongoDB의 ReplicaSet을 활용하여 DB 클러스터를 구성, 데이터베이스의 고가용성을 확보하였습니다.**

- StatefulSet을 사용하여 **Pod의 안정적인 네트워크 ID 및 볼륨 유지**
- MongoDB ReplicaSet을 구성하여 **Primary-Failover 구조 구축**
- Primary 노드 장애 시 자동 Failover → **Secondary 노드가 Primary로 승격**하여 지속적인 서비스 운영 가능

### [ Architecture ]

### **❖ StatefulSet 기반 MongoDB ReplicaSet 구성**

## **📌 PV (Persistent Volume) 설정**

이번 프로젝트에서는 **MongoDB ReplicaSet을 Kubernetes StatefulSet으로 구현**하면서, **각 Pod가 독립적인 Persistent Volume(PV)을 사용하도록 구성**하였습니다. 이를 통해 **데이터 유실을 방지하고, 장애 발생 시에도 복구가 가능**하도록 설계하였습니다.

### **📄 `pv.yaml` 파일**

```yaml
# 3개의 pod가 사용할 PV를 생성
# primary DB는 반드시 node1에 뜨도록 설정
# secondary는 랜덤
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv-1
spec:
  capacity:
    storage: 4G
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /home/student/mongodb1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - {key: kubernetes.io/hostname, operator: In, values: [k8s-node1]}
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv-2
spec:
  capacity:
    storage: 4G
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /home/student/mongodb2
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv-3
spec:
  capacity:
    storage: 4G
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /home/student/mongodb3

```

위 설정에서는 **Primary DB를 특정 노드(`k8s-node1`)에 배치**하고, **Secondary DB는 랜덤으로 배포**되도록 구성했습니다.

![Image](https://github.com/user-attachments/assets/8159f4b0-6a1c-4f92-85ff-67838b9d2328)



## **📌 StatefulSet 설정**

MongoDB의 **Primary/Secondary 구조를 유지하면서, 안정적인 데이터 저장과 복구를 보장하기 위해 StatefulSet을 활용**하였습니다.

### **📄 `StatefulSet.yaml` 파일**

```yaml
# StatefulSet이 뜨면서 자동으로 PVC 생성
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: mongodb  # 네임스페이스를 mongodb로 설정
spec:
  serviceName: mongodb-svc
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:4.2.3
        ports:
        - containerPort: 27017
        command:
        - "mongod"
        - "--bind_ip"
        - "0.0.0.0"
        - "--replSet"
        - "rs0"
        volumeMounts:
        - name: mongodb-data
          mountPath: /data/db
  volumeClaimTemplates:
  - metadata:
      name: mongodb-data
      namespace: mongodb
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi

```

여기서 **`volumeClaimTemplates`를 통해 PVC가 자동 생성**되도록 설정하여, 각 Pod가 자체적인 스토리지를 확보할 수 있도록 구성했습니다.

또한 **각 DB 인스턴스에 대한 직접 접근을 위한 Service 설정**도 추가했습니다.



## **📌 MongoDB ReplicaSet 설정 및 서비스 배포**

MongoDB의 Primary 및 Secondary 노드에 직접 접근할 수 있도록 NodePort 기반의 서비스를 추가했습니다.

```yaml
# PRIMARY MongoDB에 다이렉트로 접속하기 위한 svc
apiVersion: v1
kind: Service
metadata:
  name: mongodb-pri-svc
  namespace: mongodb
  labels:
    app: mongodb
    statefulset.kubernetes.io/pod-name: mongodb-0
spec:
  type: NodePort
  selector:
    app: mongodb
    statefulset.kubernetes.io/pod-name: mongodb-0
  ports:
  - port: 27017
    targetPort: 27017
    nodePort: 32017

```

각 Secondary 노드에도 동일한 방식으로 접근할 수 있도록 `NodePort` 서비스를 추가했습니다.



## **📌 확인용**

```yaml
# 필수 설정
kubectl run dns-verify -it --rm --restart=Never --image=busybox -- cat /etc/resolv.conf
sudo vi /etc/resolv.conf
nameserver 172.96.0.10

kubectl -n mongodb get pv,pvc,statefulsets,po,svc -o wide | grep mongo

kubectl -n mongodb exec -it mongodb-0 /bin/bash
mongo

# 아래는 test
# clusterIP접속
curl mongodb-svc.mongodb.svc.cluster.local:27017
curl mongodb-0.mongodb-svc.default.svc.cluster.local:27017
curl mongodb-1.mongodb-svc.default.svc.cluster.local:27017
curl mongodb-2.mongodb-svc.default.svc.cluster.local:27017

# Nodeport접속
curl mongodb-pri-svc.default.svc.cluster.local:27017
curl mongodb-sec1-svc.default.svc.cluster.local:27017
curl mongodb-sec2-svc.default.svc.cluster.local:27017

```

![Image](https://github.com/user-attachments/assets/9173a5d3-8f41-4a88-bedb-8855ab5f18ac)




## **🛠 MongoDB ReplicaSet 설정**

MongoDB의 **Primary, Secondary 노드를 구성하기 위한 ReplicaSet 초기 설정 명령어**입니다.

```yaml
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongodb-0.mongodb-svc.mongodb.svc.cluster.local:27017", priority: 100},
    { _id: 1, host: "mongodb-1.mongodb-svc.mongodb.svc.cluster.local:27017", priority: 50},
    { _id: 2, host: "mongodb-2.mongodb-svc.mongodb.svc.cluster.local:27017", priority: 50}
  ]
})

```

➡ **우선순위(`priority`)를 설정하여 Primary/Secondary 역할을 자동으로 조정하도록 구성**

## **📌 ReplicaSet 상태 확인 (`rs.status()`)**

MongoDB ReplicaSet이 정상적으로 동작하는지 확인하기 위해 **`rs.status()` 명령어를 실행**하면, 다음과 같은 결과가 출력됩니다.

```yaml
rs0:PRIMARY> rs.status()
{
        "set" : "rs0",
        "date" : ISODate("2024-05-01T15:49:59.899Z"),
        "myState" : 1,
        "term" : NumberLong(2),
        "syncingTo" : "",
        "syncSourceHost" : "",
        "syncSourceId" : -1,
        "heartbeatIntervalMillis" : NumberLong(2000),
        "majorityVoteCount" : 2,
        "writeMajorityCount" : 2,
        "optimes" : {
                "lastCommittedOpTime" : {
                        "ts" : Timestamp(1714578594, 1),
                        "t" : NumberLong(2)
                },
                "lastCommittedWallTime" : ISODate("2024-05-01T15:49:54.307Z"),
                "readConcernMajorityOpTime" : {
                        "ts" : Timestamp(1714578594, 1),
                        "t" : NumberLong(2)
                },
                "readConcernMajorityWallTime" : ISODate("2024-05-01T15:49:54.307Z"),
                "appliedOpTime" : {
                        "ts" : Timestamp(1714578594, 1),
                        "t" : NumberLong(2)
                },
                "durableOpTime" : {
                        "ts" : Timestamp(1714578594, 1),
                        "t" : NumberLong(2)
                },
                "lastAppliedWallTime" : ISODate("2024-05-01T15:49:54.307Z"),
                "lastDurableWallTime" : ISODate("2024-05-01T15:49:54.307Z")
        },
        "members" : [
                {
                        "_id" : 0,
                        "name" : "mongodb-0.mongodb-svc.default.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 1,
                        "stateStr" : "PRIMARY",
                        "uptime" : 294,
                        "optime" : {
                                "ts" : Timestamp(1714578594, 1),
                                "t" : NumberLong(2)
                        },
                        "optimeDate" : ISODate("2024-05-01T15:49:54Z"),
                        "syncingTo" : "",
                        "syncSourceHost" : "",
                        "syncSourceId" : -1,
                        "infoMessage" : "",
                        "electionTime" : Timestamp(1714578593, 1),
                        "electionDate" : ISODate("2024-05-01T15:49:53Z"),
                        "configVersion" : 1,
                        "self" : true,
                        "lastHeartbeatMessage" : ""
                },
                {
                        "_id" : 1,
                        "name" : "mongodb-1.mongodb-svc.default.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 27,
                        "optime" : {
                                "ts" : Timestamp(1714578594, 1),
                                "t" : NumberLong(2)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1714578594, 1),
                                "t" : NumberLong(2)
                        },
                        "optimeDate" : ISODate("2024-05-01T15:49:54Z"),
                        "optimeDurableDate" : ISODate("2024-05-01T15:49:54Z"),
                        "lastHeartbeat" : ISODate("2024-05-01T15:49:57.916Z"),
                        "lastHeartbeatRecv" : ISODate("2024-05-01T15:49:59.646Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncingTo" : "mongodb-0.mongodb-svc.default.svc.cluster.local:27017",
                        "syncSourceHost" : "mongodb-0.mongodb-svc.default.svc.cluster.local:27017",
                        "syncSourceId" : 0,
                        "infoMessage" : "",
                        "configVersion" : 1
                },
                {
                        "_id" : 2,
                        "name" : "mongodb-2.mongodb-svc.default.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 27,
                        "optime" : {
                                "ts" : Timestamp(1714578594, 1),
                                "t" : NumberLong(2)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1714578594, 1),
                                "t" : NumberLong(2)
                        },
                        "optimeDate" : ISODate("2024-05-01T15:49:54Z"),
                        "optimeDurableDate" : ISODate("2024-05-01T15:49:54Z"),
                        "lastHeartbeat" : ISODate("2024-05-01T15:49:57.919Z"),
                        "lastHeartbeatRecv" : ISODate("2024-05-01T15:49:57.917Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncingTo" : "mongodb-0.mongodb-svc.default.svc.cluster.local:27017",
                        "syncSourceHost" : "mongodb-0.mongodb-svc.default.svc.cluster.local:27017",
                        "syncSourceId" : 0,
                        "infoMessage" : "",
                        "configVersion" : 1
                }
        ],
        "ok" : 1
}

```

