### Step-by-Step Solution to the Problem

#### **Summary of Tasks**

1. **Application Setup**:
   - **PHP Version**: 5.6
   - **Admin Access**: 
     - Username: `admin`
     - Password: `admin`
   - **Database Snapshot**: Located in the database folder (`tms.sql`)

2. **Docker Setup**:
   - Containerize the application and database separately.
   - Use Adminer for database management.

3. **Kubernetes Setup**:
   - Create a Deployment for the application.
   - Establish a StatefulSet for the database.
   - Configure an Ingress resource for external exposure.
   - Use ConfigMap and Secret resources for configuration management.
   - Create PersistentVolumes (PV) and PersistentVolumeClaims (PVC) for storage.
   - Create a Helm chart for deployment.
   - Setup an NFS server and create a storage class using Helm for PV.

#### **Implementation Steps**

1. **Download the Project Code**: 
   - Download the project code

2. **Dockerize the Application**:
   - **Create Dockerfiles for both the PHP application and the database**:
     - **Dockerfile for PHP Application**:
       ```dockerfile
       # Use an official PHP runtime as a parent image
       FROM php:5.6-apache
       # Set the working directory
       WORKDIR /var/www/html
       # Copy the current directory contents into the container at /var/www/html
       COPY . /var/www/html
       # Install necessary PHP extensions
       RUN docker-php-ext-install mysqli
       ```
     - **Dockerfile for Database**:
       ```dockerfile
       # Use the official MySQL image
       FROM mysql:5.7
       # Set environment variables for database configuration
       ENV MYSQL_DATABASE=tms
       ENV MYSQL_ROOT_PASSWORD=root
       # Copy the database initialization script
       COPY ./database/tms.sql /docker-entrypoint-initdb.d/
       ```

3. **Docker Compose File**:
   - Create a `docker-compose.yml` file to manage both containers:
     ```yaml
     version: '3'
     services:
       php:
         build: .
         ports:
           - "80:80"
       db:
         image: mysql:5.7
         environment:
           MYSQL_DATABASE: tms
           MYSQL_ROOT_PASSWORD: root
         ports:
           - "3306:3306"
         volumes:
           - db-data:/var/lib/mysql
     volumes:
       db-data:
     ```

4. **Kubernetes Deployment**:
   - **Deployment for PHP Application**:
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: php-deployment
     spec:
       replicas: 3
       selector:
         matchLabels:
           app: php
       template:
         metadata:
           labels:
             app: php
         spec:
           containers:
           - name: php
             image: php:5.6-apache
             ports:
             - containerPort: 80
             volumeMounts:
             - name: php-volume
               mountPath: /var/www/html
           volumes:
           - name: php-volume
             persistentVolumeClaim:
               claimName: php-pvc
     ```

   - **StatefulSet for Database**:
     ```yaml
     apiVersion: apps/v1
     kind: StatefulSet
     metadata:
       name: mysql
     spec:
       serviceName: "mysql"
       replicas: 1
       selector:
         matchLabels:
           app: mysql
       template:
         metadata:
           labels:
             app: mysql
         spec:
           containers:
           - name: mysql
             image: mysql:5.7
             env:
             - name: MYSQL_ROOT_PASSWORD
               value: root
             - name: MYSQL_DATABASE
               value: tms
             ports:
             - containerPort: 3306
             volumeMounts:
             - name: mysql-persistent-storage
               mountPath: /var/lib/mysql
       volumeClaimTemplates:
       - metadata:
           name: mysql-persistent-storage
         spec:
           accessModes: [ "ReadWriteOnce" ]
           resources:
             requests:
               storage: 1Gi
     ```

   - **Ingress Resource**:
     ```yaml
     apiVersion: networking.k8s.io/v1
     kind: Ingress
     metadata:
       name: php-ingress
     spec:
       rules:
       - host: php.local
         http:
           paths:
           - path: /
             pathType: Prefix
             backend:
               service:
                 name: php-service
                 port:
                   number: 80
     ```

   - **PersistentVolume and PersistentVolumeClaim**:
     ```yaml
     apiVersion: v1
     kind: PersistentVolume
     metadata:
       name: php-pv
     spec:
       capacity:
         storage: 1Gi
       accessModes:
         - ReadWriteOnce
       hostPath:
         path: "/mnt/data"
     ---
     apiVersion: v1
     kind: PersistentVolumeClaim
     metadata:
       name: php-pvc
     spec:
       accessModes:
         - ReadWriteOnce
       resources:
         requests:
           storage: 1Gi
     ```

5. **Helm Chart**:
   - Create a Helm chart for the application. The Helm chart will include templates for deployments, statefulsets, services, ingress, and persistent volumes.

   - **Example Helm Chart Structure**:
     ```
     my-helm-chart/
     ├── Chart.yaml
     ├── values.yaml
     └── templates/
         ├── deployment.yaml
         ├── statefulset.yaml
         ├── service.yaml
         ├── ingress.yaml
         └── pv-pvc.yaml
     ```

6. **NFS Server Setup**:
   - Set up an NFS server to provide storage for the application.

   - **NFS Storage Class**:
     ```yaml
     apiVersion: storage.k8s.io/v1
     kind: StorageClass
     metadata:
       name: nfs
     provisioner: nfs-provisioner
     ```

