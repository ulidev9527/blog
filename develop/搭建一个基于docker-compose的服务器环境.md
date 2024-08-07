# 搭建一个基于docker-compose的服务器环境

工作中巧合下重拾服务器开发，所以觉得稍微细致的准备一下。  
考虑到部署会在内网、外网和、虚拟机等几个环境下使用，`docker`是最好的选择。  
调研阶段尝试过`k8s`，在性能消耗和学习成本下直接抛弃了。  

## 核心内容
- 容器管理 [Portainer](https://www.portainer.io) 安装使用
- 代理管理 [nginx proxy manager](https://nginxproxymanager.com) 安装使用
- DNS服务 [sameersbn/docker-bind](https://github.com/sameersbn/docker-bind) 安装使用
- 容器日志收集 [Grafana Loki](https://grafana.com/oss/loki/) 安装和使用

## 准备
- 阅读前可以参考[VirtualBox虚拟机安装Ubuntu22.0.4.server](./VirtualBox虚拟机安装Ubuntu22.0.4.server.md)准备一个虚拟机环境。
- 本篇文章使用的服务器是: [Ubuntu 22.0.4](https://ubuntu.com)  
- 本篇`DNS` 相关内容可以用直接修改电脑 `hosts` 文件方式，不用安装 `dns` 服务服务也可以阅读。
    - 相比修改电脑 `hosts` 文件，使用 `dns` 服务可用性更高
- **文中一些命令需要 `root` 权限, 建议直接 `su` 提权后再进行命令执行, 避免一直输入密码**
## 安装

### 安装/下载/配置 ubuntu 
- 参考文章: [VirtualBox虚拟机安装Ubuntu22.0.4.server](./VirtualBox虚拟机安装Ubuntu22.0.4.server.md)
- 主要参考文章中的 `root` 密码配置和网络配置

### Docker 安装
#### 官网安装  
- [官方文档](https://docs.docker.com/engine/install/ubuntu/)  

- 下面是 `Ubuntu` 系统的安装最新稳定版 `Docker` 的命令, 直接复制后粘贴到命令行等待执行完成
    - 也可以按照[安官方文档]((https://docs.docker.com/engine/install/ubuntu/))安装  

    ```
    # Add Docker's official GPG key:
    sudo apt-get update
    sudo apt-get install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    # Add the repository to Apt sources:
    echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update

    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
    ```

- 验证 `$ docker version`  
    - 正常情况会输出下面这些内容, `...` 是本文档省略的一些内容  
    ```
    Client: Docker Engine - Community
    Version:           25.0.3
    ....

    Server: Docker Engine - Community
    Engine:
    Version:          25.0.3
    ....
    ```
~~- 修改 `docker` 配置  
    - 内容主要是添加国内镜像和日志切割
    - **如果文件已存在需要进行手动修改**
    ```
    cat > /etc/docker/daemon.json <<EOF
    {
        "registry-mirrors": [
            "https://hub-mirror.c.163.com"
        ]
    }
    EOF
    ```~~  国内镜像G了
- 创建默认网络 `$ docker network create net_def`  
    - 命令创建可能会导致和云服务器之间的 `IP` 段冲突，必要情况下可以[根据文档](https://docs.docker.com/network/)配置 `IP` 段。

- 重启 `docker`
    - 重启主要是更新配置
    - `$ sudo systemctl restart docker`
- 拉取需要的镜像
    ```
    docker pull portainer/portainer-ce:latest
    docker pull jc21/nginx-proxy-manager:latest
    docker pull sameersbn/bind:latest
    docker pull grafana/loki:2.9.4
    docker pull grafana/promtail:2.9.4
    docker pull grafana/grafana-enterprise:latest
    docker plugin install grafana/loki-docker-driver:2.9.4 --alias loki --grant-all-permissions
    ```
- 修改配置 
    ```
    cat > /etc/docker/daemon.json <<EOF
    {
        "log-driver": "loki",
        "log-opts": {        
            "loki-url": "http://localhost:3100/loki/api/v1/push",
            "max-size": "10m",
            "max-file": "10",
            "loki-max-backoff": "800ms",
            "keep-file": "true",
            "loki-timeout": "1s"
        }
    }
    EOF
    ```

- 重启 `docker`
    - 重启主要是更新配置
    - `$ sudo systemctl restart docker`

### 容器管理 [Portainer](https://www.portainer.io) 安装
#### 临时的 [Portainer](https://www.portainer.io) 服务  
- 启动一个临时的 `Portainer`  
    ```
    docker run -d \
    -p 8000:8000 -p 9443:9443 -p 9000:9000 \
    --name portainer_temp_runtime \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v portainer_data:/data \
    portainer/portainer-ce:latest
    ```

- 访问: `https://服务器IP:9443` 进行账号密码配置
    - 如果配置密码账号超时，重启 `$ docker restart portainer_temp_runtime` 后再次访问  
#### 使用 `docker-compose` 创建
- **注: 这一步在后面的`nginxProxyMnanager`和`bindDNS服务`创建时都有使用**
- 点击 `Home` > `local 容器` > `Stacks` > `Add stack` 进行创建
    - `Name` 字段填写对于名称  
    - `Build method` 选择 `Web editor(默认)`
        - 打开链接 [portainer](https://github.com/ulidev9527/docker-compose-template/blob/main/portainer/portainer.yml) 复制内容粘贴到 `Web editor` 下面的输入框里面
    - `Environment variables` 根据需要添加环境变量
    - `Deploy the stack` 按钮点击完成创建
#### 关闭临时的 `portainer`, 启动 `docker-compose` 配置的 `portainer`
- **注: 这一步在文章后面的 `nginx proxy manager` 创建 和 `bind DNS 服务` 创建完成后执行**
- 切换到服务器命令行
    - `$ docker stop portainer_temp_runtime`
    - `$ docker restart [对应的 portainer 容器]`
    - 访问[http://portainer.self.local](http://portainer.self.local)
        - 这个域名需要后面的 `bind DNS 服务` 创建和配置完成后即可使用
#### 如果忘记 `portainer` 密码  
- 停止`portainer` 容器
    - 临时方案: `$ docker stop portainer_start`
    - `docker-compose`方案: `$ docker stop portainer`
- 拉取镜像
    - `docker pull portainer/helper-reset-password`
- 查看密码
    - `$ docker run --rm -v /var/lib/docker/volumes/portainer_data/_data:/data portainer/helper-reset-password`
    - 输出  
        ```
        {"level":"info","filename":"portainer.db","time":"2024-02-22T14:29:46Z","message":"loading PortainerDB"}
        2024/02/22 14:29:46 Password successfully updated for user: admin
        2024/02/22 14:29:46 Use the following password to login: 4?o3Q@7b`A0vH)N<s8XjyC$lZ9d6-]U5
        ```
        - 其中: 
            - `Password successfully updated for user: admin` 中的 `admin` 是账号
            - `... Use the following password to login: ****` 中的 `****` 是密码  
- 启动 `portainer`容器
    - `$ docker start [对应的 portainer 容器]`
- 访问`portainer`对应地址,输入账号密码登陆

### 代理管理 [nginx proxy manager](https://nginxproxymanager.com) 安装
#### 创建
- 参考上面: 使用 `docker-compose` 创建
- 配置文件参考: [https://github.com/ulidev9527/docker-compose-template/blob/main/nginx/nginx.proxy.manager.yml](https://github.com/ulidev9527/docker-compose-template/blob/main/nginx/nginx.proxy.manager.yml)
- 完成后访问: `http://服务器IP:81`
- 初始邮箱: `admin@example.com`
- 初始密码: `changeme`
- 进行账号密码配置
- 如果密码账号不正确, 参考[官方文档](https://nginxproxymanager.com/guide/#quick-setup)
#### 配置反向代理
- http 代理
    - 选择 `Hosts` > `Proxy Hosts` > `Add Proxy Host` (右上按钮)
    - `Domain Names` 里面填写自己自定义的一个域名
        - 下面三个为我自定义的域名
        - `portainer`: [protainer.self.local](http://protainer.self.local)
            - 服务名: `portainer`
            - 端口: 9000
        - `nginxProxyManager`: [nginx.self.local](http://nginx.self.local)
            - 服务器: `nginxproxymanager`
            - 端口: 81
        - `bind DNS`: [bind.self.local](http://bind.self.local)
            - 需要下面步骤 `bind DNS 服务` 创建完成后添加
            - 服务名: bind
            - 端口: 10000
    - `Forward Hostname / IP`: `docker-compose.yml` 中的服务名称, 文件中 `services:` 的下级就是服务名, 也可以配置其他的域名或者直接配置 `IP`
    - `Forward Port`: 服务的访问端口
    - 其他配置根据需要进行添加
    - 点击 `Save` 按钮存储
    - 退出创建界面后，服务的 `Status` 为 `Online` 表示正常, 如果不正常可以查看容器日志
- Stream 代理
    - 选择 `Hosts` > `Streamd` > `Add Stream` (右上)
    - `Incoming Port` 填写服务器自己的端口，需要 [nginx代理](./docker/sys_nginxproxymanager/docker-compose.yml) `ports` 中开放
    - `Forward Host` 同上面 `Forward Hostname / IP`
    - `Forward Port` 同上面 `Forward Port`
    - 点击 `Save` 按钮存储
    - 退出创建界面后，服务的 `Status` 为 `Online` 表示正常, 如果不正常可以查看容器日志
#### 注意事项
- `docker`重启后或者服务器重启后, 如果`nginx proxy manager`配置的代理服务没有启动，可能导致 `nginx proxy manager` 内部的 `*.conf` 验证失败, 此时你需要使用 `docker` 命令启动其它关联服务的容器再启动`nginx proxy manager`容器即可


###  bind DNS 服务 [sameersbn/docker-bind](https://github.com/sameersbn/docker-bind) 安装
#### 关闭 `systemd-resolved` 服务
- 关闭服务
    ```
    sudo systemctl stop systemd-resolved
    sudo systemctl disable systemd-resolved
    ```
- 修改 `DNS` 解析
    
    修改系统 `DNS` 解析  
    - 内容为添加两个 `DNS` 解析地址和`hostname`补全
    - `nameserver 127.0.0.1` 为后面会在服务器添加 `DNS` 解析服务, 如果服务器不建 `DNS` 解析服务可以不添加
    - `DNS` 解析可以根据需要手动修改
    - **重启服务器后需要再次执行**
    ```
    rm /etc/resolv.conf
    cat > /etc/resolv.conf <<EOF
    # 这两个注释的 dns，如果在安装过程中服务器需要访问域名而添加的，可以不需要。
    # nameserver 223.6.6.6 # 阿里 DNS
    # nameserver 8.8.8.8 # Google DNS
    nameserver 127.0.0.1 # 自建 DNS
    search .
    EOF
    ```
#### 创建
- 参考上面: 使用 docker-compose 创建
- 配置文件参考: [https://github.com/ulidev9527/docker-compose-template/blob/main/bind/bind.yml](https://github.com/ulidev9527/docker-compose-template/blob/main/bind/bind.yml)
    - 环境变量
        - `BIND_HOST`: `bind.self.local`
        - `BIND_ROOT_PASSWORD`: `123!@#abcABC`
            - 这里密码自定义, 账号是 `root`
- 访问  
    - `http//:服务器IP:10000`
    - 账号: `root` 密码: 是 `BIND_ROOT_PASSWORD` 环境变量

- `DNS` 服务配置  
    - 创建 `DNS` 解析
        - 进入: `Servers > BIND DNS Server > Global Server Options > Zone Defaults > Default zone settings`
            - `Allow transfers from..`: 选择 ` Listed..`, 内容填: `any`
            - `Allow queries from..`: 选择 ` Listed..`, 内容填: `any`
        - 进入: `Servers > BIND DNS Server > Existing DNS Zones > Create master zone`
            - `Domain name / Network`: `self.local`, 
                - 域名可以自定义
                - 示例: `a.local` / `a.com` / `a.io` ...`
            - `Master server`: 填写 `localhost.`
            - `Email address`: 填写一个正常的邮箱地址
                - 可以随意填写，只要是邮箱格式, 比如: `a@b.c`
        - 点击 `Create` 按钮创建
        - 创建完成自动进入到: `Edit Master Zone`
            - 这个也可以从: `Servers (服务) > BIND DNS Server (BIND dns 服务) > Existing DNS Zones > 对应域名(地球图标位置)` 进入
            - 我们这里进入的是 `self.local` 配置
        - 点击 `Address` 按钮
            - `Name`: `*.self.local`
                - 这里填写的就是你要解析的域名和子域名
                - 示例: `a.com` / `bind.a.com` / `*.a.com`
            - `Address`: `服务器IP` 
                - 这里填写解析的目标IP,
                - 示例: `127.0.0.1`, **是否可以使用 `IP:Port` 待测**
            - 点击 `Create` 按钮创建一个域名解析

    - 添加第三方解析
        - 主要用于可以解析本机配置的 `DNS` 解析之外的域名解析功能
        - 进入: `Servers > BIND DNS Server > Global Server Options > Forwarding and Transfers`
        - 添加 IP `8.8.8.8` `8.8.4.4`, 这个是第三方 `dns` 解析添加，可以解析非自定义的一些域名
        - `Save` 按钮保存
    - 重启
        - `$ docker restart bind`
        - `$ docker restart nginxproxymanager`
#### 电脑修改 `DNS`
- 各系统修改 `DNS` 解析都不一样, 自行搜索解析教程
- 修改后刷新一下 `DNS`, 自行搜索教程
- 浏览器输入对应域名测试解析情况, 也可以使用 `dig` / `host` / `ping` / `curl` 等方法测试, 自行搜索教程
- 完成后即可进行域名访问

#### 回头配置上面未配置的内容
- 先添加 `bind DNS` 的方向代理
    - 搜索本文: `配置反向代理` 中的 `bind DNS: bind.self.local`
- 再按文中 `关闭临时的 portainer, 启动 docker-compose 配置的 portainer` 一节中的方案重启 `portainer`

### 日志收集 [Grafana Loki](https://grafana.com/oss/loki/) 安装

#### 启动相关容器
- [复制内容](https://github.com/ulidev9527/docker-compose-template/blob/main/grafana/grafana.and.loki.yaml)到 `portainer`并启动
- 在 `nginx proxy manager` 里面配置代理
    - 添加 `http://grafana.self.local`
        - 服务名: `grafana`
        - 端口: 3000
- 访问 `http://grafana.self.local`
    - 默认账号密码： `admin` `admin`
    - 配置新密码
-  重启 `docker`
    - `systemctl restart docker`
- 配置可视化
    - `Home > Connections > Data sources`
    - 点击右上的 `add new date source`
    - 搜索 `loki` 点击进入
    - 在 `url` 处填写 `http://loki:3100`
    - 点击最下边 `save & test` 按钮进行保存
- 查看可视化界面
    - `Home > Explore` 里面可以看到刚才配置的 `loki` 日志
#### 注意
- 注: 如果容器在安装 `Grafana Loki` 之前创建的，需要在 `portainer` 的 `Stacks` 中重新创建日志才会生效
    - 重启临时的 `portainer`
        - `$ docker stop [对应的 portainer 容器]`
        - `$ docker start portainer_temp_runtime`
    - 访问: `https://服务器IP:9443`
    - 在 `Containers` 全选，其中`portainer_temp_runtime`不选，然后删除.
    - 进入 `Stacks`, 按照 `bind > portainer > grafana_and_loki > nginx` 的顺序进去，在每个`Stack` 的 `Editor` 中点击 `Update the stack` 即可
        - 过程中可以考虑关闭 `bind` 的 `10000` 和 `nginxproxymanager` 的 `81` 减少端口暴露。
        - 创建后账号密码和配置不会丢失。
    - 完成后执行
        - `$ docker stop portainer_temp_runtime`
        - `$ docker restart [对应的 portainer 容器]`
    - 访问 `http://grafana.self.local` 即可查看日志
- 建议打印日志直接打印到一行, 不要用多行日志，多行日志需要特殊处理: [https://grafana.com/docs/loki/latest/send-data/promtail/stages/multiline/](https://grafana.com/docs/loki/latest/send-data/promtail/stages/multiline/)

### 访问

- `portainer`: [protainer.self.local](http://protainer.self.local)
- `nginxProxyManager`: [nginx.self.local](http://nginx.self.local)
- `bind DNS`: [bind.self.local](http://bind.self.local)
- `grafana`: [grafana.self.local](http://grafana.self.local)