## Backgrounds

- 用户发现距离自己最近的商家
- 客户端通过网关发现时延最低的下游处理集群

## Functional Requirements

### 用户-PS-商家

- 用户坐标
- 用户需求的最大半径/到达时间
- 商家 CRUD 坐标到 PS

  - 不要求实时更新, 可隔天更新

### 客户端-PS-服务端

- 客户端 IP
- 可接受最大时延
- 要求实时性

## Non-functional Requirements (Constraints, SLA)

- 1 亿客户日活(DAU) + 每人 5 次 call

  - 100m _5 / (60_ 60 \* 24) = 6k qps

    - 峰值 10k qps

- 2 亿商家

  - 渲染表: 200m \* 1KB = 200GB 可放入磁盘, mysql, 考虑分表或 cache
  - 地理表: 200m \* 24b = 5GB 可放入内存, 考虑 nosql

- 用户读, 商家读时延 < 50ms, 商家写时延 < 1d
- HA99999

## High-Level Design

### API

#### 用户

- GET /v1/search/nearby

  - Double: 经纬度
  - Int: 半径
  - Return:

    ```JSON
    {
        "total": 10,
        "biz":
        [
            {
                "id": 1,
                "desc": "星巴克"
            },
            {
                "id": 7,
                "desc": "coco"
            },
            ...
        ]
    }
    ```

#### 商家

- GET /v1/biz/{id}
- POST /v1/biz
- PUT /v1/biz/{id}
- DELETE /v1/biz/{id}

### Data Schema

#### 商家

- 渲染表

  id(PK)

  address

  long

  lat

  city

  state

  country

- 地理表 v1

  id(PK)

  long(地理索引)

  lat(地理索引)

- 为什么不可以只用渲染表, 并给经纬度`(long, lat)`加联合索引?

  - 考虑 SQL

  ```SQL
  SELECT
      id, latitude, longtitude
  FROM
      渲染表
  WHERE
      (latitude BETWEEN user_lat - radius AND user_lat + radius)
  AND
      (longitude BETWEEN user_long - radius AND user_long + radius)
  ```

  - 使用 explain, 发现不会使用索引, 而是进行全表扫描, 达不到时延需求

  ```Shell
  mysql> explain select id,lat,`long` from t_geo where ( lat between 150.0 - 50.0 and 150.0 + 50.0) and (`long` between 150.7 - 50.5 and 178.24 + 50.12);
  +----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+-------------+
  | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra       |
  +----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+-------------+
  |  1 | SIMPLE      | t_geo | NULL       | ALL  | k1,k2,k3      | NULL | NULL    | NULL | 99913 |    25.00 | Using where |
  +----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+-------------+
  ```

  - 解决思路: 将经纬度 map 成单个值, 在此值上建立索引

    ![](https://bytedancecampus1.feishu.cn/space/api/box/stream/download/asynccode/?code=MTJiMjFiMzRlZDFmYWRhNmExNmU1ODkyNmRkYjVmZGJfVVVQTlFKdXJCa2hDNWlmWDAwUWZuWHN1YkZwOUhQeTRfVG9rZW46Ym94Y25nVDlreXFaTWdxeXlPamtxRHUwQklkXzE2Nzg2Nzk3NTM6MTY3ODY4MzM1M19WNA)

    - Geohash: 递归地将地图分成 4 等份, 直到能最小覆盖用户要求的半径

      - 用 2bit 表示坐标, 第一个 bit 表示经度, 第二个 bit 表示纬度

      ![](https://bytedancecampus1.feishu.cn/space/api/box/stream/download/asynccode/?code=ZmQ3YTAzZGNjMDRiOWYzZjMzYTVkYjkyMWVmZjcwZGRfU0RwbnhrdlF4aXlGUURRTmZYVTEzbEo1NFNPWGcwUExfVG9rZW46Ym94Y255SnA3U1NBTmNKRmd6TmNvRWxsNk9EXzE2Nzg2Nzk3NTM6MTY3ODY4MzM1M19WNA)

      - 压缩空间: 用 base32 表示二进制串, 用前缀相似度判断两者距离
      - 边界问题: 在**主分割线**(赤道, 子午线)两侧, 即使商家离得很近, 前缀相似度也很低

        - 对策: 额外取 8 个临近 block

    - Quadtree: 预设终止条件(如商家数<100), 递归地建立四叉树分割地图, 直到满足终止条件

      - 收集满足终止条件的叶节点, 返回其坐标

        ![](https://bytedancecampus1.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDU0ODM4OTk5NTYyOGI2ZTdmN2M1YmMyODMxMmY1NDNfbmVpdlhUWUhVREkzcDNzWU4xYjFFMjd3eVl1QllYSWtfVG9rZW46Ym94Y245M2JSY0JrTGZTNjBGb1dxTzMxdW5iXzE2Nzg2Nzk3NTM6MTY3ODY4MzM1M19WNA)

- 地理表 v2

  geohash(PK)

  id

  long

  lat

  ```SQL
  SELECT id, long, lat FROM t_geo WHERE geohash LIKE `prefix%`;
  ```

### Diagram

![](https://bytedancecampus1.feishu.cn/space/api/box/stream/download/asynccode/?code=MmU3NGEyZWY1OTkzZjJiMGQwNDM5ZGYyNGZmNDk3YWJfRXhCNlBBNE8xRkJQMHlWN2hVemZSVkRnbnhjZUNKeUZfVG9rZW46Ym94Y251WE1WSHEwT1RnWjA2UUl3SENsR0poXzE2Nzg2Nzk3NTM6MTY3ODY4MzM1M19WNA)

- 用户 API 高频只读, 商家 API 低频读写 -> 读写分离, 商家主库, 用户从库, LB 分离请求流量
- LBS 只读 -> 无状态 -> 可水平拓展满足高 qps
- BS 读写 -> 读 cache, 写 DB

### Recap (Life Tracing, Dry-run)

1. user 发送请求到 LB, LB 转发至 LBS, 请求包含 user_lat, user_long, radius
2. LBS 根据 radius 得到地图分割粒度, 根据`(user_lat, user_long)`得到 user_geohash
3. LBS 计算 user_geohash 附近 8 格的 biz_geohash
4. LBS 向 Replica 发送请求, 获取格子里`biz_geohash`包括的所有`biz_id`, 以及`(biz_long, biz_lat)`
5. LBS 根据返回的坐标, 计算与 user 坐标的真实距离并排序分页, 将页内 biz_id 返回给 LB
6. LB 向 BS 发送请求, 拉取这些 biz_id 对应的渲染信息
7. 返回给用户
