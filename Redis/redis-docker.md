#自定义配置的redis镜像
1.拉取redis官方镜像，可以选择latest获取其他版本
`docker pull redis:6.0.5`
2.编写`Dockerfile`，自定义镜像，主要是重写启动命令，让`redis-server`使用指定的配置文件
```
FROM redis:6.0.5
CMD [ "redis-server", "/usr/local/etc/redis/redis.conf" ]
```
3.根据`Dockerfile`，构建自定义的redis镜像
`docker build -t redis:local .`  
4.使用自定义的redis镜像，编写`docker-compose.yml`
```
version: "3.7"
services:
  redis:
    image: redis:local
    container_name: redis-local
    restart: always
    ports:
      - "6379:6379"
    volumes:
      - D:\Docker\Redis\data:/data
      - D:\Docker\Redis\redis.conf:/usr/local/etc/redis/redis.conf

```
5.在对应的目录创建data目录，作为映射redis持久化的目录
6.在对应的目录创建`redis.conf`，作为自定义的配置
```
# 具体配置文件参考 redis6.conf
```
7.启动redis `docker-compose up -d`

