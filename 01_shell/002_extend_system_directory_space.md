# 扩展docker使用的/var/lib 目录空间

* 当前系统目录如图

  ![系统目录](shell\file.jpg)

​       可以看到，`/var` 只有15G， `/data` 有1T空间，希望从 `/data`空间中切出来100G，分配给`/var`

>  注意：/data 目录下没有文件和数据

* 非root用户登录

  ```shell
  #切换当前目录为/tmp
  cd /tmp
  su -
  #切换root用户
  umount /data
  #rhel是vg名称，lvs 查看vg 名称，删除/data对应的lv ！！！注意/data数据会丢失
  lvremove /dev/rhel/data
  #扩容
  lvextend -L +100G /dev/rhel/var
  #同步文件系统，该示例中文件系统为xfs， xfs_growfs只支持增大！
  xfs_growfs /var
  #如果还需要挂载剩余空间到/data
  lvcreate -l 100%VG -n data rhel
  mount /dev/rhel/data /data
  #检查
  lvs
  df -h
  ```


