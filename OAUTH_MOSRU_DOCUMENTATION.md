# OAuth 2.0 авторизация через mos.ru для МЭШ

## Оглавление
1. [Общие сведения](#общие-сведения)
2. [PKCE (Proof Key for Code Exchange)](#pkce-proof-key-for-code-exchange)
3. [Константы и конфигурация](#константы-и-конфигурация)
4. [Процесс авторизации](#процесс-авторизации)
5. [Регистрация приложения](#регистрация-приложения)
6. [Генерация PKCE](#генерация-pkce)
7. [Формирование URL авторизации](#формирование-url-авторизации)
8. [Обработка callback](#обработка-callback)
9. [Обмен кода на токен](#обмен-кода-на-токен)
10. [Получение токена МЭШ](#получение-токена-мэш)
11. [Обновление токенов](#обновление-токенов)
12. [Примеры кода](#примеры-кода)
13. [Безопасность](#безопасность)
14. [Ошибки и их обработка](#ошибки-и-их-обработка)

## Общие сведения

OAuth 2.0 авторизация через mos.ru используется для получения доступа к API МЭШ (Московская Электронная Школа). Процесс включает несколько этапов:

1. **Регистрация приложения** - получение client_id и client_secret
2. **PKCE генерация** - создание code_verifier и code_challenge для безопасности
3. **Перенаправление на mos.ru** - пользователь авторизуется на портале
4. **Получение кода авторизации** - mos.ru возвращает код через callback
5. **Обмен кода на токен mos.ru** - получение access_token и refresh_token
6. **Обмен токена mos.ru на токен МЭШ** - получение токена для API МЭШ

## PKCE (Proof Key for Code Exchange)

PKCE - это расширение OAuth 2.0, которое повышает безопасность авторизации, особенно для мобильных и одностраничных приложений.

### Принцип работы PKCE:
1. Клиент генерирует случайную строку `code_verifier`
2. Создает `code_challenge` = Base64UrlSafe(SHA256(code_verifier))
3. Отправляет `code_challenge` при запросе авторизации
4. При обмене кода на токен отправляет оригинальный `code_verifier`
5. Сервер проверяет соответствие challenge и verifier

## Константы и конфигурация

```javascript
const OAUTH_CONFIG = {
  // Базовые URL
  MOS_AUTH_BASE_URL: "https://login.mos.ru/",
  MOS_AUTH_GATE_URL: "https://login.mos.ru/sps/oauth/ae",
  MOS_TOKEN_URL: "https://login.mos.ru/sps/oauth/te",
  MOS_ISSUE_URL: "https://login.mos.ru/sps/oauth/issue",
  
  // OAuth параметры
  SCOPE: "birthday contacts openid profile snils blitz_change_password blitz_user_rights blitz_qr_auth",
  RESPONSE_TYPE: "code",
  PROMPT: "login",
  BIP_ACTION_HINT: "used_sms",
  REDIRECT_URI: "dnevnik-mes://oauth2redirect", // или ваш callback URL
  ACCESS_TYPE: "offline",
  CODE_CHALLENGE_METHOD: "S256",
  
  // Приложение
  SOFTWARE_ID: "dnevnik.mos.ru",
  DEVICE_TYPE: "android_phone", // или "ios_phone", "web_browser"
  
  // Grant types
  GRANT_TYPE_CODE: "authorization_code",
  GRANT_TYPE_REFRESH: "refresh_token",
  
  // Software Statement (JWT токен для регистрации)
  SOFTWARE_STATEMENT: "eyJ0eXAiOiJKV1QiLCJibGl0ejpraW5kIjoiU09GVF9TVE0iLCJhbGciOiJSUzI1NiJ9..."
};
```

## Процесс авторизации

### Схема процесса:

```
┌─────────────┐    1. Issue Call     ┌─────────────┐
│   Клиент    │ ──────────────────► │   mos.ru    │
│             │ ◄────────────────── │             │
└─────────────┘   client_id/secret  └─────────────┘
        │
        │ 2. Генерация PKCE
        ▼
┌─────────────┐
│code_verifier│
│code_challenge│
└─────────────┘
        │
        │ 3. Redirect to mos.ru
        ▼
┌─────────────┐    4. User Login     ┌─────────────┐
│   Browser   │ ──────────────────► │   mos.ru    │
│             │ ◄────────────────── │             │
└─────────────┘   authorization_code└─────────────┘
        │
        │ 5. Callback with code
        ▼
┌─────────────┐    6. Token Exchange ┌─────────────┐
│   Клиент    │ ──────────────────► │   mos.ru    │
│             │ ◄────────────────── │             │
└─────────────┘  access_token/refresh└─────────────┘
        │
        │ 7. MOS → MES Token
        ▼
┌─────────────┐    8. Session Create ┌─────────────┐
│   Клиент    │ ──────────────────► │     МЭШ     │
│             │ ◄────────────────── │             │
└─────────────┘    mes_access_token  └─────────────┘
```

## Регистрация приложения

### 1. Issue Call - регистрация приложения

**Эндпоинт:** `POST https://login.mos.ru/sps/oauth/issue`

**Заголовки:**
```http
Authorization: Bearer FqzGn1dTJ9BQCHgV0rmMjtYFIgaFf9TrGVEzgtju-zbtIbeJSkIyDcl0e2QMirTNpEqovTT8NvOLZI0XklVEIw
Content-Type: application/json
```

**Тело запроса:**
```json
{
  "software_id": "dnevnik.mos.ru",
  "device_type": "android_phone",
  "software_statement": "eyJ0eXAiOiJKV1QiLCJibGl0ejpraW5kIjoiU09GVF9TVE0iLCJhbGciOiJSUzI1NiJ9..."
}
```

**Ответ:**
```json
{
  "client_id": "dnevnik.mos.ru_mobile_app_12345",
  "client_secret": "abcdef123456789",
  "expires_at": "2025-12-31T23:59:59Z"
}
```

### Пример кода (Python):

```python
import requests
import json

def register_application():
    url = "https://login.mos.ru/sps/oauth/issue"
    
    headers = {
        "Authorization": "Bearer FqzGn1dTJ9BQCHgV0rmMjtYFIgaFf9TrGVEzgtju-zbtIbeJSkIyDcl0e2QMirTNpEqovTT8NvOLZI0XklVEIw",
        "Content-Type": "application/json"
    }
    
    data = {
        "software_id": "dnevnik.mos.ru",
        "device_type": "android_phone",
        "software_statement": SOFTWARE_STATEMENT
    }
    
    response = requests.post(url, headers=headers, json=data)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"Registration failed: {response.text}")
```

## Генерация PKCE

### Алгоритм генерации:

1. **code_verifier** - случайная строка 43-128 символов из [A-Z, a-z, 0-9, -, ., _, ~]
2. **code_challenge** - Base64UrlSafe(SHA256(code_verifier)) без padding

### Пример кода (Python):

```python
import secrets
import hashlib
import base64

def generate_pkce():
    # Генерируем code_verifier (80 символов)
    code_verifier = ''.join(secrets.choice(
        'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~'
    ) for _ in range(80))
    
    # Создаем code_challenge
    digest = hashlib.sha256(code_verifier.encode('utf-8')).digest()
    code_challenge = base64.urlsafe_b64encode(digest).decode('utf-8').rstrip('=')
    
    return code_verifier, code_challenge

# Пример использования
code_verifier, code_challenge = generate_pkce()
print(f"Code Verifier: {code_verifier}")
print(f"Code Challenge: {code_challenge}")
```

### Пример кода (JavaScript):

```javascript
async function generatePKCE() {
    // Генерируем code_verifier
    const array = new Uint8Array(60);
    crypto.getRandomValues(array);
    const codeVerifier = btoa(String.fromCharCode.apply(null, array))
        .replace(/\+/g, '-')
        .replace(/\//g, '_')
        .replace(/=/g, '');
    
    // Создаем code_challenge
    const encoder = new TextEncoder();
    const data = encoder.encode(codeVerifier);
    const digest = await crypto.subtle.digest('SHA-256', data);
    const codeChallenge = btoa(String.fromCharCode.apply(null, new Uint8Array(digest)))
        .replace(/\+/g, '-')
        .replace(/\//g, '_')
        .replace(/=/g, '');
    
    return { codeVerifier, codeChallenge };
}
```

## Формирование URL авторизации

### Параметры URL:

```
https://login.mos.ru/sps/oauth/ae?
  scope=birthday+contacts+openid+profile+snils+blitz_change_password+blitz_user_rights+blitz_qr_auth
  &access_type=offline
  &response_type=code
  &client_id={client_id}
  &redirect_uri=dnevnik-mes://oauth2redirect
  &prompt=login
  &code_challenge={code_challenge}
  &code_challenge_method=S256
  &bip_action_hint=used_sms
```

### Пример кода (Python):

```python
from urllib.parse import urlencode

def build_auth_url(client_id, code_challenge):
    base_url = "https://login.mos.ru/sps/oauth/ae"
    
    params = {
        "scope": "birthday contacts openid profile snils blitz_change_password blitz_user_rights blitz_qr_auth",
        "access_type": "offline",
        "response_type": "code",
        "client_id": client_id,
        "redirect_uri": "dnevnik-mes://oauth2redirect",
        "prompt": "login",
        "code_challenge": code_challenge,
        "code_challenge_method": "S256",
        "bip_action_hint": "used_sms"
    }
    
    return f"{base_url}?{urlencode(params)}"
```

## Обработка callback

После авторизации пользователя mos.ru перенаправляет на указанный `redirect_uri` с параметрами:

### Успешная авторизация:
```
dnevnik-mes://oauth2redirect?code=AUTH_CODE&state=STATE_VALUE
```

### Ошибка авторизации:
```
dnevnik-mes://oauth2redirect?error=access_denied&error_description=User+denied+access
```

### Пример обработки (Python Flask):

```python
from flask import request, jsonify

@app.route('/oauth/callback')
def oauth_callback():
    code = request.args.get('code')
    state = request.args.get('state')
    error = request.args.get('error')
    
    if error:
        return jsonify({
            'success': False,
            'error': error,
            'error_description': request.args.get('error_description')
        }), 400
    
    if not code or not state:
        return jsonify({
            'success': False,
            'error': 'missing_parameters'
        }), 400
    
    # Обмениваем код на токен
    token_data = exchange_code_for_token(code)
    
    return jsonify({
        'success': True,
        'access_token': token_data['access_token'],
        'refresh_token': token_data['refresh_token']
    })
```

## Обмен кода на токен

### Запрос токена у mos.ru

**Эндпоинт:** `POST https://login.mos.ru/sps/oauth/te`

**Заголовки:**
```http
Authorization: Basic {base64(client_id:client_secret)}
Content-Type: application/x-www-form-urlencoded
```

**Тело запроса:**
```
grant_type=authorization_code
&redirect_uri=dnevnik-mes://oauth2redirect
&code={authorization_code}
&code_verifier={code_verifier}
```

**Ответ:**
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "def50200abc123...",
  "scope": "openid profile"
}
```

### Пример кода (Python):

```python
import base64
import requests

def exchange_code_for_token(code, code_verifier, client_id, client_secret):
    url = "https://login.mos.ru/sps/oauth/te"
    
    # Создаем Authorization header
    credentials = f"{client_id}:{client_secret}"
    auth_header = base64.b64encode(credentials.encode()).decode()
    
    headers = {
        "Authorization": f"Basic {auth_header}",
        "Content-Type": "application/x-www-form-urlencoded"
    }
    
    data = {
        "grant_type": "authorization_code",
        "redirect_uri": "dnevnik-mes://oauth2redirect",
        "code": code,
        "code_verifier": code_verifier
    }
    
    response = requests.post(url, headers=headers, data=data)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"Token exchange failed: {response.text}")
```

## Получение токена МЭШ

После получения токена от mos.ru необходимо обменять его на токен МЭШ.

### Обмен токена mos.ru на токен МЭШ

**Эндпоинт:** `POST https://school.mos.ru/lms/api/sessions`

**Заголовки:**
```http
Content-Type: application/json
```

**Тело запроса:**
```json
{
  "auth_token": "{mos_access_token}"
}
```

**Ответ:**
```json
{
  "user_id": 123456,
  "authentication_token": "mes_access_token_here",
  "profiles": [
    {
      "id": 789012,
      "type": "student",
      "user_id": 123456,
      "school_id": 345678,
      "class_unit_id": 901234
    }
  ]
}
```

### Пример кода (Python):

```python
def get_mes_token(mos_access_token):
    url = "https://school.mos.ru/lms/api/sessions"
    
    headers = {
        "Content-Type": "application/json"
    }
    
    data = {
        "auth_token": mos_access_token
    }
    
    response = requests.post(url, headers=headers, json=data)
    
    if response.status_code == 200:
        result = response.json()
        return {
            'mes_token': result['authentication_token'],
            'user_id': result['user_id'],
            'profiles': result['profiles']
        }
    else:
        raise Exception(f"MES token exchange failed: {response.text}")
```

## Обновление токенов

### Обновление токена mos.ru

**Эндпоинт:** `POST https://login.mos.ru/sps/oauth/te`

**Заголовки:**
```http
Authorization: Basic {base64(client_id:client_secret)}
Content-Type: application/x-www-form-urlencoded
```

**Тело запроса:**
```
grant_type=refresh_token
&refresh_token={refresh_token}
```

### Пример кода (Python):

```python
def refresh_mos_token(refresh_token, client_id, client_secret):
    url = "https://login.mos.ru/sps/oauth/te"
    
    credentials = f"{client_id}:{client_secret}"
    auth_header = base64.b64encode(credentials.encode()).decode()
    
    headers = {
        "Authorization": f"Basic {auth_header}",
        "Content-Type": "application/x-www-form-urlencoded"
    }
    
    data = {
        "grant_type": "refresh_token",
        "refresh_token": refresh_token
    }
    
    response = requests.post(url, headers=headers, data=data)
    
    if response.status_code == 200:
        token_data = response.json()
        # Обмениваем новый mos токен на mes токен
        mes_data = get_mes_token(token_data['access_token'])
        
        return {
            'mos_access_token': token_data['access_token'],
            'mos_refresh_token': token_data['refresh_token'],
            'mes_access_token': mes_data['mes_token'],
            'expires_in': token_data['expires_in']
        }
    else:
        raise Exception(f"Token refresh failed: {response.text}")
```

## Примеры кода

### Полный пример OAuth клиента (Python):

```python
import secrets
import hashlib
import base64
import requests
from urllib.parse import urlencode, parse_qs, urlparse

class MosRuOAuthClient:
    def __init__(self):
        self.client_id = None
        self.client_secret = None
        self.code_verifier = None
        self.code_challenge = None
        
    def register_application(self):
        """Регистрация приложения"""
        url = "https://login.mos.ru/sps/oauth/issue"
        
        headers = {
            "Authorization": "Bearer FqzGn1dTJ9BQCHgV0rmMjtYFIgaFf9TrGVEzgtju-zbtIbeJSkIyDcl0e2QMirTNpEqovTT8NvOLZI0XklVEIw",
            "Content-Type": "application/json"
        }
        
        data = {
            "software_id": "dnevnik.mos.ru",
            "device_type": "android_phone",
            "software_statement": "eyJ0eXAiOiJKV1QiLCJibGl0ejpraW5kIjoiU09GVF9TVE0iLCJhbGciOiJSUzI1NiJ9..."
        }
        
        response = requests.post(url, headers=headers, json=data)
        
        if response.status_code == 200:
            result = response.json()
            self.client_id = result['client_id']
            self.client_secret = result['client_secret']
            return result
        else:
            raise Exception(f"Registration failed: {response.text}")
    
    def generate_pkce(self):
        """Генерация PKCE"""
        # code_verifier
        self.code_verifier = ''.join(secrets.choice(
            'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~'
        ) for _ in range(80))
        
        # code_challenge
        digest = hashlib.sha256(self.code_verifier.encode('utf-8')).digest()
        self.code_challenge = base64.urlsafe_b64encode(digest).decode('utf-8').rstrip('=')
        
        return self.code_verifier, self.code_challenge
    
    def get_auth_url(self):
        """Формирование URL авторизации"""
        if not self.client_id or not self.code_challenge:
            raise Exception("Client not registered or PKCE not generated")
        
        params = {
            "scope": "birthday contacts openid profile snils blitz_change_password blitz_user_rights blitz_qr_auth",
            "access_type": "offline",
            "response_type": "code",
            "client_id": self.client_id,
            "redirect_uri": "dnevnik-mes://oauth2redirect",
            "prompt": "login",
            "code_challenge": self.code_challenge,
            "code_challenge_method": "S256",
            "bip_action_hint": "used_sms"
        }
        
        return f"https://login.mos.ru/sps/oauth/ae?{urlencode(params)}"
    
    def exchange_code_for_token(self, code):
        """Обмен кода на токен"""
        url = "https://login.mos.ru/sps/oauth/te"
        
        credentials = f"{self.client_id}:{self.client_secret}"
        auth_header = base64.b64encode(credentials.encode()).decode()
        
        headers = {
            "Authorization": f"Basic {auth_header}",
            "Content-Type": "application/x-www-form-urlencoded"
        }
        
        data = {
            "grant_type": "authorization_code",
            "redirect_uri": "dnevnik-mes://oauth2redirect",
            "code": code,
            "code_verifier": self.code_verifier
        }
        
        response = requests.post(url, headers=headers, data=data)
        
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f"Token exchange failed: {response.text}")
    
    def get_mes_token(self, mos_access_token):
        """Получение токена МЭШ"""
        url = "https://school.mos.ru/lms/api/sessions"
        
        headers = {
            "Content-Type": "application/json"
        }
        
        data = {
            "auth_token": mos_access_token
        }
        
        response = requests.post(url, headers=headers, json=data)
        
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f"MES token exchange failed: {response.text}")

# Пример использования
oauth_client = MosRuOAuthClient()

# 1. Регистрация
oauth_client.register_application()

# 2. Генерация PKCE
oauth_client.generate_pkce()

# 3. Получение URL авторизации
auth_url = oauth_client.get_auth_url()
print(f"Перейдите по ссылке: {auth_url}")

# 4. После получения кода из callback
# code = "полученный_код_авторизации"
# mos_tokens = oauth_client.exchange_code_for_token(code)
# mes_data = oauth_client.get_mes_token(mos_tokens['access_token'])
```

## Безопасность

### Рекомендации по безопасности:

1. **Используйте PKCE** - обязательно для всех типов приложений
2. **Храните секреты безопасно** - client_secret не должен быть в коде
3. **Проверяйте state параметр** - защита от CSRF атак
4. **Используйте HTTPS** - для всех запросов
5. **Ограничьте время жизни токенов** - регулярно обновляйте
6. **Валидируйте redirect_uri** - точное совпадение с зарегистрированным

### Хранение токенов:

```python
# Плохо - в коде
CLIENT_SECRET = "abc123"

# Хорошо - в переменных окружения
import os
CLIENT_SECRET = os.getenv('MOS_CLIENT_SECRET')

# Хорошо - в защищенном хранилище
from keyring import get_password
CLIENT_SECRET = get_password('myapp', 'mos_client_secret')
```

## Ошибки и их обработка

### Типичные ошибки OAuth:

#### 1. Ошибки регистрации приложения:
```json
{
  "error": "invalid_software_statement",
  "error_description": "Software statement is invalid or expired"
}
```

#### 2. Ошибки авторизации:
```
?error=access_denied&error_description=User+denied+access
?error=invalid_request&error_description=Missing+required+parameter
```

#### 3. Ошибки обмена токенов:
```json
{
  "error": "invalid_grant",
  "error_description": "Authorization code is invalid or expired"
}
```

#### 4. Ошибки PKCE:
```json
{
  "error": "invalid_request",
  "error_description": "Code verifier does not match code challenge"
}
```

### Обработка ошибок:

```python
def handle_oauth_error(response):
    """Обработка ошибок OAuth"""
    if response.status_code == 400:
        error_data = response.json()
        error_code = error_data.get('error')
        error_description = error_data.get('error_description')
        
        if error_code == 'invalid_grant':
            # Код авторизации истек или недействителен
            return "Необходимо повторить авторизацию"
        elif error_code == 'invalid_client':
            # Неверные client_id или client_secret
            return "Ошибка конфигурации приложения"
        elif error_code == 'invalid_request':
            # Неверные параметры запроса
            return f"Неверный запрос: {error_description}"
        else:
            return f"Ошибка OAuth: {error_description}"
    
    elif response.status_code == 401:
        return "Неверные учетные данные"
    elif response.status_code == 403:
        return "Доступ запрещен"
    elif response.status_code == 429:
        return "Превышен лимит запросов"
    else:
        return f"Неизвестная ошибка: {response.status_code}"
```

### Логирование и мониторинг:

```python
import logging

logger = logging.getLogger(__name__)

def safe_oauth_request(func, *args, **kwargs):
    """Безопасное выполнение OAuth запроса с логированием"""
    try:
        result = func(*args, **kwargs)
        logger.info(f"OAuth request successful: {func.__name__}")
        return result
    except Exception as e:
        logger.error(f"OAuth request failed: {func.__name__}, error: {str(e)}")
        raise

# Использование
try:
    tokens = safe_oauth_request(oauth_client.exchange_code_for_token, code)
except Exception as e:
    print(f"Ошибка получения токенов: {e}")
```

## Заключение

OAuth 2.0 авторизация через mos.ru с использованием PKCE обеспечивает безопасный доступ к API МЭШ. Важно следовать рекомендациям по безопасности и правильно обрабатывать ошибки для создания надежного приложения.

### Основные моменты:
- Всегда используйте PKCE для дополнительной безопасности
- Храните секреты приложения в безопасном месте
- Обрабатывайте все возможные ошибки OAuth
- Регулярно обновляйте токены доступа
- Логируйте важные события для мониторинга
