# AKS VoteApp Deployment Instructions

The following instructions are used to demonstrate how to provision an AKS cluster on Azure and deploy a cloud native application into it.

:metal: 

The cloud native application is architected using microservices and is presented to the user as a web application. The application frontend provides the end-user with the ability to vote on one of 6 programming languages: C#, Python, JavaScript, Go, Java, and/or NodeJS. Voting results in AJAX calls being made from the browser to an API which in turn then saves the results into a MongoDB database.

![VoteApp](./doc/voteapp.png)

Along the way, you'll get to see how to work with the following AKS cluster resources:
* Pods
* ReplicaSets
* Deployments
* Services
* StatefulSets
* PersistentVolumes
* PersistentVolumeClaims
* IngressController
* Ingress

![AKSDeployment](./doc/AKSDeployment.png)

# STEP 1:
Create a new AKS cluster

## STEP 1.1:

Define the name of the cluster and the azure resource group it will be allocated in

```
CLUSTER_NAME=akstest
RESOURCE_GROUP=aks
```

## STEP 1.2:

Create a new service principal. The AKS cluster will later be created with this.

```
SP=$(az ad sp create-for-rbac --name spdemocluster --skip-assignment)

APPID=$(echo $SP | jq -r .appId)
PASSWD=$(echo $SP | jq -r .password)

echo APPID: $APPID
echo PASSWD: $PASSWD
```

## STEP 1.3:

Create a new vnet and subnet for the AKS cluster

```
az network vnet create \
    --name cloudacademy-aks-vnet \
    --resource-group $RESOURCE_GROUP \
    --address-prefixes 10.0.0.0/8 \
    --subnet-name aks-subnet \
    --subnet-prefix 10.240.0.0/16
```

## STEP 1.4:

Assign the contributor role to the service principal scoped on the vnet previously created

```
VNETID=$(az network vnet show \
  --name cloudacademy-aks-vnet \
  --resource-group $RESOURCE_GROUP \
  --query id \
  -o tsv)

echo VNETID: $VNETID

az role assignment create \
  --assignee $APPID \
  --scope $VNETID \
  --role Contributor
```

## STEP 1.5:

Create the AKS cluster and place it in the vnet subnet previously created

```
SUBNETID=$(az network vnet subnet show \
  --name aks-subnet \
  --resource-group $RESOURCE_GROUP \
  --vnet-name cloudacademy-aks-vnet \
  --query id \
  -o tsv)

echo SUBNETID: $SUBNETID

az aks create \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --node-count 2 \
  --node-vm-size Standard_B2ms \
  --vm-set-type VirtualMachineScaleSets \
  --kubernetes-version 1.15.10 \
  --network-plugin azure \
  --service-cidr 10.0.0.0/16 \
  --dns-service-ip 10.0.0.10 \
  --docker-bridge-address 172.17.0.1/16 \
  --vnet-subnet-id $SUBNETID \
  --generate-ssh-keys \
  --network-policy azure \
  --service-principal $APPID \
  --client-secret $PASSWD 
```

This takes between **5-10 minutes** to complete so sit back and relax, its major chill time :+1:

Congrats!! 
You've just baked yourself a fresh AKS Kubernetes cluster!!

# STEP 2:

Test the kubectl client cluster authencation

```
az aks get-credentials -g aks --name akstest --admin
kubectl get nodes
```

```
kubectl config view
kubectl config get-contexts
kubectl config current-context
```

# STEP 3:

Install the Nginx Ingress Controller. This will allow us to direct inbound exteranl calls to the Frontend and API services that will be deployed into the AKS cluster.

## STEP 3.1:

Use Helm to install the Nginx Ingress Controller.

Notes:
1. The ```helm``` client needs to be installed locally
2. This has beem successfully tested with ```helm``` version v3.0.2
3. The ```helm``` client authenticates to the AKS cluster using the same ```~/.kube/config``` credentials established earlier

```
helm version
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo update
helm search repo stable
helm install aks-nginx-ingress stable/nginx-ingress
```

## STEP 3.2:

Query the Nginx Ingress Controller and determine the public ip address that has been assigned to it.

Notes:
1. The public IP address will be used to create both the API and Frontend service FQDNs used later on
2. The API FQDN will be used to within the API's Ingress resource for host based path routing
3. The Frontend FQDN will be used to within the Frontend's Ingress resource for host based path routing
4. The https://nip.io/ dynamic DNS service is being used to provide wildcard DNS

```
kubectl get svc aks-nginx-ingress-controller -o json

INGRESS_PUBLIC_IP=$(kubectl get svc aks-nginx-ingress-controller -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo INGRESS_PUBLIC_IP: $INGRESS_PUBLIC_IP

API_PUBLIC_FQDN=api.$INGRESS_PUBLIC_IP.nip.io
FRONTEND_PUBLIC_FQDN=frontend.$INGRESS_PUBLIC_IP.nip.io

echo API_PUBLIC_FQDN: $API_PUBLIC_FQDN
echo FRONTEND_PUBLIC_FQDN: $FRONTEND_PUBLIC_FQDN
```

# STEP 4:

Create the ```cloudacademy``` project (namespace)

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: cloudacademy
EOF
```

Configure the ```cloudacademy``` namespace to be the default

```
kubectl config set-context --current --namespace cloudacademy
```

# STEP 5:

Display the available AKS storage classes. We use the default storage class in the following MongoDb deployment.

```
kubectl get storageclass
```

# STEP 6:

Create a new Mongo StatefulSet name ```mongo```

```
cat << EOF | kubectl apply -f -
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mongo
  namespace: cloudacademy
spec:
  serviceName: mongo
  replicas: 3
  template:
    metadata:
      labels:
        role: db
        env: demo
        replicaset: rs0.main
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: replicaset
                  operator: In
                  values:
                  - rs0.main
              topologyKey: kubernetes.io/hostname
      terminationGracePeriodSeconds: 10
      containers:
        - name: mongo
          image: mongo
          command:
            - "numactl"
            - "--interleave=all"
            - "mongod"
            - "--wiredTigerCacheSizeGB"
            - "0.1"
            - "--bind_ip"
            - "0.0.0.0"
            - "--replSet"
            - "rs0"
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongodb-persistent-storage-claim
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: mongodb-persistent-storage-claim
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: default
        resources:
          requests:
            storage: 0.5Gi
EOF
```

Examine the Mongo Pods launch ordered sequence

```
kubectl get pods --watch
kubectl get pods
kubectl get pods --show-labels
kubectl get pods -l role=db
```

Note: **Ctrl-C** to exit the watch

Display the MongoDB Pods, Persistent Volumes and Persistent Volume Claims

```
kubectl get pod,pv,pvc
```

# STEP 7:

Create a new Headless Service for Mongo named ```mongo```

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: mongo
  namespace: cloudacademy
  labels:
    role: db
    env: demo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    role: db
EOF
```

Examine the Mongo Headless Service

```
kubectl get svc
```

Examine the DNS records for the Mongo Headless Service

```
kubectl run --generator=run-pod/v1 --rm utils -it --image eddiehale/utils bash
```

Within the new utils container run the following DNS queries

```
host mongo
for i in {0..2}; do host mongo-$i.mongo; done
exit
```

# STEP 8:

Initialise the Mongo database replica set

```
cat > db.init.js << EOF
rs.initiate();
rs.add("mongo-1.mongo:27017");
rs.add("mongo-2.mongo:27017");
cfg = rs.conf();
cfg.members[0].host = "mongo-0.mongo:27017";
rs.reconfig(cfg, {force: true});
sleep(5000);
EOF
```

```
kubectl exec -it mongo-0 mongo < db.init.js
kubectl exec -it mongo-0 -- mongo --eval "rs.status()"
```

# STEP 9:

Load the initial voting app data into the Mongo database

```
cat > db.load.js << EOF
use langdb;
db.languages.insert({"name" : "csharp", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 5, "compiled" : false, "homepage" : "https://dotnet.microsoft.com/learn/csharp", "download" : "https://dotnet.microsoft.com/download/", "votes" : 0}});
db.languages.insert({"name" : "python", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 3, "script" : false, "homepage" : "https://www.python.org/", "download" : "https://www.python.org/downloads/", "votes" : 0}});
db.languages.insert({"name" : "javascript", "codedetail" : { "usecase" : "web, client-side", "rank" : 7, "script" : false, "homepage" : "https://en.wikipedia.org/wiki/JavaScript", "download" : "n/a", "votes" : 0}});
db.languages.insert({"name" : "go", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 12, "compiled" : true, "homepage" : "https://golang.org", "download" : "https://golang.org/dl/", "votes" : 0}});
db.languages.insert({"name" : "java", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 1, "compiled" : true, "homepage" : "https://www.java.com/en/", "download" : "https://www.java.com/en/download/", "votes" : 0}});
db.languages.insert({"name" : "nodejs", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 20, "script" : false, "homepage" : "https://nodejs.org/en/", "download" : "https://nodejs.org/en/download/", "votes" : 0}});

db.languages.find().pretty();
EOF
```

```
kubectl exec -it mongo-0 mongo < db.load.js
kubectl exec -it mongo-0 -- mongo langdb --eval "db.languages.find().pretty()"
```

# STEP 10:

![AKSDeployment - API](./doc/AKSDeployment-API.png)

Deploy the API consisting of a Deployment, Service, and Ingress:

## STEP 10.1:

API: create **deployment** resource

```
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: cloudacademy
  labels:
    role: api
    env: demo
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 25%
  selector:
    matchLabels:
      role: api
  template:
    metadata:
      labels:
        role: api
    spec:
      containers:
      - name: api
        image: cloudacademydevops/api:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /ok
            port: 8080
          initialDelaySeconds: 2
          periodSeconds: 5
        readinessProbe:
          httpGet:
             path: /ok
             port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 1
EOF
```

## STEP 10.2:

API: create **service** resource

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: cloudacademy
  labels:
    role: api
    env: demo
spec:
  ports:
   - protocol: TCP
     port: 8080
  selector:
    role: api
EOF
```

## STEP 10.3:

API: create **ingress** resource

```
cat << EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: api
  namespace: cloudacademy
spec:
  rules:
    - host: $API_PUBLIC_FQDN
      http:
        paths:
          - backend:
              serviceName: api
              servicePort: 8080
            path: /
EOF
```

- Examine the rollout of the API deployment
- Examine the pods to confirm that they are up and running
- Examine the API pod log to see that it has successfully connected to the MongoDB replicaset
- Examine the API service details

```
kubectl rollout status deployment api
kubectl get pods
kubectl get pods -l role=api
kubectl logs API_POD_NAME_HERE
kubectl get svc
```

# STEP 11:

Test the API route url - test the ```/ok```, ```/languages```, and ```/languages/{name}``` endpoints

```
curl -s $API_PUBLIC_FQDN/ok
```

Note: The following commands leverage the [jq](https://stedolan.github.io/jq/) utility to format the json data responses

```
curl -s $API_PUBLIC_FQDN/languages | jq .
curl -s $API_PUBLIC_FQDN/languages/go | jq .
curl -s $API_PUBLIC_FQDN/languages/java | jq .
curl -s $API_PUBLIC_FQDN/languages/nodejs | jq .
```

# STEP 12:

![AKSDeployment](./doc/AKSDeployment-Frontend.png)

Create a new frontend Deployment

Notes: 
1. The value stored in the ```$API_PUBLIC_FQDN``` variable is injected into the frontend container's ```REACT_APP_APIHOSTPORT``` environment var - this tells the frontend where to send browser initiated API AJAX calls

## STEP 12.1:

Frontend: create **deployment** resource

```
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: cloudacademy
  labels:
    role: frontend
    env: demo
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 25%
  selector:
    matchLabels:
      role: frontend
  template:
    metadata:
      labels:
        role: frontend
    spec:
      containers:
      - name: frontend
        image: cloudacademydevops/frontend:v10
        imagePullPolicy: Always
        env:
          - name: REACT_APP_APIHOSTPORT
            value: $API_PUBLIC_FQDN
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /ok
            port: 8080
          initialDelaySeconds: 2
          periodSeconds: 5
        readinessProbe:
          httpGet:
             path: /ok
             port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 1
EOF
```

## STEP 12.2:

Frontend: create **service** resource

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: cloudacademy
  labels:
    role: frontend
    env: demo
spec:
  ports:
   - protocol: TCP
     port: 8080
  selector:
    role: frontend
EOF
```

## STEP 12.3:

Frontend: create **ingress** resource

```
cat << EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: frontend
  namespace: cloudacademy
spec:
  rules:
    - host: $FRONTEND_PUBLIC_FQDN
      http:
        paths:
          - backend:
              serviceName: frontend
              servicePort: 8080
            path: /
EOF
```

Examine the rollout of the Frontend Deployment

```
kubectl rollout status deployment frontend
kubectl get pods
kubectl get pods -l role=frontend
```

# STEP 13

Use the ```curl``` command to test the application via the frontend route url

```
curl -s -I $FRONTEND_PUBLIC_FQDN
curl -s -i $FRONTEND_PUBLIC_FQDN
```

Now test the full end-to-end application using the Chrome browser...

Note: Use the Developer Tools within the Chrome browser to record, filter, and observe the AJAX traffic (XHR) which is generated when any of the +1 vote buttons are clicked.

![VoteApp](./doc/voteapp.png)

# STEP 14

Query the MongoDb database directly to observe the updated vote data.

```
kubectl exec -it mongo-0 -- mongo langdb --eval "db.languages.find().pretty()"
```

# STEP 15

When you've finished with the AKS cluster and no longer need tear it down to avoid ongoing charges!!