# 数据库集成

> Dify 与各类数据库的集成指南

## 概述

Dify 可以通过插件与各种数据库系统集成，实现数据持久化和查询。

## PostgreSQL/MySQL

### 基础连接

```python
import psycopg2  # PostgreSQL
# 或
import pymysql  # MySQL

from dify_plugin import Tool
from dify_plugin.entities.tool import ToolInvokeMessage
from typing import Generator, Any
import json

class DatabaseTool(Tool):
    """数据库工具基类"""

    def __init__(self):
        super().__init__()
        self._conn = None

    def _get_connection(self):
        """获取数据库连接"""
        if self._conn is None:
            self._conn = psycopg2.connect(
                host=self.runtime_credentials.get("host"),
                port=self.runtime_credentials.get("port", 5432),
                database=self.runtime_credentials.get("database"),
                user=self.runtime_credentials.get("user"),
                password=self.runtime_credentials.get("password")
            )
        return self._conn

    def _close_connection(self):
        """关闭连接"""
        if self._conn:
            self._conn.close()
            self._conn = None
```

### PostgreSQL 工具

```python
class PostgresTool(DatabaseTool):
    """PostgreSQL 工具"""

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        operation = tool_parameters.get("operation")
        query = tool_parameters.get("query")

        conn = self._get_connection()
        cursor = conn.cursor()

        try:
            cursor.execute(query)

            if operation == "select":
                # 查询操作
                columns = [desc[0] for desc in cursor.description]
                rows = cursor.fetchall()

                result = {
                    "columns": columns,
                    "rows": [dict(zip(columns, row)) for row in rows],
                    "count": len(rows)
                }
                yield self.create_json_message(result)

            else:
                # 增删改操作
                conn.commit()
                yield self.create_text_message(
                    f"Affected rows: {cursor.rowcount}"
                )

        except Exception as e:
            conn.rollback()
            yield self.create_text_message(f"Error: {str(e)}")

        finally:
            cursor.close()
            self._close_connection()
```

### MySQL 工具

```python
import pymysql

class MySQLTool(Tool):
    """MySQL 工具"""

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        operation = tool_parameters.get("operation")
        query = tool_parameters.get("query")

        conn = pymysql.connect(
            host=self.runtime_credentials.get("host"),
            port=self.runtime_credentials.get("port", 3306),
            database=self.runtime_credentials.get("database"),
            user=self.runtime_credentials.get("user"),
            password=self.runtime_credentials.get("password"),
            charset='utf8mb4'
        )

        cursor = conn.cursor()

        try:
            cursor.execute(query)

            if operation == "select":
                columns = [desc[0] for desc in cursor.description]
                rows = cursor.fetchall()

                result = {
                    "columns": columns,
                    "rows": [dict(zip(columns, row)) for row in rows],
                    "count": len(rows)
                }
                yield self.create_json_message(result)
            else:
                conn.commit()
                yield self.create_text_message(
                    f"Affected rows: {cursor.rowcount}"
                )

        finally:
            cursor.close()
            conn.close()
```

## Redis

### 基本操作

```python
import redis

class RedisTool(Tool):
    """Redis 工具"""

    def _get_client(self):
        """获取 Redis 客户端"""
        return redis.Redis(
            host=self.runtime_credentials.get("host"),
            port=self.runtime_credentials.get("port", 6379),
            password=self.runtime_credentials.get("password"),
            db=self.runtime_credentials.get("db", 0),
            decode_responses=True
        )

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        operation = tool_parameters.get("operation")
        r = self._get_client()

        if operation == "get":
            key = tool_parameters.get("key")
            value = r.get(key)
            yield self.create_text_message(value or "")

        elif operation == "set":
            key = tool_parameters.get("key")
            value = tool_parameters.get("value")
            ttl = tool_parameters.get("ttl")  # 可选过期时间
            if ttl:
                r.setex(key, ttl, value)
            else:
                r.set(key, value)
            yield self.create_text_message("OK")

        elif operation == "delete":
            key = tool_parameters.get("key")
            r.delete(key)
            yield self.create_text_message("Deleted")

        elif operation == "exists":
            key = tool_parameters.get("key")
            exists = r.exists(key)
            yield self.create_text_message(str(bool(exists)))

        elif operation == "keys":
            pattern = tool_parameters.get("pattern", "*")
            keys = r.keys(pattern)
            yield self.create_json_message({"keys": keys})

        elif operation == "incr":
            key = tool_parameters.get("key")
            amount = tool_parameters.get("amount", 1)
            value = r.incrby(key, amount)
            yield self.create_text_message(str(value))
```

### 高级功能

```python
class RedisAdvancedTool(Tool):
    """Redis 高级工具"""

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        operation = tool_parameters.get("operation")
        r = self._get_client()

        if operation == "hash":
            # Hash 操作
            key = tool_parameters.get("key")
            hash_operation = tool_parameters.get("hash_operation")

            if hash_operation == "hset":
                field = tool_parameters.get("field")
                value = tool_parameters.get("value")
                r.hset(key, field, value)
                yield self.create_text_message("OK")

            elif hash_operation == "hget":
                field = tool_parameters.get("field")
                value = r.hget(key, field)
                yield self.create_text_message(value or "")

            elif hash_operation == "hgetall":
                data = r.hgetall(key)
                yield self.create_json_message(data)

        elif operation == "list":
            # List 操作
            key = tool_parameters.get("key")
            list_operation = tool_parameters.get("list_operation")

            if list_operation == "lpush":
                value = tool_parameters.get("value")
                r.lpush(key, value)
                yield self.create_text_message("OK")

            elif list_operation == "lrange":
                start = tool_parameters.get("start", 0)
                end = tool_parameters.get("end", -1)
                data = r.lrange(key, start, end)
                yield self.create_json_message({"items": data})

        elif operation == "pub/sub":
            # 发布/订阅
            channel = tool_parameters.get("channel")
            pubsub_operation = tool_parameters.get("pubsub_operation")

            if pubsub_operation == "publish":
                message = tool_parameters.get("message")
                subscribers = r.publish(channel, message)
                yield self.create_text_message(f"Message published to {subscribers} subscribers")
```

## MongoDB

```python
from pymongo import MongoClient

class MongoDBTool(Tool):
    """MongoDB 工具"""

    def _get_client(self):
        """获取 MongoDB 客户端"""
        return MongoClient(
            host=self.runtime_credentials.get("host"),
            port=self.runtime_credentials.get("port", 27017),
            username=self.runtime_credentials.get("user"),
            password=self.runtime_credentials.get("password")
        )

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        operation = tool_parameters.get("operation")
        client = self._get_client()
        db = client[self.runtime_credentials.get("database")]
        collection = db[tool_parameters.get("collection")]

        if operation == "find":
            query = tool_parameters.get("query", {})
            limit = tool_parameters.get("limit", 10)

            cursor = collection.find(query).limit(limit)
            results = [doc for doc in cursor]
            yield self.create_json_message({
                "count": len(results),
                "documents": results
            })

        elif operation == "insert":
            document = tool_parameters.get("document")
            result = collection.insert_one(document)
            yield self.create_json_message({
                "inserted_id": str(result.inserted_id)
            })

        elif operation == "update":
            query = tool_parameters.get("query")
            update = tool_parameters.get("update")
            result = collection.update_many(query, update)
            yield self.create_json_message({
                "matched_count": result.matched_count,
                "modified_count": result.modified_count
            })

        elif operation == "delete":
            query = tool_parameters.get("query")
            result = collection.delete_many(query)
            yield self.create_json_message({
                "deleted_count": result.deleted_count
            })
```

## 连接池管理

```python
from queue import Queue
import threading

class ConnectionPool:
    """数据库连接池"""

    def __init__(self, create_connection, pool_size: int = 10):
        self.pool = Queue(maxsize=pool_size)
        self.create_connection = create_connection

        # 初始化连接池
        for _ in range(pool_size):
            self.pool.put(create_connection())

    def get_connection(self, timeout: float = 30.0):
        """获取连接"""
        return self.pool.get(timeout=timeout)

    def return_connection(self, conn):
        """归还连接"""
        self.pool.put(conn)

    def close_all(self):
        """关闭所有连接"""
        while not self.pool.empty():
            conn = self.pool.get()
            conn.close()


class PooledDatabaseTool(Tool):
    """带连接池的数据库工具"""

    def __init__(self):
        super().__init__()
        self._pool = None

    def _get_pool(self):
        """获取连接池"""
        if self._pool is None:
            self._pool = ConnectionPool(
                create_connection=self._create_connection,
                pool_size=5
            )
        return self._pool

    def _create_connection(self):
        """创建新连接"""
        return psycopg2.connect(
            host=self.runtime_credentials.get("host"),
            database=self.runtime_credentials.get("database"),
            user=self.runtime_credentials.get("user"),
            password=self.runtime_credentials.get("password")
        )

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        pool = self._get_pool()
        conn = pool.get_connection()

        try:
            # 执行操作
            cursor = conn.cursor()
            cursor.execute(tool_parameters.get("query"))
            result = cursor.fetchall()
            cursor.close()

            yield self.create_json_message({"rows": result})

        finally:
            pool.return_connection(conn)
```
