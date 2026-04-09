---
layout: post
title: "웹 애플리케이션에서 업로드 이미지를 어디에 저장해야 할까?"
date: 2026-04-09
categories: [Web, Backend]
tags: [upload, static, fastapi, flask, nginx, storage, python]
---

사용자가 업로드한 이미지를 웹 페이지에 표시하려면,  
**"어디에 저장하느냐"** 가 핵심입니다.

---

## 핵심 원칙

브라우저가 이미지를 표시하려면 **URL로 접근 가능**해야 합니다.  
즉, 저장 위치가 **HTTP로 서빙되는 경로** 안에 있어야 합니다.

```
업로드 → 서버 특정 경로에 저장 → 그 경로를 URL로 노출 → <img src="URL">
```

---

## 1. static 폴더 하위에 저장 (가장 기본)

Flask, FastAPI 같은 프레임워크는 `static/` 폴더를 정적 파일 서빙 경로로 사용합니다.  
업로드 파일을 `static/uploads/` 에 저장하면 바로 URL로 접근 가능합니다.

```
프로젝트/
  static/
    uploads/        ← 업로드 이미지 저장
      cat.jpg
  templates/
  main.py
```

### FastAPI 예시

```python
from fastapi import FastAPI, UploadFile
from fastapi.staticfiles import StaticFiles
import shutil, uuid
from pathlib import Path

app = FastAPI()

UPLOAD_DIR = Path("static/uploads")
UPLOAD_DIR.mkdir(parents=True, exist_ok=True)

# /static 경로로 정적 파일 서빙
app.mount("/static", StaticFiles(directory="static"), name="static")

@app.post("/upload")
async def upload_image(file: UploadFile):
    ext = Path(file.filename).suffix
    filename = f"{uuid.uuid4()}{ext}"          # 이름 충돌 방지
    save_path = UPLOAD_DIR / filename

    with save_path.open("wb") as f:
        shutil.copyfileobj(file.file, f)

    url = f"/static/uploads/{filename}"
    return {"url": url}
```

### Flask 예시

```python
from flask import Flask, request, url_for
from werkzeug.utils import secure_filename
import os, uuid

app = Flask(__name__)
UPLOAD_FOLDER = os.path.join(app.static_folder, "uploads")
os.makedirs(UPLOAD_FOLDER, exist_ok=True)

@app.post("/upload")
def upload():
    file = request.files["image"]
    filename = f"{uuid.uuid4()}_{secure_filename(file.filename)}"
    file.save(os.path.join(UPLOAD_FOLDER, filename))

    url = url_for("static", filename=f"uploads/{filename}")
    return {"url": url}
```

업로드 후 반환된 URL을 템플릿에서 바로 사용할 수 있습니다:

```html
<img src="{{ url }}">
```

---

## 2. static 외부 별도 디렉토리에 저장 + Nginx 서빙

운영 환경에서는 업로드 파일을 **코드와 분리된 디렉토리**에 저장하는 것이 일반적입니다.  
그리고 **Nginx가 직접 파일을 서빙**하게 하면 Python 서버를 거치지 않아 성능이 좋습니다.

```
/var/www/uploads/      ← 업로드 파일 저장 (코드 바깥)
/home/user/myapp/      ← 애플리케이션 코드
```

### Nginx 설정

```nginx
server {
    listen 80;
    server_name example.com;

    # 업로드 파일은 Nginx가 직접 서빙
    location /uploads/ {
        alias /var/www/uploads/;
    }

    # 나머지 요청은 FastAPI/Flask로 프록시
    location / {
        proxy_pass http://127.0.0.1:8000;
    }
}
```

FastAPI 코드에서는 저장 경로만 변경:

```python
UPLOAD_DIR = Path("/var/www/uploads")

# URL 반환 시
url = f"/uploads/{filename}"
```

---

## 3. 클라우드 스토리지 (S3, GCS 등)

서버가 여러 대거나 파일을 영구적으로 안전하게 보관해야 할 때 사용합니다.

```python
import boto3, uuid
from pathlib import Path

s3 = boto3.client("s3")
BUCKET = "my-bucket"

@app.post("/upload")
async def upload(file: UploadFile):
    filename = f"uploads/{uuid.uuid4()}{Path(file.filename).suffix}"
    s3.upload_fileobj(file.file, BUCKET, filename)

    url = f"https://{BUCKET}.s3.amazonaws.com/{filename}"
    return {"url": url}
```

---

## 저장 방식 비교

| 방식 | 장점 | 단점 | 적합한 상황 |
|------|------|------|------------|
| `static/uploads/` | 설정 간단, 바로 서빙 | 코드와 파일 혼재 | 개발/소규모 |
| 외부 디렉토리 + Nginx | 성능 좋음, 코드 분리 | Nginx 설정 필요 | 운영 환경 |
| 클라우드 스토리지 | 확장성, 내구성 | 비용, SDK 설정 | 대규모 서비스 |

---

## 파일명 처리 주의사항

사용자가 올린 파일명을 그대로 쓰면 **보안 문제**가 생깁니다.

```python
# 나쁜 예 — 경로 조작 공격 가능
save_path = UPLOAD_DIR / file.filename          # ../../etc/passwd 등 위험

# 좋은 예 1 — UUID로 새 이름 생성
filename = f"{uuid.uuid4()}{Path(file.filename).suffix}"

# 좋은 예 2 — secure_filename으로 정리 (Flask/Werkzeug)
from werkzeug.utils import secure_filename
filename = secure_filename(file.filename)
```

---

## 정리

1. **개발 단계**: `static/uploads/` 에 저장하고 프레임워크가 서빙
2. **운영 단계**: 코드 외부 디렉토리에 저장, Nginx가 직접 서빙
3. **대규모**: S3/GCS 등 클라우드 스토리지 활용
4. **공통 원칙**: 파일명은 반드시 UUID 또는 `secure_filename`으로 처리
