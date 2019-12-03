# KubernetesMongodb
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

