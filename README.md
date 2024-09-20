<!--
 * @Author: aspnmy support@e2bank.cn
 * @Date: 2024-09-20 23:50:38
 * @LastEditors: aspnmy support@e2bank.cn
 * @LastEditTime: 2024-09-21 04:15:07
 * @FilePath: \1Panel_in_podman\README.md
 * @Description: 不改"1Pane面板"代码，使"1Pane面板"、宝塔面板完美支持podman并且可以使"1Pane面板"自带docker-compose编排共存
-->
# 1Panel_bt_in_podman （不改"1Pane面板"代码，使"1Pane面板"、宝塔面板完美支持podman并且可以使"1Pane面板"自带docker-compose编排共存）
## 项目说明
- 针对centos9版本以上的系统，已经拥抱下一代rootless容器podman，特点是不需要root权限，不需要守护进程，而国内的主机"1Pane面板"，如开源"1Pane面板"1Panel，宝塔目前基本都还是docker环境，安装使用的时候会和podman有冲突，无法完美使用。本项目就是为了能完美共存podman-rootless模式而出现的，并且使"1Pane面板"自带的docker-compose模式不受影响。

## 实现思路（系统版本：Debian v12 | 主机操作面板：1Panel 社区版v1.10.15-lts）
- podman的既然取代docker，所以它本身是系统完美调用的，就是指输入docker的命令系统自动呼起podman组件，至于在docker环境的"1Pane面板"上使用不兼容的主要原因是，特征化命令不一致，以及配置文件被"1Pane面板"docker组件覆盖。所以我们先安装"1Pane面板"组件前先安装podman及podman-docker中间件及docker-compose组件，然后进行统一化兼容配置设置，最后在安装"1Pane面板"脚本的时候，跳过docker及docker-compose环境的安装即可。
### 思考历程
- 第一、我们要先安装podman 以及podman-docker这个中间件
- 第二、为了让"1Pane面板"能正确拉起docker-compose服务，还需要全局安装docker-compose:v1、docker-compose:v2两个版本，以非插件形式安装，可以被systemld全局调用
- 第三、最关键是这个，软连接 ( ln -s /run/user/$rootless/podman/podman.sock /var/run/docker.sock ),让/var/run/docker.sock与podman.sock软连接，这样"1Pane面板"调用套接字/var/run/docker.sock时就能呼起/run/user/$rootless/podman/podman.sock
- ($rootless 代表非root用户的uid，为了和docker兼容，一般使用uid=1000的路径，如果podman不用rootless模式，就直接挂在/run/user/0/podman/podman.sock路径上即可)

```bash
    #rootless模式podman与docker套接字挂接，docker-compose环境挂接
    ln -s /run/user/1000/podman/podman.sock /var/run/docker.sock
    ln -s /run/user/1000/podman/podman.sock /run/user/0/docker.sock
    ln -s /run/user/1000/podman/podman.sock /run/user/1000/docker.sock

    #root模式podman与docker套接字挂接，docker-compose环境挂接
    ln -s /run/user/0/podman/podman.sock /var/run/docker.sock
    ln -s /run/user/0/podman/podman.sock /run/user/0/docker.sock
    ln -s /run/user/0/podman/podman.sock /run/user/1000/docker.sock
```

- docker-compose系统变量环境配置：默认podman容器环境中，"1Pane面板"拉起配置文件docker-compose up -d 很多时候报错，可能是系统环境变量没有配置，所以我们需要手工配置一下，没有就新增，有就覆盖，统一化配置以后，才能和"1Pane面板"兼容。

```bash
    # 没有就新增，有的就覆盖，统一化配置以后，才能和"1Pane面板"兼容
    touch ~/ .bash_profile \
    && export DOCKER_HOST=unix:///run/user/1000/docker.sock\
    && source ~/.bash_profile
```
- podman启动套接字服务：为了与docker兼容，是必须启用user sock模式的

```bash
    # 开启podman系统套接字
    # 开启podman用户套接字
    systemctl enable --now podman.socket \
    && systemctl --user enable --now podman.socket
        
```

- 为uid=1000的用户配置docker组件权限（解释一下：为什么要手工配置参数，因为让podman和"1Pane面板"环境兼容，不修改"1Pane面板"底层代码的情况下，所有配置参数都必须手工统一化配置，然后"1Pane面板"安装的时候docker及docker-compose都必须跳过安装，不然配置参数就覆盖了。虽然配置参数覆盖的情况下可以使用补丁sh脚本修复，但是生产环境这样做风险太高）

```bash
    # 新增用户组docker
    groupadd docker
    # 新增一个用户docker_usr,uid=1000
    adduser --uid 1000 docker_usr
    # 为用户docker_usr划入docker业务组
    usermod -aG docker docker_usr
    # 为用户docker_usr配置业务权限(为了避免意外错误，主要配置的权限组件时sudo、docker、podman、docker-compose，这几个组件已经够"1Pane面板"呼起docker环境了)
    # Debian v11 以上的系统默认是没有安装sudo组件的，可以自己先安装好再进行配置。centos9以上系统都自带sudo可以直接执行配置脚本
    /sbin/usermod -aG sudo docker_usr \
    && /sbin/usermod -aG docker docker_usr\
    && /sbin/usermod -aG podman docker_usr\
    && /sbin/usermod -aG docker-compose docker_usr
    
```

### 思考总结：
- 到此步骤基本上"1Pane面板"已经可以完美兼容podman-rootless模式了，下面开始做实践操作，调整一下脚本的兼容性顺序。


## 实践步骤（系统版本：Debian v12 | 主机操作面板：1Panel 社区版v1.10.15-lts）
### 1、root用户安装sudo、podman环境、podman-docker、docker-compose:v1环境(apt官方源提供的是apt install docker-compose，一般为docker-compose:v1组件)
```bash
    apt-get update \
    && apt install podman podman-docker docker-compose
    
```
### 2、root用户安装docker-compose:v2环境(docker-compose:v2组件一般从github拉取最新版本，目前最新版本是v2.29.7)
```bash
ver="v2.29.7";
curl -LO https://github.com/docker/compose/releases/download/$ver/docker-compose-linux-x86_64 \
&& chmod +x docker-compose-linux-x86_64 \
&& mv docker-compose-linux-x86_64 /usr/local/bin/docker-compose \
&& docker-compose -v    
```
-  正确跳出docker-compose版本为2.29.7后，代表docker-compose:v2安装成功，既然可以装docker-compose:v2为什么还要上面安装docker-compose:v1呢，主要是有些业务组件的源码是在v1上的，如果不安装的话，v2组件拉不到旧的组件源码会莫名其妙的报错，生产环境下我们还是都需要安装，开发环境随便你们折腾。

### 3、新增用户组docker，新增uid=1000的用户docker_usr，为docker_usr用户设置工作组，设置组件权限
```bash
    # 新增用户组docker
    groupadd docker \
    # 新增一个用户docker_usr,uid=1000
    && adduser --uid 1000 docker_usr \
    # 为用户docker_usr划入docker业务组
    && usermod -aG docker docker_usr \
    # 为用户docker_usr配置业务权限(为了避免意外错误，主要配置的权限组件时sudo、docker、podman、docker-compose，这几个组件已经够"1Pane面板"呼起docker环境了)
    # Debian v11 以上的系统默认是没有安装sudo组件的，可以自己先安装好再进行配置。centos9以上系统都自带sudo可以直接执行配置脚本
    && /sbin/usermod -aG sudo docker_usr \
    && /sbin/usermod -aG docker docker_usr\
    && /sbin/usermod -aG podman docker_usr\
    && /sbin/usermod -aG docker-compose docker_usr  
```
### 4、启动podman系统套接字、用户套接字
```bash
    systemctl enable --now podman.socket \
    && systemctl --user enable --now podman.socket  
```

### 5、配置软连接，配置兼容性系统环境，生产环境采用podman-rootless形式部署，docker采用uid=1000的用户单独给予执行权限部署。"1Pane面板"web界面使用中，编排文件中添加sudo命令就可执行root提权，部分老的docker镜像需要完全使用root账号执行，这种可以在ssh中单独启动。
```bash
    # 配置docker-compose环境变量并生效没有就新增，有的就覆盖，统一化配置以后，才能和"1Pane面板"兼容
    touch ~/ .bash_profile \
    && export DOCKER_HOST=unix:///run/user/1000/docker.sock\
    && source ~/.bash_profile
```
```bash
    #rootless模式podman与docker套接字挂接，docker-compose环境挂接
    ln -s /run/user/1000/podman/podman.sock /var/run/docker.sock \
    && ln -s /run/user/1000/podman/podman.sock /run/user/0/docker.sock \
    && ln -s /run/user/1000/podman/podman.sock /run/user/1000/docker.sock

    # #root模式podman与docker套接字挂接，docker-compose环境挂接
    # ln -s /run/user/0/podman/podman.sock /var/run/docker.sock
    # ln -s /run/user/0/podman/podman.sock /run/user/0/docker.sock
    # ln -s /run/user/0/podman/podman.sock /run/user/1000/docker.sock

    # 复制配置文件/lib/systemd/system/podman.service 到 /lib/systemd/system/docker.service 
    # 复制配置文件/lib/systemd/system/podman.socket 到 /lib/systemd/system/docker.socket
    cp /lib/systemd/system/podman.service /lib/systemd/system/docker.service
    cp /lib/systemd/system/podman.socket /lib/systemd/system/docker.socket
    # 配置docker守护进程，其实软链接的是podman的进程，但是配置好以后"1Pane面板"前端就能完美调用了，podman是无守护的意思就是podman的服务进程可以无限启动不会锁死
    chmod +x /lib/systemd/system/docker.service
    chmod +x /lib/systemd/system/docker.socket
    systemctl daemon-reload
    systemctl enable docker.service
    systemctl start docker
    # 查询状态已经是守护启动模式
    systemctl status docker.service
```

### 6、测试组件权限，切换到docker_usr用户下，分别执行组件命令看看是否能取得正确版本号
```bash
sudo docker -v
suod docker-compose -v
sudo podman -v
```

### 7、第6步测试通过后，开始安装"1Pane面板"脚本，采用线上安装脚本
- （注意如果使用其他系统安装的，第一步骤的安装docker环境脚本这个切记不要安装，不然上面的配置全部就失效了）
- 使用官方支持的系统脚本安装的使用，提示出现已经安装docker环境和docker-compose的，这里别选默认，选N，不安装，不然覆盖配置后兼容性就出错了，虽然有兼容性补丁，但是兼容性补丁不一定能覆盖所以的系统版本，所以从零开始纯净安装是最后的。
- 宝塔"1Pane面板"组件安装的时候提示，已安装容器环境，请纯净安装，脚本会退出，这里需要修改宝塔安装脚本的命令，也是可以共存环境的
- 本项目推荐1Panel"1Pane主机面板"，主要原因目前比较便宜专业版，其次自定义程度高，特别是docker环境配置可以自己设置系统套接字位置，这个会方便一点，
- 不过使用本项目中的脚本配置以后，直接使用1Panel"1Pane主机面板"中的默认套接字入口即可：/var/run/docker.sock
```bash
curl -sSL https://resource.fit2cloud.com/1panel/package/quick_start.sh -o quick_start.sh && bash quick_start.sh
```

### 8、登录"1Pane面板"web端，进行docker环境的配置，主要是修改的地方如下：
#### 8.1 加速url和私有化库配置：
- 此处的配置主要影响docker-compose的预拉取功能，如果习惯使用docker-compose启动镜像的，此处需要配置一下，不配置用"1Pane面板"默认值也行。
- 如果习惯独立镜像的直接以docker run的命令执行，本项目中调取的其实是podman run，所以还需要配置podman的regis配置文件
```bash
vim /etc/containers/registries.conf
```
- 主要修改加速url和私有库，具体如何查可以参考网络，或者直接下载我维护的系统加速镜像源文件，及自动更新镜像源的自动化脚本
- https://github.com/aspnmy/mirrors-repolist.git 找到对应的 containers/registries.conf下载覆盖就行

![加速url和私有化库配置](/images/asASASa.png)

#### 8.2 关闭"1Pane面板"的守护进程，修改cg的启动形式，配置套接字

![关闭"1Pane面板"的守护进程，修改cg的启动形式，配置套接字](/images/sdfs33.png)

#### 8.3 "1Pane面板"前端报错一般说明
##### 8.3.1 "服务内部错误: Error response from daemon: Not Found"
- 由于"1Pane面板"底层是调用docker daemon守护的，这个podman不需要守护，所以此处会报错,不影响使用，ssh处用podman脚本就行，或者安装portainer-ce容器组件，使用portainer来管理对支持性更好，当然也可以用podman自己的可视化管理组件
- 兼容性修复，就是按照上方第五点， 
```bash
    # 复制配置文件/lib/systemd/system/podman.service 到 /lib/systemd/system/docker.service 
    # 复制配置文件/lib/systemd/system/podman.socket 到 /lib/systemd/system/docker.socket
    cp /lib/systemd/system/podman.service /lib/systemd/system/docker.service
    cp /lib/systemd/system/podman.socket /lib/systemd/system/docker.socket
    # 配置docker守护进程，其实软链接的是podman的进程，但是配置好以后"1Pane面板"前端就能完美调用了，podman是无守护的意思就是podman的服务进程可以无限启动不会锁死
    chmod +x /lib/systemd/system/docker.service
    chmod +x /lib/systemd/system/docker.socket
    systemctl daemon-reload
    #启动docker.service支持面版兼容性问题，其实启动的是podman容器
    systemctl enable docker.service
    systemctl start docker
```
##### 8.3.2 "容器生产以后，点端口，会提示无效的端口"
- 这主要因为iptables防火墙没有安装，以Debian v12为例子，默认不带iptables防火墙规则，而"1Pane面板"需要这组件，只需要单独安装即可，卸载其他类型防火墙比如ufw
```bash
apt-get install iptables
```
- 我个人更喜欢用ufw这个防火墙，所以ufw和iptables同时存在的时候，"1Pane面板"上的docker端口自动放行功能是无效的，需要ufw防火墙手工开启端口

## 一键脚本
在sh目录下，按需使用