
# 部署postgresql
>注意：不能高版本

使用docker安装

`docker-compose.yml`文件内容如下：
````
version: '2'
services:
  postgresql:
    restart: always
    privileged: true
    container_name: postgresql
    image: hub.vpclub.cn/xyyd/postgres:11.7
    volumes:
    - ./data/postgresql:/var/lib/postgresql/data:Z
    ports:
    - 5432:5432
    environment:
    - POSTGRES_PASSWORD=V9c1u691d6
    - TZ=PRC
````

## 创建Kong用户 
### 进入容器
````
docker exec -it postgresql /bin/bash 
````
### 创建数据库用户与数据库
````
su root
su - postgres   
psql 

 #用户名和密码可以自己定，后续部署kong时需要使用
create user kong with password 'kong';
create user konga with password 'kong';
create database kong  owner kong ;
create database konga_db  owner konga ;
#创建数据库指定所属者
\l   # 查看数据库
````

# 初始化数据-注意切换自己的postgresql机器ip 
初始化kong数据库
````

docker run -ti  -e KONG_PG_HOST=XXXXIP -e KONG_PG_PORT=5432 -e KONG_PG_PASSWORD=kong hub.vpclub.cn/xyyd/kong:2.1 sh

kong migrations bootstrap
````
初始化konga数据库
````
docker run --rm hub.vpclub.cn/xyyd/pantsel/konga -c prepare -a postgres -u postgresql://konga:kong@XXXXIP:5432/konga_db
````


# 安装kong
````
export pg_host=XXXXIP
sed -i 's#postgresqlhost#$pg_host#g' kong-external-postgres.yaml
sed -i 's#postgresqlhost#$pg_host#g' kong-control-plane-postgres.yaml
sed -i 's#postgresqlhost#$pg_host#g' konga.yaml

````

## yaml 部署方式(推荐)
>注意：记得修改yaml中postgres对应的ip

### 安装ingress-kong
````
kubectl -n kong apply -f kong-external-postgres.yaml
````

### 安装kong-control-plane
````
kubectl -n kong apply -f kong-control-plane-postgres.yaml
````
此时查看deploy应如下所示
````
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
ingress-kong         1/1     1            1           ...
kong-control-plane   0/1     1            1           ...
````

### 安装证书
````
sh setup_certificate.sh
````
若使用脚本安装失败，尝试手动安装：
````
mkdir temp
cd temp
````

````
cat <<EOF | kubectl  create -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: kong-control-plane.kong.svc
spec:
  request: $(openssl req -new -nodes -batch -keyout privkey.pem -subj /CN=kong-control-plane.kong.svc | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
````

````
kubectl certificate approve kong-control-plane.kong.svc
kubectl -n kong create secret tls kong-control-plane.kong.svc --key=privkey.pem --cert=<(kubectl get csr kong-control-plane.kong.svc -o jsonpath='{.status.certificate}' | base64 --decode)
kubectl delete csr kong-control-plane.kong.svc
````

````
rm privkey.pem
````
此时稍等再查看deploy会看到如下结果
````
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
ingress-kong         1/1     1            1           ...
kong-control-plane   1/1     1            1           ...
````

测试安装是否成功
````
curl -i kong-control-plane.kong:8001
curl -i kong-proxy.kong:80
````


k8s安装konga：
````
kubectl apply -n kong -f konga.yaml
````
添加ingress：
````
kubectl apply -f konga-ingress.yaml -n kong
````