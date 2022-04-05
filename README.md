# The Road To DevOps

## devops-main

### 系统配置

1.openEuler-20.03-LTS-SP3

### 网络配置

1.可获得$IFNAME -> enp3s0

```shell
nmcli connection show
```

2.连接网络设备

```shell
nmcli device connect "$IFNAME"
```

3.配置静态IP -> 192.168.2.28

以enp3s0网络接口进行静态网络设置为例，通过在root权限下修改ifcfg文件实现，在/etc/sysconfig/network-scripts/目录中生成名为ifcfg-enp3s0的文件中，修改参数配置，示例如下：

```shell
vi /etc/sysconfig/network-scripts/ifcfg-enp0s3
```

```porperties
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
IPADDR=192.168.2.28
PREFIX=24
GATEWAY=192.168.2.1
DNS1=8.8.8.8
DNS2=114.114.114.114
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=enp0s3static
UUID=08c3a30e-c5e2-4d7b-831f-26c3cdc29293
DEVICE=enp0s3
ONBOOT=yes
```

4.reboot

```shell
reboot
```

5.查看配置的连接详情 -> enp0s3static

```shell
nmcli -p con show enp0s3static
```

### 管理软件包,使用DNF

1.检查更新

```shell
dnf check-update
```

2.更新所有的包和它们的依赖

```shell
dnf update
```

### Docker & Docker Compose

1.安装docker

```shell
dnf install docker
```

2.安装docker-compose

```shell
dnf install docker-compose
```

### Gitlab EE

#### 准备

1.准备docker工作目录：/usr/local/docker

2.安装docker gitlab

2.1 准备docker gitlab目录

1. 工作目录:/usr/local/docker/gitlab
2. 创建目录

```shell
mkdir /usr/local/docker/gitlab
mkdir /usr/local/docker/gitlab/volume/
mkdir /usr/local/docker/gitlab/volume/config
mkdir /usr/local/docker/gitlab/volume/logs
mkdir /usr/local/docker/gitlab/license/
```

2.2 配置ssl

```shell
mkdir /usr/local/docker/gitlab/volume/config/ssl
cd /usr/local/docker/gitlab/volume/config/ssl
openssl genrsa -des3 -out 192.168.2.28.key 2048
# pass phrase
#783a544f59d298255005a0f39c8bea52
openssl req -new -key 192.168.2.28.key -out 192.168.2.28.csr
#Country Name (2 letter code) [AU]:: CN
#State or Province Name (full name) [Some-State]:
#Locality Name (eg, city) []:
#Organization Name (eg, company) [Internet Widgits Pty Ltd]:lastsunday
#Organizational Unit Name (eg, section) []:
#Common Name (e.g. server FQDN or YOUR name) []:192.168.2.28
#Email Address []:
#A challenge password []:c294bd40ad4956fefbef
#An optional company name []:
nano v3.ext #见下面的v3.ext内容
openssl x509 -in 192.168.2.28.csr -out 192.168.2.28.crt -req -signkey 192.168.2.28.key -days 3650 -sha256 -extfile v3.ext
#If the certificate.key file is password protected, NGINX will not ask for the password when you reconfigure GitLab. In that case, Omnibus GitLab will fail silently with no error messages. To remove the password from the key, run:
cp 192.168.2.28.key 192.168.2.28.key.bak
openssl rsa -in 192.168.2.28.key.bak -out 192.168.2.28.key
```

***v3.ext***

```txt
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = IP:192.168.2.28
```

2.3 安装gitlab

```shell
cd /usr/local/docker/gitlab
nano docker-compose.yml
docker-compose up
```

***docker-compose.yml***

```yml
gitlab:
  image: 'gitlab/gitlab-ee:14.9.2-ee.0'
  restart: always
  hostname: '192.168.2.28'
  environment:
    GITLAB_OMNIBUS_CONFIG: |
      external_url 'https://192.168.2.28:8081'
      nginx['redirect_http_to_https_port'] = 8081
      puma['exporter_enabled'] = true
      puma['exporter_port'] = 2443
      gitlab_rails['gitlab_shell_ssh_port'] = 2022
      nginx['enable'] = true
      registry_external_url 'https://192.168.2.28:4567'
      registry_nginx['redirect_http_to_https'] = true
  ports:
    - '8081:8081'
    - '2443:2443'
    - '2022:22'
    - '4567:4567'
  volumes:
    - './volume/config:/etc/gitlab'
    - './volume/logs:/var/log/gitlab'
    - './volume/data:/var/opt/gitlab'
```

2.4 登录gitlab

重新启动GitLab，登录管理员账户root（密码见：/usr/local/docker/gitlab/volume/config/initial_root_password）

```shell
cat /usr/local/docker/gitlab/volume/config/initial_root_password
```
