# MatrixDB部署与监控

Pigsty可用于部署与监控MatrixDB(等效Greenplum 7)
因为目前MatrixDB使用的是PostgreSQL 12的内核，而原生Greenplum仍然使用9.6内核，因此优先使用MatrixDB作为Greenplum实现。


## 实体概念模型

MatrixDB在逻辑上由两部分组成，Master与Segments，两者均由PostgreSQL实例组成，实例分为四类：Master/Standby/Primary/Mirror

* Master为用户直接接触的访问端点，用于承接查询，一套MatrixDB部署仅有一个，通常使用独立节点部署。
* Standby是Master实例的物理从库，用于当Master故障时顶替，是可选的组件，通常也使用独立节点部署。
* 一套MatrixDB部署通常有多个Segment，每个Segment通常由一个必选的 primary 实例与一个 可选的 mirror 实例组成。
* Segment的primary负责实际存储与计算，mirror通常不承担读写流量，当primary宕机时顶替primary，通常与primary分布在不同节点上。
* Segment的primary与mirror分布由MatrixDB安装向导决定，在集群的Segments节点上通常可能存在有多个不同的Segment实例

**部署惯例**
* Master集群 (master/standby) (`gp_role-master`) 构成一个PostgreSQL集群，通常命名包含`mdw`，如`mx-mdw`
* 每个Segment (primary/mirror) 构成一个PostgreSQL集群，通常集群命名包含`seg`，如 `mx-seg1`, `mx-seg2`
* 用户应当显式为集群节点命名，例如 `mx-sdw-1`, `mx-sdw-2`, ...



## 准备安装

MatrixDB / Greenplum 的安装将复用绝大多数 PostgreSQL 任务。配置文件

* [`pigsty-mxdb.yml`](https://github.com/Vonng/pigsty/blob/master/files/conf/pigsty-mxdb.yml) 给出了一个在四节点沙箱环境部署MatrixDB的样例。

使用 `configure -m mxdb`，将自动使用该配置文件作为配置模板。

接下来，您需要准备好MatrixDB的本地安装包，Pigsty提供了社区版MatrixDB的预制软件包 [`matrix.tgz`](https://github.com/Vonng/pigsty/releases/download/v1.4.0-rc/matrix.tgz)，
MatrixDB & Greenplum 的RPM包及其完整依赖已经放入该压缩包中，您可以复用Pigsty的本地yum源：将其解压至 `/www` ，供本地yum源使用。通过以下命令建立MatrixDB本地源：

```bash
# 将MatrixDB RPM包解压至 /www/matrix (nginx home)
curl -SL https://github.com/Vonng/pigsty/releases/download/v1.4.0-rc/matrix.tgz -o /tmp/matrix.tgz
sudo mkdir -p /www
sudo tar -xf /tmp/matrix.tgz -C /www/

# 创建本地repo文件
cat > /www/matrix.repo <<-'EOF'
[matrix]
name=Local Yum Repo MatrixDB
baseurl=http://pigsty/matrix
skip_if_unavailable = 1
enabled = 1
priority = 1
gpgcheck = 0
EOF
```

注意修改 [`node_local_repo_url`](v-nodes.md#node_local_repo_url)，向其添加新Yum源地址，确保所有节点都可以访问该Repo

```yaml
node_local_repo_url:
  - http://pigsty/pigsty.repo
  - http://pigsty/matrix.repo
```



## 开始部署

在四节点沙箱环境中部署MatrixDB，注意，默认将使用DBSU `mxadmin:mxadmin` 作为监控用户名与密码

```bash
./infra.yml -e no_cmdb=true   # 如果您准备在meta节点上部署 MatrixDB Master，添加no_cmdb选项
./nodes.yml                   # 初始化集群的节点，纳入监控
./pigsty-matrix.yml           # 完成MatrixDB安装准备与监控
```

安装完成后，您需要通过MatrixDB 提供的WEB UI完成接下来的安装。打开 [http://matrix.pigsty](http://matrix.pigsty) 或访问 http://10.10.10.10:8240，填入 `pgsql-matrix.yml` 最后输出的初始用户密码进入安装向导。 

按照提示依次添加MatrixDB的节点：10.10.10.11, 10.10.10.12, 10.10.10.13，点击确认安装并等待完成后，进行下一步。

因为监控默认使用 `mxadmin:mxadmin` 作为监控用户名密码，请填入`mxadmin` 或您自己的密码。 

如果您在安装向导中指定了不同的密码， 请一并更改 [`pg_monitor_username`](v-pgsql.md#pg_monitor_username) 与 [`pg_monitor_password`](v-pgsql.md#pg_monitor_password) 变量（如果您使用不同于dbsu的用户，通常还需要在所有实例上配置额外的HBA）。

请注意，目前MatrixDB / Greenplum 在节点上分配 Segment的逻辑并不确定。当初始化完成后，您可以修改 [`pg_instances`](v-pgsql.md#pg_instances) 中Segment实例的定义，并重新部署监控以反映真实拓扑。



## 收尾工作

最后，在Greenplum/MatrixDB Master节点上手工执行以下命令，允许监控组件访问**从库**，并重启生效。

```bash
sudo su - mxadmin
psql postgres -c "ALTER SYSTEM SET hot_standby = on;"  # 配置 hot_standby=on 以允许从库查询
psql matrixmgr -c 'SELECT mxmgr_init_local();'         # 初始化MatrixDB自身监控
gpconfig -c hot_standby -v on -m on                    # 配置 hot_standby=on 以允许从库查询
gpstop -a -r -M immediate                              # 立即重启MatrixDB以生效
```

然后，您便可以从监控系统中，观察到所有MatrixDB集群。MatrixDB Dashboard 提供了关于数据仓库的整体监控概览。



## 可选项目

您可以将 MatrixDB 的 Master集群视作一个普通 PostgreSQL 集群，使用 [`pgsql-createdb`](p-pgsql.md#pgsql-createdb) 与 [`pgsql-createuser`](p-pgsql.md#pgsql-createuser) 创建业务数据库与用户。

```bash
bin/createuser mx-mdw  dbuser_monitor   # 在Master主库上创建监控用户
bin/createdb   mx-mdw  matrixmgr        # 在Master主库上创建监控专用数据库
bin/createdb   mx-mdw  meta             # 在Master主库上创建新数据库
```


