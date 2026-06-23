# 开发指南

## 本地命令

```bash
npm install
npm run api:dev
npm run web:dev
npm run check
npm run e2e
npm run api:check
npm run web:check
npm run deploy:test
```

本地 npm 开发需要自行准备 `apps/api/.env`、`apps/web/.env`、PostgreSQL 和 Redis。项目部署不再以本地 npm 启动为主路径。

`npm run e2e` 会启动 Next.js Web 和本地 mock API，只覆盖浏览器主流程，不依赖真实 AI Key、PostgreSQL 或 Redis。它当前不并入默认 `npm run check`，避免把浏览器安装和本地端口占用变成所有代码检查的前置条件。

## 目录

```text
apps/api        Fastify JSON API
apps/web        Next.js Web
packages/contracts  前后端共享 TypeScript DTO
deploy          Caddy、种子数据、演示脚本
scripts         部署、备份、管理员脚本
compose.yaml    默认服务器 Compose 栈
Dockerfile.api  API 镜像
Dockerfile.web  Web 镜像
```

## 质量检查

改 Docker、Compose、Caddy、环境变量或启动脚本后至少跑：

```bash
npm run check
npm run deploy:test
./scripts/deployment-stack.sh server config
sh -n scripts/deployment-stack.sh
sh -n scripts/deployment-smoke.sh
sh -n scripts/prod-deploy.sh
```

有 Docker daemon 和目标反代网络时再跑：

```bash
./scripts/deployment-stack.sh server preflight
UNFREQ_BASE_URL=https://你的域名 ./scripts/deployment-stack.sh server smoke
```
