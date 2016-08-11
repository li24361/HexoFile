title: "Tair 安装指南"
date: 2016-08-11 21:41:01
tags: ”教程“
---

###准备工作
1. 先配置 yum 源，这里直接用阿里云的 yum 源即可。配置文档见：http://mirrors.aliyun.com/help/centos  ，另外这个 minimal 版本是没有 wget 的，可以 yum 装下，我干脆直接把相关文件配置贴过去了（首次用 yum 从官方源下载一次列表文件够慢的）。
配置好之后可以执行yum update再重启让新内核生效（非必要，建议更新到最新版本）。
2. 接下来是 Tair 的安装和配置，先用 yum 安装依赖包和构件工具，直接执行：
```
yum install -y svn automake autoconf libtool vim gcc gcc-c++ gdb zlib-devel boost-devel
```
3. 检出tb-common-utils代码 
```
svn co -r 18 http://code.taobao.org/svn/tb-common-utils/trunk tb-common-utils
```
__注意： 这里不要checkout最新版本，version18以后的修改导致部分接口不能前向兼容。__
4. 检出tair
```
svn checkout http://code.taobao.org/svn/tair/trunk/ tair
```

---------------
####安装工作
1. 设置库文件的安装目录 (我直接加到 ~/.bashrc 了，别忘了执行 source ~/.bashrc)
路径按照实际安装的路径进行修改
```
export TBLIB_ROOT="/root/lib"
```

2. 编译安装 tb-common-utils
```
cd ~/tb-common-utils
./build.sh
```
3. 编译安装 Tair 
```
cd ~/tair
./bootstrap.sh
./configure --with-release=yes
make
make install
```
4.   复制配置文件
```
cd ~/tair_bin
cp etc/configserver.conf.default etc/configserver.conf
cp etc/group.conf.default etc/group.conf
cp etc/dataserver.conf.default etc/dataserver.conf
```
5. 查看本机ip和网卡，如下图，
```
>[admin@hry_server4 ~]$ ip a
    ……中间省略……
6: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether c8:1f:66:e5:6c:25 brd ff:ff:ff:ff:ff:ff
    inet 10.135.111.22/24 brd 10.135.111.255 scope global bond0
    inet6 fe80::ca1f:66ff:fee5:6c25/64 scope link 
       valid_lft forever preferred_lft forever
```
可知 __本机ip:10.135.111.22 网卡bond0__
6. 修改配置文件
####configserver.conf
```
[public]
config_server=10.135.111.22:5198 #设置为本机ip
#config_server=192.168.1.2:5198 #单机需要注释第二个
[configserver]
port=5198
log_file=logs/config.log
pid_file=logs/config.pid
log_level=warn
group_file=etc/group.conf
data_dir=data/data
dev_name=bond0 #这里改成上面查到的网卡
```
####group.conf
```
~
[admin@hry_server4 etc]$ vi group.conf

#group name
[group_1]
# data move is 1 means when some data serve down, the migrating will be start.
# default value is 0
_data_move=0
#_min_data_server_count: when data servers left in a group less than this value, config server will stop serve for this group
#default value is copy count.
_min_data_server_count=1
#_plugIns_list=libStaticPlugIn.so
_build_strategy=1 #1 normal 2 rack
_build_diff_ratio=0.6 #how much difference is allowd between different rack
# diff_ratio =  |data_sever_count_in_rack1 - data_server_count_in_rack2| / max (data_sever_count_in_rack1, data_server_count_in_rack2)
# diff_ration must less than _build_diff_ratio
_pos_mask=65535  # 65535 is 0xffff  this will be used to gernerate rack info. 64 bit serverId & _pos_mask is the rack info,
_copy_count=1
_bucket_number=1023
# accept ds strategy. 1 means accept ds automatically
_accept_strategy=1

# data center A
_server_list=10.135.111.22:5191 #修改为本机ip，其余地址注释掉
#_server_list=192.168.1.2:5191
#_server_list=192.168.1.3:5191
#_server_list=192.168.1.4:5191

# data center B
#_server_list=192.168.2.1:5191
#_server_list=192.168.2.2:5191
#_server_list=192.168.2.3:5191
#_server_list=192.168.2.4:5191

#quota info
#这里将0这个namespace（area）的配额稍微改大了一点，之后的客户端使用namespace 0进行读写访问就行
_areaCapacity_list=0,1124000;
```
####dataserver.conf
```
[public]
config_server=10.135.111.22:5198
#config_server=192.168.1.2:5198
……
#
#mdb size in MB
#建议设置2^n倍数大，但是最小512MB
slab_mem_size=1024 
……
dev_name=bond0#这里一定要修改为本机网卡，否则否则会在put的时候，报 put: server can not work
```
7. 最后是启动步骤，如果报错，则为配置有问题
```
# 设置 tmpfs 运行大小
./set_shm.sh
# 启动 DataServer
./tair.sh start_ds
# 启动 ConfigServer
./tair.sh start_cs
# 检查下进程在否
pgrep -lf tair
```
结果如下，大功告成：
![](http://obq9fd5ou.bkt.clouddn.com/16-8-11/96152199.jpg)
8. 连接测试
![](http://obq9fd5ou.bkt.clouddn.com/16-8-11/88089514.jpg)
java端使用：
```
    public TairOperatorImpl(String masterConfigServer,
                            String slaveConfigServer,
                            String groupName,
                            int namespace) {
        System.out.println("init tair manager");
        this.namespace = namespace;

        // 创建config server列表
        List<String> confServers = new ArrayList<String>();
        confServers.add(masterConfigServer);
//        confServers.add(slaveConfigServer); // 可选

        // 创建客户端实例
        tairManager = new DefaultTairManager();
        tairManager.setConfigServerList(confServers);

        // 设置组名
        tairManager.setGroupName(groupName);
        // 初始化客户端
        tairManager.init();
    }
    
    //调用
    TairOperatorImpl tairOperator = new TairOperatorImpl("10.135.111.22:5198", "10.135.111.22:5198","group_1","0");
```

>参考资料：https://bbs.aliyun.com/read/279531.html?spm=5176.bbsl254.0.0.9lyObN
    




