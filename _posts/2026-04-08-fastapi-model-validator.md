---
layout: post
title: "FastAPI @model_validator 완벽 정리"
date: 2026-04-08
categories: [FastAPI, Pydantic]
tags: [fastapi, pydantic, model_validator, python]
---

## @model_validator란?

모델 **전체 데이터**를 검증하거나 변환할 때 사용하는 데코레이터입니다.  
필드 간 상호 의존적인 유효성 검사에 특히 유용합니다.

---

## mode 옵션

| mode | 실행 시점 | 받는 인자 타입 |
|------|-----------|-------------|
| `'before'` | 필드 파싱 전 | `dict` (raw 데이터) |
| `'after'` | 필드 파싱 후 | 모델 인스턴스 |

---

## before vs after — 핵심 차이

`after`는 Pydantic이 타입 파싱을 **완료한 후** 실행됩니다.  
즉, **파싱 자체가 실패하면 `after`는 실행조차 안 됩니다.**

```
클라이언트 요청 (raw dict)
    ↓
before 실행  →  데이터 전처리 / 타입 변환
    ↓
Pydantic 파싱  →  타입 체크
    ↓
after 실행  →  필드 간 비즈니스 로직 검증
```

---

## before가 필요한 이유

클라이언트가 `list` 대신 문자열로 보내는 경우:

```json
{ "items": "apple,banana,cherry" }
```

`after`만 쓰면 파싱 단계에서 바로 422 에러가 나고, validator는 실행조차 안 됩니다.  
`before`에서 미리 변환해줘야 합니다.

```python
from fastapi import FastAPI
from pydantic import BaseModel, model_validator

app = FastAPI()

class OrderRequest(BaseModel):
    items: list[str]

    @model_validator(mode="before")
    @classmethod
    def normalize_items(cls, values: dict) -> dict:
        # 문자열로 오면 split()으로 리스트로 변환
        if isinstance(values.get("items"), str):
            values["items"] = values["items"].split(",")
        return values

    @model_validator(mode="after")
    def check_items(self) -> "OrderRequest":
        if len(self.items) == 0:
            raise ValueError("items는 비어있을 수 없습니다.")
        return self
```

### split()이 리스트로 변환하는 원리

```python
"apple,banana,cherry".split(",")
# → ["apple", "banana", "cherry"]  ✅ list[str]
```

---

## cls vs self

`before`에서는 `@classmethod` + `cls`를 사용하고, `after`에서는 `self`를 사용합니다.

| | before | after |
|--|--------|-------|
| 데코레이터 | `@classmethod` 필요 | 불필요 |
| 첫 번째 인자 | `cls` (클래스 자체) | `self` (인스턴스) |
| 이유 | 아직 인스턴스가 없음 | 파싱 후라 인스턴스 존재 |

```python
class OrderRequest(BaseModel):
    items: list[str]

    @model_validator(mode="before")
    @classmethod
    def before_example(cls, values):
        print(cls)     # <class 'OrderRequest'>  ← 클래스
        print(values)  # {"items": "apple,banana"}  ← raw dict
        return values

    @model_validator(mode="after")
    def after_example(self):
        print(self)        # items=['apple', 'banana']  ← 인스턴스
        print(self.items)  # ['apple', 'banana']
        return self
```

---

## 활용 예시

### 1. 날짜 범위 검증 (after)

```python
from datetime import date

class DateRangeRequest(BaseModel):
    start_date: date
    end_date: date

    @model_validator(mode="after")
    def check_date_range(self) -> "DateRangeRequest":
        if self.end_date <= self.start_date:
            raise ValueError("end_date는 start_date보다 이후여야 합니다.")
        return self
```

### 2. 비밀번호 확인 (after)

```python
class UserSignup(BaseModel):
    username: str
    password: str
    password_confirm: str

    @model_validator(mode="after")
    def passwords_match(self) -> "UserSignup":
        if self.password != self.password_confirm:
            raise ValueError("비밀번호가 일치하지 않습니다.")
        return self
```

### 3. 조건부 필수 필드 (after)

```python
class PaymentRequest(BaseModel):
    method: str          # "card" or "bank"
    card_number: str | None = None
    bank_account: str | None = None

    @model_validator(mode="after")
    def check_payment_info(self) -> "PaymentRequest":
        if self.method == "card" and not self.card_number:
            raise ValueError("카드 결제 시 card_number가 필요합니다.")
        if self.method == "bank" and not self.bank_account:
            raise ValueError("계좌 이체 시 bank_account가 필요합니다.")
        return self
```

---

## 핵심 정리

> - **before** = 파싱 전에 데이터를 "고쳐서" 넘겨주는 **전처리기**
> - **after** = 파싱된 데이터로 "검사"하는 **검증기**
> - `after`는 파싱 실패 시 실행조차 안 되므로, 타입 변환이 필요하면 반드시 `before` 사용
> - `before`는 `@classmethod` + `cls` 필수, `after`는 `self` 사용
