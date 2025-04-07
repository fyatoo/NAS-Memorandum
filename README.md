# NAS-Memorandum
记录折腾Ubuntu用作NAS的相关操作
## 准备工作
### 安装Docker设置断点续传
文档
- https://docs.docker.com/engine/install/ubuntu/

设置断点续传
```bash
nano /etc/docker/daemon.json
```
添加
```bash
{  "features": { "containerd-snapshotter": true } }
```
重启docker

```bash
systemctl restart docker
```
### 设置自动登录
每隔10分钟执行一次自动登录的python代码，并保存log
```bash
crontab -e
```
添加
```bash
*/10 * * * * python3 /home/[user]/autologin/login.py >> /home/[user]/autologin/crontab.log 2>&1
```
### 设置开机自动挂载SMB
创建被挂载的文件夹
```bash
sudo mkdir /mnt/Important
sudo mkdir /mnt/Trivial
```
安装cifs-utils
```bash
sudo apt update
sudo apt install cifs-utils
```
创建credentials文件
```bash
nano /home/[user]/.smbcreds
```
输入
```bash
username=...
password=...
```
设置文件权限为仅所有者可读
```bash
chmod 600 /home/[user]/.smbcreds
```
编辑`/etc/fstab`
```bash
nano /etc/fstab
```
添加
```bash
//192.168.1.130/Important /mnt/Important cifs credentials=/home/[user]/.smbcreds,iocharset=utf8,file_mode=0700,dir_mode=0700,noperm,_netdev,uid=1000,gid=1000 0 0

//192.168.1.130/Trivial /mnt/Trivial cifs credentials=/home/[user]/.smbcreds,iocharset=utf8,file_mode=0700,dir_mode=0700,noperm,_netdev,uid=1000,gid=1000 0 0
```
测试
```bash
sudo mount -a
```
## 各类任务
### 安装与配置Rclone
文档
- https://rclone.org/install/
- https://rclone.org/docs/
- https://rclone.org/flags/

配置Rclone（指定配置文件保存位置）
```bash
rclone config --config /home/[user]/rclone/rclone.conf
```
调整该配置权限，仅所有者读写
```bash
chmod 600 /home/[user]/rclone/rclone.conf
```
### Rclone挂载webdav
创建目标挂载位置
```bash
mkdir /mnt/pan
sudo chown -R [user]:[user] /mnt/pan
```
在Rclone中添加一个remote，名字为pan，跟随指引输入相关配置
```bash
rclone config --config /home/[user]/rclone/rclone.conf
```
使用systemd服务实现开机自动挂载
- 创建配置文件
    ```bash
    sudo nano /etc/systemd/system/rclone-mount.service
    ```
- 编写
    ```bash
    [Unit]
    Description=Rclone Mount Pan
    After=network.target

    [Service]
    Type=simple
    User=[user]
    ExecStart=rclone --config /home/[user]/rclone/rclone.conf mount pan:/ /mnt/pan --vfs-cache-mode off --cache-dir /mnt/Trivial/rclone_cache
    Restart=on-failure
    RestartSec=5s

    [Install]
    WantedBy=multi-user.target
    ```
- 应用服务
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable rclone-mount.service
    sudo systemctl start rclone-mount.service
    ```
### Duplicacy文件备份
将`/mnt/Important`中的数据用duplicacy备份到`/mnt/pan/duplicacy_repo`中
文档
- https://github.com/gilbertchen/duplicacy/wiki

安装duplicacy
```bash
cd ~
mkdir duplicacy
cd duplicacy
wget https://github.com/gilbertchen/duplicacy/releases/download/v3.2.4/duplicacy_linux_x64_3.2.4
chmod 700 duplicacy_linux_x64_3.2.4
mv duplicacy_linux_x64_3.2.4 duplicacy
```
测试是否安装成功
```
/home/[user]/duplicacy/duplicacy
```
初始化
```bash
cd /mnt/Important
/home/[user]/duplicacy/duplicacy init -encrypt -erasure-coding 5:2  ImportantBackup /mnt/pan/duplicacy_repo
```
备份
```bash
cd /mnt/Important
/home/[user]/duplicacy/duplicacy backup -stats -threads 1 -limit-rate 10000
```
维护
```bash
cd /mnt/Important
/home/[user]/duplicacy/duplicacy check -a -fossils -resurrect -threads 1 -stats -rewrite [-files -chunks]
/home/[user]/duplicacy/duplicacy prune -a -threads 1 -keep 180:360 -keep 30:180 -keep 7:30 -keep 1:7
/home/[user]/duplicacy/duplicacy list -a
```
自动运行
保存密码
```bash
duplicacy set -key password -value "..."
```
```bash
crontab -e
```
添加
```bash
0 */6 * * * cd /mnt/Important && /home/[user]/duplicacy/duplicacy backup -stats -threads 1 -limit-rate 10000 >> /home/[user]/duplicacy/backup.log 2>&1

0 1 * * * cd /mnt/Important && /home/[user]/duplicacy/duplicacy check -a -fossils -resurrect -threads 1 -stats -rewrite >> /home/[user]/duplicacy/check.log 2>&1

0 0 * * * cd /mnt/Important && /home/[user]/duplicacy/duplicacy prune -a -threads 1 -keep 180:360 -keep 30:180 -keep 7:30 -keep 1:7 >> /home/[user]/duplicacy/prune.log 2>&1
```
### 安装Alist
文档
- https://alist.nn.ci/zh/guide/
- https://alist.nn.ci/zh/guide/install/docker.html

操作
- 进入web页面
- 设置两步验证
- 添加普通用户
- 挂载网盘
- 添加Crypt驱动
- 账户设置启动支持webdav

### Rclone数据备份到网盘
在Rclone中添加Alist的remote，名字为alist，跟随指引输入相关配置
```bash
rclone config --config /home/[user]/rclone/rclone.conf
```
使用`rclone sync`同步数据到网盘中
```bash
rclone sync -P --bwlimit 10m /mnt/Important alist:/Crypt --checkers 1 --transfers 1
```
自动运行
```bash
mkdir /home/[user]/rclone_alist
crontab -e
```
添加
```bash
0 5 * * * rclone --config /home/[user]/rclone/rclone.conf sync -P --bwlimit 10m /mnt/Important alist:/Crypt --checkers 1 --transfers 1 --webdav-pacer-min-sleep 100ms --webdav-encoding "" --timeout 120m --exclude=/.duplicacy/** --no-update-dir-modtime --no-update-modtime --fast-list >> /home/[user]/rclone_alist/sync.log 2>&1
```
### nextcloud-all-in-one局域网部署
文档
- https://github.com/nextcloud/all-in-one/discussions/5439
- https://github.com/nextcloud/all-in-one/blob/main/local-instance.md


### immich
文档
- https://github.com/immich-app/immich/issues/6616
- https://wiki.slarker.me/unraid/immich_ai_model.html
- https://huggingface.co/immich-app/XLM-Roberta-Large-Vit-B-16Plus/tree/main

安装git-lfs
- https://git-lfs.com
```bash
sudo apt install git-lfs 
```
下载模型
```bash
cd /var/lib/docker/volumes/immich_model-cache/_data/clip
git lfs install
git clone https://huggingface.co/immich-app/XLM-Roberta-Large-Vit-B-16Plus
```

### 利用Rclone和Alist将百度网盘中所有图片保存到本地
文档
- https://rclone.org/filtering/
- https://immich.app/docs/features/supported-formats
- https://rclone.org/commands/rclone_sync/


```bash
nano /home/[user]/rclone/include-file.txt
```
填入
```
*.3fr
*.ari
*.arw
*.cap
*.cin
*.cr2
*.cr3
*.crw
*.dcr
*.dng
*.erf
*.fff
*.iiq
*.k25
*.kdc
*.mrw
*.nef
*.nrw
*.orf
*.ori
*.pef
*.psd
*.raf
*.raw
*.rw2
*.rwl
*.sr2
*.srf
*.srw
*.x3f
*.avif
*.bmp
*.gif
*.heic
*.heif
*.hif
*.insp
*.jp2
*.jpe
*.jpeg
*.jpg
*.jxl
*.png
*.svg
*.tif
*.tiff
*.webp
*.3gp
*.3gpp
*.avi
*.flv
*.insv
*.m2t
*.m2ts
*.m4v
*.mkv
*.mov
*.mp4
*.mpe
*.mpeg
*.mpg
*.mts
*.vob
*.webm
*.wmv
*.xmp
```
执行
```
rclone --config /home/[user]/rclone/rclone.conf sync --progress --dry-run alist:/cloud /mnt/Important --checkers 1 --transfers 1 --include-from /home/[user]/rclone/include-file.txt --webdav-pacer-min-sleep 100ms --webdav-encoding "Asterisk,BackQuote,BackSlash,Colon,CrLf,Ctl,Del,Dollar,Dot,DoubleQuote,Exclamation,Hash,InvalidUtf8,LeftCrLfHtVt,LeftPeriod,LeftSpace,LeftTilde,LtGt,None,Percent,Pipe,Question,RightCrLfHtVt,RightPeriod,RightSpace,Semicolon,SingleQuote,Slash,SquareBracket" --local-unicode-normalization
```

### Rclone文件名特殊符号问题
文档
- https://www.reddit.com/r/rclone/comments/jeigft/it_seems_like_rclone_cant_handle_a_filename_with/
- https://rclone.org/local/#filenames
- https://rclone.org/onedrive/#restricted-filename-characters
- https://rclone.org/overview/#encoding
- https://rclone.org/local/#local-unicode-normalization


在命令`rclone sync`中添加flag
```
--webdav-encoding "Asterisk,BackQuote,BackSlash,Colon,CrLf,Ctl,Del,Dollar,Dot,DoubleQuote,Exclamation,Hash,InvalidUtf8,LeftCrLfHtVt,LeftPeriod,LeftSpace,LeftTilde,LtGt,None,Percent,Pipe,Question,RightCrLfHtVt,RightPeriod,RightSpace,Semicolon,SingleQuote,Slash,SquareBracket" --local-unicode-normalization
```
上述不对
仅需添加
```
--webdav-encoding ""
```

### Rclone避免频繁访问
在命令`rclone sync`中添加flag
```
--webdav-pacer-min-sleep 100ms
```

### jellyfin添加中文字体
将windows的`C:\Windows\Fonts\simhei.ttf`复制到jellyfin的某个目录下
在设置里设置启用备用字体

### fix back101v1.1
- `rclone-mount.service`中添加
    - `--vfs-cache-max-size 15G`

### 搜索并提取iso中的文件
```bash
7z l my.iso | grep '*'
7z e my.iso dir/to_be_extracted.txt -o/home/extracted
```


### KopiaCli

文档
- https://kopia.io/docs/getting-started/#kopia-cli
- https://kopia.io/docs/reference/command-line/


安装
```bash
mkdir /home/[user]/kopia-cli
cd /home/[user]/kopia-cli
wget https://github.com/kopia/kopia/releases/download/v0.19.0/kopia-0.19.0-linux-x64.tar.gz
tar -xzvf kopia-0.19.0-linux-x64.tar.gz
cd kopia-0.19.0-linux-x64

sudo cp kopia /usr/bin/
sudo chown root:root /usr/bin/kopia
sudo chmod 755 /usr/bin/kopia
```

创建仓库
```bash
kopia repository create webdav --url=... --webdav-password=... --webdav-username=...
```
设置缓存位置
```bash
kopia cache set --cache-directory=/mnt/Trivial/kopia_cache
```
连接仓库
```bash
kopia repository connect webdav --url=... --webdav-password=... --webdav-username=...
```
创建快照
```bash
kopia snapshot create /mnt/Important
```
查看已有快照
```bash
kopia snapshot list /mnt/Important
```
检查快照
```bash
kopia snapshot verify --verify-files-percent=5.0 
```