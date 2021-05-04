# redis-ha

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

