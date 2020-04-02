# Ansible 命令行

命令格式: ansible <pattern_goes_here> -m <module_name> -a <argument>

重要的命令行参数

``` readme
-m MODULE_NAME: 要执行的模块, 默认 command. 该模块不支持管道. 使用支持管道的 shell 模块代替它.
-a MODULE_ARGS, --args=MODULE_ARGS: 模块的参数
-C, --check: 不做任何改变; 相反，试着预测一些可能发生的变化.
-l SUBSET, --limit=SUBSET: 进一步将所选主机限制为附加模式
--list-hosts: 显示匹配的 host 列表.
-f FORKS, --forks=FORKS: 并发连接主机数. 默认为 5 .
-u REMOTE_USER, --user=REMOTE_USER: 连接用的远程的用户名. 默认 root. ansible.cfg 中可以修改.
-k, --ask-pass: 提示输入连接登录密码.
--private-key=PRIVATE_KEY_FILE, --key-file=PRIVATE_KEY_FILE: 用指定的密钥文件验证连接
-s: sudo 运行. 已经不推荐( deprecated ). 推荐使用 -b, --become
-U SUDO_USER, --sudo-user=SUDO_USER: sudo 用户, 默认 root. sudo相关的已经不推荐, 都推荐使用 -b, --become
-b, --become: 以 become 模式执行操作.
--become-user=BECOME_USER: 权限提升( privilege escalation) 的目标用户. 默认提升为 root.
--become-method=BECOME_METHOD: 权限提升方法. 可选: sudo | su | pbrun | pfexec | doas | dzdo | ksu 等. 默认 sudo.
-K, --ask-become-pass: 当不是 NOPASSWD 模式下, 提示输入 BECOME_USER 的密码.
-t TREE, --tree=TREE: 日志输出到 TREE 目录中
--syntax-check: 仅做语法检测
-o,  --one-line: 结果压缩到一行输出.
```
