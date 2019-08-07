# ansible 简明使用手册

## 安装

``` shell
# 安装 centos7 的 epel-release 源
sudo yum install -y epel-release
sudo yum install -y ansible

# 安装 sshpass 和 bind-utils , 用于免密
sudo yum install -y sshpass bind-utils

```

## 使用方法

### 目标机免密登录

安装 ansible 的主机作为主控机. 一般通过 ssh 连接目标机, 在目标机上执行任务. 需要在目标机上做免密登录.

ansible 和 ansible-playbook 的区别是: ansible 命令一般用于临时性简单的任务. 而 ansible-playbook 可以将多个任务组合起来, 更加工程化的组织任务.

#### 第一步 创建公私钥对

``` shell
## 创建 pri/pub key 对. 一直回车即可.
ssh-keygen -t rsa
```

#### 第二步 编写第一个 ansible inventory 文件

inventory 文件类似 /etc/hosts . 目的是指明目标主机和给目标主机分组, 定义组变量, 主机变量等. 默认的 inventory 文件是 /etc/ansible/hosts. inventory 格式为 ini 格式. 下面描述基本写法, 更多用法参考 [Inventory 文件](http://www.ansible.com.cn/docs/intro_inventory.html). 假定第一个 inventory 文件名为 example-hosts.

``` ini
# 注释
## 组名. 正确的组名, 主机别名遵守域名规范. 使用小写字母开头, 可以使用字母,数字, 可以用 dot(.) 分隔多个单词. 
## 格式如下
# [groupname]
# target.host.alias key=value
# [groupname:vars]
# key=value

## [] 用于分组. 下面例子中组名为 candy_console
[candy_console]
## 组下面是主机信息. 第一列是主机别名. 后面是主机变量. ansible 内置了很多变量, 一般以 ansible_ 开头.
## ansible_ssh_host: 主机ip或名称. 
## ansible_ssh_port: ssh 端口. 
## ansible_ssh_user: ssh 登录用户
## ansible_ssh_pass: ssh 登录用户的密码. 一般在目标机登录用户不一致,并且未做免密前需要填写. 如果密码一致, 不用填写. 免密后删除该变量.
n1.console.candy ansible_host=192.168.39.101
n2.console.candy ansible_host=192.168.39.102
# 批量主机
#n[01:30].example.org

## 组变量. 适用于整个 candy_console 组.
[candy_console:vars]
ntp_server=ntp.example.org
```

#### 第三步 通过 playbook 免密

保存下面的内容到 add-ssh-keys.yml 中

``` yml
---
# https://gist.github.com/EntropyWorks/a768b3bc4444146d56be81af05d73fed
# Original idea found at http://stackoverflow.com/a/39083724
#
#   ansible -i inventory.ini add-ssh-keys.yml
#
# 需要安装 sudo yum install -y sshpass bind-utils
#
- name: Store known hosts of 'all' the hosts in the inventory file
  hosts: localhost
  connection: local
  vars:
    ssh_known_hosts_command: "ssh-keyscan -T 10"
    ssh_known_hosts_file: "{{ lookup('env','HOME') + '/.ssh/known_hosts' }}"
    ssh_known_hosts: "{{ groups['all'] }}"


  tasks:
  - name: For each host, scan for its ssh public key
    shell: "ssh-keyscan {{ item }},`dig +short {{ item }}`"
    with_items: "{{ ssh_known_hosts }}"
    register: ssh_known_host_results
    ignore_errors: yes
    tags:
      - ssh

  - name: Remove the public key in the '{{ ssh_known_hosts_file }}'
    known_hosts:
      name: "{{ item.item }}"
      state: "absent"
      path: "{{ ssh_known_hosts_file }}"
    with_items: "{{ ssh_known_host_results.results }}"
    tags:
      - ssh

  - name: Add/update the public key in the '{{ ssh_known_hosts_file }}'
    known_hosts:
      name: "{{ item.item }}"
      key: "{{ item.stdout }}"
      state: "present"
      path: "{{ ssh_known_hosts_file }}"
    with_items: "{{ ssh_known_host_results.results }}"
    tags:
      - ssh

  - name: For each host, ssh-copy-id my ssh public keys to the host
    shell: "sshpass -p {{ ansible_ssh_pass }} ssh-copy-id {{ item }}"
    with_items: "{{ ssh_known_hosts }}"
    when: not (( ansible_ssh_pass is undefined ) or ( ansible_ssh_pass is none ) or ( ansible_ssh_pass | trim == ''))
    tags:
      - sshcopy

```

使用 ansible-playbook 在目标机执行 add-ssh-keys.yml 进行目标机免密. 结果中全部为绿色文字描述成功信息, 红色文字描述失败信息.

``` shell
## 如果 inventory 中写了 ansible_ssh_pass 主机变量使用下面命令
ansible-playbook -i example-hosts add-ssh-keys.yml
## 如果所有主机的登录密码一致, 使用下面命令. -k 表示询问登录用户的密码
ansible-playbook -k -i example-hosts add-ssh-keys.yml
```

### 第一个 ansible 命令

``` shell
## 使用 ansible ping 模块验证目标机是否可正常访问
## 下面命令中 all 是 ansible 内置变量. 代表 inventory 内所有主机.
## 还可以指定主机别名
ansible all -i example-hosts -m ping
```

### 第一个项目

保存下面内容到~/ansible-init.sh , 并修改为可i执行 chmod +x ansible-init.sh. 这个文件作为以后项目目录的"脚手架"脚本.

``` shell
#!/bin/bash

## 创建 inventory 文件
touch inventory
## 创建 playbook.yml
touch playbook.yml
## 主机变量目录. 文件名格式: host.yml. 内容格式: key: value
mkdir -p host_vars
## 组变量目录. 文件名格式: groupname.yml. 内容格式: key: value
mkdir -p group_vars
## 角色目录
mkdir -p roles
cd roles
## 创建 common 角色
ansible-galaxy init common
```

1. 建立项目目录

``` shell
## 进人项目目录, 假定 ~/example
~/ansible-init.sh 
```

2. 创建角色. 用角色来管理任务. 角色内部的目录结构都是约定好的, 不要修改.

``` shell
## ansible-playbook 命令会查找 playbook.yml(剧本文件, 文件名可以修改. 在 ansible-playbook 命令的参数指定剧本文件)所在目录的 roles 目录下的 $role_name/tasks/main.yml 作为任务的入口.
## 例子中假定角色名为 candy_console.
cd roles
ansible-galaxy init candy_console
```

3. 编写 tasks

   $PWD/roles/cand_console/tasks/main.yml . 

   内容简要说明: 格式为 {{ variable_name }} 是变量名. 变量来源: ansible 内置变量, ansible 收集的变量, 自定义变量. 自定义变量来源: 组变量在 $PWD/group_vars/ \*.yml ; 主机变量: $PWD/host_vars/\*.yml ; inventory 内定义变量: $PWD/example-hosts ; 角色变量: $PWD/roles/candy_console/vars/main.yml

``` yml
---
# tasks file for candy_console
## 一个任务对象以 - 开头
## name: 任务名称
- name: 创建根目录
  ## 任务使用的模块
  file:
    ## 模块参数
    ## "{{ app_home }}" 为变量. 
    path: "{{ app_home }}"
    ## 模块参数
    state: directory
  ## 任务的自定义标签. 可以看作是多个任务的横向切面. 需要合理规划 tag. 便于批量执行任务.
  tags:
    - deploy
    - init

- name: 部署目录和静态文件
  unarchive:
    src: "{{ static_files_zip }}"
    dest: "{{ app_home }}"
  tags:
    - deploy

- name: 部署动态文件-application.yml
  template:
    dest: "{{ service_application_file_path }}"
    src: application.yml.j2
  tags:
    - deploy

- name: 拉取镜像
  shell: chdir="{{ service_home }}" docker-compose pull
  tags:
    - deploy

- name: 启动
  shell: chdir="{{ service_home }}" if [ `docker ps|grep {{ service_name }}| wc -l` ]; then docker-compose up -d; fi
  tags:
    - start


- name: 停止
  shell: chdir="{{ service_home }}" docker-compose down
  tags:
    - stop

- name: 检测
  shell: "docker ps|grep {{ service_name }}"
  register: check_result
  tags:
    - check

- name: 打印检测结果
  debug: var=check_result
  tags:
    - check

```

4. 编写变量

   $PWD/roles/candy_console/vars/main.yml

``` yml
app_home: ~/app
static_files_zip: candy-console-service.zip
service_home: ~/app/candy-console-service
service_application_file_path: ~/app/candy-console-service/config/application.yml
service_name: candy-console-service

```

5. 打包静态文件和目录结构

   假定目标目录结构为: 

   ├── candy-console-service
   │   ├── config
   │   │   └── logback-spring.xml
   │   ├── docker-compose.yml
   │   └── logs

``` shell
## 打包静态文件(不再修改的文件)和目录结构
zip candy-console-service.zip candy-console-service/
## 拷贝到 roles/candy_console/files 目录中
## roles/$rolename/files/ 是 ansible 模块默认搜索的目录. 比如 "部署目录和静态文件" 任务中使用的 unarchive 模块, 仅需指定文件名即可, 无需指定目录.
```

6. 修改可变文件

   可变文件以 jinja2 文件为模板, 再通过 template 模块替换掉模板文件内部的变量后部署到目标机中.

   "{% %}" 是 jinja2 的代码块. jinja2 的[语法](http://jinja.pocoo.org/), [中文版](http://docs.jinkan.org/docs/jinja2/).

``` jin
## roles/$rolename/templates/ 目录是 template 模块搜索模板文件的目录.
## roles/candy_console/templates/application.yml.j2
# for framework
spring:
  application:
    name: candy-console-service
  # MQ配置
  rabbitmq:
    # [A] 集群节点地址
    addresses: {% for node in groups.rabbitmq %}{{ node }}:5672{% if not loop.last %},{% endif %} {% endfor %}
    # [A] 连接虚拟主机的用户名
    username: u1
    # [A] 连接虚拟主机的密码
    password: 111
    # [A] 连接虚拟主机的名称
    virtual-host: candy
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      #candy项目业务Oracle数据库
      oracle-datasource:
        # [A] 数据库连接配置
        url: jdbc:oracle:thin:@n1.oracle.nfcos:1521:nfcos
        # [A] 数据库用户名配置
        username: u1
        # [A] 数据库用户密码配置
        password: 111
        driver-class-name: oracle.jdbc.OracleDriver
        initial-size: 1
        min-idle: 1
        max-active: 3
        max-wait: 2000
        # 打开PSCache，并且指定每个连接上PSCache的大小(rowPrefetchCount)
        pool-prepared-statements: true
        #内存大小为rowSize*rowPrefetchCount,rowSize = col_1_size + col_2_size + ... + col_n_size
        max-open-prepared-statements: 20
        #连接初始化时执行sql保证第一次sql快速完成
        connection-init-sqls: SELECT 1 FROM DUAL
      #candy项目业务统一登录Oracle数据库  -主机房中控台数据库
      oracle-datasource-acc:
        # [A] 数据库连接配置
        url: jdbc:oracle:thin:@n1.oracle.nfcos:1521:nfcos
        # [A] 数据库用户名配置
        username: u1
        # [A] 数据库用户密码配置
        password: 111
        driver-class-name: oracle.jdbc.OracleDriver
        initial-size: 1
        min-idle: 1
        max-active: 3
        max-wait: 2000
        # 打开PSCache，并且指定每个连接上PSCache的大小(rowPrefetchCount)
        pool-prepared-statements: true
        #内存大小为rowSize*rowPrefetchCount,rowSize = col_1_size + col_2_size + ... + col_n_size
        max-open-prepared-statements: 20
        #连接初始化时执行sql保证第一次sql快速完成
        connection-init-sqls: SELECT 1 FROM DUAL
      #控制台统计数据库，需要新建
      mariadb-datasource-stat:
        # [A] 数据库连接配置
        url: jdbc:mariadb://n1.mariadb.nfcos:3306/nfcos_comm_console?useUnicode=true&characterEncoding=utf8&useSSL=false
        # [A] 数据库用户名配置
        username: u1
        # [A] 数据库用户密码配置
        password: 111
        driver-class-name: org.mariadb.jdbc.Driver
        initial-size: 5
        min-idle: 5
        max-active: 10
        max-wait: 2000
        validation-query: SELECT 1
        # 验证连接的有效性
        test-while-idle: true
        # 空闲连接回收的时间间隔，与test-while-idle一起使用，设置10分钟
        time-between-eviction-runs-millis: 600000
        test-on-borrow: true
        #连接初始化时执行sql保证第一次sql快速完成
        connection-init-sqls: SELECT 1
      #candy项目TSM任务数据库
      mariadb-datasource-tsm:
        # [A] 数据库连接配置
        url: jdbc:mariadb://n1.mariadb.nfcos:3306/nfcos_comm_tsm?useUnicode=true&characterEncoding=utf8&useSSL=false
        # [A] 数据库用户密码配置
        username: u1
        password: 111

        driver-class-name: org.mariadb.jdbc.Driver
        initial-size: 5
        min-idle: 5
        max-active: 10
        max-wait: 2000
        validation-query: SELECT 1
        # 验证连接的有效性
        test-while-idle: true
        # 空闲连接回收的时间间隔，与test-while-idle一起使用，设置10分钟
        time-between-eviction-runs-millis: 600000
        test-on-borrow: true
        #连接初始化时执行sql保证第一次sql快速完成
        connection-init-sqls: SELECT 1
#[A] Redis连接配置
redisson: '{
     "clusterServersConfig":{
        "idleConnectionTimeout":10000,
        "pingTimeout":1000,
        "connectTimeout":10000,
        "timeout":3000,
        "retryAttempts":3,
        "retryInterval":1500,
        "reconnectionTimeout":3000,
        "failedAttempts":3,
        "password":null,
        "subscriptionsPerConnection":5,
        "clientName":null,
        "loadBalancer":{
        "class":"org.redisson.connection.balancer.RoundRobinLoadBalancer"
        },
        "slaveSubscriptionConnectionMinimumIdleSize":1,
        "slaveSubscriptionConnectionPoolSize":50,
        "slaveConnectionMinimumIdleSize":32,
        "slaveConnectionPoolSize":64,
        "masterConnectionMinimumIdleSize":32,
        "masterConnectionPoolSize":64,
    "readMode":"SLAVE",
    "nodeAddresses":[{% for node in groups.redis %}"redis://{{ node }}:6981"{% if not loop.last %},{% endif %} {% endfor %}
    ],
    "scanInterval":1000
     },
     "threads":0,
     "nettyThreads": 0,
     "codec":{
        "class":"org.redisson.codec.JsonJacksonCodec"
     },
     "transportMode":"NIO"
  }'
mybatis:
  configuration:
    map-underscore-to-camel-case: true
# [A] 日志配置地址
logging:
  config: /usr/fmsh/configs/logback-spring.xml
# [A] 服务端口号
server:
  port: 8087

# for application


# 统计定时任务配置
scheduled:
  # [A] fpan统计15分钟一次
  fpanStatistics: 0 0,15,30,45 * * * ?
  # [A]dpan统计15分钟一次
  dpanStatistics: 0 10,25,40,55 * * * ?
  # [A] 业务统计15分钟一次
  businessStatistics: 0 10,25,40,55 * * * ?
  # [A]发卡、退卡、充值每小时一次
  statisticsByHour: 0 0 0-23 * * ?
  # [A] 发卡、退卡、充值每天一次
  statisticsByDay: 0 0 3 * * ?
  # [A]手机厂商卡类订单统计每天一次
  merchCardStatistics: 0 0 4 * * ?
  # [A]手机厂商充值统计每天一次
  merchTopupStatistics: 0 0 5 * * ?
  # [A]InApp业务统计每天一次
  inappStatistics: 0 0 6 * * ?
  # [A]FPAN可用个数告警每天一次
  fpanMonitor: 0 0 2 * * ?
  # [A]FPAN回收每天一次
  fpanRecycle: 0 0 2 * * ?
  # [A]业务告警
  bizMonitor: 0 10,25,40,55 * * * ?
  # [A]退款统计每天一次
  bizRefundStatistics: 0 0 2 * * ?
  #系统上线时间，用于初始统计
  systemInitTime: 2018-10-25 00:00:00

#仪表盘显示统计数据的时间范围
limitRange:
  # 按小时显示，推荐显示24小时内的统计数据
  hour: 24
  # 按天显示，推荐显示31天内的统计数据
  day: 31
  # 按周显示，推荐显示172天内的统计数据
  week: 172
  # 按小时显示，推荐显示360内的统计数据
  month: 360
  #主订单和支付查询按日期查询允许的最大时间范围
  limitedDate: 90
jwtusing:
  #jwt签名秘钥,与统一登录服务配置一致
  secret: fmsh
  #用户登录过期时间（单位：秒）
  expiration: 1800

```

7. 编写 playbook

   ansible 将多个任务组合为一个剧本. 通过 role 方式管理相关的任务.

``` yml
---
## hosts: 指定要执行剧本的目标机. 可以是 all, 组名, 目标机别名
## 名称来源: inventory 文件
- hosts: candy_console
  ## 收集目标机信息. 比较慢. bool 型.
  gather_facts: no
  ## 包含的角色. 等同于 include roles/$rolesname/tasks/main.yml
  roles:
    - role: candy_consol
```

8. 执行 playbook

``` shell
## ansible-playbook 执行剧本
## -i: inventory 文件.
## -t: tags. 多个tag, 用','分隔, 中间有空格的话需要用 "" 包起来.
## --check: 检查语法或模拟执行.
## tags 来源: roles/$rolename/tasks/main.yml 中的任务中的 tags 字符串数组.
## 部署
ansible-playbook -i example-hosts -t deploy playbook.yml

## 部署后启动
ansible-playbook -i example-hosts -t "deploy,start" playbook.yml
```

## ansible 模块

ansible 是通过模块实现具体的功能. 

 通过 "ansible-doc -s 模块名" 查看模块的文档.

``` shell
## 列出已安装模块
ansible-doc -l

## 查看模块的文档
ansible-doc -s 模块名
```



下面介绍下常用的模块. 参数中"[]" 括起来为默认值.

### command

ansible 默认的模块. 在 ansible 命令中不需要通过 -m 参数指定. 

command 命令不支持管道. 如果需要用到管道需要使用 shell 模块. 

``` shell
## 查看文档
ansible-doc -s command

## -a: 指定模块的参数. command 的参数就是 shell 命令.
## 比如: 查看当前登录用户. 通过在目标机执行 id 命令达成.
ansible all -i inventory -a "id"

## chdir: 执行任务前先改变工作目录
## 例子: 查看 /app 目录
ansible all -i inventory -a "chdir=/app ls -l"
```

### shell

支持管道. 可以替代 command 模块. 需要 -m shell 指明模块 

``` shell
## 查看文档
ansible-doc -s shell

ansible all -i inventory -m shell -a "chdir=/app ls | wc -l"
```

### ping

ping 模块. 测试连通性.

``` shell
## 查看文档
ansible-doc -s ping

ansible all -i inventory -m ping
```



### copy

拷贝文件模块

``` shell
## 查看文档
ansible-doc -s copy

## 参数
## src: 源文件路径. 
## content: 当用 content 替代 src 时, content 的值作为目标文件的内容.
## dest: 必选. 目标文件路径
## force: [yes]/no. 是否覆盖同名文件.
## backup: yes/no. 是否创建备份文件.
## mode: 目标文件权限. 例: 0777, 0600
## owner: 指定目标文件所属者
## group: 指定目标文件所属组
```

### file

文件管理模块.

``` shell
## 查看文档
ansible-doc -s file

## 参数
## path: 必选. 目标文件或目录的路径.
## mode: 文件权限. 例: 0777, 0600
## group: 文件所属组
## owner: 文件所属用户
## recurse: 递归修改文件属性, 仅对目录有效.
## state: absent: 删除; directory: 创建目录; [file]: 创建文件(不存在时才创建); hard: 创建硬链接; link: 创建软链接; touch: 文件不存在则创建文件, 存在则更新访问时间和修改时间.
## force: yes/[no]. 是否强制创建软链接.
## src: 仅在 state=link 或 state=hard 有效. 被链接的源路径
## dest: 仅在 state=link 或 state=hard 有效. 被链接的目标路径

```

### unarchive

拆包模块. 支持 .tar.gz, .zip, .xgz 格式.

``` shell
## 查看文档
ansible-doc -s unarchive

## 参数
## src: 源文件绝对路径. 当 remote_src=no (默认)时, 拷贝本地路径上的源文件到目标机再拆包. remote_src=yes 时, src 指目标机上的路径.
## dest: 目标绝对路径. 
## mode: 文件权限. 例: 0777, 0600
## group: 文件所属组
## owner: 文件所属用户
## creates: 一个文件或目录的绝对路径. 如果该路径的文件或目录已经存在, 则 *不* 执行该任务.
## keep_newer: yes/no. 是否替换目标路径中已经存在的比包中更新的文件
```

### template

模板模块. 模板文件类型为 jinja2. 后缀名一般是 .j2 .

``` shell
## 查看文档
ansible-doc -s template

## 参数
## src: 源文件路径. 
## dest: 目标文件路径.
## backup: yes/no . 是否创建备份文件.
## force: yes/no . 是否替换已存在文件.
## mode: 文件权限. 例: 0777, 0600
## group: 文件所属组
## owner: 文件所属用户
```



### service

服务管理模块

``` shell
## 查看文档
ansible-doc -s service

## 参数
## name: 服务名称
## enabled: yes/no. 是否开机启动. 
## state: start: 启动； stopped: 停止; restarted: 重启; reloaded: 重新加载
## arguments: 附加参数
## pattern: 定义一个查询模式. 如果通过 status 命令查看服务状态时, 没有响应, 那么 service 模块通过 ps 命令在进程中通过该模式查找. 如果找到, 则认为该服务还在运行.
## sleep: 如果 state 指定为 restarted. 那么, 在 stop 和 start 之间等待多少秒.
```





