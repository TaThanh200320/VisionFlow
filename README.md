# VisionFlow
[中](https://github.com/dan246/VisionFlow/blob/main/README.md)
[EN](https://github.com/dan246/VisionFlow/blob/main/README_en.md)

VisionFlow 是一個用於影像辨識與通知系統的後端應用程式。該專案使用 Flask 框架，並透過 PostgreSQL 作為資料庫進行資料管理。應用程式中整合了 Redis 來管理攝影機影像的處理與分配。所有環境均可使用 Docker 進行配置和管理。

## 目錄

- [專案介紹](#專案介紹)
- [快速開始](#快速開始)
  - [先決條件](#先決條件)
  - [設置步驟](#設置步驟)
- [Redis 功能](#redis-功能)
- [API 說明](#api-說明)
  - [使用者認證 API](#使用者認證-api)
  - [攝影機管理 API](#攝影機管理-api)
  - [通知管理 API](#通知管理-api)
  - [LINE Token 管理 API](#line-token-管理-api)
  - [郵件接收者管理 API](#郵件接收者管理-api)
  - [影像處理與流媒體 API](#影像處理與流媒體-api)
- [使用範例](#使用範例)
- [注意事項](#注意事項)

## 專案介紹

VisionFlow 是一個後端應用程式，旨在處理影像辨識與通知系統的相關操作。該應用程式提供了使用者認證、攝影機管理、通知發送以及與 LINE 和郵件通知相關的功能。

## 快速開始

### 先決條件

在開始之前，請確保你已經安裝了以下工具：

- [Docker](https://www.docker.com/)
- [Docker Compose](https://docs.docker.com/compose/)

### 設置步驟

1. 先克隆此專案到你的本地環境：

    ```bash
    git clone https://github.com/yourusername/VisionFlow.git
    cd VisionFlow
    ```

2. 使用 Docker Compose 啟動服務：

    ```bash
    docker-compose -f docker-compose.yaml -f docker-compose-redis.yaml up -d
    ```

    這會自動下載所需的 Docker image，安裝相關套件，並在 `http://localhost:5000` 上啟動 Flask 應用程式。

4. 若需要遷移資料庫（在第一次運行或模型更新時執行）：
    1. 進入容器
        ```bash
        docker-compose exec -it backend 
        ```
    2. 進入後依照 update_db.txt 操作即可

5. 若需要配置 Redis 與多個 worker 節點，請使用 `docker-compose-redis.yaml` 進行設置：

    ```bash
    docker-compose -f docker-compose-redis.yaml up -D
    ```

    這會啟動 Redis 服務與多個 worker 節點來處理影像辨識與攝影機分配工作。

6. objectrecognition 須將模型替換為自己的模型URL，且檔案須為 best.pt(可設置多個模型網址，不用擔心 pt 模型被蓋掉)
## Redis 功能

### 影像處理

VisionFlow 使用 Redis 來管理攝影機的影像資料。攝影機的影像會被儲存到 Redis 中，並分配到不同的 worker 虛擬機進行處理。每個影像在辨識後會存儲為一個獨立的 Redis key，以供後續使用。

1. **影像儲存與取得**:
   - 每個攝影機的最新影像會存儲在 Redis 中，使用 `camera_{camera_id}_latest_frame` 作為 key。
   - 透過 `camera_{camera_id}_boxed_image` 取得經過影像辨識後的結果。

2. **影像辨識流程**:
   - 當攝影機捕捉到影像後，該影像會以 `camera_{camera_id}_latest_frame` 的 key 存入 Redis。
   - worker 會從 Redis 中提取該影像進行辨識處理，並將處理後的影像結果存入 `camera_{camera_id}_boxed_image`。

### 攝影機分配

為了有效管理多台攝影機的影像處理，VisionFlow 配置了多個 worker 節點，這些節點能夠分散處理工作負載，提升系統效率。每個 worker 都從 Redis 中提取攝影機影像進行處理，確保系統的穩定性與擴展性。

## API 說明

### 使用者認證 API

- **註冊新使用者**

    ```http
    POST /register
    ```

    **Request Body:**

    ```json
    {
        "username": "your_username",
        "email": "your_email",
        "password": "your_password"
    }
    ```

    **Response:**

    ```json
    {
        "message": "User registered successfully",
        "account_uuid": "account-uuid-string"
    }
    ```

- **使用者登入**

    ```http
    POST /login
    ```

    **Request Body:**

    ```json
    {
        "username": "your_username",
        "password": "your_password"
    }
    ```

    **Response:**

    ```json
    {
        "message": "Login successful",
        "account_uuid": "account-uuid-string"
    }
    ```

### 攝影機管理 API

- **獲取所有攝影機列表**

    ```http
    GET /cameras
    ```

    **Response:**

    ```json
    [
        {
            "id": 1,
            "name": "Camera 1",
            "stream_url": "http://camera-stream-url",
            "location": "Entrance"
        },
        {
            "id": 2,
            "name": "Camera 2",
            "stream_url": "http://camera-stream-url",
            "location": "Lobby"
        }
    ]
    ```

- **添加新攝影機**

    ```http
    POST /cameras
    ```

    **Request Body:**

    ```json
    {
        "name": "Camera 1",
        "stream_url": "http://camera-stream-url",
        "location": "Entrance"
    }
    ```

    **Response:**

    ```json
    {
        "message": "Camera added successfully"
    }
    ```

### 通知管理 API

- **獲取所有通知**

    ```http
    GET /notifications
    ```

    **Response:**

    ```json
    [
        {
            "id": 1,
            "account_uuid": "account-uuid-string",
            "camera_id": 1,
            "message": "Detected event",
            "image_path": "/path/to/image.jpg",
            "created_at": "2023-01-01T12:00:00"
        }
    ]
    ```

- **創建新通知**

    ```http
    POST /notifications
    ```

    **Request Body:**

    ```json
    {
        "account_uuid": "account-uuid-string",
        "camera_id": 1,
        "message": "Detected event",
        "image_path": "/path/to/image.jpg"
    }
    ```

    **Response:**

    ```json
    {
        "message": "Notification created successfully"
    }
    ```

### LINE Token 管理 API

- **獲取所有 LINE Token**

    ```http
    GET /line_tokens
    ```

    **Response:**

    ```json
    [
        {
            "id": 1,
            "account_uuid": "account-uuid-string",
            "token": "line-token-string"
        }
    ]
    ```

- **添加新 LINE Token**

    ```http
    POST /line_tokens
    ```

    **Request Body:**

    ```json
    {
        "account_uuid": "account-uuid-string",
        "token": "line-token-string"
    }
    ```

    **Response:**

    ```json
    {
        "message": "Line token added successfully"
    }
    ```

### 郵件接收者管理 API

- **獲取所有郵件接收者**

    ```http
    GET /email_recipients
    ```

    **Response:**

    ```json
    [
        {
            "id": 1,
            "account_uuid": "account-uuid-string",
            "email": "recipient@example.com"
        }
    ]
    ```

- **添加新郵件接收者**

    ```http
    POST /email_recipients
    ```

    **Request Body:**

    ```json
    {
        "account_uuid": "account-uuid-string",
        "email": "recipient@example.com"
    }
    ```

    **Response:**

    ```json
    {
        "message": "Email recipient added successfully"
    }
    ```

### 影像處理與流媒體 API

- **獲取攝影機狀態**

    ```http
    GET /camera_status
    ```

    **Response:**

    ```json
    {
        "camera_1": "active",
        "camera_2": "inactive",
        ...
    }
    ```

- **獲取攝影機最新快照**

    ```http
    GET /get_snapshot/<camera_id>
    ```

    **Response:**

    直接返回 JPEG 影像資料流。

- **取得指定路徑的影像**

    ```http
    GET /image/<path:image_path>
    ```

    **Response:**

    返回指定路徑的影像文件。

- **獲取即時影像流**

    ```http
    GET /get_stream/<int:ID>
    ```

    **Response:**

    返回攝影機的實時影像流。

- **獲取辨識後的即時影像流**

    ```http
    GET /recognized_stream/<ID>
    ```

    **Response:**

    返回經過辨識處理後的實時影像流。

- **顯示攝影機快照與矩形區域**

    ```http
    GET /snapshot_ui/<ID>
    ```

    **Response:**

    以 HTML 模板的方式顯示攝影機的快照以及畫出的矩形區域(關注區域)，設定後模型將只關注框框內的區域。

- **管理矩形區域**

    ```http
    POST /rectangles/<ID>
    GET /rectangles/<ID>
    DELETE /rectangles/<ID>
    ```

    **Request Body (POST):**

    ```json
    [
        {
            "x": 100,
            "y": 200,
            "width": 50,
            "height": 60
        },
        ...
    ]
    ```

    **Response:**

    - POST: `矩形已儲存`
    - GET: 返回攝影機的所有矩形區域。
    - DELETE: `所有矩形已清除`

## 使用範例

以下是使用 VisionFlow API 的一些範例：

1. **註冊新使用者並登入**

    ```bash
    curl -X POST http://localhost:5000/register -H "Content-Type: application/json" -d '{"username": "user1", "email": "user1@example.com", "password": "password123"}'
    curl -X POST http://localhost:5000/login -H "Content-Type: application/json" -d '{"username": "user1", "password": "password123"}'
    ```

2. **添加新攝影機**

    ```bash
    curl -X POST http://localhost:5000/cameras -H "Content-Type: application/json" -d '{"name": "Camera 1", "stream_url": "http://camera-stream-url", "location": "Entrance"}'
    ```

3. **創建新通知**

    ```bash
    curl -X POST http://localhost:5000/notifications -H "Content-Type: application/json" -d '{"account_uuid": "your-account-uuid", "camera_id": 1, "message": "Detected event", "image_path": "/path/to/image.jpg"}'
    ```

4. **獲取攝影機狀態**

    ```bash
    curl -X GET http://localhost:15440/camera_status
    ```

5. **獲取攝影機即時影像流**

    ```bash
    curl -X GET http://localhost:15440/get_stream/1
    ```

6. **獲取辨識後的即時影像流**

    ```bash
    curl -X GET http://localhost:15440/recognized_stream/1
    ```

7. **管理矩形區域**

    - 儲存矩形區域：

        ```bash
        curl -X POST http://localhost:15440/rectangles/1 -H "Content-Type: application/json" -d '[{"x": 100, "y": 200, "width": 50, "height": 60}]'
        ```

    - 獲取所有矩形區域：

        ```bash
        curl -X GET http://localhost:15440/rectangles/1
        ```

    - 清除所有矩形區域：

        ```bash
        curl -X DELETE http://localhost:15440/rectangles/1
        ```

## 注意事項

1. **環境變數**: 如果有需要，請確保在 `.env` 文件中正確設置了 `DATABASE_URL`、`SECRET_KEY` 和 `REDIS_URL`，這裡直接將預設變數寫在 code 裡，所以也能跳過這步直接執行。

2. **資料庫遷移**: 如需更新資料庫或新增資料表，修改完 \web\models\ 後，請執行 `flask db migrate` 和 `flask db upgrade` 來更新資料庫。

3. **Redis 配置**: 使用 Redis 來管理影像資料的儲存與處理，確保其正常運行並與 worker 節點連接。
4. **Docker 啟動**: 請使用 Docker Compose 來管理應用程式的啟動和停止，尤其是當需要啟動多個 worker 節點時。
5. **資料備份**: 定期備份你的 PostgreSQL 資料庫與 Redis 資料以防止數據丟失。
6. **模型路徑**: 模型請替換成自己的模型(位於\object_recognition\app.py)。

## License

本專案使用 [MIT License](LICENSE) 授權。


## Contributions & Contact

If you have any questions or contributions, please feel free to contact me. Your input is very welcome and will help improve this project. You can open an issue or submit a pull request on GitHub. Alternatively, you can reach me directly through the contact details provided below.

### 聯繫與貢獻

如果您對此項目有任何疑問或想要做出貢獻，歡迎與我聯繫。您的反饋對於改進項目非常寶貴。您可以在 GitHub 上開啟問題(issue)或提交拉取請求(pull request)。或者，您也可以通過下方提供的聯繫方式直接與我聯繫。

sky328423@gmail.com
