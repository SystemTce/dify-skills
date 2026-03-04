# OAuth 2.0 集成指南

## OAuth 流程概述

Dify 支持完整的 OAuth 2.0 授权流程：

```
1. 用户点击授权
   ↓
2. 跳转到第三方授权页面
   ↓
3. 用户授权
   ↓
4. 返回授权码
   ↓
5. 交换访问令牌
   ↓
6. 使用访问令牌调用 API
   ↓
7. 令牌过期时刷新
```

## 配置 OAuth Schema

在 `provider.yaml` 中配置：

```yaml
oauth_schema:
  client_schema:
    - name: client_id
      type: secret-input
      required: true
      label:
        en_US: Client ID
        zh_Hans: 客户端 ID
      help:
        en_US: Get from service provider
        zh_Hans: 从服务提供商获取
      url: https://console.example.com/credentials

    - name: client_secret
      type: secret-input
      required: true
      label:
        en_US: Client Secret
        zh_Hans: 客户端密钥

  credentials_schema:
    - name: access_token
      type: secret-input
      label:
        en_US: Access Token
        zh_Hans: 访问令牌

    - name: refresh_token
      type: secret-input
      label:
        en_US: Refresh Token
        zh_Hans: 刷新令牌
```

## 实现 OAuth Provider

```python
from typing import Any
from dify_plugin import ToolProvider
from dify_plugin.errors.tool import ToolProviderCredentialValidationError
import requests


class MyOAuthProvider(ToolProvider):
    def _validate_credentials(self, credentials: dict[str, Any]) -> None:
        """验证 OAuth 凭证"""
        access_token = credentials.get("access_token")
        if not access_token:
            raise ToolProviderCredentialValidationError(
                "Access token is required"
            )

        # 测试访问令牌
        try:
            response = requests.get(
                "https://api.example.com/user",
                headers={"Authorization": f"Bearer {access_token}"}
            )
            response.raise_for_status()
        except Exception as e:
            raise ToolProviderCredentialValidationError(
                f"Invalid access token: {str(e)}"
            )
```

## 使用 OAuth 令牌

```python
from dify_plugin import Tool


class MyOAuthTool(Tool):
    def _invoke(self, tool_parameters):
        # 获取 OAuth 凭证
        credentials = self.runtime.credentials
        access_token = credentials.get("access_token")

        # 调用 API
        response = requests.get(
            "https://api.example.com/data",
            headers={"Authorization": f"Bearer {access_token}"}
        )

        yield self.create_text_message(response.text)
```

## 令牌刷新

Dify 自动处理令牌刷新，你只需要在 API 返回 401 时抛出异常：

```python
response = requests.get(url, headers=headers)
if response.status_code == 401:
    raise ToolProviderCredentialValidationError("Token expired")
```

## 完整示例：Google Contacts

### provider.yaml

```yaml
oauth_schema:
  client_schema:
    - name: client_id
      type: secret-input
      required: true
      label:
        en_US: Client ID
      url: https://console.cloud.google.com/apis/credentials

    - name: client_secret
      type: secret-input
      required: true
      label:
        en_US: Client Secret

  credentials_schema:
    - name: access_token
      type: secret-input
      label:
        en_US: Access Token

    - name: refresh_token
      type: secret-input
      label:
        en_US: Refresh Token
```

### provider.py

```python
from typing import Any
from dify_plugin import ToolProvider
from dify_plugin.errors.tool import ToolProviderCredentialValidationError
import requests


class GoogleContactsProvider(ToolProvider):
    def _validate_credentials(self, credentials: dict[str, Any]) -> None:
        access_token = credentials.get("access_token")
        if not access_token:
            raise ToolProviderCredentialValidationError(
                "Access token is required"
            )

        try:
            response = requests.get(
                "https://people.googleapis.com/v1/people/me",
                headers={"Authorization": f"Bearer {access_token}"}
            )
            response.raise_for_status()
        except Exception as e:
            raise ToolProviderCredentialValidationError(str(e))
```

### tool.py

```python
from typing import Any, Generator
from dify_plugin.entities.tool import ToolInvokeMessage
from dify_plugin import Tool
import requests


class ListContacts(Tool):
    def _invoke(
        self, tool_parameters: dict[str, Any]
    ) -> Generator[ToolInvokeMessage, None, None]:
        credentials = self.runtime.credentials
        access_token = credentials.get("access_token")

        try:
            response = requests.get(
                "https://people.googleapis.com/v1/people/me/connections",
                headers={"Authorization": f"Bearer {access_token}"},
                params={"personFields": "names,emailAddresses"}
            )
            response.raise_for_status()

            contacts = response.json().get("connections", [])
            result = "\n".join([
                f"{c.get('names', [{}])[0].get('displayName', 'N/A')}: "
                f"{c.get('emailAddresses', [{}])[0].get('value', 'N/A')}"
                for c in contacts
            ])

            yield self.create_text_message(result)

        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 401:
                raise ToolProviderCredentialValidationError("Token expired")
            yield self.create_text_message(f"API Error: {str(e)}")
        except Exception as e:
            yield self.create_text_message(f"Error: {str(e)}")
```

## PRIVACY.md

使用 OAuth 的插件必须提供 PRIVACY.md 文件：

```markdown
# Privacy Policy

## Data Collection

This plugin collects:
- User's contact information from Google Contacts
- Access tokens for API authentication

## Data Usage

Data is used only for:
- Displaying contacts in Dify workflows
- No data is stored permanently

## Data Sharing

We do not share your data with third parties.

## Security

- All tokens are encrypted
- Tokens expire after 1 hour
- Refresh tokens are securely stored
```

## 最佳实践

1. **最小权限**：只请求必需的 OAuth scope
2. **令牌安全**：使用 `secret-input` 类型存储令牌
3. **错误处理**：正确处理 401 错误触发令牌刷新
4. **隐私政策**：提供清晰的 PRIVACY.md
5. **用户体验**：提供清晰的授权说明
