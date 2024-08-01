# Serv00-PM2-Autorun

本项目借鉴了kuoihao仓库Serv00_Auto_Run的思路利用自动工作流进行远程运行脚本

原库地址https://github.com/kuoihao/Serv00_Auto_Run


# 功能
旨在通过判断Web服务是否运行，否则通过远程方式登录Serv00的ssh运行脚本以恢复PM2快照 
本项目分三种方式来解决

## 方法一：利用vps远程监控并保活serv00中的PM2

### Serv00服务器端
#### 在Serv00中编写恢复快照脚本
```
cd ~
touch run.sh
chmod +x run.sh


##写入命令（或者自己复制进去）：
cat << 'EOF' > run.sh
#!/bin/sh
~/.npm-global/bin/pm2 kill
/home/dino/.npm-global/bin/pm2 resurrect >/dev/null 2>&1
sleep 10
/home/dino/.npm-global/bin/pm2 restart all
~/.npm-global/bin/pm2 save
EOF
```
#### -Serv00中生成密钥对
```
ssh-keygen -t rsa -b 4096 -C "dino@milaone.app"
```
#### -将公钥并添加到authorized_keys中
    
```
cat ~/.ssh/id_rsa.pub | tee -a ~/.ssh/authorized_keys
```
#### -打印私钥内容，复制私钥全部内容
```
cat ~/.ssh/id_rsa
```
复制私钥内容作为SSH_PRIVATE_KEY的值,需要包括全部内容
```
-----BEGIN OPENSSH PRIVATE KEY-----
xxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxx
-----END OPENSSH PRIVATE KEY-----
```

- -----BEGIN OPENSSH PRIVATE KEY-----
- -----END OPENSSH PRIVATE KEY-----

这两行也要包含进来
### VPS端
#### -配置私钥
```
cd ~
nano .ssh/serv00key
#把私钥中内容粘贴进来，保存退出
chmod 600 .ssh/serv00key
```
#### -远程探测、PM2恢复快照脚本

```
cd ~
touch keep.sh
chmod +x keep.sh

```

keep.sh的内容，直接复制粘贴
```
#!/bin/bash

# 目标URL
URL="https://memos.milaone.app"

# 远程运行的脚本命令
RUN_SCRIPT="ssh -i .ssh/serv00key dino@s4.serv00.com '/home/dino/run.sh'"

# 检查服务是否运行
curl --head --silent --fail $URL > /dev/null
SERVICE_STATUS=$?

if [ $SERVICE_STATUS -eq 0 ]; then
    echo "Service is running."
else
    echo "Service is not running. Starting the service..."
    $RUN_SCRIPT
fi

```
#### -设置Cron计划任务执行脚本
```
sudo crontab -e
#加入下面任务，keep.sh文件完整路径
*/2 * * * * /home/ubuntu/keep.sh >/dev/null 2>&1
```
---
---

## 方法二：利用openwrt远程check https://memos.milaone.app 的运行状态，出错就ssh登录Serv00的ssh运行脚本
整体思路跟方法一是一样的，只是openwrt中的ssh是dropbear提供的，它不能使用serv00生成的证书
其实说这个方法，我就是为了说一下openwrt下的ssh是特殊的，大家注意一下
#### -openwrt中生成密钥对
```
dropbearkey -f ~/.ssh/id_dropbear -t rsa -s 2048
```
默认私钥就已经保存到了/root/.ssh/id_dropbear中了，后面我们直接使用即可

显示公钥就出现在屏幕中，我们复制下来，

#### -Serv00中添加公钥
编辑~/.ssh/authorized_keys文件，粘贴在文件末尾即可

#### 其他部分同方法一的脚本一样，只是注意对应相应证书文件、脚本文件的目录即可


## 利用github的Actions中自动工作流脚本每5分钟check一下 https://memos.milaone.app 的运行状态，出错就ssh登录运行脚本