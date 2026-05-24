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
                    ┌──────────────────────┐
                    │   Nginx (port 80)    │
                    │   172.25.0.4         │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  Spring Boot :8012   │
                    │  172.25.0.5          │
                    │  (exposed :8888)     │
                    └──────┬──────┬────────┘
                           │      │
              ┌────────────▼──┐ ┌─▼──────────────┐
              │ MySQL 8.2.0   │ │ Redis 5.0.12   │
              │ 172.25.0.2    │ │ 172.25.0.3     │
              │ (:3307 host)  │ │                │
              └───────────────┘ └────────────────┘
```

4 个容器通过自定义桥接网络 `financial-ce`（子网 172.25.0.0/24）互联。

## 项目结构

```
/opt/fenxiyun/
├── docker/                        # Docker 部署配置
│   ├── docker-compose.yml
│   ├── application.yml            # Spring Boot 生产配置
│   ├── nginx.conf                 # Nginx 虚拟主机
│   ├── .env                       # 环境变量
│   ├── mysql/
│   │   ├── data/                  # MySQL 数据持久化
│   │   └── init/                  # 初始化脚本 -> financial.sql
│   ├── server/
│   │   └── financial.jar          # Spring Boot JAR
│   ├── logs/                      # 应用日志
│   └── nginx/
│       ├── vhost/                 # Nginx 配置目录
│       └── wwwroot/               # 前端静态文件 (index.html)
├── bs-server/                     # 后端源码 (Gradle, Spring Boot 2.1.8)
├── front-end/                     # 前端源码 (Vue2 + HeyUI)
│   └── dist/                      # 构建产物
├── kernel/                        # 核心模块
└── financial.sql                  # 数据库初始化脚本 (24 张表)
```

## 数据库

### 连接方式

- **外部连接**: 47.96.17.173:3307
- **容器内部**: 172.25.0.2:3306
- **数据库**: financial
- **用户/密码**: root / fenxiyun_root_2024

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
# 1. 构建后端 (在本地或服务器)
cd /opt/fenxiyun/bs-server
./gradlew build -x test
cp build/libs/financial-0.1.jar /opt/fenxiyun/docker/server/financial.jar

# 2. 构建前端
cd /opt/fenxiyun/front-end
npm install && npm run build
cp -r dist/* /opt/fenxiyun/docker/nginx/wwwroot/

# 3. 重启后端容器
cd /opt/fenxiyun/docker
docker compose restart client
```

## 数据备份

```bash
# 备份数据库
docker exec financial_mysql8 mysqldump -uroot -pfenxiyun_root_2024 financial > backup_$(date +%Y%m%d).sql

# 备份全部配置
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
rm -rf mysql/data
docker compose up -d
```

## 已知问题

1. **服务器内存有限 (2G)**: npm build 可能 OOM，建议在本地构建前端后上传 dist 目录
2. **JVM 内存限制**: 已设 `-Xms256m -Xmx512m`，避免 OOM
3. **阿里云 SMS 配置**: application.yml 中 aliyun 短信配置为空值，如需使用需填入实际 accessKey
4. **Docker 镜像拉取**: 国内服务器已配置镜像加速器（/etc/docker/daemon.json），如需修改见下方

## Docker 镜像加速器配置

```json
{
  "registry-mirrors": [
    "https://docker.1panel.live",
    "https://hub.rat.dev",
    "https://docker.m.daocloud.io"
  ]
}
```

## 技术栈

- **后端**: Spring Boot 2.1.8 + MyBatis-Plus + MySQL 8 + Redis 5
- **前端**: Vue 2 + HeyUI 组件库
- **认证**: sa-token
- **构建**: Gradle (多模块)
- **部署**: Docker Compose
