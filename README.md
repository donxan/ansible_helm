## 安装 helm

`Helm`致力于成为k8s集群的应用包管理工具，希望像linux 系统的`RPM` `DPKG`那样成功；确实在k8s上部署复杂一点的应用很麻烦，需要管理很多yaml文件（configmap,controller,service,rbac,pv,pvc等等），而helm能够整齐管理这些文档：版本控制，参数化安装，方便的打包与分享等。  
- 建议积累一定k8s经验以后再去使用helm；对于初学者来说手工去配置那些yaml文件对于快速学习k8s的设计理念和运行原理非常有帮助，而不是直接去使用helm，面对又一层封装与复杂度。

为了安全，在helm客户端和tiller服务器间建立安全的SSL/TLS认证机制；tiller服务器和helm客户端都是使用同一CA签发的`client cert`，然后互相识别对方身份。建议通过本项目提供的`ansible role`安装，符合官网上介绍的安全加固措施，在delpoy节点运行:  
首先克隆ansible playbook：

```
git clone https://github.com/donxan/ansible_helm.git

```


``` bash
# 1.如果已安装，需要重新安装，使用 helm reset 清理
# 2.配置默认helm参数 vim /etc/ansible/roles/helm/vars/main.yml
# 3.执行安装
# ansible-playbook /etc/ansible/roles/helm/helm.yml

```

简单介绍下`/roles/helm/tasks/main.yml`中的步骤

```  
[root@master software]# wget -qO- https://storage.googleapis.com/kubernetes-helm/helm-v2.12.1-linux-amd64.tar.gz | tar -zx  
[root@master software]# ls  
harbor linux-amd64 traefik  
[root@master software]# mv linux-amd64/helm /etc/ansible/bin/  

```
- 1-下载最新release的helm客户端到/etc/ansible/bin目录下，再由它自动推送到deploy的{{ bin_dir }}目录下

- 2-由集群CA签发helm客户端证书和私钥,使用ansible部署时会自动签发。
- 3-由集群CA签发tiller服务端证书和私钥
- 4-创建tiller专用的RBAC配置，只允许helm在指定的namespace查看和安装应用
- 5-安全安装tiller到集群，tiller服务启用tls验证
- 6-配置helm客户端使用tls方式与tiller服务端通讯
### 安装完成后，查看版本
**注意client,server的版本要一致，否则会报错，使用helm安装应用也会报错** 


```
# helm version --tls
Client: &version.Version{SemVer:"v2.12.1", GitCommit:"02a47c7249b1fc6d8fd3b94e6b4babf9d818144e", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.12.1", GitCommit:"02a47c7249b1fc6d8fd3b94e6b4babf9d818144e", GitTreeState:"clean"}

```

### 注意因使用了TLS认证，所以helm命令执行分以下两种情况 

- 执行与tiller服务有关的命令，比如 `helm ls` `helm version` `helm install`等需要加`--tls`参数
- 执行其他命令，比如`helm search` `helm fetch` `helm home`等不需要加`--tls`

```
[root@master harbor-helm]# kubectl get po,svc -n kube-system -l app=helm
NAME                                READY     STATUS    RESTARTS   AGE
pod/tiller-deploy-d658b9c47-fqrx7   1/1       Running   0          4s

NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
service/tiller-deploy   ClusterIP   10.68.42.136   <none>        44134/TCP   4s

```


## 准备pv
这里使用了nfs pv,先到 NFS Server 上建立四個資料夾`mkdir nfs{3..6}`,本次不需要配置pvc,后面会自动配置

```
for i in {3..6}; do
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv00${i}
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce #需要注意
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /volume1/harbor/nfs${i}
    server: 192.168.2.4
EOF
done 


```

```
[root@master pv]# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                     STORAGECLASS   REASON    AGE
pv001     10Gi       RWX            Retain           Bound       default/myclaim                                    21h
pv002     100Gi      RWX            Retain           Bound       default/nginx-svc-claim                            20h
pv003     100Gi      RWO            Recycle          Available                                                      12s
pv004     100Gi      RWO            Recycle          Available                                                      12s
pv005     100Gi      RWO            Recycle          Available                                                      12s
pv006     100Gi      RWO            Recycle          Available                                                      11s


```


## 使用helm安装harbor到kubernetes上
当Persistent Volume准备完成后,隆Harbor helm chart代码:

```
git clone https://github.com/goharbor/harbor-helm
cd harbor-helm
git checkout 0.3.0 #目前最新的分支是0.3.0

```
更新依赖,使用 Helm 部署 Harbor
`values.yaml`包含很多配置参数，根据需要修改。参考[官方文档](https://github.com/vmware/harbor/tree/master/contrib/helm/harbor#configuration)

```
helm dependency update
elm install . --debug --name hub --set externalDomain=harbor.abcgogo.com --tls

```
使用`kubectl`查看 Harbor 是否部署成功

```
[root@master harbor-helm]# kubectl get pod -o wide | grep harbor            
hub-harbor-adminserver-77dc5bb8c4-xchm7     1/1       Running   4          7h        172.20.4.95    192.168.2.12   <none>
hub-harbor-chartmuseum-77895d6c6-wxh7q      1/1       Running   0          7h        172.20.2.101   192.168.2.11   <none>
hub-harbor-clair-6575949b87-wrgs7           1/1       Running   4          7h        172.20.4.97    192.168.2.12   <none>
hub-harbor-database-0                       1/1       Running   0          6h        172.20.2.103   192.168.2.11   <none>
hub-harbor-jobservice-7c5f74d9d-jrrxr       1/1       Running   2          7h        172.20.3.83    192.168.2.13   <none>
hub-harbor-notary-server-75f64bfcd-7tlts    1/1       Running   0          7h        172.20.3.82    192.168.2.13   <none>
hub-harbor-notary-signer-7fbf77648d-4485x   1/1       Running   0          7h        172.20.4.96    192.168.2.12   <none>
hub-harbor-registry-674c7f487d-5xzqt        1/1       Running   0          7h        172.20.2.102   192.168.2.11   <none>
hub-harbor-ui-759b87c94c-kg7gj              1/1       Running   1          6h        172.20.4.99    192.168.2.12   <none>

```
可以使用`kubectl get pod,svc,ingress -o wide | grep harbor
`查看更多信息,因之前配置了traefik，所以这里使用traefik ingree,这里已经配置成功。
![](https://s1.51cto.com/images/blog/201901/17/ff8dcd67690edd7e30e612de4b03ab20.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)
dns服务器上配置harbor.abcgogo.com的dns，也可以修改本地hosts
接著可以使用浏览器查看Harbor Web UI,默认登录账号密码：admin/Harbor12345
![](https://s1.51cto.com/images/blog/201901/17/fd99c81ccc39ac54fab0dd4891b2ccdf.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

## 开始使用harbor

确认Helm部署的Harbour没问题后，可以开始使用Harbor。以下将说明如何让Docker Client如何存取私有的Registry以及一些基本操作。

首先，要让Docker能存取私有的注册表需要对Docker做一些小小的设定，而设定方式有以下两种方式：

1.  **使用自带的凭证（CA）**：为了安全性考量，私有的注册表自带凭证。当Docker Client与私有的注册表进行沟通时都需要使用此凭证，而这也是目前Docker官方推荐的做法;
2.  **设置不安全的注册表**：直接设定为不安全的注册表，因为安全性的考量目前此方法官方并不推荐。

而两种方法选择其中一种设定即可。

## 设定凭证（CA）

因为我们部署的Harbour是有自带凭证（CA），所以需要再Docker Client加入凭证，这样Docker Client才有办法存取到私有的注册表。

首先，在Kubernetes Master使用以下指令取得凭证

```
pv006     100Gi      RWO            Recycle          Bound     default/redis-data-hub-redis-master-0                                  12h
[root@master harbor-helm]# kubectl get secret/hub-harbor-ingress -o jsonpath="{.data.ca\.crt}" | base64 --decode 
-----BEGIN CERTIFICATE-----
adfadsfaAwIBAgIRAJmXXxn40kWHcoOj6dfjtgIwDQYJKoZIhvcNAQELBQAw
FDESMBAGA1UEAxMJaGFyYm9yLWNhMB4XDTE5MDExNTA4MzExMloXDTI5MDExMjA4
MzExMlowFDESMBAGA1UEAxMJaGFyYm9yLWNhMIIBIjANBgkqhkiG9w0BAQEFAAOC
AQ8AMIIBCgKCAQEA34RNLkEvdHQDufGgZRJmL3Tki6IJyPnKQc0PdtIZvKYCSMut
wyiOeS/VEk/GNtEMet1+Vf6EbclnH0kR6aHl4t11S/9C1kSwRwm48lTkeDKk79Q8
4p/z8GfFW25BTLLcDE9BjdE71Zl4vKX3Spf9iFUWmKiSDi682xXC66/CUjGlyts3
AZOXpGUdmgOGKWNGQ0EBWThVo1krytj/6qKLt7sB08+/KzUSMX+k9Dl5G6yN/7Lt
rwmAPW3KqZY6ZqYvSb7Big/9xgCE2lO3C/rVOQIDAQABo0IwQDAOBgNVHQ8BAf8E
BAMCAqQwHQYDVR0lBBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMA8GA1UdEwEB/wQF
MAMBAf8wDQYJKoZIhvcNAQELBQADggEBAHoqJZDqAiMkcbO273n9GjWTXQgqBIkb
mBltkXU1oWa6wDrF/ZrTU25RUftDZ1QPYGsXRGpz/9pODcGDVDPK+45QH2Fjtldj
KOycOWIEdolFP6aDuqxiSaRrC6XjM9fyPSRSjS3kSHVQJ91c7PwD+9v1U6kwNkvh
CsfwerqerwerwsdfMr8eLvjpKitaHQLHkgqOIDquxV8dNIMSzvfGJw77lzhJ3ere
y+UgzIBpPLc8FpsuwjKmBnSDjOvj8OWGmJyTBM2KfDC1dk+ZXTsErpY=
-----END CERTIFICATE-----

```
取得凭证后，在每一台Docker Client加入以下凭证：

```
mkdir -p /etc/docker/certs.d/harbor.abcgogo.com/
cat <<EOF > /etc/docker/certs.d/harbor.abcgogo.com/ca.crt
-----BEGIN CERTIFICATE-----
MIIC9TCCAd2gAwIBAgIRAJmXXxn40kWHcoOj6dfjtgIwDQYJKoZIhvcNAQELBQAw
FDESMBAGA1UEAxMJaGFyYm9yLWNhMB4XDTE5MDExNTA4MzExMloXDTI5MDExMjA4
MzExMlowFDESMBAGA1UEAxMJaGFyYm9yLWNhMIIBIjANBgkqhkiG9w0BAQEFAAOC
AQ8AMIIBCgKCAQEA34RNLkEvdHQDufGgZRJmL3Tki6IJyPnKQc0PdtIZvKYCSMut
wyiOeS/VEk/GNtEMet1+Vf6EbclnH0kR6aHl4t11S/9C1kSwRwm48lTkeDKk79Q8
4p/z8GfFW25BTLLcDE9BjdE71Zl4vKX3Spf9iFUWmKiSDi682xXC66/CUjGlyts3
AZOXpGUdmgOGKWNGQ0EBWThVo1krytj/6qKLt7sB08+/KzUSMX+k9Dl5G6yN/7Lt
mBltkXU1oWa6wDrF/ZrTU25RUftDZ1QPYGsXRGpz/9pODcGDVDPK+45QH2Fjtldj
KOycOWIEdolFP6aDuqxiSaRrC6XjM9fyPSRSjS3kSHVQJ91c7PwD+9v1U6kwNkvh
CqEWg9ejsw0jNmxNwoJfQlz0Y+qz3fzhQxXnaZdDDXrvq9wKMr8eLvjpKitaHQLH
DIcQ11JR7dU3qVmegv3YxEB5S1cxwvyGH12kgqOIDquxV8dNIMSzvfGJw77lzhJ3
y+UgzIBpPLc8FpsuwjKmBnSDjOvj8OWGmJyTBM2KfDC1dk+ZXTsErpY=
-----END CERTIFICATE-----
EOF

```

修改完成后，重新启动docker.service：

    
    systemctl daemon-reload
    systemctl restart docker.service

更快捷高效的方法，使用for循环同步ca:
```
for host in `seq 11 14`;do
rsync -av /etc/docker/certs.d/harbor.abcgogo.com/ca.crt root@192.168.2.${host}:/etc/docker/certs.d/harbor.abcgogo.com/
ssh root@192.168.2.${host} "systemctl daemon-reload && systemctl restart docker.service"
done       
```

通过命令行登录测试：

    [root@master harbor-helm]# docker login harbor.abcgogo.com
    Username: admin
    Password: 
    Login Succeeded

**在kubernetes中使用harbor,为了避免输入账号密码，需要创建secret**

以下操作在master上执行：
 1）创建secret 
 

     kubectl create secret docker-registry harbor-secret --docker-server=harbor.abcgogo.com --docker-username=admin --docker-password=Harbor123
 

 
创建完成后，可以用以下命令查看：

     # kubectl get secret
推送镜像到harbor

```
[root@master harbor-helm]# docker tag tomcat harbor.abcgogo.com/aikerlinux/tomcat:latest
[root@master harbor-helm]# docker push harbor.abcgogo.com/aikerlinux/tomcat:latest
The push refers to a repository [harbor.abcgogo.com/aikerlinux/tomcat]


```
推送成功后,可以通过web查看
![](https://s1.51cto.com/images/blog/201901/17/6863beca2266ce046b63c3b9475aac5c.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)
从harbor获取image

```
docker pull harbor.abcgogo.com/aikerlinux/tomcat:latest


```
