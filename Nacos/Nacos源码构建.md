1.先到GitHub pull下最新的代码  
2.导入IDEA中  
3.找到console模块下的/Nacos主类，启动  
4.启动前，需设置必要的JVM参数 `-Dnacos.standalone=true -Dnacos.home=D:\\Run\\Nacos`  
5.启动成功，开始debug吧  

###############
build docker image
在`console`moudle下创建Dockerfile,内容如下：
```
FROM openjdk:latest
ENV WORK_PATH /app

COPY target/nacos-server.jar $WORK_PATH/nacos-server.jar
WORKDIR $WORK_PATH

ENV JAVA_OPTS="-Xms256m -Xmx256m -Dnacos.standalone=true -Dnacos.home=/app/nacos -Duser.timezone=GMT+08 -Dfile.encoding=UTF-8 -Djava.security.egd=file:/dev/./urandom "

ENTRYPOINT java $JAVA_OPTS -jar nacos-server.jar
```