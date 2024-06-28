---
title: slurm--JSON Web Tokens (JWT) Authentication 
description: 
published: true
date: 2024-06-28T14:35:33.338Z
tags: slurm
editor: markdown
dateCreated: 2024-06-28T14:35:33.338Z
---



Slurm 提供符合 RFC7519 标准的 [JSON Web Tokens](https://jwt.io/) (JWT) 实现。这种身份验证可用作 **AuthAltType**，通常与 **auth/munge** 并列为 **AuthType**。唯一支持的通信方向是从客户端连接到 **slurmctld** 和 **slurmdbd**。这意味着启用了 **auth/jwt** 的客户端（或环境中有 SLURM_JWT）目前不支持某些场景（特别是使用 **srun** 的交互式作业）。

## 前提

JWT 需要 [libjwt](https://slurm.schedmd.com/related_software.html#jwt)。编译 Slurm 时，库和开发头文件都必须可用。

## 独立使用设置

1. [配置和构建支持 JWT 的 Slurm](https://slurm.schedmd.com/related_software.html#jwt)

2. 在状态保存位置的控制器中添加 JWT 密钥。下面是一个示例，JWT 密钥位于 /var/spool/slurm/statesave/：

   ```
   dd if=/dev/random of=/var/spool/slurm/statesave/jwt_hs256.key bs=32 count=1
   chown slurm:slurm /var/spool/slurm/statesave/jwt_hs256.key
   chmod 0600 /var/spool/slurm/statesave/jwt_hs256.key
   chown slurm:slurm /var/spool/slurm/statesave
   chmod 0755 /var/spool/slurm/statesave
   ```

   密钥不一定要放在 StateSaveLocation 中，但如果有多个控制器，这个位置会比较方便，因为控制器之间可以共享密钥。密钥不应放在非管理员用户可能访问的目录中。密钥文件应为 **SlurmUser** 或 **root** 所有，建议权限为 0400。该文件不能被'other'访问。

3. 在 slurm.conf 和 slurmdbd.conf 中添加 JWT 作为替代身份验证：

   ```
   AuthAltTypes=auth/jwt
   AuthAltParameters=jwt_key=/var/spool/slurm/statesave/jwt_hs256.key
   ```

4. 重启slurmctld

5. 根据需要为用户创建令牌

   ```
   scontrol token username=$USER
   ```

   可以使用可选的 **lifespan=$LIFESPAN** 选项来更改令牌寿命（默认为 1800 秒）。根账户或 SlurmUser 账户可用于为任何用户生成令牌。另外，用户也可以使用该命令为自己生成令牌，只需调用

   ```
   scontrol token
   ```

   请注意，管理员可以通过在 slurm.conf 中设置以下参数来阻止用户生成令牌：

   ```
   AuthAltParameters=disable_token_creation
   ```

   提供该功能是为了让网站控制向用户提供令牌的时间和方式，以及控制令牌的有效期。

6. 在调用任何 Slurm 命令前导出 **SLURM_JWT** 环境变量。
7. 在启动 slurmrestd 守护进程前导出 **SLURM_JWT=daemon** 环境变量，以激活 *AuthAltTypes=auth/jwt* 作为主要身份验证机制。

## 外部认证与 JWKS 和 RS256 令牌集成

从 21.08 版开始，Slurm 可以支持 RS256 令牌，例如由 [Amazon Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-verifying-a-jwt.html)、[Azure AD](https://azure.github.io/azure-workload-identity/docs/installation/self-managed-clusters/oidc-issuer/jwks.html) 或 [Keycloak](https://www.keycloak.org/docs/latest/securing_apps/#_client_authentication_adapter) 生成的令牌。

要启用 Slurm 的 RS256 令牌支持，必须下载并配置适当的 JWKS 文件：

```
AuthAltTypes=auth/jwt
AuthAltParameters=jwks=/var/spool/slurm/statesave/jwks.json
```

jwks 文件应为 **SlurmUser** 或 **root** 所有，必须为 **SlurmUser** 可读，建议权限为 0400。该文件不能被 'other'写入。

请注意，默认情况下，当启用 JWKS 支持时，生成 HS256 标记的内置功能将被禁用。可以通过显式配置 **jwt_key=** 选项和 jwks= 选项来重新启用该功能。

注意：Slurm 会忽略 **x5c** 和 **x5t** 字段，并且不会尝试验证 JWKS 文件中的证书链。JWT 只根据通过 **e** 和 **n** 字段提供的 RSA 256 位密钥进行验证。

## Compatibility 

Slurm 使用 libjwt 来查看和验证 RFC7519 JWT 标记。只要满足以下要求，就可以使用其他解决方案生成的兼容令牌：

1. 存在 Slurm 所需的令牌：
   - iat: Unix timestamp of creation date.
   - exp: Unix timestamp of expiration date.
   - sun or username: Slurm UserName ( [POSIX.1-2017 User Name ](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap03.html#tag_03_437)).
2. 令牌采用符合 RFC7518 标准的 HS256 算法签名。尽管 Slurm 无法直接创建令牌，但也支持 RS256 来验证令牌。
3. 签名密钥提供给 slurmctld 和 slurmdbd，以便解密令牌。Slurm 目前只支持单个签名密钥。

以下脚本需要安装 JWT Python 模块。此脚本可作为生成 JWT 密钥供 Slurm 使用的示例。

```
#!/usr/bin/env python3
import sys
import os
import pprint
import json
import time
from datetime import datetime, timedelta, timezone

from jwt import JWT
from jwt.jwa import HS256
from jwt.jwk import jwk_from_dict
from jwt.utils import b64decode,b64encode

if len(sys.argv) != 3:
    sys.exit("gen_jwt.py [user name] [expiration time (seconds)]");

with open("/var/spool/slurm/statesave/jwt.key", "rb") as f:
    priv_key = f.read()

signing_key = jwk_from_dict({
    'kty': 'oct',
    'k': b64encode(priv_key)
})

message = {
    "exp": int(time.time() + int(sys.argv[2])),
    "iat": int(time.time()),
    "sun": sys.argv[1]
}

a = JWT()
compact_jws = a.encode(message, signing_key, alg='HS256')
print("SLURM_JWT={}".format(compact_jws))
```

同样，下面的脚本可以作为一个例子，说明如何验证 jwt 密钥是否可以与 Slurm 一起使用。

```
#!/usr/bin/env python3
import sys
import os
import pprint
import json
import time
from datetime import datetime, timedelta, timezone

from jwt import JWT
from jwt.jwa import HS256
from jwt.jwk import jwk_from_dict
from jwt.utils import b64decode,b64encode

if len(sys.argv) != 2:
    sys.exit("verify_jwt.py [JWT Token]");

with open("/var/spool/slurm/statesave/jwt.key", "rb") as f:
    priv_key = f.read()

signing_key = jwk_from_dict({
    'kty': 'oct',
    'k': b64encode(priv_key)
})

a = JWT()
b = a.decode(sys.argv[1], signing_key, algorithms=["HS256"])
print(b)
```

