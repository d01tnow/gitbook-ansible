# ansible-variable

## 变量语法

- 变量名可用字符为字母, 数字和下划线. 推荐用下划线连接的小写字母做变量名.
- 在 ini-style 的 inventory 文件中使用 key=value 格式. 仅支持 boolean 和 string 类型的变量
- 在 yaml 格式的文件内必须使用 key: value 格式.
  - 在 group_vars 文件中使用 full key 名称: ansible_network_os: vos.
  - 在 playbooks 中使用 短格式 key 名称, 去掉 ansible 前缀: network_os: vos.

### ini 转换到 yaml

- 主机变量

``` ini
[atlanta]
host1 http_port=80 maxRequests=800
host2 http_port=90 maxRequests=900
```

``` yml
atlanta:
  host1:
    http_port: 80
    maxRequests: 800
  host2:
    http_port: 90
    maxRequests: 900
```

- 组变量

``` ini
[atlanta]
host1
host2
[atlanta:vars]
ntp_server=ntp.test.com
proxy=proxy.test.com
```

``` yml
atlanta:
  hosts:
    host1:
    host2:
  vars:
    ntp_server: ntp.test.com
    proxy: proxy.test.com
```

### host_vars 或 group_vars 目录下的文件格式

变量可以放到 inventory 文件中, 也可以分散存储到约定的目录下. 变量过多时, 推荐将变量分散存储到约定目录下文件中. ansible 或 ansible-playbook 命令的 --inventory, -i 参数可以指定目录, inventory 插件会扫描该目录下的所有文件作为 inventory(按名称排序). 经验证, 各文件中的同名变量, 仅最后加载的有效. 指定目录时, 按文件名排序加载. 多文件时, 按参数先后顺序排序.

约定:

- 主机变量目录结构: host_vars/$host_name .

- 组变量目录结构: group_vars/$group_name.

- 角色变量目录结构: roles/$role/defaults/main.yml 或 roles/$role/vars/main.yml

- 文件后缀是 .yml 或 .yaml 或 .json .

- group_vars 或 host_vars 目录可以存在于 playbook 所在目录或 inventory 目录. 如果同时存在于 playbook 所在目录和 inventory 目录, playbook 所在目录中的变量覆盖 inventory 目录中的变量.

- ansible-playbook 默认查找当前工作目录下的 group_vars 和 host_vars. 其他命令, 默认情况下仅查看 inventory 目录下的 group_vars 和 host_vars. 可以通过命令行指定参数 --playbook-dir 改变查找变量的目录.

- yaml 文件格式 . 直接是 key: value. 不用写 vars 关键字.

  ``` yaml
  ---
  ntp_server: acme.test.org
  db: db.test.org
  ```

## 常用的 magic 变量

[参考](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html)

- hostvars: 收集的所有主机的变量列表. 通过 hostsvars['$hostname']访问具体主机的变量. 比如: hostname=db.test.com 的 eth0 网卡的 ipv4 地址: hostvars['db.test.com'].ansible_eth0.ipv4.address
- groups: 是 inventory 文件中的所有组的列表
- group_names: 当前执行的 task 的目标主机所在的组.
- inventory_hostname: inventory 中的主机别名. ansible_hostname 是 ansible 自主发现的主机名.
- inventory_hostname_short: inventory 中的主机别名的第一部分. e.g.: n1.test.com 返回 n1
- inentory_hostnames: 从 inventory 中匹配' host patterns '的主机名列表
- play_hosts: 当前 play 范围中可用的一组主机名.
- delegate_to: 是使用 'delegate_to' 代理的任务中的主机的 inventory 主机名.
- inventory_dir: 是 inventory 的目录
- inventory_file: 是 inventory 的包含路径的文件名
- role_path: 是当前 role 的目录名, 只有在 role 中才能使用.
