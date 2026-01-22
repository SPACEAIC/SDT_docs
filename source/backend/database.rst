====================================
Database gRPC API 接口文档
====================================

.. contents:: 目录
   :depth: 3
   :local:

概述
====

基于 gRPC 的卫星星座管理系统，提供用户认证、基座管理、星座管理和卫星管理等功能。

服务信息
--------

:服务地址: ``localhost:50051``
:协议: gRPC (HTTP/2)
:序列化: Protocol Buffers (proto3)
:认证方式: User ID (用户ID)

快速开始
--------

Python 客户端连接示例：

.. code-block:: python

    import grpc
    from grpc_generated import auth_pb2, auth_pb2_grpc

    # 创建 channel
    channel = grpc.insecure_channel('localhost:50051')

    # 创建客户端
    auth_stub = auth_pb2_grpc.AuthServiceStub(channel)

    # 调用接口
    response = auth_stub.Login(auth_pb2.LoginRequest(
        username="user",
        password="pass"
    ))



服务定义
--------

.. code-block:: protobuf

    service AuthService {
        rpc Register(RegisterRequest) returns (RegisterResponse);
        rpc Login(LoginRequest) returns (LoginResponse);
        rpc GetCurrentUser(GetCurrentUserRequest) returns (GetCurrentUserResponse);
    }

Register (用户注册)
-------------------

创建新用户账户。

**请求消息：**

.. code-block:: protobuf

    message RegisterRequest {
        string username = 1;  // 用户名，必填，唯一
        string password = 2;  // 密码，必填
    }

**响应消息：**

.. code-block:: protobuf

    message RegisterResponse {
        Status status = 1;    // 响应状态
        User user = 2;        // 创建的用户信息
    }

**调用示例：**

.. code-block:: python

    response = auth_stub.Register(auth_pb2.RegisterRequest(
        username="newuser",
        password="securepass123"
    ))

    if response.status.code == 0:
        print(f"注册成功: {response.user.username}")
    else:
        print(f"注册失败: {response.status.message}")

**错误情况：**

- ``409 CONFLICT``: 用户名已存在
- ``400 BAD_REQUEST``: 用户名或密码格式不正确

Login (用户登录)
----------------

用户登录，验证用户名和密码，返回用户信息。

**请求消息：**

.. code-block:: protobuf

    message LoginRequest {
        string username = 1;  // 用户名
        string password = 2;  // 密码
    }

**响应消息：**

.. code-block:: protobuf

    message LoginResponse {
        Status status = 1;    // 响应状态
        User user = 2;        // 用户信息（包含用户ID）
        string user_id = 3;   // 已废弃，请使用 user.id
    }

**调用示例：**

.. code-block:: python

    response = auth_stub.Login(auth_pb2.LoginRequest(
        username="testuser",
        password="password123"
    ))

    if response.status.code == 0:
        user_id = response.user.id  # 获取用户ID
        print(f"登录成功，用户ID: {user_id}, 用户名: {response.user.username}")
    else:
        print(f"登录失败: {response.status.message}")

**错误情况：**

- ``401 UNAUTHORIZED``: 用户名或密码错误
- ``404 NOT_FOUND``: 用户不存在

GetCurrentUser (获取当前用户)
------------------------------

根据用户ID获取当前登录用户信息。

**请求消息：**

.. code-block:: protobuf

    message GetCurrentUserRequest {
        string user_id = 1;  // 用户ID（字符串格式的整数）
    }

**响应消息：**

.. code-block:: protobuf

    message GetCurrentUserResponse {
        Status status = 1;   // 响应状态
        User user = 2;       // 用户信息
    }

**调用示例：**

.. code-block:: python

    response = auth_stub.GetCurrentUser(auth_pb2.GetCurrentUserRequest(
        user_id=str(user_id)
    ))

    if response.status.code == 0:
        print(f"当前用户: {response.user.username}")

**错误情况：**

- ``401 UNAUTHORIZED``: 用户ID无效或用户不存在

BaseService (基座管理服务)
==========================

基座（Base）是卫星地面站的抽象，用于管理地面基础设施。

服务定义
--------

.. code-block:: protobuf

    service BaseService {
        rpc ListBases(ListBasesRequest) returns (ListBasesResponse);
        rpc GetBase(GetBaseRequest) returns (GetBaseResponse);
        rpc CreateBase(CreateBaseRequest) returns (CreateBaseResponse);
        rpc UpdateBase(UpdateBaseRequest) returns (UpdateBaseResponse);
        rpc DeleteBase(DeleteBaseRequest) returns (DeleteBaseResponse);
    }

Base (基座消息)
---------------

.. code-block:: protobuf

    message Base {
        int32 id = 1;            // 基座ID
        string base_name = 2;    // 基座名称
        string info = 3;         // 基座信息描述
        int32 user_id = 4;       // 所属用户ID
    }

ListBases (获取基座列表)
------------------------

获取当前用户的所有基座。

**请求消息：**

.. code-block:: protobuf

    message ListBasesRequest {
        string user_id = 1;  // 用户ID（字符串格式）
    }

**响应消息：**

.. code-block:: protobuf

    message ListBasesResponse {
        Status status = 1;      // 响应状态
        repeated Base bases = 2;  // 基座列表
    }

**调用示例：**

.. code-block:: python

    response = base_stub.ListBases(base_pb2.ListBasesRequest(
        user_id=user_id
    ))

    for base in response.bases:
        print(f"基座: {base.base_name} - {base.info}")

GetBase (获取基座详情)
----------------------

根据ID获取特定基座的详细信息。

**请求消息：**

.. code-block:: protobuf

    message GetBaseRequest {
        string user_id = 1;  // 用户ID（字符串格式）
        int32 base_id = 2;   // 基座ID
    }

**响应消息：**

.. code-block:: protobuf

    message GetBaseResponse {
        Status status = 1;  // 响应状态
        Base base = 2;      // 基座信息
    }

**调用示例：**

.. code-block:: python

    response = base_stub.GetBase(base_pb2.GetBaseRequest(
        user_id=user_id,
        base_id=1
    ))

    if response.status.code == 0:
        base = response.base
        print(f"基座详情: {base.base_name}")

**错误情况：**

- ``404 NOT_FOUND``: 基座不存在
- ``403 FORBIDDEN``: 无权访问该基座

CreateBase (创建基座)
---------------------

创建新的基座。

**请求消息：**

.. code-block:: protobuf

    message CreateBaseRequest {
        string user_id = 1;     // JWT Token
        string base_name = 2;   // 基座名称，必填
        string info = 3;        // 基座信息，可选
    }

**响应消息：**

.. code-block:: protobuf

    message CreateBaseResponse {
        Status status = 1;  // 响应状态
        Base base = 2;      // 创建的基座信息
    }

**调用示例：**

.. code-block:: python

    response = base_stub.CreateBase(base_pb2.CreateBaseRequest(
        user_id=user_id,
        base_name="北京基站",
        info="位于北京的卫星地面站"
    ))

    if response.status.code == 0:
        print(f"创建成功，基座ID: {response.base.id}")

**错误情况：**

- ``400 BAD_REQUEST``: 参数错误（如基座名称为空）

UpdateBase (更新基座)
---------------------

更新现有基座的信息。

**请求消息：**

.. code-block:: protobuf

    message UpdateBaseRequest {
        string user_id = 1;     // JWT Token
        int32 base_id = 2;      // 基座ID
        string base_name = 3;   // 新的基座名称
        string info = 4;        // 新的基座信息
    }

**响应消息：**

.. code-block:: protobuf

    message UpdateBaseResponse {
        Status status = 1;  // 响应状态
        Base base = 2;      // 更新后的基座信息
    }

**调用示例：**

.. code-block:: python

    response = base_stub.UpdateBase(base_pb2.UpdateBaseRequest(
        user_id=user_id,
        base_id=1,
        base_name="北京基站（更新）",
        info="更新后的信息"
    ))

**错误情况：**

- ``404 NOT_FOUND``: 基座不存在
- ``403 FORBIDDEN``: 无权修改该基座

DeleteBase (删除基座)
---------------------

删除指定的基座。

**请求消息：**

.. code-block:: protobuf

    message DeleteBaseRequest {
        string user_id = 1;  // 用户ID（字符串格式）
        int32 base_id = 2;   // 基座ID
    }

**响应消息：**

.. code-block:: protobuf

    message DeleteBaseResponse {
        Status status = 1;  // 响应状态
    }

**调用示例：**

.. code-block:: python

    response = base_stub.DeleteBase(base_pb2.DeleteBaseRequest(
        user_id=user_id,
        base_id=1
    ))

    if response.status.code == 0:
        print("删除成功")

**错误情况：**

- ``404 NOT_FOUND``: 基座不存在
- ``403 FORBIDDEN``: 无权删除该基座

ConstellationService (星座管理服务)
====================================

星座（Constellation）是卫星的集合，用于组织和管理一组相关的卫星。

服务定义
--------

.. code-block:: protobuf

    service ConstellationService {
        rpc ListConstellations(ListConstellationsRequest) returns (ListConstellationsResponse);
        rpc GetConstellation(GetConstellationRequest) returns (GetConstellationResponse);
        rpc CreateConstellation(CreateConstellationRequest) returns (CreateConstellationResponse);
        rpc UpdateConstellation(UpdateConstellationRequest) returns (UpdateConstellationResponse);
        rpc DeleteConstellation(DeleteConstellationRequest) returns (DeleteConstellationResponse);
        rpc ImportSatellites(stream ImportSatellitesRequest) returns (ImportSatellitesResponse);
        rpc ExportConstellations(ExportConstellationsRequest) returns (ExportConstellationsResponse);
    }

Constellation (星座消息)
------------------------

.. code-block:: protobuf

    message Constellation {
        int32 id = 1;                      // 星座ID
        string constellation_name = 2;     // 星座名称
        int32 satellite_count = 3;         // 卫星数量
        int32 user_id = 4;                 // 所属用户ID
        string description = 5;            // 星座描述
    }

SatelliteInfo (卫星简要信息)
----------------------------

.. code-block:: protobuf

    message SatelliteInfo {
        int32 id = 1;                  // 数据库记录ID
        int32 satellite_id = 2;        // 卫星ID
        int32 constellation_id = 3;    // 所属星座ID
        string info_line1 = 4;         // TLE第一行
        string info_line2 = 5;         // TLE第二行
    }

ListConstellations (获取星座列表)
---------------------------------

获取当前用户的所有星座。

**请求消息：**

.. code-block:: protobuf

    message ListConstellationsRequest {
        string user_id = 1;  // 用户ID（字符串格式）
    }

**响应消息：**

.. code-block:: protobuf

    message ListConstellationsResponse {
        Status status = 1;                       // 响应状态
        repeated Constellation constellations = 2;  // 星座列表
    }

**调用示例：**

.. code-block:: python

    response = constellation_stub.ListConstellations(
        constellation_pb2.ListConstellationsRequest(user_id=user_id)
    )

    for constellation in response.constellations:
        print(f"星座: {constellation.constellation_name}, "
              f"卫星数: {constellation.satellite_count}")

GetConstellation (获取星座详情)
-------------------------------

获取星座的详细信息，包括其包含的卫星列表（支持分页）。

**请求消息：**

.. code-block:: protobuf

    message GetConstellationRequest {
        string user_id = 1;              // JWT Token
        int32 constellation_id = 2;      // 星座ID
        PaginationRequest pagination = 3;  // 分页参数（可选）
    }

**响应消息：**

.. code-block:: protobuf

    message GetConstellationResponse {
        Status status = 1;                    // 响应状态
        Constellation constellation = 2;       // 星座信息
        repeated SatelliteInfo satellites = 3; // 卫星列表
        PaginationResponse pagination = 4;     // 分页信息
    }

**调用示例：**

.. code-block:: python

    response = constellation_stub.GetConstellation(
        constellation_pb2.GetConstellationRequest(
            user_id=user_id,
            constellation_id=1,
            pagination=common_pb2.PaginationRequest(page=1, per_page=20)
        )
    )

    constellation = response.constellation
    print(f"星座: {constellation.constellation_name}")
    print(f"描述: {constellation.description}")
    print(f"包含卫星:")
    for sat in response.satellites:
        print(f"  - 卫星ID: {sat.satellite_id}")

**错误情况：**

- ``404 NOT_FOUND``: 星座不存在
- ``403 FORBIDDEN``: 无权访问该星座

CreateConstellation (创建星座)
------------------------------

创建新的星座。

**请求消息：**

.. code-block:: protobuf

    message CreateConstellationRequest {
        string user_id = 1;            // JWT Token
        string constellation_name = 2;  // 星座名称，必填
        string description = 3;         // 星座描述，可选
    }

**响应消息：**

.. code-block:: protobuf

    message CreateConstellationResponse {
        Status status = 1;          // 响应状态
        Constellation constellation = 2;  // 创建的星座信息
    }

**调用示例：**

.. code-block:: python

    response = constellation_stub.CreateConstellation(
        constellation_pb2.CreateConstellationRequest(
            user_id=user_id,
            constellation_name="北斗卫星系统",
            description="中国自主研发的全球卫星导航系统"
        )
    )

    if response.status.code == 0:
        print(f"创建成功，星座ID: {response.constellation.id}")

**错误情况：**

- ``400 BAD_REQUEST``: 参数错误（如星座名称为空）

UpdateConstellation (更新星座)
------------------------------

更新现有星座的信息。

**请求消息：**

.. code-block:: protobuf

    message UpdateConstellationRequest {
        string user_id = 1;             // JWT Token
        int32 constellation_id = 2;     // 星座ID
        string constellation_name = 3;  // 新的星座名称
        string description = 4;         // 新的星座描述
    }

**响应消息：**

.. code-block:: protobuf

    message UpdateConstellationResponse {
        Status status = 1;          // 响应状态
        Constellation constellation = 2;  // 更新后的星座信息
    }

**调用示例：**

.. code-block:: python

    response = constellation_stub.UpdateConstellation(
        constellation_pb2.UpdateConstellationRequest(
            user_id=user_id,
            constellation_id=1,
            constellation_name="北斗卫星系统（更新）",
            description="更新后的描述"
        )
    )

**错误情况：**

- ``404 NOT_FOUND``: 星座不存在
- ``403 FORBIDDEN``: 无权修改该星座

DeleteConstellation (删除星座)
------------------------------

删除指定的星座及其所有关联的卫星。

**请求消息：**

.. code-block:: protobuf

    message DeleteConstellationRequest {
        string user_id = 1;          // JWT Token
        int32 constellation_id = 2;  // 星座ID
    }

**响应消息：**

.. code-block:: protobuf

    message DeleteConstellationResponse {
        Status status = 1;  // 响应状态
    }

**调用示例：**

.. code-block:: python

    response = constellation_stub.DeleteConstellation(
        constellation_pb2.DeleteConstellationRequest(
            user_id=user_id,
            constellation_id=1
        )
    )

    if response.status.code == 0:
        print("删除成功")

**注意事项：**

- 删除星座会同时删除该星座下的所有卫星
- 此操作不可逆，请谨慎使用

**错误情况：**

- ``404 NOT_FOUND``: 星座不存在
- ``403 FORBIDDEN``: 无权删除该星座

ImportSatellites (批量导入卫星)
-------------------------------

使用客户端流式传输批量导入卫星数据（TLE格式）。

**请求消息（流式）：**

.. code-block:: protobuf

    message ImportSatellitesRequest {
        string user_id = 1;         // JWT Token
        int32 constellation_id = 2;  // 星座ID
        int32 satellite_id = 3;      // 卫星ID
        string info_line1 = 4;       // TLE第一行
        string info_line2 = 5;       // TLE第二行
    }

**响应消息：**

.. code-block:: protobuf

    message ImportSatellitesResponse {
        Status status = 1;           // 响应状态
        int32 success_count = 2;     // 成功导入数量
        int32 fail_count = 3;        // 失败数量
        repeated string errors = 4;  // 错误信息列表
    }

**调用示例：**

.. code-block:: python

    def generate_satellite_requests(user_id, constellation_id, tle_file):
        """从TLE文件生成请求流"""
        with open(tle_file, 'r') as f:
            lines = f.readlines()
            for i in range(0, len(lines), 2):
                if i + 1 < len(lines):
                    yield constellation_pb2.ImportSatellitesRequest(
                        user_id=user_id,
                        constellation_id=constellation_id,
                        satellite_id=i // 2 + 1,
                        info_line1=lines[i].strip(),
                        info_line2=lines[i+1].strip()
                    )

    # 调用流式接口
    response = constellation_stub.ImportSatellites(
        generate_satellite_requests(user_id, 1, 'satellites.tle')
    )

    print(f"成功导入: {response.success_count} 颗卫星")
    print(f"失败: {response.fail_count} 颗")
    if response.errors:
        print("错误信息:")
        for error in response.errors:
            print(f"  - {error}")

**注意事项：**

- 这是一个客户端流式RPC，客户端发送多个请求，服务器返回一个响应
- 适合大批量卫星数据导入
- TLE格式：两行元素格式（Two-Line Element Set）

ExportConstellations (导出星座数据)
-----------------------------------

导出一个或多个星座的完整数据（TLE和ISL），返回ZIP文件。

**请求消息：**

.. code-block:: protobuf

    message ExportConstellationsRequest {
        string user_id = 1;                    // JWT Token
        repeated int32 constellation_ids = 2;  // 星座ID列表
    }

**响应消息：**

.. code-block:: protobuf

    message ExportConstellationsResponse {
        Status status = 1;    // 响应状态
        bytes zip_data = 2;   // ZIP文件的二进制数据
    }

**调用示例：**

.. code-block:: python

    response = constellation_stub.ExportConstellations(
        constellation_pb2.ExportConstellationsRequest(
            user_id=user_id,
            constellation_ids=[1, 2, 3]
        )
    )

    if response.status.code == 0:
        # 保存ZIP文件
        with open('export.zip', 'wb') as f:
            f.write(response.zip_data)
        print("导出成功，文件已保存为 export.zip")

**导出内容：**

ZIP文件包含以下内容：

- ``constellation_<id>_tle.txt``: TLE格式的卫星轨道数据
- ``constellation_<id>_isl.txt``: ISL格式的卫星链接数据

SatelliteService (卫星管理服务)
================================

卫星（Satellite）管理服务，包括卫星的增删改查以及卫星间链接管理。

服务定义
--------

.. code-block:: protobuf

    service SatelliteService {
        rpc ListSatellites(ListSatellitesRequest) returns (ListSatellitesResponse);
        rpc GetSatellite(GetSatelliteRequest) returns (GetSatelliteResponse);
        rpc CreateSatellite(CreateSatelliteRequest) returns (CreateSatelliteResponse);
        rpc UpdateSatellite(UpdateSatelliteRequest) returns (UpdateSatelliteResponse);
        rpc DeleteSatellite(DeleteSatelliteRequest) returns (DeleteSatelliteResponse);
        rpc GetSatellitesByConstellation(GetSatellitesByConstellationRequest)
            returns (GetSatellitesByConstellationResponse);
        rpc CreateLink(CreateLinkRequest) returns (CreateLinkResponse);
        rpc DeleteLink(DeleteLinkRequest) returns (DeleteLinkResponse);
        rpc ImportLinks(stream ImportLinksRequest) returns (ImportLinksResponse);
    }

Satellite (卫星消息)
--------------------

.. code-block:: protobuf

    message Satellite {
        int32 id = 1;                  // 数据库记录ID
        int32 satellite_id = 2;        // 卫星ID（业务ID）
        int32 constellation_id = 3;    // 所属星座ID
        string info_line1 = 4;         // TLE第一行数据
        string info_line2 = 5;         // TLE第二行数据
        string ext_info = 6;           // 扩展信息（JSON格式字符串）
    }

SatelliteLink (卫星链接消息)
----------------------------

表示两颗卫星之间的链接关系（ISL - Inter-Satellite Link）。

.. code-block:: protobuf

    message SatelliteLink {
        int32 id = 1;                  // 链接ID
        int32 satellite_id1 = 2;       // 卫星1的ID
        int32 satellite_id2 = 3;       // 卫星2的ID
        int32 constellation_id = 4;    // 所属星座ID
    }

ListSatellites (获取卫星列表)
-----------------------------

获取卫星列表，支持分页。

**请求消息：**

.. code-block:: protobuf

    message ListSatellitesRequest {
        string user_id = 1;              // JWT Token
        PaginationRequest pagination = 2;  // 分页参数
    }

**响应消息：**

.. code-block:: protobuf

    message ListSatellitesResponse {
        Status status = 1;                 // 响应状态
        repeated Satellite satellites = 2;  // 卫星列表
        PaginationResponse pagination = 3;  // 分页信息
    }

**调用示例：**

.. code-block:: python

    response = satellite_stub.ListSatellites(
        satellite_pb2.ListSatellitesRequest(
            user_id=user_id,
            pagination=common_pb2.PaginationRequest(page=1, per_page=50)
        )
    )

    print(f"总共 {response.pagination.total_items} 颗卫星")
    for satellite in response.satellites:
        print(f"卫星 {satellite.satellite_id}: 星座 {satellite.constellation_id}")

GetSatellite (获取卫星详情)
---------------------------

获取特定卫星的详细信息，包括其所有链接关系。

**请求消息：**

.. code-block:: protobuf

    message GetSatelliteRequest {
        string user_id = 1;      // JWT Token
        int32 satellite_id = 2;  // 卫星ID（业务ID，非数据库ID）
    }

**响应消息：**

.. code-block:: protobuf

    message GetSatelliteResponse {
        Status status = 1;                   // 响应状态
        Satellite satellite = 2;              // 卫星信息
        repeated SatelliteLink links_from = 3;  // 从该卫星出发的链接
        repeated SatelliteLink links_to = 4;    // 指向该卫星的链接
    }

**调用示例：**

.. code-block:: python

    response = satellite_stub.GetSatellite(
        satellite_pb2.GetSatelliteRequest(
            user_id=user_id,
            satellite_id=101
        )
    )

    if response.status.code == 0:
        sat = response.satellite
        print(f"卫星详情: ID={sat.satellite_id}")
        print(f"TLE Line 1: {sat.info_line1}")
        print(f"TLE Line 2: {sat.info_line2}")
        print(f"链接数量: {len(response.links_from) + len(response.links_to)}")

**错误情况：**

- ``404 NOT_FOUND``: 卫星不存在

CreateSatellite (创建卫星)
--------------------------

创建新的卫星。

**请求消息：**

.. code-block:: protobuf

    message CreateSatelliteRequest {
        string user_id = 1;         // JWT Token
        int32 satellite_id = 2;     // 卫星ID（业务ID）
        int32 constellation_id = 3;  // 所属星座ID
        string info_line1 = 4;      // TLE第一行
        string info_line2 = 5;      // TLE第二行
        string ext_info = 6;        // 扩展信息（JSON格式）
    }

**响应消息：**

.. code-block:: protobuf

    message CreateSatelliteResponse {
        Status status = 1;       // 响应状态
        Satellite satellite = 2;  // 创建的卫星信息
    }

**调用示例：**

.. code-block:: python

    response = satellite_stub.CreateSatellite(
        satellite_pb2.CreateSatelliteRequest(
            user_id=user_id,
            satellite_id=101,
            constellation_id=1,
            info_line1="1 25544U 98067A   21275.51700231  .00003354  00000-0  69463-4 0  9998",
            info_line2="2 25544  51.6442 297.6680 0003015  65.3647  23.0452 15.48928894305719",
            ext_info='{"name": "ISS", "type": "space_station"}'
        )
    )

    if response.status.code == 0:
        print(f"创建成功，卫星数据库ID: {response.satellite.id}")

**错误情况：**

- ``400 BAD_REQUEST``: 参数错误或TLE格式错误
- ``409 CONFLICT``: 卫星ID已存在

UpdateSatellite (更新卫星)
--------------------------

更新现有卫星的信息。

**请求消息：**

.. code-block:: protobuf

    message UpdateSatelliteRequest {
        string user_id = 1;         // JWT Token
        int32 id = 2;               // 卫星数据库ID
        int32 satellite_id = 3;     // 新的卫星业务ID
        int32 constellation_id = 4;  // 新的星座ID
        string info_line1 = 5;      // 新的TLE第一行
        string info_line2 = 6;      // 新的TLE第二行
        string ext_info = 7;        // 新的扩展信息
    }

**响应消息：**

.. code-block:: protobuf

    message UpdateSatelliteResponse {
        Status status = 1;       // 响应状态
        Satellite satellite = 2;  // 更新后的卫星信息
    }

**调用示例：**

.. code-block:: python

    response = satellite_stub.UpdateSatellite(
        satellite_pb2.UpdateSatelliteRequest(
            user_id=user_id,
            id=1,
            satellite_id=101,
            constellation_id=1,
            info_line1="更新后的TLE第一行",
            info_line2="更新后的TLE第二行",
            ext_info='{"name": "ISS", "status": "active"}'
        )
    )

**错误情况：**

- ``404 NOT_FOUND``: 卫星不存在
- ``403 FORBIDDEN``: 无权修改该卫星

DeleteSatellite (删除卫星)
--------------------------

删除指定的卫星及其所有链接关系。

**请求消息：**

.. code-block:: protobuf

    message DeleteSatelliteRequest {
        string user_id = 1;  // 用户ID（字符串格式）
        int32 id = 2;        // 卫星数据库ID
    }

**响应消息：**

.. code-block:: protobuf

    message DeleteSatelliteResponse {
        Status status = 1;  // 响应状态
    }

**调用示例：**

.. code-block:: python

    response = satellite_stub.DeleteSatellite(
        satellite_pb2.DeleteSatelliteRequest(
            user_id=user_id,
            id=1
        )
    )

    if response.status.code == 0:
        print("删除成功")

**注意事项：**

- 删除卫星会同时删除该卫星的所有链接关系

**错误情况：**

- ``404 NOT_FOUND``: 卫星不存在
- ``403 FORBIDDEN``: 无权删除该卫星

GetSatellitesByConstellation (按星座查询卫星)
---------------------------------------------

获取指定星座下的所有卫星。

**请求消息：**

.. code-block:: protobuf

    message GetSatellitesByConstellationRequest {
        string user_id = 1;          // JWT Token
        int32 constellation_id = 2;  // 星座ID
    }

**响应消息：**

.. code-block:: protobuf

    message GetSatellitesByConstellationResponse {
        Status status = 1;                 // 响应状态
        repeated Satellite satellites = 2;  // 卫星列表
    }

**调用示例：**

.. code-block:: python

    response = satellite_stub.GetSatellitesByConstellation(
        satellite_pb2.GetSatellitesByConstellationRequest(
            user_id=user_id,
            constellation_id=1
        )
    )

    print(f"星座包含 {len(response.satellites)} 颗卫星")
    for sat in response.satellites:
        print(f"  - 卫星ID: {sat.satellite_id}")

CreateLink (创建卫星链接)
-------------------------

在两颗卫星之间创建链接关系。

**请求消息：**

.. code-block:: protobuf

    message CreateLinkRequest {
        string user_id = 1;          // JWT Token
        int32 constellation_id = 2;  // 星座ID
        int32 satellite_id1 = 3;     // 卫星1的ID
        int32 satellite_id2 = 4;     // 卫星2的ID
    }

**响应消息：**

.. code-block:: protobuf

    message CreateLinkResponse {
        Status status = 1;      // 响应状态
        SatelliteLink link = 2;  // 创建的链接信息
    }

**调用示例：**

.. code-block:: python

    response = satellite_stub.CreateLink(
        satellite_pb2.CreateLinkRequest(
            user_id=user_id,
            constellation_id=1,
            satellite_id1=101,
            satellite_id2=102
        )
    )

    if response.status.code == 0:
        print(f"链接创建成功，链接ID: {response.link.id}")

**注意事项：**

- 链接是无向的，卫星1和卫星2的顺序不影响结果
- 不能创建重复的链接

**错误情况：**

- ``404 NOT_FOUND``: 卫星不存在
- ``409 CONFLICT``: 链接已存在

DeleteLink (删除卫星链接)
-------------------------

删除指定的卫星链接。

**请求消息：**

.. code-block:: protobuf

    message DeleteLinkRequest {
        string user_id = 1;  // 用户ID（字符串格式）
        int32 link_id = 2;   // 链接ID
    }

**响应消息：**

.. code-block:: protobuf

    message DeleteLinkResponse {
        Status status = 1;  // 响应状态
    }

**调用示例：**

.. code-block:: python

    response = satellite_stub.DeleteLink(
        satellite_pb2.DeleteLinkRequest(
            user_id=user_id,
            link_id=1
        )
    )

    if response.status.code == 0:
        print("链接删除成功")

**错误情况：**

- ``404 NOT_FOUND``: 链接不存在
- ``403 FORBIDDEN``: 无权删除该链接

ImportLinks (批量导入卫星链接)
------------------------------

使用客户端流式传输批量导入卫星链接数据（ISL格式）。

**请求消息（流式）：**

.. code-block:: protobuf

    message ImportLinksRequest {
        string user_id = 1;          // JWT Token
        int32 constellation_id = 2;  // 星座ID
        int32 satellite_id1 = 3;     // 卫星1的ID
        int32 satellite_id2 = 4;     // 卫星2的ID
    }

**响应消息：**

.. code-block:: protobuf

    message ImportLinksResponse {
        Status status = 1;           // 响应状态
        int32 success_count = 2;     // 成功导入数量
        int32 fail_count = 3;        // 失败数量
        repeated string errors = 4;  // 错误信息列表
    }

**调用示例：**

.. code-block:: python

    def generate_link_requests(user_id, constellation_id, isl_file):
        """从ISL文件生成请求流"""
        with open(isl_file, 'r') as f:
            for line in f:
                parts = line.strip().split()
                if len(parts) >= 2:
                    yield satellite_pb2.ImportLinksRequest(
                        user_id=user_id,
                        constellation_id=constellation_id,
                        satellite_id1=int(parts[0]),
                        satellite_id2=int(parts[1])
                    )

    # 调用流式接口
    response = satellite_stub.ImportLinks(
        generate_link_requests(user_id, 1, 'links.isl')
    )

    print(f"成功导入: {response.success_count} 条链接")
    print(f"失败: {response.fail_count} 条")
    if response.errors:
        print("错误信息:")
        for error in response.errors:
            print(f"  - {error}")

**ISL文件格式：**

每行表示一个链接，格式为：``satellite_id1 satellite_id2``

示例::

    101 102
    102 103
    103 104

错误处理
========

gRPC 状态码
-----------

除了响应消息中的 Status 字段外，gRPC 还会返回标准的状态码：

===================  ========================================
gRPC 状态码           说明
===================  ========================================
OK                   请求成功
INVALID_ARGUMENT     无效参数
UNAUTHENTICATED      未认证
PERMISSION_DENIED    权限不足
NOT_FOUND            资源不存在
ALREADY_EXISTS       资源已存在
INTERNAL             服务器内部错误
===================  ========================================

错误处理示例
------------

.. code-block:: python

    import grpc

    try:
        response = auth_stub.Login(auth_pb2.LoginRequest(
            username="user",
            password="pass"
        ))

        if response.status.code == 0:
            print("登录成功")
        else:
            print(f"登录失败: {response.status.message}")

    except grpc.RpcError as e:
        print(f"gRPC错误: {e.code()}")
        print(f"错误详情: {e.details()}")

完整示例
========

用户注册、登录和管理星座
------------------------

.. code-block:: python

    import grpc
    from grpc_generated import (
        auth_pb2, auth_pb2_grpc,
        constellation_pb2, constellation_pb2_grpc,
        satellite_pb2, satellite_pb2_grpc
    )

    def main():
        # 连接服务器
        channel = grpc.insecure_channel('localhost:50051')

        # 创建客户端
        auth_stub = auth_pb2_grpc.AuthServiceStub(channel)
        constellation_stub = constellation_pb2_grpc.ConstellationServiceStub(channel)
        satellite_stub = satellite_pb2_grpc.SatelliteServiceStub(channel)

        # 1. 用户注册
        print("=== 用户注册 ===")
        register_resp = auth_stub.Register(auth_pb2.RegisterRequest(
            username="demo_user",
            password="demo_pass123"
        ))
        print(f"注册: {register_resp.status.message}")

        # 2. 用户登录
        print("\n=== 用户登录 ===")
        login_resp = auth_stub.Login(auth_pb2.LoginRequest(
            username="demo_user",
            password="demo_pass123"
        ))

        if login_resp.status.code != 0:
            print(f"登录失败: {login_resp.status.message}")
            return

        # 获取用户ID并转换为字符串（proto定义为string类型）
        user_id = str(login_resp.user.id)
        print(f"登录成功，用户: {login_resp.user.username}, 用户ID: {user_id}")

        # 3. 创建星座
        print("\n=== 创建星座 ===")
        constellation_resp = constellation_stub.CreateConstellation(
            constellation_pb2.CreateConstellationRequest(
                user_id=user_id,
                constellation_name="测试星座",
                description="这是一个测试星座"
            )
        )

        if constellation_resp.status.code != 0:
            print(f"创建星座失败: {constellation_resp.status.message}")
            return

        constellation_id = constellation_resp.constellation.id
        print(f"创建星座成功，ID: {constellation_id}")

        # 4. 创建卫星
        print("\n=== 创建卫星 ===")
        for i in range(1, 4):
            sat_resp = satellite_stub.CreateSatellite(
                satellite_pb2.CreateSatelliteRequest(
                    user_id=user_id,
                    satellite_id=100 + i,
                    constellation_id=constellation_id,
                    info_line1=f"1 {100+i}U 98067A   21275.51700231  .00003354  00000-0  69463-4 0  9998",
                    info_line2=f"2 {100+i}  51.6442 297.6680 0003015  65.3647  23.0452 15.48928894305719"
                )
            )
            print(f"卫星 {100+i}: {sat_resp.status.message}")

        # 5. 创建链接
        print("\n=== 创建卫星链接 ===")
        link_resp = satellite_stub.CreateLink(
            satellite_pb2.CreateLinkRequest(
                user_id=user_id,
                constellation_id=constellation_id,
                satellite_id1=101,
                satellite_id2=102
            )
        )
        print(f"链接创建: {link_resp.status.message}")

        # 6. 查询星座详情
        print("\n=== 查询星座详情 ===")
        detail_resp = constellation_stub.GetConstellation(
            constellation_pb2.GetConstellationRequest(
                user_id=user_id,
                constellation_id=constellation_id
            )
        )

        if detail_resp.status.code == 0:
            c = detail_resp.constellation
            print(f"星座: {c.constellation_name}")
            print(f"描述: {c.description}")
            print(f"卫星数量: {c.satellite_count}")
            print(f"卫星列表:")
            for sat in detail_resp.satellites:
                print(f"  - 卫星ID: {sat.satellite_id}")

        print("\n=== 完成 ===")
        channel.close()

    if __name__ == "__main__":
        main()

批量导入示例
------------

.. code-block:: python

    import grpc
    from grpc_generated import constellation_pb2, constellation_pb2_grpc

    def import_tle_file(user_id, constellation_id, tle_file_path):
        """从TLE文件批量导入卫星"""
        channel = grpc.insecure_channel('localhost:50051')
        stub = constellation_pb2_grpc.ConstellationServiceStub(channel)

        def request_generator():
            with open(tle_file_path, 'r') as f:
                lines = [line.strip() for line in f if line.strip()]

                satellite_id = 1
                for i in range(0, len(lines), 2):
                    if i + 1 < len(lines):
                        yield constellation_pb2.ImportSatellitesRequest(
                            user_id=user_id,
                            constellation_id=constellation_id,
                            satellite_id=satellite_id,
                            info_line1=lines[i],
                            info_line2=lines[i + 1]
                        )
                        satellite_id += 1

        # 调用流式接口
        response = stub.ImportSatellites(request_generator())

        print(f"导入完成:")
        print(f"  成功: {response.success_count}")
        print(f"  失败: {response.fail_count}")

        if response.errors:
            print("错误信息:")
            for error in response.errors:
                print(f"  - {error}")

        channel.close()
        return response

附录
====

Proto 文件位置
--------------

项目中的 Protocol Buffer 定义文件位于：

- ``protos/common.proto`` - 通用消息定义
- ``protos/auth.proto`` - 认证服务
- ``protos/base.proto`` - 基座服务
- ``protos/constellation.proto`` - 星座服务
- ``protos/satellite.proto`` - 卫星服务

生成客户端代码
--------------

使用项目提供的脚本生成 gRPC 代码：

.. code-block:: bash

    python generate_grpc.py

生成的代码位于 ``grpc_generated/`` 目录。

其他语言客户端
--------------

gRPC 支持多种编程语言，可以使用相同的 ``.proto`` 文件生成对应语言的客户端代码。

**Java 示例：**

.. code-block:: bash

    protoc --java_out=./java --grpc-java_out=./java protos/*.proto

**Go 示例：**

.. code-block:: bash

    protoc --go_out=. --go-grpc_out=. protos/*.proto

**JavaScript/TypeScript 示例：**

.. code-block:: bash

    grpc_tools_node_protoc --js_out=import_style=commonjs:./js \
        --grpc_out=grpc_js:./js protos/*.proto

参考资源
--------

- `gRPC 官方文档 <https://grpc.io/docs/>`_
- `Protocol Buffers 文档 <https://developers.google.com/protocol-buffers>`_
- `gRPC Python 快速入门 <https://grpc.io/docs/languages/python/quickstart/>`_

版本历史
--------

:v1.0.0: 初始版本，包含所有核心服务
:当前版本: v1.0.0

支持与反馈
----------

如有问题或建议，请联系开发团队或提交 Issue。
