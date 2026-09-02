# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

RuoYi Office（宇擎）：由 yudao-cloud 脚手架演进的中小企业全业务办公一体化平台。技术栈：Spring Boot 3.5.9 + Spring Cloud 2025 + MyBatis Plus + Flowable 7（工作流）+ Redis/Redisson + Vue3 + Vben Admin（前端在独立仓库）。JDK 17。

详细模块文档见 `docs/功能模块说明.md`（每个模块的作用、依赖关系），部署见 `docs/后端部署指南.md`。

## 构建与运行（本地开发）

**必须用 `-P boot`（单体模式）构建，建议加 `-Dmaven.test.skip=true` 跳过测试以提速**（测试源码已可正常编译）：
- 默认 cloud profile 会把子模块打成 fat jar，嵌套进 yudao-server 后类不可见，启动报 bean 缺失
- 本机默认 JDK 是 18，需 `export JAVA_HOME=/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home`

```bash
# 全量构建（约 1-2 分钟）
mvn clean install -Dmaven.test.skip=true -P boot

# 增量构建（只改某模块时，秒级）
mvn install -Dmaven.test.skip=true -P boot -pl <module-path> -am

# 重启后端（端口 48080，profile=local）
lsof -tiTCP:48080 -sTCP:LISTEN | xargs kill
cd yudao-server && $JAVA_HOME/bin/java -jar target/yudao-server.jar
```

验证：`curl -X POST http://127.0.0.1:48080/admin-api/system/auth/login -H "Content-Type: application/json" -H "tenant-id: 1" -d '{"username":"admin","password":"admin123"}'` 返回 `code:0`。

本地依赖：MySQL `127.0.0.1:33061`（root/123456，库 `ruoyi-office`）、Redis 6379 无密码、登录账号 admin/admin123、接口需 header `tenant-id: 1`（本地已关验证码）。

前端：独立仓库 `~/Documents/ruoyi-office-vben`（vben5 monorepo），`pnpm run dev:antd` 起在 5666，代理 `/admin-api` → 48080。本仓库 `yudao-ui/` 下只有 README 占位。

运行单个测试：`mvn test -pl <module-path> -am -Dtest=ClassName#methodName`。

Swagger：http://127.0.0.1:48080/swagger-ui/index.html

## 架构

### 双部署模式（核心机制）

根 pom 定义两个 profile，决定同一套代码的两种运行形态：
- **boot（本地/单体）**：`skip.repackage=true`，所有模块聚合进 `yudao-server` 单进程启动；yudao-server 排除了 openfeign 依赖，api 接口直接注入同容器内的 `@RestController` 实现
- **cloud（默认，生产）**：各模块独立 repackage 成可执行 jar，经 `yudao-gateway` + Nacos 注册发现，api 接口走 Feign

`yudao-server` 是空壳聚合器——启用/禁用业务模块靠增删其 pom 中的依赖（默认启用：system、infra、bpm、oa、hrm，其余注释掉以加速编译）。

### 模块结构

每个业务模块拆两个子工程：
- `yudao-module-xxx-api`：跨模块 RPC 接口（`@FeignClient`，boot 模式下作为本地 bean）+ DTO + 枚举 + **该模块的 `ErrorCodeConstants`**
- `yudao-module-xxx-server`：业务实现。包结构约定：`controller/admin/<domain>`（含 `vo/`）、`service/<domain>`、`dal/dataobject/<domain>`、`dal/mysql/<domain>`、`framework/`（模块内安全/RPC 配置）、`enums/`

基础包：`cn.iocoder.yudao.module.<xxx>`。`yudao-framework/` 下是 18 个自研 Starter（web、security、mybatis、redis、mq、tenant、data-permission、excel、rpc 等），`yudao-common-server` 提供跨模块通用服务（如附件 AttachmentService、流程通知监听基类）。

### BPM 流程集成（FlowBill 模式，最重要的跨模块架构）

业务单据（OA 用车/用印、HRM 入职/转正/调动/离职等）与 BPM 审批引擎解耦协作：

1. **提交**：业务 Service 调 `BpmProcessInstanceApi.submitProcessInstance(userId, reqDTO)`，`processDefinitionKey` 来自单据类型枚举（如 `OaBillTypeEnum.OA_CAR_APPLY_BILL.getProcessDefinitionKey()`），`businessKey` = 单据 ID，单据回存 `processInstanceId`
2. **状态回调**：业务 Service 实现 `FlowBillService<XBillTypeEnum>`（`updateProcessStatus` / `onProcessApproved` / `onProcessRejected` / `onProcessCancelled`），注册到模块的 `XFlowBillServiceFactory`
3. **通知通道**（各模块 `process/` 包，三选一按部署模式）：boot 用本地事件监听（继承 `AbstractFlowLocalNotificationListener`）；cloud 用 MQ 消费者（继承 `AbstractFlowMqNotificationConsumer`）或 Feign 回调（如 `OaFeignNotificationApi` → `AbstractFlowNotificationController`）

### 通用代码约定

- Controller：`@PreAuthorize("@ss.hasPermission('oa:car:create')")` 权限格式为 `模块:实体:动作`；统一返回 `CommonResult<T>`，分页 `PageResult<T>`；导出用 `ExcelUtils.write`
- VO/DO 转换用 `BeanUtils.toBean(...)`（框架封装）；VO 三件套：`XxxSaveReqVO`（创建/更新共用）、`XxxPageReqVO`、`XxxRespVO`
- 错误码：每模块 api 子工程的 `ErrorCodeConstants`，分段编号（OA 用 `1-101-xxx-xxx` 段），抛出用 `ServiceExceptionUtil.exception(ERROR_CODE)`
- DO 继承 `BaseDO`（creator/createTime/updater/updateTime/tenantId + 逻辑删除 `deleted`）；Mapper 继承 `BaseMapperX`，条件构造用 `LambdaQueryWrapperX`
- 多租户全局开启（DB/Redis/Web/Security 等 7 层隔离），新表需考虑 `tenant_id`；跳过租户的表加到 `application.yaml` 的 `yudao.tenant.ignore-tables`
- MapStruct + Lombok 已通过 `lombok-mapstruct-binding` 协同（根 pom 已配好，无需额外处理）

## 灵镜测试工作流（Lingjing）

测试/诊断类请求（含自然语言，如"测试一下""启动服务跑个用例"）先读 `.claude/lingjing/CLAUDE.md` 按其路由执行；自然语言请求不构成执行授权，副作用动作须经 policy-guard 确认。

## Git 工作流

- 主分支 `master-jdk17`；功能分支命名 `YY.MM_名字+日期`（如 `26.08_lichunping0817`），完成后发 PR 合并
- 提交信息格式：`feat: ...` / `fix：【模块】...`（中文描述）

## 安全红线

本仓库为**公开仓库**：严禁提交数据库 dump（`sql/mysql/dump/` 已 gitignore）、密码哈希、手机号、内网 IP 等敏感数据；配置文件中的密钥不要在回复或文档中扩散。
