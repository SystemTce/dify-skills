# 存储服务集成

> Dify 与云存储服务集成指南

## 概述

Dify 支持与多种存储服务集成，包括：
- S3 兼容存储 (AWS S3, MinIO, 阿里云 OSS)
- 本地文件系统
- CDN 加速

## S3 兼容存储

### 基础工具

```python
import boto3
from botocore.exceptions import ClientError
from dify_plugin import Tool
from dify_plugin.entities.tool import ToolInvokeMessage
from typing import Generator
import base64

class S3Tool(Tool):
    """S3 存储工具"""

    def _get_client(self):
        """获取 S3 客户端"""
        return boto3.client(
            's3',
            endpoint_url=self.runtime_credentials.get("endpoint"),  # 自定义端点
            aws_access_key_id=self.runtime_credentials.get("access_key"),
            aws_secret_access_key=self.runtime_credentials.get("secret_key"),
            region_name=self.runtime_credentials.get("region", "us-east-1")
        )

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        operation = tool_parameters.get("operation")
        s3 = self._get_client()
        bucket = tool_parameters.get("bucket")
        key = tool_parameters.get("key")

        if operation == "upload":
            content = tool_parameters.get("content")
            content_type = tool_parameters.get("content_type", "text/plain")

            s3.put_object(
                Bucket=bucket,
                Key=key,
                Body=content.encode() if isinstance(content, str) else content,
                ContentType=content_type
            )
            yield self.create_text_message(
                f"Uploaded to s3://{bucket}/{key}"
            )

        elif operation == "download":
            response = s3.get_object(Bucket=bucket, Key=key)
            body = response['Body'].read()

            # 根据内容类型返回
            content_type = response.get('ContentType', '')
            if 'image' in content_type:
                yield self.create_image_message(
                    base64.b64encode(body).decode(),
                    content_type
                )
            else:
                yield self.create_text_message(body.decode())

        elif operation == "delete":
            s3.delete_object(Bucket=bucket, Key=key)
            yield self.create_text_message(f"Deleted s3://{bucket}/{key}")

        elif operation == "list":
            prefix = tool_parameters.get("prefix", "")
            max_keys = tool_parameters.get("max_keys", 100)

            response = s3.list_objects_v2(
                Bucket=bucket,
                Prefix=prefix,
                MaxKeys=max_keys
            )

            objects = []
            if 'Contents' in response:
                for obj in response['Contents']:
                    objects.append({
                        "key": obj['Key'],
                        "size": obj['Size'],
                        "last_modified": obj['LastModified'].isoformat()
                    })

            yield self.create_json_message({
                "count": len(objects),
                "objects": objects
            })

        elif operation == "exists":
            try:
                s3.head_object(Bucket=bucket, Key=key)
                yield self.create_text_message("true")
            except ClientError:
                yield self.create_text_message("false")

        elif operation == "copy":
            source_key = tool_parameters.get("source_key")
            dest_key = tool_parameters.get("dest_key")

            copy_source = {'Bucket': bucket, 'Key': source_key}
            s3.copy_object(
                Bucket=bucket,
                Key=dest_key,
                CopySource=copy_source
            )
            yield self.create_text_message(
                f"Copied s3://{bucket}/{source_key} to s3://{bucket}/{dest_key}"
            )
```

### 预签名 URL

```python
class S3PresignedTool(Tool):
    """S3 预签名 URL 工具"""

    def _get_client(self):
        return boto3.client(
            's3',
            endpoint_url=self.runtime_credentials.get("endpoint"),
            aws_access_key_id=self.runtime_credentials.get("access_key"),
            aws_secret_access_key=self.runtime_credentials.get("secret_key")
        )

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        operation = tool_parameters.get("operation")
        s3 = self._get_client()
        bucket = tool_parameters.get("bucket")
        key = tool_parameters.get("key")

        if operation == "generate_presigned_url":
            expires_in = tool_parameters.get("expires_in", 3600)

            url = s3.generate_presigned_url(
                'get_object',
                Params={'Bucket': bucket, 'Key': key},
                ExpiresIn=expires_in
            )
            yield self.create_json_message({
                "url": url,
                "expires_in": expires_in
            })

        elif operation == "generate_upload_url":
            content_type = tool_parameters.get("content_type", "application/octet-stream")
            expires_in = tool_parameters.get("expires_in", 3600)

            url = s3.generate_presigned_url(
                'put_object',
                Params={
                    'Bucket': bucket,
                    'Key': key,
                    'ContentType': content_type
                },
                ExpiresIn=expires_in
            )
            yield self.create_json_message({
                "url": url,
                "key": key,
                "content_type": content_type,
                "expires_in": expires_in
            })
```

## 阿里云 OSS

```python
import oss2

class OSSTool(Tool):
    """阿里云 OSS 工具"""

    def _get_client(self):
        """获取 OSS 客户端"""
        auth = oss2.Auth(
            self.runtime_credentials.get("access_key_id"),
            self.runtime_credentials.get("access_key_secret")
        )
        return oss2.Bucket(
            auth,
            self.runtime_credentials.get("endpoint"),
            self.runtime_credentials.get("bucket")
        )

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        operation = tool_parameters.get("operation")
        oss = self._get_client()
        key = tool_parameters.get("key")

        if operation == "upload":
            content = tool_parameters.get("content")
            oss.put_object(key, content.encode() if isinstance(content, str) else content)
            yield self.create_text_message(f"Uploaded to oss://{key}")

        elif operation == "download":
            result = oss.get_object(key)
            content = result.read()
            yield self.create_text_message(content.decode())

        elif operation == "delete":
            oss.delete_object(key)
            yield self.create_text_message(f"Deleted oss://{key}")

        elif operation == "list":
            prefix = tool_parameters.get("prefix", "")
            files = list(oss.object_iterate(prefix, max_keys=100))
            yield self.create_json_message({
                "count": len(files),
                "files": [{"key": f.key, "size": f.size} for f in files]
            })
```

## 本地文件系统

```python
import os
import shutil
from pathlib import Path

class FileSystemTool(Tool):
    """本地文件系统工具"""

    def __init__(self):
        super().__init__()
        self.base_path = Path("/tmp/dify-files")

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        operation = tool_parameters.get("operation")
        path = tool_parameters.get("path")

        # 确保路径安全（防止路径穿越）
        full_path = (self.base_path / path).resolve()
        if not str(full_path).startswith(str(self.base_path)):
            yield self.create_text_message("Error: Invalid path")
            return

        if operation == "read":
            if full_path.is_file():
                content = full_path.read_text()
                yield self.create_text_message(content)
            else:
                yield self.create_text_message("Error: Not a file")

        elif operation == "write":
            content = tool_parameters.get("content")
            full_path.parent.mkdir(parents=True, exist_ok=True)
            full_path.write_text(content)
            yield self.create_text_message(f"Written to {path}")

        elif operation == "delete":
            if full_path.is_file():
                full_path.unlink()
            elif full_path.is_dir():
                shutil.rmtree(full_path)
            yield self.create_text_message(f"Deleted {path}")

        elif operation == "list":
            if full_path.is_dir():
                files = [f.name for f in full_path.iterdir()]
                yield self.create_json_message({"files": files})
            else:
                yield self.create_text_message("Error: Not a directory")

        elif operation == "exists":
            exists = full_path.exists()
            yield self.create_text_message(str(exists))
```

## 批量操作

```python
class BatchStorageTool(Tool):
    """批量存储工具"""

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        operation = tool_parameters.get("operation")
        items = tool_parameters.get("items", [])

        results = []
        for item in items:
            try:
                result = self._process_item(operation, item)
                results.append({"success": True, "result": result})
            except Exception as e:
                results.append({"success": False, "error": str(e)})

        yield self.create_json_message({
            "total": len(items),
            "success": sum(1 for r in results if r["success"]),
            "failed": sum(1 for r in results if not r["success"]),
            "results": results
        })

    def _process_item(self, operation: str, item: dict):
        """处理单个项目"""
        # 实现具体的处理逻辑
        pass
```
