# ShardingSphere-Proxy 基于Docker搭建部署文档

## 文档说明

本文档详细记录基于Docker搭建ShardingSphere-Proxy的完整步骤、前置数据准备、分片规则配置流程，并整理实操过程中遇到的问题及解决方案，适用于分库分表场景的快速部署落地。文档中已隐藏私密IP、账号等敏感信息，可直接作为标准化部署参考，使用时需根据实际环境替换占位符内容。

## 一、环境准备

- 宿主机已安装Docker环境（推荐Docker Engine 20.10及以上版本，确保容器正常启动和挂载功能可用）；

- 已搭建MySQL数据库集群（作为分片物理库，数量根据业务分库需求准备，确保各物理库网络互通）；

- 宿主机具备目录挂载权限，用于映射容器内配置文件（conf）和扩展库（ext-lib）目录，避免容器重启后配置丢失；

- 网络互通：宿主机可正常访问MySQL集群，应用服务可访问ShardingSphere-Proxy映射的宿主机端口。

## 二、ShardingSphere-Proxy 容器部署

### 2.1 拉取官方Docker镜像

推荐从DockerHub拉取Apache官方镜像，确保镜像版本稳定，执行命令如下：

```bash
docker pull apache/shardingsphere-proxy
```

### 2.2 拷贝容器内配置文件模板到宿主机

容器内置默认配置文件模板，需先拷贝到宿主机自定义目录，方便后续修改配置（如数据源、端口等），执行以下命令：

```bash
# 临时启动容器用于拷贝配置文件
docker run -d --name tmp --entrypoint=bash apache/shardingsphere-proxy
# 将容器内conf目录拷贝到宿主机自定义路径（替换/host/path/to/conf为实际宿主机路径，如/root/shardingsphere/conf）
docker cp tmp:/opt/shardingsphere-proxy/conf /host/path/to/conf
# 删除临时容器（拷贝完成后无需保留）
docker rm tmp
```

### 2.3 启动ShardingSphere-Proxy容器

将宿主机的conf配置目录和ext-lib扩展库目录挂载到容器，指定容器内端口、宿主机映射端口及JVM内存参数，启动命令如下：

```bash
docker run -d \
    -v /host/path/to/conf:/opt/shardingsphere-proxy/conf \
    -v /host/path/to/ext-lib:/opt/shardingsphere-proxy/ext-lib \
    -e PORT=3308 -p13308:3308 \
    -e "CGROUP_MEM_OPTS=-XX:InitialRAMPercentage=5.0 -XX:MaxRAMPercentage=5.0 -XX:MinRAMPercentage=5.0" \
    apache/shardingsphere-proxy:latest
```

#### 参数说明

- -v /host/path/to/conf:/opt/shardingsphere-proxy/conf：宿主机配置目录与容器配置目录映射，后续修改配置直接操作宿主机目录即可；

- -v /host/path/to/ext-lib:/opt/shardingsphere-proxy/ext-lib：宿主机扩展库目录与容器扩展库目录映射，用于放置MySQL JDBC驱动包等扩展文件；

- PORT=3308：容器内ShardingSphere-Proxy的监听端口，可根据容器内端口占用情况调整；

- p13308:3308：宿主机端口与容器端口映射，应用服务通过宿主机13308端口访问Proxy（宿主机端口可自定义，避免端口冲突）；

- CGROUP_MEM_OPTS：JVM内存占比配置，可根据宿主机资源情况按需调整（如宿主机内存较大，可提高MaxRAMPercentage）。

## 三、MySQL物理库前置数据准备

在配置ShardingSphere-Proxy分片规则前，需先对MySQL物理库做前置准备，确保各物理库表结构一致、分片字段齐全，避免后续数据路由异常。

### 3.1 创建分片物理库

根据业务分库需求，在MySQL集群中创建对应的物理数据库，示例命名：pilotroster_caac_0、pilotroster_caac_1、pilotroster_caac_2（可按业务规范自定义命名，确保名称统一，便于后续配置分片规则）。

### 3.2 为所有表统一添加分片字段

为所有物理库中的基础表添加分片键字段SIMULATION_INSTANCE_ID（仿真实例ID，可按业务需求自定义分片键名），先执行查询语句生成新增字段的SQL，再执行生成的SQL完成字段添加，避免重复添加：

```sql
SELECT CONCAT('ALTER TABLE `', t.table_name, '` ADD COLUMN SIMULATION_INSTANCE_ID int null comment \'仿真实例ID\';') AS sql_statement
FROM information_schema.tables t
LEFT JOIN information_schema.columns c
  ON t.table_name = c.table_name
  AND c.table_schema = t.table_schema
  AND c.column_name = 'SIMULATION_INSTANCE_ID'  -- 替换为实际分片键字段名
WHERE t.table_schema = '目标物理库名'  -- 替换为实际MySQL物理库名（如pilotroster_caac_0）
  AND t.table_type = 'BASE TABLE'     -- 仅处理基础表，不处理视图
  AND c.column_name IS NULL;          -- 仅对无该分片字段的表生成SQL
```

说明：需为所有分片物理库逐一执行上述操作，确保各物理库的同一张表结构完全一致，避免分片路由时出现表结构不匹配错误。

### 3.3 初始化分片字段值

为已有数据的表统一初始化分片字段值，先执行查询语句生成更新SQL，再执行生成的SQL（避免手动逐表更新，提高效率）：

```sql
SELECT CONCAT('UPDATE `', t.table_name, '` SET `SIMULATION_INSTANCE_ID` = 1 where 1=1;') AS sql_statement
FROM information_schema.tables t
JOIN information_schema.columns c
  ON t.table_name = c.table_name
  AND t.table_schema = c.table_schema
WHERE t.table_schema = '目标物理库名'       -- 替换为实际MySQL物理库名
  AND t.table_type = 'BASE TABLE'                 -- 仅处理基础表
  AND c.column_name = 'SIMULATION_INSTANCE_ID';   -- 替换为实际分片键字段名
```

说明：示例中分片字段值设为1，可根据业务实际数据规则调整赋值（如按业务ID范围分配分片值），确保数据能按预期路由到对应物理库。

## 四、ShardingSphere-Proxy 分片规则配置

通过DistSQL语句连接ShardingSphere-Proxy进行配置，核心步骤为：创建逻辑数据库→注册物理数据源→创建表分片规则→验证路由规则，所有操作需在Proxy连接会话中执行，确保上下文正确。

### 4.1 连接ShardingSphere-Proxy

使用MySQL客户端（如Navicat、DBeaver、命令行）连接Proxy，连接信息如下：

- 地址：宿主机IP（无需填写容器IP，通过宿主机映射端口访问）；

- 端口：宿主机映射端口（本文示例为13308，与容器启动时配置一致）；

- 账号/密码：默认root/root（可在容器conf目录的server.yaml配置文件中修改，提高安全性）。

### 4.2 执行DistSQL配置分片规则

连接Proxy后，执行以下DistSQL语句，完成逻辑库创建、数据源注册和分片规则配置（按实际环境替换占位符内容）：

```sql
-- 1. 创建逻辑数据库（首次配置必做，自定义逻辑库名，如pilotroster）
CREATE DATABASE pilotroster;
USE pilotroster;  -- 切换到创建的逻辑库，后续操作均在该上下文下执行

-- 2. 注册物理数据源（按实际物理库数量逐一注册，替换为实际配置）
REGISTER STORAGE UNIT 物理库名1 (
    URL="jdbc:mysql://MySQL地址:MySQL端口/物理库名1?serverTimezone=GMT%2B8&useSSL=false",
    USER="MySQL账号",
    PASSWORD="MySQL密码",
    PROPERTIES("maximumPoolSize"=10,"idleTimeout"=30000)
);
REGISTER STORAGE UNIT 物理库名2 (
    URL="jdbc:mysql://MySQL地址:MySQL端口/物理库名2?serverTimezone=GMT%2B8&useSSL=false",
    USER="MySQL账号",
    PASSWORD="MySQL密码",
    PROPERTIES("maximumPoolSize"=10,"idleTimeout"=30000)
);
REGISTER STORAGE UNIT 物理库名3 (
    URL="jdbc:mysql://MySQL地址:MySQL端口/物理库名3?serverTimezone=GMT%2B8&useSSL=false",
    USER="MySQL账号",
    PASSWORD="MySQL密码",
    PROPERTIES("maximumPoolSize"=10,"idleTimeout"=30000)
);

-- 3. 创建表分片规则（以opt_filiale表为例，仅分库不分表，适配业务需求）
CREATE SHARDING TABLE RULE opt_filiale (
    DATANODES("物理库名${0..2}.opt_filiale"),  -- 匹配已注册的物理库名，表名与物理库表名一致
    DATABASE_STRATEGY(
        TYPE="standard",
        SHARDING_COLUMN=SIMULATION_INSTANCE_ID,  -- 分片键字段名，与物理库表中字段一致
        SHARDING_ALGORITHM(
            TYPE(NAME="inline",PROPERTIES("algorithm-expression"="物理库名${SIMULATION_INSTANCE_ID.intdiv(1000)}"))
        )
    ),
    -- 审计策略：禁止无分片条件的操作，避免全库扫描导致性能下降（按需配置）
    AUDIT_STRATEGY (TYPE(NAME="DML_SHARDING_CONDITIONS"),ALLOW_HINT_DISABLE=true)
);

-- 4. 验证路由规则（插入测试数据，按需修改字段和值，确保字段与物理表结构一致）
insert into opt_filiale (code, SIMULATION_INSTANCE_ID) values ('MF', 2);
```

#### 配置说明

- 逻辑数据库：ShardingSphere-Proxy的虚拟库，仅用于管理物理数据源和分片规则，不实际存储数据，所有数据最终存储在注册的物理库中；

- 物理数据源注册：maximumPoolSize（连接池最大连接数）、idleTimeout（连接空闲超时时间）可按业务性能需求调整，避免连接池耗尽或空闲连接过多；

- 分片算法：本文使用inline行内算法，通过SIMULATION_INSTANCE_ID.intdiv(1000)实现按分片键整数除法分库（每1000个ID分一个库），可按业务需求替换为hash_mod、range等算法；

- 审计策略：DML_SHARDING_CONDITIONS用于拦截无分片条件的DML/查询操作，防止全库扫描导致Proxy性能暴跌，属于推荐配置。

### 4.3 规则验证方法

配置完成后，通过以下方法验证分片规则是否生效，确保数据路由正常：

1. 查看分片规则：验证规则是否成功创建，确认配置与预期一致
            `-- 查看当前逻辑库下所有分片表规则
SHOW SHARDING TABLE RULES;
-- 查看指定表的详细分片规则（如opt_filiale）
` `DESCRIBE SHARDING TABLE RULE opt_filiale;`

2. 验证路由逻辑：通过EXPLAIN查看SQL路由到的物理库，确认分库规则生效
            `EXPLAIN SELECT * FROM opt_filiale WHERE SIMULATION_INSTANCE_ID=2;`预期结果：输出中会显示SQL路由到的具体物理库（如物理库名2.opt_filiale），无分表后缀，说明仅分库规则生效。

3. 数据操作验证：插入/查询数据，确认数据能正确路由到对应物理库（注：无分片条件的全局查询会被审计策略拦截，无法执行）
`-- 插入不同分片键的数据
insert into opt_filiale (code, SIMULATION_INSTANCE_ID) values ('MF1', 999), ('MF2', 1000), ('MF3', 2001);
-- 指定分片键查询（精准路由到对应物理库，可正常执行）
SELECT * FROM opt_filiale WHERE SIMULATION_INSTANCE_ID=999;
-- 注意：无分片条件的全局查询（如下）会被审计策略拦截，执行失败
` `-- SELECT * FROM opt_filiale;`

## 五、实操常见问题及解决方案

结合实操过程中遇到的各类问题，整理以下高频问题及解决方案，帮助快速避坑、定位问题，提高部署效率。

### 问题1：注册物理数据源时报错，无法连接MySQL数据库

#### 现象

在ShardingSphere-Proxy中执行DistSQL注册物理数据源（即添加物理库）时，直接报错提示“无法连接数据源”，无法完成物理库注册。排查过程中发现，此前部署容器时虽挂载了ext-lib扩展目录，但未将MySQL JDBC驱动jar包拷贝到该目录，倒回去将驱动包补充到宿主机挂载的ext-lib目录并重启容器后，重新注册物理数据源，报错解决。

#### 根因

ShardingSphere-Proxy官方Docker镜像默认未内置MySQL JDBC驱动包，仅内置PostgreSQL等其他数据库驱动；且部署时虽配置了ext-lib目录挂载（用于放置扩展驱动），但未及时补充MySQL驱动包，导致Proxy无法加载MySQL驱动，进而无法连接物理库，注册数据源时触发报错。

#### 解决方案

1. 下载与MySQL版本匹配的MySQL Connector/J驱动包（推荐8.0+版本，兼容性更好），下载地址：https://dev.mysql.com/downloads/connector/j/；

2. 将下载的mysql-connector-java-8.0.xx.jar驱动包，拷贝到宿主机挂载的ext-lib目录（即/host/path/to/ext-lib）；

3. 重启ShardingSphere-Proxy容器，使驱动包生效：docker restart <容器ID>。

注意：1. 驱动版本需与MySQL服务版本兼容，8.0版本驱动可兼容5.7版本MySQL，但5.1版本驱动不兼容8.0版本MySQL，避免版本不匹配导致连接失败；2. 建议在容器启动前，提前将驱动包拷贝到ext-lib目录，避免后续注册物理数据源时出现报错，节省排查时间。

### 问题2：DistSQL语法错误提示不友好，无法定位错误原因

#### 现象

编写分片规则DistSQL后执行失败，仅返回模糊提示，无具体错误位置和真实原因，难以定位问题，甚至会误导排查方向。例如实操中曾遇到报错：[2026-02-27 14:15:44] [42000][1064] You have an error in your SQL syntax: no viable alternative at input 'CREATESHARDING' near '[@2,47:54='SHARDING',<882>,1:47]' at line 1，实际编写的SQL是“CREATE SHARDING”（中间有空格），并非提示中显示的“CREATESHARDING”（无空格），但由于整体SQL存在其他语法问题，报错却误导性地指向“CREATESHARDING”拼写错误，导致在错误方向浪费大量排查时间，最终通过参考他人可正常运行的DistSQL语句修改后，才成功解决问题，避免了继续在错误方向内消耗精力。

#### 根因

ShardingSphere-Proxy对DistSQL语法校验的错误提示未做精细化处理，不仅无具体错误位置，还可能出现误导性提示——即使SQL中某部分书写正确（如“CREATE SHARDING”中间有空格），若整体存在其他语法问题，报错也可能指向该正确部分，误导排查方向；同时，无论是括号缺失、引号错误、参数写错、关键词空格缺失等小错误，均会返回通用提示，大幅增加排查难度。

#### 解决方案

1. 逐段拆分DistSQL执行：先创建逻辑库，再注册数据源，最后创建分片规则，逐步缩小错误范围，定位到具体出错的语句段；

2. 严格检查语法细节：括号、逗号需成对且位置正确，分片键、物理库、表名需与实际一致（区分大小写，MySQL表名默认小写），算法表达式语法正确（如整数除法用intdiv而非/）；

3. 对照官方文档：根据当前ShardingSphere-Proxy版本，参考官方DistSQL语法规范编写语句，避免语法错误；

4. 避免手动输入：可复制官方示例语句或他人可正常运行的DistSQL语句，替换占位符内容，减少手动输入导致的空格、符号错误，同时也能规避因整体语法错误引发的误导性提示。

### 问题3：DistSQL语法向下兼容性差，旧版本语法无法在新版本执行

#### 现象

参考ShardingSphere-Proxy旧版本（如5.0）的DistSQL示例编写语句，在高版本（如5.4+）中执行直接失败，无“语法过时”“版本不兼容”等提示，仅返回语法错误。

#### 示例

- 5.0版本分片算法写法（5.4+版本不兼容）：SHARDING_ALGORITHM(TYPE="inline",PROPERTIES("algorithm-expression"="库名${字段/1000}"))；

- 5.4+版本正确写法：SHARDING_ALGORITHM(TYPE(NAME="inline",PROPERTIES("algorithm-expression"="库名${字段.intdiv(1000)}")))。

#### 解决方案

1. 确认当前ShardingSphere-Proxy版本：执行docker exec <容器ID> cat /opt/shardingsphere-proxy/VERSION，获取准确版本号；

2. 严格参考对应版本的官方文档，避免混用不同版本语法，优先使用当前版本的示例语句；

3. 若从旧版本升级，参考官方提供的语法迁移指南，对旧版本DistSQL语句进行调整后再执行。

### 问题4：删除分片规则后重建失败，提示物理数据节点已被占用

#### 现象

执行DROP SHARDING TABLE RULE 表名（如DROP SHARDING TABLE RULE opt_filiale）删除规则后，重新创建同名表的分片规则，提示“Same actual data node cannot be configured in multiple logic tables”，即使执行SHOW SHARDING TABLE RULES显示无规则，重启容器也无法解决。

#### 根因

ShardingSphere-Proxy元数据管理存在残留问题，删除表规则后，逻辑库的元数据未完全清理，物理数据节点（物理库.物理表）仍被标记为“已占用”，导致重建时触发唯一性校验。

#### 解决方案

1. 备份分片规则配置：先将当前逻辑库的DistSQL配置（创建逻辑库、注册数据源、分片规则）备份，避免删除后需重新编写；

2. 彻底删除逻辑库：执行DROP DATABASE 逻辑库名（如DROP DATABASE pilotroster），清理所有关联的元数据残留；

3. 重新执行完整的DistSQL配置：从头创建逻辑库→注册物理数据源→创建分片规则，此时无元数据残留，可正常创建。

注意：生产环境修改分片规则前，务必先备份当前DistSQL配置，避免删除后需重新编写，影响业务正常运行。

### 问题5：执行无分片条件的查询，提示Not allow DML operation without sharding conditions

#### 现象

执行SELECT * FROM 表名 WHERE 1=1等无分片键条件的查询时，被拦截并返回该提示，无法执行查询。

#### 根因

创建分片规则时配置了DML_SHARDING_CONDITIONS审计策略，该策略的作用是禁止无分片条件的DML/查询操作，避免全库扫描导致Proxy性能下降，属于正常的策略生效行为，并非报错。

#### 解决方案

1. 正常业务查询（推荐）：必须携带分片键条件（如WHERE SIMULATION_INSTANCE_ID=999），使SQL能精准路由到对应物理库，避免全库扫描；

2. 临时测试查询（仅测试用）：通过提示绕过审计规则，执行语句如下（生产环境禁止使用，避免影响性能）：
        `/*+SHARDINGSPHERE:ALLOW_HINT_DISABLE_AUDIT()*/ SELECT * FROM opt_filiale WHERE 1=1;`

3. 永久关闭审计（不推荐）：若业务确有频繁执行无分片条件查询的需求，可删除分片规则后重新创建，去掉AUDIT_STRATEGY相关配置。

## 六、总结与注意事项

### 部署总结

基于Docker搭建ShardingSphere-Proxy的核心流程为：镜像拉取→配置文件拷贝→容器启动→MySQL物理库准备→DistSQL规则配置，其中补充MySQL JDBC驱动包（避免注册物理数据源时报错）和保证物理库表结构一致性是部署成功的关键前提；同时，DistSQL语法版本依赖性极强，配置时需严格匹配当前Proxy版本，且其语法错误提示不友好，建议优先参考官方示例或他人可正常运行的语句，避免因语法问题及误导性提示导致部署失败、浪费排查时间；此外，开启审计策略后，无分片条件的全局查询会被拦截，需注意查询时必须携带分片键条件。

### 核心注意事项

- 配置文件与驱动包：宿主机需持久化挂载conf和ext-lib目录，避免容器重启后配置和驱动丢失，驱动包版本需与MySQL版本兼容；

- 语法兼容性：DistSQL语法版本依赖性强，不同版本语法差异较大，需严格对照当前Proxy版本的官方文档编写语句，避免混用旧版本语法；

- 元数据管理：修改/删除分片规则后若出现元数据残留，会导致规则重建失败，可通过删除并重建逻辑库解决，操作前务必备份配置，避免数据或配置丢失；

- 性能优化：开启审计策略防止无分片条件的全库扫描，根据业务需求调整数据源连接池参数（如maximumPoolSize），避免性能瓶颈；

- 表结构一致性：所有分片物理库的同一张表，表结构（字段、类型、分片键）必须完全一致，否则会导致数据插入、查询异常；

- 私密信息保护：部署文档中需隐藏IP、账号、密码等敏感信息，实际使用时替换占位符内容，避免信息泄露；

- 上下文一致性：执行DistSQL配置时，需先切换到目标逻辑库（USE 逻辑库名），避免因上下文错误导致规则配置失败。
> （注：文档部分内容可能由 AI 生成）