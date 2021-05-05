# redis-ha
搭建集群

参考链接：https://github.com/Mountains-and-rivers/mongo-replica-set

制作镜像  

```
cd docker 

docker build -t redis-sentinel-ha-cluster:1.0 .
```

部署

```
kubectl apply -f .
```
```
kubectl get svc
NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
kubernetes               ClusterIP   10.96.0.1      <none>        443/TCP     93m
redis-master-service     ClusterIP   10.109.40.60   <none>        6379/TCP    23s
redis-sentinel-service   ClusterIP   10.102.2.210   <none>        26379/TCP   23s
```
```
kubectl get pod
NAME                       READY   STATUS    RESTARTS   AGE
busybox                    1/1     Running   0          26m
dnsutils                   1/1     Running   0          26m
redis-master-service-0     1/1     Running   0          25m
redis-sentinel-service-0   1/1     Running   0          26m
redis-sentinel-service-1   1/1     Running   0          26m
redis-sentinel-service-2   1/1     Running   0          26m
redis-slave-service-0      1/1     Running   0          25m
```
域名解析验证  
 #格式 $(podname).$(service name).$(namespace)

```
kubectl exec -ti busybox sh -- nslookup redis-sentinel-service-0.redis-sentinel-service
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      redis-sentinel-service-0.redis-sentinel-service
Address 1: 10.244.2.27 redis-sentinel-service-0.redis-sentinel-service.default.svc.cluster.local
```

查看密码
```

kubectl exec -ti redis-master-service-0 sh -- cat /redis-master/redis.conf |grep pass

# If the master is password protected (using the "requirepass" configuration
masterauth rpasswd
# resync is enough, just passing the portion of data the slave missed while
# 150k passwords per second against a good box. This means that you should
# use a very strong password otherwise it will be very easy to break.
requirepass rpasswd

```
查看角色

```
kubectl exec -ti redis-master-service-0 sh -- redis-cli -a rpasswd
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
127.0.0.1:6379> role
1) "master"
2) (integer) 0
3) (empty array)
127.0.0.1:6379> 
```

nodejs连接测试[TODO]

```
# This official base image contains node.js and npm
FROM node:7

ARG VERSION=1.0.0

# Copy the application files
WORKDIR /usr/src/app
COPY package.json app.js LICENSE /usr/src/app/
COPY lib /usr/src/app/lib/

LABEL license=MIT \
      version=$VERSION

# Set required environment variables
ENV NODE_ENV production

# Download the required packages for production
RUN npm update

# Make the application run when running the container
CMD ["node", "app.js"]
```
app.js
```
const sentinel = require('redis-sentinel');
const sentinels = [ // 哨兵节点的地址与端口集合
    { host: 'redis-sentinel-service.default.svc.cluster.local', port: 26379 }
]
const masterName = 'edis-master-service.default.svc.cluster.local'; // master节点的名字
const opts = { // node_redis的相关属性设置
    auth_pass: 'password', // 在版本较低的node_redis中使用auth_password作为密码，redis-sentinel及时属于版本较低的node_redis
    // password: 'password', // 在版本高的node_redis中的密码属性
    db: 0, // 如果设置，客户端将在连接上运行Redis select命令。
};

// 创建redisClient实例
const redisClient = sentinel.createClient(sentinels, masterName, opts);

redisClient.set('testName', 'lxjTest1');
```

