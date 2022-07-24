# Kubernetes 部署 Vue 项目

### (1) 新建 Dockerfile

```
(1) 新建Dockerfile
# 1
# node

### FROM 引用基础镜像
### - as的作用：
###   - 1. 在输入 ( docker build --target=as的值 )，表示构建具体的镜像，因为 Dockerfile中啃根有多个 FROM 指定基准对象
###   - 2. 在多阶段构建中，( COPY --from=as的值 ) 获取中间镜像层的数据
FROM node as node

### ENV 表示容器中的环境变量
ENV WORK_DIR /app

### WORKDIR 表示进入容器后，默认的工作目录
WORKDIR ${WORK_DIR}

### COPY 拷贝文件
### - 语法：COPY 源路径 目标路径
### - 语法：COPY 宿主机Dockerfile所在的文件夹 容器中的文件夹
### - 语法：COPY --from=as名称  -------- 主要用来拷贝中间镜像次层的数据，主要用于多阶段构建
### - 注意区分 COPY 和 ADD 的区别
COPY . ${WORK_DIR}

### RUN
### - 构建镜像阶段：在新建镜像时执行的命令

### RUN 和 CMD 的区别
### - RUN：构建镜像阶段 ---- 在新建镜像时执行的命令
### - CMD：容器运行阶段 ---- 在容器启动后默认执行的命令和参数
RUN npm install -g cnpm --registry=https://registry.npmmirror.com

RUN cnpm install

RUN cnpm run build

# ------------------------------------
# 2
# nginx
# 这个文件是 ( 多阶段构建 )，同时指定了 node 和 nginx

FROM nginx as nginx

ENV WORK_DIR /app

### COPY from=node 表示将 ( node阶段node容器中的 /app/build文件夹 ) 拷贝到 ( nginx容器的  /usr/share/nginx/html 文件中 )
COPY from=node ${WORK_DIR}/build /usr/share/nginx/html
COPY from=node ${WORK_DIR}/docker/nginx/conf.d /etc/nginx/conf.d
# COPY --from=build ${WORK_DIR}/nginx.conf /etc/nginx/nginx.conf 这个是nginx的配置，而不是容器中nginx的配置
# COPY /docker/nginx/conf.d /root/etc/nginx/conf.d

### EXPOSE 表示暴露端口，类似于 docker run -p
EXPOSE 80

### CMD 是容器启动后默认执行的命令和参数
### 这里之所以要执行nginx -g daemon off; 是为了让容器退出后不关闭
CMD ["nginx", "-g", "daemon off;"]

# 3
# 通过 Dockerfile 构建镜像
# docker build -t ngin_deploy .

# 4
# 4.1 通过镜像生成容器 - docker run -d -p 8020:80 ngin_deploy
# 4.2 如何是通过 kubernetes 来部署的话，直接可以在 deployment 的 pod template 中指定 容器镜像
```

### (2) build 镜像

```
在Dockerfile文件所在目录执行build命令，构建镜像
---

docker build -t xia-vue .
```

### (3) kubernetes 部署

```
(1)
新建 namespace
- cubectl create ns xia



(2)
创建 deployment 的配置yaml文件
(2.1) 创建 deployment 的配置yaml文件
- 注意：我们一般不直接创建pod，而是创建pod控制器deployment
- 命名：xia-deployment-nginx.yaml
apiVersion: apps/v1 # --------------------- 创建该对象所使用的 Kubernetes API 的版本
kind: Deployment # ------------------------ 对象类别：deployment
metadata:
  name: xia-nginx-deployment # ------------ deployment名字
  namespace: xia # ------------------------ 所属命令空间
  labels: # ------------------------------- 标签，( 注意区分 labels 和 namespace 的区别 )
    run: xia-nginx
spec:
  replicas: 3 # ---------------------------- pod副本数
  selector: # ----------------------------- pod选择器
    matchLabels:
      run: xia-nginx # -------------------- [ pod中的label和deployment一一对应 ] deployment所属标签，和template中的pod的标签对应表示匹配
  template: # ----------------------------- 定义 pod 模版
    metadata:
      name: xia-nginx-pod
      namespace: xia
      labels:
        run: xia-nginx # ------------------ [ pod中的label和deployment一一对应 ]
      spec:
        containers:
        - name: xia-nginx-container
          image: xia-vue
          imagePullPolicy: IfNotPresent # - 镜像拉取策略，IfNotPresent Never Alwarys
          ports:
          - containerPort: 80
(2.2) 利用 xia-deployment-nginx.yaml 新建 deployment
- 新建deployment: kubectl create -f xia-deployment-ngnx.yaml
- 查询deployment: kubectl get deployment -n xia



(3)
创建service
(3.1) 创建 service 的配置yaml文件
- 命名：xia-service-nginx.yaml
apiVersion: v1
kind: Service
metadata:
  name: xia-nginx-service
  namespace: xia
  labels:
    run: xia-nginx
spec:
  type: NodePort
  selector:
    run: nginx
  ports:
  - port: 77
    targetPort: 80
(3.2) 利用 xia-service-nginx.yaml 新建 service
- 新建service: kubectl create -f xia-service-nginx.yaml
- 查询service: kubectl get service -n xia

service信息如下
NAME                TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
xia-nginx-service   NodePort   10.0.0.179   <none>        77:32244/TCP   15s



(4)
- 外网访问kubernetes中的服务
  - curl node节点ip + 端口
- 集群内部kubernetes访问
  - node节点输入：curl 10.0.0.179:77


(5)
优化：我们可以把 ( xia-deployment-nginx.yaml ) 和 ( xia-service-nginx.yaml ) 合并成一个
  - 因为：我们两个文件执行的命令是一样的，即 kubectl create -f xxxxxx.yaml
  - 并且：yaml 规范中可以使用 --- 来分隔文件，就可以把多个资源写在同一个yaml文件中，就不用写多个
```

![1658562604497.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2812ccb3765c4dbe809b61ac2c6cd2d0~tplv-k3u1fbpfcp-watermark.image?)
