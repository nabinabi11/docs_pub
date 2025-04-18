# API Документация для Нейро-повара

---

## 1. Общая информация

- **Базовый URL:**  
  ```
  http://neuro.demiand.ru:8001
  ```

- **Аутентификация:**  
  Каждая операция требует наличия специального HTTP-заголовка:
  ```
  X-API-Key: demiand_api_app_key
  ```
  Если заголовок не передан или ключ неверный, сервер возвращает статус **403 Forbidden**.

---

## 2. Эндпоинты

### 2.1 GET /messages

**Описание:**  
Возвращает список сообщений для заданного чата.

**Метод:** `GET`

**Параметры запроса (Query):**

- **limit** (integer, по умолчанию 10): Количество сообщений для возврата.
- **offset** (integer, по умолчанию 0): Смещение (начальная позиция).
- **chat_id** (string, по умолчанию "default_chat"): Идентификатор чата.

**Пример запроса (cURL):**
```bash
curl -H "X-API-Key: demiand_api_app_key" "http://neuro.demiand.ru:8001/messages?limit=10&offset=0&chat_id=test_chat"
```

**Пример ответа (JSON):**
```json
[
  {
    "content_type": "text",
    "content": "Пример текста от пользователя",
    "role": "user",
    "id": "server_generated_id",
    "client_id": "client_generated_id",
    "created_at": 1744108979,
    "extra_info": null
  },
  {
    "content_type": "voice",
    "content": {
      "voice": "http://server-or-cdn/voice_message.wav"
    },
    "role": "user",
    "id": "server_generated_id",
    "client_id": "client_generated_id",
    "created_at": 1744110979,
    "extra_info": null
  },
  {
    "content_type": "images",
    "content": {
      "images": [
        "http://server-or-cdn/image_1.jpg",
        "http://server-or-cdn/image_2.jpg"
      ],
      "text": "Описание или подпись"
    },
    "role": "user",
    "id": "server_generated_id",
    "client_id": "client_generated_id",
    "created_at": 1744111979,
    "extra_info": null
  },
  {
    "content_type": "text",
    "content": "Пример ответа ассистента",
    "role": "assistant",
    "id": "server_generated_id",
    "client_id": null,
    "created_at": 1744109979,
    "extra_info": null
  }
]

```

---

### 2.2 POST /message/send

**Описание:**  
Отправка сообщения от клиента. Поддерживаются три типа сообщений:

- **text** — текстовое сообщение.
- **voice** — голосовое сообщение (файл, который преобразуется в текст через STT-пайплайн).
- **images** — сообщение с изображением (из списка файлов используется только первое).

**Метод:** `POST`  
**Content-Type:** `multipart/form-data`

#### Обязательные поля формы:

| Поле          | Тип     | Описание                                                                                                                                                              |
|---------------|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| content_type  | string  | Тип сообщения. Допустимые значения: `"text"`, `"images"`, `"voice"`.                                                                                                  |
| content       | string  | Текст сообщения или подпись/описание (для изображений). Для текстовых сообщений – текст запроса, для изображений – описание (может быть пустым).                   |
| client_id     | string  | Идентификатор, генерируемый клиентом (например, `"client_test_123"`).                                                                                                 |
| chat_id       | string  | Идентификатор чата (например, `"test_chat"`).                                                                                                                         |
| created_at    | string  | Временная метка или идентификатор чата. Это поле обязательно, но сервер может игнорировать его и генерировать собственное время. (Например, `"test_chat"`)         |

**Дополнительно:**

- Если `content_type` равен `"voice"`, необходимо передать файл в поле `voice`.
- Если `content_type` равен `"images"`, необходимо передать файл(ы) в поле `images` (если передано несколько, используется только первое).

---

#### Примеры запросов

**Пример 1: Текстовое сообщение**
```bash
curl -X POST "http://neuro.demiand.ru:8001/message/send" \
  -H "X-API-Key: demiand_api_app_key" \
  -F "content_type=text" \
  -F "content=забор стена вторник" \
  -F "client_id=client_test_123" \
  -F "chat_id=test_chat" \
  -F "created_at=test_chat"
```

**Пример 2: Голосовое сообщение**
```bash
curl -X POST "http://neuro.demiand.ru:8001/message/send" \
  -H "X-API-Key: demiand_api_app_key" \
  -F "content_type=voice" \
  -F "client_id=client_test_456" \
  -F "chat_id=test_chat" \
  -F "created_at=test_chat" \
  -F "voice=@/path/to/voice_message.ogg"
```

**Пример 3: Отправка изображения (без подписи)**
```bash
curl -X POST "http://neuro.demiand.ru:8001/message/send" \
  -H "X-API-Key: demiand_api_app_key" \
  -F "content_type=images" \
  -F "content=" \
  -F "client_id=client_test_789" \
  -F "chat_id=test_chat" \
  -F "created_at=test_chat" \
  -F "images=@/path/to/image.jpg;type=image/jpeg"
```

---

## 3. Ответ API

Все запросы к `POST /message/send` возвращают **потоковый ответ** в формате Server-Sent Events (SSE). Каждая строка ответа будет начинаться с префикса:
```
data: <часть ответа>
```
Все полученные части составляют итоговый ответ от ассистента.

---

## 4. Дополнительно

- **Избыточные поля:** Если клиент отправляет дополнительные поля, сервер их игнорирует.
- **Хранение сообщений:** После обработки запроса сервер сохраняет пару сообщений (от пользователя и ответ ассистента) под ключом, равным значению `chat_id`(*cliend_id*).
- **Обработка голоса:** Голосовые файлы обрабатываются с помощью STT-пайплайна, после чего распознанный текст используется для генерации ответа.
- **Обработка изображений:** Если передано несколько изображений, используется только первое. Параметр `content` используется как подпись изображения.

---
