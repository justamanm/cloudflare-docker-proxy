# cloudflare-docker-proxy

![deploy](https://github.com/ciiiii/cloudflare-docker-proxy/actions/workflows/deploy.yaml/badge.svg)


1.首先，需要fork仓库[cloudflare-docker-proxy](https://github.com/ciiiii/cloudflare-docker-proxy)


2.修改代码

（拉取到本地修改比较方便）

具体修改，参看修改后的完整版，[cloudflare-docker-proxy](https://github.com/justamanm/cloudflare-docker-proxy)

更改wrangler.toml

- wrangler是cloudflare的worker客户端工具
- 由于[env.production.vars]不会继承[vars]的变量，所以需要在生产环节中添加CUSTOM_DOMAIN

```bash
[env.vars]  # 整体删除，只有[vars]块，没有[env.vars]块

[env.production.vars]
MODE = "production"
TARGET_UPSTREAM = ""
CUSTOM_DOMAIN = "timex.fun"  # 添加
```

新增help.html

- 访问docker.domain.com时返回帮助信息
- 参看：[基于 Cloudflare Workers 和 cloudflare-docker-proxy 搭建镜像加速服务 ](https://www.cnblogs.com/KubeExplorer/p/18264358)

3.获取Account ID和API Token

Account ID

- 需要先创建一个worker后才会显示，去计算-> Workers 和 Pages页面中创建
- 点击 创建->快速开始->部署 即可完成

API Token

只要有workers编辑权限的API Token即可

- 创建：https://dash.cloudflare.com/profile/api-tokens 中创建
- 点击创建令牌
- 使用模版：编辑 Cloudflare Workers
- 账户资源：选择唯一的账户
- 区域资源：选择域名


4.点击`Deploy With Workers`按钮，然后按照步骤来即可

[![Deploy to Cloudflare Workers](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/justamanm/cloudflare-docker-proxy)


完成后可以在fork的项目的Action中看到deploy_to_cf_workers的任务

在中cloudflare workers中也能看到部署的woker了


5.修改仓库的Action

原Action设置的CUSTOM_DOMAIN有问题，始终会设置为libcuda.so，而不会从wrangler.toml配置文件中读取

```yaml
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    name: Build & Deploy
    steps:
      - uses: actions/checkout@v4  # GitHub 官方的 checkout action，将仓库代码检出到 runner（虚拟机）的工作目录（默认 $GITHUB_WORKSPACE），代码（src/index.js）可用
      - name: Publish
        uses: cloudflare/wrangler-action@v3  # 使用 Cloudflare 提供的 wrangler-action，在 runner 上运行 Wrangler CLI，执行 Workers 的部署
        # env:  # 定义环境变量 CUSTOM_DOMAIN，如果仓库->设置->secrets 中存在 CUSTOM_DOMAIN，使用其值（未设置）
        #   CUSTOM_DOMAIN: ${{ secrets.CUSTOM_DOMAIN || 'libcuda.so' }}
        with:  # 输入参数
          apiToken: ${{ secrets.CF_API_TOKEN }}  # Wrangler 使用此 API Token 认证 Cloudflare 账户
          accountId: ${{secrets.CF_ACCOUNT_ID}}  # 指定 Cloudflare 账户 ID，确定部署目标
          # vars:
          #   CUSTOM_DOMAIN  # 将 CUSTOM_DOMAIN 作为 Worker 的环境变量注入，在 Worker 脚本中可以通过 CUSTOM_DOMAIN 访问
          # 部署到生产环境（需在 wrangler.toml 定义）；压缩 JavaScript 代码，减小体积；指定入口脚本
          command: deploy --env production --minify src/index.js
          environment: production  # 指定 GitHub Actions 的环境为 production，与 Wrangler 的 --env 无直接关联
```





6.自定义域名

在 worker首页->右侧的子域 中可以看到worker的子域名，为`your_name.workers.dev`

如果存在名为cloudflare-docker-proxy的worker，则其访问域名则为`cloudflare-docker-proxy.your_name.workers.dev`



为单独worker配置子域名，如cloudflare-docker-proxy

设置 -> 域和路由

- 添加 -> 自定义域（将 Worker 连接到 Cloudflare 区域内的域或子域）

会在dns中添加一条记录，自定义域名指向了此worker

本地配置镜像源为`docker.timex.fun`

修改 `/etc/docker/daemon.json`

```bash
# sudo vim /etc/docker/daemon.json

# 配置
{
    "registry-mirrors": [
        "https://docker.timex.fun"
    ]
}
```

必须重启docker

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker

# 重启后有的容器不会被启动，使用脚本一键启动
~/library/nginx/conf/switch_nginx_conf.sh
```

查看更改是否已生效

```bash
docker info --format '{{.RegistryConfig.Mirrors}}'
```

<br><br>
之后的请求流程

1. 客户端配置了镜像源为`docker.timex.fun`，然后拉取镜像
2. dns转发到名为cloudflare-docker-proxy的worker中，执行其代码
3. 代码中会判断请求url
    1. 如果是docker.timex.fun，则请求docker hub的官方地址`https://registry-1.docker.io`
    2. 如果是k8s.timex.fun，则请求k8s的官方地址`https://registry.k8s.io`
    3. 其他同理...

目前只添加了docker.timex.fun，如果还要代理其他镜像源，则还可以添加，如`k8s.timex.fun`、`ghcr.timex.fun`，cloudflare-docker-proxy这个worker都会自动转发，拉取对应源上的镜像然后返回给客户端


## Deploy

1. click the "Deploy With Workers" button
2. follow the instructions to fork and deploy
3. update routes as you requirement

[![Deploy to Cloudflare Workers](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/justamanm/cloudflare-docker-proxy)

## Routes configuration tutorial

1. use cloudflare worker host: only support proxy one registry
   ```javascript
   const routes = {
     "${workername}.${username}.workers.dev/": "https://registry-1.docker.io",
   };
   ```
2. use custom domain: support proxy multiple registries route by host
   - host your domain DNS on cloudflare
   - add `A` record of xxx.example.com to `192.0.2.1`
   - deploy this project to cloudflare workers
   - add `xxx.example.com/*` to HTTP routes of workers
   - add more records and modify the config as you need
   ```javascript
   const routes = {
     "docker.libcuda.so": "https://registry-1.docker.io",
     "quay.libcuda.so": "https://quay.io",
     "gcr.libcuda.so": "https://k8s.gcr.io",
     "k8s-gcr.libcuda.so": "https://k8s.gcr.io",
     "ghcr.libcuda.so": "https://ghcr.io",
   };
   ```

