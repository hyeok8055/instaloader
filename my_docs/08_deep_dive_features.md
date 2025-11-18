# Instaloader 기능별 심층 분석

## 목차
0. [기초 개념: 웹은 어떻게 작동하는가?](#0-기초-개념-웹은-어떻게-작동하는가)
1. [익명 다운로드는 어떻게 가능한가?](#1-익명-다운로드는-어떻게-가능한가)
2. [로그인이 필수인 기능들과 그 이유](#2-로그인이-필수인-기능들과-그-이유)
3. [핵심 메커니즘: GraphQL API 활용](#3-핵심-메커니즘-graphql-api-활용)
4. [Selenium과의 차이점](#4-selenium과의-차이점)
5. [기능별 실행 플로우](#5-기능별-실행-플로우)

---

## 0. 기초 개념: 웹은 어떻게 작동하는가?

Instaloader를 이해하려면 먼저 **웹이 어떻게 작동하는지** 알아야 함. 많은 사람들이 당연하게 여기지만, 실제로는 복잡한 과정이 숨어있음.

### 0.1 웹 브라우저는 무엇을 하는가?

#### 일반인이 보는 과정
```
1. 주소창에 "instagram.com/nike" 입력
2. 엔터 키
3. 페이지가 짠! 하고 나타남
```

간단해 보이지만, 실제로는 **수십 개의 복잡한 단계**가 숨어있음.

#### 실제로 벌어지는 일 (상세 버전)

```mermaid
sequenceDiagram
    participant U as 사용자
    participant B as 브라우저
    participant DNS as DNS 서버
    participant IG as Instagram 서버
    participant CDN as CDN (이미지 서버)

    U->>B: instagram.com/nike 입력

    Note over B: 1단계: 주소 해석
    B->>DNS: "instagram.com이 어디 있어?"
    DNS-->>B: "IP: 157.240.24.174"

    Note over B: 2단계: 서버 연결
    B->>IG: TCP 연결 (3-way handshake)
    IG-->>B: 연결 성공

    Note over B: 3단계: HTTPS 보안 설정
    B->>IG: SSL/TLS 인증서 요청
    IG-->>B: 인증서 전달
    B->>B: 인증서 검증

    Note over B: 4단계: HTML 요청
    B->>IG: GET /nike HTTP/1.1
    Note over B,IG: Host: instagram.com, User-Agent: Chrome/142...
    IG-->>B: HTML 파일 (약 50KB)

    Note over B: 5단계: HTML 파싱
    B->>B: HTML 읽기 시작
    B->>B: script 태그 발견!
    B->>B: link 태그 발견!
    B->>B: img 태그 발견!

    Note over B: 6단계: 추가 자원 다운로드
    B->>IG: JavaScript 파일 10개 요청
    IG-->>B: JS 파일들 (총 2MB)
    B->>IG: CSS 파일 5개 요청
    IG-->>B: CSS 파일들 (총 500KB)

    Note over B: 7단계: JavaScript 실행
    B->>B: JavaScript 엔진 가동
    B->>B: React 초기화
    B->>IG: GraphQL API 호출 (프로필 데이터 요청!)
    IG-->>B: JSON 데이터 (프로필 정보)

    Note over B: 8단계: 이미지 다운로드
    B->>CDN: 프로필 사진 요청
    CDN-->>B: 프로필 사진 (200KB)
    B->>CDN: 게시글 이미지 12개 요청
    CDN-->>B: 이미지들 (총 5MB)

    Note over B: 9단계: 렌더링
    B->>B: CSS 적용
    B->>B: 레이아웃 계산
    B->>B: 화면에 그리기

    B->>U: 페이지 표시 완료!
```

💡 **핵심 포인트:**
- 브라우저는 **단순히 HTML만 받는 게 아님**
- HTML → JavaScript → CSS → 이미지 → 데이터(JSON) 순서로 여러 번 요청함
- 총 **30~50번의 네트워크 요청**이 발생함
- 전체 다운로드 용량: **5~10MB** 정도

#### 왜 이렇게 복잡한가?

**역사적 이유:**
```
1990년대 웹 (단순):
  - HTML만 있음
  - 정적인 페이지 (변화 없음)
  - 예: <html><body>안녕하세요</body></html>

2000년대 웹 (동적):
  - HTML + CSS + JavaScript
  - 사용자 상호작용 가능
  - 예: 버튼 클릭하면 색깔 바뀜

2020년대 웹 (앱처럼):
  - HTML + CSS + JavaScript + API + 실시간 데이터
  - 완전한 애플리케이션처럼 동작
  - 예: Instagram = 웹인데 앱처럼 느껴짐
```

### 0.2 HTTP 요청과 응답이란?

웹의 모든 통신은 **HTTP(HyperText Transfer Protocol)** 프로토콜을 사용함.

#### HTTP 요청 (Request) 구조

실제로 브라우저가 보내는 데이터:

```http
GET /nike HTTP/1.1
Host: instagram.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/142.0.0.0
Accept: text/html,application/json
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Cookie: sessionid=abc123xyz; csrftoken=def456
Connection: keep-alive
```

**각 줄의 의미:**

| 부분 | 설명 | 비유 |
|------|------|------|
| `GET /nike` | "nike 페이지를 달라" | 우체국에서 "우편물 주세요" |
| `Host: instagram.com` | "instagram.com 서버야" | "서울 강남구 주소로" |
| `User-Agent: Chrome/142...` | "나는 크롬 브라우저야" | "제 신분증입니다" |
| `Accept: text/html` | "HTML 형식으로 줘" | "한글로 적어주세요" |
| `Accept-Language: en-US` | "영어로 줘" | "영어 버전 주세요" |
| `Accept-Encoding: gzip` | "압축해서 보내도 돼" | "압축 파일로 보내도 됩니다" |
| `Cookie: sessionid=...` | "내 신원 확인용 티켓" | "회원카드입니다" |

#### HTTP 응답 (Response) 구조

Instagram 서버가 보내는 데이터:

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 51234
Content-Encoding: gzip
Set-Cookie: csrftoken=new_token_xyz; Path=/
Cache-Control: no-cache, no-store
Date: Sat, 18 Jan 2025 12:00:00 GMT

<!DOCTYPE html>
<html>
<head>
  <title>Nike (@nike) • Instagram</title>
  ...
</head>
<body>
  ...실제 HTML 내용...
</body>
</html>
```

**각 줄의 의미:**

| 부분 | 설명 | 비유 |
|------|------|------|
| `HTTP/1.1 200 OK` | "성공적으로 처리했어" | "우편물 찾았습니다" |
| `Content-Type: text/html` | "HTML 파일이야" | "문서 형식입니다" |
| `Content-Length: 51234` | "크기는 51,234바이트야" | "무게 51kg" |
| `Content-Encoding: gzip` | "gzip으로 압축했어" | "압축해서 보냈어요" |
| `Set-Cookie: ...` | "이 티켓 저장해줘" | "다음엔 이 번호표 보여주세요" |
| `Cache-Control: no-cache` | "캐시하지 마" | "매번 새로 확인하세요" |

#### 💡 핵심 개념: 상태 코드 (Status Code)

서버의 응답 상태를 숫자로 표현함:

```
2XX: 성공
  - 200 OK: 정상 처리
  - 201 Created: 새로 만들어짐

3XX: 리다이렉션 (다른 곳으로 이동)
  - 301 Moved Permanently: 영구 이동
  - 302 Found: 임시 이동

4XX: 클라이언트 에러 (요청이 잘못됨)
  - 400 Bad Request: 잘못된 요청
  - 401 Unauthorized: 인증 필요
  - 403 Forbidden: 권한 없음
  - 404 Not Found: 없음
  - 429 Too Many Requests: 요청 너무 많음 ← Instagram이 자주 보냄!

5XX: 서버 에러 (서버 문제)
  - 500 Internal Server Error: 서버 오류
  - 503 Service Unavailable: 서비스 불가
```

### 0.3 쿠키(Cookie)란 무엇인가?

쿠키는 **브라우저가 저장하는 작은 텍스트 파일**임.

#### 왜 쿠키가 필요한가?

**문제 상황:**
```
HTTP는 "무상태(Stateless)" 프로토콜임 = 기억력이 없음

사용자: (로그인) "아이디: user1, 비밀번호: pass1"
서버: "OK! 로그인 성공"

[1분 후]
사용자: (게시글 보기) "내 피드 보여줘"
서버: "넌 누구야? 다시 로그인해" ← 방금 로그인했는데!

이래서는 매 요청마다 로그인해야 함 → 불편!
```

**해결책: 쿠키**
```
사용자: (로그인) "아이디: user1, 비밀번호: pass1"
서버: "OK! 이 티켓 받아: sessionid=abc123xyz" ← 쿠키!
브라우저: (쿠키 저장)

[1분 후]
사용자: (게시글 보기) "내 피드 보여줘. 티켓: sessionid=abc123xyz"
서버: "아, abc123xyz는 user1이구나! 여기 피드" ← 기억함!
```

#### 쿠키의 구조

```javascript
// 실제 Instagram 쿠키 예시
document.cookie = "sessionid=IGSCab8d3f2e1a4c5b6d7e8f9; Domain=.instagram.com; Path=/; Secure; HttpOnly; Max-Age=31536000"
```

**각 부분의 의미:**

| 속성 | 값 | 설명 |
|------|-----|------|
| `sessionid` | 이름 | 이 쿠키의 이름 (로그인 세션 ID) |
| `IGSCab8d3f2e1a4c5b6d7e8f9` | 값 | 실제 세션 식별자 (암호화됨) |
| `Domain=.instagram.com` | 도메인 | instagram.com과 모든 서브도메인에서 사용 |
| `Path=/` | 경로 | 모든 경로에서 이 쿠키 전송 |
| `Secure` | 보안 | HTTPS에서만 전송 (HTTP는 안 됨) |
| `HttpOnly` | HTTP 전용 | JavaScript로 접근 불가 (보안) |
| `Max-Age=31536000` | 유효기간 | 1년 (초 단위) |

#### 쿠키 vs 세션

```
쿠키 (Cookie):
  - 브라우저에 저장됨
  - 클라이언트가 관리
  - 용량 제한: 4KB
  - 예: sessionid=abc123

세션 (Session):
  - 서버에 저장됨
  - 서버가 관리
  - 용량 제한 없음
  - 예: {user_id: 123, username: "nike", login_time: ...}

둘의 관계:
  쿠키(sessionid) ← 열쇠
  세션(서버 데이터) ← 금고

  쿠키를 보내면 → 서버가 세션을 찾음
```

### 0.4 API란 무엇인가?

**API (Application Programming Interface)** = **프로그램들이 대화하는 방법**

#### 일상 비유

```
레스토랑 = 서버
손님 = 클라이언트 (브라우저)
메뉴판 = API 문서
주문 = API 요청
음식 = API 응답

손님: "메뉴판 보고 불고기버거 주세요" ← 정해진 형식으로 주문
직원: "네, 여기 불고기버거입니다" ← 정해진 형식으로 응답

만약 메뉴판 없이:
손님: "고기 있는 빵 사이에 낀 거요"
직원: "??? 뭘 원하시는지 모르겠어요"
```

#### 웹 API의 종류

**1. REST API (전통적 방식)**

```
규칙: URL로 자원을 표현함

예시:
  GET /api/users/123          → 사용자 123번 정보 가져오기
  GET /api/users/123/posts    → 사용자 123번의 게시글 가져오기
  POST /api/users             → 새 사용자 만들기
  PUT /api/users/123          → 사용자 123번 수정
  DELETE /api/users/123       → 사용자 123번 삭제

장점: 이해하기 쉬움
단점: 여러 정보 받으려면 여러 번 요청해야 함
```

**2. GraphQL API (현대적 방식, Instagram이 사용)**

```
규칙: 한 번에 원하는 것만 정확히 요청

예시:
  POST /graphql
  {
    user(id: 123) {
      name
      posts(first: 10) {
        title
        likes
      }
    }
  }

  → 사용자 이름 + 최근 게시글 10개(제목, 좋아요)를 한 번에!

장점: 필요한 것만 받음, 요청 횟수 적음
단점: 배우기 어려움
```

#### Instagram의 GraphQL API 구조

```
엔드포인트 (Endpoint):
  https://www.instagram.com/graphql/query

요청 방식:
  POST (데이터를 body에 담아 보냄)

요청 형식:
  {
    "doc_id": "7950326061742207",     ← 어떤 쿼리인지 (메뉴 번호)
    "variables": {                    ← 상세 옵션
      "id": "25025320",               ← nike의 사용자 ID
      "first": 12                     ← 12개만 가져와
    }
  }

응답 형식:
  {
    "data": {
      "user": {
        "id": "25025320",
        "username": "nike",
        "edge_owner_to_timeline_media": {
          "edges": [
            {
              "node": {
                "shortcode": "ABC123",
                "display_url": "https://...",
                "edge_liked_by": {"count": 52341}
              }
            }
          ]
        }
      }
    }
  }
```

### 0.5 JSON이란 무엇인가?

**JSON (JavaScript Object Notation)** = **데이터를 표현하는 텍스트 형식**

#### 왜 JSON을 사용하는가?

**문제:**
```
컴퓨터끼리 데이터를 주고받을 때, 어떻게 표현할까?

나쁜 방법:
  "사용자 이름은 nike고 팔로워는 300만명이고 게시글은 1234개임"
  → 파싱하기 어려움

좋은 방법:
  {"username": "nike", "followers": 3000000, "posts": 1234}
  → 구조화되어 있어서 프로그램이 쉽게 읽음
```

#### JSON 문법 기초

```javascript
// 1. 기본 타입
{
  "문자열": "Hello",
  "숫자": 123,
  "소수": 3.14,
  "참거짓": true,
  "없음": null
}

// 2. 배열 (리스트)
{
  "좋아하는_과일": ["사과", "바나나", "딸기"]
}

// 3. 중첩 (객체 안에 객체)
{
  "사용자": {
    "이름": "nike",
    "프로필": {
      "팔로워": 3000000,
      "팔로잉": 150
    }
  }
}

// 4. 배열 안에 객체
{
  "게시글들": [
    {"제목": "Just Do It", "좋아요": 5000},
    {"제목": "New Shoes", "좋아요": 3000}
  ]
}
```

#### Instagram JSON 응답 실제 예시

```json
{
  "data": {
    "user": {
      "id": "25025320",
      "username": "nike",
      "full_name": "Nike",
      "biography": "Just Do It. Est. 1972",
      "edge_followed_by": {
        "count": 300147823
      },
      "edge_follow": {
        "count": 150
      },
      "edge_owner_to_timeline_media": {
        "count": 1234,
        "edges": [
          {
            "node": {
              "id": "3287362888123456789",
              "shortcode": "ABC123xyz",
              "display_url": "https://scontent.cdninstagram.com/.../photo.jpg",
              "is_video": false,
              "taken_at_timestamp": 1705622400,
              "edge_liked_by": {
                "count": 52341
              },
              "edge_media_to_caption": {
                "edges": [
                  {
                    "node": {
                      "text": "Just Do It #nike #running"
                    }
                  }
                ]
              }
            }
          }
        ]
      }
    }
  }
}
```

**이 JSON을 Python에서 읽는 법:**

```python
import json

# JSON 문자열을 Python 딕셔너리로 변환
data = json.loads(json_string)

# 데이터 접근
username = data['data']['user']['username']  # "nike"
followers = data['data']['user']['edge_followed_by']['count']  # 300147823
first_post = data['data']['user']['edge_owner_to_timeline_media']['edges'][0]['node']
post_url = first_post['display_url']  # 이미지 URL
post_likes = first_post['edge_liked_by']['count']  # 52341
```

### 0.6 Python requests 라이브러리 기초

**requests** = **Python에서 HTTP 요청을 쉽게 보낼 수 있는 라이브러리**

#### 기본 사용법

```python
import requests

# 1. 간단한 GET 요청
response = requests.get('https://instagram.com')
print(response.status_code)  # 200
print(response.text)         # HTML 내용

# 2. 헤더 추가
headers = {
    'User-Agent': 'Mozilla/5.0 ...',
    'Accept-Language': 'en-US'
}
response = requests.get('https://instagram.com', headers=headers)

# 3. 쿠키 전송
cookies = {
    'sessionid': 'abc123xyz'
}
response = requests.get('https://instagram.com', cookies=cookies)

# 4. POST 요청 (데이터 보내기)
data = {
    'username': 'myuser',
    'password': 'mypass'
}
response = requests.post('https://instagram.com/login', data=data)

# 5. JSON 응답 파싱
response = requests.get('https://api.example.com/user/123')
user_data = response.json()  # 자동으로 JSON → Python 딕셔너리
print(user_data['name'])
```

#### requests.Session() - 쿠키 자동 관리

```python
import requests

# Session 없이 (쿠키 수동 관리)
response1 = requests.get('https://instagram.com')
cookies = response1.cookies  # 쿠키 저장
response2 = requests.get('https://instagram.com/nike', cookies=cookies)  # 쿠키 재사용

# Session 사용 (쿠키 자동 관리) ← Instaloader가 사용하는 방법!
session = requests.Session()
session.get('https://instagram.com')  # 쿠키 자동 저장됨
session.get('https://instagram.com/nike')  # 저장된 쿠키 자동으로 전송됨
session.get('https://instagram.com/adidas')  # 또 자동 전송됨!
```

💡 **Session의 장점:**
- 쿠키 자동 관리 (매번 cookies= 안 써도 됨)
- 헤더 한 번만 설정하면 계속 사용됨
- 연결 재사용으로 속도 향상 (TCP keep-alive)

#### Instaloader가 requests를 사용하는 방법

```python
class InstaloaderContext:
    def __init__(self):
        # Session 객체 생성
        self._session = requests.Session()

        # 기본 헤더 설정 (모든 요청에 적용됨)
        self._session.headers.update({
            'User-Agent': 'Mozilla/5.0 ...',
            'Accept-Language': 'en-US',
            'X-Instagram-AJAX': '1'
        })

    def get_json(self, path, params):
        """Instagram API 호출"""
        url = f'https://www.instagram.com/{path}'

        # Session으로 요청 (헤더, 쿠키 자동 포함됨)
        response = self._session.get(url, params=params)

        # JSON 파싱
        return response.json()
```

---

## 1. 익명 다운로드는 어떻게 가능한가?

### 1.1 기본 사용법의 비밀

```bash
# 로그인 없이 프로필 전체 다운로드
instaloader nike
```

**"어떻게 로그인 없이 다른 사람의 게시글, 이미지, 동영상을 모두 가져올 수 있을까?"**

#### 핵심 원리: Instagram은 사실 웹에 데이터를 공개하고 있음

우리가 웹 브라우저로 Instagram을 방문하면, 페이지가 보이기 위해서는 **뒤에서 데이터를 가져오는 작업**이 필요함.

예를 들어:
1. 크롬 브라우저로 `instagram.com/nike`를 방문함
2. 브라우저는 Instagram 서버에 "nike의 게시글 좀 보여줘"라고 요청함
3. Instagram은 **GraphQL API**라는 통로를 통해 데이터를 JSON 형태로 줌
4. 브라우저가 받은 JSON 데이터를 예쁘게 꾸며서 우리에게 보여줌

**Instaloader의 핵심 아이디어:**
> "브라우저가 하는 3번 과정을 우리가 직접 하면 되지 않을까?"

즉, HTML을 파싱할 필요 없이 **Instagram이 브라우저에게 주는 순수한 데이터를 직접 받아오는 것**임.

```mermaid
sequenceDiagram
    participant User as 사용자
    participant IL as Instaloader
    participant IG as Instagram GraphQL API

    User->>IL: instaloader nike
    Note over IL: 브라우저인 척 하기 시작
    IL->>IL: 익명 세션 생성<br/>(빈 쿠키로 브라우저 흉내)
    IL->>IG: "nike 프로필 정보 주세요"<br/>GraphQL Query (doc_id: 7950326061742207)
    Note over IG: "공개 프로필이네? OK!"
    IG-->>IL: {username: "nike", userid: 12345,<br/>게시글 12개 데이터}

    Note over IL: "아직 더 있네?"
    IL->>IG: "다음 12개 주세요" (페이지네이션)
    IG-->>IL: 게시글 12개 더...

    Note over IL: 이미지 URL 발견!
    IL->>IG: 이미지 파일 다운로드
    IG-->>IL: 실제 JPG 파일
    IL->>User: 폴더에 파일 저장 완료!
```

#### 코드 레벨로 보는 동작 원리

**1단계: 브라우저처럼 위장하기** ([instaloadercontext.py:201-211](../insta/instaloader/instaloadercontext.py#L201-L211))

```python
def get_anonymous_session(self) -> requests.Session:
    """익명 세션 = 로그인 안 한 브라우저 흉내내기"""
    session = requests.Session()  # HTTP 요청을 보낼 도구

    # Instagram이 기대하는 쿠키들을 빈 값으로 세팅
    # (실제 브라우저도 처음 방문하면 이렇게 빈 쿠키를 보냄)
    session.cookies.update({
        'sessionid': '',      # 로그인 ID: 비어있음 = 익명
        'mid': '',            # 머신 ID: 어떤 PC인지
        'ig_pr': '1',         # Instagram 설정값들
        'ig_vw': '1920',      # 화면 너비: 1920px
        'csrftoken': '',      # 보안 토큰
        'ds_user_id': ''      # 유저 ID: 비어있음 = 익명
    })

    # 브라우저처럼 보이는 헤더 추가
    session.headers.update(self._default_http_header(empty_session_only=True))
    return session
```

💡 **왜 이렇게 하면 될까?**
- Instagram은 공개 프로필의 경우 **익명 사용자에게도 데이터를 제공**함
- 웹사이트가 작동하려면 어쩔 수 없이 제공해야 함 (안 그러면 검색엔진도 못 읽음)
- 단, "너 봇 아니지?" 확인을 위해 **브라우저처럼 보여야 함**

**2단계: Instagram에게 데이터 요청하기** ([instaloadercontext.py:498-529](../insta/instaloader/instaloadercontext.py#L498-L529))

```python
def graphql_query(self, query_hash: str, variables: Dict[str, Any]) -> Dict[str, Any]:
    """
    GraphQL API 호출 = Instagram의 데이터 창고에 접근

    Args:
        query_hash: 어떤 데이터를 원하는지 (일종의 메뉴 번호)
        variables: 상세 조건 (예: 사용자 ID, 몇 개 가져올지)
    """
    # 요청 조건을 JSON 문자열로 변환
    variables_json = json.dumps(variables, separators=(',', ':'))

    # Instagram API 엔드포인트에 요청
    resp_json = self.get_json(
        'graphql/query',  # API 주소
        params={
            'query_hash': query_hash,      # 메뉴 번호
            'variables': variables_json     # 상세 주문
        },
        session=tmpsession
    )
    return resp_json  # Instagram이 준 데이터 반환
```

**3단계: 프로필의 게시글 가져오기** ([structures.py:1204-1236](../insta/instaloader/structures.py#L1204-L1236))

```python
def get_posts(self) -> NodeIterator[Post]:
    """프로필의 모든 게시글을 순회하며 가져오기"""

    # 현재 로그인 상태인지 확인
    logged_in = self._context.is_logged_in

    return NodeIterator(
        context=self._context,

        # 핵심: 로그인 여부에 따라 다른 데이터 경로 사용
        edge_extractor=(
            # 로그인했으면 이 경로에서 데이터 추출
            (lambda d: d["data"]["xdt_api__v1__feed__user_timeline_graphql_connection"])
            if logged_in
            # 익명이면 이 경로에서 데이터 추출
            else (lambda d: d["data"]["user"]["edge_owner_to_timeline_media"])
        ),

        query_variables={
            "data": {"count": 12, ...},  # 한 번에 12개씩 가져옴
            # 로그인: username으로 검색, 익명: userid로 검색
            **({"username": self.username} if logged_in else {"id": self.userid}),
        },

        # 💡 여기가 핵심!
        # 로그인 여부에 따라 다른 "메뉴 번호"(doc_id) 사용
        doc_id="7898261790222653" if logged_in else "7950326061742207",
        query_hash=None,

        # 익명일 때는 첫 페이지 데이터를 미리 받아옴
        first_data=(None if logged_in else self._metadata("edge_owner_to_timeline_media")),
    )
```

💡 **여기서 중요한 점:**
- `doc_id="7950326061742207"` 이라는 숫자는 Instagram이 정해놓은 **"익명 사용자용 프로필 게시글 조회 API"**의 식별번호임
- Instagram 개발자들이 만든 수백 개의 API 중 하나임
- 브라우저 개발자 도구(F12)로 네트워크 탭을 보면 이 요청들이 실제로 날아가는 걸 볼 수 있음

### 1.2 왜 차단되지 않을까?

#### 이유 1: 공식 API를 사용하기 때문

Instagram 입장에서는 **Instaloader의 요청과 브라우저의 요청을 구별할 수 없음**.

```
일반 브라우저:
  크롬 → Instagram API → "7950326061742207" 호출 → 데이터 받음

Instaloader:
  Python → Instagram API → "7950326061742207" 호출 → 데이터 받음
```

둘 다 똑같은 API를 호출하고 있음! 차이점은 받은 데이터를 어떻게 사용하느냐만 다를 뿐.

**실제 네트워크 패킷 비교:**

```http
=== 크롬 브라우저가 보내는 요청 ===
POST /graphql/query HTTP/1.1
Host: www.instagram.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/142.0.0.0
Accept: */*
Accept-Language: en-US,en;q=0.9
Content-Type: application/x-www-form-urlencoded
Cookie: sessionid=...; csrftoken=...
X-Instagram-AJAX: 1
X-CSRFToken: ...

doc_id=7950326061742207&variables={"id":"25025320","first":12}

=== Instaloader가 보내는 요청 ===
POST /graphql/query HTTP/1.1
Host: www.instagram.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/142.0.0.0
Accept: */*
Accept-Language: en-US,en;q=0.9
Content-Type: application/x-www-form-urlencoded
Cookie: sessionid=...; csrftoken=...
X-Instagram-AJAX: 1
X-CSRFToken: ...

doc_id=7950326061742207&variables={"id":"25025320","first":12}
```

→ **완전히 동일함!** Instagram 서버는 둘을 구별할 수 없음.

#### 이유 2: 속도 제한을 지킴 (Rate Limiting)

##### Rate Limiting이란?

**Rate Limiting** = 일정 시간 동안 허용되는 요청 횟수 제한

```
Instagram의 Rate Limit (추정):
  - 익명 사용자: 분당 ~20개 요청
  - 로그인 사용자: 분당 ~60개 요청
  - 이 이상 요청하면 → 429 Too Many Requests 에러

왜 이런 제한이 있을까?
  1. 서버 과부하 방지 (수백만 명이 동시 접속)
  2. 봇/크롤러 차단 (정상 사용자 보호)
  3. 공정한 자원 배분
```

##### Instagram이 봇을 탐지하는 방법

```mermaid
flowchart TD
    A[Instagram 서버] --> B{요청 패턴 분석}

    B --> C[요청 빈도 체크]
    C --> C1{"1분에 100개 요청?"}
    C1 -->|Yes| BLOCK[차단!]
    C1 -->|No| D

    B --> D[행동 패턴 분석]
    D --> D1{"모든 요청이<br/>정확히 1초 간격?"}
    D1 -->|Yes| BLOCK
    D1 -->|No| E

    B --> E[IP 주소 체크]
    E --> E1{"같은 IP에서<br/>수천 개 계정 접속?"}
    E1 -->|Yes| BLOCK
    E1 -->|No| F

    B --> F[세션 수명 체크]
    F --> F1{"세션이 10초 만에<br/>100개 프로필 방문?"}
    F1 -->|Yes| BLOCK
    F1 -->|No| PASS[통과]

    style BLOCK fill:#f99
    style PASS fill:#9f9
```

##### Instaloader의 Rate Limiting 전략

**전략 1: 랜덤 대기 시간**

```python
import time
import random

class RateController:
    def __init__(self):
        self.sleep = True  # 대기 활성화
        self.min_delay = 1.0  # 최소 1초
        self.max_delay = 3.0  # 최대 3초

    def wait_before_query(self, query_type: str):
        """요청 전에 랜덤하게 대기"""
        if self.sleep:
            delay = random.uniform(self.min_delay, self.max_delay)
            time.sleep(delay)

# 실제 사용 예:
게시글 1 다운로드 → 2.34초 대기
게시글 2 다운로드 → 1.87초 대기
게시글 3 다운로드 → 2.91초 대기
게시글 4 다운로드 → 1.23초 대기

# 나쁜 봇의 예:
게시글 1 다운로드 → 1.00초 대기 ← 규칙적!
게시글 2 다운로드 → 1.00초 대기
게시글 3 다운로드 → 1.00초 대기
게시글 4 다운로드 → 1.00초 대기 ← 봇으로 감지됨!
```

💡 **왜 랜덤이 중요한가?**
- 사람은 **불규칙적**으로 행동함 (클릭 간격이 매번 다름)
- 봇은 **규칙적**으로 행동함 (프로그램이니까)
- 랜덤 = 사람처럼 보이게 하는 핵심!

**전략 2: 429 에러 자동 처리**

```python
class InstaloaderContext:
    def get_json(self, path, params, session=None, max_retries=3):
        """JSON 요청 (에러 자동 처리)"""

        for attempt in range(max_retries):
            try:
                response = self._session.get(url, params=params)

                # 상태 코드 체크
                if response.status_code == 200:
                    return response.json()  # 성공!

                elif response.status_code == 429:
                    # Too Many Requests - 요청 한도 초과!
                    self._handle_rate_limit(response)
                    # → handle_rate_limit에서 자동으로 대기하고 재시도

                elif response.status_code == 404:
                    raise ProfileNotFoundError("프로필을 찾을 수 없음")

                elif response.status_code == 403:
                    raise ForbiddenError("접근 권한 없음")

            except requests.exceptions.ConnectionError:
                # 네트워크 오류 - 재시도
                if attempt < max_retries - 1:
                    time.sleep(5)
                    continue
                raise

    def _handle_rate_limit(self, response):
        """429 에러 처리"""

        # Retry-After 헤더 확인 (Instagram이 알려주는 대기 시간)
        retry_after = response.headers.get('Retry-After')

        if retry_after:
            # Instagram: "600초 후에 다시 시도하세요"
            wait_time = int(retry_after)
        else:
            # 헤더 없으면 기본 600초 (10분)
            wait_time = 600

        self.log(f"Rate limit! {wait_time}초 대기 중...")

        # 10분 대기를 1분씩 나눠서 진행 상황 표시
        for i in range(wait_time // 60):
            time.sleep(60)
            remaining = wait_time - (i + 1) * 60
            self.log(f"남은 시간: {remaining}초")

        # 마지막 나머지 초
        time.sleep(wait_time % 60)
```

**전략 3: 쿼리 타입별 별도 제한**

```python
class RateController:
    def __init__(self):
        # 쿼리 타입별 마지막 요청 시간 기록
        self.query_timestamps = {
            'graphql': [],      # GraphQL 쿼리
            'iphone': [],       # iPhone API
            'media': [],        # 미디어 다운로드
        }

        # 타입별 제한 (초당 요청 수)
        self.limits = {
            'graphql': 0.5,   # 2초당 1개
            'iphone': 1.0,    # 1초당 1개
            'media': 2.0,     # 0.5초당 1개 (미디어는 조금 빠름)
        }

    def wait_before_query(self, query_type: str):
        """쿼리 타입에 따라 다른 대기 시간 적용"""

        # 최근 요청 시간 가져오기
        timestamps = self.query_timestamps[query_type]
        now = time.time()

        # 최근 1분간의 요청만 유지
        timestamps = [t for t in timestamps if now - t < 60]
        self.query_timestamps[query_type] = timestamps

        # 요청 수 체크
        if len(timestamps) > 0:
            # 마지막 요청으로부터 경과 시간
            time_since_last = now - timestamps[-1]

            # 제한 시간보다 빠르면 대기
            min_interval = 1.0 / self.limits[query_type]
            if time_since_last < min_interval:
                wait = min_interval - time_since_last
                time.sleep(wait)

        # 현재 시간 기록
        self.query_timestamps[query_type].append(time.time())
```

**전략 4: 지수 백오프 (Exponential Backoff)**

```python
def download_with_backoff(self, url, max_retries=5):
    """재시도 시 대기 시간을 점진적으로 늘림"""

    for attempt in range(max_retries):
        try:
            return self._download(url)

        except TooManyRequestsException:
            if attempt == max_retries - 1:
                raise  # 마지막 시도는 에러 발생

            # 지수 백오프: 2^attempt초 대기
            wait_time = 2 ** attempt  # 1초 → 2초 → 4초 → 8초 → 16초
            jitter = random.uniform(0, wait_time * 0.1)  # ±10% 랜덤
            total_wait = wait_time + jitter

            self.log(f"시도 {attempt + 1} 실패, {total_wait:.1f}초 대기")
            time.sleep(total_wait)

# 실제 실행 예:
시도 1: 실패 → 1.05초 대기
시도 2: 실패 → 2.18초 대기
시도 3: 실패 → 4.32초 대기
시도 4: 실패 → 8.77초 대기
시도 5: 성공!
```

💡 **지수 백오프가 효과적인 이유:**
- 일시적 문제 (네트워크 끊김): 빠르게 재시도
- 지속적 문제 (서버 과부하): 천천히 재시도
- 서버 부담 감소

#### 이유 3: 브라우저처럼 보이는 헤더

##### User-Agent란?

**User-Agent** = "나는 어떤 브라우저/프로그램인지" 알려주는 문자열

```http
실제 크롬 브라우저:
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/142.0.0.0 Safari/537.36

파이썬 requests (기본값):
User-Agent: python-requests/2.31.0

→ Instagram은 두 번째를 즉시 차단함!
```

**User-Agent 문자열 분석:**

```
Mozilla/5.0
  ↑ 역사적 이유 (모든 브라우저가 이렇게 시작함)

(Windows NT 10.0; Win64; x64)
  ↑ 운영체제: Windows 10, 64비트

AppleWebKit/537.36
  ↑ 렌더링 엔진 버전

(KHTML, like Gecko)
  ↑ 호환성 표시 (역사적 이유)

Chrome/142.0.0.0
  ↑ 크롬 버전: 142

Safari/537.36
  ↑ Safari 호환성 표시
```

##### Instaloader의 헤더 전략

```python
def _default_http_header(self, empty_session_only: bool = False) -> Dict[str, str]:
    """실제 브라우저가 보내는 것과 100% 동일한 헤더"""

    header = {
        # 1. User-Agent: 최신 크롬 브라우저로 위장
        'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 '
                      '(KHTML, like Gecko) Chrome/142.0.0.0 Safari/537.36',

        # 2. Accept: 어떤 형식을 받을 수 있는지
        'Accept': '*/*',  # 모든 형식 OK

        # 3. Accept-Language: 선호 언어
        'Accept-Language': 'en-US,en;q=0.9',  # 영어 선호 (90%)

        # 4. Accept-Encoding: 압축 지원
        'Accept-Encoding': 'gzip, deflate, br',  # gzip, deflate, brotli

        # 5. Connection: 연결 유지
        'Connection': 'keep-alive',  # 여러 요청에 같은 연결 재사용

        # 6. Referer: 어디서 왔는지 (이전 페이지)
        'Referer': 'https://www.instagram.com/',

        # 7. Instagram 전용 헤더들
        'X-Instagram-AJAX': '1',  # AJAX 요청임을 표시
        'X-Requested-With': 'XMLHttpRequest',  # XHR 요청

        # 8. 보안 헤더
        'Sec-Fetch-Dest': 'empty',
        'Sec-Fetch-Mode': 'cors',
        'Sec-Fetch-Site': 'same-origin',

        # 9. 브라우저 힌트 (최신 크롬만 보냄)
        'Sec-Ch-Ua': '"Not_A Brand";v="8", "Chromium";v="142"',
        'Sec-Ch-Ua-Mobile': '?0',  # 모바일 아님
        'Sec-Ch-Ua-Platform': '"Linux"',
    }

    if not empty_session_only and self._session:
        # 로그인한 경우 추가 헤더
        header['X-CSRFToken'] = self._session.cookies.get('csrftoken', '')

    return header
```

##### 왜 이렇게 많은 헤더가 필요한가?

**실험: 헤더를 하나씩 빼보면?**

```python
# 1. User-Agent만 빼면?
→ 403 Forbidden (즉시 차단)

# 2. X-Instagram-AJAX 빼면?
→ 400 Bad Request (잘못된 요청)

# 3. Referer 빼면?
→ 200 OK (동작은 하지만 의심스러움)

# 4. Accept-Language 빼면?
→ 200 OK (동작함, 영어로 응답)

# 5. Sec-Fetch-* 빼면?
→ 200 OK (오래된 브라우저로 인식됨)

결론: User-Agent, X-Instagram-AJAX는 필수!
```

##### 실제 브라우저 vs Instaloader 헤더 비교

```http
=== 실제 크롬 브라우저 (F12로 확인) ===
:authority: www.instagram.com
:method: POST
:path: /graphql/query
:scheme: https
accept: */*
accept-encoding: gzip, deflate, br
accept-language: en-US,en;q=0.9
content-type: application/x-www-form-urlencoded
cookie: sessionid=...; csrftoken=...
origin: https://www.instagram.com
referer: https://www.instagram.com/nike/
sec-ch-ua: "Not_A Brand";v="8", "Chromium";v="142"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Linux"
sec-fetch-dest: empty
sec-fetch-mode: cors
sec-fetch-site: same-origin
user-agent: Mozilla/5.0 (X11; Linux x86_64) Chrome/142.0.0.0
x-csrftoken: ABC123...
x-instagram-ajax: 1
x-requested-with: XMLHttpRequest

=== Instaloader ===
(위와 100% 동일!)
```

---

## 2. 로그인이 필수인 기능들과 그 이유

### 2.1 왜 어떤 건 로그인 없이 되고, 어떤 건 안 될까?

#### Instagram의 데이터 공개 정책

Instagram은 데이터를 3단계로 나눔:

```
1. 완전 공개 데이터
   - 공개 프로필의 게시글 목록
   - 게시글 이미지 (표준 화질)
   - 기본 프로필 정보 (이름, 바이오)
   → 익명으로 접근 가능 ✅

2. 제한적 공개 데이터
   - 댓글 내용
   - 좋아요 누른 사람 목록
   - 고화질 이미지
   → 로그인 필요 🔐

3. 완전 비공개 데이터
   - 스토리
   - 하이라이트
   - 비공개 계정 게시글
   - 저장한 게시물
   → 로그인 + 권한 필요 🔐🔐
```

### 2.2 로그인 필수 기능 목록과 이유

| 기능 | 명령어 | 왜 로그인이 필요할까? |
|------|--------|---------------------|
| **스토리 다운로드** | `--stories` | 스토리는 24시간 후 사라짐 → Instagram이 신원 확인하려고 함 |
| **비공개 프로필** | (자동) | 팔로워만 볼 수 있게 설정했으니 당연히 로그인 필요 |
| **하이라이트** | `--highlights` | 스토리 모음집 → 스토리와 동일한 정책 |
| **댓글 가져오기** | 자동 | 스팸 방지 목적, 익명 크롤링 막기 위함 |
| **좋아요 목록** | `post.get_likes()` | 프라이버시 보호 (누가 좋아요 눌렀는지는 민감 정보) |
| **저장된 게시물** | `--saved` | 본인만 볼 수 있는 개인 북마크 |
| **고화질 미디어** | (자동) | 로그인하면 iPhone API 사용 가능 → 원본 화질 제공 |

### 2.3 스토리 다운로드는 어떻게 작동할까?

```bash
instaloader --login myaccount --stories nike
```

```mermaid
sequenceDiagram
    participant User as 사용자
    participant IL as Instaloader
    participant IG as Instagram API

    User->>IL: --login myaccount
    Note over IL: "로그인 과정 시작"
    IL->>IG: POST /api/v1/web/accounts/login/ajax/<br/>{username: "myaccount", password: "암호화됨"}
    Note over IG: 비밀번호 확인 중...
    IG-->>IL: ✅ 로그인 성공!<br/>쿠키: sessionid=ABC123, csrftoken=XYZ
    IL->>IL: 쿠키를 파일로 저장 (다음엔 재사용)

    User->>IL: --stories nike
    IL->>IG: GET /api/v1/feed/user/nike/story/<br/>쿠키: sessionid=ABC123 (로그인 증명)
    Note over IG: "이 사람 로그인했네?<br/>nike를 팔로우하나 확인..."

    alt nike를 팔로우 중이거나 공개 스토리
        IG-->>IL: ✅ {스토리 3개 데이터}
        loop 각 스토리
            IL->>IG: 이미지/동영상 URL 다운로드
            IG-->>IL: 고화질 파일
            IL->>User: 파일 저장: nike_story_20250118_001.jpg
        end
    else nike를 팔로우 안 함
        IG-->>IL: ❌ 403 Forbidden (권한 없음)
        IL->>User: 에러: nike의 스토리를 볼 권한이 없습니다
    end
```

#### 로그인 과정 상세 분석

**1단계: CSRF 토큰 받아오기**
```python
def login(self, user, passwd):
    """Instagram 로그인 과정"""

    # 먼저 Instagram 홈페이지 방문 (토큰 받기 위해)
    session.get('https://www.instagram.com/')

    # 받은 쿠키에서 CSRF 토큰 추출
    # CSRF = 보안 공격 방지용 임시 토큰
    csrf_token = session.cookies.get_dict()['csrftoken']
    session.headers.update({'X-CSRFToken': csrf_token})
```

💡 **CSRF 토큰이 뭘까?**
- 웹사이트가 "이 요청이 진짜 우리 사이트에서 온 거 맞아?"를 확인하는 방법
- 비유: 콘서트 입장 팔찌 (들어올 때 받고, 나갔다 들어올 때 확인)

**2단계: 비밀번호 암호화해서 보내기**
```python
    # 비밀번호를 그대로 보내면 위험! 암호화함
    # 형식: #PWD_INSTAGRAM_BROWSER:0:현재시각:비밀번호
    enc_password = '#PWD_INSTAGRAM_BROWSER:0:{}:{}'.format(
        int(datetime.now().timestamp()),  # 현재 시각 (초 단위)
        passwd
    )

    # 로그인 API 호출
    login = session.post(
        'https://www.instagram.com/api/v1/web/accounts/login/ajax/',
        data={
            'enc_password': enc_password,
            'username': user
        }
    )
```

**3단계: 세션 저장 (다음부터는 로그인 안 해도 됨)**
```python
    # 로그인 성공! sessionid 쿠키 받음
    # 이 쿠키만 있으면 "나 로그인한 사람이야" 증명 가능
    self._session = session
    self.username = user

    # 나중에 이 세션을 파일로 저장할 수 있음
    # → 다음 실행 때 비밀번호 다시 입력 안 해도 됨
```

💡 **세션 쿠키 = 영화관 팔찌**
- 한 번 표 끊으면 팔찌 받음
- 화장실 갔다 와도 팔찌만 보여주면 됨 (표 다시 안 끊어도 됨)
- Instaloader는 이 팔찌를 파일로 저장해서 다음에도 사용

### 2.4 익명 vs 로그인: 무엇이 다를까?

```mermaid
flowchart TD
    Start[instaloader nike 실행] --> Check{로그인했나요?}

    Check -->|아니요 익명| Anon[익명 세션 시작]
    Check -->|네 로그인함| Auth[로그인 세션 시작]

    Anon --> AnonAPI["GraphQL 요청<br/>doc_id: 7950326061742207<br/>(익명용 API)"]
    Auth --> AuthAPI["GraphQL 요청<br/>doc_id: 7898261790222653<br/>(로그인용 API)<br/><br/>추가로 iPhone API도 사용"]

    AnonAPI --> AnonLimit[제한된 데이터만 받음]
    AuthAPI --> AuthFull[모든 데이터 + 고화질]

    AnonLimit --> AnonResult["받는 것:<br/>✅ 공개 게시글<br/>✅ 표준 화질 이미지<br/>❌ 댓글<br/>❌ 좋아요 목록<br/>❌ 스토리"]
    AuthFull --> AuthResult["받는 것:<br/>✅ 모든 게시글<br/>✅ 최고 화질 이미지<br/>✅ 댓글<br/>✅ 좋아요 목록<br/>✅ 스토리"]
```

#### 코드에서 어떻게 분기될까?

**예시 1: 이미지 화질 선택**
```python
@property
def url(self) -> str:
    """게시글 이미지 URL 가져오기"""

    # 조건: 이미지 + 로그인함 + iPhone API 지원
    if (self.typename == "GraphImage" and
        self._context.iphone_support and
        self._context.is_logged_in):
        try:
            # 로그인했으면 iPhone API에서 최고 화질 가져옴
            # image_versions2.candidates[0] = 가장 높은 해상도
            return self._iphone_struct['image_versions2']['candidates'][0]['url']
        except:
            pass

    # 익명이면 표준 화질 (display_url)
    # 브라우저에서 보이는 것과 같은 화질
    return self._field('display_url')
```

💡 **비유:**
- 익명: 유튜브 480p 화질 (공짜)
- 로그인: 유튜브 4K 화질 (프리미엄 멤버십)

**예시 2: 댓글 가져오기**
```python
def get_comments(self):
    """게시글의 댓글 가져오기"""

    # 첫 번째 체크: 로그인 안 했으면 바로 에러
    if not self._context.is_logged_in:
        raise LoginRequiredException("댓글을 보려면 로그인이 필요합니다")

    # 로그인했으면 댓글 API 호출...
```

**예시 3: 비공개 프로필 체크**
```python
if profile.is_private:
    # 비공개 프로필 발견!

    if not self.context.is_logged_in:
        # 로그인 안 했으면 볼 수 없음
        raise LoginRequiredException("이 프로필은 로그인이 필요합니다")

    # 로그인은 했는데...
    if not profile.followed_by_viewer and self.context.username != profile.username:
        # 팔로우 안 했고 본인 계정도 아니면 볼 수 없음
        raise PrivateProfileNotFollowedException("비공개 계정이라 팔로우가 필요합니다")
```

💡 **체크 순서:**
1. 공개 프로필인가? → 익명도 OK
2. 비공개인데 로그인 안 함? → ❌
3. 로그인했는데 팔로우 안 함? → ❌
4. 팔로우함 또는 본인 계정? → ✅

---

## 3. 핵심 메커니즘: GraphQL API 활용

### 3.1 GraphQL이 뭘까? (쉽게 설명)

#### 전통적인 REST API vs GraphQL

**REST API (옛날 방식):**
```
사용자 정보를 얻으려면:
GET /api/user/123         → {id, name, email, phone, address, ...}

게시글을 얻으려면:
GET /api/user/123/posts   → [{post1}, {post2}, ...]

댓글을 얻으려면:
GET /api/post/456/comments → [{comment1}, comment2}, ...]
```
→ 3번의 요청 필요, 필요 없는 데이터도 많이 받음

**GraphQL (새로운 방식, Instagram이 사용):**
```
한 번에 원하는 것만 요청:
POST /graphql/query
{
  user(id: 123) {
    name
    posts(first: 10) {
      image_url
      comments {
        text
      }
    }
  }
}
```
→ 1번 요청으로 끝, 필요한 것만 정확히 받음

💡 **현실 비유:**
- REST API = 식당에서 세트 메뉴만 주문 가능 (A세트: 햄버거+콜라+감자튀김, 싫어도 다 받음)
- GraphQL = 원하는 것만 골라담기 (햄버거만, 콜라만, 감자튀김 빼고 등)

### 3.2 Instagram의 GraphQL 구조

#### doc_id가 뭐길래?

Instagram은 수백 개의 "미리 만들어진 쿼리"를 제공함. 각각에 고유 번호(doc_id)가 있음.

**왜 이렇게 복잡하게 만들었을까?**

```
일반 GraphQL (Facebook이 만든 원본):
  - 사용자가 직접 쿼리 작성
  - 예: "{ user(id: 123) { name, posts { title } } }"
  - 유연하지만, 악의적 쿼리 가능
  - 예: "{ user { posts { comments { replies { ... 무한 반복 } } } }" ← 서버 폭파!

Instagram의 GraphQL (제한된 버전):
  - Instagram이 미리 만든 쿼리만 사용 가능
  - 각 쿼리에 번호(doc_id) 부여
  - 사용자는 번호만 선택 + 변수만 입력
  - 안전하고 성능 최적화됨
```

**실제 doc_id 목록:**

```python
# Instagram이 정해놓은 쿼리 목록 (일부)

# 프로필 관련
"7950326061742207"  # 익명으로 프로필 게시글 조회 (GraphImage, GraphVideo 등)
"7898261790222653"  # 로그인해서 프로필 게시글 조회 (더 많은 정보 포함)
"7c16654f22c819fb63d1183034a5162f"  # 프로필 메타데이터 (팔로워, 팔로잉 수 등)

# 게시글 관련
"8845758582119845"  # 특정 게시글 상세 정보 (shortcode로 조회)
"7845543455542541"  # 릴스 목록
"f883d95537fbcd400f466f63d42bd8a1"  # 저장한 게시물 (로그인 필요)
"e31a871f7301132ceaab56507a66bbb7"  # 태그된 게시물

# 스토리/하이라이트 관련
"ba71d2e6d08839fc7b1897f04e2e1a90"  # 스토리 목록 (로그인 필요)
"45246d3fe16ccc6577e0bd297a5db1ab"  # 하이라이트 목록

# 인터랙션 관련
"d4d88dc1500312af6f937f7b804c68c3"  # 좋아요 누른 사용자 목록
"bc3296d1ce80a24b1b6e40b1e72903f5"  # 댓글 목록
"1cb6ec562846122743b61e492c85999f"  # 팔로워 목록
"c76146de99bb02f6415203be841dd25a"  # 팔로잉 목록

# 검색 관련
"7c8a1055f69f5069e4c7ae3f2b2f7e6c"  # 해시태그 검색
"6b802d0e3a64a3e3d6c1f7b2e8f7e6d5"  # 위치 태그 검색
```

#### doc_id는 어떻게 찾았을까?

**방법 1: 브라우저 개발자 도구로 직접 보기**

```
1. 크롬에서 Instagram 열기
2. F12 누르기 (개발자 도구)
3. Network 탭 선택
4. Fetch/XHR 필터 켜기
5. Instagram에서 프로필 방문
6. graphql/query 요청 찾기
7. Request Payload 확인
   → doc_id: "7950326061742207" 발견!
```

**방법 2: Instagram JavaScript 파일 분석**

Instagram의 웹 앱은 React로 만들어짐. JavaScript 파일 안에 모든 doc_id가 하드코딩되어 있음:

```javascript
// Instagram의 JavaScript 코드 (난독화되어 있음)
e.exports = {
  ProfilePostsQuery: "7950326061742207",
  FeedTimelineQuery: "7898261790222653",
  PostDetailQuery: "8845758582119845",
  // ... 수백 개 더
}
```

Instaloader 개발자들은 이 파일을 정기적으로 분석해서 doc_id를 업데이트함.

💡 **비유: 패스트푸드 메뉴판**
```
일반 주문:
  손님: "빵, 패티, 치즈, 양상추, 토마토, 소스 조합해주세요"
  직원: "???" (복잡함, 실수 가능)

메뉴 번호 주문:
  손님: "1번 세트요" (doc_id)
  직원: "불고기버거 세트죠? 사이즈는요?" (variables)
  손님: "라지요"
  직원: "바로 나갑니다" (빠르고 정확)

Instagram도 마찬가지:
  - doc_id = 메뉴 번호
  - variables = 사이즈, 옵션
```

#### doc_id의 생명주기

```mermaid
flowchart TD
    A[Instagram 개발자가<br/>새 기능 개발] --> B[GraphQL 쿼리 작성]
    B --> C[쿼리에 doc_id 부여<br/>예: 7950326061742207]
    C --> D[JavaScript 번들에 포함]
    D --> E[웹 배포]

    E --> F{Instaloader 개발자}
    F --> G[JavaScript 파일 다운로드]
    G --> H[난독화 코드 분석]
    H --> I[doc_id 추출]
    I --> J[Instaloader 코드 업데이트]

    E --> K{사용자}
    K --> L[브라우저로 Instagram 방문]
    L --> M[JavaScript 실행]
    M --> N[doc_id로 API 호출]

    J --> O{Instaloader 사용자}
    O --> P[instaloader nike 실행]
    P --> N

    style C fill:#9f9
    style I fill:#9f9
    style N fill:#9f9
```

#### variables란 무엇인가?

doc_id는 "어떤 종류의 쿼리"인지 결정하고, variables는 "구체적인 조건"을 지정함.

**예시: 프로필 게시글 조회**

```python
# doc_id는 고정
doc_id = "7950326061742207"  # "프로필 게시글 조회" 쿼리

# variables로 구체적 조건 지정
variables = {
    "id": "25025320",       # 어떤 사용자? (nike의 ID)
    "first": 12,            # 몇 개? (12개)
    "after": None           # 어디서부터? (처음부터 = None)
}

# 다른 사용자 조회는 variables만 바꾸면 됨
variables2 = {
    "id": "123456789",      # adidas의 ID
    "first": 24,            # 24개 가져오기
    "after": "QVFBb..."     # 두 번째 페이지부터
}
```

**variables의 일반적인 필드들:**

| 필드 | 타입 | 설명 | 예시 |
|------|------|------|------|
| `id` | 문자열 | 사용자 ID (숫자지만 문자열로 전송) | `"25025320"` |
| `username` | 문자열 | 사용자 이름 | `"nike"` |
| `first` | 숫자 | 가져올 개수 (페이지 크기) | `12` |
| `after` | 문자열 | 페이지네이션 커서 (다음 페이지) | `"QVFBb3dzODE..."` |
| `shortcode` | 문자열 | 게시글 ID | `"ABC123xyz"` |
| `include_reel` | 불린 | 릴스 포함 여부 | `true` |
| `fetch_mutual` | 불린 | 맞팔 여부 포함 | `false` |

**실제 요청 예시:**

```http
POST /graphql/query HTTP/1.1
Host: www.instagram.com
Content-Type: application/x-www-form-urlencoded

doc_id=7950326061742207&variables={"id":"25025320","first":12,"after":null}
```

이것을 URL 인코딩하면:

```
doc_id=7950326061742207&variables=%7B%22id%22%3A%2225025320%22%2C%22first%22%3A12%2C%22after%22%3Anull%7D
```

### 3.3 실제 요청 과정 보기

**Python 코드가 보내는 요청:**
```python
def doc_id_graphql_query(self, doc_id: str, variables: Dict[str, Any]) -> Dict[str, Any]:
    """
    doc_id를 사용한 GraphQL 쿼리

    Args:
        doc_id: 어떤 쿼리인지 (메뉴 번호)
        variables: 상세 옵션 (사이즈, 맵기 등)
    """
    # 옵션을 JSON 문자열로 변환
    variables_json = json.dumps(variables, separators=(',', ':'))

    # Instagram API에 POST 요청
    resp_json = self.get_json(
        'graphql/query',  # Instagram API 주소
        params={
            'variables': variables_json,     # 상세 옵션
            'doc_id': doc_id,                # 메뉴 번호
            'server_timestamps': 'true'      # 시간 정보 포함
        },
        use_post=True  # POST 방식 사용
    )
    return resp_json
```

**실제 HTTP 요청 (네트워크로 날아가는 것):**
```
POST https://www.instagram.com/graphql/query
Content-Type: application/x-www-form-urlencoded
Cookie: sessionid=...; csrftoken=...;
User-Agent: Mozilla/5.0 ...

doc_id=7950326061742207
&variables={"id":"25025320","count":12}
&server_timestamps=true
```

**Instagram이 응답하는 데이터:**
```json
{
  "data": {
    "user": {
      "id": "25025320",
      "username": "nike",
      "full_name": "Nike",
      "edge_owner_to_timeline_media": {
        "count": 1234,
        "edges": [
          {
            "node": {
              "shortcode": "ABC123xyz",
              "display_url": "https://scontent.cdninstagram.com/.../photo.jpg",
              "is_video": false,
              "taken_at_timestamp": 1705622400,
              "edge_liked_by": {"count": 52341},
              "edge_media_to_caption": {
                "edges": [{"node": {"text": "Just Do It"}}]
              }
            }
          }
          // ... 11개 더
        ],
        "page_info": {
          "has_next_page": true,
          "end_cursor": "QVFBa... (다음 페이지 토큰)"
        }
      }
    }
  }
}
```

### 3.4 페이지네이션: 게시글 1000개를 어떻게 다 가져올까?

Instagram은 한 번에 12개씩만 줌. 1000개를 받으려면 83번 요청해야 함.

```mermaid
sequenceDiagram
    participant IL as Instaloader
    participant IG as Instagram API

    Note over IL: nike 프로필 (게시글 1000개)

    IL->>IG: 1차 요청: doc_id + {id: "nike", count: 12}
    IG-->>IL: 게시글 #1~12 + 토큰:"AAAA"

    Note over IL: "아직 988개 남았네?"
    IL->>IG: 2차 요청: doc_id + {id: "nike", count: 12, after: "AAAA"}
    IG-->>IL: 게시글 #13~24 + 토큰:"BBBB"

    IL->>IG: 3차 요청: doc_id + {id: "nike", count: 12, after: "BBBB"}
    IG-->>IL: 게시글 #25~36 + 토큰:"CCCC"

    Note over IL: ... 80번 더 반복 ...

    IL->>IG: 83차 요청: doc_id + {id: "nike", count: 12, after: "ZZZZ"}
    IG-->>IL: 게시글 #989~1000 + has_next_page: false

    Note over IL: "끝!"
```

**코드로 보는 페이지네이션:**
```python
class NodeIterator:
    """게시글을 하나씩 순회하는 반복자"""

    def __iter__(self):
        # 첫 페이지 요청
        data = self._query()  # Instagram API 호출

        # 첫 페이지의 게시글들 반환
        for edge in data['edges']:
            yield self._node_wrapper(edge['node'])

        # 다음 페이지가 있는 동안 계속 반복
        while data['page_info']['has_next_page']:
            # 다음 페이지 토큰을 변수에 추가
            self._query_variables['after'] = data['page_info']['end_cursor']

            # 다음 페이지 요청
            data = self._query()

            # 받은 게시글들 반환
            for edge in data['edges']:
                yield self._node_wrapper(edge['node'])

        # has_next_page가 false면 루프 종료 → 끝!
```

💡 **비유: 도서관에서 책 빌리기**
- 한 번에 12권씩만 대출 가능
- 1권째~12권째: 첫 방문, "다음 12권 예약" 쪽지 받음
- 13권째~24권째: 쪽지 제시, 또 "다음 12권 예약" 쪽지 받음
- 반복...
- 마지막 책: "더 이상 없음" 쪽지 받음

---

## 4. Selenium과의 차이점

### 4.1 왜 Selenium으로 하면 막힐까?

#### Selenium의 접근 방식

```mermaid
flowchart LR
    A[Selenium 실행] --> B[Chrome 브라우저 켜짐]
    B --> C[Instagram 접속]
    C --> D{Instagram 봇 탐지 시스템}

    D -->|체크1| E1["JavaScript:<br/>navigator.webdriver?"]
    D -->|체크2| E2["마우스 움직임 있나?"]
    D -->|체크3| E3["스크롤 속도 정상?"]
    D -->|체크4| E4["헤더 일치하나?"]

    E1 -->|True| F[❌ 봇 감지!]
    E2 -->|No| F
    E3 -->|비정상| F
    E4 -->|불일치| F

    F --> G[로그인 차단 또는 Captcha]

    style F fill:#f99
    style G fill:#f99
```

**Selenium이 걸리는 이유들:**

**1. navigator.webdriver 속성**
```javascript
// Instagram이 실행하는 JavaScript 코드
if (navigator.webdriver === true) {
    // "어? 이 사람 Selenium 쓰네?"
    console.log("봇 감지!");
    showCaptcha();
}
```

실제 브라우저: `navigator.webdriver = undefined`
Selenium: `navigator.webdriver = true` ← 이게 문제!

**2. 비정상적 행동 패턴**
```python
# Selenium 코드 예시
driver.get("https://instagram.com/nike")
time.sleep(0.1)  # 너무 짧음!
driver.find_element_by_class_name("post").click()
time.sleep(0.1)  # 너무 짧음!
driver.find_element_by_class_name("next").click()
# 0.1초마다 클릭 → 사람이 아님!
```

**3. 마우스 움직임 없음**
- 실제 사람: 마우스 움직임 → 클릭 → 스크롤 (자연스러움)
- Selenium: 클릭만 일어남 (마우스 궤적 없음)

**4. 리소스 낭비**
- Selenium은 전체 브라우저를 실행 → 메모리 500MB+
- JavaScript, CSS, 이미지 전부 렌더링 → 느림
- 우리는 JSON 데이터만 필요한데...

### 4.2 Instaloader는 어떻게 다를까?

#### 핵심 차이점

| 관점 | Selenium | Instaloader |
|------|----------|-------------|
| **무엇을 실행?** | Chrome/Firefox 브라우저 전체 | Python HTTP 요청만 |
| **데이터 수집 방법** | HTML 파싱 (BeautifulSoup 등) | JSON 직접 받기 |
| **Instagram 입장** | "브라우저 + 봇 = 의심" | "그냥 브라우저처럼 보임" |
| **속도** | 게시글 1개당 5~10초 | 게시글 1개당 0.5초 |
| **메모리** | 500MB+ | 50MB |
| **차단 위험** | 높음 ⚠️ | 낮음 ✅ |

#### Instaloader의 우회 전략

**전략 1: 브라우저를 실행하지 않음**
```python
# Selenium (브라우저 필요)
from selenium import webdriver
driver = webdriver.Chrome()  # Chrome 실행 (무거움)
driver.get("https://instagram.com/nike")

# Instaloader (브라우저 불필요)
import requests
session = requests.Session()  # 가벼움
response = session.get("https://instagram.com/graphql/query?...")
```

💡 **비유:**
- Selenium = 자동차 전체를 운전해서 마트 가기
- Instaloader = 전화로 배달 주문하기 (더 빠르고 효율적)

**전략 2: 실제 브라우저의 헤더 완벽 복사**
```python
def _default_http_header(self) -> Dict[str, str]:
    """Chrome 브라우저가 보내는 것과 100% 동일한 헤더"""
    return {
        # 크롬 142 버전이라고 주장
        'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64) '
                      'AppleWebKit/537.36 (KHTML, like Gecko) '
                      'Chrome/142.0.0.0 Safari/537.36',

        # 브라우저가 지원하는 압축 방식
        'Accept-Encoding': 'gzip, deflate',

        # 선호하는 언어
        'Accept-Language': 'en-US,en;q=0.8',

        # Instagram 전용 헤더들
        'X-Instagram-AJAX': '1',
        'X-Requested-With': 'XMLHttpRequest',

        # 어디서 왔는지
        'Referer': 'https://www.instagram.com/',

        # ... 실제 브라우저의 모든 헤더 포함
    }
```

이 헤더들은 실제로 크롬 개발자 도구(F12)에서 복사한 것임!

**전략 3: 사람처럼 행동하기**
```python
def do_sleep(self):
    """사람처럼 불규칙하게 대기"""
    if self.sleep:
        # 1~3초 사이 랜덤 대기
        wait_time = random.uniform(1, 3)
        time.sleep(wait_time)

# 사용 예:
게시글 1 다운로드 → 2.3초 대기 ✅
게시글 2 다운로드 → 1.7초 대기 ✅
게시글 3 다운로드 → 2.8초 대기 ✅
# 불규칙 → 사람처럼 보임!

# Selenium (나쁜 예):
게시글 1 다운로드 → 1초 대기 ❌
게시글 2 다운로드 → 1초 대기 ❌
게시글 3 다운로드 → 1초 대기 ❌
# 너무 규칙적 → 봇으로 보임!
```

**전략 4: 세션 재사용 (로그인 반복 안 함)**
```python
def save_session_to_file(self, sessionfile):
    """로그인 쿠키를 파일로 저장"""
    pickle.dump(self.save_session(), sessionfile)

# 첫 실행:
instaloader --login myaccount --stories nike
# → 비밀번호 입력, 세션 파일 저장 (~/.config/instaloader/session-myaccount)

# 다음 실행 (몇 주 후):
instaloader --login myaccount --stories nike
# → 비밀번호 입력 안 함! 저장된 세션 재사용
```

Selenium은 매번 로그인 → 의심받음
Instaloader는 한 번 로그인 → 오래 사용 → 정상 사용자로 보임

**전략 5: 에러 자동 처리**
```python
def get_json(self, path, params):
    """JSON 받기 (에러 자동 처리)"""
    try:
        # API 요청
        response = session.get(url)
        return response.json()

    except TooManyRequestsException:  # 429 에러
        # Instagram: "너무 많이 요청했어!"
        print("요청 한도 초과, 10분 대기 중...")
        time.sleep(600)  # 10분 대기
        return self.get_json(path, params)  # 다시 시도

    except ConnectionException:  # 네트워크 오류
        # 최대 3번까지 재시도
        if attempt < 3:
            time.sleep(5)
            return self.get_json(path, params)
```

Selenium은 에러나면 멈춤 → 사람이 직접 처리
Instaloader는 에러나면 자동으로 대기/재시도 → 안정적

### 4.3 비교 요약

```mermaid
flowchart TD
    subgraph Selenium
        S1[Chrome 실행 500MB] --> S2[Instagram 접속]
        S2 --> S3[JavaScript 실행]
        S3 --> S4[HTML 렌더링]
        S4 --> S5[BeautifulSoup 파싱]
        S5 --> S6[❌ navigator.webdriver 감지]
        S6 --> S7[차단!]
        style S7 fill:#f99
    end

    subgraph Instaloader
        I1[requests 모듈 50MB] --> I2[GraphQL API 호출]
        I2 --> I3[JSON 직접 받음]
        I3 --> I4[✅ 브라우저처럼 보임]
        I4 --> I5[데이터 저장]
        style I5 fill:#9f9
    end
```

---

## 5. 기능별 실행 플로우

### 5.1 기본 프로필 다운로드 상세 플로우

```bash
instaloader nike
```

이 한 줄 명령어가 내부에서 어떻게 작동하는지 단계별로 보자:

```mermaid
flowchart TD
    Start[instaloader nike 입력] --> Parse[명령줄 인자 파싱]
    Parse --> Init[Instaloader 객체 생성]

    Init --> AnonSession["익명 세션 생성<br/>(빈 쿠키, 브라우저 헤더)"]

    AnonSession --> ProfileReq["Profile.from_username 호출"]
    ProfileReq --> API1["1차 API 요청<br/>doc_id: 7950326061742207<br/>variables: {username: 'nike'}"]

    API1 --> Meta["응답 파싱<br/>userid, 프로필사진 URL,<br/>게시글 수, 바이오 등"]

    Meta --> PicDownload{프로필 사진 다운로드?}
    PicDownload -->|Yes| DownloadPic["프로필 사진 다운로드<br/>파일명: nike_12345.jpg"]
    PicDownload -->|No| GetPosts
    DownloadPic --> GetPosts["profile.get_posts() 호출"]

    GetPosts --> Iterator["NodeIterator 생성<br/>(게시글 순회 준비)"]

    Iterator --> Loop{더 가져올 게시글?}
    Loop -->|첫 페이지| FirstPage["첫 12개 게시글 받음"]
    Loop -->|다음 페이지| NextPage["페이지네이션 요청<br/>(after: 토큰)"]

    FirstPage --> ProcessPost
    NextPage --> ProcessPost["각 게시글 처리"]

    ProcessPost --> CheckType{게시글 타입?}
    CheckType -->|이미지| DownloadImg["이미지 다운로드"]
    CheckType -->|동영상| DownloadVid["동영상 다운로드"]
    CheckType -->|여러 개| DownloadMulti["슬라이드 모두 다운로드"]

    DownloadImg --> SaveMeta
    DownloadVid --> SaveMeta
    DownloadMulti --> SaveMeta["메타데이터 JSON 저장"]

    SaveMeta --> Loop
    Loop -->|끝| End[완료!]

    style Start fill:#9f9
    style End fill:#9f9
```

**코드로 보는 실행 과정:**

```python
# 1단계: 초기화
loader = Instaloader(
    download_comments=False,  # 댓글 다운 안 함 (익명이라 불가능)
    download_geotags=False,   # 위치 정보 안 함
    save_metadata=True        # JSON 메타데이터는 저장
)

# 2단계: 프로필 객체 생성
profile = Profile.from_username(loader.context, "nike")
# 내부적으로:
#   - GraphQL API 호출 (doc_id: 7950326061742207)
#   - Instagram이 응답: {userid: 12345, username: "nike", ...}

# 3단계: 프로필 사진 다운로드
loader.download_profilepic(profile)
# 파일 생성: ./nike/2025-01-18_12-34-56_UTC_profile_pic.jpg

# 4단계: 게시글 순회
for post in profile.get_posts():
    # 각 게시글마다:

    # 4-1: 게시글 타입 확인
    if post.typename == "GraphImage":
        # 단일 이미지
        url = post.url  # 이미지 URL
        loader.download_pic(filename, url, ...)

    elif post.typename == "GraphVideo":
        # 동영상
        video_url = post.video_url
        loader.download_pic(filename, video_url, ...)

    elif post.typename == "GraphSidecar":
        # 여러 개 (슬라이드)
        for i, sidecar_node in enumerate(post.get_sidecar_nodes()):
            url = sidecar_node.display_url
            loader.download_pic(f"{filename}_{i+1}", url, ...)

    # 4-2: 메타데이터 저장 (JSON)
    metadata = {
        "shortcode": post.shortcode,
        "caption": post.caption,
        "likes": post.likes,
        "date": post.date.isoformat(),
        ...
    }
    with open(f"{filename}.json", "w") as f:
        json.dump(metadata, f)

    # 4-3: 사람처럼 대기
    loader.context.do_sleep()  # 1~3초 랜덤 대기
```

**생성되는 파일 구조:**
```
./nike/
├── 2025-01-18_12-34-56_UTC_profile_pic.jpg
├── 2025-01-18_10-20-30_UTC.jpg          (게시글 1)
├── 2025-01-18_10-20-30_UTC.json         (메타데이터 1)
├── 2025-01-17_15-40-22_UTC.mp4          (게시글 2, 동영상)
├── 2025-01-17_15-40-22_UTC.json         (메타데이터 2)
├── 2025-01-16_08-15-10_UTC_1.jpg        (게시글 3, 슬라이드 1번)
├── 2025-01-16_08-15-10_UTC_2.jpg        (게시글 3, 슬라이드 2번)
├── 2025-01-16_08-15-10_UTC_3.jpg        (게시글 3, 슬라이드 3번)
└── 2025-01-16_08-15-10_UTC.json         (메타데이터 3)
```

### 5.2 해시태그 다운로드

```bash
instaloader "#photography"
```

```mermaid
sequenceDiagram
    participant User as 사용자
    participant IL as Instaloader
    participant IG as Instagram API

    User->>IL: instaloader "#photography"

    Note over IL: "#photography" 파싱<br/>(# 제거 → "photography")

    IL->>IL: Hashtag 객체 생성

    IL->>IG: GraphQL: 해시태그 메타데이터 요청
    Note over IG: doc_id로 해시태그 정보 조회
    IG-->>IL: {name: "photography",<br/>게시글 수: 1.5M,<br/>프로필 사진: ...}

    IL->>IG: get_posts() 시작<br/>(최신 게시글 12개)
    IG-->>IL: [{post1}, {post2}, ..., {post12}]

    loop 각 게시글
        IL->>IL: 게시글 작성자 확인
        Note over IL: photography 해시태그를<br/>사용한 여러 사용자의 게시글

        IL->>IG: 미디어 다운로드 요청
        IG-->>IL: 이미지/비디오 파일

        IL->>IL: 파일 저장<br/>폴더: #photography/<br/>파일명: 작성자명_날짜_시간.jpg
    end

    IL->>User: 다운로드 완료!
```

**파일 구조:**
```
./#photography/
├── user123_2025-01-18_15-20-30_UTC.jpg
├── user123_2025-01-18_15-20-30_UTC.json
├── photographer_2025-01-18_14-10-15_UTC.jpg
├── photographer_2025-01-18_14-10-15_UTC.json
├── artist_2025-01-18_13-05-42_UTC.jpg
├── artist_2025-01-18_13-05-42_UTC.json
...
```

### 5.3 스토리 다운로드 (로그인 필요)

```bash
instaloader --login myaccount --stories nike
```

```mermaid
flowchart TD
    Start["instaloader --login myaccount<br/>--stories nike"] --> CheckSession{세션 파일 존재?}

    CheckSession -->|Yes 있음| LoadFile["파일 로드:<br/>~/.config/instaloader/<br/>session-myaccount"]
    CheckSession -->|No 없음| AskPW["비밀번호 입력 프롬프트"]

    AskPW --> LoginAPI["Instagram 로그인 API 호출<br/>/api/v1/web/accounts/login/ajax/"]
    LoginAPI --> SaveSession["세션 쿠키 파일로 저장"]

    LoadFile --> Verify["세션 유효성 검증<br/>(test_login 호출)"]
    SaveSession --> Verify

    Verify --> Valid{유효함?}
    Valid -->|No 만료됨| AskPW
    Valid -->|Yes 유효함| GetStory["스토리 요청<br/>iPhone API:<br/>/api/v1/feed/user/{userid}/story/"]

    GetStory --> StoryExist{스토리 존재?}
    StoryExist -->|No| ErrorMsg["에러 출력:<br/>'nike에 스토리 없음'"]
    StoryExist -->|Yes| ParseStory["스토리 데이터 파싱<br/>reel.items[] 순회"]

    ParseStory --> Loop{더 있음?}
    Loop -->|Yes| CheckType{타입?}

    CheckType -->|이미지| DownloadImg["이미지 다운로드<br/>최고 화질<br/>image_versions2.candidates[0]"]
    CheckType -->|동영상| DownloadVid["동영상 다운로드<br/>video_versions[0]"]

    DownloadImg --> SaveFile["파일 저장:<br/>nike_story_2025-01-18_001.jpg"]
    DownloadVid --> SaveFile

    SaveFile --> SaveJSON["메타데이터 JSON 저장:<br/>작성 시간, 뷰 수, 스티커 등"]
    SaveJSON --> Loop

    Loop -->|No 끝| End["완료!"]
    ErrorMsg --> End

    style Start fill:#9f9
    style End fill:#9f9
    style ErrorMsg fill:#f99
```

**핵심 코드:**
```python
def download_stories(self, userids: List[int], filename_target: str):
    """스토리 다운로드 - 로그인 필수"""

    # 1. 로그인 체크
    if not self.context.is_logged_in:
        raise LoginRequiredException("스토리는 로그인이 필요합니다")

    # 2. 각 사용자별 스토리 가져오기
    for userid in userids:
        # iPhone API 사용 (웹 API보다 데이터 많음)
        story_data = self.context.get_iphone_json(
            path=f'api/v1/feed/user/{userid}/story/',
            params={}
        )

        # 3. 스토리 없으면 건너뛰기
        if not story_data.get('reel'):
            self.context.log(f"{filename_target}에 스토리 없음")
            continue

        # 4. 각 스토리 아이템 처리
        for item in story_data['reel']['items']:
            story_item = StoryItem(self.context, item)

            # 5. 파일명 생성 (시간 포함)
            date_str = story_item.date.strftime('%Y-%m-%d_%H-%M-%S')
            filename = f"{filename_target}_story_{date_str}"

            # 6. 다운로드
            if story_item.is_video:
                # 동영상: 가장 높은 화질
                url = story_item.video_versions[0]['url']
                self.download_pic(filename, url, story_item.date)
            else:
                # 이미지: 가장 높은 해상도
                url = story_item.image_versions2['candidates'][0]['url']
                self.download_pic(filename, url, story_item.date)

            # 7. 메타데이터 저장
            metadata = {
                "date": story_item.date.isoformat(),
                "expiring_at": story_item.expiring_at.isoformat(),
                "view_count": story_item.view_count,
                "has_audio": story_item.has_audio if story_item.is_video else None,
                ...
            }
            self.save_metadata(metadata, filename)
```

### 5.4 옵션 활용 예시

#### 예시 1: 최근 20개만, 비디오 제외

```bash
instaloader --fast-update --no-videos --count 20 nike
```

```mermaid
flowchart LR
    Start[명령어 실행] --> Parse[옵션 파싱]

    Parse --> Opt1[--fast-update<br/>= 중복 다운 방지]
    Parse --> Opt2[--no-videos<br/>= 비디오 건너뛰기]
    Parse --> Opt3[--count 20<br/>= 20개만]

    Opt1 --> Loop[게시글 순회 시작]
    Opt2 --> Loop
    Opt3 --> Loop

    Loop --> Post1{게시글 1}
    Post1 -->|동영상| Skip1[건너뛰기]
    Post1 -->|이미지| Check1{파일 존재?}
    Check1 -->|Yes| Stop[중단!<br/>fast-update]
    Check1 -->|No| DL1[다운로드 count=1]

    Skip1 --> Post2
    DL1 --> Post2{게시글 2}
    Post2 -->|이미지| DL2[다운로드 count=2]
    DL2 --> Post3[...]

    Post3 --> Post20{게시글 20}
    Post20 --> DL20[다운로드 count=20]
    DL20 --> CheckCount{count == 20?}
    CheckCount -->|Yes| End[중단!]

    style Stop fill:#ff9
    style End fill:#9f9
```

**코드 로직:**
```python
def download_profile(self, profile, post_filter=None):
    """프로필 다운로드 (옵션 적용)"""

    downloaded_count = 0  # 다운로드한 개수
    max_count = 20        # --count 20
    no_videos = True      # --no-videos
    fast_update = True    # --fast-update

    for post in profile.get_posts():
        # 옵션 1: count 체크
        if downloaded_count >= max_count:
            print(f"{max_count}개 다운로드 완료, 중단")
            break

        # 옵션 2: 비디오 필터
        if no_videos and post.is_video:
            print(f"비디오 건너뛰기: {post.shortcode}")
            continue

        # 파일명 생성
        target_file = f"{post.date}_{post.shortcode}.jpg"

        # 옵션 3: fast_update (이미 있으면 중단)
        if fast_update and os.path.exists(target_file):
            print(f"이미 다운로드됨: {target_file}, 중단")
            break  # 이미 있으면 더 이상 다운로드 안 함

        # 다운로드 실행
        self.download_post(post, target="nike")
        downloaded_count += 1

        # 사람처럼 대기
        self.context.do_sleep()
```

💡 **fast-update의 동작 원리:**
- Instagram 게시글은 **최신순**으로 정렬됨
- 이미 다운로드한 파일을 만나면 = "여기까지는 전에 받았음"
- 더 이상 진행할 필요 없음 → 중단

#### 예시 2: 특정 기간만 다운로드

```bash
# 코드로 필터 적용 (CLI에는 직접 옵션 없음)
from datetime import datetime

loader = Instaloader()
profile = Profile.from_username(loader.context, "nike")

# 2024년 1월의 게시글만
for post in profile.get_posts():
    if post.date.year == 2024 and post.date.month == 1:
        loader.download_post(post, target="nike")
    elif post.date.year < 2024:
        # 2024년 이전이면 중단 (최신순이니까)
        break
```

---

## 6. 핵심 정리

### 6.1 Instaloader가 강력한 5가지 이유

#### 1. 공식 API 활용 → 안정성

```python
# Instaloader의 접근
requests.get('https://www.instagram.com/graphql/query?doc_id=...')
# ↑ Instagram이 공식적으로 제공하는 API

# Selenium의 접근
driver.get('https://www.instagram.com/nike/')
driver.find_element_by_xpath('//div[@class="...")
# ↑ HTML 구조가 바뀌면 작동 안 함
```

💡 **비유:**
- Instaloader = 정문으로 들어감 (공식 입구)
- Selenium = 창문으로 들어감 (비공식, 막힐 수 있음)

#### 2. 지능적 인증 관리 → 편리함

```python
# 첫 실행
instaloader --login myaccount
# 비밀번호 입력: ****
# 세션 저장됨!

# 다음 실행 (며칠 후)
instaloader --login myaccount
# 비밀번호 입력 없음!
# 자동으로 저장된 세션 사용

# 세션 유효 기간: 약 90일
# 만료되면 자동으로 재로그인 요청
```

#### 3. Rate Limiting 내장 → 차단 방지

```python
# 429 에러 자동 처리
try:
    data = get_json(...)
except TooManyRequestsException:
    print("요청 한도 초과, 10분 대기...")
    time.sleep(600)  # 자동 대기
    data = get_json(...)  # 자동 재시도

# 랜덤 대기로 봇 탐지 회피
time.sleep(random.uniform(1, 3))  # 1~3초 랜덤
```

#### 4. 고화질 미디어 → 품질

```python
# 익명: 표준 화질 (웹에서 보이는 것)
url = post.display_url
# 해상도: 1080x1080

# 로그인: 최고 화질 (iPhone API)
url = post._iphone_struct['image_versions2']['candidates'][0]['url']
# 해상도: 4000x4000 (원본!)
```

💡 **비교:**
- 익명 = YouTube 720p
- 로그인 = YouTube 4K

#### 5. 안정적 에러 핸들링 → 무인 운영 가능

```python
# 에러가 나도 계속 진행
for post in profile.get_posts():
    try:
        loader.download_post(post, target="nike")
    except Exception as e:
        # 에러 로그에 기록하고 다음으로
        loader.context.error(f"에러: {e}")
        continue  # 다음 게시글 계속 진행

# 프로그램 종료 시 모든 에러 요약 출력
loader.context.close()
# "3개의 에러 발생:
#  1. 게시글 ABC123: 네트워크 타임아웃
#  2. 게시글 DEF456: 파일 저장 실패
#  3. 게시글 GHI789: 동영상 URL 없음"
```

→ 1000개 게시글 다운로드 중 3개 실패해도 나머지 997개는 성공!

### 6.2 Selenium이 실패하는 3가지 이유

#### 1. 봇 탐지 시스템에 걸림

```javascript
// Instagram이 실행하는 봇 탐지 코드
if (navigator.webdriver) {
    // Selenium 감지!
    blockUser();
}

if (noMouseMovement() && fastClicking()) {
    // 비정상 행동 감지!
    showCaptcha();
}
```

#### 2. 너무 복잡하고 무거움

```
Selenium 실행 과정:
1. Chrome 브라우저 실행 (500MB 메모리)
2. JavaScript 엔진 초기화
3. HTML 다운로드
4. CSS 다운로드
5. 이미지 렌더링
6. JavaScript 실행
7. DOM 생성
8. BeautifulSoup로 파싱
9. 데이터 추출
→ 총 5~10초

Instaloader 실행 과정:
1. HTTP 요청
2. JSON 받기
3. 데이터 추출
→ 총 0.5초
```

#### 3. 유지보수 어려움

```html
<!-- Instagram이 HTML 구조 변경 -->
<!-- 변경 전 -->
<div class="post-container">
  <img src="..." class="post-image" />
</div>

<!-- 변경 후 -->
<div class="content-wrapper">
  <picture class="media-element">
    <img src="..." />
  </picture>
</div>
```

```python
# Selenium 코드 (깨짐)
img = driver.find_element_by_class_name("post-image")
# ❌ 에러: 클래스명이 바뀌어서 못 찾음!

# Instaloader (영향 없음)
data = get_json(..., doc_id="7950326061742207")
url = data['data']['user']['edge_owner_to_timeline_media']['edges'][0]['node']['display_url']
# ✅ JSON 구조는 안 바뀜!
```

### 6.3 언제 로그인이 필요한가?

| 기능 | 익명 (로그인 없음) | 로그인 |
|------|:-----------------:|:------:|
| **공개 프로필 게시글** | ✅ 가능 | ✅ 가능 (더 빠름) |
| **비공개 프로필** | ❌ 불가능 | ✅ 가능 (팔로우 시) |
| **스토리** | ❌ 불가능 | ✅ 가능 |
| **하이라이트** | ❌ 불가능 | ✅ 가능 |
| **댓글** | ❌ 불가능 | ✅ 가능 |
| **좋아요 목록** | ❌ 불가능 | ✅ 가능 |
| **고화질 미디어** | ❌ 표준 화질만 | ✅ 최고 화질 |
| **저장된 게시물** | ❌ 불가능 | ✅ 가능 (본인만) |
| **릴스** | ✅ 가능 | ✅ 가능 |
| **해시태그** | ✅ 가능 | ✅ 가능 |
| **위치 태그** | ✅ 가능 | ✅ 가능 |

💡 **간단 규칙:**
- **공개 정보** → 익명 OK
- **개인 정보, 일시적 콘텐츠** → 로그인 필요
- **최고 품질** → 로그인 필요

---

## 참고: 주요 Doc ID 목록

Instagram API의 "메뉴판"이라고 생각하면 됨:

```python
# 프로필 관련
ANONYMOUS_PROFILE_POSTS = "7950326061742207"  # 익명으로 게시글 보기
LOGGED_IN_PROFILE_POSTS = "7898261790222653"  # 로그인해서 게시글 보기
PROFILE_METADATA = "7c16654f22c819fb63d1183034a5162f"  # 프로필 정보

# 게시글 관련
POST_DETAIL = "8845758582119845"  # 게시글 상세 정보
REELS = "7845543455542541"        # 릴스 목록
SAVED_POSTS = "f883d95537fbcd400f466f63d42bd8a1"  # 저장한 게시물
TAGGED_POSTS = "e31a871f7301132ceaab56507a66bbb7"  # 태그된 게시물

# 인증 관련
LOGIN_TEST = "d6f4427fbe92d846298cf93df0b937d3"  # 로그인 상태 확인
```

💡 **이 숫자들은 Instagram이 API를 업데이트하면 바뀔 수 있음**
- Instaloader 개발자들이 주기적으로 업데이트함
- 만약 갑자기 안 되면 → Instaloader 최신 버전으로 업데이트 필요

---

## 마치며

이제 `instaloader nike` 같은 간단한 명령어 뒤에서 벌어지는 복잡한 과정을 완전히 이해했을 것임!

핵심 요약:
1. **Instagram의 공개 GraphQL API를 직접 호출**함 (브라우저 중간 단계 건너뛰기)
2. **브라우저처럼 위장**하여 차단 회피함
3. **로그인은 선택사항**이지만, 하면 더 많은 데이터와 고화질을 받을 수 있음
4. **Selenium보다 10배 빠르고 안정적**임
