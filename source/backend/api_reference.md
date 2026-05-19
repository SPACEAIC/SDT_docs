# SDT API Reference

本文档面向前端和集成方，整理当前 SDT 工作区的完整接口。

## 入口与状态

默认 HTTP 入口：

```text
https://127.0.0.1:12100
```

当前 Envoy 已接入：

| 模块 | HTTP 前缀 | 后端 | 状态 |
|------|-----------|------|------|
| Auth/Admin | `/v1/auth`, `/v1/admin` | Auth HTTP `12101` | 可用 |
| DB | `/v1/db` | gRPC DB `12000` | 可用 |
| Topology | `/v1/topology` | gRPC Topology `12001` | 可用 |
| Data | `/v1/data` | Data gRPC `12004` | proto 已声明，当前 Envoy 未接入 |
| Scheduler | `/v1/scheduler` | DagSim Scheduler gRPC `50050` | proto 已声明，当前 Envoy 未接入 |

字段命名约定：

- `/v1/auth/*` 和 `/v1/admin/*` 是普通 HTTP 服务，字段保持 snake_case。
- `/v1/db/*` 和 `/v1/topology/*` 经过 Envoy `grpc_json_transcoder`，响应字段默认转成 camelCase；请求体建议使用 snake_case，camelCase 通常也可被 transcoder 接受。
- DB 业务错误通常是 HTTP 200 + `success: false`；Topology 业务错误通常是 HTTP 200 + `status: "error"`。
- `/v1/db/*` 和 `/v1/topology/*` 都需要 `Authorization: Bearer <token>`。用户 ID 由 JWT 注入，不要在请求体里传 `user_id`。

## 能力总览

| 需求 | 当前可用方式 | 说明 |
|------|--------------|------|
| 星座查询 | HTTP `/v1/db/constellations`, `/v1/db/constellation`, `/v1/db/constellation/tles.txt` | 已接入 Envoy |
| 星座创建 | DataService 直连 gRPC `sdt.tle.TleService/GenerateConstellation`；`/v1/data/constellation/generate` 为 proto 预留 HTTP 路径 | 当前未接入 Envoy |
| 星座更新 | DataService 直连 gRPC `sdt.tle.TleService/UpdateTle` 可更新 TLE 数据 | 通用星座元数据 update HTTP API 未实现 |
| 星座删除 | 未实现 | 需要新增 DB/Data API |
| 地面站增删改查 | HTTP `/v1/db/groundstations*` | 已接入 Envoy |
| 仿真配置查询 | Scheduler 直连 gRPC `GetSimulationContext`, `GetDag` | `/v1/scheduler/*` 当前未接入 Envoy |
| 仿真配置写入 | Redis key `simulation:context:{context_id}:yaml` 等输入 key | 当前没有 HTTP 配置 CRUD |
| 仿真创建 | HTTP `/v1/db/simulation/init` | 已接入 Envoy，返回 `contextId` |
| 仿真运行 | Scheduler 直连 gRPC `WatchSchedule` | HTTP 路径 `/v1/scheduler/schedule/watch` 为预留 |
| 仿真监控 | Scheduler 直连 gRPC `GetSchedulerStatus`, `GetSimulationStatus`, `WatchSchedule` | HTTP 路径为预留 |
| 仿真结果查询 | HTTP `/v1/topology/*`、HTTP `/v1/db/simulation/state`、Scheduler gRPC `ListObjects`、Redis 输出 key | 结果来源分两套：DB/Topology 和 DagSim |

## 鉴权

### POST /v1/auth/jwks.json

获取登录加密和 JWT 校验使用的 JWKS 公钥。

鉴权：无。

```bash
curl -ksS -X POST "$BASE/v1/auth/jwks.json"
```

### POST /v1/auth/login

登录并返回 JWT。`password` 需要使用 JWKS 中的 RSA 公钥做 RSA-OAEP/SHA-256 加密后再 base64 传输。

鉴权：无。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| username | string | 是 | 用户名 |
| password | string | 是 | 加密后的密码 |

响应主要字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| access_token | string | JWT |
| token_type | string | `Bearer` |
| expires_at | string | 过期时间 |
| user.id | int | 用户 ID |
| user.username | string | 用户名 |
| user.role | string | `admin` 或 `user` |

### POST /v1/auth/me

查询当前登录用户。

鉴权：Bearer token。

```bash
curl -ksS -X POST "$BASE/v1/auth/me" \
  -H "Authorization: Bearer $TOKEN"
```

## 星座接口

### POST /v1/db/constellations

查询当前用户拥有的星座列表。

状态：可用。

鉴权：Bearer token。

请求：空 body。

响应：

| 字段 | 类型 | 说明 |
|------|------|------|
| success | bool | 是否成功 |
| constellations[].id | int | 星座 ID |
| constellations[].name | string | 星座名称 |
| constellations[].satelliteCount | int | 卫星数量 |
| message | string | 消息 |

```bash
curl -ksS -X POST "$BASE/v1/db/constellations" \
  -H "Authorization: Bearer $TOKEN"
```

### POST /v1/db/constellation

读取单个星座的完整数据，包括元数据、TLE、ISL 和地面站。

状态：可用。

鉴权：Bearer token，星座必须属于当前用户。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| constellation_id | int | 是 | 星座 ID |
| include_tles | bool | 否 | 是否返回 TLE，默认 true |

响应：

| 字段 | 类型 | 说明 |
|------|------|------|
| success | bool | 是否成功 |
| contextId | string | 星座上下文 ID |
| context.constellationId | int | 星座 ID |
| context.constellationName | string | 星座名称 |
| context.satelliteCount | int | 卫星数量 |
| context.islCount | int | ISL 数量 |
| context.groundstationCount | int | 地面站数量 |
| tles[].satelliteId | int | 卫星 ID |
| tles[].tleLine1 | string | TLE Line 1 |
| tles[].tleLine2 | string | TLE Line 2 |
| isls[].satelliteId1 | int | ISL 端点 1 |
| isls[].satelliteId2 | int | ISL 端点 2 |
| groundstations[] | array | 地面站列表 |
| message | string | 消息 |

```bash
curl -ksS -X POST "$BASE/v1/db/constellation" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"constellation_id":1,"include_tles":true}'
```

### POST /v1/db/constellation/tles.txt

导出星座 TLE 文本。

状态：可用。

鉴权：Bearer token，星座必须属于当前用户。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| constellation_id | int | 是 | 星座 ID |

响应：`text/plain; charset=utf-8`，每颗卫星三行：名称行、TLE Line 1、TLE Line 2。

```bash
curl -ksS -X POST "$BASE/v1/db/constellation/tles.txt" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"constellation_id":1}'
```

### sdt.tle.TleService/GenerateConstellation

按模板生成星座并写入数据库。

状态：当前为 DataService 直连 gRPC；`POST /v1/data/constellation/generate` 已在 `SDT_Proto` 声明，但当前 Envoy 未接入 `/v1/data`。

直连地址：通常为 `127.0.0.1:12004`。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| template.constellation_name | string | 是 | 星座名称 |
| template.num_orbits | int | 是 | 轨道面数量 |
| template.satellites_per_orbit | int | 是 | 每轨卫星数量 |
| template.phase_factor | int | 是 | 相位因子 |
| template.inclination_deg | double | 是 | 倾角 |
| template.mean_motion_rev_per_day | double | 是 | 平均运动 |
| template.epoch_utc | string | 是 | 历元 |
| template.eccentricity | double | 是 | 偏心率 |
| template.arg_of_perigee_deg | double | 是 | 近地点幅角 |
| template.isl_config.mode | enum | 否 | `ISL_MODE_NONE` 或 `ISL_MODE_PLUS_GRID` |
| template.overwrite_existing | bool | 否 | 是否覆盖同名星座 |

响应：

| 字段 | 类型 | 说明 |
|------|------|------|
| success | bool | 是否成功 |
| message | string | 消息 |
| constellation.constellation_id | int | 新星座 ID |
| constellation.constellation_name | string | 星座名称 |
| constellation.satellite_count | int | 卫星数量 |
| constellation.isl_count | int | ISL 数量 |
| constellation.epoch_utc | string | 历元 |

示例：

```bash
grpcurl -plaintext \
  -import-path SDT_data_service/tle_service/proto \
  -proto tle_service.proto \
  -d '{
    "template": {
      "constellation_name": "demo-walker",
      "num_orbits": 6,
      "satellites_per_orbit": 12,
      "phase_factor": 1,
      "inclination_deg": 53,
      "mean_motion_rev_per_day": 15.2,
      "epoch_utc": "2026-05-18T00:00:00Z",
      "eccentricity": 0.0001,
      "arg_of_perigee_deg": 0,
      "isl_config": {
        "mode": "ISL_MODE_PLUS_GRID",
        "isl_shift": 1,
        "include_same_orbit_links": true,
        "include_adjacent_orbit_links": true
      },
      "overwrite_existing": false
    }
  }' \
  127.0.0.1:12004 \
  sdt.tle.TleService/GenerateConstellation
```

### sdt.tle.TleService/UpdateTle

更新指定星座的 TLE 数据。`constellation_names` 为空时更新全部。

状态：当前为 DataService 直连 gRPC。

```bash
grpcurl -plaintext \
  -import-path SDT_data_service/tle_service/proto \
  -proto tle_service.proto \
  -d '{"constellation_names":["starlink"]}' \
  127.0.0.1:12004 \
  sdt.tle.TleService/UpdateTle
```

### 星座 Update/Delete 状态

当前没有已接入 Envoy 的星座元数据更新和删除接口，也没有通用的 HTTP `DELETE constellation` 接口。若前端需要完整 CRUD，建议后端补充：

| 建议接口 | 方法 | 说明 |
|----------|------|------|
| `/v1/db/constellation/update` | POST | 更新名称、模板参数或归属信息 |
| `/v1/db/constellation/delete` | POST | 删除星座及其 TLE/ISL/仿真缓存 |

## 地面站接口

### POST /v1/db/groundstations

查询当前用户的地面站列表。

状态：可用。

鉴权：Bearer token。

请求：空 body。

响应：

| 字段 | 类型 | 说明 |
|------|------|------|
| success | bool | 是否成功 |
| groundstations[].id | int | 地面站 ID |
| groundstations[].name | string | 名称 |
| groundstations[].latitude | double | 纬度 |
| groundstations[].longitude | double | 经度 |
| message | string | 消息 |

```bash
curl -ksS -X POST "$BASE/v1/db/groundstations" \
  -H "Authorization: Bearer $TOKEN"
```

### POST /v1/db/groundstations/add

新增地面站。

状态：可用。

鉴权：Bearer token。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| name | string | 是 | 名称 |
| latitude | double | 是 | 纬度 |
| longitude | double | 是 | 经度 |

响应：

| 字段 | 类型 | 说明 |
|------|------|------|
| success | bool | 是否成功 |
| id | int | 新地面站 ID |
| message | string | 消息 |

```bash
curl -ksS -X POST "$BASE/v1/db/groundstations/add" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Shanghai","latitude":31.23,"longitude":121.47}'
```

### POST /v1/db/groundstations/update

更新地面站。

状态：可用。

鉴权：Bearer token，地面站必须属于当前用户。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| groundstation.id | int | 是 | 地面站 ID |
| groundstation.name | string | 是 | 名称 |
| groundstation.latitude | double | 是 | 纬度 |
| groundstation.longitude | double | 是 | 经度 |

```bash
curl -ksS -X POST "$BASE/v1/db/groundstations/update" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"groundstation":{"id":1,"name":"Beijing","latitude":39.9,"longitude":116.4}}'
```

### POST /v1/db/groundstations/delete

删除地面站。

状态：可用。

鉴权：Bearer token，地面站必须属于当前用户。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | int | 是 | 地面站 ID |

响应：不存在或不属于当前用户时 `success=false`。

```bash
curl -ksS -X POST "$BASE/v1/db/groundstations/delete" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"id":1}'
```

### POST /v1/db/groundstations/getbyname

按名称查询地面站。

状态：可用。

鉴权：Bearer token。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| name | string | 是 | 地面站名称 |

响应：未找到时 `success=false`。

```bash
curl -ksS -X POST "$BASE/v1/db/groundstations/getbyname" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Shanghai"}'
```

## 仿真配置

### Redis 配置输入

DagSim Scheduler 运行前会从 Redis 读取配置和原始输入：

```text
simulation:context:{context_id}:yaml
simulation:context:{context_id}:raw_input:tle
simulation:context:{context_id}:raw_input:isls
simulation:context:{context_id}:raw_input:ground_stations
simulation:context:{context_id}:raw_input:meta
simulation:context:{context_id}:raw_input:sdt_config
```

`yaml` 是 DagSim 的完整配置，包含：

| 字段 | 说明 |
|------|------|
| services | worker 服务列表 |
| pipeline | bootstrap/step 阶段依赖 DAG |
| simulation.time_origin_ns | 仿真起点，Unix 纳秒 |
| simulation.duration_s | 仿真总时长 |
| simulation.step_s | 仿真步长 |
| object_store | 对象快照配置 |

`raw_input:meta` / `raw_input:sdt_config` 常用字段：

| 字段 | 说明 |
|------|------|
| max_gsl_distance_m | 星地链路最大距离 |
| max_isl_distance_m | 星间链路最大距离 |
| src_gs_id | 最短路径源地面站 ID |
| dst_gs_id | 最短路径目标地面站 ID |

写入示例：

```bash
redis-cli -h 127.0.0.1 -p 6379 -x SET \
  "simulation:context:sdt_topology:yaml" \
  < dag_sim/ray_core/config/sdt_topology.yaml

redis-cli -h 127.0.0.1 -p 6379 HSET \
  "simulation:context:sdt_topology:raw_input:sdt_config" \
  src_gs_id 0 \
  dst_gs_id 9 \
  max_gsl_distance_m 5089686.4181956202 \
  max_isl_distance_m 5016591.2330984278
```

### scheduler.v1.SchedulerService/GetSimulationContext

查询解析后的仿真配置。

状态：当前为 Scheduler 直连 gRPC；`POST /v1/scheduler/simulations/context` 是 proto 预留 HTTP 路径，当前 Envoy 未接入。

直连地址：`127.0.0.1:50050`。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| context_id | string | 是 | 仿真上下文 ID |

响应：

| 字段 | 类型 | 说明 |
|------|------|------|
| success | bool | 是否成功 |
| context_id | string | 上下文 ID |
| services[] | array | worker 服务配置 |
| pipeline[] | array | stage DAG 配置 |
| simulation | object | 时间配置 |
| object_store | object | 对象存储配置 |

```bash
grpcurl -plaintext \
  -import-path dag_sim/ray_core/proto \
  -proto scheduler/v1/scheduler.proto \
  -d '{"context_id":"sdt_topology"}' \
  127.0.0.1:50050 \
  scheduler.v1.SchedulerService/GetSimulationContext
```

### scheduler.v1.SchedulerService/GetDag

查询 bootstrap 和 step 两张 DAG，用于前端可视化。

状态：当前为 Scheduler 直连 gRPC；`POST /v1/scheduler/simulations/dag` 是 proto 预留 HTTP 路径，当前 Envoy 未接入。

```bash
grpcurl -plaintext \
  -import-path dag_sim/ray_core/proto \
  -proto scheduler/v1/scheduler.proto \
  -d '{"context_id":"sdt_topology"}' \
  127.0.0.1:50050 \
  scheduler.v1.SchedulerService/GetDag
```

### 配置 CRUD 状态

当前没有已接入 Envoy 的仿真配置新增、修改、删除 HTTP API。若需要前端直接管理配置，建议补充：

| 建议接口 | 方法 | 说明 |
|----------|------|------|
| `/v1/sim/configs` | POST | 创建或保存配置 YAML |
| `/v1/sim/configs/list` | POST | 查询当前用户配置列表 |
| `/v1/sim/configs/get` | POST | 读取配置详情 |
| `/v1/sim/configs/delete` | POST | 删除配置 |

## 仿真创建

### POST /v1/db/simulation/init

初始化仿真缓存，生成或复用 `contextId`。该接口会把星座数据写入 Redis，供 Topology 服务使用。

状态：可用。

鉴权：Bearer token，星座必须属于当前用户。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| constellation_id | int | 是 | 星座 ID |
| context_id | string | 否 | 指定 context ID；为空时服务端生成 |
| force_refresh | bool | 否 | 是否强制重建缓存 |
| ttl_seconds | int | 否 | 缓存 TTL；0 表示服务端默认 |

响应：

| 字段 | 类型 | 说明 |
|------|------|------|
| success | bool | 是否成功 |
| contextId | string | 仿真上下文 ID |
| cacheHit | bool | 是否命中已有缓存 |
| message | string | 消息 |
| metaKey | string | Redis meta key |

```bash
CONTEXT_ID=$(curl -ksS -X POST "$BASE/v1/db/simulation/init" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"constellation_id":1,"force_refresh":false}' | jq -r .contextId)
```

## 仿真运行

### scheduler.v1.SchedulerService/WatchSchedule

触发一次 DagSim 仿真，并以服务端流返回运行事件。

状态：当前为 Scheduler 直连 gRPC；`POST /v1/scheduler/schedule/watch` 是 proto 预留 HTTP 路径，当前 Envoy 未接入。

直连地址：`127.0.0.1:50050`。

鉴权：当前直连 gRPC 无鉴权；接入 Envoy 后建议使用 Bearer token。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| context_id | string | 是 | 仿真上下文 ID |

响应流 `ScheduleEvent`：

| 字段 | 类型 | 说明 |
|------|------|------|
| type | enum | 事件类型 |
| wall_time_ms | int64 | 服务端 wall clock |
| step_index | int | step 序号，`-1` 表示 bootstrap |
| sim_time_s | double | 当前仿真时间 |
| stage_name | string | stage 名称 |
| elapsed_ms | double | 事件耗时 |
| error | string | 错误信息 |
| object | object | 对象快照元数据 |

关键事件：

| 枚举 | 说明 |
|------|------|
| EVENT_TYPE_SIMULATION_STARTED | 仿真开始 |
| EVENT_TYPE_BOOTSTRAP_STARTED | bootstrap 开始 |
| EVENT_TYPE_STEP_STARTED | step 开始 |
| EVENT_TYPE_STAGE_LAUNCHED | stage 启动 |
| EVENT_TYPE_STAGE_COMPLETED | stage 完成 |
| EVENT_TYPE_STEP_COMPLETED | step 完成 |
| EVENT_TYPE_SIMULATION_COMPLETED | 仿真成功结束 |
| EVENT_TYPE_SIMULATION_FAILED | 仿真失败结束 |

```bash
grpcurl -plaintext \
  -import-path dag_sim/ray_core/proto \
  -proto scheduler/v1/scheduler.proto \
  -d '{"context_id":"sdt_topology"}' \
  127.0.0.1:50050 \
  scheduler.v1.SchedulerService/WatchSchedule
```

客户端读到 `EVENT_TYPE_SIMULATION_COMPLETED` 或 `EVENT_TYPE_SIMULATION_FAILED` 后，本次仿真结束。

## 仿真监控

### scheduler.v1.SchedulerService/GetHealth

Scheduler 健康检查。

状态：当前为 Scheduler 直连 gRPC；`POST /v1/scheduler/health` 是 proto 预留 HTTP 路径，当前 Envoy 未接入。

```bash
grpcurl -plaintext \
  -import-path dag_sim/ray_core/proto \
  -proto scheduler/v1/scheduler.proto \
  -d '{}' \
  127.0.0.1:50050 \
  scheduler.v1.SchedulerService/GetHealth
```

### scheduler.v1.SchedulerService/GetSchedulerStatus

查询调度器实例状态。

状态：当前为 Scheduler 直连 gRPC；`POST /v1/scheduler/status` 是 proto 预留 HTTP 路径，当前 Envoy 未接入。

响应：

| 字段 | 类型 | 说明 |
|------|------|------|
| success | bool | 是否成功 |
| version | string | 调度器版本 |
| uptime_s | int64 | 运行时长 |
| schedule_count_total | int64 | 累计调度次数 |
| schedule_count_active | int64 | 当前运行数量 |
| active_context_ids[] | array | 运行中的 context |

```bash
grpcurl -plaintext \
  -import-path dag_sim/ray_core/proto \
  -proto scheduler/v1/scheduler.proto \
  -d '{}' \
  127.0.0.1:50050 \
  scheduler.v1.SchedulerService/GetSchedulerStatus
```

### scheduler.v1.SchedulerService/GetSimulationStatus

查询指定 context 的实时仿真状态。

状态：当前为 Scheduler 直连 gRPC；`POST /v1/scheduler/simulations/status` 是 proto 预留 HTTP 路径，当前 Envoy 未接入。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| context_id | string | 是 | 仿真上下文 ID |

响应：

| 字段 | 类型 | 说明 |
|------|------|------|
| success | bool | 是否成功 |
| running | bool | 是否正在运行 |
| current_step | int | 当前 step |
| total_steps | int | 总 step |
| current_sim_time_s | double | 当前仿真时间 |
| elapsed_s | double | 已运行秒数 |
| stages[] | array | stage 状态 |
| last_error | string | 最近错误 |

```bash
grpcurl -plaintext \
  -import-path dag_sim/ray_core/proto \
  -proto scheduler/v1/scheduler.proto \
  -d '{"context_id":"sdt_topology"}' \
  127.0.0.1:50050 \
  scheduler.v1.SchedulerService/GetSimulationStatus
```

## 仿真结果查询

### POST /v1/topology/graph

查询指定 context 和时刻的拓扑图。

状态：可用。

鉴权：Bearer token，context 必须属于当前用户。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| context_id | string | 是 | `/v1/db/simulation/init` 返回的 `contextId` |
| time_ns | int64 | 是 | Unix 纳秒时间戳 |

响应：

| 字段 | 类型 | 说明 |
|------|------|------|
| status | string | `success` 或 `error` |
| message | string | 消息，成功时包含性能指标 |
| nodes[].nodeId | int | 节点 ID |
| nodes[].attributes | map | 节点属性 |
| edges[].sourceId | int | 源节点 |
| edges[].targetId | int | 目标节点 |
| edges[].attributes | map | 边属性 |

```bash
curl -ksS -X POST "$BASE/v1/topology/graph" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"context_id":"'"$CONTEXT_ID"'","time_ns":1700000000000000000}'
```

### POST /v1/topology/path

查询两个地面站之间的最短路径。

状态：可用。当前测试数据集可能返回 HTTP 200 + `status=error,message=no path`，这表示 RPC 正常但业务上没有路径。

鉴权：Bearer token，context 必须属于当前用户。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| context_id | string | 是 | 仿真上下文 ID |
| time_ns | int64 | 是 | Unix 纳秒时间戳 |
| src_gs_id | int | 是 | 源地面站 ID |
| dst_gs_id | int | 是 | 目标地面站 ID |

响应：

| 字段 | 类型 | 说明 |
|------|------|------|
| status | string | `success` 或 `error` |
| pathNodeIds[] | array | 路径节点 |
| pathEdges[] | array | 路径边 |
| totalDistanceM | double | 总距离 |
| message | string | 消息 |

```bash
curl -ksS -X POST "$BASE/v1/topology/path" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"context_id":"'"$CONTEXT_ID"'","time_ns":1700000000000000000,"src_gs_id":1,"dst_gs_id":2}'
```

### POST /v1/topology/tle

查询 context 内的 TLE。

状态：可用。

鉴权：Bearer token。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| context_id | string | 是 | 仿真上下文 ID |
| time_ns | int64 | 是 | Unix 纳秒时间戳 |

### POST /v1/topology/isls

查询 context 内的 ISL。

状态：可用。

鉴权：Bearer token。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| context_id | string | 是 | 仿真上下文 ID |
| time_ns | int64 | 是 | Unix 纳秒时间戳 |

### POST /v1/db/simulation/state

查询保存在 DB 中的仿真状态。

状态：可用。

鉴权：Bearer token，context 必须属于当前用户。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| context_id | string | 是 | 仿真上下文 ID |
| timestamp | int64 | 否 | 查询时间戳 |

响应：

| 字段 | 类型 | 说明 |
|------|------|------|
| success | bool | 是否成功 |
| state.contextId | string | 仿真上下文 ID |
| state.timestamp | int64 | 时间戳 |
| state.satellitePositions | string | JSON 字符串 |
| state.linkStates | string | JSON 字符串 |
| state.visibilityMatrix | string | JSON 字符串 |
| state.metrics | string | JSON 字符串 |

### POST /v1/db/simulation/state/save

保存仿真状态到 DB。

状态：可用。

鉴权：Bearer token，context 必须属于当前用户。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| state.context_id | string | 是 | 仿真上下文 ID |
| state.timestamp | int64 | 是 | 时间戳 |
| state.satellite_positions | string | 否 | JSON 字符串 |
| state.link_states | string | 否 | JSON 字符串 |
| state.visibility_matrix | string | 否 | JSON 字符串 |
| state.metrics | string | 否 | JSON 字符串 |

```bash
curl -ksS -X POST "$BASE/v1/db/simulation/state/save" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"state":{"context_id":"'"$CONTEXT_ID"'","timestamp":1700000000000000000}}'
```

### scheduler.v1.SchedulerService/ListObjects

查询对象快照元数据。

状态：当前为 Scheduler 直连 gRPC；`POST /v1/scheduler/objects` 是 proto 预留 HTTP 路径，当前 Envoy 未接入。

请求：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| context_id | string | 是 | 仿真上下文 ID |
| limit | int | 否 | 返回上限 |
| step_index_min | int | 否 | 最小 step，`-1` 表示不过滤 |
| step_index_max | int | 否 | 最大 step，`-1` 表示不过滤 |

```bash
grpcurl -plaintext \
  -import-path dag_sim/ray_core/proto \
  -proto scheduler/v1/scheduler.proto \
  -d '{"context_id":"sdt_topology","limit":20,"step_index_min":-1,"step_index_max":-1}' \
  127.0.0.1:50050 \
  scheduler.v1.SchedulerService/ListObjects
```

### DagSim Redis 输出

DagSim 完整结果从 Redis 读取。动态结果使用 `latest:message:*`，静态结果使用 `static:message:*`。

主要 key：

| 结果 | Redis key |
|------|-----------|
| 地面站目录 | `simulation:context:{context_id}:static:message:ground_stations` |
| 卫星位置/速度 | `simulation:context:{context_id}:latest:message:spacecraft_states` |
| 环境访问关系 | `simulation:context:{context_id}:latest:message:environment_states` |
| 拓扑图 | `simulation:context:{context_id}:latest:message:topology_graph` |
| 最短路径 | `simulation:context:{context_id}:latest:message:topology_path` |
| 地面站可见卫星 | `simulation:context:{context_id}:latest:message:satellite_visibility` |
| 星间链路 | `simulation:context:{context_id}:latest:message:inter_satellite_links` |
| 星地链路 | `simulation:context:{context_id}:latest:message:ground_satellite_links` |

读取示例：

```bash
redis-cli -h 127.0.0.1 -p 6379 --raw GET \
  "simulation:context:sdt_topology:latest:message:topology_path" | jq .
```

统一 envelope：

```json
{
  "header": {
    "message_type": "topology_path",
    "step_index": 2,
    "sim_time_ns": 1742394724000000000
  },
  "body": {}
}
```

## 典型调用流程

### HTTP Topology 流程

```bash
BASE=https://127.0.0.1:12100

# 1. 登录获取 TOKEN，password 需先在客户端加密
TOKEN=$(curl -ksS -X POST "$BASE/v1/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"<encrypted_password>"}' | jq -r .access_token)

# 2. 查询星座
curl -ksS -X POST "$BASE/v1/db/constellations" \
  -H "Authorization: Bearer $TOKEN"

# 3. 创建仿真 context
CONTEXT_ID=$(curl -ksS -X POST "$BASE/v1/db/simulation/init" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"constellation_id":1,"force_refresh":false}' | jq -r .contextId)

# 4. 查询拓扑图
curl -ksS -X POST "$BASE/v1/topology/graph" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"context_id":"'"$CONTEXT_ID"'","time_ns":1700000000000000000}'
```

### DagSim 运行流程

```bash
# 1. 写入配置和输入
redis-cli -h 127.0.0.1 -p 6379 -x SET \
  "simulation:context:sdt_topology:yaml" \
  < dag_sim/ray_core/config/sdt_topology.yaml

redis-cli -h 127.0.0.1 -p 6379 -x SET \
  "simulation:context:sdt_topology:raw_input:tle" \
  < dag_sim/tests/example_inputs/tles.txt

redis-cli -h 127.0.0.1 -p 6379 -x SET \
  "simulation:context:sdt_topology:raw_input:isls" \
  < dag_sim/tests/example_inputs/isls.txt

redis-cli -h 127.0.0.1 -p 6379 -x SET \
  "simulation:context:sdt_topology:raw_input:ground_stations" \
  < dag_sim/tests/example_inputs/ground_stations.txt

# 2. 触发仿真
grpcurl -plaintext \
  -import-path dag_sim/ray_core/proto \
  -proto scheduler/v1/scheduler.proto \
  -d '{"context_id":"sdt_topology"}' \
  127.0.0.1:50050 \
  scheduler.v1.SchedulerService/WatchSchedule

# 3. 查询结果
redis-cli -h 127.0.0.1 -p 6379 --raw GET \
  "simulation:context:sdt_topology:latest:message:topology_graph" | jq .
```

## 前端接入建议

当前前端可以直接使用这些 HTTP 接口：

| 功能 | 接口 |
|------|------|
| 登录 | `/v1/auth/jwks.json`, `/v1/auth/login`, `/v1/auth/me` |
| 星座列表/详情 | `/v1/db/constellations`, `/v1/db/constellation` |
| 地面站 CRUD | `/v1/db/groundstations*` |
| 创建仿真 context | `/v1/db/simulation/init` |
| 查询拓扑结果 | `/v1/topology/graph`, `/v1/topology/path`, `/v1/topology/tle`, `/v1/topology/isls` |
| 保存/查询 DB 仿真状态 | `/v1/db/simulation/state`, `/v1/db/simulation/state/save` |

如果前端需要直接调用星座创建、仿真运行、仿真监控和 DagSim 对象查询，需要先把 `/v1/data/*` 和 `/v1/scheduler/*` 接入 Envoy，或者增加一个 HTTP gateway。
