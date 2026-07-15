# MS SQL Server

Installing a Deployment type resource with default PVC storage.

## Set env vars

```bash
# target namespace
_TNS="my-mssqlserver"

# ServiceAccount
_MSSQL_SA=mssql-anyuid

# Storage
_MSSQL_PVC_NAME="mssql-data"
_MSSQL_PVC_SIZE="8Gi"

# Deplyment & secret
_MSSQL_SECRET_NAME=mssql
_MSSQL_PWD="myPassw0rdDem0s"
_MSSQL_DEPLOYMENT_NAME=mssql
_MSSQL_DEPLOYMENT_LABEL="${_MSSQL_DEPLOYMENT_NAME}"
_MSSQL_IMAGE="mcr.microsoft.com/mssql/server:2025-latest"

# Services
_MSSQL_SERVICE_NODEPORT_NAME="mssql-service-node-port"
_MSSQL_SERVICE_NAME="mssql-service"
_MSSQL_NODE_PORT=31433
```

## Create resources

Create namespace and service account
```bash
oc new-project ${_TNS}

cat << EOF | oc create -n ${_TNS} -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ${_MSSQL_SA}
EOF

oc adm policy add-scc-to-user anyuid -z ${_MSSQL_SA} -n ${_TNS}
```

Create secret
```bash
oc delete secret -n ${_TNS} ${_MSSQL_SECRET_NAME} 2>/dev/null 
oc create secret generic -n ${_TNS} ${_MSSQL_SECRET_NAME} --from-literal=SA_PASSWORD="${_MSSQL_PWD}"
```

Create PVC on default storage class
```bash
oc delete pvc -n ${_TNS} ${_MSSQL_PVC_NAME} 2>/dev/null 
cat <<EOF | oc apply -f -
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ${_MSSQL_PVC_NAME}
  namespace: ${_TNS}
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: ${_MSSQL_PVC_SIZE}
EOF
```

Create deployment
```bash
oc delete deployment -n ${_TNS} ${_MSSQL_DEPLOYMENT_NAME} 2>/dev/null 
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${_MSSQL_DEPLOYMENT_NAME}
  namespace: ${_TNS}
spec:
  selector:
    matchLabels:
      app: ${_MSSQL_DEPLOYMENT_LABEL}
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: ${_MSSQL_DEPLOYMENT_LABEL}
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: mssql
        image: ${_MSSQL_IMAGE}
        ports:
        - containerPort: 1433
        env:
        - name: MSSQL_PID
          value: "Developer"
        - name: ACCEPT_EULA
          value: "Y"
        - name: SA_PASSWORD
          valueFrom:
            secretKeyRef:
              name: ${_MSSQL_SECRET_NAME}
              key: SA_PASSWORD
        volumeMounts:
        - name: mssqldb
          mountPath: /var/opt/mssql
      serviceAccount: ${_MSSQL_SA}
      securityContext:
        runAsUser: 0
        runAsGroup: 0   
        fsGroup: 0
      volumes:
      - name: mssqldb
        persistentVolumeClaim:
          claimName: ${_MSSQL_PVC_NAME}
EOF
```

Create services
```bash
oc delete service -n ${_TNS} ${_MSSQL_SERVICE_NODEPORT_NAME} 2>/dev/null 
oc delete service -n ${_TNS} ${_MSSQL_SERVICE_NAME} 2>/dev/null 
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: ${_MSSQL_SERVICE_NODEPORT_NAME}
  namespace: ${_TNS}
spec:
  selector:
    app: ${_MSSQL_DEPLOYMENT_LABEL}
  ports:
    - protocol: TCP
      port: ${_MSSQL_NODE_PORT}
      targetPort: 1433
  type: NodePort
---
apiVersion: v1
kind: Service
metadata:
  name: ${_MSSQL_SERVICE_NAME}
  namespace: ${_TNS}
spec:
  selector:
    app: ${_MSSQL_DEPLOYMENT_LABEL}
  ports:
    - protocol: TCP
      port: 1433
      targetPort: 1433
EOF
```


## Get endpoints

NodePort service
```bash
_SRV_CLUSTER_IP=$(oc get service -n ${_TNS} ${_MSSQL_SERVICE_NODEPORT_NAME} -o jsonpath='{.spec.clusterIP}')
_SRV_NODEPORT=$(oc get service -n ${_TNS} ${_MSSQL_SERVICE_NODEPORT_NAME} -o jsonpath='{.spec.ports[*].nodePort}')
_SRV_PORT=$(oc get service -n ${_TNS} ${_MSSQL_SERVICE_NODEPORT_NAME} -o jsonpath='{.spec.ports[*].port}')

echo "MSSqlServer via NodePort: ClusterIP [${_SRV_CLUSTER_IP}] NodePort [${_SRV_NODEPORT}] Port [${_SRV_PORT}]"
```

Service
```bash
_SRV_CLUSTER_IP=$(oc get service -n ${_TNS} ${_MSSQL_SERVICE_NAME} -o jsonpath='{.spec.clusterIP}')
_SRV_PORT=$(oc get service -n ${_TNS} ${_MSSQL_SERVICE_NAME} -o jsonpath='{.spec.ports[*].port}')

echo "MSSqlServer via Service: ClusterIP [${_SRV_CLUSTER_IP}] Port [${_SRV_PORT}]"
```

## Test

### Connect to db server 

From remote linux box
```bash
# port forward
oc port-forward -n ${_TNS} service/${_MSSQL_SERVICE_NAME} 1433:${_SRV_PORT}
```

From another shell
```bash
# test connectivity
curl -kv localhost:1433

# connect to remote db using port forwarding
sqlcmd -C -S tcp:127.0.0.1,1433 -U sa -P myPassw0rdDem0s
```

From inside the db server pod
```bash
_POD_NAME=$(oc get pod -l app=${_MSSQL_DEPLOYMENT_LABEL} --no-headers | tail -n 1 | awk '{print $1}')
[[ ! -z "${_POD_NAME}" ]]; oc rsh -n ${_TNS} pod/${_POD_NAME} /bin/bash

# use sqlcmf from inside pod
/opt/mssql-tools18/bin/sqlcmd -C -S tcp:localhost,1433 -U sa -P myPassw0rdDem0s
```

### Create database/table

When connected in sqlcmd issue the following commands

# create db

```
create database dbtest
select name from sys.databases
go
```

```
use dbtest
create table tphone(id int, name nvarchar(50))
insert into tphone values(100,'Marco')
go
```

```
select * from tphone
go
```

```
alter database dbtest set single_user with rollback immediate
use master
drop database dbtest
go
```

```
select name from sys.databases
go

exit
```


## References

https://learn.microsoft.com/en-us/sql/linux/install-upgrade/quickstart-containers-azure?view=sql-server-linux-ver17&tabs=kubectl

https://learn.microsoft.com/en-us/sql/tools/sqlcmd/sqlcmd-utility?view=sql-server-ver17&tabs=go%2Cwindows-support&pivots=cs1-bash

https://learn.microsoft.com/en-us/sql/tools/sqlcmd/sqlcmd-use-utility?view=sql-server-ver17

https://learn.microsoft.com/en-us/sql/connect/jdbc/connecting-with-ssl-encryption?view=sql-server-ver17

https://learn.microsoft.com/en-us/sql/linux/install-upgrade/setup-tools?view=sql-server-ver17&tabs=redhat-install%2Codbc-ubuntu-1804#RHEL
