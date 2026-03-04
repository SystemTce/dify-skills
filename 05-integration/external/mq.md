# 消息队列集成

> Dify 与消息队列系统集成指南

## 概述

Dify 可以与消息队列系统集成，实现异步处理和系统解耦：
- RabbitMQ
- Apache Kafka
- Redis Queue

## RabbitMQ

### 发布者工具

```python
import pika
from dify_plugin import Tool
from dify_plugin.entities.tool import ToolInvokeMessage
from typing import Generator
import json

class RabbitMQPublisherTool(Tool):
    """RabbitMQ 发布者工具"""

    def _get_connection(self):
        """获取连接"""
        credentials = pika.PlainCredentials(
            self.runtime_credentials.get("username"),
            self.runtime_credentials.get("password")
        )
        parameters = pika.ConnectionParameters(
            host=self.runtime_credentials.get("host"),
            port=self.runtime_credentials.get("port", 5672),
            credentials=credentials,
            heartbeat=600,
            blocked_connection_timeout=300
        )
        return pika.BlockingConnection(parameters)

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        operation = tool_parameters.get("operation")
        queue = tool_parameters.get("queue")

        connection = self._get_connection()
        channel = connection.channel()

        try:
            # 声明队列
            channel.queue_declare(queue=queue, durable=True)

            if operation == "publish":
                message = tool_parameters.get("message")
                # 支持字符串或 JSON
                if isinstance(message, dict):
                    body = json.dumps(message).encode()
                else:
                    body = message.encode()

                properties = pika.BasicProperties(
                    delivery_mode=2,  # 持久化
                    content_type='application/json'
                )

                channel.basic_publish(
                    exchange='',
                    routing_key=queue,
                    body=body,
                    properties=properties
                )
                yield self.create_text_message(f"Published to {queue}")

            elif operation == "publish_batch":
                messages = tool_parameters.get("messages", [])
                for msg in messages:
                    body = json.dumps(msg).encode() if isinstance(msg, dict) else msg.encode()
                    channel.basic_publish(
                        exchange='',
                        routing_key=queue,
                        body=body,
                        properties=pika.BasicProperties(delivery_mode=2)
                    )
                yield self.create_text_message(f"Published {len(messages)} messages")

        finally:
            connection.close()
```

### 消费者工具

```python
class RabbitMQConsumerTool(Tool):
    """RabbitMQ 消费者工具"""

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        queue = tool_parameters.get("queue")
        max_messages = tool_parameters.get("max_messages", 10)
        auto_ack = tool_parameters.get("auto_ack", False)

        connection = self._get_connection()
        channel = connection.channel()

        # 声明队列
        channel.queue_declare(queue=queue, durable=True)

        messages = []
        for _ in range(max_messages):
            method, properties, body = channel.basic_get(queue=queue, auto_ack=auto_ack)

            if body is None:
                break

            try:
                message = json.loads(body.decode())
            except json.JSONDecodeError:
                message = body.decode()

            messages.append({
                "body": message,
                "delivery_tag": method.delivery_tag if method else None
            })

            # 确认消息
            if not auto_ack and method:
                channel.basic_ack(method.delivery_tag)

        connection.close()

        yield self.create_json_message({
            "count": len(messages),
            "messages": messages
        })
```

## Apache Kafka

### 生产者工具

```python
from kafka import KafkaProducer, KafkaConsumer
from kafka.errors import KafkaError
from dify_plugin import Tool
from dify_plugin.entities.tool import ToolInvokeMessage
from typing import Generator
import json
import time

class KafkaTool(Tool):
    """Kafka 工具"""

    def _get_producer(self):
        """获取生产者"""
        return KafkaProducer(
            bootstrap_servers=self.runtime_credentials.get("servers").split(","),
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8') if k else None,
            acks='all',
            retries=3
        )

    def _get_consumer(self, topic: str, group_id: str):
        """获取消费者"""
        return KafkaConsumer(
            topic,
            bootstrap_servers=self.runtime_credentials.get("servers").split(","),
            group_id=group_id,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            auto_offset_reset='earliest',
            enable_auto_commit=True
        )

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        operation = tool_parameters.get("operation")
        topic = tool_parameters.get("topic")

        if operation == "publish":
            producer = self._get_producer()
            message = tool_parameters.get("message")
            key = tool_parameters.get("key")

            try:
                future = producer.send(
                    topic,
                    value=message,
                    key=key
                )
                record_metadata = future.get(timeout=10)

                yield self.create_json_message({
                    "success": True,
                    "topic": record_metadata.topic,
                    "partition": record_metadata.partition,
                    "offset": record_metadata.offset
                })
            except KafkaError as e:
                yield self.create_json_message({
                    "success": False,
                    "error": str(e)
                })
            finally:
                producer.close()

        elif operation == "publish_batch":
            producer = self._get_producer()
            messages = tool_parameters.get("messages", [])

            futures = []
            for msg in messages:
                future = producer.send(topic, value=msg)
                futures.append(future)

            # 等待所有消息发送完成
            for future in futures:
                future.get(timeout=30)

            producer.close()

            yield self.create_json_message({
                "success": True,
                "sent": len(messages)
            })

        elif operation == "consume":
            group_id = tool_parameters.get("group_id", "dify-consumer")
            max_messages = tool_parameters.get("max_messages", 10)
            timeout_ms = tool_parameters.get("timeout_ms", 5000)

            consumer = self._get_consumer(topic, group_id)

            messages = []
            start_time = time.time()

            for message in consumer:
                messages.append({
                    "key": message.key,
                    "value": message.value,
                    "partition": message.partition,
                    "offset": message.offset,
                    "timestamp": message.timestamp
                })

                if len(messages) >= max_messages:
                    break

                if (time.time() - start_time) * 1000 > timeout_ms:
                    break

            consumer.close()

            yield self.create_json_message({
                "count": len(messages),
                "messages": messages
            })

        elif operation == "create_topic":
            # 注意: 创建 topic 需要 admin 权限
            yield self.create_text_message(
                "Topic creation not implemented in plugin. "
                "Use Kafka admin tools to create topics."
            )
```

## Redis Queue

### 简单队列

```python
import redis
from dify_plugin import Tool
from dify_plugin.entities.tool import ToolInvokeMessage
from typing import Generator
import json

class RedisQueueTool(Tool):
    """Redis 队列工具"""

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
        queue_name = tool_parameters.get("queue")
        r = self._get_client()

        if operation == "push":
            message = tool_parameters.get("message")
            if isinstance(message, dict):
                message = json.dumps(message)
            r.lpush(queue_name, message)
            yield self.create_text_message(f"Pushed to {queue_name}")

        elif operation == "pop":
            timeout = tool_parameters.get("timeout", 0)
            result = r.brpop(queue_name, timeout=timeout)

            if result:
                _, message = result
                try:
                    data = json.loads(message)
                except json.JSONDecodeError:
                    data = message
                yield self.create_json_message({"message": data})
            else:
                yield self.create_json_message({"message": None})

        elif operation == "length":
            length = r.llen(queue_name)
            yield self.create_text_message(str(length))

        elif operation == "peek":
            start = tool_parameters.get("start", 0)
            end = tool_parameters.get("end", -1)
            messages = r.lrange(queue_name, start, end)

            try:
                data = [json.loads(m) for m in messages]
            except json.JSONDecodeError:
                data = messages

            yield self.create_json_message({"messages": data})
```

### 发布/订阅

```python
class RedisPubSubTool(Tool):
    """Redis 发布/订阅工具"""

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        operation = tool_parameters.get("operation")
        channel = tool_parameters.get("channel")
        r = self._get_client()

        if operation == "publish":
            message = tool_parameters.get("message")
            if isinstance(message, dict):
                message = json.dumps(message)

            subscribers = r.publish(channel, message)
            yield self.create_text_message(
                f"Published to {subscribers} subscribers"
            )

        elif operation == "subscribe":
            # 注意: 订阅是阻塞操作，通常在独立线程中运行
            pubsub = r.pubsub()
            pubsub.subscribe(channel)

            messages = []
            for message in pubsub.listen():
                if message['type'] == 'message':
                    try:
                        data = json.loads(message['data'])
                    except json.JSONDecodeError:
                        data = message['data']
                    messages.append(data)

                if len(messages) >= 10:  # 限制接收数量
                    break

            pubsub.unsubscribe(channel)
            pubsub.close()

            yield self.create_json_message({"messages": messages})
```

## 死信队列处理

```python
class DLQTool(Tool):
    """死信队列处理工具"""

    def __init__(self):
        super().__init__()
        self.dlq_suffix = "_dlq"

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        operation = tool_parameters.get("operation")
        main_queue = tool_parameters.get("queue")

        r = self._get_client()
        dlq_name = main_queue + self.dlq_suffix

        if operation == "get_dlq_messages":
            messages = r.lrange(dlq_name, 0, -1)
            parsed = []
            for msg in messages:
                try:
                    parsed.append(json.loads(msg))
                except json.JSONDecodeError:
                    parsed.append(msg)

            yield self.create_json_message({
                "count": len(parsed),
                "messages": parsed
            })

        elif operation == "retry":
            # 从 DLQ 重新入队到主队列
            while r.llen(dlq_name) > 0:
                msg = r.rpop(dlq_name)
                if msg:
                    r.lpush(main_queue, msg)

            yield self.create_text_message("Messages retried")

        elif operation == "clear_dlq":
            r.delete(dlq_name)
            yield self.create_text_message("DLQ cleared")
```
