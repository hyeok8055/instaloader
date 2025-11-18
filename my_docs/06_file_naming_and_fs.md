# 파일 네이밍 / 디렉토리 구조 / 파일시스템 대응 정리

## 학습 목표

이 문서는 Instaloader가 Instagram 콘텐츠를 파일시스템에 저장할 때 사용하는 **파일 네이밍 전략**, **디렉토리 구조 패턴**, 그리고 **크로스 플랫폼 호환성**을 위한 파일시스템 대응 메커니즘을 다룹니다.

학습을 마치면 다음을 이해할 수 있습니다:
- 패턴 기반 파일 경로 생성 메커니즘
- OS별 파일시스템 제약사항과 해결 방법
- `_PostPathFormatter`와 `sanitize_path`의 동작 원리
- 다양한 다운로드 시나리오별 디렉토리 구조

---

## 1. 파일시스템 설계 개요

### 1.1 설계 철학

Instaloader의 파일시스템 설계는 다음 원칙을 따릅니다:

1. **패턴 기반 유연성**: 사용자가 정의한 패턴으로 파일 경로를 동적 생성
2. **크로스 플랫폼 호환성**: Windows, Linux, macOS 모두에서 동작
3. **가독성 유지**: 특수 문자를 유사한 유니코드로 치환하여 인간이 읽을 수 있도록 유지
4. **충돌 방지**: OS 예약어와 금지 문자를 자동으로 회피

### 1.2 핵심 구성 요소

```python
# Instaloader 클래스 초기화 시 설정
L = Instaloader(
    dirname_pattern="{target}",           # 디렉토리 경로 패턴
    filename_pattern="{date_utc}_UTC",    # 파일명 패턴
    title_pattern="{date_utc}_UTC_{typename}",  # 제목 파일 패턴 (프로필 사진 등)
    sanitize_paths=False                   # 강제 Windows 경로 정리 여부
)
```

**패턴 변수 예시**:
- `{target}`: 대상 이름 (프로필명, 해시태그 등)
- `{profile}`: 프로필 소유자 이름
- `{date_utc}`: UTC 날짜/시간 (기본 포맷: `YYYY-MM-DD_HH-MM-SS`)
- `{date_local}`: 로컬 타임존 날짜/시간
- `{typename}`: Post 타입 (GraphImage, GraphVideo, GraphSidecar 등)
- `{shortcode}`: Instagram 게시물 고유 코드
- `{mediaid}`: 미디어 ID
- `{filename}`: 원본 URL 파일명
- `{owner_id}`: 소유자 ID
- `{owner_username}`: 소유자 사용자명

---

## 2. 파일명 패턴 예시 (10가지 이상)

### 2.1 기본 패턴들

#### 예시 1: 기본 설정 (날짜 기반)
```python
dirname_pattern = "{target}"
filename_pattern = "{date_utc}_UTC"

# 결과 (프로필: instagram, 2024-01-15 10:30:45 UTC 게시)
instagram/2024-01-15_10-30-45_UTC.jpg
instagram/2024-01-15_10-30-45_UTC.json.xz
```

#### 예시 2: 프로필별 연도/월 분류
```python
dirname_pattern = "{target}/{date_utc:%Y}/{date_utc:%m}"
filename_pattern = "{date_utc}_UTC"

# 결과
instagram/2024/01/2024-01-15_10-30-45_UTC.jpg
instagram/2024/01/2024-01-15_10-30-45_UTC.json.xz
```

#### 예시 3: 게시물 타입별 분류
```python
dirname_pattern = "{target}/{typename}"
filename_pattern = "{date_utc}_UTC_{shortcode}"

# 결과
instagram/GraphImage/2024-01-15_10-30-45_UTC_CaBcD123456.jpg
instagram/GraphVideo/2024-01-15_11-20-30_UTC_DeF789ghijk.mp4
instagram/GraphSidecar/2024-01-15_12-00-00_UTC_XyZ456abc78_1.jpg
instagram/GraphSidecar/2024-01-15_12-00-00_UTC_XyZ456abc78_2.jpg
```

#### 예시 4: Shortcode 중심 (Instagram URL 기반)
```python
dirname_pattern = "{target}"
filename_pattern = "{shortcode}"

# 결과 (간결하지만 날짜 정보 없음)
instagram/CaBcD123456.jpg
instagram/DeF789ghijk.mp4
```

#### 예시 5: 미디어 ID 기반 (고유성 보장)
```python
dirname_pattern = "{target}"
filename_pattern = "{mediaid}_{date_utc}"

# 결과
instagram/1234567890123456789_2024-01-15_10-30-45_UTC.jpg
```

### 2.2 고급 패턴들

#### 예시 6: 프로필과 타겟 분리 (여러 프로필 다운로드 시 유용)
```python
dirname_pattern = "{profile}/{target}"
filename_pattern = "{date_utc}_{typename}"

# 결과 (프로필 travel_blogger의 게시물을 hashtag #sunset으로 다운로드)
travel_blogger/sunset/2024-01-15_10-30-45_GraphImage.jpg

# 프로필 자체 게시물
travel_blogger/travel_blogger/2024-01-15_10-30-45_GraphImage.jpg
```

#### 예시 7: 원본 파일명 보존
```python
dirname_pattern = "{target}"
filename_pattern = "{filename}"

# 결과 (Instagram 원본 URL 파일명 사용)
instagram/123456789_987654321_n.jpg
instagram/234567890_876543210_n.mp4
```

#### 예시 8: 복합 정보 (날짜 + 타입 + Shortcode)
```python
dirname_pattern = "{target}"
filename_pattern = "{date_utc:%Y%m%d}_{typename}_{shortcode}"

# 결과
instagram/20240115_GraphImage_CaBcD123456.jpg
instagram/20240115_GraphVideo_DeF789ghijk.mp4
```

#### 예시 9: 소유자 정보 포함 (리포스트 추적 시 유용)
```python
dirname_pattern = "{target}"
filename_pattern = "{date_utc}_by_{owner_username}_{shortcode}"

# 결과 (다른 사용자의 게시물을 해시태그로 다운로드)
hashtag_nature/2024-01-15_10-30-45_by_photographer123_CaBcD123456.jpg
```

#### 예시 10: 로컬 시간대 사용
```python
dirname_pattern = "{target}"
filename_pattern = "{date_local:%Y-%m-%d_%H-%M-%S}_local"

# 결과 (한국 KST 기준, UTC+9)
instagram/2024-01-15_19-30-45_local.jpg
```

#### 예시 11: 계층적 연도/월/일 구조
```python
dirname_pattern = "{target}/{date_utc:%Y}/{date_utc:%m-%B}/{date_utc:%d}"
filename_pattern = "{date_utc:%H%M%S}_{shortcode}"

# 결과
instagram/2024/01-January/15/103045_CaBcD123456.jpg
instagram/2024/01-January/15/112030_DeF789ghijk.mp4
```

#### 예시 12: 플랫 구조 (모든 정보를 파일명에)
```python
dirname_pattern = "downloads"
filename_pattern = "{target}_{date_utc}_{typename}_{shortcode}"

# 결과 (모든 파일이 하나의 디렉토리에)
downloads/instagram_2024-01-15_10-30-45_GraphImage_CaBcD123456.jpg
downloads/travel_blogger_2024-01-15_11-20-30_GraphVideo_DeF789ghijk.mp4
downloads/hashtag_nature_2024-01-15_12-00-00_GraphSidecar_XyZ456abc78_1.jpg
```

### 2.3 특수 용도 패턴

#### 예시 13: 스토리 전용 구조
```python
# 스토리는 일반적으로 별도 디렉토리에 저장
dirname_pattern = "{target}/stories"
filename_pattern = "{date_utc}_UTC"

# 결과
instagram/stories/2024-01-15_10-30-45_UTC.jpg
instagram/stories/2024-01-15_10-31-20_UTC.mp4
```

#### 예시 14: 백업 중심 구조 (타임스탬프 기반)
```python
dirname_pattern = "backup/{date_utc:%Y-%m-%d}"
filename_pattern = "{target}_{date_utc:%H%M%S}_{mediaid}"

# 결과
backup/2024-01-15/instagram_103045_1234567890123456789.jpg
backup/2024-01-15/travel_blogger_112030_2345678901234567890.mp4
```

---

## 3. _PostPathFormatter 동작 원리

### 3.1 클래스 계층 구조

```
string.Formatter (Python 표준 라이브러리)
    ↓
_ArbitraryItemFormatter (속성 기반 포맷팅)
    ↓
_PostPathFormatter (파일시스템 안전성 추가)
```

### 3.2 _PostPathFormatter 핵심 메서드

```python
class _PostPathFormatter(_ArbitraryItemFormatter):
    RESERVED = {'CON', 'PRN', 'AUX', 'NUL',
                'COM1', 'COM2', 'COM3', 'COM4', 'COM5', 'COM6', 'COM7', 'COM8', 'COM9',
                'LPT1', 'LPT2', 'LPT3', 'LPT4', 'LPT5', 'LPT6', 'LPT7', 'LPT8', 'LPT9'}

    def __init__(self, item: Any, force_windows_path: bool = False):
        super().__init__(item)
        self.force_windows_path = force_windows_path

    def get_value(self, key, args, kwargs):
        # 1. 부모 클래스에서 속성 값 추출
        ret = super().get_value(key, args, kwargs)
        # 2. 문자열인 경우에만 sanitize
        if not isinstance(ret, str):
            return ret
        # 3. 파일시스템 안전 문자열로 변환
        return self.sanitize_path(ret, self.force_windows_path)
```

### 3.3 속성 값 추출 흐름

```
패턴 문자열: "{target}/{date_utc}_UTC"
     ↓
1. string.Formatter.parse() - 토큰 분리
   → literal: "", field: "target"
   → literal: "/", field: "date_utc"
   → literal: "_UTC", field: None
     ↓
2. _ArbitraryItemFormatter.get_value() - 속성 접근
   → item.target → "instagram"
   → item.date_utc → datetime(2024, 1, 15, 10, 30, 45)
     ↓
3. _PostPathFormatter.get_value() - Sanitize 적용
   → "instagram" → sanitize_path("instagram")
     ↓
4. format_field() - 타입별 포맷팅
   → datetime → "2024-01-15_10-30-45" (기본 포맷)
     ↓
결과: "instagram/2024-01-15_10-30-45_UTC"
```

---

## 4. sanitize_path 동작 메커니즘

### 4.1 동작 순서 Flowchart

```mermaid
flowchart TD
    A[입력: 원본 문자열 ret] --> B{문자열에 / 포함?}
    B -->|Yes| C[/ → ∕ U+2215 치환]
    B -->|No| D{문자열이 . 으로 시작?}
    C --> D
    D -->|Yes| E[첫 번째 . → ‥ U+2024 치환]
    D -->|No| F{Windows 경로?<br/>platform.system == Windows<br/>OR force_windows_path}
    E --> F

    F -->|Yes| G[Windows 금지 문자 치환<br/>: < > \" \\ | ? *]
    F -->|No| Z[반환: ret]

    G --> H[줄바꿈 문자를 공백으로 치환<br/>\n \r → space]
    H --> I[파일명과 확장자 분리<br/>os.path.splitext]
    I --> J{루트 파일명이<br/>Windows 예약어?}

    J -->|Yes CON, PRN, etc| K[루트 파일명에 _ 추가]
    J -->|No| L{확장자가 . 만?}

    K --> L
    L -->|Yes| M[확장자를 ‥ U+2024로 치환]
    L -->|No| N[루트 + 확장자 결합]

    M --> N
    N --> Z[반환: ret]

    style A fill:#e1f5ff
    style Z fill:#c8e6c9
    style F fill:#fff9c4
    style J fill:#fff9c4
```

### 4.2 상세 치환 규칙

#### 4.2.1 공통 규칙 (모든 OS)

| 원본 문자 | 치환 문자 | 유니코드 | 설명 |
|-----------|-----------|----------|------|
| `/` | `∕` | U+2215 | Division Slash - 디렉토리 구분자 충돌 방지 |
| 선두 `.` | `‥` | U+2024 | One Dot Leader - 숨김 파일 방지 |

```python
# 예시
"photo/2024" → "photo∕2024"
".hidden_file" → "‥hidden_file"
"normal.jpg" → "normal.jpg"  # 중간의 . 은 유지
```

#### 4.2.2 Windows 전용 규칙

| 원본 문자 | 치환 문자 | 유니코드 | 설명 |
|-----------|-----------|----------|------|
| `:` | `:` (전각) | U+FF1A | Fullwidth Colon |
| `<` | `<` (전각) | U+FE64 | Small Less-Than Sign |
| `>` | `>` (전각) | U+FE65 | Small Greater-Than Sign |
| `"` | `"` (전각) | U+FF02 | Fullwidth Quotation Mark |
| `\` | `\` (전각) | U+FE68 | Small Reverse Solidus |
| `|` | `|` (전각) | U+FF5C | Fullwidth Vertical Line |
| `?` | `?` (전각) | U+FE16 | Presentation Form Question Mark |
| `*` | `*` (전각) | U+FF0A | Fullwidth Asterisk |
| `\n`, `\r` | ` ` (공백) | U+0020 | 줄바꿈을 공백으로 |

#### 4.2.3 Windows 예약어 처리

```python
RESERVED = {
    'CON',   'PRN',   'AUX',   'NUL',
    'COM1',  'COM2',  'COM3',  'COM4',  'COM5',  'COM6',  'COM7',  'COM8',  'COM9',
    'LPT1',  'LPT2',  'LPT3',  'LPT4',  'LPT5',  'LPT6',  'LPT7',  'LPT8',  'LPT9'
}

# 예약어 확인 로직
root, ext = os.path.splitext(filename)
if root.upper() in RESERVED:
    root += '_'
filename = root + ext
```

**예약어 예시**:
- `CON.jpg` → `CON_.jpg`
- `PRN.txt` → `PRN_.txt`
- `COM1.dat` → `COM1_.dat`
- `con.jpg` → `con_.jpg` (대소문자 구분 없음)

### 4.3 Before/After 예시

#### 예시 1: 슬래시 포함 캡션
```
Before: "Summer 2024/25 Collection"
After:  "Summer 2024∕25 Collection"
```

#### 예시 2: 숨김 파일로 시작
```
Before: ".secret_photo"
After:  "‥secret_photo"
```

#### 예시 3: Windows 금지 문자 (시간 표기)
```
Before: "Meeting at 14:30"
After:  "Meeting at 14:30"  (Windows)
After:  "Meeting at 14:30"  (Linux/macOS - 변경 없음)
```

#### 예시 4: 비교 연산자
```
Before: "Price <$100>"
After:  "Price <$100>"  (Windows)
After:  "Price <$100>"  (Linux/macOS - 변경 없음)
```

#### 예시 5: 파일명 질문
```
Before: "Why?.jpg"
After:  "Why?.jpg"  (Windows)
After:  "Why?.jpg"  (Linux/macOS - 변경 없음)
```

#### 예시 6: 와일드카드 문자
```
Before: "IMG_*.jpg"
After:  "IMG_*.jpg"  (Windows)
After:  "IMG_*.jpg"  (Linux/macOS - 변경 없음)
```

#### 예시 7: Windows 예약어
```
Before: "CON.txt"
After:  "CON_.txt"  (Windows)
After:  "CON_.txt"  (Linux/macOS - sanitize_paths=True 시)
```

#### 예시 8: 복합 케이스
```
Before: ".config/data:2024/photo<1>.jpg"
After:  "‥config∕data:2024∕photo<1>.jpg"  (Windows)
After:  "‥config∕data:2024/photo<1>.jpg"  (Linux/macOS)
```

#### 예시 9: 줄바꿈이 포함된 캡션
```
Before: "Beautiful\nSunset\nView"
After:  "Beautiful Sunset View"  (Windows)
After:  "Beautiful\nSunset\nView"  (Linux/macOS - 변경 없음)
```

#### 예시 10: 확장자가 점만 있는 경우
```
Before: "filename."
After:  "filename‥"  (Windows)
After:  "filename."  (Linux/macOS - 변경 없음)
```

---

## 5. dirname_pattern과 filename_pattern 조합 시뮬레이션

### 5.1 시나리오 설정

다음 Instagram 데이터를 다운로드한다고 가정:
- **Profile**: `travel_blogger`
- **Post 1**: 2024-01-15 10:30:45 UTC, GraphImage, Shortcode: `CaBcD123456`
- **Post 2**: 2024-01-15 14:20:30 UTC, GraphVideo, Shortcode: `DeF789ghijk`
- **Post 3**: 2024-01-16 09:00:00 UTC, GraphSidecar (3장), Shortcode: `XyZ456abc78`
- **Hashtag**: `#travel` 다운로드
- **Story**: 2024-01-17 12:00:00 UTC

### 5.2 조합 1: 기본 설정

```python
dirname_pattern = "{target}"
filename_pattern = "{date_utc}_UTC"
```

**결과 디렉토리 구조**:
```
travel_blogger/
├── 2024-01-15_10-30-45_UTC.jpg          # Post 1
├── 2024-01-15_10-30-45_UTC.json.xz
├── 2024-01-15_14-20-30_UTC.mp4          # Post 2
├── 2024-01-15_14-20-30_UTC.jpg          # 비디오 썸네일
├── 2024-01-15_14-20-30_UTC.json.xz
├── 2024-01-16_09-00-00_UTC_1.jpg        # Post 3 - 이미지 1
├── 2024-01-16_09-00-00_UTC_2.jpg        # Post 3 - 이미지 2
├── 2024-01-16_09-00-00_UTC_3.jpg        # Post 3 - 이미지 3
└── 2024-01-16_09-00-00_UTC.json.xz

travel/                                  # Hashtag 다운로드
├── 2024-01-15_10-30-45_UTC.jpg
├── 2024-01-15_10-30-45_UTC.json.xz
└── ...
```

### 5.3 조합 2: 프로필/타겟 분리 + Shortcode

```python
dirname_pattern = "{profile}/{target}"
filename_pattern = "{shortcode}_{date_utc}"
```

**결과 디렉토리 구조**:
```
travel_blogger/travel_blogger/           # 프로필 자체 게시물
├── CaBcD123456_2024-01-15_10-30-45.jpg
├── CaBcD123456_2024-01-15_10-30-45.json.xz
├── DeF789ghijk_2024-01-15_14-20-30.mp4
├── DeF789ghijk_2024-01-15_14-20-30.jpg
├── DeF789ghijk_2024-01-15_14-20-30.json.xz
├── XyZ456abc78_2024-01-16_09-00-00_1.jpg
├── XyZ456abc78_2024-01-16_09-00-00_2.jpg
├── XyZ456abc78_2024-01-16_09-00-00_3.jpg
└── XyZ456abc78_2024-01-16_09-00-00.json.xz

travel_blogger/travel/                   # 프로필이 해시태그로 다운로드
├── CaBcD123456_2024-01-15_10-30-45.jpg
└── ...
```

### 5.4 조합 3: 연도/월 계층 구조

```python
dirname_pattern = "{target}/{date_utc:%Y}/{date_utc:%m}"
filename_pattern = "{date_utc:%d}_{date_utc:%H%M%S}_{typename}"
```

**결과 디렉토리 구조**:
```
travel_blogger/
├── 2024/
│   └── 01/
│       ├── 15_103045_GraphImage.jpg
│       ├── 15_103045_GraphImage.json.xz
│       ├── 15_142030_GraphVideo.mp4
│       ├── 15_142030_GraphVideo.jpg
│       ├── 15_142030_GraphVideo.json.xz
│       ├── 16_090000_GraphSidecar_1.jpg
│       ├── 16_090000_GraphSidecar_2.jpg
│       ├── 16_090000_GraphSidecar_3.jpg
│       └── 16_090000_GraphSidecar.json.xz
└── 2025/
    └── ...
```

### 5.5 조합 4: 타입별 디렉토리 분류

```python
dirname_pattern = "{target}/{typename}"
filename_pattern = "{date_utc}_{shortcode}"
```

**결과 디렉토리 구조**:
```
travel_blogger/
├── GraphImage/
│   ├── 2024-01-15_10-30-45_CaBcD123456.jpg
│   └── 2024-01-15_10-30-45_CaBcD123456.json.xz
├── GraphVideo/
│   ├── 2024-01-15_14-20-30_DeF789ghijk.mp4
│   ├── 2024-01-15_14-20-30_DeF789ghijk.jpg
│   └── 2024-01-15_14-20-30_DeF789ghijk.json.xz
└── GraphSidecar/
    ├── 2024-01-16_09-00-00_XyZ456abc78_1.jpg
    ├── 2024-01-16_09-00-00_XyZ456abc78_2.jpg
    ├── 2024-01-16_09-00-00_XyZ456abc78_3.jpg
    └── 2024-01-16_09-00-00_XyZ456abc78.json.xz
```

### 5.6 조합 5: 플랫 구조 (파일명에 모든 정보)

```python
dirname_pattern = "instagram_archive"
filename_pattern = "{target}_{date_utc:%Y%m%d_%H%M%S}_{typename}_{shortcode}"
```

**결과 디렉토리 구조**:
```
instagram_archive/
├── travel_blogger_20240115_103045_GraphImage_CaBcD123456.jpg
├── travel_blogger_20240115_103045_GraphImage_CaBcD123456.json.xz
├── travel_blogger_20240115_142030_GraphVideo_DeF789ghijk.mp4
├── travel_blogger_20240115_142030_GraphVideo_DeF789ghijk.jpg
├── travel_blogger_20240115_142030_GraphVideo_DeF789ghijk.json.xz
├── travel_blogger_20240116_090000_GraphSidecar_XyZ456abc78_1.jpg
├── travel_blogger_20240116_090000_GraphSidecar_XyZ456abc78_2.jpg
├── travel_blogger_20240116_090000_GraphSidecar_XyZ456abc78_3.jpg
├── travel_blogger_20240116_090000_GraphSidecar_XyZ456abc78.json.xz
├── travel_20240115_103045_GraphImage_CaBcD123456.jpg           # 해시태그
└── ...
```

---

## 6. 실제 파일시스템 구조 예시

### 6.1 전형적인 프로필 다운로드

#### 설정
```python
L = Instaloader(
    dirname_pattern="{target}",
    filename_pattern="{date_utc}_UTC",
    download_comments=True,
    download_geotags=True,
    save_metadata=True
)
L.download_profile("travel_blogger", profile_pic=True)
```

#### 결과 Tree
```
travel_blogger/
├── id                                           # 프로필 ID 파일
├── 2024-01-15_10-30-45_UTC_profile_pic.jpg      # 프로필 사진
├── 2024-01-15_10-30-45_UTC.jpg                  # 게시물 이미지
├── 2024-01-15_10-30-45_UTC.json.xz              # 메타데이터 (압축)
├── 2024-01-15_10-30-45_UTC.txt                  # 캡션
├── 2024-01-15_10-30-45_UTC_comments.json        # 댓글
├── 2024-01-15_10-30-45_UTC_location.txt         # 위치 정보
├── 2024-01-15_14-20-30_UTC.mp4                  # 비디오
├── 2024-01-15_14-20-30_UTC.jpg                  # 비디오 썸네일
├── 2024-01-15_14-20-30_UTC.json.xz
├── 2024-01-15_14-20-30_UTC.txt
└── 2024-01-15_14-20-30_UTC_comments.json
```

**파일 설명**:
- `id`: 프로필 고유 ID (프로필명 변경 감지 용도)
- `*_profile_pic.jpg`: 프로필 사진 (TitlePic)
- `*.jpg`, `*.mp4`: 실제 미디어 파일
- `*.json.xz`: 압축된 JSON 메타데이터 (Post 객체 전체)
- `*.txt`: 캡션 텍스트 (post_metadata_txt_pattern)
- `*_comments.json`: 댓글 목록 (download_comments=True)
- `*_location.txt`: 위치 정보 (download_geotags=True)

### 6.2 복잡한 계층 구조

#### 설정
```python
L = Instaloader(
    dirname_pattern="{profile}/{target}/{date_utc:%Y-%m}",
    filename_pattern="{date_utc:%d}_{typename}_{shortcode}",
    compress_json=False  # 비압축 JSON
)
```

#### 결과 Tree
```
travel_blogger/
├── travel_blogger/                              # 프로필 자체 게시물
│   ├── 2024-01/
│   │   ├── 15_GraphImage_CaBcD123456.jpg
│   │   ├── 15_GraphImage_CaBcD123456.json       # 비압축
│   │   ├── 15_GraphImage_CaBcD123456.txt
│   │   ├── 15_GraphVideo_DeF789ghijk.mp4
│   │   ├── 15_GraphVideo_DeF789ghijk.jpg
│   │   ├── 15_GraphVideo_DeF789ghijk.json
│   │   ├── 16_GraphSidecar_XyZ456abc78_1.jpg
│   │   ├── 16_GraphSidecar_XyZ456abc78_2.jpg
│   │   ├── 16_GraphSidecar_XyZ456abc78_3.jpg
│   │   └── 16_GraphSidecar_XyZ456abc78.json
│   └── 2024-02/
│       └── ...
└── travel/                                      # 해시태그로 다운로드
    ├── 2024-01/
    │   └── ...
    └── 2024-02/
        └── ...
```

### 6.3 스토리 아카이빙 구조

#### 설정
```python
# 스토리는 자동으로 {target}/stories/ 에 저장
L = Instaloader()
L.download_stories(userids=["travel_blogger"])
```

#### 결과 Tree
```
travel_blogger/
├── stories/
│   ├── 2024-01-17_12-00-00_UTC.jpg              # 스토리 이미지
│   ├── 2024-01-17_12-00-00_UTC.json.xz
│   ├── 2024-01-17_12-05-30_UTC.mp4              # 스토리 비디오
│   ├── 2024-01-17_12-05-30_UTC.json.xz
│   └── ...
└── ...
```

### 6.4 하이라이트 구조

#### 설정
```python
L = Instaloader()
L.download_highlights(profile)
```

#### 결과 Tree
```
travel_blogger/
├── highlights/
│   ├── Best of 2023/                            # 하이라이트 제목
│   │   ├── 2024-01-10_15-30-00_UTC_cover.jpg   # 커버 이미지
│   │   ├── 2023-12-31_23-59-00_UTC.jpg
│   │   ├── 2023-12-31_23-59-00_UTC.json.xz
│   │   └── ...
│   └── Travel Tips/
│       ├── 2024-01-05_10-00-00_UTC_cover.jpg
│       └── ...
└── ...
```

### 6.5 다중 프로필 백업 구조

#### 설정
```python
L = Instaloader(
    dirname_pattern="backup/{profile}",
    filename_pattern="{date_utc:%Y%m%d_%H%M%S}_{mediaid}"
)

profiles = ["travel_blogger", "food_lover", "tech_guru"]
for profile_name in profiles:
    L.download_profile(profile_name)
```

#### 결과 Tree
```
backup/
├── travel_blogger/
│   ├── id
│   ├── 20240115_103045_1234567890123456789.jpg
│   ├── 20240115_103045_1234567890123456789.json.xz
│   ├── 20240115_142030_2345678901234567890.mp4
│   └── ...
├── food_lover/
│   ├── id
│   ├── 20240114_090000_3456789012345678901.jpg
│   └── ...
└── tech_guru/
    ├── id
    ├── 20240113_180000_4567890123456789012.jpg
    └── ...
```

### 6.6 Sidecar (여러 이미지 게시물) 상세 구조

#### 설정
```python
# Post가 3개 이미지를 포함하는 Sidecar
L = Instaloader(
    dirname_pattern="{target}",
    filename_pattern="{date_utc}_UTC_{shortcode}"
)
```

#### 결과 Tree
```
travel_blogger/
├── 2024-01-16_09-00-00_UTC_XyZ456abc78_1.jpg    # 첫 번째 이미지
├── 2024-01-16_09-00-00_UTC_XyZ456abc78_2.jpg    # 두 번째 이미지
├── 2024-01-16_09-00-00_UTC_XyZ456abc78_3.jpg    # 세 번째 이미지
├── 2024-01-16_09-00-00_UTC_XyZ456abc78.json.xz  # 메타데이터 (1개만)
├── 2024-01-16_09-00-00_UTC_XyZ456abc78.txt      # 캡션 (1개만)
└── 2024-01-16_09-00-00_UTC_XyZ456abc78_comments.json  # 댓글 (1개만)
```

**특징**:
- 이미지는 `_1`, `_2`, `_3` 등 번호가 붙음
- 메타데이터, 캡션, 댓글은 게시물 단위로 1개만 저장

---

## 7. __prepare_filename 메커니즘

### 7.1 함수 역할

`__prepare_filename`은 다음 두 가지 역할을 수행합니다:

1. **`{filename}` 토큰 처리**: URL에서 실제 파일명 추출
2. **디렉토리 자동 생성**: 파일 저장 전 디렉토리 구조 생성

### 7.2 동작 흐름

```python
@staticmethod
def __prepare_filename(filename_template: str, url: Callable[[], str]) -> str:
    """
    filename_template: "{target}/{date_utc}_UTC_{filename}"
    url: lambda 함수, 필요할 때만 URL 가져옴
    """
    # 1. {filename} 토큰 확인
    if "{filename}" in filename_template:
        # 2. URL에서 파일명 추출
        actual_url = url()  # 예: "https://instagram.com/.../123456_abc.jpg?param=value"
        parsed = urlparse(actual_url)  # path: "/path/to/123456_abc.jpg"
        basename = os.path.basename(parsed.path)  # "123456_abc.jpg"
        filename_no_ext = os.path.splitext(basename)[0]  # "123456_abc"

        # 3. 토큰 치환
        filename = filename_template.replace("{filename}", filename_no_ext)
        # 결과: "travel_blogger/2024-01-15_10-30-45_UTC_123456_abc"
    else:
        filename = filename_template

    # 4. 디렉토리 자동 생성
    dirname = os.path.dirname(filename)  # "travel_blogger"
    os.makedirs(dirname, exist_ok=True)  # 재귀적으로 디렉토리 생성

    return filename
```

### 7.3 {filename} 토큰 사용 예시

#### 설정
```python
dirname_pattern = "{target}"
filename_pattern = "{date_utc}_{filename}"
```

#### 실제 URL
```
https://scontent-lax3-1.cdninstagram.com/v/t51.2885-15/123456789_987654321_n.jpg?_nc_cat=1&...
```

#### 처리 과정
```
1. URL 파싱
   → path: "/v/t51.2885-15/123456789_987654321_n.jpg"
   → basename: "123456789_987654321_n.jpg"
   → no_ext: "123456789_987654321_n"

2. 패턴 치환
   "2024-01-15_10-30-45_{filename}"
   → "2024-01-15_10-30-45_123456789_987654321_n"

3. 최종 경로
   "travel_blogger/2024-01-15_10-30-45_123456789_987654321_n.jpg"
```

### 7.4 디렉토리 생성 메커니즘

```python
# 예시: 복잡한 경로
filename = "backup/2024/01/travel_blogger/2024-01-15_10-30-45_UTC.jpg"

# os.makedirs의 동작
os.makedirs("backup/2024/01/travel_blogger", exist_ok=True)

# 생성되는 디렉토리들
backup/                    # 없으면 생성
backup/2024/               # 없으면 생성
backup/2024/01/            # 없으면 생성
backup/2024/01/travel_blogger/  # 없으면 생성
```

**`exist_ok=True` 효과**:
- 이미 존재하는 디렉토리는 무시 (에러 없음)
- 필요한 부분만 생성
- 동시 다운로드 시 경쟁 조건(race condition) 방지

---

## 8. 프로필 ID 파일 메커니즘

### 8.1 ID 파일의 필요성

Instagram에서 **프로필 이름은 변경 가능**하지만, **프로필 ID는 불변**입니다.

**문제 시나리오**:
```
Day 1: @travel_blogger (ID: 123456789) 다운로드
Day 30: 사용자가 이름을 @world_explorer로 변경
Day 31: 다시 @travel_blogger 다운로드 시도
       → 다른 사람이 @travel_blogger 이름을 가져갔을 수 있음!
```

**해결책**: 프로필 ID를 로컬에 저장하여 검증

### 8.2 ID 파일 경로 결정 로직

```python
def _get_id_filename(self, profile_name: str) -> str:
    # Case 1: dirname_pattern에 {profile} 또는 {target} 포함
    if (format_string_contains_key(self.dirname_pattern, 'profile') or
        format_string_contains_key(self.dirname_pattern, 'target')):
        # 프로필 전용 디렉토리가 생성됨
        return os.path.join(
            self.dirname_pattern.format(
                profile=profile_name.lower(),
                target=profile_name.lower()
            ),
            'id'  # 파일명은 단순히 "id"
        )

    # Case 2: dirname_pattern이 고정 디렉토리 (예: "downloads")
    else:
        # 여러 프로필이 같은 디렉토리에 섞임
        return os.path.join(
            self.dirname_pattern.format(),
            f'{profile_name.lower()}_id'  # 프로필명_id로 구분
        )
```

### 8.3 ID 파일 예시

#### Case 1: 프로필별 디렉토리
```python
dirname_pattern = "{target}"  # {target} 포함

# 프로필: travel_blogger
# ID 파일 경로: travel_blogger/id
```

**파일 내용** (`travel_blogger/id`):
```
123456789
```

#### Case 2: 공유 디렉토리
```python
dirname_pattern = "downloads"  # {target} 미포함

# 프로필: travel_blogger
# ID 파일 경로: downloads/travel_blogger_id

# 프로필: food_lover
# ID 파일 경로: downloads/food_lover_id
```

**파일 내용** (`downloads/travel_blogger_id`):
```
123456789
```

**파일 내용** (`downloads/food_lover_id`):
```
987654321
```

### 8.4 ID 검증 흐름

```python
# 1. 프로필 다운로드 시도
profile_name = "travel_blogger"

# 2. 로컬 ID 로드
local_id = load_profile_id(profile_name)  # 파일에서 읽음

# 3. Instagram에서 프로필 정보 가져오기
profile = Profile.from_username(context, profile_name)
remote_id = profile.userid

# 4. ID 비교
if local_id is not None and local_id != remote_id:
    # ID 불일치 → 프로필 이름이 변경되었거나 다른 사람이 가져감
    raise ProfileNotExistsException(f"Profile {profile_name} does not match the stored ID")

# 5. ID 저장 (최초 다운로드 또는 업데이트)
save_profile_id(profile_name, remote_id)
```

---

## 9. 크로스 플랫폼 고려사항

### 9.1 운영체제별 파일시스템 제약

| 특성 | Windows (NTFS) | Linux (ext4) | macOS (APFS/HFS+) |
|------|----------------|--------------|-------------------|
| **경로 구분자** | `\` (백슬래시) | `/` (슬래시) | `/` (슬래시) |
| **금지 문자** | `< > : " / \ | ? *` | `/` (일부 특수문자) | `/` (일부 특수문자) |
| **예약어** | CON, PRN, AUX, NUL, COM1-9, LPT1-9 | 없음 | 없음 |
| **대소문자** | 구분 안 함 (저장은 됨) | 구분함 | 기본 구분 안 함 |
| **경로 길이** | 260자 제한 (MAX_PATH) | 4096자 | 1024자 |
| **파일명 길이** | 255자 | 255 바이트 | 255 UTF-16 코드 단위 |
| **숨김 파일** | 속성 플래그 | `.`으로 시작 | `.`으로 시작 |

### 9.2 Instaloader의 크로스 플랫폼 전략

#### 전략 1: 공통 치환 (모든 OS)
```python
# 슬래시는 모든 OS에서 디렉토리 구분자이므로 항상 치환
ret = ret.replace('/', '\u2215')

# 선두 점은 Unix 계열에서 숨김 파일이므로 항상 치환
if ret.startswith('.'):
    ret = ret.replace('.', '\u2024', 1)
```

#### 전략 2: Windows 전용 치환
```python
if platform.system() == 'Windows':
    # Windows에서만 실행
    ret = ret.replace(':', '\uff1a').replace('<', '\ufe64')...
```

#### 전략 3: 강제 Windows 모드
```python
# sanitize_paths=True 설정 시
L = Instaloader(sanitize_paths=True)

# 모든 OS에서 Windows 규칙 적용
_PostPathFormatter(item, force_windows_path=True)
```

**용도**: Linux/macOS에서 다운로드하여 Windows로 이동할 파일 생성

### 9.3 플랫폼 이동 시나리오

#### 시나리오 1: Linux → Windows

**문제**:
```bash
# Linux에서 다운로드 (금지 문자 허용됨)
travel_blogger/Meeting at 14:30.jpg
travel_blogger/Price <$100>.jpg

# Windows로 복사 시도
# → 에러 발생! Windows가 : < > 를 파일명에 허용하지 않음
```

**해결책**:
```python
# Linux에서도 sanitize_paths=True 사용
L = Instaloader(sanitize_paths=True)

# 결과 (Windows 호환 파일명)
travel_blogger/Meeting at 14:30.jpg
travel_blogger/Price <$100>.jpg
```

#### 시나리오 2: macOS → Windows

**문제**:
```bash
# macOS에서 다운로드
travel_blogger/Document:.pdf        # macOS는 : 허용
travel_blogger/CON.jpg              # macOS는 CON 예약어 없음

# Windows로 복사
# → 에러 발생!
```

**해결책**:
```python
L = Instaloader(sanitize_paths=True)

# 결과
travel_blogger/Document:.pdf       # : → 전각 콜론
travel_blogger/CON_.jpg            # CON → CON_
```

#### 시나리오 3: Windows에서 시작 (자동 처리)

```python
# Windows에서 실행 시 자동으로 sanitize 적용
L = Instaloader()  # sanitize_paths=False여도

# platform.system() == 'Windows' 자동 감지
# 모든 파일명이 Windows 호환으로 생성됨
```

### 9.4 경로 길이 제한 대응

#### Windows MAX_PATH 문제

**문제**:
```python
dirname_pattern = "{profile}/{target}/{date_utc:%Y}/{date_utc:%m}"
filename_pattern = "{date_utc:%d}_{typename}_{shortcode}_{owner_username}_{mediaid}"

# 긴 경로 생성
# C:\Users\VeryLongUsername\Documents\Instagram\travel_blogger_with_long_name\
# travel_destination_with_long_name\2024\01\15_GraphSidecar_XyZ456abc78_
# another_long_username_1234567890123456789.jpg
#
# → 260자 초과 → 에러!
```

**해결책**:
```python
# 1. 짧은 패턴 사용
dirname_pattern = "{target}"
filename_pattern = "{date_utc}"

# 2. 축약 사용
dirname_pattern = "{target}"
filename_pattern = "{date_utc:%y%m%d}_{shortcode}"  # 연도 2자리

# 3. Windows 10 1607 이상: 긴 경로 지원 활성화
# 레지스트리: HKLM\SYSTEM\CurrentControlSet\Control\FileSystem\LongPathsEnabled = 1
# 또는 매니페스트에서 longPathAware 설정
```

### 9.5 문자 인코딩 고려사항

#### UTF-8 지원

```python
# Python 3의 기본 인코딩은 UTF-8
# Instaloader는 모든 파일명을 UTF-8로 처리

# 다양한 언어 지원
"여행_블로거/2024-01-15_10-30-45_UTC.jpg"          # 한국어
"путешествия/2024-01-15_10-30-45_UTC.jpg"         # 러시아어
"旅行者/2024-01-15_10-30-45_UTC.jpg"               # 중국어
"السفر/2024-01-15_10-30-45_UTC.jpg"                # 아랍어
"🌍_travel/2024-01-15_10-30-45_UTC.jpg"            # 이모지
```

#### 파일시스템 인코딩 제한

| OS | 파일시스템 | 인코딩 | UTF-8 지원 |
|----|-----------|--------|-----------|
| Windows | NTFS | UTF-16 | 완전 지원 |
| Linux | ext4 | UTF-8 | 완전 지원 |
| macOS | APFS | UTF-8 (NFD) | 완전 지원 (정규화 차이 주의) |

**macOS NFD 정규화 이슈**:
```python
# 입력: "café" (NFC: é = U+00E9)
# macOS가 저장: "café" (NFD: e + ́  = U+0065 + U+0301)

# 문제: 다른 OS로 이동 시 파일명 비교 실패 가능
# Instaloader는 이를 직접 처리하지 않음 (OS 레벨에서 처리)
```

### 9.6 네트워크 파일시스템 고려사항

#### SMB/CIFS (Windows 공유)

```python
# SMB 마운트에서 다운로드
dirname_pattern = "//server/share/instagram/{target}"

# 주의사항:
# 1. 네트워크 지연 → 파일 쓰기 느림
# 2. 일부 특수 문자 추가 제약
# 3. 대소문자 구분 설정 확인 필요
```

#### NFS (Unix 공유)

```python
# NFS 마운트에서 다운로드
dirname_pattern = "/mnt/nfs/instagram/{target}"

# 주의사항:
# 1. 파일 잠금(locking) 동작 확인
# 2. 권한 설정 (UID/GID 매핑)
# 3. 대소문자 구분 유지
```

### 9.7 크로스 플랫폼 권장 패턴

#### 안전한 패턴 (모든 OS에서 동작)

```python
# 1. ASCII 문자 + 숫자만 사용
dirname_pattern = "{target}"
filename_pattern = "{date_utc:%Y%m%d_%H%M%S}_{mediaid}"

# 2. 짧은 경로
dirname_pattern = "{target}"
filename_pattern = "{date_utc}"

# 3. Windows 호환 모드
L = Instaloader(sanitize_paths=True)
```

#### 위험한 패턴 (플랫폼 특정 동작)

```python
# 1. 특수 문자 포함 (sanitize_paths=False + Linux)
filename_pattern = "{caption}"  # 캡션에 금지 문자 있을 수 있음

# 2. 긴 경로 (Windows에서 문제)
dirname_pattern = "{profile}/{target}/{date_utc:%Y}/{date_utc:%B}/{date_utc:%A}"
filename_pattern = "{date_utc:%d}_{typename}_{shortcode}_{owner_username}"

# 3. 대소문자만 다른 파일 (Windows에서 충돌)
# Example.jpg와 example.jpg를 구별할 수 없음
```

---

## 10. 파일 유형별 저장 규칙

### 10.1 저장되는 파일 종류 전체 목록

| 파일 유형 | 확장자 | 생성 조건 | 내용 |
|----------|--------|----------|------|
| **미디어 파일** | | | |
| 이미지 | `.jpg` | `download_pictures=True` | JPEG 이미지 |
| 비디오 | `.mp4` | `download_videos=True` | MP4 비디오 |
| 비디오 썸네일 | `.jpg` | `download_video_thumbnails=True` | 비디오 미리보기 이미지 |
| **메타데이터 파일** | | | |
| 압축 메타데이터 | `.json.xz` | `save_metadata=True`, `compress_json=True` | LZMA 압축 JSON |
| 비압축 메타데이터 | `.json` | `save_metadata=True`, `compress_json=False` | 일반 JSON |
| 캡션 | `.txt` | `post_metadata_txt_pattern` 설정 | 게시물 캡션 텍스트 |
| 댓글 | `_comments.json` | `download_comments=True` | 댓글 목록 JSON |
| 위치 정보 | `_location.txt` | `download_geotags=True` | 위치 정보 텍스트 |
| **프로필 파일** | | | |
| 프로필 ID | `id` 또는 `{profile}_id` | 항상 | 프로필 고유 ID |
| 프로필 사진 | `_profile_pic.jpg` | `download_profilepic=True` | 프로필 이미지 |
| **하이라이트 파일** | | | |
| 하이라이트 커버 | `_cover.jpg` | 하이라이트 다운로드 시 | 커버 이미지 |

### 10.2 파일 명명 규칙

#### 기본 파일
```
{filename_pattern}.{extension}

예:
2024-01-15_10-30-45_UTC.jpg
2024-01-15_10-30-45_UTC.json.xz
```

#### Sidecar (여러 미디어)
```
{filename_pattern}_{index}.{extension}

예:
2024-01-15_10-30-45_UTC_1.jpg
2024-01-15_10-30-45_UTC_2.jpg
2024-01-15_10-30-45_UTC_3.jpg
2024-01-15_10-30-45_UTC.json.xz  (인덱스 없음)
```

#### 부가 파일
```
{filename_pattern}_{suffix}.{extension}

예:
2024-01-15_10-30-45_UTC_comments.json
2024-01-15_10-30-45_UTC_location.txt
2024-01-15_10-30-45_UTC_profile_pic.jpg
```

### 10.3 메타데이터 JSON 구조

#### 압축 파일 (.json.xz)
```bash
# 압축 해제
xz -d 2024-01-15_10-30-45_UTC.json.xz

# 결과
2024-01-15_10-30-45_UTC.json
```

#### JSON 내용 예시
```json
{
  "node": {
    "__typename": "GraphImage",
    "id": "1234567890123456789",
    "shortcode": "CaBcD123456",
    "dimensions": {"height": 1080, "width": 1080},
    "display_url": "https://instagram.com/.../image.jpg",
    "edge_media_to_caption": {
      "edges": [
        {"node": {"text": "Beautiful sunset 🌅"}}
      ]
    },
    "taken_at_timestamp": 1705315845,
    "edge_media_to_comment": {"count": 42},
    "edge_liked_by": {"count": 1337},
    "location": {
      "id": "123456",
      "name": "Santorini, Greece",
      "slug": "santorini-greece"
    },
    "owner": {
      "id": "987654321",
      "username": "travel_blogger"
    }
  }
}
```

---

## 11. 고급 활용 시나리오

### 11.1 타임스탬프 기반 정렬

```python
# 오래된 것부터 새것 순으로 브라우징하기 쉬운 구조
dirname_pattern = "{target}"
filename_pattern = "{date_utc}"

# 파일 시스템에서 정렬 시 시간순 자동 정렬
# 2024-01-15_10-30-45_UTC.jpg
# 2024-01-15_14-20-30_UTC.jpg
# 2024-01-16_09-00-00_UTC.jpg
```

### 11.2 백업 스크립트용 구조

```python
from datetime import datetime

# 백업 실행 날짜별 디렉토리
backup_date = datetime.now().strftime("%Y%m%d")
dirname_pattern = f"backup/{backup_date}/{{target}}"
filename_pattern = "{date_utc}_{mediaid}"

# 결과
# backup/20240120/travel_blogger/2024-01-15_10-30-45_1234567890.jpg
# backup/20240121/travel_blogger/2024-01-16_09-00-00_2345678901.jpg
```

### 11.3 증분 백업 (이미 존재하는 파일 스킵)

```python
# Instaloader는 기본적으로 이미 존재하는 파일을 스킵
L = Instaloader()

# 동일 경로에 반복 실행 시
L.download_profile("travel_blogger")
# → 이미 다운로드된 파일은 "already exists" 메시지 출력 후 스킵
```

### 11.4 아카이브 분리 (타입별)

```python
# 이미지와 비디오를 별도 디렉토리에
dirname_pattern = "{target}/{typename}"
filename_pattern = "{date_utc}"

# 결과
# travel_blogger/GraphImage/*.jpg
# travel_blogger/GraphVideo/*.mp4
# travel_blogger/GraphSidecar/*.jpg
```

### 11.5 데이터베이스 연동용 구조

```python
# 파일명에 모든 메타데이터 포함 (DB 인덱싱 용이)
dirname_pattern = "archive"
filename_pattern = "{owner_id}_{mediaid}_{date_utc:%Y%m%d%H%M%S}_{typename}_{shortcode}"

# DB 테이블 예시
"""
CREATE TABLE instagram_posts (
    owner_id BIGINT,
    media_id BIGINT PRIMARY KEY,
    timestamp DATETIME,
    typename VARCHAR(50),
    shortcode VARCHAR(20),
    file_path VARCHAR(500)
);
"""

# 파일명을 파싱하여 DB에 삽입 가능
# 987654321_1234567890123456789_20240115103045_GraphImage_CaBcD123456.jpg
```

---

## 12. 문제 해결 (Troubleshooting)

### 12.1 일반적인 문제

#### 문제 1: 파일명에 잘못된 문자
```
FileNotFoundError: [Errno 2] No such file or directory: 'user/Meeting at 14:30.jpg'
```

**원인**: Windows에서 `:` 문자 사용 불가

**해결**:
```python
L = Instaloader(sanitize_paths=True)
```

#### 문제 2: 경로가 너무 김
```
OSError: [Errno 63] File name too long
```

**해결**:
```python
# 짧은 패턴 사용
dirname_pattern = "{target}"
filename_pattern = "{date_utc:%y%m%d}_{mediaid}"
```

#### 문제 3: 대소문자 충돌
```
# Windows에서
user/Photo.jpg  # 먼저 생성됨
user/photo.jpg  # 덮어씌워짐!
```

**해결**:
```python
# 패턴에서 대소문자 통일
filename_pattern = "{date_utc}_{shortcode}".lower()
```

### 12.2 디버깅 팁

#### 파일 경로 확인
```python
L = Instaloader()
post = Post.from_shortcode(L.context, "CaBcD123456")

# 실제 생성될 경로 확인
dirname = _PostPathFormatter(post, L.sanitize_paths).format(
    L.dirname_pattern, target="travel_blogger"
)
filename = L.format_filename(post, target="travel_blogger")
full_path = os.path.join(dirname, filename)
print(f"File will be saved to: {full_path}")
```

#### Sanitize 테스트
```python
from instaloader.instaloader import _PostPathFormatter

test_strings = [
    "normal_filename",
    "file/with/slash",
    ".hidden_file",
    "Meeting at 14:30",
    "CON.jpg",
    "Price <$100>"
]

for s in test_strings:
    sanitized = _PostPathFormatter.sanitize_path(s, force_windows_path=True)
    print(f"{s:30} → {sanitized}")
```

---

## 13. 요약 및 베스트 프랙티스

### 13.1 핵심 원칙

1. **`dirname_pattern`**: 디렉토리 구조 결정
2. **`filename_pattern`**: 파일명 결정
3. **`sanitize_path`**: OS 제약 자동 회피
4. **`_PostPathFormatter`**: 패턴 → 실제 경로 변환

### 13.2 권장 설정

#### 일반 사용자 (간단함)
```python
L = Instaloader(
    dirname_pattern="{target}",
    filename_pattern="{date_utc}_UTC"
)
```

#### 파워 유저 (정교한 분류)
```python
L = Instaloader(
    dirname_pattern="{profile}/{target}/{date_utc:%Y/%m}",
    filename_pattern="{date_utc:%d}_{typename}_{shortcode}",
    sanitize_paths=True  # 크로스 플랫폼
)
```

#### 아카이브/백업 (안정성)
```python
L = Instaloader(
    dirname_pattern="backup/{profile}",
    filename_pattern="{date_utc:%Y%m%d_%H%M%S}_{mediaid}",
    compress_json=False,  # 호환성
    sanitize_paths=True
)
```

### 13.3 체크리스트

- [ ] 패턴에 필요한 변수가 모두 포함되었는가?
- [ ] 경로 길이가 260자(Windows) 이하인가?
- [ ] 크로스 플랫폼 사용 시 `sanitize_paths=True`를 설정했는가?
- [ ] `{filename}` 토큰 사용 시 URL 접근이 가능한가?
- [ ] 여러 프로필 다운로드 시 디렉토리가 구분되는가?
- [ ] ID 파일 경로가 올바르게 설정되었는가?

### 13.4 학습 자료

- **Instaloader 공식 문서**: https://instaloader.github.io/
- **코드 참고**: `/workspace/instaloader/instaloader/instaloader.py`
  - `_PostPathFormatter` 클래스 (138-172줄)
  - `sanitize_path` 메서드 (154-171줄)
  - `format_filename` 메서드 (681-686줄)
  - `__prepare_filename` 메서드 (669-679줄)

---

## 부록 A: 전체 패턴 변수 레퍼런스

### Post 객체 사용 가능 변수

| 변수 | 타입 | 설명 | 예시 |
|------|------|------|------|
| `{target}` | str | 다운로드 대상 이름 | `travel_blogger`, `travel` |
| `{profile}` | str | 프로필 소유자 이름 | `travel_blogger` |
| `{shortcode}` | str | Instagram shortcode | `CaBcD123456` |
| `{mediaid}` | int | 미디어 ID | `1234567890123456789` |
| `{typename}` | str | Post 타입 | `GraphImage`, `GraphVideo`, `GraphSidecar` |
| `{date_utc}` | datetime | UTC 날짜/시간 | `2024-01-15 10:30:45` |
| `{date_local}` | datetime | 로컬 날짜/시간 | `2024-01-15 19:30:45` (KST) |
| `{date}` | datetime | `date_local`과 동일 | |
| `{owner_id}` | int | 소유자 ID | `987654321` |
| `{owner_username}` | str | 소유자 사용자명 | `travel_blogger` |
| `{filename}` | str | 원본 URL 파일명 (특수 토큰) | `123456_abc` |
| `{caption}` | str | 캡션 텍스트 | `Beautiful sunset` |
| `{caption_hashtags}` | str | 해시태그만 | `#travel #sunset` |
| `{caption_mentions}` | str | 멘션만 | `@friend` |
| `{tagged}` | list | 태그된 사용자 목록 | |
| `{location}` | str | 위치 이름 | `Santorini, Greece` |

### datetime 포맷 지정자

```python
# 기본 포맷 (포맷 지정 없을 시)
{date_utc} → 2024-01-15_10-30-45

# 사용자 정의 포맷
{date_utc:%Y} → 2024
{date_utc:%m} → 01
{date_utc:%d} → 15
{date_utc:%H} → 10
{date_utc:%M} → 30
{date_utc:%S} → 45
{date_utc:%Y%m%d} → 20240115
{date_utc:%Y-%m-%d} → 2024-01-15
{date_utc:%B} → January
{date_utc:%A} → Monday
```

---

이 문서는 Instaloader의 파일시스템 설계를 종합적으로 다루며, 실제 사용 시나리오와 크로스 플랫폼 호환성을 고려한 대학 수준의 강의 자료입니다.
