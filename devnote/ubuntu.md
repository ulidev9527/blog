
#### 开启 root 登陆
```shell
sudo sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```
#### 免密登陆
```shell
ssh-keygen -f /root/.ssh/auto_login -P ''
export IP="10.223.223.1"
export SSHPASS=123123
for HOST in $IP;do
sshpass -e ssh-copy-id -o StrictHostKeyChecking=no $HOST
done

# 这段脚本的作用是在一台机器上安装sshpass工具，并通过sshpass自动将本机的SSH公钥复制到多个远程主机上，以实现无需手动输入密码的SSH登录。
# 
# 具体解释如下：
# 
# 1. `apt install -y sshpass` 或 `yum install -y sshpass`：通过包管理器（apt或yum）安装sshpass工具，使得后续可以使用sshpass命令。
# 
# 2. `ssh-keygen -f /root/.ssh/auto_login -P ''`：生成SSH密钥对。该命令会在/root/.ssh目录下生成私钥文件id_rsa和公钥文件id_rsa.pub，同时不设置密码（即-P参数后面为空），方便后续通过ssh-copy-id命令自动复制公钥。
# 
# 3. `export IP="10.223.1.2 192.168.1.32 192.168.1.33 192.168.1.34 192.168.1.35"`：设置一个包含多个远程主机IP地址的环境变量IP，用空格分隔开，表示要将SSH公钥复制到这些远程主机上。
# 
# 4. `export SSHPASS=123123`：设置环境变量SSHPASS，将sshpass所需的SSH密码（在这里是"123123"）赋值给它，这样sshpass命令可以自动使用这个密码进行登录。
# 
# 5. `for HOST in $IP;do`：遍历环境变量IP中的每个IP地址，并将当前IP地址赋值给变量HOST。
# 
# 6. `sshpass -e ssh-copy-id -o StrictHostKeyChecking=no $HOST`：使用sshpass工具复制本机的SSH公钥到远程主机。其中，-e选项表示使用环境变量中的密码（即SSHPASS）进行登录，-o StrictHostKeyChecking=no选项表示连接时不检查远程主机的公钥，以避免交互式确认。
# 
# 通过这段脚本，可以方便地将本机的SSH公钥复制到多个远程主机上，实现无需手动输入密码的SSH登录。

```
#### 扩容
```shell
# 扩容
lvextend -L +10G /dev/mapper/ubuntu--vg-ubuntu--lv # 添加或者减少容量
lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv # 添加全部
resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv # 刷新容量
df -h # 查看

# 新加盘扩容
# https://blog.pc530.com/index.php/archives/94/
fdisk /dev/sda
m 回车 n 回车 回车 回车 回车 w 回车
pvcreate  /dev/sda4
vgextend ubuntu-vg /dev/sda4
lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv 
df -h # 查看
```
#### 查看文件夹大小
- `du -h`
#### 拉去三分镜像上传到harbor
- `docker pull 镜像地址`
#### 读取输入
```shell
read -p "你要输入什么：" name
```

#### 查看解析
- https://sites.ipaddress.com/url
- https://sites.ipaddress.com/raw.githubusercontent.com 示例

#### dock儿