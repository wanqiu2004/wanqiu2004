## 一、API 设计（路径区分）

### 图片 (p)

* **获取图片列表**
  `GET /api/v1/p`
  返回：

  ```json
  [
    {"url":"","txt":""},{"url":"","txt":""}
  ]
  ```

* **获取单个图片详情**
  `GET /api/v1/p/{filename_hash}`
  返回：

  ```json
  {img:bin，txt:""}
  ```

* **获取雪碧图**
  `GET /api/v1/p/{filename_hash}/sprite`
  {img:bin, txt:""}
  → 返回图片二进制

---

### 视频 (v)

* **获取视频列表**
  `GET /api/v1/v`

  ```json
  [
    {"url":"","txt":""},{"url":"","txt":""}...
  ]
  ```

* **获取单个视频详情**
  `GET /api/v1/v/{filename_hash}`

  ```json
  {"url":"m3u8","spriteSheet":8}
  ```

* **获取视频雪碧图**
  `GET /api/v1/v/{filename_hash}/sprite/{number}`
img
---

### 用户注册/登录

* **注册**
  `POST /api/v1/s`

  ```json
  {"username":"alice","password":"xxx"}
  ```

* **登录**
  `POST /api/v1/l`

  ```json
  {"username":"alice","password":"xxx"}
  ```

  返回：

  ```http
  Set-Cookie: session=xxxx; HttpOnly; Secure; SameSite=Lax
  ```

  ```json
  {"status":"ok"}
  ```

---

### 上传 (断点续传)

* **上传分片**
  `POST /api/v1/u`

  ```json
  {
    "filename":"movie.mp4",
    "chunk_index":2,
    "chunk_total":10,
    "data":"<binary>"
  }
  ```

* **查询上传状态**
  `GET /api/v1/u/filename/status`

  ```json
  {"uploaded_chunks":[0,1,2],"chunk_total":10}
  ```

---

## 二、PostgreSQL 数据库设计

### 1. 用户表 `users`

```sql
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    email VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### 2. 文件表 `files`

```sql
CREATE TABLE files (
    file_id SERIAL PRIMARY KEY,
    filename VARCHAR(255) NOT NULL,
    filename_hash CHAR(64) UNIQUE NOT NULL,
    file_type VARCHAR(10) NOT NULL CHECK (file_type IN ('image','video')),
    user_id INT REFERENCES users(user_id),
    size BIGINT NOT NULL,
    cover_placeholder VARCHAR(100) NOT NULL DEFAULT 'DEFAULT_IMAGE',  -- 封面占位符字符串
    sprite_count INT DEFAULT 0,
    status VARCHAR(20) DEFAULT 'uploaded',
    created_at TIMESTAMP DEFAULT NOW()
);
```

### 3. 上传进度表 `upload_chunks`

```sql
CREATE TABLE upload_chunks (
    id SERIAL PRIMARY KEY,
    filename_hash CHAR(64) NOT NULL REFERENCES files(filename_hash) ON DELETE CASCADE,
    chunk_index INT NOT NULL,
    uploaded BOOLEAN DEFAULT FALSE,
    UNIQUE(filename_hash, chunk_index)
);
```

### 4. 会话表 `sessions`

```sql
CREATE TABLE sessions (
    session_id CHAR(64) PRIMARY KEY,
    user_id INT REFERENCES users(user_id),
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP
);
```

---

## 三、Linux 目录结构

```
/srv/myapp/
│── backend/               # 后端服务
│── frontend/              # Vue3 前端
│── data/
│   ├── images/{hash}/original.jpg
│   ├── videos/{hash}/original.mp4
│   ├── uploads_tmp/{hash}/part_*
│── logs/
│── config/
```
