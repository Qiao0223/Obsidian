# 1. Vmare手动挂载文件夹
```
sudo mkdir -p /mnt/hgfs
sudo vmhgfs-fuse .host:/shared /mnt/hgfs -o allow_other
```
# 2. code2prompt

```
code2prompt . -O prompt.txt --hidden --full-directory-tree
```