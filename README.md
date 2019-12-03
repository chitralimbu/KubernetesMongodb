# KubernetesMongodb (K8s v1.15.0)
Scaling containerized MongoDb using K8s and Docker

### Follow tutorial: 
https://kubernetes.io/blog/2017/01/running-mongodb-on-kubernetes-with-statefulsets/

### All YAML definitions can be found here: 
https://github.com/cvallance/mongo-k8s-sidecar/tree/master/example/StatefulSet

### MongoDB ReplicaSets
https://docs.mongodb.com/manual/replication/

Depending on which K8s implementation you are using use the relavant YAML file to create a StorageClass. 
For my purposes at home I am using minikube so I used: 

This is where the data for the MongoDb nodes will be stored. 

### minikube_hostpath.yaml

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  namespace: kube-system
  name: fast
  annotations:
    storageclass.beta.kubernetes.io/is-default-class: "true"
  labels:
    addonmanager.kubernetes.io/mode: Reconcile

provisioner: k8s.io/minikube-hostpath

```

Create a headless service to give each MongoDB node a DNS addresses. 

```
apiVersion: v1
kind: Service
metadata:
  name: mongo
  labels:
    name: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    role: mongo
```

Create a StatefulSet which runs MongoDB and orchestrates everything together. Pods created under StatefulSet
will get an ordinal name, combined with a HeadlessService, this allows pods to have stable identification. 

The "sidecar" container will configure the MongoDB replica set automatically. 
Sidecar: Helper container which helps the main container do it's work. 

```
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: "mongo"
  replicas: 3
  template:
    metadata:
      labels:
        role: mongo
        environment: test
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: mongo
          image: mongo:3.4
          command:
            - mongod
            - "--replSet"
            - rs0
            - "--bind_ip"
            - 0.0.0.0
            - "--smallfiles"
            - "--noprealloc"
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongo-persistent-storage
              mountPath: /data/db
        - name: mongo-sidecar
          image: cvallance/mongo-k8s-sidecar
          env:
            - name: MONGO_SIDECAR_POD_LABELS
              value: "role=mongo,environment=test"
  volumeClaimTemplates:
  - metadata:
      name: mongo-persistent-storage
      annotations:
        volume.beta.kubernetes.io/storage-class: "fast"
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 2Gi
```

Create ClusterRoleBinding to give sidecar permissions to manage the MongoDB replica sets. 

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: default-view
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
  - kind: ServiceAccount
    name: default
    namespace: default
```
### Examining what was just created

```
kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
mongo-0   2/2     Running   0          8m4s
mongo-1   2/2     Running   0          7m40s
mongo-2   2/2     Running   0          7m36s
```
```
kubectl get svc
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP     52m
mongo        ClusterIP   None         <none>        27017/TCP   9m45s
```

```
kubectl get statefulset
NAME    READY   AGE
mongo   3/3     11m
```

```
kubectl exec -it mongo-0 mongo
rs0:PRIMARY> rs.status().members
[
        {
                "_id" : 0,
                "name" : "172.18.0.5:27017",
                "health" : 1,
                "state" : 1,
                "stateStr" : "PRIMARY",
                .......
        },
        {
                "_id" : 1,
                "name" : "172.18.0.6:27017",
                "health" : 1,
                "state" : 2,
                "stateStr" : "SECONDARY",
                .......
        },
        {
                "_id" : 2,
                "name" : "172.18.0.7:27017",
                "health" : 1,
                "state" : 2,
                "stateStr" : "SECONDARY",
                .......
        }
]
```
### Scaling up

```
kubectl scale --replicas=5 statefulset mongo

NAME      READY   STATUS    RESTARTS   AGE
mongo-0   2/2     Running   0          17m
mongo-1   2/2     Running   0          16m
mongo-2   2/2     Running   0          16m
mongo-3   2/2     Running   0          11s
mongo-4   2/2     Running   0          5s
```
```
kubectl exec -it mongo-0 mongo
[
        {
                "_id" : 0,
                "name" : "172.18.0.5:27017",
                "health" : 1,
                "state" : 1,
                "stateStr" : "PRIMARY",
				......
		},
        {
                "_id" : 1,
                "name" : "172.18.0.6:27017",
                "health" : 1,
                "state" : 2,
                "stateStr" : "SECONDARY",
				......
		},
        {
                "_id" : 2,
                "name" : "172.18.0.7:27017",
                "health" : 1,
                "state" : 2,
                "stateStr" : "SECONDARY",
				......
		},
        {
                "_id" : 3,
                "name" : "172.18.0.8:27017",
                "health" : 1,
                "state" : 2,
                "stateStr" : "SECONDARY",
				......
		},
        {
                "_id" : 4,
                "name" : "172.18.0.9:27017",
                "health" : 1,
                "state" : 2,
                "stateStr" : "SECONDARY",
				......
        }
]
```
As you can see we now have 5 MongoDB nodes all as members of the ReplicaSet. 
```
kubectl delete pods mongo-0
kubectl get pods
mongo-0   2/2     Running   0          18s
mongo-1   2/2     Running   0          4m44s
mongo-2   2/2     Running   0          4m39s
mongo-3   2/2     Running   0          4m11s
mongo-4   2/2     Running   0          4m6s
```
If we delete pod mongo-0 K8s will automatically spin up another one and sidecar will automatically add it to the MongoDB replicaset. 
As it was the primary node "mongo-0" we deleted another primary node is automatically elected by MongoDB.

```
kubectl exec -it mongo-0 mongo
*mongo-0 is now a secondary node as another node was elected as primary*
rs0:SECONDARY> rs.slaveOk()
rs0:SECONDARY> rs.status().members
[
        {
                "_id" : 0,
                "name" : "172.18.0.5:27017",
                "health" : 1,
                "state" : 2,
                "stateStr" : "SECONDARY",
                ......
        },
        {
                "_id" : 1,
                "name" : "172.18.0.6:27017",
                "health" : 1,
                "state" : 2,
                "stateStr" : "SECONDARY",
                ......
        },
        {
                "_id" : 2,
                "name" : "172.18.0.7:27017",
                "health" : 1,
                "state" : 1,
                "stateStr" : "PRIMARY",
                ......
        },
		.........................
		.........................
]
```
```
kubectl exec -it mongo-2 mongo
rs0:PRIMARY>
```
mongo-2 is now our primary node. 

### Connecting to MongoDB replicaSet. 
If you want to access each node externally then you will need to create a Load Balancer for each mongoDB pod. The load balancer will give you a public IP to connect to the specific mongodb node. 

```
apiVersion: v1
kind: Service
metadata:
  name: mongo-0
spec:
  type: LoadBalancer
  ports:
  - port: 27017
    protocol: TCP
    targetPort: 27017
  selector: 
    app: mongo-0
```

```
kubectl get svc 
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP           21m
mongo        ClusterIP      None            <none>        27017/TCP         14m
mongo-0      LoadBalancer   10.108.111.98   <pending>     27017:30108/TCP   3s
```
As I'm using minikube I dont have an external IP. However cloud proivers such as GCP and AWS will provide a public IP. 

If you want to connect to the MongoDB ReplicaSet using an application, for example in Spring Boot
In your application.yaml file. 
```
spring:
  profiles: testing
  data:
    mongodb:
      uri: mongodb://mongo-0.mongo,mongo-1.mongo,mongo-2.mongo:27017/dbname
```
