# K8-deployment-application
This is a 3-tier application with mysql as databse, Php as web application and nginx as reverse-proxy.

We are deplpoying this multi-tier application using yaml file manifests in kubernetes as a separate containers.

### 1. Deploying the application using the manifest files

This is the deployment file for the Php appliaction
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php
  labels:
    tier: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php
      tier: backend
  template:
    metadata:
      labels:
        app: php
        tier: backend
    spec:
      volumes:
      - name: code
        persistentVolumeClaim:
          claimName: code
      containers:
      - name: php
        image: php:7-fpm
        volumeMounts:
        - name: code
          mountPath: /code
        env:
          - name: MYSQL_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mysql-secret
                key: password
      initContainers:
      - name: install
        image: busybox
        volumeMounts:
        - name: code
          mountPath: /code
        command:
        - wget
        - "-O"
        - "/code/index.php"
        - https://raw.githubusercontent.com/do-community/php-kubernetes/master/index.php
  ```
Lets create the deployment for Php application
```
kubectl create -f php_deployment.yaml
```

This is the service file for Php which enables the accesssability to the application
```
apiVersion: v1
kind: Service
metadata:
  name: php
  labels:
    tier: backend
spec:
  selector:
    app: php
    tier: backend
  ports:
  - protocol: TCP
    port: 9000
```

Lets apply this service
```
kubectl create -f php_service.yaml
```
Now lets go to Nginx section

This is the nginx deployment file
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    tier: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
      tier: backend
  template:
    metadata:
      labels:
        app: nginx
        tier: backend
    spec:
      volumes:
      - name: code
        persistentVolumeClaim:
          claimName: code
      - name: config
        configMap:
          name: nginx-config
          items:
          - key: config
            path: site.conf
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
        volumeMounts:
        - name: code
          mountPath: /code
        - name: config
          mountPath: /etc/nginx/conf.d
```          
Lets deploy the nginx
```
kubectl create -f nginx_deployment.yaml
```
Srvice file for nginx
```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    tier: backend
spec:
  selector:
    app: nginx
    tier: backend
  ports:
  - protocol: TCP
    port: 80
```    
applying the above service for nginx
```
kubectl create -f nginx_service.yaml
```
Lets create a ConfigMap for nginx and it keeps all your required configurations in a key-value format

This is the nginx configmap file
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  labels:
    tier: backend
data:
  config : |
    server {
      index index.php index.html;
      error_log  /var/log/nginx/error.log;
      access_log /var/log/nginx/access.log;
      root /code;
      location / {
          try_files $uri $uri/ /index.php?$query_string;
      }
      location ~ \.php$ {
          try_files $uri =404;
          fastcgi_split_path_info ^(.+\.php)(/.+)$;
          fastcgi_pass php:9000;
          fastcgi_index index.php;
          include fastcgi_params;
          fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
          fastcgi_param PATH_INFO $fastcgi_path_info;
        }
    }
```
Lets deploy this configmap
```
kubectl create -f nginx_configMap.yaml
```
Now lets deploy the databse for the this setup, where we have defined deployment, service and scecret in a single file
```
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
stringData:
  password: mysql
type: Opaque  
---
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  labels:
    app: mysql
    tier: backend
spec:
  containers:
  - image: mysql:5.6
    name: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
           name: mysql-secret
           key: password
    ports:
    - containerPort: 3306
      name: mysql
    volumeMounts:
    - name: mysql-persistent-storage
      mountPath: /var/lib/mysql
  volumes:
  - name: mysql-persistent-storage
    persistentVolumeClaim:
      claimName: code
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
    tier: backend
```
deploy the databse application
```
kubectl create -f mysql-deployment.yaml
```

### 2. Creating Local Storage and Persistent Volume

create a local StorageClass which can be further used for creating Persistent Volume

This is the StorageClass file with storage name "my-local-storage"
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: my-local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: Immediate
```
Create a local storage
```
kubectl create -f storageclass.yaml
```
Now lets create Persistent Volume by using this storage class we just created

This is the Persistent Volume file
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-local-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: my-local-storage
  local:
    path: /mnt/disk/vol
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - dev928
```

Create Persistent Volume
```
kubectl create -f pv.yaml
```
Now go ahead and create a Persistent Volume Claim to hold your application code and configuration files

Persistent Volume Claim file
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: code
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: my-local-storage
```
Lets create craete the Persistent Volume Claim
```
kubectl create -f pvc.yaml
```

### 3. Verifying and accessing the application
After deploying the manifest files, let make sure whether the application pods are running
```
kubectl get pod
```
![Screen Shot 2022-05-10 at 3 27 56 PM](https://user-images.githubusercontent.com/35251635/167602909-4d93d8df-db02-4db2-8caa-28dcc05dd091.png)

Lets check the creation of storage class, pv and pvc

![Screen Shot 2022-05-10 at 3 31 35 PM](https://user-images.githubusercontent.com/35251635/167603710-34ed9aac-3e93-41c8-ac97-d04a7384ebd1.png)

You can see above that the pvc is bound to pv

Now lets check for the services

![Screen Shot 2022-05-10 at 3 35 59 PM](https://user-images.githubusercontent.com/35251635/167604591-2eb3a0cc-7663-4fcb-94d3-e78622c45cb8.png)


Lets access the Php application using localhost ip and the nginx nodeport

![Screen Shot 2022-05-10 at 3 38 20 PM](https://user-images.githubusercontent.com/35251635/167604993-f9dd6c88-4a43-4880-8e26-344e9b646b4c.png)
