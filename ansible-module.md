# Ansible 模块

查看模块列表: ansible -l
查看指定模块的详细帮助: ansible -s MODULE_NAME

## 常用模块

以下内容基于 ansible-doc v2.7.10

### ping

``` shell
## ping 所有主机
ansible all -m ping

## ping 指定主机
ansible 192.168.1.1 -m ping
```

### command

默认模块. 用于在远程主机执行命令.
缺点: 运行的命令中无法使用变量, 管道. 如果需要变量或模块, 请使用 shell 模块. 如果目标主机缺少 python 或关键依赖, 比如: python-simplejson, 使用 raw 模块.

``` shell
ansible-doc -s command
- name: 该任务的名称
  action: command
    chdir:  # 执行命令前先改变到工作目录
    creates: # 后指定一个文件名或者 glob 模式, 指定的文件存在时, 不执行命令
    removes: # 后指定一个文件名或者 glob 模式, 指定的文件不存在时, 不执行命令
    stdin: # 将命令的stdin直接设置为指定的值
    warn: yes or no # 是否显示警告, 默认 yes

## 例子:
## 查看所有机器的日期
ansible all -a "date -R"
## 在 /path/to/file 存在时, 不执行命令
ansible all -a "creates=/path/to/file ip a"
```

### shell

执行命令. 可以包含管道和变量.

```shell
ansible-doc -s shell
- name: 该任务的名称
  action: shell
    free_from: # 必须. 并没有 free_from 选项. 用执行的命令和参数代替.
    chdir:  # 执行命令前先改变到工作目录
    creates: # 后指定一个文件名或者 glob 模式, 指定的文件存在时, 不执行命令
    removes: # 后指定一个文件名或者 glob 模式, 指定的文件不存在时, 不执行命令
    stdin: # 将命令的stdin直接设置为指定的值
    warn: yes or no # 是否显示警告, 默认 yes
    executable: # 切换执行命令用的 shell. 必须使用命令的绝对路径.

## 例子:
## 更改 user 的密码
ansible all -m shell -a "echo new_password | passwd --stdin user-name"
```

### raw

执行一个低级且脏的SSH命令(Executes a low-down and dirty SSH command)

### script

将脚本复制到远程主机并执行之.

``` shell
ansible-doc -s script
- name: 该任务的名称
  action: script
    free_from: # 必须. 本地脚本路径.
    chdir:  # 执行脚本前先改变到工作目录
    creates: # 后指定一个文件名, 指定的文件存在时, 不执行命令
    removes: # 后指定一个文件名, 指定的文件不存在时, 不执行命令
    executable: # 切换执行命令用的 shell. 必须使用命令的绝对路径.
    decrypt: # This option controls the autodecryption of source files using vault.

## 例子, 本地脚本要存在
ansible all -m script -a '/root/test-script.sh'
```

### yum

使用 yum 管理目标机器上的软件包

``` shell
ansible-doc -s yum
- name: 使用 yum 管理目标机器上的软件包
  action: yum
    conf_file: # yum 的配置文件
    enablerepo: # 启用某个源. comma 分割多个源.
    disablerepo: # 禁用某个源. comma 分割多个源.
    disable_gpg_check: # 禁用 gpg 签名检测. 仅在 state 为 present 或 latest 下有效.
    list: # 用于 ad-hoc 下, 不能用于 playbook. 后跟 <package> 或者 installed or updates or available . 同 yum list <package>
    bugfix: # 如果设置为 yes 并且 state=latest, 那么仅安装更新标记为 bugfix 的包.
    state: # present or installed: 安装, 如果已经安装则不在执行安装; latest: 安装或更新到最新版; absent or removed: 卸载
    autoremove: # 如果为 yes 则卸载所有该包的依赖包(这些依赖包已经不再被其他包依赖了, "leaf" package)
    allow_downgrade: # True or False. 指定的包是否允许降级, 可能已安装了更高版本。任务可能会得到一组与要安装的指定包的完整列表不匹配的包(因为降级包与其他包之间的依赖关系可能导致对前面事务中的包的更改).
    name: # 包名 或 url 或本地路径(rpm), 支持版本(格式: name-1.0). 支持 comma(',') 分割多个名字. 当 state=latest 并且 name=* 表示 yum -y update.
    update_cache: # yes or no. 强制更新缓存, 在 state 为 present or latest 有效.

## 例子
## 列出某个包
ansible all -m yum -a "list=bind-utils"
## 列出所有安装的包
ansible all -m yum -a "list=installed"

## 更新所有包, 并更新缓存
ansible all -m yum -a "name=* state=latest update_cache=yes"
```

### group

``` shell
ansible-doc -s group

- name: 添加 or 移除组
  action: group
    name: # 必选. 组名.
    gid: # 可选, 设置组ID
    state: # present: 添加组; absent: 移除组
    system: # yes/no: 是否创建为系统组

```

### user

``` shell
ansible-doc -s user

- name: 管理用户: 添加, 删除, 修改密码
  action: user
    name: # 用户名
    password: # 密码, 必须是加密后的字符串.
    update_password: # always: 更新密码. on_create: 仅对新创建用户设置该密码
    uid: # 用户的UID
    state: # present: 添加用户, absent: 删除用户
    remove: # state=absent 时有效. yes/no: 是否删除用户的 home 目录. 等同于 userdel --remove user_name
    home: # 指定用户的 home 目录
    create_home: # 是否创建用户 home 目录. yes/no, 默认 yes.
    move_home: # yes: 且设置了 home 参数, 表示从以前的 home 目录移动到 home 参数指定目录
    group: # 用户组
    groups: # 用户附属组. 多个组用 comma 分割. 如果为 null or '~' or '', 则移除用户所有附属组.
    shell: # 设置用户的 shell . 默认 /bin/bash.
    system: # yes/no: 是否创建为系统用户. 默认为 no. 仅创建用户时有效. 不能修改现有用户.
    comment: # 注释

## centos7 生成加密的 passwd 格式密码方法
## makepassword 是一款生成指定长度随机密码的工具. yum install -y makepassword
pwd=$(python -c 'import crypt,getpass;pw="123456";print(crypt.crypt(pw))')
ansible all -m user -a 'name=test password="$pwd" update_password=always'
```

### copy

``` shell
ansible-doc -s copy

- name: 文件拷贝模块
  action: copy
    src: # 指定源文件路径. 如果拷贝的源是文件夹. 结尾为 '/' 表示仅拷贝文件夹下的文件, 不包含源目录. 结尾没有 '/' 表示拷贝源目录.
    content: # 当用 content 替代 src 时, content 的值作为目标文件的内容.
    dest: # 必选项. 指定目标文件路径
    mode: # 目标文件权限. 类似: 0777, 0600
    force: # yes/no: 是否覆盖同名文件, 默认 yes.
    backup: # yes/no: 在覆盖前是否备份
    owner: # 指定目标文件的所有者名称
    group: # 指定目标文件的所有组名称

```

### service

``` shell
ansible-doc -s service

- name: 服务管理模块
  action: service
    name: # 服务名称
    enabled: # 是否开机启动
    arguments: # 附加参数
    state: # started: 启动; stopped: 停止; restarted: 重启; reloaded: 重新加载
    pattern: # 定义一个模式，如果通过status指令来查看服务状态时，没有响应，它会通过ps命令在进程中根据该模式进行查找，如果匹配到，则认为该服务依然运行
    sleep: # 如果当前的服务状态为 restarted , 那么, 在 stop 和 start 之间等待多少秒. 这有助于解决行为不佳的init脚本，这些脚本在发出停止进程的信号后立即退出。

## 例子
## 如果匹配了 /usr/sbin/nginx 表示该服务已经启动, 则不再执行启动命令
ansible all -m service -a 'name=nginx pattern=/usr/sbin/nginx state=started'
```

### setup

``` shell
ansible-doc -s setup

- name: 获取目标主机参数信息, 例如: IP, CPU 数量, 网卡信息等
  action: setup
    fact_path: # 目标机上自定义信息目录. 默认: /etc/ansible/facts.d/ . 在目录下查找 *.fact 文件(ini 格式或 json 格式). 这些自定义信息会放到结果中的 "ansible_local" 对象中.
    filter: # 过滤获取哪些信息. 默认: *
    gather_subset: # 获取参数子集. 可能的值有: all | min | hardware | network | virtual | ohai | facter. 可以用 '!' 标记该子集不收集. 多个子集用 comma 分割. 默认: all
    gather_timeout: # 收集信息的超时时间, 单位: 秒
```

### cron

``` shell
ansible-doc -s cron

- name: 定时任务模块
  action: cron
    name: # 任务描述
    job: # 要执行的任务. 仅在 state=present 有效. 别名: value.
    state: # present: 创建任务; absent: 删除任务. 默认: present.
    disable: # yes/no: 是否注释掉该任务. 默认: no . 仅在 state=present 有效.
    backup: # yes/no: 修改前是否先备份. 默认: no
    cron_file: # 用这个指定的文件替换远程主机上 cron.d 目录下的用户的定时任务. 最后配合 user 参数, 指定用户.
    user: # 指定被修改定时任务的用户. 默认:  user.
    day: # 日. 范围: 1-31, 所有: *, 每2天: */2 等等. 默认: *
    hour: # 小时. 范围: 0-23, 所有: *, 每2小时: */2 等等. 默认: *
    minute: # 分钟.  范围: 0-59, 所有: *, 每2分钟: */2 等等. 默认: *
    month: # 月.  范围: 1-12, 所有: *, 每2月: */2 等等. 默认: *
    weekday: # 周几.  范围: 0-6(表示: 周日-周六), 所有: * , 等等. 默认: *
    special_time: # 特定时间的别名. 枚举值: reboot,yearly,annually,monthly,weekly,daily,hourly
    env: # yes/no: 是否设定 crontab 的新环境变量. 默认: no. yes: 此节中, name 为环境变量名, value 为环境变量的值. 见下方例子.
    insertafter: # 在 state=present 并且 env=yes 时, 环境变量的值将会插入到 insertafter 的环境变量尾部
    insertbefore: # 在 state=present 并且 env=yes 时, 环境变量的值将会插入到 insertafter 的环境变量尾部

## 例子: https://docs.ansible.com/ansible/latest/modules/cron_module.html
```

### file

``` shell
ansible-doc -s file

- name: 文件管理模块
  action: file
    path: # 文件或目录的路径, 必选.
    group: # 文件/目录的属组
    mode: # 文件/目录的权限
    owner: # 文件/目录的属主
    recurse: # 递归修改文件属性, 仅对目录有效.
    state: # absent: 删除 | directory: 创建目录 | file: 文件不存在时不创建 | hard: 创建硬链接 | link: 创建软链接 | touch: 文件不存在则创建, 存在则更新访问时间和修改时间. 默认: file
    force: # yes/no: 是否强制创建软链接, 默认: no
    src: # 仅在 state=link 或 state=hard 有效, 被链接的源路径
    dest: # 仅在 state=link 或 state=hard  有效, 链接的路径
```

### synchronize

``` shell
ansible-doc -s synchronize

- name: 文件同步模块, 类似 rsync
  action: synchronize
    src: # 必选, 源目录.
    dest: # 必选, 目标目录.
    archive: # yes/no: 是否启用归档, 默认: yes, 相当于同时开启 chechsum, recursive, links, perms, times, owner, gourp, -D


```
