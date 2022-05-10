# K8-deployment-application
This is a 3-tier application with mysql as databse, Php as web application and nginx as reverse-proxy.

We are deplpoying this multi-tier application using yaml file manifests in kubernetes as a separate containers.

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

This the service file for Php which enables the accesssability to the application
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

