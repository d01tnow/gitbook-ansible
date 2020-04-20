# Ansible Packages

## inventory

### manager

#### InventoryManager

管理 inventory 对象的类

``` python
from ansible.inventory.manager import InventoryManager
from ansible.parsing.dataloader import DataLoader
SRC = r'path_to_inventory'
# 创建
dl = DataLoader()
im = InventoryManager(dl, sources=SRC)
```

