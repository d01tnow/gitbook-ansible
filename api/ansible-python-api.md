# ansible python api

使用 python api 进行 ansible 的二次开发



## 虚拟环境

基于 conda 构建虚拟环境.

``` shell
# 克隆 base 环境创建虚拟环境 ansible-env 
conda create -n ansible-env --clone base

# 如果想删除虚拟环境
# conda remove --name ansible-dev --all

# 激活虚拟环境
conda activate ansible-env

# deactivate 虚拟环境
# conda deactivate

# 查看已有的虚拟环境
conda env list

# 安装 ansible
# 由于 Ansible 不是 Conda 默认通道的一部分，因此 -c 用于从备用通道搜索和安装。
conda install -c conda-forge ansible
```

