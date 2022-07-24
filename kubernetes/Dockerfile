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
