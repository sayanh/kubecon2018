### AGENDA
- Why kubernetes
- What is kubernetes
- Neo4j Causal Architecture
- Neo4j infrastructure definition
- Minikube Installation / Intro
- Kubectl
- Pod
- Service
+++
   ### AGENDA
- Deployment
- Stateful set
- Volumes
- Config sets
- Secrets
---
### Background
- Deprecate Graph Service
- Migrate Identity to talk to neo4j  <!-- .element: class="fragment" -->
- Bosh Deployment / Obsolete setup / boshctl issue  <!-- .element: class="fragment" -->
- Tedious: Release -> Zip -> upload  <!-- .element: class="fragment" -->
- Single AZ   <!-- .element: class="fragment" -->
- Manual IP / security groups setups   <!-- .element: class="fragment" -->
---
### Kubernetes
- Cloud agnostic system for deployment of applications/services/databases
- Developed by google, now open source  <!-- .element: class="fragment" -->
- Promotes Infrastructure as code  <!-- .element: class="fragment" -->
- Out of the the box Service Discovery & scaling of applications  <!-- .element: class="fragment" -->
- Containerized deployment (Docker/rkt)  <!-- .element: class="fragment" -->
+++
![Image](./assets/md/assets/Slide6.1.png)
+++
![Image](./assets/md/assets/Slide6.2.png)
+++
![Image](./assets/md/assets/Slide6.png)
---
#### Neo4j Causal Cluster
- Write and read load is balanced between dedicated neo4j instances.
- Client app must not differentiate between Write/Read operations.  <!-- .element: class="fragment" -->
- Native read/write consistency, via bookmarks (Client needs to adapt)  <!-- .element: class="fragment" -->
- No Ha proxy/LB is needed  <!-- .element: class="fragment" -->
+++
#### Neo4j Causal Cluster
- Allows to use bolt+routing protocol for native routing
- Allows to use bolt+routing+context protocol for context specific routing of requests  <!-- .element: class="fragment" -->
- Proportionate load on each neo4j instance, even in case of consistent reads.   <!-- .element: class="fragment" -->
- Hot backups  <!-- .element: class="fragment" -->
+++
#### Neo4j Causal Cluster
![Image](./assets/md/assets/Slide3.jpg)
---
### Core Servers Definition
+++

<span class='menu-title slide-title'>Core Servers Definition</span>
```yml
apiVersion: v1
kind: Service
metadata:
  name: neo4jcc
  namespace: neo4j-k8s
spec:
  clusterIP: None
  selector:
    app: neo4jcc
  ports:
  - name: rest
    port: 7474 #we need to open atleast 1 port to be able to make service discoverable
#---
apiVersion: v1
kind: Service
metadata:
  labels:
    run: neo4jcc
  name: neo4jcc-lb
  namespace: neo4j-k8s
spec:
  ports:
  - port: 7474
    protocol: TCP
    targetPort: 7474
    name: rest
  - port: 7687
    protocol: TCP
    targetPort: 7687
    name: bolt
  - name: backup
    port: 6362
    targetPort: 6362
  selector:
    app: neo4jcc
  sessionAffinity: None
  type: ClusterIP
#---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: neo4jcc
  namespace: neo4j-k8s
spec:
  serviceName: neo4jcc #imp for stateful sets
  replicas: 3
  updateStrategy:
    type: RollingUpdate
  volumeClaimTemplates:
  - metadata:
      name: neo4j-data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: aws-ebs-gp2
  template:
    metadata:
      labels:
         app: neo4jcc
    spec:
      volumes:
      - name: entrypoint-volume
        configMap:
          name: neo4jcc-entrypoint
          defaultMode: 0744
      imagePullSecrets:
      - name: artefactorysecret
      containers:
        - name: neo4jcc-container
          imagePullPolicy: Always
          image: github.io/neo4j-k8s/neo4j-3.3-enterprise:v1   
          resources:
            requests:
              memory: "1Gi"
              cpu: "0.25"
          ports:
          - containerPort: 5000
            name: disc
          - containerPort: 6000
            name: tran
          - containerPort: 7000
            name: raft
          - containerPort: 6362
            name: backup
          - containerPort: 7474
            name: rest
          - containerPort: 7687
            name: bolt
          env:
          - name: NEO4J_ACCEPT_LICENSE_AGREEMENT
            value: "yes"
          - name: NEO4J_dbms_mode
            value: CORE
          - name: NEO4J_causalClustering_initialDiscoveryMembers
            value: neo4jcc-0.neo4jcc.neo4j-k8s.svc.cluster.local:5000,neo4jcc-1.neo4jcc.neo4j-k8s.svc.cluster.local:5000,neo4jcc-2.neo4jcc.neo4j-k8s.svc.cluster.local:5000
          - name: NEO4J_causalClustering_expectedCoreClusterSize
            value: "3"
          - name: NEO4J_AUTH  
            valueFrom:
              secretKeyRef:
                name: neo4j-secret-auth
                key: simpleauth
          volumeMounts:
          - name: entrypoint-volume
            mountPath: /docker-entrypoint.sh
            subPath: docker-entrypoint.sh
          - name: neo4j-data
            mountPath: /data
```

<span class="code-presenting-annotation fragment current-only" data-code-focus="2">Define Service</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="7">Service Cluster IP set to none - creates a headless service</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="13">Resource Seperator</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="23-33">Ports that must be exposed by service</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="40">Statefulset definition</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="46, 74, 80-91"></span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="93-100">Ports of service that is running in container</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="110-111"></span>
Note:
A headless service is a service that is not assigned an IP.
But is used to discover all the pods it serves.
Is nice for stateful sets for DBs/queues etc so that they can discover all pods involved.
Cluster service is used to route traffic to actual pods.
---
### Read Replica Servers Definition
+++

<span class='menu-title slide-title'>Read Replica Servers Definition</span>
```yml
apiVersion: v1
kind: Service
metadata:
  name: neo4jrr
  namespace: neo4j-k8s
spec:
  clusterIP: None
  selector:
    app: neo4jrr
  ports:
  - name: rest
    port: 7474
#---
apiVersion: v1
kind: Service
metadata:
  labels:
    run: neo4jrr
  name: neo4jrr-lb
  namespace: neo4j-k8s
spec:
  ports:
  - port: 7474
    protocol: TCP
    targetPort: 7474
    name: rest
  - port: 7687
    protocol: TCP
    targetPort: 7687
    name: bolt
  selector:
    app: neo4jrr
  sessionAffinity: None
  type: ClusterIP
#---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: neo4jrr
  namespace: neo4j-k8s
spec:
  serviceName: neo4jrr
  replicas: 3
  updateStrategy:
    type: RollingUpdate
  volumeClaimTemplates:
  - metadata:
      name: neo4j-data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: aws-ebs-gp2
  template:
    metadata:
      labels:
         app: neo4jrr
    spec:
      volumes:
      - name: entrypoint-volume
        configMap:
          name: neo4jcc-entrypoint
          defaultMode: 0744
      imagePullSecrets:
      - name: artefactorysecret
      containers:
        - name: neo4jrr-container
          imagePullPolicy: Always
          image: github.io/neo4j-k8s/neo4j-3.3-enterprise:v1
          resources:
            requests:
              memory: "1Gi"
              cpu: "0.25"
          ports:
          - containerPort: 5000
            name: disc
          - containerPort: 6000
            name: tran
          - containerPort: 7000
            name: raft
          - containerPort: 6362
            name: backup
          - containerPort: 7474
            name: rest
          - containerPort: 7687
            name: bolt
          env:
          - name: NEO4J_ACCEPT_LICENSE_AGREEMENT
            value: "yes"
          - name: NEO4J_dbms_mode
            value: READ_REPLICA
          - name: NEO4J_causalClustering_initialDiscoveryMembers
            value: neo4jcc-0.neo4jcc.neo4j-k8s.svc.cluster.local:5000,neo4jcc-1.neo4jcc.neo4j-k8s.svc.cluster.local:5000,neo4jcc-2.neo4jcc.neo4j-k8s.svc.cluster.local:5000
          - name: NEO4J_AUTH  
            valueFrom:
              secretKeyRef:
                name: neo4j-secret-auth
                key: simpleauth
          volumeMounts:
          - name: entrypoint-volume
            mountPath: /docker-entrypoint.sh
            subPath: docker-entrypoint.sh    
          - name: neo4j-data
            mountPath: /data     
```

<span class="code-presenting-annotation fragment current-only" data-code-focus="71">Same docker image</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="90-96"></span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="105-106"></span>
---
### Neo4j Docker image and setup
- Standard Image
    -  neo4j:3.3-enterprise  <!-- .element: class="fragment" -->
- Hybris Image  <!-- .element: class="fragment" -->
    - github.io/neo4j-k8s/neo4j-3.3-enterprise:v1  <!-- .element: class="fragment" -->
+++

<span class='menu-title slide-title'>entrypoint.sh</span>
```sh
#!/bin/bash -eu
    #custom
    export NEO4J_ha_serverId=$(hostname | awk -F'-' '{print $2}')
    export CANONICAL_HOSTNAME=$(getent hosts $(hostname) | awk '{ print $2 }')
    export SERVICE_NAME=$(getent hosts $(hostname) | awk '{ print $2 }' | awk -F'.' '{ print $2  }')
    #custom

    # Env variable naming convention:
    # - prefix NEO4J_
    # - double underscore char '__' instead of single underscore '_' char in the setting name
    # - underscore char '_' instead of dot '.' char in the setting name
    # Example:
    # NEO4J_dbms_tx__log_rotation_retention__policy env variable to set
    #       dbms.tx_log.rotation.retention_policy setting

    # Backward compatibility - map old hardcoded env variables into new naming convention (if they aren't set already)
    # Set some to default values if unset
    : ${NEO4J_dbms_tx__log_rotation_retention__policy:=${NEO4J_dbms_txLog_rotation_retentionPolicy:-"100M size"}}
    : ${NEO4J_wrapper_java_additional:=${NEO4J_UDC_SOURCE:-"-Dneo4j.ext.udc.source=docker"}}
    : ${NEO4J_dbms_memory_heap_initial__size:=${NEO4J_dbms_memory_heap_maxSize:-"512M"}}
    : ${NEO4J_dbms_memory_heap_max__size:=${NEO4J_dbms_memory_heap_maxSize:-"512M"}}
    : ${NEO4J_dbms_unmanaged__extension__classes:=${NEO4J_dbms_unmanagedExtensionClasses:-}}
    : ${NEO4J_dbms_allow__format__migration:=${NEO4J_dbms_allowFormatMigration:-}}
    : ${NEO4J_dbms_connectors_default__advertised__address:=${NEO4J_dbms_connectors_defaultAdvertisedAddress:-"$CANONICAL_HOSTNAME"}}
    : ${NEO4J_ha_server__id:=${NEO4J_ha_serverId:-}}
    : ${NEO4J_ha_initial__hosts:=${NEO4J_ha_initialHosts:-}}
    : ${NEO4J_causal__clustering_expected__core__cluster__size:=${NEO4J_causalClustering_expectedCoreClusterSize:-}}
    : ${NEO4J_causal__clustering_initial__discovery__members:=${NEO4J_causalClustering_initialDiscoveryMembers:-}}
    : ${NEO4J_causal__clustering_discovery__listen__address:=${NEO4J_causalClustering_discoveryListenAddress:-"0.0.0.0:5000"}}
    : ${NEO4J_causal__clustering_discovery__advertised__address:=${NEO4J_causalClustering_discoveryAdvertisedAddress:-"$CANONICAL_HOSTNAME:5000"}}
    : ${NEO4J_causal__clustering_transaction__listen__address:=${NEO4J_causalClustering_transactionListenAddress:-"0.0.0.0:6000"}}
    : ${NEO4J_causal__clustering_transaction__advertised__address:=${NEO4J_causalClustering_transactionAdvertisedAddress:-"$CANONICAL_HOSTNAME:6000"}}
    : ${NEO4J_causal__clustering_raft__listen__address:=${NEO4J_causalClustering_raftListenAddress:-"0.0.0.0:7000"}}
    : ${NEO4J_causal__clustering_raft__advertised__address:=${NEO4J_causalClustering_raftAdvertisedAddress:-"$CANONICAL_HOSTNAME:7000"}}

    : ${NEO4J_dbms_connectors_default__listen__address:="0.0.0.0"}
    : ${NEO4J_dbms_connector_http_listen__address:="0.0.0.0:7474"}
    : ${NEO4J_dbms_connector_https_listen__address:="0.0.0.0:7473"}
    : ${NEO4J_dbms_connector_bolt_listen__address:="0.0.0.0:7687"}
    : ${NEO4J_ha_host_coordination:="$CANONICAL_HOSTNAME:5001"}
    : ${NEO4J_ha_host_data:="$CANONICAL_HOSTNAME:6001"}

    # unset old hardcoded unsupported env variables
    unset NEO4J_dbms_txLog_rotation_retentionPolicy NEO4J_UDC_SOURCE \
        NEO4J_dbms_memory_heap_maxSize NEO4J_dbms_memory_heap_maxSize \
        NEO4J_dbms_unmanagedExtensionClasses NEO4J_dbms_allowFormatMigration \
        NEO4J_dbms_connectors_defaultAdvertisedAddress NEO4J_ha_serverId \
        NEO4J_ha_initialHosts NEO4J_causalClustering_expectedCoreClusterSize \
        NEO4J_causalClustering_initialDiscoveryMembers \
        NEO4J_causalClustering_discoveryListenAddress \
        NEO4J_causalClustering_discoveryAdvertisedAddress \
        NEO4J_causalClustering_transactionListenAddress \
        NEO4J_causalClustering_transactionAdvertisedAddress \
        NEO4J_causalClustering_raftListenAddress \
        NEO4J_causalClustering_raftAdvertisedAddress

    # Custom settings for dockerized neo4j
    : ${NEO4J_dbms_tx__log_rotation_retention__policy:=100M size}
    : ${NEO4J_dbms_memory_pagecache_size:=512M}
    : ${NEO4J_wrapper_java_additional:=-Dneo4j.ext.udc.source=docker}
    : ${NEO4J_dbms_memory_heap_initial__size:=512M}
    : ${NEO4J_dbms_memory_heap_max__size:=512M}
    : ${NEO4J_dbms_connectors_default__listen__address:=0.0.0.0}
    : ${NEO4J_dbms_connector_http_listen__address:=0.0.0.0:7474}
    : ${NEO4J_dbms_connector_https_listen__address:=0.0.0.0:7473}
    : ${NEO4J_dbms_connector_bolt_listen__address:=0.0.0.0:7687}
    : ${NEO4J_ha_host_coordination:=$CANONICAL_HOSTNAME:5001}
    : ${NEO4J_ha_host_data:=$CANONICAL_HOSTNAME:6001}
    : ${NEO4J_causal__clustering_discovery__listen__address:=0.0.0.0:5000}
    : ${NEO4J_causal__clustering_discovery__advertised__address:=$CANONICAL_HOSTNAME:5000}
    : ${NEO4J_causal__clustering_transaction__listen__address:=0.0.0.0:6000}
    : ${NEO4J_causal__clustering_transaction__advertised__address:=$CANONICAL_HOSTNAME:6000}
    : ${NEO4J_causal__clustering_raft__listen__address:=0.0.0.0:7000}
    : ${NEO4J_causal__clustering_raft__advertised__address:=$CANONICAL_HOSTNAME:7000}

    if [ -d /conf ]; then
        find /conf -type f -exec cp {} conf \;
    fi

    if [ -d /ssl ]; then
        NEO4J_dbms_directories_certificates="/ssl"
    fi

    if [ -d /plugins ]; then
        NEO4J_dbms_directories_plugins="/plugins"
    fi

    if [ -d /logs ]; then
        NEO4J_dbms_directories_logs="/logs"
    fi

    if [ -d /import ]; then
        NEO4J_dbms_directories_import="/import"
    fi

    if [ -d /metrics ]; then
        NEO4J_dbms_directories_metrics="/metrics"
    fi

    if [ "${NEO4J_AUTH:-}" == "none" ]; then
        NEO4J_dbms_security_auth__enabled=false
    elif [[ "${NEO4J_AUTH:-}" == neo4j/* ]]; then
        password="${NEO4J_AUTH#neo4j/}"
        if [ "${password}" == "neo4j" ]; then
            echo "Invalid value for password. It cannot be 'neo4j', which is the default."
            exit 1
        fi
        # Will exit with error if users already exist (and print a message explaining that)
        bin/neo4j-admin set-initial-password "${password}" || true
    elif [ -n "${NEO4J_AUTH:-}" ]; then
        echo "Invalid value for NEO4J_AUTH: '${NEO4J_AUTH}'"
        exit 1
    fi

    # list env variables with prefix NEO4J_ and create settings from them
    unset NEO4J_AUTH NEO4J_SHA256 NEO4J_TARBALL
    for i in $( set | grep ^NEO4J_ | awk -F'=' '{print $1}' | sort -rn ); do
        setting=$(echo ${i} | sed 's|^NEO4J_||' | sed 's|_|.|g' | sed 's|\.\.|_|g')
        value=$(echo ${!i})
        if [[ -n ${value} ]]; then
            if grep -q -F "${setting}=" conf/neo4j.conf; then
                # Remove any lines containing the setting already
                sed --in-place "/${setting}=.*/d" conf/neo4j.conf
            fi
            # Then always append setting to file
            echo "${setting}=${value}" >> conf/neo4j.conf
        fi
    done

    [ -f "${EXTENSION_SCRIPT:-}" ] && . ${EXTENSION_SCRIPT}

    printenv

    exec bin/neo4j console
elif [ "$1" == "dump-config" ]; then
    if [ -d /conf ]; then
        cp --recursive conf/* /conf
    else
        echo "You must provide a /conf volume"
        exit 1
    fi
else
    exec "$@"
fi
```

<span class="code-presenting-annotation fragment current-only" data-code-focus="4, 5"></span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="36-41"></span>
+++
#### Neo4j Pods
- Pods are smallest unit of resource in Kubernetes
- Pod runs 1 or more containers in it
- They are scheduled and managed by services
+++
```sh
23:06 $ kubectl get pods | grep neo4j
neo4j-0                                    1/1       Running   4          6d
neo4j-1                                    1/1       Running   3          13d
neo4j-2                                    1/1       Running   0          13d
neo4j-ha-3247771895-8tkdj                  1/1       Running   2          6d
neo4jcc-0                                  0/1       Pending   0          6h
neo4jcc-1                                  1/1       Running   0          8d
neo4jcc-2                                  1/1       Running   3          8d
neo4jrr-0                                  0/1       Pending   0          6d
neo4jrr-1                                  1/1       Running   0          8d
neo4jrr-2                                  1/1       Running   2          8d
```

<span class="code-presenting-annotation fragment current-only" data-code-focus="1"></span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="2-5">HA Cluster</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="6-11">Causal Cluster</span>
---
### Minikube
- Minikube runs Kubernetes locally in a VM
- Runs a single node
- Can be used to test K8 stack manifests
- <i class="fa fa-hand-o-right" aria-hidden="true"> </i> Maximum number of pods are limited to resources of VM.
  - max 3 Neo4j instances.
+++
### Minikube installation
- brew install kubectl
- brew install cask
- brew cask install minikube
- or curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.24.1/minikube-darwin-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
+++
### Virtual box for Minikube installation
- Virtual box
  - wget -P ~/Downloads/ http://download.virtualbox.org/virtualbox/5.2.2/VirtualBox-5.2.2-119230-OSX.dmg
  - double click the dmg file and begin installing virtualbox
- minikube start --kubernetes-version v1.8.0
---
### Exercise - 1
#### Create a Pod
```
$ kubectl run hello-node --image=ajitchahal/hello-node:v3 --port=8080
$ kubectl get pods
$ kubectl port-forward hello-node 8080:8080
$ kubectl get deployments
$ kubectl expose deployment hello-node --type=NodePort
$ kubectl get services
   - note down hello-node   10.0.0.107   <nodes>       8080:32620/TCP   1m
$ minikube ip
run in browser http://192.168.99.100:32620/   
$ or  #minikube service hello-node  -n neo4j-k8s
```
<span class="code-presenting-annotation fragment current-only" data-code-focus="1">Create a pod from a node-js app, that returns current date-time</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="2">Check if pods are in running state</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="3">Lets access the service in pod</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="4">Creating pod creates a deployment</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="5">Lets expose the pod on a VM port</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="6-7">Note the node port</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="8-9">Get VM ip and run in a browser</span>
Note:
Do not do type load-balancer in AWS as it creates an ELB.
+++
### Exercise - 2
#### Create a pod & deployment & service via manifest
+++

<span class='menu-title slide-title'>Create a pod via manifest</span>
```yml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: kuad-web-pod
  name: dpl-kuad-pod
  namespace: default
spec:
  volumes:
  - hostPath:
      path: /tmp
    name: kuard-data
  containers:
  - image: gcr.io/kuar-demo/kuard-amd64:1
    imagePullPolicy: Always
    name: kuad-container-pod
    ports:
    - containerPort: 8080
      name: web-port
      protocol: TCP
    volumeMounts:
    - mountPath: /tmp-folder
      name: kuard-data


```

<span class="code-presenting-annotation fragment current-only" data-code-focus="2">kubectl apply -f pod.yml</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="9-12">bind a local directory</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="21-24">mount the host folder as volume in container</span>
Note:
Try deleting a pod and see it is not getting recreated again
+++

<span class='menu-title slide-title'>Create deployment via manifest</span>
```yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: dpl-kuad
  labels:
   run: kuad-example
spec:
  replicas: 1
  selector:
    matchLabels:
      run: kuad-web
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      name: kuad-pod
      labels:
        run: kuad-web
    spec:
      volumes:
      - name: kuard-data
        hostPath:
          path: /var
        #type: DirectoryOrCreate
      containers:
      - name: kuad-container
        imagePullPolicy: Always
        image: gcr.io/kuar-demo/kuard-amd64:1
        volumeMounts:
        - mountPath: /data
          name: kuard-data
        ports:
        - containerPort: 8080
          name: web-port
          protocol: TCP
```

<span class="code-presenting-annotation fragment current-only" data-code-focus="2">kubectl apply -f deployment.yml</span>
Note:
Try deleting a pod and see it is getting recreated again
+++

<span class='menu-title slide-title'>Expose pod via service</span>
```yml
kind: Service
apiVersion: v1
metadata:
  name: kuad-service
spec:
  selector:
    run: kuad-web
  type: NodePort
  ports:
  - protocol: TCP
    port: 8080 # port of service, it will be exposed to other pods inside the cluster
    targetPort: 8080 # port of container app runnning e.g. spring boot on 8080
    nodePort: 32711 # port opened on host VM so it is accessible outside cluster
```

<span class="code-presenting-annotation fragment current-only" data-code-focus="2">kubectl apply -f service.yml</span>
Note:
Access the pod in browser http://192.168.99.100:32620/
+++
## Exercise - 3
### SSH into the container & try some tricks
```
kubectl exec -it dpl-kuad-5db8f969cb-9rzjz sh
wget hello-node-c4ccdf565-9ss8j:8080 -P /tmp/
wget hello-node:8080 -P /tmp/
nslookup hello-node
nslookup hello-node-c4ccdf565-9ss8j
```
<span class="code-presenting-annotation fragment current-only" data-code-focus="1">Use - kubectl get pods | grep kuad to know pod name</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="2">Lets access our hello-node app from kuad app</span>
<span class="code-presenting-annotation fragment current-only" data-code-focus="3">Lets try again with service</span>
+++
#### Moral: Services Expose pods to other pods
#### Moral-2: Services Expose pods to applications external to Kubernetes
---
### Connect to kubernetes in AWS
```
apiVersion: v1
kind: Config
clusters:
- name: k8s-neo4j-k8s-stage
  cluster:
    server: https://my-kubernetes.io:443
    certificate-authority-data: xxxx
users:
- name: neo4j-k8s
  user:
    client-certificate-data: xxxx
    client-key-data: xxxxx
contexts:
- name: neo4j-k8s
  context:
    cluster: k8s-neo4j-k8s-stage
    user: neo4j-k8s
    namespace: neo4j-k8s
current-context: neo4j-k8s
```
#### Copy content & Save to k8_stage.config file
+++
- To connect to Stage
  - export KUBECONFIG=~/kubernetes/k8_stage.config
  - export KUBECONFIG=~/.kube/config
---
### Statefulsets
- Static & Predictable names of pods
- DNS discovery of individual pods with services  <!-- .element: class="fragment" -->
- Persistent Volumes, that are independent of POD lifecycle  <!-- .element: class="fragment" -->
- Same PV gets attached to same POD, regardless of which node it scheduled on.  <!-- .element: class="fragment" -->
- Persistent volume claims  <!-- .element: class="fragment" -->
  - claims leads to creation of volumes in correct Availability Zones  <!-- .element: class="fragment" -->
+++
## Exercise - 4
### Play with statefulset
+++

<span class='menu-title slide-title'>Create a service for Statefulset pods</span>
```yml
apiVersion: v1
kind: Service
metadata:
  name: neo4j-ha
spec:
  clusterIP: None
  selector:
    app: neo4j-ha
  ports:
  - name: rest
    port: 7474

```
<span class="code-presenting-annotation fragment current-only" data-code-focus="2">save to a file and run kubectl apply -f svc.yml</span>

+++

<span class='menu-title slide-title'>Create a Statefulset</span>
```yml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: neo4j
spec:
  serviceName: neo4j-ha
  replicas: 2
  template:
    metadata:
      labels:
         app: neo4j-ha
    spec:
      containers:
        - name: neo4j-container
          imagePullPolicy: IfNotPresent
          image: ajitchahal/neo4j-test:v2
          ports:
          - containerPort: 5001
            name: ha-host-cord
          - containerPort: 6001
            name: host-data
          - containerPort: 6362
            name: backup
          - containerPort: 7474
            name: rest
          - containerPort: 7687
            name: bolt
          env:
          - name: NEO4J_dbms_mode
            value: HA
          - name: NEO4J_ha_initialHosts
            value: neo4j-0.neo4j:5001,neo4j-1.neo4j:5001 #,neo4j-2.neo4j:5001
          - name: NEO4J_AUTH
            value: neo4j/admin
          - name: NEO4J_ha_host_coordination
            value: 0.0.0.0:5001
          - name: NEO4J_ha_host_data
            value: 0.0.0.0:6001
          command: ["/bin/sh"]
          args: [ "-c",
                  "export NEO4J_ha_serverId=$(hostname | awk -F'-' '{print $2}') \
                  && /neo4j-enterprise-3.2.6/bin/neo4j-admin set-initial-password 'ajit' && /neo4j-enterprise-3.2.6/bin/neo4j console"]
          #bin/neo4j start" #sleep 1000000"

```
<span class="code-presenting-annotation fragment current-only" data-code-focus="2">save to a file and run kubectl apply -f set.yml</span>

+++

<span class='menu-title slide-title'>Create a service so we can expose pod of Statefulset</span>
```yml
apiVersion: v1
kind: Service
metadata:
  name: neo4j-ha-lb
spec:
  type: NodePort
  selector:
    app: neo4j-ha
  ports:
  - name: rest
    port: 7474
    targetPort: 7474
    nodePort: 30074
  - name: bolt
    port: 7687
    targetPort: 7687
    nodePort: 30087
```
<span class="code-presenting-annotation fragment current-only" data-code-focus="2">save to a file and run kubectl apply -f svc-lb.yml</span>

+++
#### Lets access some info about pods & access the neo4j in browser
```
kubectl get statefulsets
kubectl get pods
kubectl get services
http://192.168.99.100:30074/browser/
create(n:Emp {name:'Ajit'})
match(n) return n
```
---
#### Ich bedanke mich für Ihre aufmerksamkeit
#### Les agradezco su atención y su apoyo.
#### आपके ध्यान के लिए धन्यवाद.
---
