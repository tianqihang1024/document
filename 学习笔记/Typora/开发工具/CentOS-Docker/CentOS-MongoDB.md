

# 待安装







```bash
docker run -d --name mongodb -p 27018:27017 -v /usr/local/mongodb/datadb:/data/db -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=admin --privileged=true --restart always mongo:4.2

docker run -d --name mongodb -p 27018:27017 -v /usr/local/mongodb/data:/data/db -v /usr/local/mongodb/backup:/data/backup -v /usr/local/mongodb/logs:/data/log -v /usr/local/mongodb/conf:/data/conf -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=admin --privileged=true --restart always mongo:4.2


```



```

db.createUser({ user: 'admin', pwd: 'admin', roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] });
db.auth("admin","admin");

use item;

db.item.save({name:"zhangsan"});

db.item.find();
```

