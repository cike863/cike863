Apache APISIX 是一个动态、实时、高性能的云原生 API 网关，提供了负载均衡、动态上游、灰度发布、服务熔断、身份认证、可观测性等丰富的流量管理功能。具体应用场景可参考官网（https://apisix.apache.org）说明，以下简单介绍Windows下借助Docker Desktop来安装APISIX。

**环境**

**Windows操作系统：Windows10（21H2，19044.1766）**

**Linux操作系统：Ubuntu 22.04 LTS**

**Docker Desktop：v4.10.1**

**下载**

- 下载脚本：首先下载安装脚本

git clone https://github.com/apache/apisix-docker.git

![img](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20221027_3204c058-55da-11ed-85bb-fa163eb4f6be.png)

- 安装：然后进入示例目录，执行安装过程

cd apisix-docker/example

docker-compose -p docker-apisix up -d

![img](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20221027_321ff1de-55da-11ed-85bb-fa163eb4f6be.png)

![img](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20221027_3238cdee-55da-11ed-85bb-fa163eb4f6be.png)

- 检查：安装完成后，可使用docker ps命令查看APISIX容器是否正常运行

![img](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20221027_325a97ee-55da-11ed-85bb-fa163eb4f6be.png)

**验证**

- http接口：使用http接口测试APISIX是否正常安装

curl "http://127.0.0.1:9080/apisix/admin/services/" -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1'

![img](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20221027_3285682a-55da-11ed-85bb-fa163eb4f6be.png)

- 网页登录：然后可打开网页版（http://localhost:9000/）登录(admin/admin)查看

![img](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20221027_32a9cef4-55da-11ed-85bb-fa163eb4f6be.png)

- 页面效果：登录完成后的效果

![img](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20221027_32c71f5e-55da-11ed-85bb-fa163eb4f6be.png)

**测试**

- 启动测试服务：其中hashicorp/http-echo是http测试镜像

docker run -d -p 6001:5678 --name echo-web1 hashicorp/http-echo -text="echo web1"

docker run -d -p 6002:5678 --name echo-web2 hashicorp/http-echo -text="echo web2"

![img](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20221027_32e7182c-55da-11ed-85bb-fa163eb4f6be.png)

- 添加路由

![img](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20221027_330cd76a-55da-11ed-85bb-fa163eb4f6be.png)



![img](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20221027_3336ed5c-55da-11ed-85bb-fa163eb4f6be.png)

- 测试路由

curl 192.168.0.110:6001

curl 192.168.0.110:6002

curl 192.168.0.110:9080/echo

![img](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20221027_335d2792-55da-11ed-85bb-fa163eb4f6be.png)

测试中，会根据路由规则分别轮训调用web1和web2服务

***说明：***

***1.可在Docker Desktop配置界面设置docker源***

![img](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20221027_33e0d876-55da-11ed-85bb-fa163eb4f6be.png)

***2.路由配置过程中上游服务需要配置具体IP地址，而不能是127.0.0.1***