Table of Contents
=================

   * [一、nginx使用nfs静态PV](#一nginx使用nfs静态pv)
      * [1、静态nfs-static-nginx-rc.yaml](#1静态nfs-static-nginx-rcyaml)
      * [2、静态nfs-static-nginx-deployment.yaml](#2静态nfs-static-nginx-deploymentyaml)
      * [3、nginx多目录挂载](#3nginx多目录挂载)
   * [二、nginx使用nfs动态PV](#二nginx使用nfs动态pv)
      * [1、动态nfs-dynamic-nginx.yaml](#1动态nfs-dynamic-nginxyaml)
      
# 一、nginx使用nfs静态PV

## 1、静态nfs-static-nginx-rc.yaml

```bash
##清理资源
kubectl delete -f nfs-static-nginx-rc.yaml -n test

cat >nfs-static-nginx-rc.yaml<<\EOF
##创建namespace
---
apiVersion: v1
kind: Namespace
metadata:
   name: test
   labels:
     name: test
##创建nfs-pv
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
  labels:
    pv: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs  # 注意这里使用nfs的storageClassName，如果没改k8s的默认storageClassName的话，这里可以省略
  nfs:
    path: /data/nfs/nginx/
    server: 10.198.1.155
##创建nfs-pvc
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nfs-pvc
  namespace: test
  labels:
    pvc: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: nfs
  selector:
    matchLabels:
      pv: nfs-pv
##部署应用nginx
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-test
  namespace: test
  labels:
    name: nginx-test
spec:
  replicas: 2
  selector:
    name: nginx-test
  template:
    metadata:
      labels:
       name: nginx-test
    spec:
      containers:
      - name: nginx-test
        image: docker.io/nginx
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: nginx-data
        ports:
        - containerPort: 80
      volumes:
      - name: nginx-data
        persistentVolumeClaim:
          claimName: nfs-pvc
##创建service
---
apiVersion: v1
kind: Service
metadata:
  namespace: test
  name: nginx-test
  labels:
    name: nginx-test
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    name: http
    nodePort: 30080
  selector:
    name: nginx-test
EOF

##创建资源
kubectl apply -f nfs-static-nginx-rc.yaml -n test

##查看pv资源
kubectl get pv -n test --show-labels

##查看pvc资源
kubectl get pvc -n test --show-labels

##查看pod
$ kubectl get pods -n test
NAME               READY   STATUS    RESTARTS   AGE
nginx-test-r4n2j   1/1     Running   0          54s
nginx-test-zstf5   1/1     Running   0          54s

#可以看到，nginx应用已经部署成功。
#nginx应用的数据目录是使用的nfs共享存储，我们在nfs共享的目录里加入index.html文件，然后再访问nginx-service暴露的端口
#切换到到nfs-server服务器上

echo "Test NFS Share discovery with nfs-static-nginx-rc" > /data/nfs/nginx/index.html

#在浏览器上访问kubernetes主节点的 http://master:30080，就能访问到这个页面内容了
```

## 2、静态nfs-static-nginx-deployment.yaml

```bash
##清理资源
kubectl delete -f nfs-static-nginx-deployment.yaml -n test

cat >nfs-static-nginx-deployment.yaml<<\EOF
##创建namespace
---
apiVersion: v1
kind: Namespace
metadata:
   name: test
   labels:
     name: test
##创建nfs-pv
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
  labels:
    pv: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs  # 注意这里使用nfs的storageClassName，如果没改k8s的默认storageClassName的话，这里可以省略
  nfs:
    path: /data/nfs/nginx/
    server: 10.198.1.155
##创建nfs-pvc
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nfs-pvc
  namespace: test
  labels:
    pvc: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: nfs
  selector:
    matchLabels:
      pv: nfs-pv
##部署应用nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: test
  labels:
    name: nginx-test
spec:
  replicas: 2
  selector:
    matchLabels:
      name: nginx-test
  template:
    metadata:
      labels:
       name: nginx-test
    spec:
      containers:
      - name: nginx-test
        image: docker.io/nginx
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: nginx-data
        ports:
        - containerPort: 80
      volumes:
      - name: nginx-data
        persistentVolumeClaim:
          claimName: nfs-pvc
##创建service
---
apiVersion: v1
kind: Service
metadata:
  namespace: test
  name: nginx-test
  labels:
    name: nginx-test
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    name: http
    nodePort: 30080
  selector:
    name: nginx-test
EOF

##创建资源
kubectl apply -f nfs-static-nginx-deployment.yaml -n test

##查看pv资源
kubectl get pv -n test --show-labels

##查看pvc资源
kubectl get pvc -n test --show-labels

##查看pod
$ kubectl get pods -n test
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-64d6f78cdf-8bw8t   1/1     Running   0          55s
nginx-deployment-64d6f78cdf-n5n4q   1/1     Running   0          55s

#可以看到，nginx应用已经部署成功。
#nginx应用的数据目录是使用的nfs共享存储，我们在nfs共享的目录里加入index.html文件，然后再访问nginx-service暴露的端口
#切换到到nfs-server服务器上

echo "Test NFS Share discovery with nfs-static-nginx-deployment" > /data/nfs/nginx/index.html

#在浏览器上访问kubernetes主节点的 http://master:30080，就能访问到这个页面内容了
```

## 3、nginx多目录挂载

```
1、PV和PVC是一一对应关系，当有PV被某个PVC所占用时，会显示banding，其它PVC不能再使用绑定过的PV。

2、PVC一旦绑定PV，就相当于是一个存储卷，此时PVC可以被多个Pod所使用。（PVC支不支持被多个Pod访问，取决于访问模型accessMode的定义）。

3、PVC若没有找到合适的PV时，则会处于pending状态。

4、PV是属于集群级别的，不能定义在名称空间中。

5、PVC时属于名称空间级别的。
```

```bash
##清理资源
kubectl delete -f nfs-static-nginx-dp-many.yaml -n test

cat >nfs-static-nginx-dp-many.yaml<<\EOF
##创建namespace
---
apiVersion: v1
kind: Namespace
metadata:
   name: test
   labels:
     name: test
##创建nginx-data-pv
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-data-pv
  labels:
    pv: nginx-data-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs  # 注意这里使用nfs的storageClassName，如果没改k8s的默认storageClassName的话，这里可以省略
  nfs:
    path: /data/nfs/nginx/
    server: 10.198.1.155
##创建nginx-etc-pv
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-etc-pv
  labels:
    pv: nginx-etc-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs  # 注意这里使用nfs的storageClassName，如果没改k8s的默认storageClassName的话，这里可以省略
  nfs:
    path: /data/nfs/nginx/
    server: 10.198.1.155
##创建pvc名字为nfs-nginx-data,存放数据
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nfs-nginx-data
  namespace: test
  labels:
    pvc: nfs-nginx-data
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: nfs
  selector:
    matchLabels:
      pv: nginx-data-pv
##创建pvc名字为nfs-nginx-etc,存放配置文件
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nfs-nginx-etc
  namespace: test
  labels:
    pvc: nfs-nginx-etc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: nfs
  selector:
    matchLabels:
      pv: nginx-etc-pv
##部署应用nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: test
  labels:
    name: nginx-test
spec:
  replicas: 2
  selector:
    matchLabels:
      name: nginx-test
  template:
    metadata:
      labels:
       name: nginx-test
    spec:
      containers:
      - name: nginx-test
        image: docker.io/nginx
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: nginx-data
        # - mountPath: /etc/nginx   #--这里需要注意，如果是这么挂载，那么需要事先现在/data/nfs/nginx/目录下把nginx的完整配置提前拷贝好
        #   name: nginx-etc
        ports:
        - containerPort: 80
      volumes:
      - name: nginx-data
        persistentVolumeClaim:
          claimName: nfs-nginx-data
      # - name: nginx-etc
      #   persistentVolumeClaim:
      #     claimName: nfs-nginx-etc
##创建service
---
apiVersion: v1
kind: Service
metadata:
  namespace: test
  name: nginx-test
  labels:
    name: nginx-test
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    name: http
    nodePort: 30080
  selector:
    name: nginx-test
EOF

##创建资源
kubectl apply -f nfs-static-nginx-dp-many.yaml -n test

##查看pv资源
kubectl get pv -n test --show-labels

##查看pvc资源
kubectl get pvc -n test --show-labels

##查看pod
$ kubectl get pods -n test
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-64d6f78cdf-8bw8t   1/1     Running   0          55s
nginx-deployment-64d6f78cdf-n5n4q   1/1     Running   0          55s

##进入容器
kubectl exec -it nginx-deployment-f687cdf47-xncj8 -n test /bin/bash

#可以看到，nginx应用已经部署成功。
#nginx应用的数据目录是使用的nfs共享存储，我们在nfs共享的目录里加入index.html文件，然后再访问nginx-service暴露的端口
#切换到到nfs-server服务器上

echo "Test NFS Share discovery with nfs-static-nginx-dp-many" > /data/nfs/nginx/index.html

#在浏览器上访问kubernetes主节点的 http://master:30080，就能访问到这个页面内容了
```

## 4、参数namespace

```bash
##清理资源
export NAMESPACE="mos-namespace"

kubectl delete -f nfs-static-nginx-dp-many.yaml -n ${NAMESPACE}

cat >nfs-static-nginx-dp-many.yaml<<-EOF
##创建namespace
---
apiVersion: v1
kind: Namespace
metadata:
   name: ${NAMESPACE}
   labels:
     name: ${NAMESPACE}
##创建nginx-data-pv
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-data-pv
  labels:
    pv: nginx-data-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs  # 注意这里使用nfs的storageClassName，如果没改k8s的默认storageClassName的话，这里可以省略
  nfs:
    path: /data/nfs/nginx/
    server: 10.198.1.155
##创建nginx-log-pv
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-log-pv
  labels:
    pv: nginx-log-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs  # 注意这里使用nfs的storageClassName，如果没改k8s的默认storageClassName的话，这里可以省略
  nfs:
    path: /data/nfs/nginx/
    server: 10.198.1.155
##创建pvc名字为nfs-nginx-data,存放数据
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nfs-nginx-data
  labels:
    pvc: nfs-nginx-data
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: nfs
  selector:
    matchLabels:
      pv: nginx-data-pv
##创建pvc名字为nfs-nginx-log,存放日志文件
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nfs-nginx-log
  labels:
    pvc: nfs-nginx-log
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: nfs
  selector:
    matchLabels:
      pv: nginx-log-pv
##部署应用nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    name: nginx-test
spec:
  replicas: 2
  selector:
    matchLabels:
      name: nginx-test
  template:
    metadata:
      labels:
       name: nginx-test
    spec:
      containers:
      - name: nginx-test
        image: docker.io/nginx
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: nginx-data
        - mountPath: /var/log/nginx
          name: nginx-log
        ports:
        - containerPort: 80
      volumes:
      - name: nginx-data
        persistentVolumeClaim:
          claimName: nfs-nginx-data
      - name: nginx-log
        persistentVolumeClaim:
          claimName: nfs-nginx-log
##创建service
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-test
  labels:
    name: nginx-test
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    name: http
    nodePort: 30180
  selector:
    name: nginx-test
EOF

##创建资源
kubectl apply -f nfs-static-nginx-dp-many.yaml -n ${NAMESPACE}
```


# 二、nginx使用nfs动态PV

`https://github.com/Lancger/opsfull/blob/master/components/external-storage/3%E3%80%81%E5%8A%A8%E6%80%81%E7%94%B3%E8%AF%B7PV%E5%8D%B7.md`

## 1、动态nfs-dynamic-nginx.yaml

通过参数控制在哪个命名空间创建

```bash
##清理命名空间
kubectl delete ns k8s-public

##创建命名空间
kubectl create ns k8s-public

##清理资源
kubectl delete -f nfs-dynamic-nginx-deployment.yaml -n k8s-public

cat >nfs-dynamic-nginx-deployment.yaml<<\EOF
##动态申请nfs-dynamic-pvc
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nfs-dynamic-claim
spec:
  storageClassName: nfs-storage #--需要与上面创建的storageclass的名称一致
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 90Gi
##部署应用nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    name: nginx-test
spec:
  replicas: 3
  selector:
    matchLabels:
      name: nginx-test
  template:
    metadata:
      labels:
       name: nginx-test
    spec:
      containers:
      - name: nginx-test
        image: docker.io/nginx
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: nginx-data
        ports:
        - containerPort: 80
      volumes:
      - name: nginx-data
        persistentVolumeClaim:
          claimName: nfs-dynamic-claim
##创建service
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-test
  labels:
    name: nginx-test
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    name: http
    nodePort: 30090
  selector:
    name: nginx-test
EOF

##创建资源
kubectl apply -f nfs-dynamic-nginx-deployment.yaml -n k8s-public

##查看pv资源
kubectl get pv -n k8s-public --show-labels

##查看pvc资源
kubectl get pvc -n k8s-public --show-labels

##查看pod
$ kubectl get pods -n k8s-public
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-544f569478-5t8wm   1/1     Running   0          40s
nginx-deployment-544f569478-8gks5   1/1     Running   0          40s
nginx-deployment-544f569478-pw96x   1/1     Running   0          40s

#可以看到，nginx应用已经部署成功。
#nginx应用的数据目录是使用的nfs共享存储，我们在nfs共享的目录里加入index.html文件，然后再访问nginx-service暴露的端口
#切换到到nfs-server服务器上

#注意动态的在这个目录，创建的目录命名方式为 “namespace名称-pvc名称-pv名称”
/data/nfs/kube-public-test-claim-pvc-ad304939-e75d-414f-81b5-7586ef17db6c

echo "Test NFS Share discovery with nfs-dynamic-nginx-deployment" > /data/nfs/kube-public-test-claim-pvc-ad304939-e75d-414f-81b5-7586ef17db6c/index.html

#在浏览器上访问kubernetes主节点的 http://master:30090，就能访问到这个页面内容了
```

![](https://github.com/Lancger/opsfull/blob/master/images/dynamic-pv.png)


参考文档：

https://kubernetes.io/zh/docs/tasks/run-application/run-stateless-application-deployment/  

https://blog.51cto.com/ylw6006/2071845 在kubernetes集群中运行nginx
