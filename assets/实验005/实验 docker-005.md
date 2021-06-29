# 实验 docker-005 docker的生命周期
```bash
 docker run centos sleep 10
 docker run -itd ubuntu /bin/bash -c "while true; do sleep 1; echo I_am_in_container; done"
```

 两种连接容器的方式：

 attach

 exec

