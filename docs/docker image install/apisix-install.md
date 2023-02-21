# APISIX 安装指南

本文将介绍如何在你的环境中安装并运行 APISIX。

关于如何快速运行 Apache APISIX，请参考[入门指南](https://apisix.apache.org/zh/docs/apisix/getting-started/)。

## 安装 APISIX[#](https://apisix.apache.org/zh/docs/apisix/installation-guide/#安装-apisix)

你可以选择以下任意一种方式安装 APISIX：

- Docker
- Helm
- RPM
- Source Code

使用此方法安装 APISIX，你需要安装 [Docker](https://www.docker.com/) 和 [Docker Compose](https://docs.docker.com/compose/)。

首先下载 [apisix-docker](https://github.com/apache/apisix-docker) 仓库。

```shell
git clone https://github.com/apache/apisix-docker.git
cd apisix-docker/example
```

Copy

然后，使用 `docker-compose` 启用 APISIX。

- x86
- ARM/M1

```shell
docker-compose -p docker-apisix up -d
```

Copy

## 安装 etcd[#](https://apisix.apache.org/zh/docs/apisix/installation-guide/#安装-etcd)

APISIX 使用 [etcd](https://github.com/etcd-io/etcd) 作为配置中心进行保存和同步配置。在安装 APISIX 之前，需要在你的主机上安装 etcd。

如果你在安装 APISIX 时选择了 Docker 或 Helm 安装，那么 etcd 将会自动安装；如果你选择其他方法或者需要手动安装 APISIX，请参考以下步骤安装 etcd：

- Linux
- macOS

```shell
ETCD_VERSION='3.5.4'
wget https://github.com/etcd-io/etcd/releases/download/v${ETCD_VERSION}/etcd-v${ETCD_VERSION}-linux-amd64.tar.gz
tar -xvf etcd-v${ETCD_VERSION}-linux-amd64.tar.gz && \
  cd etcd-v${ETCD_VERSION}-linux-amd64 && \
  sudo cp -a etcd etcdctl /usr/bin/
nohup etcd >/tmp/etcd.log 2>&1 &
```

Copy

## 后续操作[#](https://apisix.apache.org/zh/docs/apisix/installation-guide/#后续操作)

### 配置 APISIX[#](https://apisix.apache.org/zh/docs/apisix/installation-guide/#配置-apisix)

通过修改本地 `./conf/config.yaml` 文件，或者在启动 APISIX 时使用 `-c` 或 `--config` 添加文件路径参数 `apisix start -c <path string>`，完成对 APISIX 服务本身的基本配置。

比如将 APISIX 默认监听端口修改为 8000，其他配置保持默认，在 `./conf/config.yaml` 中只需这样配置：

“./conf/config.yaml”

```yaml
apisix:
  node_listen: 8000 # APISIX listening port
```

Copy

比如指定 APISIX 默认监听端口为 8000，并且设置 etcd 地址为 `http://foo:2379`，其他配置保持默认。在 `./conf/config.yaml` 中只需这样配置：

“./conf/config.yaml”

```yaml
apisix:
  node_listen: 8000 # APISIX listening port

deployment:
  role: traditional
  role_traditional:
    config_provider: etcd
  etcd:
    host:
      - "http://foo:2379"
```

Copy

##### WARNING

APISIX 的默认配置可以在 `./conf/config-default.yaml` 文件中看到，该文件与 APISIX 源码强绑定，请不要手动修改 `./conf/config-default.yaml` 文件。如果需要自定义任何配置，都应在 `./conf/config.yaml` 文件中完成。

##### WARNING

请不要手动修改 APISIX 安装目录下的 `./conf/nginx.conf` 文件。当 APISIX 启动时，会根据 `config.yaml` 的配置自动生成新的 `nginx.conf` 并自动启动服务。

### 更新 Admin API key[#](https://apisix.apache.org/zh/docs/apisix/installation-guide/#更新-admin-api-key)

建议修改 Admin API 的 key，保护 APISIX 的安全。

请参考如下信息更新配置文件：

./conf/config.yaml

```yaml
deployment:
  admin:
    admin_key
      -
        name: "admin"
        key: newsupersecurekey  # 请修改 key 的值
        role: admin
```

Copy

更新完成后，你可以使用新的 key 访问 Admin API：

```shell
curl http://127.0.0.1:9180/apisix/admin/routes?api_key=newsupersecurekey -i
```

Copy

### 为 APISIX 添加 systemd 配置文件[#](https://apisix.apache.org/zh/docs/apisix/installation-guide/#为-apisix-添加-systemd-配置文件)

如果你是通过 RPM 包安装 APISIX，配置文件已经自动安装，你可以直接使用以下命令：

```shell
systemctl start apisix
systemctl stop apisix
```

Copy

如果你是通过其他方法安装的 APISIX，可以参考[配置文件模板](https://github.com/api7/apisix-build-tools/blob/master/usr/lib/systemd/system/apisix.service)进行修改，并将其添加在 `/usr/lib/systemd/system/apisix.service` 路径下。

如需了解 APISIX 后续使用，请参考[入门指南](https://apisix.apache.org/zh/docs/apisix/getting-started/)获取更多信息。