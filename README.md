# 📋 게시판 데이터 크롤링 가이드

이 문서는 AnyFive 그룹웨어(office.anyfive.com)의 게시판 데이터를 크롤링하여 cmds 형식으로 변환하는 전체 과정을 설명합니다. 

---

## 📑 목차

1. [개요](#1-개요)
2. [사전 준비사항](#2-사전-준비사항)
3. [Python 설치](#3-python-설치)
4. [필수 프로그램 설치](#4-필수-프로그램-설치)
5. [MariaDB 설치 및 설정](#5-mariadb-설치-및-설정)
6. [Python 라이브러리 설치](#6-python-라이브러리-설치)
7. [프로젝트 폴더 구성](#7-프로젝트-폴더-구성)
8. [스크립트 상세 설명](#8-스크립트-상세-설명)
9. [단계별 실행 가이드](#9-단계별-실행-가이드)
10. [문제 해결 (Troubleshooting)](#10-문제-해결-troubleshooting)
11. [자주 묻는 질문 (FAQ)](#11-자주-묻는-질문-faq)

---

## 1. 개요

### 1.1 이 프로젝트가 하는 일

이 프로젝트는 다음 작업을 수행합니다:

1. **크롤링**: AnyFive 그룹웨어에서 게시판 데이터(게시글, 댓글, 첨부파일) 수집
2. **저장**: 수집한 데이터를 MariaDB 데이터베이스에 저장
3. **이미지 처리**: 게시글 본문의 이미지 경로를 마이그레이션 가능한 형태로 변환
4. **첨부파일 처리**: 첨부파일 경로 변환 및 파일 복사
5. **내보내기**: 다우오피스에서 읽을 수 있는 `.cmds` 파일 생성

### 1.2 전체 작업 흐름

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          전체 마이그레이션 흐름                           │
└─────────────────────────────────────────────────────────────────────────┘

[1단계] 크롤링
    │
    ▼
┌──────────────────┐     ┌──────────────────┐
│  AnyFive 게시판   │ ──▶ │   MariaDB 저장    │
│  (웹 크롤링)      │     │  (board_posts)   │
└──────────────────┘     └──────────────────┘
                                  │
                                  ▼
[2단계] 이미지 처리 ──────────────────────────────────────────┐
    │                                                        │
    ▼                                                        ▼
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  원본 이미지 폴더  │ ──▶ │   이미지 복사     │ ──▶ │  DB content 갱신  │
└──────────────────┘     │  (확장자 추가)    │     └──────────────────┘
                         └──────────────────┘
                                  │
                                  ▼
[3단계] 첨부파일 처리 ────────────────────────────────────────┐
    │                                                        │
    ▼                                                        ▼
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  원본 첨부파일     │ ──▶ │   파일 복사       │ ──▶ │  DB attaches 갱신 │
└──────────────────┘     └──────────────────┘     └──────────────────┘
                                  │
                                  ▼
[4단계] targetBoardId 설정 ───────────────────────────────────┐
    │                                                        │
    ▼                                                        ▼
┌──────────────────┐                              ┌──────────────────┐
│  다우오피스 게시판  │ ──────────────────────────▶ │  DB 일괄 업데이트  │
│  ID 확인          │                              └──────────────────┘
└──────────────────┘
                                  │
                                  ▼
[5단계] cmds 파일 생성 ───────────────────────────────────────┐
    │                                                        │
    ▼                                                        ▼
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  posts_xxx.cmds  │     │ comments_xxx.cmds │     │  다우오피스 import │
└──────────────────┘     └──────────────────┘     └──────────────────┘
```

### 1.3 스크립트 목록

| 순서 | 파일명 | 역할 |
|------|--------|------|
| 1 | `any_board_crawling_게시판_251113.ipynb` | 게시판 크롤링 및 DB 저장 |
| 2 | `2_board_image_path_convert_DB.ipynb` | 이미지 경로 변환 |
| 3 | `3_board_path_convert_DB.ipynb` | 첨부파일 경로 변환 |
| 4 | `4_targetBoardId_insert_DB.ipynb` | targetBoardId 일괄 설정 |
| 5 | `5_generate_cmds_from_db.ipynb` | 게시글 cmds 파일 생성 |
| 6 | `6_generate_comment_cmds_from_db.ipynb` | 댓글 cmds 파일 생성 |

---

## 2. 사전 준비사항

### 2.1 필요한 것들

| 항목 | 설명 | 필수 여부 |
|------|------|----------|
| Windows PC | Windows 10 이상 권장 | ✅ 필수 |
| AnyFive 계정 | 관리자 권한 계정 권장 | ✅ 필수 |
| 원본 파일 | 서버에서 백업한 이미지/첨부파일 폴더 | ✅ 필수 |
| 사용자 목록 CSV | 사용자 매핑용 (`users_filtered.csv`) | ✅ 필수 |

---

## 3. Python 설치

### 3.1 Python 다운로드

1. **공식 웹사이트 접속**
   - 브라우저에서 https://www.python.org/downloads/ 접속

2. **Python 3.11 이상 버전 다운로드**
   - "Download Python 3.11.x" 버튼 클릭
   - 또는 특정 버전 선택: https://www.python.org/downloads/release/python-3119/

   ![Python 다운로드 페이지](https://www.python.org/static/img/python-logo.png)

### 3.2 Python 설치

1. **설치 파일 실행**
   - 다운로드된 `python-3.11.x-amd64.exe` 파일 실행

2. **⚠️ 중요: "Add Python to PATH" 체크**
   
   ```
   ┌─────────────────────────────────────────────────────────┐
   │  Install Python 3.11.x (64-bit)                        │
   │                                                         │
   │  ☑ Install launcher for all users (recommended)        │
   │  ☑ Add Python 3.11 to PATH  ◀── 반드시 체크!           │
   │                                                         │
   │  [Install Now]                                          │
   │  [Customize installation]                               │
   └─────────────────────────────────────────────────────────┘
   ```

3. **"Install Now" 클릭**

4. **설치 완료 확인**
   - "Setup was successful" 메시지 확인
   - "Disable path length limit" 버튼이 보이면 클릭 (권장)

### 3.3 설치 확인

1. **명령 프롬프트 열기**
   - `Windows 키` + `R` 누르기
   - `cmd` 입력 후 Enter

2. **Python 버전 확인**
   ```cmd
   python --version
   ```
   
   결과 예시:
   ```
   Python 3.11.9
   ```

3. **pip 버전 확인**
   ```cmd
   pip --version
   ```
   
   결과 예시:
   ```
   pip 24.0 from C:\Users\...\Python311\Lib\site-packages\pip (python 3.11)
   ```

> ❌ **오류 발생 시**: "'python'은(는) 내부 또는 외부 명령, 실행할 수 있는 프로그램..."
> 
> 해결방법:
> 1. Python 재설치 (이번엔 "Add to PATH" 체크 확인)
> 2. 또는 시스템 환경변수에 Python 경로 수동 추가

---

## 4. 필수 프로그램 설치

### 4.1 Chrome 브라우저

크롤링에 Chrome 브라우저가 필요합니다.

1. https://www.google.com/chrome/ 접속
2. "Chrome 다운로드" 클릭
3. 설치 파일 실행하여 설치 완료

### 4.2 Chrome 버전 확인

1. Chrome 브라우저 실행
2. 주소창에 `chrome://version` 입력
3. 버전 번호 확인 (예: 120.0.6099.130)

> 💡 **참고**: ChromeDriver는 Selenium이 자동으로 관리하므로 별도 설치 불필요

---

## 5. MariaDB 설치 및 설정

### 5.1 MariaDB 다운로드

1. https://mariadb.org/download/ 접속
2. 다음 옵션 선택:
   - **Version**: 10.11.x (LTS) 권장
   - **OS**: Windows
   - **Package Type**: MSI Package

3. 다운로드 클릭

### 5.2 MariaDB 설치

1. **설치 파일 실행**
   - `mariadb-10.11.x-winx64.msi` 실행

2. **설치 과정**

   ```
   ┌─────────────────────────────────────────────────────────┐
   │  MariaDB 10.11 Setup                                   │
   │                                                         │
   │  [1] License Agreement                                  │
   │      ☑ I accept the terms                              │
   │                                                         │
   │  [2] Custom Setup                                       │
   │      기본 설정 그대로 사용                               │
   │                                                         │
   │  [3] Database Configuration                             │
   │      - root password: 1234  ◀── 기억해두세요!          │
   │      ☑ Use UTF8 as default server's character set      │
   │      ☑ Enable access from remote machines for 'root'   │
   │                                                         │
   │  [4] Install                                            │
   └─────────────────────────────────────────────────────────┘
   ```

3. **중요 설정값**
   - **root 비밀번호**: `1234` (스크립트에서 사용하는 기본값)
   - **포트**: `3306` (기본값)
   - **UTF8 사용**: 체크 권장

### 5.3 MariaDB 서비스 확인

1. **서비스 실행 확인**
   - `Windows 키` + `R` → `services.msc` 입력 → Enter
   - "MariaDB" 서비스가 "실행 중" 상태인지 확인

2. **명령줄에서 접속 테스트**
   ```cmd
   mysql -u root -p1234
   ```
   
   성공 시:
   ```
   Welcome to the MariaDB monitor. Commands end with ; or \g.
   MariaDB [(none)]>
   ```

### 5.4 데이터베이스 생성

MariaDB에 접속한 상태에서:

```sql
-- 데이터베이스 생성
CREATE DATABASE crawling CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 확인
SHOW DATABASES;

-- 종료
exit;
```

---

## 6. Python 라이브러리 설치

### 6.1 필수 라이브러리 목록

| 라이브러리 | 용도 |
|-----------|------|
| selenium | 웹 브라우저 자동화 (크롤링) |
| beautifulsoup4 | HTML 파싱 |
| lxml | 고속 HTML/XML 파서 |
| pandas | 데이터 처리 및 Excel 저장 |
| openpyxl | Excel 파일 생성 |
| pymysql | MariaDB 연결 |
| sqlalchemy | ORM 및 DB 연결 관리 |

### 6.2 설치 명령어
아래 스크립트 실행
```
pip install selenium beautifulsoup4 lxml pandas openpyxl pymysql sqlalchemy
```

### 6.4 라이브러리 업그레이드 (선택)

최신 버전으로 업그레이드하려면:

```
pip install --upgrade selenium beautifulsoup4 lxml pandas openpyxl pymysql sqlalchemy
```
---

## 8. 스크립트 상세 설명

### 8.1 any_board_crawling_게시판_251113.ipynb (게시판 크롤러)

#### 역할
- AnyFive 그룹웨어에 로그인(master, five4190any#), 게시판선택은 모두 수동으로 해야합니다. 
- 게시판 목록 페이지에서 모든 게시글 순회
- 각 게시글의 상세 정보(제목, 본문, 첨부파일, 댓글) 수집
- MariaDB에 저장

#### 설정 항목

```python
# BASE URL (수정 불필요)
BASE = "http://office.anyfive.com/home/"

# 게시판 타입 - 실행 시 입력받음 (맨 아래 '게시판 타입(type), targetBoardId' 참조)
BOARD_TYPE = input("게시판 타입을 입력하세요 (예: hredu, cp, sales): ").strip()

# DB 설정 (필요 시 수정)
USER = "root"
PASS = "1234"
HOST = "127.0.0.1"
PORT = 3306
DB = "crawling"

# 페이지당 수집 개수 (테스트 시 줄이기)
max_test = 50  # 50개씩 수집
max_pages = 999  # 최대 페이지 수
```

#### 실행 방법
해당 스크립트 실행 (ctrl + enter)

#### 실행 과정
1. 게시판 타입 입력 (예: `hredu`)
2. 브라우저 자동 실행 → AnyFive 로그인 페이지
3. **수동으로 로그인** 후 Enter
4. **좌측 메뉴에서 게시판 선택** 후 Enter
5. 자동 크롤링 시작

#### 출력 파일

| 파일명 | 설명 |
|--------|------|
| `board_posts.json` | 게시글 JSON |
| `board_comments.json` | 댓글 JSON |
| `게시판_목록.xlsx` | 게시글 Excel |
| `게시판_목록.csv` | 게시글 CSV |
| `게시판_댓글.xlsx` | 댓글 Excel |
| `게시판_댓글.csv` | 댓글 CSV |
+ DB 저장 board_posts, board_comments 테이블
---

### 8. 2_board_image_path_convert_DB.ipynb (이미지 경로 변환)

#### 역할
- DB에서 게시글 본문(content) 읽기
- `<img src="...">` 태그에서 이미지 경로 추출
- 원본 이미지 파일 찾아서 복사 (확장자 `.jpg` 추가)
- DB의 content 컬럼 업데이트

#### 설정 항목 (⚠️ 수정 필요)

```python
# 게시판 타입
BOARD_TYPE = "hredu"  # 처리할 게시판 타입

# 이미지 원본 경로들 - 본인 환경에 맞게 수정
BASE_IMAGE_PATHS = [
    r"C:\Users\LEEJUHWAN\Downloads\2021-01-01~2025-10-31\html\resource\image\namoeditor",
    r"C:\Users\LEEJUHWAN\Downloads\2021-01-01~2025-10-31\html\resource\image\daumeditor",
    r"C:\Users\LEEJUHWAN\Downloads\2016-01-01~2020-12-31\html\resource\image\namoeditor",
    r"C:\Users\LEEJUHWAN\Downloads\2016-01-01~2020-12-31\html\resource\image\daumeditor",
]

# 이미지 복사 저장 폴더
OUTPUT_IMAGE_FOLDER = rf"C:\Users\LEEJUHWAN\Downloads\board_{BOARD_TYPE}_img"

# 경로에서 제거할 접두사
REMOVE_PREFIX = r"C:\Users\LEEJUHWAN\Downloads"
```
#### 실행 방법
해당 스크립트 실행 (ctrl + enter)
---

### 8. 3_board_path_convert_DB.ipynb (첨부파일 경로 변환)

#### 역할
- DB에서 첨부파일 정보(attaches_json) 읽기
- 원본 첨부파일 찾아서 복사
- DB의 attaches_json 컬럼 업데이트

#### 설정 항목 (⚠️ 수정 필요)

```python
# 게시판 타입
BOARD_TYPE = "hredu"

# 첨부파일 원본 경로들
BASE_FILE_PATHS = [
    r"C:\Users\LEEJUHWAN\Downloads\2021-01-01~2025-10-31\html\resource\file\brd",
    r"C:\Users\LEEJUHWAN\Downloads\2016-01-01~2020-12-31\html\resource\file\brd",
    r"C:\Users\LEEJUHWAN\Downloads\2011-01-01~2015-12-31\html\resource\file\brd",
    r"C:\Users\LEEJUHWAN\Downloads\2010-01-01~2010-12-31\html\resource\file\brd",
]

# 첨부파일 복사 저장 폴더
OUTPUT_FILE_FOLDER = rf"C:\Users\LEEJUHWAN\Downloads\board_{BOARD_TYPE}_attachments"
```
#### 실행 방법
해당 스크립트 실행 (ctrl + enter)
---

### 8. 4_targetBoardId_insert_DB.ipynb (대상 게시판 ID 설정)

#### 역할
- DB에서 특정 type의 모든 게시글에 동일한 targetBoardId 적용

#### 설정 항목 (⚠️ 수정 필요)

```python
# 게시판 타입
BOARD_TYPE = "hredu"

# targetBoardId는 맨 아래 '게시판 타입(type), targetBoardId'참고
TARGET_BOARD_ID = "1438157806459977728"
```
#### 실행 방법
해당 스크립트 실행 (ctrl + enter)

---

### 8. 5_generate_cmds_from_db.ipynb (게시글 cmds 생성)

#### 역할
- DB에서 게시글 데이터 읽기
- `.cmds`으로 변환

#### 설정 항목

```python
BOARD_TYPE = "hredu"
TABLE_NAME = "board_posts"
OUTPUT_FILE = f"posts_{BOARD_TYPE}.cmds"  # 자동 생성
```

#### 출력 형식

```
addPost{"sourceId":"123_456","targetBoardId":"1438...","createdAt":"1234567890123",...}
addPost{"sourceId":"123_457","targetBoardId":"1438...","createdAt":"1234567890456",...}
```

#### 실행 방법
해당 스크립트 실행 (ctrl + enter)


---

### 8. 6_generate_comment_cmds_from_db.ipynb (댓글 cmds 생성)

#### 역할
- DB에서 댓글 데이터 읽기
- `.cmds`으로 변환

#### 설정 항목

```python
BOARD_TYPE = "hredu"
TABLE_NAME = "board_comments"
OUTPUT_FILE = f"comments_{BOARD_TYPE}.cmds"  # 자동 생성
```

#### 출력 형식

```
addPostComment{"sourceId":"123_456_0","sourcePostId":"123_456","createdAt":"1234567890123",...}
addPostComment{"sourceId":"123_456_1","sourcePostId":"123_456","createdAt":"1234567890456",...}
```

#### 실행 방법
해당 스크립트 실행 (ctrl + enter)


---

## 9. 단계별 실행 가이드

### 9.1 전체 실행 순서

```
┌────────────────────────────────────────────────────────────────────┐
│  1. 크롤링          →  any_board_crawling_게시판이름_251113.ipynb   │
│  2. 이미지 변환      →  2_board_image_path_convert_DB.ipynb        │
│  3. 첨부파일 변환    →  3_board_path_convert_DB.ipynb              │
│  4. 게시판 ID 설정   →  4_targetBoardId_insert_DB.ipynb            │
│  5. 게시글 cmds 생성 →  5_generate_cmds_from_db.ipynb              │
│  6. 댓글 cmds 생성   →  6_generate_comment_cmds_from_db.ipynb      │
└────────────────────────────────────────────────────────────────────┘
```

### 9.2 Step 1: 크롤링 실행

#### 준비물
- [ ] Chrome 브라우저 설치 완료
- [ ] MariaDB 실행 중
- [ ] `users_filtered.csv` 파일 준비
- [ ] AnyFive 로그인 계정 준비

#### 실행

```cmd
cd C:\board_migration\scripts
python any_board_crawling_게시판이름_251113.ipynb
```

#### 진행 순서

1. **게시판 타입 입력**
   ```
   게시판 타입을 입력하세요 (예: hredu, cp, sales): hredu
   ```

2. **로그인 대기**
   - Chrome 브라우저가 자동으로 열림
   - AnyFive 로그인 페이지에서 **수동으로 로그인**
   - "다음에 변경" 버튼이 있으면 클릭
   
   ```
   로그인(필요시 '다음에 변경')까지 완료 후 Enter...
   ```

3. **게시판 선택**
   - 원하는 게시판 선택 (예: HR교육)
   
   ```
   ======================================================================
   📋 [hredu] 게시판으로 이동해주세요
   ======================================================================
   1. 좌측 메뉴에서 '게시판' 클릭
   2. '(주)애니파이브' 하위 메뉴 확장
   3. 원하는 상세 게시판 선택
   4. 게시판 목록이 화면에 표시되면 Enter를 눌러주세요
   ======================================================================
   준비 완료 후 Enter...
   ```

4. **크롤링 진행**
   ```
   📊 크롤링 시작...
   
   ======================================================================
   📄 페이지 1 수집 중...
   ======================================================================
     ✅ 게시글 20개 발견
   
   📄 [1/20] 2024년 신입사원 교육 안내
      게시판: HR교육, 댓글: 3개
     ✅ 수집 완료: 2024년 신입사원 교육 안내
        - 댓글 3개
   
   📄 [2/20] 온라인 교육 플랫폼 이용 가이드
   ...
   ```

5. **완료 확인**
   ```
   ======================================================================
   📊 수집 완료
   ======================================================================
   ✅ 게시글: 150건
   ✅ 댓글: 89건
   ✅ JSON 저장 완료: board_posts.json, board_comments.json
   ✅ 파일 저장 완료: 게시판_목록.xlsx / 게시판_목록.csv
   ✅ DB 저장 완료 (테이블: board_posts, type: hredu)
   ✅ 파일 저장 완료: 게시판_댓글.xlsx / 게시판_댓글.csv
   ✅ DB 저장 완료 (테이블: board_comments, type: hredu)
   
   🎉 모든 작업 완료!
   ```

---

### 9.3 Step 2: 이미지 경로 변환

#### 스크립트 수정

`2_board_image_path_convert_DB.ipynb` 열어서 수정:

```python
# 1. 게시판 타입 설정
BOARD_TYPE = "hredu"  # ← 크롤링한 타입과 동일하게

# 2. 원본 이미지 경로 설정 (본인 환경에 맞게)
BASE_IMAGE_PATHS = [
    r"C:\Users\본인계정\Downloads\2021-01-01~2025-10-31\html\resource\image\namoeditor",
    r"C:\Users\본인계정\Downloads\2021-01-01~2025-10-31\html\resource\image\daumeditor",
    # 필요한 경로 추가...
]

# 3. 출력 폴더 설정
OUTPUT_IMAGE_FOLDER = rf"C:\board_migration\output\board_{BOARD_TYPE}_img"
```

#### 실행

```cmd
python 2_board_image_path_convert_DB.ipynb
```

#### 결과 확인

```
📂 DB 연결 중...
📂 게시판 타입: hredu
📂 검색할 이미지 경로: 4개

✅ 총 150개의 posts 발견

📄 [1] 문서ID: 12345
   원본: C:\Users\...\namoeditor\brd12345
   복사: C:\board_migration\output\board_hredu_img\brd12345
   ✅ 이미지 3개 발견
      ✅ 0 → 0.jpg
      ✅ 1 → 1.jpg
      ✅ 2 → 2.jpg
   📊 복사 완료: 3/3개
...

============================================================
✅ 처리 완료!
   - 게시판 타입: hredu
   - 처리된 posts: 45/150개
   - 복사된 이미지 위치: C:\board_migration\output\board_hredu_img
   - DB 테이블: board_posts (content 컬럼 업데이트)
============================================================
```

---

### 9.4 Step 3: 첨부파일 경로 변환

#### 스크립트 수정

`3_board_path_convert_DB.ipynb ` 열어서 수정:

```python
BOARD_TYPE = "hredu"

BASE_FILE_PATHS = [
    r"C:\Users\본인계정\Downloads\2021-01-01~2025-10-31\html\resource\file\brd",
    r"C:\Users\본인계정\Downloads\2016-01-01~2020-12-31\html\resource\file\brd",
    # 필요한 경로 추가...
]

OUTPUT_FILE_FOLDER = rf"C:\board_migration\output\board_{BOARD_TYPE}_attachments"
```

#### 실행

```cmd
python 3_board_path_convert_DB.ipynb 
```

---

### 9.5 Step 4: targetBoardId 설정

#### targetBoardId는 맨 아래 '게시판 타입(type), targetBoardId' 참고

#### 스크립트 수정

```python
BOARD_TYPE = "hredu"
TARGET_BOARD_ID = "1438157806459977728"  
```

#### 실행

```cmd
python 4_targetBoardId_insert_DB.ipynb 
```

---

### 9.6 Step 5 & 6: cmds 파일 생성

#### 게시글 cmds 생성

```cmd
python 5_generate_cmds_from_db.ipynb 
```

출력: `posts_hredu.cmds`

#### 댓글 cmds 생성

```cmd
python 6_generate_comment_cmds_from_db.ipynb
```

출력: `comments_hredu.cmds`

---

### 9.7 여러 게시판 처리하기

다른 게시판도 전체 과정을 반복합니다:

```
1회차: BOARD_TYPE = "hredu"   → 크롤링 ~ cmds 생성
2회차: BOARD_TYPE = "cp"      → 크롤링 ~ cmds 생성  
3회차: BOARD_TYPE = "biz"   → 크롤링 ~ cmds 생성
                      .
                      .
                      .
```

---

### 이미 크롤링한 게시판을 다시 크롤링하면 DB에 중복 데이터가 추가됩니다. 재크롤링 전에 해당 type의 데이터를 삭제하세요

```sql
DELETE FROM board_posts WHERE type = 'hredu';
DELETE FROM board_comments WHERE type = 'hredu';
```
---

### 현재 스크립트는 전체 게시글을 수집합니다. 기간 필터링이 필요하면 크롤링 후 DB에서 삭제하거나, 스크립트를 수정해야 합니다.

---

### 게시판 타입(type), targetBoardId
> 💡  DB 테이블은 통합되어 있으므로 `type` 컬럼으로 구분됩니다.
- `hredu` - HR교육 게시판 - 1438157806459977728
- `biz` - 전사 게시판 - 1427170574110560256
- `cp` - cp표준단가 게시판 - 1438157587093684224
- `qna` - TechQnA - 1433257924444065792
- `techInfo` - 기술/시장정보 - 1433257766968922112
- `club` - 사내동호회 - 1433257719275487232
- `docForm` - 사내문서양식 - 1427170574127337472
- `event` - 사우회/경조 - 1433257621279768576
- `free` - 자유게시판 - 1433257972863111168
- 업무처리지침 - 1433257128352579584
- 인사관리규정 - 1433257404648165376

