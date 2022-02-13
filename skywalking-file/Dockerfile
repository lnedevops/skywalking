# 构建springCloud微服务镜像
FROM openjdk:8-jdk
LABEL maintainer=fengsulin

#为每个微服务都开启8080，指定prob生成环境，nacos服务地址和配置中心地址，config配置文件在nacos中的命名空间，文件格式为yml
ENV APP_OPTS="--server.port=8080 --spring.profiles.active=prod --spring.cloud.nacos.discovery.server-addr=nacos-headless.devops:8848 --spring.cloud.nacos.config.server-addr=nacos-headless.devops:8848 --spring.cloud.nacos.config.namespace=prod --spring.cloud.nacos.config.file-extension=yml"
ENV JAVA_OPTS=""
ENV JVM_OPTS="-Dfile.encoding=utf8 -Djava.security.egd=file:/dev/.urandom"

RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone

COPY target /app.jar
EXPOSE 8080
ENTRYPOINT ["/bin/bash","-c","java ${JVM_OPTS} ${JAVA_OPTS} -jar app.jar ${APP_OPTS}"]