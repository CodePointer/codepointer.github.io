---
  title: Mr. Gingerpaw Project Log - Environment Variable Configuration 
  description: >-
    A development log for the Mr. Gingerpaw Inventory App, documenting how to configure environment variables for frontend and backend across development and production.
  # author: pointer
  date: 2025-06-10 16:17:00 +0930
  categories: [Blogging, MrGingerpaw]
  tags: [Python, React, Deployment]
  # media_subpath: /images/2022-08-10-ipv6-version-of-frp/
  pin: false
---

## Introduction

### Background

The frontend and backend applications have some variable differences between development and production environments.

For example, I use a separate testing database, and both the frontend and backend during testing connect to different services compared to the production setup. So, I need to load some variables from the environment automatically.

To handle this, I need to configure environment variables for both the frontend and backend software. What I initially thought would be simple turned out to be full of unexpected issues — definitely worth writing a dedicated blog post about.

### Requirements

I need to configure separate sets of environment variables for both frontend and backend applications. This is because I may be developing the frontend or backend independently, and during development, I need to connect each to different services.

My frontend platform uses *Expo Router*, and the backend is based on *FastAPI*.

Moreover, the environment variables must switch automatically. Once deployed, the production version should be used without manual changes; while locally, I want to easily toggle between development and production modes for efficient development and testing.

### Solutions

Given the above needs, I decided to manage environment variables using the following strategies:

1. **Backend:** Managed via `Pydantic v2 (pydantic-settings)` and separate `.env.[development/production]` files.

2. **Frontend:** Leveraging Expo CLI's built-in environment variable support, using a `.env` file with `NODE_ENV`, along with `.env.[development/production]` files for each mode.

3. The above configurations are used during local development. For deployment, relevant environment variables are configured via Azure or GitHub.


## Backend: Pydantic v2 (pydantic-settings)

My backend code needs to handle the following environment variables:

```ini
SECRET_KEY="SOMESECRETKEYHERE"  # Key for JWT
ALGORITHM="XXX"
DATABASE_URL=http://myadminname:mypassword@localhost:5432/mydatabase
ACCESS_TOKEN_EXPIRE_MINUTES=1440  # 24 * 60
RESET_TOKEN_EXPIRE_MINUTES=720
CORS_ORIGINS=http://localhost:8081,http://192.168.1.128:8081
```

Most variables are `string` or `int` types, but `CORS_ORIGINS` is a `list` of URLs and needs special handling in code.

I use `Pydantic v2` to manage these variables. First, add the following to `requirements.txt`:

```txt
pydantic>=2.0
pydantic-settings
```

> Note: In Pydantic v2, settings-related features have been moved to the `pydantic-settings` package. So we must install it separately.
{: .prompt-warning }

Once installed, I define the `Settings` class as follows:

```python
# backend/app/core/config.py

import os
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field

class Settings(BaseSettings):
    SECRET_KEY: str
    ALGORITHM: str
    DATABASE_URL: str
    ACCESS_TOKEN_EXPIRE_MINUTES: int
    RESET_TOKEN_EXPIRE_MINUTES: int

    model_config = SettingsConfigDict(
        env_file = f".env.{os.getenv('APP_ENV', 'development')}",
        env_file_encoding="utf-8"
    )

settings = Settings()
```

This works for everything except `CORS_ORIGINS`. To handle that, I explored two approaches.

### Parsing `list[*]` Type Variables

**Option 1: Use JSON Format**

```python
# backend/.env.*
CORS_ORIGINS=["http://localhost:8081","http://192.168.1.128:8081"]

# backend/app/core/config.py
class Settings(BaseSettings):
    ...
    CORS_ORIGINS: list[str]
    ...
```

This uses Pydantic's default JSON parsing. But the format is verbose and error-prone with escaping. If we prefer a simpler comma-separated format, we need a different solution.

**Option 2: Custom Parsing via Intermediate Property**

```python
# backend/.env.*
CORS_ORIGINS=http://localhost:8081,http://192.168.1.128:8081

# backend/app/core/config.py
class Settings(BaseSettings):
    ...
    RAW_CORS_ORIGINS: str = Field("", alias="CORS_ORIGINS")

    @property
    def CORS_ORIGINS(self):
        return [u.strip() for u in self.RAW_CORS_ORIGINS.split(",") if u.strip()]
    ...
```

This allows custom parsing and keeps the env format simple.

> Pitfalls to avoid:
>
> - `@field_validator` or `@model_validator`: Won’t work if initial parsing fails (e.g., JSONDecodeError). Validation only happens after raw values are loaded.
> - Using `Field(..., env=None)` to block JSON parsing: Doesn’t work in Pydantic v2.
> - `env="CORS_ORIGINS"` should now be `alias="CORS_ORIGINS"` in v2. Many docs you can find from internet (especially when you query this to AI) are based on v1.
{: .prompt-warning }

Anyway, I went with **Option 2** in the end. Here’s my final code:

```python
# backend/app/core/config.py

import os
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field

class Settings(BaseSettings):
    SECRET_KEY: str
    ALGORITHM: str
    DATABASE_URL: str
    ACCESS_TOKEN_EXPIRE_MINUTES: int
    RESET_TOKEN_EXPIRE_MINUTES: int
    RAW_CORS_ORIGINS: str = Field("", alias="CORS_ORIGINS")

    @property
    def CORS_ORIGINS(self):
        return [u.strip() for u in self.RAW_CORS_ORIGINS.split(",") if u.strip()]

    model_config = SettingsConfigDict(
        env_file = f".env.{os.getenv('APP_ENV', 'development')}",
        env_file_encoding="utf-8"
    )

settings = Settings()
```

To use it:

```python
# backend/app/main.py
from fastapi import FastAPI
from app.core.config import settings

app = FastAPI(title="Mr.Gingerpaw API")
app.add_middleware(
    ...
    allow_origins=settings.CORS_ORIGINS,
    ...
)
```

## Frontend: Expo CLI Built-in Env Support

Since I use SDK `53.0`, Expo CLI can now auto-load env files via `NODE_ENV`. Specifically, I need these files:

```ini
# frontend/.env
NODE_ENV=development

# frontend/.env.development
EXPO_PUBLIC_API_BASE_URL=http://127.0.0.1:8000

# frontend/.env.production
EXPO_PUBLIC_API_BASE_URL=https://[MyServiceName].azurewebsites.net
```

Switching between environments is as simple as updating `NODE_ENV`. Note that variables **must** start with `EXPO_PUBLIC_` to be inlined into code.

> Do **not** use `react-native-dotenv`. It overrides Expo Router’s Babel config and breaks route detection in `app/`. 
{: .prompt-warning }

You can then access variables like this:

```js
// frontend/services/utils/axioInstance.ts
const API_BASE_URL = process.env.EXPO_PUBLIC_API_BASE_URL;
```

## Summary

- **Backend:** Use `Pydantic v2 (pydantic-settings)` with `.env.[development/production]` files.

  - `string` and `int` types work directly.
  - `list` types need JSON format or custom parsing via `@property`.

- **Frontend:** Use Expo CLI’s built-in env support.

  - Use `.env` + `.env.[development/production]`.
  - Don’t use `react-native-dotenv`, it breaks Expo Router.


That’s it!
