---
layout: post
title: "Jinja2 템플릿 엔진: include와 extends 쉽게 이해하기"
date: 2026-04-09
categories: [Python, Jinja2]
tags: [jinja2, template, fastapi, flask, python]
---

Jinja2에서 템플릿을 재사용하는 방법은 크게 두 가지입니다: `include`와 `extends`.  
비슷해 보이지만 **사용 목적이 다릅니다.**

---

## 한 줄 요약

| 키워드 | 개념 | 비유 |
|--------|------|------|
| `include` | 다른 템플릿을 **그대로 삽입** | 복사-붙여넣기 |
| `extends` | 부모 템플릿을 **상속받아 채우기** | 빈칸 채우기 |

---

## include — 그냥 갖다 붙이기

`include`는 다른 템플릿 파일을 현재 위치에 **통째로 삽입**합니다.  
공통 네비게이션, 푸터, 광고 배너처럼 **여러 페이지에서 반복되는 조각**에 적합합니다.

### 예시

**`_nav.html`** (공통 네비게이션)
```html
<nav>
  <a href="/">홈</a>
  <a href="/about">소개</a>
</nav>
```

**`index.html`**
{% raw %}
```html
{% include "_nav.html" %}

<h1>메인 페이지입니다</h1>
```
{% endraw %}

**렌더링 결과:**
```html
<nav>
  <a href="/">홈</a>
  <a href="/about">소개</a>
</nav>

<h1>메인 페이지입니다</h1>
```

`include`는 단순 삽입이라 **부모-자식 관계가 없습니다.**  
포함된 템플릿은 현재 컨텍스트(변수)를 그대로 공유합니다.

---

## extends — 틀을 물려받아 채우기

`extends`는 **부모 템플릿의 뼈대(레이아웃)를 상속**받고,  
자식 템플릿에서 `block`으로 지정된 영역만 **덮어씁니다.**

### 부모 템플릿 (`base.html`)

{% raw %}
```html
<!DOCTYPE html>
<html>
<head>
  <title>{% block title %}기본 제목{% endblock %}</title>
</head>
<body>
  <header>공통 헤더</header>

  <main>
    {% block content %}
    <!-- 자식이 이 부분을 채웁니다 -->
    {% endblock %}
  </main>

  <footer>공통 푸터</footer>
</body>
</html>
```
{% endraw %}

### 자식 템플릿 (`page.html`)

{% raw %}
```html
{% extends "base.html" %}

{% block title %}상품 목록{% endblock %}

{% block content %}
  <h1>오늘의 상품</h1>
  <ul>
    <li>사과</li>
    <li>바나나</li>
  </ul>
{% endblock %}
```
{% endraw %}

**렌더링 결과:**
```html
<!DOCTYPE html>
<html>
<head>
  <title>상품 목록</title>
</head>
<body>
  <header>공통 헤더</header>

  <main>
    <h1>오늘의 상품</h1>
    <ul>
      <li>사과</li>
      <li>바나나</li>
    </ul>
  </main>

  <footer>공통 푸터</footer>
</body>
</html>
```

자식 템플릿에서 `block`을 정의하지 않으면 부모의 기본값이 그대로 사용됩니다.

---

## super() — 부모 내용 유지하면서 추가하기

`extends`에서 부모 block의 내용을 **완전히 교체하지 않고 유지하면서 추가**하고 싶을 때 `super()`를 씁니다.

{% raw %}
```html
{% block content %}
  {{ super() }}  {# 부모의 기본 내용 포함 #}
  <p>추가로 보여줄 내용</p>
{% endblock %}
```
{% endraw %}

---

## include vs extends 언제 쓸까?

```
페이지 전체 레이아웃이 공통인가?
        │
       Yes → extends (헤더/푸터/사이드바가 전부 고정된 틀)
        │
        No
        │
반복되는 작은 조각인가?
        │
       Yes → include (버튼 컴포넌트, 광고 배너, 폼 일부)
```

### 실전 예시 (Flask/FastAPI)

```python
# FastAPI + Jinja2
from fastapi.templating import Jinja2Templates

templates = Jinja2Templates(directory="templates")

@app.get("/products")
def products(request: Request):
    return templates.TemplateResponse("products.html", {"request": request})
```

{% raw %}
```
templates/
  base.html         ← 공통 레이아웃 (extends 대상)
  _nav.html         ← 네비게이션 조각 (include 대상)
  products.html     ← {% extends "base.html" %}
  about.html        ← {% extends "base.html" %}
```
{% endraw %}

---

## 정리

- **`include`**: 재사용할 HTML 조각을 그 자리에 삽입. 부모-자식 관계 없음.
- **`extends`**: 레이아웃 틀을 상속받고, `block`으로 지정된 부분만 자식이 교체.
- 둘을 함께 쓸 수도 있습니다 — `extends`로 레이아웃을 상속받은 자식이 `include`로 작은 컴포넌트를 삽입하는 패턴이 일반적입니다.
