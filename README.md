# MySQL Server and Cluster Deployment using helm

## Mysql Server Deployment
### Objective:
This repository provides a reference implementation for deploying a single-instance MySQL InnoDB database on a Kubernetes cluster. The deployment leverages native Kubernetes resources to ensure security, persistence, and internal accessibility:
- **Deployment** : Manages the lifecycle of the MySQL Pod.
- **PersistentVolumeClaim (PVC)** : Ensures data persistence across Pod restarts.
- **Secret** : Stores and securely injects the MySQL root password
- **Service** (ClusterIP): Exposes the MySQL database for access within the cluster

### Steps to Deploy the MySQL Server
1. Clone the repository
    ```commandline
    git clone https://github.com/eswarmaganti/mysql-deployment.git
    ```
2. Create a `.env` environment file in project root directory with below content
    ```commandline
    MYSQL_ROOT_USERNAME=root
    MYSQL_ROOT_PASSWORD=<mysql-password>
    ENVIRONMENT=dev # deployment environment
    ```
3. Generate the custom `values.yaml` file for deployment environment.
   ```commandline
   envsubst < mysql/values.yaml > mysql/values-${ENVIRONMENT}.yaml
   ```
4. Test the helm chart templates syntax using the dry-run
   ```commandline
   helm upgrade mysql-app mysql -f mysql/values-${ENVIRONMENT}.yaml --install --dry-run
   ```
5. Deploy the helm chart
   ```commandline
   helm upgrade mysql-app mysql -f mysql/values-${ENVIRONMENT}.yaml --install 
   ```
6. Verify the helm chart release, pod and services deployed
   ```commandline
   $ helm ls -n mysql
   NAME     	NAMESPACE	REVISION	UPDATED                             	STATUS  	CHART      	APP VERSION
   mysql-app	mysql    	1       	2025-09-29 05:39:35.249386 +0530 IST	deployed	mysql-0.1.0	1.16.0     
   
   $ kubectl get all -n mysql
   NAME                             READY   STATUS    RESTARTS   AGE
   pod/mysql-app-5b8889d689-6lj8t   1/1     Running   0          23m
   
   NAME                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
   service/mysql-app   ClusterIP   None         <none>        3306/TCP   23m
   
   NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/mysql-app   1/1     1            1           23m
   
   NAME                                   DESIRED   CURRENT   READY   AGE
   replicaset.apps/mysql-app-5b8889d689   1         1         1       23m
   ```

### Accessing the MySQL instance
- Port forward the service to local and test the connection
   ```commandline
   
   $ kubectl port-forward svc/mysql-app 3307:3306 -n mysql
   Forwarding from 127.0.0.1:3307 -> 3306
   Forwarding from [::1]:3307 -> 3306
   Handling connection for 3307
   
   
   # In another terminal
   $ mysql -h 127.0.0.1 -P 3307 -u root -p 
   Enter password: 
   Welcome to the MySQL monitor.  Commands end with ; or \g.
   Your MySQL connection id is 18
   Server version: 9.4.0 MySQL Community Server - GPL
   
   Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.
   
   Oracle is a registered trademark of Oracle Corporation and/or its
   affiliates. Other names may be trademarks of their respective
   owners.
   
   Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
   
   mysql> show databses;
   ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'databses' at line 1
   mysql> show databases;
   +--------------------+
   | Database           |
   +--------------------+
   | information_schema |
   | mysql              |
   | performance_schema |
   | sys                |
   +--------------------+
   4 rows in set (0.05 sec)
   ```

- Test the direct connection to the pod using `kubectl exex` command
   ```commandline
   $ kubectl exec -it pod/mysql-app-5b8889d689-6lj8t -n mysql -- mysql -h 127.0.0.1 -P 3306 -u root -p 
   Enter password: 
   Welcome to the MySQL monitor.  Commands end with ; or \g.
   Your MySQL connection id is 137
   Server version: 9.4.0 MySQL Community Server - GPL
   
   Copyright (c) 2000, 2025, Oracle and/or its affiliates.
   
   Oracle is a registered trademark of Oracle Corporation and/or its
   affiliates. Other names may be trademarks of their respective
   owners.
   
   Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
   
   mysql> show databases;
   +--------------------+
   | Database           |
   +--------------------+
   | information_schema |
   | mysql              |
   | performance_schema |
   | sys                |
   +--------------------+
   4 rows in set (0.032 sec)
   
   mysql> CREATE DATABASE shop;
   Query OK, 1 row affected (0.023 sec)
   
   mysql> use shop;
   Database changed
   mysql> CREATE TABLE users( \
       -> name VARCHAR(256) NOT NULL,
       -> email VARCHAR(100) PRIMARY KEY,
       -> date_of_birth DATE,
       -> city VARCHAR(100) NOT NULL,
       -> pincode VARCHAR(6) NOT NULL);
   Query OK, 0 rows affected (0.040 sec)
   
   mysql> show tables;
   +----------------+
   | Tables_in_shop |
   +----------------+
   | users          |
   +----------------+
   1 row in set (0.009 sec)
   
   mysql> 
   ```

### Cleanup
- Delete the helm chart release by running the below command
   ```commandline
   $ helm delete mysql-app -n mysql
   
   $ kubectl get all -n mysql
   ```