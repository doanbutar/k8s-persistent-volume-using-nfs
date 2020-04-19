# Kubernetes persistent volume using nfs
Source: https://docs.docker.com/ee/ucp/kubernetes/storage/use-nfs-volumes/

## Prerequisite: Install NFS Server or Setup NFS server using docker container
 1. [Install NFS Server](https://www.howtoforge.com/nfs-server-and-client-on-centos-7)
 2. [Setup docker container for NFS](https://hub.docker.com/r/itsthenetwork/nfs-server-alpine)

In this example, I am using step 1(Install NFS Server on one of my worker node ip-172-31-18-40)

On NFS Server, create new DIR as below and add demo.html for demo purpose.\
*mkdir -p /export/volumes/pod*\
*echo “This is NFS Server” > /export/volumes/pod/demo.html*

Export /export/volumes DIR on NFS Server and allow access to any nodes.
*# cat /etc/exports*\
*/export/volumes *(rw,no_root_squash,no_subtree_check)* 

### Deploying PV, PVC and workload 

1. Deploy NFS PV
```
# kubectl apply -f nfs-pv.yaml
persistentvolume/pv-nfs-data created
   
# kubectl get pv pv-nfs-data
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-nfs-data  1Gi     RWX        	  Retain    Available                                    2m4s  
```

2. Deploy PVC
```
#kubectl apply -f nfs-pvc.yaml
persistentvolumeclaim/pvc-nfs-data created

# kubectl get pvc pvc-nfs-data
NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-nfs-data   Bound    pv-nfs-data   1Gi        RWX                           16s

# kubectl get pv pv-nfs-data
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
pv-nfs-data   1Gi        RWX            Retain           Bound    default/pvc-nfs-data                           17m
```

3. Define label to the worker nodes in order to set constraint of the deployment only run on worker nodes.\
In this example I have 2 worker nodes. Please change worker01 and worker02 to your real node name. \
This is just extra step. You can skip this step but you need to remove nodeSelector field on nfs-nginx.yaml.
 
```
# kubectl label nodes <worker01> role=worker
# kubectl label nodes <worker02> role=worker
```

4. Deploy nginx deployment
```
# kubectl apply -f nfs-nginx.yaml
deployment.apps/nginx-nfs-deployment created
service/nginx-nfs-service created

$ kubectl get pods -l app=nginx -o wide
NAME                                    READY   STATUS    RESTARTS   AGE     IP                NODE                        
nginx-nfs-deployment-6497f77bff-q4q8r   1/1     Running   0          6m47s   192.168.172.196   worker01

# kubectl get svc nginx-nfs-service
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-nfs-service   ClusterIP   10.96.243.201   <none>        80/TCP    2m19s

//Access the application from one of the node in the cluster using CLUSTER-IP
# curl http://10.96.243.201/web-app/demo.html
Hello from NFS Server

//Access the nginx container and check the application directory. Nginx pod running on Node worker01
# docker ps -fname=nginx
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS               NAMES
0e9d14624686        nginx                    "nginx -g 'daemon of…"   10 minutes ago      Up 10 minutes                           k8s_nginx_nginx-nfs-deployment-6497f77bff-q4q8r_default_3318cd07-8206-11ea-9243-0242ac11000b_0

# docker exec -it 0e9d14624686 bash
root@nginx-nfs-deployment-6497f77bff-q4q8r:/# ls -l /usr/share/nginx/html/web-app/
total 4
-rw-r--r--. 1 root root 30 Apr 13 17:11 demo.html
root@nginx-nfs-deployment-6497f77bff-q4q8r:/# cat /usr/share/nginx/html/web-app/demo.html
Hello from NFS Server
```
