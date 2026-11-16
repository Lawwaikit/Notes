~/.gitconfig
```ini
[http]
	proxy = http://127.0.0.1:7892
[https]
	proxy = http://127.0.0.1:7892
```


生成ssh key
```bash
ssh-keygen -t ed25519 -C "Gitee SSH Key"  # -C是注释

# 提示Enter file的时候指定 文件位置 和 文件名
Enter file in which to save the key (C:\Users\Lawwaikit/.ssh/id_ed25519):
```


