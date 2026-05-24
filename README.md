# 纷析云开源财务系统 - Docker 部署

基于 [纷析云 (Fenxiyun)](https://gitee.com/jason199111/public-financial) 开源财务系统，采用 Spring Boot + Vue2 技术栈，支持中国会计准则（复式记账、标准科目编码、三大报表、期末结转、辅助核算）。

## 服务器信息

| 项目 | 详情 |
|------|------|
| 服务器 | 阿里云 ECS 轻量型 2核2G |
| 系统 | Ubuntu 22.04 |
| IP | 47.96.17.173 |
| 访问地址 | http://47.96.17.173 |

## Docker 架构

```
                      172.25.0.0/24
              ┌─────────────────────────────────┐
              │                                 │
    :80 ──────┤   Nginx (172.25.0.6)            │
              │          │                      │
              │          ▼                      │
              │   Spring Boot :8012             │
              │   (172.25.0.5, host :8888)      │
              │      │          │               │
              │      ▼          ▼               │
              │   MySQL 8      Redis 5          │
              │   172.25.0.2   172.25.0.3        │
              │   (host :3307) (host :6479)     │
              │                                 │
              └─────────────────────────────────┘
```

4 个容器通过自定义桥接网络 `financial-ce` 互联。

## 端口暴露

| 端口 | 服务 | 说明 |
|------|------|------|
| 80 | Nginx | 前端 + API 代理 |
| 3307 | MySQL 8 | 数据库直连（建议仅内部使用） |
| 6479 | Redis 5 | 缓存（需密码认证） |
| 8888 | Spring Boot | 后端直接访问（调试用） |

## 项目结构

```
/opt/fenxiyun/
├── docker/                        # Docker 部署配置
│   ├── docker-compose.yml
│   ├── application.yml            # Spring Boot 生产配置
│   ├── .env                       # 环境变量（密码等）
│   ├── mysql/
│   │   ├── data/                  # MySQL 数据持久化
│   │   ├── conf/                  # MySQL 自定义配置
│   │   └── init/                  # 初始化 SQL 脚本
│   ├── redis/                     # Redis 持久化
│   ├── server/
│   │   └── financial.jar          # Spring Boot JAR
│   ├── logs/                      # 应用日志
│   └── nginx/
│       ├── vhost/                 # Nginx 配置
│       ├── wwwroot/               # 前端静态文件
│       └── logs/                  # Nginx 日志
├── bs-server/                     # 后端源码 (Gradle, Spring Boot 2.1.8)
├── front-end/                     # 前端源码 (Vue2 + HeyUI)
└── financial.sql                  # 数据库初始化脚本 (24 张表)
```

## 部署步骤

```bash
# 1. 将本仓库 docker-config/ 目录上传到服务器的 /opt/fenxiyun/docker/
scp -r docker-config/ root@47.96.17.173:/opt/fenxiyun/docker/

# 2. 放入 JAR 包和前端文件
# - 将 financial.jar 放到 docker/server/
# - 将前端 dist/ 内容放到 docker/nginx/wwwroot/
# - 将 financial.sql 放到 docker/mysql/init/

# 3. 启动
cd /opt/fenxiyun/docker
docker compose up -d

# 4. 查看状态
docker compose ps
docker compose logs -f client
```

## 数据库

### 连接方式

- **外部连接**: 47.96.17.173:3307
- **容器内部**: 172.25.0.2:3306
- **数据库**: financial
- **用户/密码**: root / fenxiyun_root_2024（建议修改 .env 中的密码后重新部署）

### 数据表（fxy_financial_ 前缀，共 24 张）

| 表名 | 说明 |
|------|------|
| fxy_financial_account_subject | 会计科目 |
| fxy_financial_voucher | 记账凭证 |
| fxy_financial_voucher_detail | 凭证明细 |
| fxy_financial_period | 会计期间 |
| fxy_financial_balance_sheet | 资产负债表 |
| fxy_financial_profit_statement | 利润表 |
| fxy_financial_cash_flow_statement | 现金流量表 |
| fxy_financial_customer | 客户/供应商 |
| fxy_financial_user | 用户 |
| fxy_financial_role | 角色 |
| fxy_financial_permission | 权限 |
| fxy_financial_company | 公司信息 |
| fxy_financial_auxiliary_accounting | 辅助核算 |
| fxy_financial_period_settle | 期末结转 |
| fxy_financial_department | 部门 |
| fxy_financial_project | 项目 |
| fxy_financial_employee | 员工 |
| fxy_financial_inventory | 存货 |
| fxy_financial_fixed_assets | 固定资产 |
| fxy_financial_bank_account | 银行账户 |
| fxy_financial_tax_rate | 税率 |
| fxy_financial_currency | 币种 |
| fxy_financial_log | 操作日志 |
| fxy_financial_system_config | 系统配置 |

## 修改后重新部署

```bash
# 1. 构建后端 (建议在本地构建后上传，服务器内存有限)
cd /opt/fenxiyun/bs-server
./gradlew build -x test
cp build/libs/financial-0.1.jar /opt/fenxiyun/docker/server/financial.jar

# 2. 构建前端
cd /opt/fenxiyun/front-end
npm install && npm run build
cp -r dist/* /opt/fenxiyun/docker/nginx/wwwroot/

# 3. 重启后端
cd /opt/fenxiyun/docker
docker compose restart client
```

## 数据备份

```bash
# 备份数据库
docker exec financial_mysql8 mysqldump -uroot -pfenxiyun_root_2024 financial > backup_$(date +%Y%m%d).sql

# 备份全部配置和数据
tar -czf fenxiyun_backup_$(date +%Y%m%d).tar.gz /opt/fenxiyun/docker/
```

## 常用运维命令

```bash
cd /opt/fenxiyun/docker

# 查看所有容器状态
docker compose ps

# 查看后端日志
docker compose logs -f --tail=100 client

# 重启某个容器
docker compose restart client

# 全部重启
docker compose down && docker compose up -d

# 完全重建（清空数据）
docker compose down -v
rm -rf mysql/data redis/
docker compose up -d
```

## 安全建议

- 生产环境建议配置 HTTPS（使用 Let's Encrypt / certbot）
- 修改 .env 中的默认密码
- MySQL 3307 和 Redis 6479 端口建议仅对内部 IP 开放
- 阿里云安全组仅开放 80/443 端口，其余端口建议关闭公网访问

## 已知问题

1. **服务器内存有限 (2G)**: npm build 可能 OOM，建议在本地构建前端后上传 dist 目录
2. **JVM 内存限制**: 已设 `-Xms256m -Xmx512m`，避免 OOM
3. **默认无用户**: 数据库初始化后需手动创建管理员账号
4. **短信服务**: 阿里云 SMS 配置为空，如需手机验证码功能需填入实际的 accessKey

## Docker 镜像加速器

```json
{
  "registry-mirrors": [
    "https://docker.1panel.live",
    "https://hub.rat.dev",
    "https://docker.m.daocloud.io"
  ]
}
```

> 配置文件路径: `/etc/docker/daemon.json`，修改后需执行 `systemctl reload docker`

## 技术栈

- **后端**: Spring Boot 2.1.8 + MyBatis-Plus + MySQL 8 + Redis 5
- **前端**: Vue 2 + HeyUI 组件库
- **认证**: sa-token
- **构建**: Gradle (多模块)
- **部署**: Docker Compose
