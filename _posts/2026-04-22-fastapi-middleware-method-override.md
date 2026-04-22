---
layout: post
title: "FastAPI 미들웨어 & Method Override 완전 정리"
date: 2026-04-22
categories: [Web, FastAPI]
tags: [fastapi, middleware, python, http, method-override]
---

## 1. 미들웨어란?

모든 요청/응답이 지나가는 **"검문소"**.

```
클라이언트 → [미들웨어] → 라우터(실제 코드) → [미들웨어] → 클라이언트
```

### 식당으로 비유하면

```
손님 → 안내데스크 → 주방 → 안내데스크 → 손님
```

**안내데스크 = 미들웨어**

손님이 주방에 직접 가는 게 아니라, 항상 안내데스크를 거친다.
안내데스크에서 하는 일:
- 예약 확인 (인증)
- 방문 기록 (로깅)
- 대기 시간 측정

이걸 **주방(API)마다 따로 하는 게 아니라** 데스크 한 곳에서 처리한다.

---

## 2. 미들웨어를 왜 쓰나?

### 안 쓰면 이렇게 됨

```python
@app.get("/users")
def get_users():
    # 매번 인증 체크
    token = request.headers.get("Authorization")
    if token != "Bearer 내토큰":
        return {"msg": "인증 실패"}

    # 매번 로그
    print(f"[요청] GET /users")
    start = time.time()

    result = {"users": ["철수", "영희"]}

    # 매번 시간 측정
    print(f"{time.time() - start:.3f}초")
    return result


@app.get("/items")
def get_items():
    # 또 인증 체크 (복붙)
    token = request.headers.get("Authorization")
    if token != "Bearer 내토큰":
        return {"msg": "인증 실패"}

    # 또 로그 (복붙)
    print(f"[요청] GET /items")
    start = time.time()

    result = {"items": ["사과", "바나나"]}

    # 또 시간 측정 (복붙)
    print(f"{time.time() - start:.3f}초")
    return result

# API가 100개면... 100번 복붙 💀
```

### 문제점 3가지

1. **중복 코드 폭발** - API 100개면 같은 코드 100번 씀
2. **수정할 때 지옥** - 로그 형식 하나 바꾸려면 100개 파일 다 수정
3. **빠뜨리면 보안 구멍** - 새 API 만들다가 인증 코드 깜빡하면 누구나 접근 가능

---

## 3. 미들웨어 코드 예시

### 기본 구조

```python
@app.middleware("http")
async def 미들웨어(request: Request, call_next):
    # 요청 들어올 때 실행
    
    response = await call_next(request)  # 실제 API 실행
    
    # 응답 나갈 때 실행
    return response
```

### 실행 시간 측정

```python
import time
from fastapi import FastAPI, Request

app = FastAPI()

@app.middleware("http")
async def 실행시간_측정(request: Request, call_next):
    start = time.time()

    response = await call_next(request)

    elapsed = time.time() - start
    print(f"{request.url} → {elapsed:.3f}초 걸림")

    return response
```

### 로그 남기기

```python
@app.middleware("http")
async def 로그(request: Request, call_next):
    print(f"[요청] {request.method} {request.url}")

    response = await call_next(request)

    print(f"[응답] 상태코드: {response.status_code}")
    return response
```

### 토큰 인증

```python
@app.middleware("http")
async def 인증(request: Request, call_next):
    token = request.headers.get("Authorization")

    if token != "Bearer 내토큰":
        return JSONResponse(status_code=401, content={"msg": "인증 실패"})

    response = await call_next(request)
    return response
```

### 전부 합치면

```python
import time
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

@app.middleware("http")
async def 미들웨어(request: Request, call_next):
    # ① 요청 들어올 때
    token = request.headers.get("Authorization")
    if token != "Bearer 내토큰":
        return JSONResponse(status_code=401, content={"msg": "인증 실패"})

    print(f"[요청] {request.method} {request.url}")
    start = time.time()

    # ② 실제 API 실행
    response = await call_next(request)

    # ③ 응답 나갈 때
    elapsed = time.time() - start
    print(f"[응답] {response.status_code} | {elapsed:.3f}초")

    return response


@app.get("/users")
def get_users():
    return {"users": ["철수", "영희"]}

@app.get("/items")
def get_items():
    return {"items": ["사과", "바나나"]}
```

---

## 4. DummyMiddleware

이름 그대로 **아무것도 안 하는 미들웨어**.

```python
from starlette.middleware.base import BaseHTTPMiddleware

class DummyMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)  # 그냥 통과
        return response
```

### 언제 쓰나?

- **자리 잡기용** - 나중에 로직 채울 예정일 때
- **테스트할 때** - 진짜 미들웨어 대신 통과만 시키고 싶을 때
- **구조 학습용** - 미들웨어 뼈대만 보여줄 때

---

## 5. HTTP Method Override

### 문제 상황

HTML `<form>`은 **GET이랑 POST만 지원**한다.

```html
<form method="GET">    ← 됨 ✅
<form method="POST">   ← 됨 ✅
<form method="DELETE"> ← 안 됨 ❌
<form method="PUT">    ← 안 됨 ❌
```

### 해결책: 속임수

POST로 보내되, `_method` 값을 숨겨서 보낸다.

#### form body에 숨기는 방식

```html
<form method="POST" action="/users/1">
    <input type="hidden" name="_method" value="DELETE">
    <button>삭제</button>
</form>
```

#### query string에 붙이는 방식

```html
<form method="POST" action="/users/1?_method=DELETE">
    <button>삭제</button>
</form>
```

### 미들웨어가 바꿔줌

```
브라우저가 보낸 것          미들웨어가 바꿈        서버가 받는 것
──────────────────    ──────────────────    ──────────────
POST                →       DELETE        →   delete 처리
+ _method=DELETE
```

#### form body 방식 처리

```python
@app.middleware("http")
async def method_override(request: Request, call_next):
    if request.method == "POST":
        form = await request.form()
        method = form.get("_method", "").upper()

        if method in ("PUT", "DELETE", "PATCH"):
            request.scope["method"] = method  # 바꿔치기

    return await call_next(request)
```

#### query string 방식 처리

```python
@app.middleware("http")
async def method_override(request: Request, call_next):
    method = request.query_params.get("_method", "").upper()

    if method in ("PUT", "DELETE", "PATCH"):
        request.scope["method"] = method  # 바꿔치기

    return await call_next(request)
```

### form 방식 vs query string 방식

| | form 방식 | query string 방식 |
|--|--|--|
| 위치 | body 안에 숨김 | URL에 노출 |
| 코드 | `<input type="hidden">` | `action="?_method=DELETE"` |
| 간편함 | 약간 복잡 | 더 간단 |

---

## 정리

| 개념 | 한 줄 요약 |
|--|--|
| 미들웨어 | 모든 요청/응답이 지나가는 검문소 |
| DummyMiddleware | 아무것도 안 하는 껍데기 미들웨어 |
| Method Override | HTML form 한계를 극복하기 위해 POST에 진짜 메서드를 숨겨 보내는 것 |
