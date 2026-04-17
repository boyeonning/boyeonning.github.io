---
layout: post
title: "Python 비동기(async/await) 완전 정리"
date: 2026-04-17
categories: [Python, Async]
tags: [python, async, await, coroutine, asyncio]
---

## 1. 비동기 파일 복사 코드 이해하기

```python
async with aio.open("input.jpg", "rb") as imagefile, aio.open("output.jpg", "wb") as outfile:
    while content := await imagefile.read(1024):
        await outfile.write(content)
```

이 코드는 이미지 파일을 **1024바이트씩 차례대로 읽어서** 다른 파일에 복사한다.  
순서는 동기 방식과 동일하다. 다른 점은 **기다리는 방식**이다.

---

## 2. 동기 vs 비동기

```python
# 동기
while content := imagefile.read(1024):   # 읽는 동안 프로그램 전체가 멈춤
    outfile.write(content)

# 비동기
while content := await imagefile.read(1024):  # 읽는 동안 다른 작업 가능
    await outfile.write(content)
```

1024바이트씩 차례대로 읽는 **순서는 동일**하다.  
차이는 파일을 읽는 **기다리는 시간** 동안 무엇을 하느냐이다.

- **동기**: 파일 읽는 동안 프로그램 전체가 멈춰서 기다림
- **비동기**: 파일 읽는 동안 다른 코루틴(작업)들이 그 틈에 실행됨

### 비유

| | 설명 |
|---|---|
| **동기** | 라면 끓이는 동안 냄비 앞에서 가만히 서서 기다림 |
| **비동기** | 라면 끓이는 동안 다른 요리를 하다가, 다 되면 돌아옴 |

---

## 3. await는 어디에 붙이나?

### 규칙: `async def`로 만들어진 함수 앞에 붙인다

```python
async def read():   # async def로 정의된 함수
    ...

await read()  # 그래서 await 붙임
```

### 모르겠으면 그냥 호출해보기

```python
result = imagefile.read(1024)
print(result)  # <coroutine object read at 0x...> 출력 → await 필요
               # 실제 데이터 출력 → await 불필요
```

### await가 붙는 것들
- 파일 읽기/쓰기 (aiofiles)
- 네트워크 요청 (aiohttp, httpx)
- DB 쿼리 (asyncpg, tortoise)
- `asyncio.sleep()`

### await가 안 붙는 것들
- 일반 계산 (`a + b`, `len()`)
- 일반 동기 함수 (`print()`, `str()`)
- 동기 파일 (`open()`으로 연 파일)

---

## 4. 코루틴이란?

**잠깐 멈췄다가 나중에 다시 이어서 실행할 수 있는 함수**

```python
# 일반 함수 - 시작하면 끝날 때까지 쭉 실행
def 일반함수():
    print("시작")
    print("끝")

# 코루틴 - 중간에 멈출 수 있음
async def 코루틴():
    print("시작")
    await asyncio.sleep(1)  # 여기서 잠깐 멈추고 다른 작업 양보
    print("끝")
```

### 비유

| | 설명 |
|---|---|
| **일반 함수** | 전화통화 — 통화 중엔 다른 전화 못 받음 |
| **코루틴** | 문자메시지 — 보내고 답장 기다리는 동안 다른 일 가능 |

### await가 멈추는 지점

`await`를 만나는 순간이 **"잠깐 멈추는 지점"**이다.

```python
async def 코루틴():
    print("A")
    await 뭔가()   # 여기서 멈추고 다른 코루틴한테 양보
    print("B")     # 뭔가()가 끝나면 다시 여기로 돌아옴
```

---

## 5. 왜 코루틴은 호출해도 바로 실행이 안 될까?

`async def` 함수를 호출하면 바로 실행되지 않고 **코루틴 객체**만 만들어진다.

```python
coro = imagefile.read(1024)  # 객체만 생성 (아직 실행 X)
await coro                   # 이벤트 루프한테 "이거 실행해줘" 하고 넘김
```

### 이벤트 루프가 타이밍을 조절하기 위해서

```
이벤트 루프 = 코루틴들의 교통정리 담당자
```

```
이벤트 루프 → 코루틴A 실행
코루틴A     → await 만남 → "저 잠깐 기다려야 해요"
이벤트 루프 → 코루틴B 실행
코루틴B     → await 만남 → "저도 기다려야 해요"
이벤트 루프 → 코루틴A 기다리던 거 끝났네 → 다시 실행
```

바로 실행해버리면 이벤트 루프가 타이밍을 조절할 수 없다.  
그래서 일단 객체로 만들어두고, `await`로 넘겨줄 때 실행하는 것이다.

> **한마디로**: `await`는 이벤트 루프한테 *"이 코루틴 실행해줘, 끝나면 알려줘"* 라고 넘기는 신호다.
