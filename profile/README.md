# 地王地板官网（diwang-website）

本组织存放地王地板官网 v2 的三个业务仓库：面向用户的官网、内容管理后台，以及共用的 API 服务。三者分工明确、通过同一套后端与数据联动，生产环境以 Docker 镜像分别发布。

## 仓库一览

| 仓库 | 角色 | 技术栈 | 生产域名 |
| --- | --- | --- | --- |
| [diwang-web](https://github.com/diwang-website/diwang-web) | 官网前台（访客浏览） | Vue 3 · Vite · Vuex · Less | [dwfloor.com](https://dwfloor.com) |
| [diwang-admin](https://github.com/diwang-website/diwang-admin) | 管理后台（CMS） | Vue 3 · Ant Design Vue · ECharts | [admin-v2.dwfloor.com](https://admin-v2.dwfloor.com) |
| [diwang-server](https://github.com/diwang-website/diwang-server) | 后端 API（数据与文件） | Go · Gin · GORM · MySQL · JWT | [api-v2.dwfloor.com](https://api-v2.dwfloor.com) |

本地开发时，三个项目通常与发布说明目录并列放置（`diwang-deploy` 仅为运维编排与文档，不是业务 monorepo）：

```text
diwang-web/      # 官网
diwang-admin/    # 后台
diwang-server/   # API
diwang-deploy/   # 生产编排、Nginx 片段、构建/拉取脚本说明（可选）
```

## 三者如何联动

```text
  访客浏览器                    运营人员浏览器
       │                              │
       ▼                              ▼
  diwang-web                     diwang-admin
  （官网 SPA）                   （后台 SPA）
       │                              │
       │  /api/public/*               │  /api/admin/*  + JWT
       │  /api/track                  │  上传、CRUD、统计
       │  /imagesData/*               │  /imagesData/*
       └──────────────┬───────────────┘
                      ▼
                diwang-server
                （统一 API + 静态资源）
                      │
                      ▼
               MySQL（业务库）
               上传目录 imagesData
```

### 1. 共用同一后端

- **diwang-web** 只读公开接口：站点信息、产品/系列/图案、新闻、人文、门店、轮播、视频地址等；页面访问与点击通过埋点接口写入访问事件。
- **diwang-admin** 使用管理员登录拿到的 JWT，调用管理接口完成内容维护、用户与权限相关操作、仪表盘统计等；写操作最终都落在 **diwang-server** 与同一数据库。
- **diwang-server** 是唯一业务后端：对外提供 `/api/public/*` 与 `/api/admin/*`，并托管 `/imagesData`（或等价上传目录）下的图片与下载文件。前台与后台都不直连数据库。

### 2. 内容与展示闭环

运营在 **admin** 中维护的产品、新闻、轮播、门店、标签、站点文案等，经 **server** 持久化后，由 **web** 在对应页面拉取并展示。后台改内容 → 接口与库更新 → 官网刷新或再次请求即可看到新数据，无需改前台代码。

### 3. 媒体与文件

富文本与列表中的图片、下载资料等由 **server** 接收上传并落在挂载的 `imagesData`（生产上由宿主机目录挂入容器）。**web** / **admin** 通过同一资源路径访问这些文件，保证前台展示与后台预览一致。

### 4. 访问统计

官网在浏览、点击等行为时向 **server** 上报访问事件（含路径、设备端型、IP 等；服务端可结合 IP 库补全省市）。**admin** 工作台消费这些统计（如访问量、地区、最近访问城市等），形成「前台产生数据 → 后台查看」的联动。

### 5. 本地联调关系

| 项目 | 默认开发端口 | 接口走向 |
| --- | --- | --- |
| diwang-server | `8082` | 直接提供 API 与资源 |
| diwang-web | Vite 开发端口（见各仓 README） | 将 `/api`、`/imagesData` 代理到 `8082` |
| diwang-admin | 同上 | 将管理 API 与资源代理到同一 `8082` |

因此本地需要先起 **server**，再起 **web** / **admin**，三者才能完整联动。

## 生产部署关系（简要）

- 三个仓库各自维护 Dockerfile；推送到 `main` 后由 GitHub Actions 构建镜像并推至阿里云容器镜像服务（`registry.cn-beijing.aliyuncs.com/cellular/` 下的 `diwang-web` / `diwang-admin` / `diwang-server`），再 SSH 到服务器执行对应更新脚本拉取镜像重启。
- 服务器**不编译源码**，只运行镜像；宿主机 Nginx 终结 HTTPS，反代到本机端口上的三个容器。
- 数据库与图片目录在编排外维护；三个容器通过同一后端与共享上传卷协作，而不是互相直接调用。

更细的端口、compose、证书与发布步骤见运维侧的 `diwang-deploy` 说明（若组织内另有运维文档，以现网文档为准）。

## 各仓职责边界

| | diwang-web | diwang-admin | diwang-server |
| --- | --- | --- | --- |
| 面向谁 | 终端访客 | 内部运营 / 管理员 | 两个前端 |
| 是否含业务库逻辑 | 否 | 否 | 是 |
| 鉴权 | 公开接口为主 | JWT 登录后访问管理接口 | 签发与校验 JWT；区分 public / admin |
| 典型改动 | 页面与交互、SEO/体验相关展示 | 表单、列表、工作台、权限相关 UI | 接口、模型、统计、上传、地区解析等 |

## 相关链接

- 官网：https://dwfloor.com  
- 管理后台：https://admin-v2.dwfloor.com  
- API：https://api-v2.dwfloor.com  

---

*本文说明三个业务仓库之间的分工与联动，便于在 GitHub 组织简介或 profile README 中使用。各仓细节以对应仓库 README 为准。*
