---
layout: post
title: "SQLAlchemy async - await 언제 써야 하나?"
date: 2026-04-10
categories: [Python, SQLAlchemy]
tags: [python, sqlalchemy, async, await, fastapi]
---

## 핵심 원칙 하나만 기억하기

> **DB 서버와 네트워크로 통신하는 작업 = `await` 필수**

SQLAlchemy async를 쓸 때 `await`가 헷갈리는 이유는, 결국 DB 서버가 **별도의 프로세스(네트워크 너머)**에 있기 때문입니다.  
`await`는 "이 작업이 끝날 때까지 기다려"라는 의미고, DB에 신호를 보내고 응답을 받아야 하는 작업이라면 반드시 필요합니다.

---

## await가 필요한 작업들

| 메서드 | DB에 보내는 신호 | await 필요 |
|--------|-----------------|-----------|
| `session.execute()` | 쿼리 실행 요청 | ✅ |
| `session.commit()` | "저장해!" | ✅ |
| `session.rollback()` | "되돌려!" | ✅ |
| `session.close()` | "연결 끊을게" | ✅ |
| `session.flush()` | "임시 반영해" | ✅ |

```python
# 모두 await 필요
await session.execute(stmt)
await session.commit()
await session.rollback()
await session.close()
await session.flush()
```

---

## await 없으면 어떻게 되나?

```python
# ❌ 잘못된 예시
session.rollback()  # DB에 롤백 요청만 던지고 바로 다음 줄로 넘어감
session.close()     # 롤백이 실제로 완료됐는지 보장 안 됨
```

`await` 없이 호출하면 coroutine 객체만 생성되고 **실제로는 아무것도 실행되지 않습니다.**  
에러도 안 나기 때문에 조용하게 버그가 생깁니다.

---

## fetchall()은 왜 await가 없나?

여기가 가장 헷갈리는 부분입니다.

```python
result = await session.execute(select(User))  # ← DB 통신 발생, await 필요
users = result.fetchall()                     # ← await 없음
```

`session.execute()`가 실행될 때 **DB 서버에서 모든 데이터를 받아서 메모리에 올려두는 작업까지 완료**됩니다.  
`fetchall()`은 이미 메모리에 있는 결과를 꺼내오는 것뿐이라 **네트워크 통신이 없습니다.**

```
session.execute()  →  DB 서버에 쿼리 전송 → 결과 수신 → 메모리에 저장
                       ↑ 이 과정이 비동기 (await 필요)

result.fetchall()  →  메모리에서 꺼내오기
                       ↑ 네트워크 없음, await 불필요
```

---

## 실전 예시

### 일반적인 조회 패턴

```python
async def get_users(session: AsyncSession):
    result = await session.execute(select(User))  # DB 통신
    return result.fetchall()                       # 메모리 접근
```

### 에러 처리 패턴

```python
async def create_user(session: AsyncSession, name: str):
    try:
        user = User(name=name)
        session.add(user)       # 메모리에 추가 (await 없음)
        await session.commit()  # DB에 저장 (await 필요)
    except Exception:
        await session.rollback()  # DB에 롤백 (await 필요)
        raise
    finally:
        await session.close()  # 연결 종료 (await 필요)
```

### context manager 사용 (권장)

`async with`를 쓰면 `close()`를 직접 호출하지 않아도 됩니다.

```python
async with AsyncSession(engine) as session:
    async with session.begin():  # 트랜잭션 시작
        result = await session.execute(select(User))
        users = result.fetchall()
# with 블록 종료 시 commit/rollback, close 자동 처리
```

---

## 한 눈에 정리

```
await 필요 (네트워크 통신)        await 불필요 (메모리 작업)
──────────────────────────────    ──────────────────────────
session.execute()                 result.fetchall()
session.commit()                  result.scalars()
session.rollback()                result.first()
session.close()                   session.add()
session.flush()                   session.expunge()
```

> `session.add()`는 메모리에 객체를 올리는 것뿐입니다.  
> 실제 DB 반영은 `commit()` 또는 `flush()` 때 발생합니다.
