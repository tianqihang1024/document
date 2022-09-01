# Docker-ElasticSearch

1、下载镜像

```shell
docker pull elasticsearch:7.2.0
```

2、创建容器

```shell
docker run --name elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -d elasticsearch:7.2.0
```

3、查看安装结果

测试网址：[81.70.96.232:9200](http://81.70.96.232:9200/)

4、修改配置

```shell
# 进入容器
docker exec -it elasticsearch /bin/bash
# 切换配置目录
cd /usr/share/elasticsearch/config/
# 编辑配置文件
vi elasticsearch.yml
# 配置文件追加内容
http.cors.enabled: true
http.cors.allow-origin: "*"

# 重启容器
docker restart 容器名 | 容器ID
```

5、安装`ik`分词器

```shell
# 切换到插件目录
cd /usr/share/elasticsearch/plugins/
# 下载指定的ik分词器，需要和es保持一致的版本
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.2.0/elasticsearch-analysis-ik-7.2.0.zip
# 推出docker容器
exit | ctrl + d
# 重启容器
docker restart 容器名 | 容器ID
```

6、与`Docker`绑定生命周期

```shell
sudo docker container update --restart=always 容器ID
```





# Docker-kibana

1、下载镜像

```shell
docker pull kibana:7.2.0
```

2、创建容器，使用`--link`参数连接到elasticsearch容器

```shell
docker run --name kibana --link=elasticsearch:test  -p 5601:5601 -d kibana:7.2.0
```

3、安装完成以后需要启动kibana容器

```shell
docker start 容器名 | 容器ID
```

4、查看`Docker`分配给`ElasticSearch`的`IP`地址

```shell
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' 容器ID
```

5、修改配置

```shell
# 进入容器
docker exec -it 容器名 | 容器ID /bin/bash
# 切换配置目录
cd config/
# 编辑配置文件
vi kibana.yml
# 配置文件变更
elasticsearch.hosts: [ "http://elasticsearch:9200" ]
# elasticsearch 改为查询的结果 
elasticsearch.hosts: [ "http://172.17.0.3:9200" ]
# 追加汉语配置
i18n.locale: "zh-CN"

# 重启容器
docker restart 容器名 | 容器ID
```

6、与`Docker`绑定生命周期

```shell
sudo docker container update --restart=always 容器ID
```

7、测试

`Kibana`管理页面：[Console - Kibana](http://81.70.96.232:5601/app/kibana#/dev_tools/console?_g=())
