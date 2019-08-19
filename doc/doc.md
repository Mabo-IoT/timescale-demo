## 概览
基于postgresql的一个时间序列数据库
## 安装
1. docker
```
docker pull timescale/timescaledb:latest-pg11
docker run -d --name timescaledb -p 5432:5432 -e POSTGRES_PASSWORD=password timescale/timescaledb:latest-pg11

```

## 创建数据库相关
1. 创建数据库
    ```
    docker exec -it timescaledb psql -U postgres  // 连接数据库
    CREATE database tutorial; // 创建数据库
    \c tutorial //连接数据库
    CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE; // 创建timescale插件
    ```
2. 创建表
```
psql -U postgres -h localhost  // 进入sql
\c tutorial   //连接数据库
CREATE TABLE conditions (
  time        TIMESTAMPTZ       NOT NULL,
  location    TEXT              NOT NULL,
  temperature DOUBLE PRECISION  NULL,
  humidity    DOUBLE PRECISION  NULL
);     // 创建表
SELECT create_hypertable('conditions', 'time');   //将其转换为hypertable
```

3. 插入数据

```
INSERT INTO conditions(time, location, temperature, humidity) VALUES (NOW(), 'office', 70.0, 50.0); //插入数据
SELECT * FROM conditions ORDER BY time DESC LIMIT 100;  // 查询数据
```
4. 改变数据表结构
```
ALTER TABLE conditions ADD COLUMN humidity DOUBLE PRECISION NULL  //新增一行
DROP TABLE conditions;   // 删除表
```
如何设置存储时间间隔以及空间：
- 原则在于时间范围内的数据量是内存的25%，比如一天2G数据，内存64G，那么设置7天的chunks间隔
5. 加索引
```
CREATE INDEX ON conditions (location, time DESC)； // 加索引
```
如何正确加索引呢？
- 对于离散的数据，比如tag,location这些直接加索引
- 对于温度，湿度这些连续的数据，加条件索引
    ```
    CREATE INDEX ON conditions (time DESC, temperature);
    ```
6. 触发器
```
CREATE OR REPLACE FUNCTION record_error() 
    RETURNS trigger AS $record_error$ 
BEGIN 
    IF NEW.temperature >= 1000 OR NEW.humidity >= 1000 THEN 
        INSERT INTO error_conditions 
            VALUES(NEW.time, NEW.location, NEW.temperature, NEW.humidity); 
    END IF; 
    RETURN NEW;
END;
$record_error$ LANGUAGE plpgsql;
```
```
CREATE TRIGGER record_error
    BEFORE INSERT ON conditions 
    FOR EACH ROW 
    EXECUTE PROCEDURE record_error();
```
当温湿度数据异常时，插入error表中。
7. 增加约束
```
CREATE TABLE conditions (
    time TIMESTAMPTZ 
    temp FLOAT NOT NULL, 
    device_id INTEGER CHECK (device_id > 0), 
    location INTEGER REFERENCES locations (id), 
    PRIMARY KEY(time, device_id)); 
SELECT create_hypertable('conditions', 'time');
```
这样创建的表，会有自动的约束条件
8. json 支持
```
CREATE TABLE metrics ( 
    time TIMESTAMPTZ, 
    user_id INT, 
    device_id INT, 
    data JSONB 
);
```
data是 JSONB类型，可以用GIN来对data进行index,

## data相关
### 分析
类似influxdb相关功能
### retention policy
类似influxdb相关
### backup
pass
### 数据告警
可以采用pg的生态
### 数据优化
- `timescaledb-tune`可以根据你的硬件来进行数据库设置的优化
- `timescaledb-parallel-copy`可以加速你的数据迁移


## 数据迁移
根据[influxdb迁移](https://docs.timescale.com/latest/getting-started/migrating-data#outflux)说明来进行influxdb到timescale的迁移


## 概念
### hypertable
