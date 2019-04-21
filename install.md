# Install

## 在CentOS7 上安装

``` shell

## 最简单用 yum 安装
sudo yum install -y epel-release
sudo yum install -y ansible

## 用 pip3 安装

# 安装 python3
wget https://www.python.org/ftp/python/3.7.3/Python-3.7.3.tgz
tar -zxvf Python-3.7.3.tgz
cd Python-3.7.3.tgz
# 默认安装在 /usr/local/bin, 通过参数修改安装位置: --prefix=/usr/local/bin
./configure
make
sudo make install

## 修改 pip 源
mkdir ~/.pip
cat ~/.pip/pip.conf <<EOF
[global]
timeout=6000
index-url=https://mirrors.aliyun.com/pypi/simple
trusted-host=mirrors.aliyun.com
EOF
sudo `which pip3` install ansible

## 查看帮助目录, 输出类似 less , 可以使用 '/' 查找
ansible-doc -l
## 查看模块 shell 的帮助
ansible-doc -s shell

## 安装 sshpass 和 bind-utils, 后面免密用
sudo yum install -y sshpass bind-utils

## 创建秘钥, 一路回车
ssh-keygen -t rsa

## 编写 /etc/ansible/hosts
## 所有 hosts 免密
ansible-playbook -k add-ssh-keys.yml

## ansible 默认模块是 command, 但它不支持管道. 用支持管道的模块 shell 替代.
## 但是 ansible 依赖 python 环境, 如果目标机没有, 那么只能使用 raw 模块安装一下
ansible all -m raw -a "yum install -y python-simplejson libselinux-python"
```

## 配置

默认 hosts 在 /etc/ansible/hosts. 默认的配置文件在 /etc/ansible/ansible.cfg
配置文件优先级由高到低: $PWD/ansible.cfg -> $HOME/.ansible.cfg -> /etc/ansible/ansible.cfg

``` ini
[defaults]

#inventory      = /etc/ansible/hosts  ##定义Inventory
#library        = /usr/share/my_modules/ ##自定义lib库存放目录
#remote_tmp     = ~/.ansible/tmp      ##零时文件远程主机存放目录
#local_tmp      = ~/.ansible/tmp      ##零时文件本地存放目录
#forks          = 5                   ##默认开启的并发数
#poll_interval  = 15                  ##默认轮询时间间隔
#sudo_user      = root                ##默认sudo用户
#ask_sudo_pass = True                 ##是否需要sudo密码
#ask_pass      = True                 ##是否需要密码
#host_key_checking = False            ##首次连接是否检查key认证
#roles_path    = /etc/ansible/roles   ##默认下载的Roles存放的目录
#log_path = /var/log/ansible.log      ##执行日志存放目录
#module_name = command                ##默认执行的模块
#action_plugins     = /usr/share/ansible/plugins/action     ##action插件存放目录
#callback_plugins   = /usr/share/ansible/plugins/callback   ##callback插件存放目录
#connection_plugins = /usr/share/ansible/plugins/connection ##connection插件存放目录
#lookup_plugins     = /usr/share/ansible/plugins/lookup     ##lookup插件存放目录
#vars_plugins       = /usr/share/ansible/plugins/vars       ##vars插件存放目录
#filter_plugins     = /usr/share/ansible/plugins/filter     ##filter插件存放目录
#test_plugins       = /usr/share/ansible/plugins/test       ##test插件存放目录
#strategy_plugins   = /usr/share/ansible/plugins/strategy   ##strategy插件存放目录
#fact_caching = memory                                      ##getfact缓存的主机信息存放方式
#retry_files_enabled = False
#retry_files_save_path = ~/.ansible-retry                   ##错误重启文件存放目录

[privilege_escalation]
#become=True           ##是否sudo
#become_method=sudo    ##sudo方式
#become_user=root      ##sudo后变为root用户
#become_ask_pass=False ##sudo后是否验证密码

[ssh_connection]
#pipelining = False  ##管道加速功能，需配和requiretty使用方可生效

[selinux]
#libvirt_lxc_noseclabel = yes ##selinux配置
```
