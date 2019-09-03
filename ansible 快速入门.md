# ansible 使用入门

## ansible-init 工具

``` shell

# 在当前目录创建"proj0"
ansible-init.py -p proj0

# 在当前目录创建"proj1", 并且初始化 "role1"
ansible-init.py -p proj1 -r role1

cd proj1
# 进人项目目录后, 初始化 "role2"
ansible-init -r role2

```

## ansible 相关环境

环境要求:

- 编辑脚本主机: 只要有文本编辑器, ssh 客户端即可. 文本编辑器最好是支持语法高亮的. 推荐: VS Code, SublimeText, Atom等. 编辑后的脚本文件通过 XShell/XFtp, SecureCRT 传输到 ansible 主控机, 进行验证.
- ansible主控机: 执行 ansible 命令的机器. 需要安装 ansible. 做好免密. @俞建虎会初始化.
- ansible被控机: 执行 ansible 任务的机器.



## 编辑

### 编辑脚本的主机

推荐安装命令行工具 cmder(在 windows 下也可以使用 Linux 命令). 需要安装 python3 . 拷贝 ansible-init.py 到一个目录中(例子中使用 ansible_practice )

``` shell
# ansible_practice 目录下
# 通过"脚手架"工具创建项目,
python ./ansible-init.py -p first_project
# 进人项目目录, 通过"脚手架"工具创建第一个角色 example
cd first_project
python ../ansible-init.py -r example
# 然后用编辑器打开目录 first_project. 开始编辑.
```

[最佳实践参考1](https://blog.csdn.net/wn_hello/article/details/54730278)

### 脚本规范

#### 变量命名规范

变量名采用小写字母, "_", 数字的组合. 首字母必须是字母.

#### 变量的语法


- 变量名可用字符为字母, 数字和下划线. 推荐用下划线连接的小写字母做变量名.
- 在 ini-style 的 inventory 文件中使用 key=value 格式. 仅支持 boolean 和 string 类型的变量
- 在 yaml 格式的文件内必须使用 key: value 格式.
  - 在 group_vars 文件中使用 full key 名称: ansible_network_os: vos.
  - 在 playbooks 中使用 短格式 key 名称, 去掉 ansible 前缀: network_os: vos.

#### 常用的 magic 变量

ansible 已经定义的变量. [参考](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html)

- hostvars: 收集的所有主机的变量列表. 通过 hostsvars['$hostname']访问具体主机的变量. 比如: hostname=db.test.com 的 eth0 网卡的 ipv4 地址: hostvars['db.test.com'].ansible_eth0.ipv4.address
- groups: 是 inventory 文件中的所有组的列表. 比如: groups.redis 等同于 groups['redis'], 访问 redis 组. all 代表所有组.
- group_names: 当前执行的 task 的目标主机所在的组.
- inventory_hostname: inventory 中的主机别名. ansible_host 是 ansible 自主发现的主机名. 是通用的连接插件变量. ansible_ssh_host 是 ssh 连接插件的变量.
- inventory_hostname_short: inventory 中的主机别名的第一部分. e.g.: n1.test.com 返回 n1
- inentory_hostnames: 从 inventory 中匹配' host patterns '的主机名列表
- play_hosts: 当前 play 范围中可用的一组主机名.
- delegate_to: 是使用 'delegate_to' 代理的任务中的主机的 inventory 主机名.
- inventory_dir: 是 inventory 的目录
- inventory_file: 是 inventory 的包含路径的文件名
- role_path: 是当前 role 的目录名, 只有在 role 中才能使用.
- role_name: 当前角色的名字, 只有在 role 中才能使用.


#### 约定的名称

| 名称                        | 含义                                                         |
| --------------------------- | ------------------------------------------------------------ |
| app_home                    | 安装应用程序的父目录                                         |
| service_home                | 应用程序目录                                                 |
| service_docker_compose_file | 应用程序的 docker-compose.yml文件的绝对路径                  |
| service_name                | 用于检测容器是否存在: docker ps \| grep $service_name. 同 docker-compose.yml 内services 下的服务名. 约定与角色名相同. |
| service_docker_image        | 应用程序的镜像地址                                           |

#### 约定的后缀名

| 格式       | 含义          |
| ---------- | ------------- |
| _port      | 端口号        |
| _http_port | http 的端口号 |
| _tcp_port  | tcp 的端口号  |
| _username  | 用户名        |
| _passwd    | 密码          |
| _image     | 镜像          |

#### 约定的编写位置

| 类型                   | 位置                                                         | 说明                                                         |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| playbook级             | playbook.yml 中的 vars 下                                    | 针对某个应用场景                                             |
| 环境相关变量           | inventory 中                                                 | inventory 文件名: 开发环境: dev-inventory, 测试环境: test-inventory, 生产环境: prod-inventory |
| 组相关变量             | 少量时, 放到inventory 中. 大量时放到 group_vars/group_name.yml中 | 比如: 用于练习的变量 [all:vars]app_home="~/app/"             |
| 主机相关变量           | 少量时, 放到inventory 中. 大量时放到 host_vars/host_name.yml中 | ansible_host (即被控机的ip)通常放到 inventory 中.  比如: 同一服务但在各主机上不同的变量. 假设 redis 部署的3个主机的端口不一样, 就是主机级变量. 如果一样就写到服务级变量. |
| 服务级变量             | vars/main.yml 中                                             | 同一角色(服务)共用的变量. 比如: oracle 用户名, 密码等        |
| 默认的变量             | devaults/main.yml 中                                         | 优先级最低的, 通常不用修改的变量的默认值                     |
| ansible 收集的主机变量 |                                                              | gather_facts: yes/no 控制                                    |

#### 变量优先级

优先级由低到高排列(摘自: [Using Variables](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#scoping-variables))

> 1. command line values (eg “-u user”)
> 2. role defaults [[1\]](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#id15)
> 3. inventory file or script group vars [[2\]](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#id16)
> 4. inventory group_vars/all [[3\]](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#id17)
> 5. playbook group_vars/all [[3\]](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#id17)
> 6. inventory group_vars/* [[3\]](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#id17)
> 7. playbook group_vars/* [[3\]](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#id17)
> 8. inventory file or script host vars [[2\]](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#id16)
> 9. inventory host_vars/* [[3\]](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#id17)
> 10. playbook host_vars/* [[3\]](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#id17)
> 11. host facts / cached set_facts [[4\]](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#id18)
> 12. play vars
> 13. play vars_prompt
> 14. play vars_files
> 15. role vars (defined in role/vars/main.yml)
> 16. block vars (only for tasks in block)
> 17. task vars (only for the task)
> 18. include_vars
> 19. set_facts / registered vars
> 20. role (and include_role) params
> 21. include params
> 22. extra vars (always win precedence)

#### 约定的任务的 TAG

| tag    | 含义                                |
| ------ | ----------------------------------- |
| init   | 初始化. 角色的所有任务执行的第一步. |
| deploy | 部署任务                            |
| start  | 启动服务                            |
| stop   | 停止服务                            |
| check  | 检查                                |
| svn    | 上传文件到 svn                      |



### 常见的 Jinja2 模板语法

角色的 templates/ 目录下文件都是 *.j2 格式的 jinja2 文件. jinja2解析器会解析模板文件, 通过变量替换, 控制语句控制输出内容, 生成目标文件.

**注意:**

- 由于 jinja2 中变量的语法 "{{ 变量名 }}" 同 yaml 文件中字典的定义冲突. 如果 yaml 文件中的变量定义为 key: {{ 变量名 }}, 需要用双引号把 {{ }} 括起来. 即: key: "{{ 变量名 }}".
- {% %} 控制语句块后要加个换行. 并且, 一定要验证, 目标文件的格式是否符合要求. 原因是控制块引起缩进问题和换行问题. 目前未找到解决方法.

假设 inventory 文件内容:

``` ini

[redis]
n1.redis.nfcos ansible_host=192.168.43.101 redis_tcp_port=6981
n2.redis.nfcos ansible_host=192.168.43.93  redis_tcp_port=6982
n3.redis.nfcos ansible_host=192.168.43.141 redis_tcp_port=6983
```

vars/main.yml 内容:

``` yaml
core_server_redis_maxidle: "10"
```

templates/server.properties.j2 文件内容:

``` jinja2
# 变量表示方法: {{ 变量名 }}
redis.maxIdle={{ core_server_redis_maxidle }}

# 控制语句: {% 控制语句块 %} 比如: {% for ... %} {% endfor %}
# 根据 redis 组内的节点生成 spring redis 客户端的配置
# jinja2 模板会把未被变量或控制语句块包含的字符原样输出到目标文件中
spring.redis.cluster.nodes={% for node in groups.redis %}{{ hostvars[node].ansible_host }}:{{ hostvars[node].redis_tcp_port }}{% if not loop.last %},{% endif %} {% endfor %} # 一定要加换行. 千万千万!!!
# 此处为 {% for %} 语句的换行, 必需.
spring.redis.cluster.max-redirects=3
```

说明:

- core_server_redis_maxidle: 定义在 vars/main.yml 中的角色级变量;
- node 是自定义变量;

- groups 是 ansible 内置变量, 列表类型, 内容是 inventory 文件中的所有组;

- hostvars也是 ansible 内置变量, 列表类型, 用来访问"主机级"变量;

- ansible_host 是 inventory 中定义的"主机级"变量;

- redis_tcp_port 也是是 inventory 中定义的"主机级"变量;
- loop 是 jinja2 内置变量, 指的循环变量. last 是该变量的一个[属性](http://docs.jinkan.org/docs/jinja2/templates.html#for), 如果是最后一次迭代，为 True .
- not 是 jinja2 的逻辑判断表达式. 对其后的逻辑表达式取反.
- '.' 或 '[]' 用来获取对象的属性. 两者的区别是: obj[variable_name] 内可以使用变量访问对象的属性, 而 '.' 无法使用变量访问对象的属性, obj.variable_name 被转换为 obj['variable_name'].

生成的目标文件 server.properties 内容

``` ini
# 生成的目标文件内容
redis.maxIdle=10
spring.redis.cluster.nodes=192.168.43.101:6981, 192.168.43.93:6982, 192.168.43.141:6983
```



## 执行

从编辑主机传输 ansible 项目到 ansible 主控机.


### ansible 常用命令

在主控机上执行.

``` shell
# ansible 命令.
# 格式: ansible 执行任务的主机名 | 组名 | all -i inventory文件名 -m 模块名 -a "模块参数"
# ansible 默认模块 commond. 参数就是 shell 命令. 但是, command 模块不支持管道. 如果需要请使用 shell
# 比如: 查看当前用户
ansible all -i dev-inventory -a "id"
# 查看目录下的文件数, 需要用管道
ansible all -i dev-inventory -m shell -a "ls | wc -l"

# 用 ping 模块, 查看主控机和被控机的连通性
#
ansible all -i dev-inventory -m ping

```

### ansible-playbook 命令

``` shell
# ansible-playbook 命令
# 格式: ansible-playbook -i inventory文件名 -t "要执行的tags,逗号分隔" playbook文件名
# 检查 playbook 语法
ansible-playbook -i inventory playbook.yml --syntax-check
# 列出将要执行任务的主机
ansible-playbook -i inventory playbook.yml --list-hosts
# 列出将要执行的任务
ansible-playbook -i inventory playbook.yml --list-tasks
# 列出有效的 tags
ansible-playbook -i inventory playbook.yml --list-tags
# 只执行匹配 tag 的任务
ansible-playbook -i inventory -t "tag1,tag2" playbook.yml



```

## 验证

登录到 ansible 被控机, 查看生成的目标文件是否符合预期. 如果用到了 jinja2 模板的控制块, 通常需要注意 yaml 文件的格式问题(缩进, 换行).

### 例子

#### inventory 文件

``` ini
# 所有主机的变量
[all:vars]
app_home: "~/app"

# redis 组
[redis]
# 别名: n1.redis.nfcos. 自定义的.
# 主机级变量: ansible_host. ansible 用来和被控机通讯的.
# 主机级变量: redis_tcp_port. 自定义的.
n1.redis.nfcos ansible_host=192.168.43.101 redis_tcp_port=6981
n2.redis.nfcos ansible_host=192.168.43.93  redis_tcp_port=6982
n3.redis.nfcos ansible_host=192.168.43.141 redis_tcp_port=6983
```

#### 变量访问剧本

pb-debug.yml

``` yml
---

- hosts: redis
  gather_facts: no
  tasks:
  - name: 打印 groups 变量
    run_once: yes
    debug: var=groups
    tags:
    - dbg
    - only

  - name: 打印 group_names 变量
    run_once: yes
    debug: var=group_names
    tags:
    - dbg


  - name: 打印 groups.redis 变量
    run_once: yes
    debug: var=groups.redis
    tags:
    - dbg

  - name: 打印 hostvars 变量
    run_once: yes
    debug: var=hostvars
    tags:
    - dbg

  - name: 打印 hostvars 变量
    run_once: yes
    debug: var=hostvars
    tags:
    - dbg

  - name: 打印 hostvars['n1.redis.nfcos'] 变量
    run_once: yes
    debug: var=hostvars['n1.redis.nfcos']
    tags:
    - dbg

  - name: 打印 hostvars[groups.redis[0]] 变量
    run_once: yes
    debug: var=hostvars[groups.redis[0]]
    tags:
    - dbg

  - name: 打印 hostvars[groups.redis[0]].ansible_host 变量
    run_once: yes
    debug: var=hostvars[groups.redis[0]].ansible_host
    tags:
    - dbg

  - name: 打印 inventory_hostname 变量
    run_once: yes
    debug: var=inventory_hostname
    tags:
    - dbg

  - name: 打印 inventory_dir 变量
    run_once: yes
    debug: var=inventory_dir
    tags:
    - dbg

  - name: 循环打印 groups.redis 变量
    run_once: yes
    # {{ item }} 表示循环变量
    debug: var={{ item }}
    # loop 模块, 通过遍历给定的可遍历数据, 多次运行任务.
    # 类似 python 语法: for i in iterable : task(i)
    loop: "{{ groups.redis }}"
    tags:
    - dbg

  - name: 循环打印 循环索引 变量
    run_once: yes
    # {{ item }} 表示循环变量
    debug:
      msg: "{{ item }}'s index: {{ idx }}"
    # loop 模块, 通过遍历给定的可遍历数据, 多次运行任务.
    # 类似 python 语法: for i in iterable : task(i)
    loop: "{{ groups.redis }}"
    loop_control:
      # 自定义循环变量 idx
      index_var: idx
    tags:
    - dbg


```

使用方法:

``` shell
# 执行所有任务
ansible-playbook -i inventory pb-debug.yml

# 执行 tag 为 only 的任务
# 仅执行一个任务, 就把该任务打上 only 标签, 删除其他任务的 only 标签
ansible-playbook -i inventory pb-debug.yml -t only
#
```



## 附录

### Magic 变量

``` markdown

Magic

These variables cannot be set directly by the user; Ansible will always override them to reflect internal state.

ansible_check_mode
    Boolean that indicates if we are in check mode or not
ansible_dependent_role_names
    The names of the roles currently imported into the current play as dependencies of other plays
ansible_diff_mode
    Boolean that indicates if we are in diff mode or not
ansible_forks
    Integer reflecting the number of maximum forks available to this run
ansible_inventory_sources
    List of sources used as inventory
ansible_limit
    Contents of the --limit CLI option for the current execution of Ansible
ansible_loop
    A dictionary/map containing extended loop information when enabled via loop_control.extended
ansible_loop_var
    The name of the value provided to loop_control.loop_var. Added in 2.8
ansible_play_batch
    List of active hosts in the current play run limited by the serial, aka ‘batch’. Failed/Unreachable hosts are not considered ‘active’.
ansible_play_hosts
    The same as ansible_play_batch
ansible_play_hosts_all
    List of all the hosts that were targeted by the play
ansible_play_role_names
    The names of the roles currently imported into the current play. This list does not contain the role names that are implicitly included via dependencies.
ansible_playbook_python
    The path to the python interpreter being used by Ansible on the controller
ansible_role_names
    The names of the roles currently imported into the current play, or roles referenced as dependencies of the roles imported into the current play.
ansible_run_tags
    Contents of the --tags CLI option, which specifies which tags will be included for the current run.
ansible_search_path
    Current search path for action plugins and lookups, i.e where we search for relative paths when you do template: src=myfile
ansible_skip_tags
    Contents of the --skip_tags CLI option, which specifies which tags will be skipped for the current run.
ansible_verbosity
    Current verbosity setting for Ansible
ansible_version
    Dictionary/map that contains information about the current running version of ansible, it has the following keys: full, major, minor, revision and string.
group_names
    List of groups the current host is part of
groups
    A dictionary/map with all the groups in inventory and each group has the list of hosts that belong to it
hostvars
    A dictionary/map with all the hosts in inventory and variables assigned to them
inventory_hostname
    The inventory name for the ‘current’ host being iterated over in the play
inventory_hostname_short
    The short version of inventory_hostname
inventory_dir
    The directory of the inventory source in which the inventory_hostname was first defined
inventory_file
    The file name of the inventory source in which the inventory_hostname was first defined
omit
    Special variable that allows you to ‘omit’ an option in a task, i.e - user: name=bob home={{ bobs_home|default(omit) }}
play_hosts
    Deprecated, the same as ansible_play_batch
ansible_play_name
    The name of the currently executed play. Added in 2.8.
playbook_dir
    The path to the directory of the playbook that was passed to the ansible-playbook command line.
role_name
    The name of the currently executed role
role_names
    Deprecated, the same as ansible_play_role_names
role_path
    The path to the dir of the currently running role

Facts

These are variables that contain information pertinent to the current host (inventory_hostname). They are only available if gathered first.

ansible_facts
    Contains any facts gathered or cached for the inventory_hostname Facts are normally gathered by the setup module automatically in a play, but any module can return facts.
ansible_local
    Contains any ‘local facts’ gathered or cached for the inventory_hostname. The keys available depend on the custom facts created. See the setup module for more details.

Connection variables

Connection variables are normally used to set the specifics on how to execute actions on a target. Most of them correspond to connection plugins, but not all are specific to them; other plugins like shell, terminal and become are normally involved. Only the common ones are described as each connection/become/shell/etc plugin can define its own overrides and specific variables. See Controlling how Ansible behaves: precedence rules for how connection variables interact with configuration settings, command-line options, and playbook keywords.

ansible_become_user
    The user Ansible ‘becomes’ after using privilege escalation. This must be available to the ‘login user’.
ansible_connection
    The connection plugin actually used for the task on the target host.
ansible_host
    The ip/name of the target host to use instead of inventory_hostname.
ansible_python_interpreter
    The path to the Python executable Ansible should use on the target host.
ansible_user
    The user Ansible ‘logs in’ as.


```

