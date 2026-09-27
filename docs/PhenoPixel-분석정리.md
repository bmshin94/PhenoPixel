# PhenoPixel 전수조사 · 분석 정리 📊

> 이 문서는 PhenoPixel 저장소를 전수조사(파일 1,176개)하고 정리한 분석 노트입니다.
> 작성: Claude Code 세션 (페르소나: 카리나) / 정리일: 2026-09-27

---

## 🔗 GitHub 주소

| 구분 | URL | 비고 |
| --- | --- | --- |
| **원본 저장소 (upstream)** | https://github.com/ikeda042/PhenoPixel | ⭐ 200 stars · fork 6 · MIT |
| **이 저장소 (fork)** | https://github.com/bmshin94/PhenoPixel | 작업 대상 |
| 원저자 | Yunosuke Ikeda (히로시마대학교) · d263846@hiroshima-u.ac.jp | |
| 원본 생성일 | 2026-05-27 (약 4개월 만에 200스타) | |
| 라이선스 | MIT (상업적 이용 가능) | |
| Topics | bacteria, bioinformatics-pipeline, cell-analysis, cell-contour, fastapi, fluorescence-microscopy-imaging, microbiology, nikon-nd2, opencv, react, single-cell-analysis | |

### 참고 문서 / 외부 링크

| 항목 | URL |
| --- | --- |
| Cellpose-SAM 논문 | https://doi.org/10.1101/2025.04.28.651001 |
| Cellpose 소스 | https://github.com/MouseLand/cellpose |
| Cellpose 문서 | https://cellpose.readthedocs.io/ |
| Cellpose PyPI | https://pypi.org/project/cellpose/ |
| FastAPI | https://fastapi.tiangolo.com/ |
| Chakra UI | https://chakra-ui.com/ |

---

## 1. 한 줄 정의

> **현미경으로 찍은 박테리아(대장균) 사진에서 세포를 하나씩 자동으로 잘라내고,
> 모양·형광 세기를 숫자로 뽑아주는 웹 기반 생물이미지 분석 플랫폼.**

쉬운 비유: **"졸업 단체사진에서 얼굴만 오려내고 → 잘못 오린 것 버리고 → 한 명씩 확인하고 → 평균 키 통계 내기"**
여기서 "얼굴" 자리에 "대장균 세포"가 들어간 것.

---

## 2. 저장소 구조 지도

```
PhenoPixel/
├── backend/                    Python FastAPI (15,835줄)
│   ├── main.py                 진입점 · 포트 3000 · prefix /api/v1
│   ├── requirements.txt        의존성 17개
│   ├── openapi.json            API 명세 (51 경로 / 55 오퍼레이션)
│   ├── app/
│   │   ├── nd2files/             406줄  ND2 업로드·목록·삭제
│   │   ├── nd2parser/            631줄  ND2 메타데이터 파싱
│   │   ├── cellextraction/     1,890줄  ⭐ 세포 추출 핵심 엔진
│   │   ├── database_manager/   3,405줄  ⭐ 최대 모듈 · 세포 뷰어 렌더링
│   │   ├── bulk_engine/        2,732줄  ⭐ 집단 통계 분석
│   │   ├── mother_machine/     3,495줄  ⭐ Cellpose-SAM 타임랩스
│   │   ├── graphengine/          443줄  CSV 업로드 → 그래프 PNG
│   │   ├── mcp/                1,739줄  🔥 FastMCP 서버 (AI 에이전트용)
│   │   ├── activity_tracker/     234줄  사용 기록(잔디밭) 통계
│   │   ├── slack/                259줄  작업 완료 슬랙 알림
│   │   ├── file_manager/         165줄  범용 파일 관리
│   │   ├── system/                38줄  ⚠️ git pull 셀프 업데이트
│   │   └── shared/               171줄  대물렌즈 배율 스케일 등
│   ├── autoannotation/         🤖 머신러닝 자동 라벨링
│   │   ├── artifacts/autoannotator.pkl   학습된 모델
│   │   ├── features.py                   96개 특징 추출
│   │   └── testdata/*.db                 520개 라벨 학습 데이터
│   └── tests/                  unittest 12개 파일
│
├── frontend/                   React 19 + TypeScript (15,271줄)
│   ├── src/pages/              14개 페이지
│   ├── src/components/         재사용 컴포넌트 + Storybook
│   ├── docs-site/              Docusaurus 문서 사이트 (포트 3002)
│   └── dist/                   빌드 결과물 (일본어 폰트 1000+ 커밋됨)
│
├── docker/                     Traefik + Let's Encrypt 자동 HTTPS
├── docs/                       스크린샷 30여 장 · GIF · 수식 다이어그램
├── README.md (29KB)            논문급 수식 문서
├── Readme_ja.md (19KB)         일본어판
└── CLAUDE.md                   카리나 페르소나 파일
```

---

## 3. 워크플로우 (핵심)

```
① ND2 업로드       니콘 현미경 전용 파일(.nd2)
       ↓
② Cell Extraction  Canny 엣지검출로 윤곽선 추출 → SQLite DB 생성
       ↓            🤖 AutoAnnotation ML이 "진짜 세포 vs 쓰레기" 자동 분류
       ↓
③ Annotation       사람이 검수 (N/A ↔ Label 1), Shift+드래그로 다중 선택
       ↓
④ Cells (뷰어)     세포 1개씩 상세 관찰 (9가지 모드)
       ↓
⑤ Bulk Engine      Label 집단 전체 통계 분석 (10가지 모드)
       ↓
⑥ JSON/CSV 내보내기 → 논문 그래프
```

### ND2 파일이란

| 항목 | 일반 JPG | ND2 |
| --- | --- | --- |
| 색상 정보 | RGB 3색 (사람 눈용) | 채널별 흑백 (기계용) |
| 밝기 단계 | 256단계 (8bit) | 65,536단계 (16bit) |
| 추가 정보 | 없음 | 배율, μm 단위, 촬영 시간, Z축 |
| 장수 | 1장 | 여러 채널 × 시간 × 위치 |

채널 구성 예시:
- 채널1 (PH, 위상차) → 세포의 "몸" 모양
- 채널2 (Fluo1) → GFP로 표시한 단백질 A 위치
- 채널3 (Fluo2) → 단백질 B 위치

지원 모드: `single` / `dual` / `dual(reversed)` / `triple` / `quad`

### 추출 설정 3개

| 설정 | 기본값 | 의미 |
| --- | --- | --- |
| `param1` | 130 | Canny 민감도. 낮추면 많이 잡지만 쓰레기↑ |
| `crop size` | 200px | 세포 1개 잘라낼 박스 크기 |
| `objective` | 100x | 배율. 100x=0.065 μm/px, 60x=0.108 μm/px |

기타 기본값: 히트맵 빈 35 · 중심선 다항식 차수 4 · FITC 컷오프 0.7414 · 추출 동시성 2

### 세포 뷰어 9가지 모드

| 모드 | 설명 |
| --- | --- |
| `Contour` | 윤곽선 좌표 |
| `Replot` | 세포를 정렬 좌표계로 다시 그리기 (기본값) |
| `Overlay` | 윤곽선을 원본 위에 겹치기 |
| `Overlay Raw` | 위상차 이미지 위에 형광 얹기 |
| `Overlay Fluo` | 형광 채널만 색상 합성 |
| `Heatmap` | 장축 방향 형광 세기 히트맵 |
| `Map 256` | 256×1024 고정 크기 형광 공간 맵 |
| `Map Raw` | 리사이즈 전 원본 해상도 맵 |
| `Distribution` | 세포 내 형광 세기 히스토그램 |

### Bulk Engine 10가지 분석

Cell length(μm) · Cell width · Cell area(px²) · Normalized median · FITC aggregation ratio ·
Entropy · Heatmap · Contours · Map256 · Raw data

---

## 4. 핵심 알고리즘 (쉬운 설명)

### ① PCA 주축 정렬 — "세포를 똑바로 눕히기"

세포가 사진 속에서 아무 방향으로 누워있어 비교가 불가능한 문제를 해결.
윤곽선 좌표의 공분산 행렬 Σ의 고유벡터를 구해, 큰 고유값(λ₁) 방향을 X축으로 회전.

- 변환 행렬 Q는 직교행렬 → QᵀQ = QQᵀ = I → **길이 보존** (README에 증명 있음)
- 작은 고유값 λ₂는 "두께" 지표 → AutoAnnotation 규칙에 사용

### ② 중심선 다항식 피팅 — "휘어진 세포 길이 재기"

대장균은 바나나처럼 휠 수 있어 직선거리로 재면 짧게 나옴.
정렬된 세포에 4차 다항식으로 "등뼈(중심선)"를 긋고 호길이를 적분:

```
L = ∫ sqrt(1 + (f'(u))²) du
```

⚠️ **중요**: README가 정직하게 밝혀놨듯, **현재 코드는 이 적분을 쓰지 않음.**
실제로는 "세포 내부 픽셀의 PCA 장축 방향 최대-최소 거리 × 픽셀 크기"로 근사:

```
L_API ≈ (max πᵢ - min πᵢ) × 0.065
```

논문 수식과 구현이 다르다는 것을 문서에 명시한 점은 신뢰할 만한 태도.

### ③ 35빈 맥스풀링 — "크기 다른 세포 비교하기"

세포 길이가 제각각인 문제를 해결: **무조건 35조각으로 나눠서** 각 칸의 최대값만 추출.

```
gⱼ = max{ G(pᵢ,qᵢ) | ℓᵢ* ∈ Iⱼ }
g = (g₁, ..., g₃₅)
```

→ 항상 35차원 고정 특징 벡터 = 머신러닝에 바로 투입 가능 (CNN의 Global Pooling과 같은 원리)
→ 평균이 아니라 max를 쓰는 이유: 형광 단백질이 "뭉친 곳"을 찾는 것이 목적

### ④ 정규화 중앙값 — "뭉침을 숫자 하나로"

```
Ĩᵢ = Iᵢ / max(I)          ← 세포별 최대값으로 정규화
m(C) = median(Ĩᵢ)          ← 그 다음 중앙값
```

- 골고루 퍼진 세포: [0.8, 0.9, 0.85, 0.9, 0.8] → median ≈ 0.85 (높음)
- 한 점에 뭉친 세포: [0.1, 0.1, 1.0, 0.1, 0.1] → median ≈ 0.10 (낮음)

**중앙값이 낮으면 = 응집(aggregation)**. 현미경 설정·발현량이 달라도 비교 가능.

집단 응집 비율: `R(τ) = (1/N) Σ 1[m(Cc) < τ]`, 기본 τ = 0.7414

### ⑤ Map256 4방향 대칭 평균

집단 평균 지도를 만들 때 좌우/상하 방향 편향을 제거하기 위해, 각 세포마다
원본 + 좌우반전 + 상하반전 + 둘다반전 = 4개 버전을 모두 평균:

```
M̄ = (1/4N) Σ [ Mc + Fx(Mc) + Fy(Mc) + Fy(Fx(Mc)) ]
```

→ 딥러닝의 Test-Time Augmentation과 같은 원리.
→ "중앙 집중이냐, 극(pole) 집중이냐"가 깨끗하게 드러남.

### ⑥ 대조군 기반 임계값

고정값 대신 대조군 분포에서 임계값을 뽑는 방법:

| 지표 | 임계값 정의 | 판정 |
| --- | --- | --- |
| HU-GFP 압축도 | τ = 대조군 점수의 5% 분위수 | 이보다 낮으면 비정상 |
| PI 투과성 | τ = 대조군 평균의 95% 분위수 | 이보다 높으면 죽은 세포 |

→ "정상의 95%가 들어가는 범위를 정상으로 정의" = 이상탐지(anomaly detection) 사고방식

### ⑦ AutoAnnotation 머신러닝

```
520개 수동 라벨 (Label 1: 300 / N/A: 220)
       ↓
96개 특징 추출
  - 모양: 면적, 둘레, 원형도, 볼록도, 채움도, Hu 모먼트, PCA 축 분산, 편심률
  - 밝기: PH/Fluo 내부·외부링 밝기 분위수, 대비, 그라디언트, 엣지 밀도
       ↓
여러 분류기 학습 → 5-fold 계층 교차검증으로 최고 성능 선택
       ↓
최종: 가중 kNN + L2 로지스틱 회귀 앙상블
       ↓
autoannotator.pkl
```

| 지표 | 값 |
| --- | --- |
| F1 | **0.9608** |
| 정확도 | 0.9538 |
| 정밀도 | 0.9423 |
| 재현율 | **0.9800** |

재현율 98%가 중요한 이유: 연구에서는 "놓치는 것"이 더 치명적. 잘못 넣은 건 3단계에서 지울 수 있지만, 안 잡힌 세포는 되돌릴 수 없음.

**폴백(fallback)**: 모델 로드 실패 시 기하 규칙으로 자동 전환
- λ₂ ≤ 120 (측면 압축) AND κ(C) = P(Hull)/P(C) > 0.85 (볼록도)
- `s(C) = 1[λ₂ ≤ 120] · 1[κ(C) > 0.85]`

커스텀 모델 경로: `PHENOPIXEL_AUTOANNOTATION_MODEL=/path/to/model.pkl`

---

## 5. 아키텍처

```
   [브라우저]  React 19 + Chakra UI 3 + 14개 페이지
        ↓ fetch()
   ┌─────────────────────────────────┐
   │  FastAPI  (포트 3000)            │
   │  /api/v1/**  ← 55개 엔드포인트    │
   │  /mcp/       ← AI 에이전트용 MCP  │
   │  /**         ← React 빌드 서빙    │
   └─────────────────────────────────┘
        ↓
   OpenCV / NumPy / Matplotlib   ← 이미지 처리
        ↓
   SQLite (.db 파일들)            ← 저장
        ↓
   Slack Webhook (옵션)           ← 완료 알림
```

### 설계 포인트 3개

1. **단일 서버 배포** — `frontend/dist/index.html`이 있으면 FastAPI가 React를 직접 서빙. nginx 불필요.
2. **무거운 작업 논블로킹** — `loop.run_in_executor()`로 OpenCV 처리를 스레드풀에 위임 + `CELLEXTRACTION_MAX_CONCURRENCY=2`로 동시성 제한.
3. **이미지 메모리 스트리밍** — `StreamingResponse(io.BytesIO(...), media_type="image/png")`. 디스크 낭비 없음.

### 왜 SQLite인가

파일 하나로 끝나서 이메일·USB로 공유 쉬움. "내 분석 결과 DB 보내줄게" = 파일 하나 전달.
멀티테넌시에도 유리 (작업별 독립 DB 파일).

---

## 6. MCP 서버 (내장)

`backend/app/mcp/` — FastMCP 3.2.4 기반, FastAPI 안에 `/mcp`로 마운트.

### 노출 도구 10개

| 도구 | 기능 |
| --- | --- |
| `build_context_bundle` | 질문에 맞는 코드 컨텍스트 자동 수집 |
| `search_code` | 저장소 코드 검색 |
| `read_source_fragment` | 파일 일부만 안전하게 읽기 |
| `explain_api_surface` | API 구조 설명 |
| `list_pages` | 프론트 페이지 목록 |
| `explain_page` | 페이지 하나 완전 분석 |
| `trace_page_api` | 페이지 → 백엔드 API 추적 |
| `trace_ui_action` | **"이 버튼 누르면 어느 API 호출?"** 추적 |
| `explain_algorithm` | 알고리즘 설명 |
| `build_page_context` | 페이지 + 질문 컨텍스트 |

리소스 5개: `repo://overview`, `repo://openapi`, `frontend://pages`, `frontend://page/{route}`, `frontend://page/{route}/api-map`
프롬프트 1개: `analyze_page`

### 보안 (guards.py) — 5중 방어

```python
# ① 화이트리스트 (블랙리스트보다 안전)
EXACT_ALLOWLIST = ("README.md", "backend/main.py", ...)
GLOB_ALLOWLIST  = ("backend/app/**/*.py", "frontend/src/pages/**/*.tsx", ...)

# ② 경로 탈출 차단
if ".." in raw_path.parts: raise GuardViolation(...)

# ③ resolve() 후 재확인 (심볼릭 링크 공격 방어)
resolved = candidate.resolve(strict=False)
resolved.relative_to(repo_root)

# ④ 확장자 블랙리스트 (바이너리 차단)
DENIED_NAME_SUFFIXES = (".db", ".png", ".woff", ".mp4", ...)
EXCLUDED_PREFIXES    = ("frontend/dist/", "backend/app/databases/")

# ⑤ 크기 제한 (컨텍스트 폭발 방지)
MAX_FRAGMENT_LINES = 160
MAX_FRAGMENT_BYTES = 24_000
```

③번이 중요한 이유: 심볼릭 링크로 `/etc/passwd`를 가리키면 `..` 없이도 밖으로 나갈 수 있음.
`resolve()` 후 `relative_to()`로 재검증하는 것이 정석 방어법.

### indexer.py — 벡터DB 없는 코드 검색

```python
# 한국어/일본어 지원 토크나이저
TOKEN_RE = re.compile(r"[A-Za-z0-9_/-]{2,}|[ぁ-んァ-ヶ一-龯]{2,}")

# 역인덱스 (토큰 → 파일 목록)
keyword_map: dict[str, set[str]] = defaultdict(set)

# 캐시 무효화를 (파일 수, 최대 mtime) 튜플로 판단 — 해시 계산 없이 빠름
token = (file_count, max_mtime)
if self._cached_index and self._cached_index.snapshot_token == token:
    return self._cached_index
```

→ 임베딩 API 비용 0 · 외부 의존성 0 · 즉시 응답

### 코드 그래프 (frontend_graph.py + backend_graph.py)

정규식만으로 "UI 버튼 → API → 구현 함수"를 자동 추적 (LSP/AST 없이):

```
trace_ui_action("bulk-engine", "Export CSV")
  → Export CSV / handleExportCsv
    → GET /api/v1/get-heatmap-vectors-csv
      → router: backend/app/bulk_engine/router.py:142
      → crud:   backend/app/bulk_engine/crud.py:389
      → readme: backend/app/bulk_engine/README.md:56
```

### server.py — 계층 분리 패턴

```python
# ① 비즈니스 로직은 순수 클래스 (MCP와 무관 → 테스트 쉬움)
class MCPService:
    def explain_page(self, page_ref, question=None) -> dict: ...

# ② MCP 도구는 얇은 래퍼
@mcp.tool
def explain_page(page_ref: str, question: str | None = None) -> dict:
    return get_mcp_service().explain_page(page_ref=page_ref, question=question)

# ③ 서비스는 lru_cache로 싱글톤
@lru_cache(maxsize=4)
def get_mcp_service(repo_root=None) -> MCPService: ...
```

### 응답 설계의 핵심 인사이트

```python
return {
    "page": {...},        # 구조화 데이터 (프로그램용)
    "endpoints": [...],   # 구조화 데이터
    "answer": render_sections(sections),  # ← 사람/LLM이 바로 읽는 텍스트
}
```

JSON만 주면 LLM이 해석하느라 토큰 낭비. **미리 정리된 마크다운을 같이 주면** 즉시 이해.

### FastAPI + MCP 통합

```python
from fastmcp.utilities.lifespan import combine_lifespans

mcp_app = create_mcp_http_app()
app = FastAPI(lifespan=combine_lifespans(app_lifespan, mcp_app.lifespan))  # 핵심!
app.mount("/mcp", mcp_app)

# 307 리다이렉트로 슬래시 실수 방어
@app.api_route("/mcp", methods=["GET","POST",...], include_in_schema=False)
async def redirect_mcp_root(request):
    return RedirectResponse(url=str(request.url.replace(path="/mcp/")), status_code=307)
```

`combine_lifespans`를 안 쓰면 MCP 초기화가 안 됨.

### 연결 방법

```bash
# Gemini CLI
gemini mcp add --transport http phenopixel http://localhost:3000/mcp

# Claude Code
claude mcp add --transport http phenopixel http://localhost:3000/mcp

# 도구 목록 확인
cd backend && ../venv/bin/fastmcp list http://localhost:3000/mcp --resources --prompts
```

`~/.gemini/settings.json`:

```json
{
  "mcpServers": {
    "phenopixel": {
      "httpUrl": "http://localhost:3000/mcp",
      "timeout": 10000,
      "includeTools": [
        "build_context_bundle", "search_code", "read_source_fragment",
        "explain_api_surface", "list_pages", "explain_page",
        "trace_page_api", "trace_ui_action", "explain_algorithm",
        "build_page_context"
      ]
    }
  }
}
```

### 범위와 안전

- 포함: 저장소 컨텍스트, 프론트 페이지 추적, 백엔드 API 설명 (**읽기 전용**)
- 제외: 수정 엔드포인트, 추출 실행, git/시스템 명령, DB 쓰기
- 인덱싱 제외: `frontend/dist/**`, `backend/app/databases/**`, `.db`, 이미지, 비디오, 폰트

---

## 7. 설치 및 사용법

### 준비물

| 필요한 것 | 버전 | 왜 |
| --- | --- | --- |
| Python | 3.11 ~ 3.14 | 백엔드 (README는 3.14, Docker는 3.11) |
| Node.js / npm | 최신 LTS | 프론트엔드 빌드 |
| git | 아무거나 | 클론 |
| 디스크 | 최소 5GB | 저장소 296MB + Cellpose 모델 + 의존성 |
| (선택) Docker | 최신 | 프로덕션 배포 |
| (선택) NVIDIA GPU | - | Mother Machine의 Cellpose 가속 |

> ⚠️ **Python 3.14 주의**: `opencv-python-headless==5.0.0.93`, `cellpose==4.2.1.1` 같은 무거운 패키지가
> 3.14용 휠을 안 줄 수 있음. **처음엔 3.11 또는 3.12 권장** (Docker가 3.11 쓰는 이유).

### 로컬 개발 설치

```bash
# 1. 클론
git clone https://github.com/bmshin94/PhenoPixel.git
cd PhenoPixel

# 2. 백엔드 (저장소 루트에서 venv 생성!)
python3 -m venv venv
source ./venv/bin/activate        # Windows: venv\Scripts\activate
cd backend
pip install -r requirements.txt
python main.py

# 3. 프론트엔드 (새 터미널)
cd frontend
npm install                        # 충돌 시: npm install --legacy-peer-deps
npm run dev
```

성공 시 로그:
```
PhenoPixel is available on your local network at http://192.168.x.x:3000
INFO:     Uvicorn running on http://0.0.0.0:3000
```
LAN IP를 알려주므로 같은 와이파이의 다른 기기(태블릿 포함)에서도 접속 가능.

OpenCV 에러 시 (Linux):
```bash
sudo apt-get install -y libgl1 libglib2.0-0
```

> 💡 `npm run dev`는 concurrently로 Vite(3001) + Docusaurus(3002)를 동시 실행.
> 앱만 빠르게 보려면 `npx vite`로 직접 실행.

### 접속 URL

| 서비스 | URL |
| --- | --- |
| **프론트엔드 (개발)** | http://localhost:3001 |
| 백엔드 | http://localhost:3000 |
| **Swagger UI** (API 55개 직접 테스트!) | http://localhost:3000/api/v1/docs |
| OpenAPI JSON | http://localhost:3000/api/v1/openapi.json |
| 헬스체크 | http://localhost:3000/api/v1/health |
| Docusaurus 문서 | http://localhost:3002 |
| **MCP 서버** | http://localhost:3000/mcp/ |

### Docker 프로덕션 배포

```bash
cp backend/.env.template backend/.env

export SERVER_HOST=phenopixel.example.com
export TRAEFIK_ACME_EMAIL=you@example.com

# 프론트엔드를 먼저 빌드해야 함!
cd frontend && npm install --legacy-peer-deps && npm run build

cd ../docker
docker compose -f compose.yaml up -d --build
```

컨테이너 2개: `phenopixel6-traefik` (80/443, HTTPS 자동) + `phenopixel6-backend` (내부 3000)
볼륨 7개 마운트로 데이터 영속화.

⚠️ 주의: ① 포트 80/443 비어있어야 함 ② 도메인이 서버 IP를 가리켜야 인증서 발급
③ Traefik 라벨이 `PathPrefix(/api/v1/)`만 라우팅 → React 페이지 라우팅용 라벨 추가 필요할 수 있음

### 환경변수 (전부 선택 사항)

| 변수 | 의미 |
| --- | --- |
| `SLACK_WEHBOOK_URL` | 슬랙 알림 (오타 그대로! WEBHOOK 아님) |
| `BASE_PATH` | 슬랙 알림에 넣을 앱 URL |
| `CELLEXTRACTION_MAX_CONCURRENCY` | 동시 추출 제한 (기본 2) |
| `SERVER_HOST` | Docker/Traefik 도메인 |
| `TRAEFIK_ACME_EMAIL` | Let's Encrypt 이메일 |
| `PHENOPIXEL_AUTOANNOTATION_MODEL` | 커스텀 ML 모델 경로 |

### 첫 실행 튜토리얼

**데이터가 없어도 테스트 DB로 바로 체험 가능:**
- `backend/app/databases/test_database.db`
- `backend/autoannotation/testdata/autoannotation_testdata.db` (520개 세포)

```
시나리오 A (데이터 없을 때)
1. http://localhost:3001 접속
2. Databases → test_database.db → Open
3. Cells에서 세포 클릭 → Replot/Overlay/Heatmap/Map256/Distribution 모드 전환
4. Annotation에서 Label 변경
5. Bulk Engine → Label 1 → Cell length 실행 → JSON/CSV 다운로드

시나리오 B (내 ND2로 실제 분석)
1. ND2 Files → Upload
2. Cell Extraction (layer mode / objective / param1=130 / crop=200 / AutoAnnotation On)
3. Start Extraction → 프리뷰 확인 → 나쁘면 param1 조절해서 재추출
4. Annotation에서 정리
5. Cells에서 개별 확인
6. Bulk Engine → 내보내기
```

> 💡 처음엔 작은 ND2로 테스트. param1 튜닝하려면 몇 번 재추출해야 하므로.

### 테스트 & 품질 검사

```bash
# 백엔드
source ./venv/bin/activate
PYTHONPATH=backend python -m unittest discover backend/tests

# 프론트엔드
cd frontend && npm run build && npm run lint

# 스크린샷 자동 생성
./docs/make_screenshots.sh
```

### 트러블슈팅

| 증상 | 해결 |
| --- | --- |
| Port already in use | 3000/3001/3002 확인. `lsof -i :3000` |
| `eslint: command not found` | `npm install --legacy-peer-deps` |
| npm peer dependency 충돌 | `--legacy-peer-deps` 사용 |
| `ImportError: libGL.so.1` | `sudo apt install libgl1 libglib2.0-0` |
| ND2 업로드 거부 | 확장자가 정확히 `.nd2`인지 확인 |
| 윤곽선이 안 잡힘 | layer mode, param1, crop size 재확인 + 프리뷰 먼저 |
| SQLite 권한 에러 | `databases/`, `extracted_data/`, `nd2files/`, `tempdata/` 쓰기 권한 |
| Cellpose 다운로드 느림 | Mother Machine 첫 실행 시 `cpsam_v2` 모델 받아옴 |
| Python 3.14 pip 실패 | 3.11 / 3.12로 재시도 |

---

## 8. 정체: 플러그인? 스킬? MCP?

> **정답: 독립 실행형 웹 애플리케이션 + 내부에 MCP 서버 내장**

| 종류 | 정의 | PhenoPixel |
| --- | --- | --- |
| Plugin | 호스트 프로그램에 끼우는 확장 (혼자선 못 돔) | ❌ |
| Skill | AI에게 "이럴 때 이렇게 해"를 알려주는 Markdown 지침 | ❌ |
| MCP Server | AI가 외부 도구/데이터에 접근하는 표준 프로토콜 서버 | ⭕ 내부에 1개 보유 |
| **Web App** | 브라우저로 쓰는 독립 프로그램 | ✅ **이것** |

```
PhenoPixel (독립 웹앱)
├── 🌐 웹 UI            사람용 (React 14페이지)
├── ⚙️  REST API (55개)  프론트엔드용
└── 🤖 MCP 서버 (/mcp)   AI 에이전트용  ← 이 부분만 MCP
```

⚠️ 단, 이 MCP는 **"이 저장소의 코드를 읽어주는" 코드 내비게이션 MCP**.
세포 분석을 실행하는 MCP가 아님.

---

## 9. API 토큰 필요 여부

# ✅ 전혀 필요 없음

저장소 전체 grep 결과 **0건**:

```bash
grep -rniE "api[_-]?key|api[_-]?token|bearer|oauth|jwt|authorization|
            openai|anthropic|gemini_api|secret" backend/ frontend/src/ docker/
→ 결과 없음
```

### 이유: 모든 처리가 100% 로컬

```
❌ OpenAI API 안 씀        → LLM 없이 순수 이미지 처리
❌ 클라우드 비전 API 안 씀  → OpenCV 로컬 처리
❌ 외부 저장소 안 씀        → SQLite 로컬 파일
✅ ML 모델도 로컬           → autoannotator.pkl 저장소 포함
✅ Cellpose도 로컬          → cpsam_v2 다운로드 후 로컬 추론
```

### 이점 3가지

| 이점 | 설명 |
| --- | --- |
| 💵 운영비 $0 | API 호출 비용 없음. 무한히 돌려도 공짜 |
| 🔒 데이터 주권 | 미공개 연구 데이터가 절대 외부로 안 나감 |
| ✈️ 오프라인 작동 | 인터넷 없는 실험실·병원·폐쇄망에서도 완벽 작동 |

→ **"온프레미스 / 에어갭 배포 가능"은 병원·제약사·정부기관 대상 최고의 세일즈 포인트.**

### ⚠️ 단, 토큰이 없다 = 인증도 없다

```python
# backend/main.py — 모든 도메인 허용 + credentials까지
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_credentials=True, ...)
```

```python
# backend/app/system/router.py — 인증 체크 없음!
@router_system.post("/system/git-pull")
async def git_pull() -> dict[str, str]:
    process = await asyncio.create_subprocess_exec("git", "pull", "--ff-only", ...)
```

🚨 누구나 `POST /api/v1/system/git-pull`을 호출하면 서버가 git pull 실행.
`--ff-only`라 강제 덮어쓰기는 안 되지만, 원격 브랜치를 장악하면 코드 실행까지 이어질 수 있음.

**결론**: 연구실 내부 LAN 전용은 안전 / 인터넷 노출은 절대 금지 / 상용화 시 인증 추가가 1순위.

---

## 10. 발견한 이슈 목록

| 우선순위 | 문제 | 설명 |
| --- | --- | --- |
| 🚨 P0 | `/api/v1/system/git-pull` | 인증 없이 `git pull --ff-only` 실행 |
| 🚨 P0 | CORS `allow_origins=["*"]` | `allow_credentials=True`와 함께 사용 |
| 🔓 P0 | 인증 시스템 부재 | 로그인·권한 개념이 아예 없음 (LAN 전용 설계) |
| 🗑️ P2 | `frontend/src/App.tsx` 죽은 코드 | Chakra UI 공식 문서 템플릿 그대로. `main.tsx` 라우팅에서 `/`는 `TopPage`를 씀 → 어디서도 사용 안 됨 |
| 📦 P2 | `frontend/dist/` 커밋됨 | 일본어 폰트 woff/woff2 1000+ 개 → 저장소 296MB |
| 🐍 P3 | Python 버전 불일치 | README 3.14 vs Docker 3.11 |
| 📝 P3 | `SLACK_WEHBOOK_URL` 오타 | WEBHOOK 아님. README에 "의도적으로 코드 철자에 맞춤" 명시 |
| 🇯🇵 P3 | MCP 응답이 일본어 하드코딩 | `"概要"`, `"関連ページ"`, `"コード断片"` |
| 📐 P3 | 문서-구현 불일치 | Cell length가 수식의 호길이 적분이 아니라 PCA 근사 (README에 명시돼 있음) |

---

## 11. 왜 GitHub에서 유명한가

### 팩트

| 항목 | 값 |
| --- | --- |
| ⭐ Stars | 200 |
| 🍴 Forks | 6 |
| 🐛 Open Issues | 3 |
| 📅 생성 | 2026-05-27 (4개월 만에 200스타) |

니치 분야(미생물 이미지 분석, 전 세계 잠재 사용자 수만 명)에서 4개월 200스타는 매우 빠른 속도.

### 이유 8가지

1. **정확한 페인포인트** — ImageJ 수동 측정으로 1000개 세포 = 며칠. 이걸 몇 분으로.
2. **웹 기반의 희귀성** — ImageJ/Fiji/CellProfiler/MicrobeJ는 모두 데스크톱. 웹은 랩 서버 1대 설치로 전원 공유, 아이패드 라벨링 가능, 버전 통일.
3. **압도적 README (29KB)** — LaTeX 수식 30+, 스크린샷 25장, GIF 데모 2개, 알고리즘 다이어그램, 기술스택 배지, 파라미터 표, 트러블슈팅 표, 논문 인용 형식, 일본어판 별도.
4. **수학적 엄격함** — PCA 길이 보존 증명까지. 그리고 **"수식은 이건데 코드는 이렇게 근사했다"를 정직하게 명시** → 연구자에게 최고의 신뢰 신호.
5. **검증된 ML** — 520개 데이터 + 96특징 + 5-fold CV + 학습 데이터 DB와 평가 JSON까지 저장소에 포함 → 재현 가능.
6. **논문 인용 가능 설계** — 인용 형식 제공 + 커밋 해시 보고 가이드. 논문에 인용되면 → 독자가 또 별 → 선순환.
7. **트렌디한 스택** — React 19, Vite 7, Chakra 3, FastAPI 0.128, **FastMCP 3.2.4**, Cellpose-SAM. 연구자 ⭐ + 개발자 ⭐ 양쪽 획득.
8. **프로덕션 완성도** — Docker+Traefik+Let's Encrypt, Storybook, Playwright, Docusaurus, unittest 12개, Issue/PR 템플릿, CONTRIBUTING, MIT, 이중 언어 README.

### 객관적 비교

| 프로젝트 | Stars | 범위 |
| --- | --- | --- |
| napari | ~2,400 | 범용 |
| Cellpose | ~1,700 | 범용 세그멘테이션 |
| CellProfiler | ~1,000 | 범용 |
| **PhenoPixel** | **200** (4개월) | **박테리아 + ND2 특화** |

좁지만 그 안에서는 독보적. 성장 속도로 보면 더 오를 가능성.

---

## 12. 로컬 에이전트 구축에 도움 되는가

# ⭐⭐⭐⭐⭐ 매우 도움 됨

`backend/app/mcp/` 1,739줄에 필요한 패턴이 다 들어있음.

### 그대로 재사용 가능한 자산 5개

| # | 자산 | 가치 |
| --- | --- | --- |
| 1 | **`guards.py`** 보안 계층 | AI 에이전트 보안의 80%를 해결. 화이트리스트 + 경로탈출 + 심볼릭링크 + 확장자 + 크기 제한 5중 방어 |
| 2 | **`indexer.py`** 경량 검색 | 벡터DB 없이 코드 검색. 비용 0, 의존성 0. 한국어/일본어 토크나이저 포함 |
| 3 | **코드 그래프 빌더** | 정규식만으로 "UI → API → 구현" 자동 추적. LSP/AST 없이 "충분히 좋은" 엔지니어링 |
| 4 | **`server.py`** 구조 | Service 클래스 + 얇은 MCP 래퍼 분리 → 테스트 쉬움. `answer` 필드로 LLM 토큰 절약 |
| 5 | **FastAPI+MCP 통합** | `combine_lifespans` + `mount` + 307 리다이렉트 패턴 |

### 학습 로드맵

```
1주차: guards.py 완전 이해 → 내 프로젝트에 이식
2주차: indexer.py 이해 → 내 코드베이스 검색 구현
3주차: server.py 구조 모방 → MCPService 패턴으로 내 도구 만들기
4주차: 코드 그래프 아이디어 차용 → 내 프로젝트 호출 그래프
5주차: tests/test_mcp_*.py 참고 → MCP 도구 테스트 작성
```

참고 테스트: `test_mcp_tools.py`, `test_mcp_guards.py` (보안 테스트!), `test_mcp_backend_graph.py`, `test_mcp_frontend_graph.py`

### 주의

- MCP 응답 섹션 제목이 일본어 하드코딩 → 한국어/영어로 변경 필요
- 이 MCP는 "코드 읽기"용. 분석 실행 MCP는 직접 만들어야 함 (↓ 수익 기회)

---

## 13. React / PHP로 만들 수 있는가

### React → 이미 React

프론트엔드 전체가 React 19 + TypeScript, 15,271줄. 14개 페이지.
질문은 "만들 수 있나?"가 아니라 "어떻게 개선하나?"가 맞음.

| # | 개선 항목 | 이유 | 난이도 |
| --- | --- | --- | --- |
| 1 | 죽은 `App.tsx` 정리 | Chakra 문서 템플릿이 그대로 남아있고 라우팅에서 미사용 | ⭐ |
| 2 | TanStack Query 도입 | 현재 `fetch()` + `useState`. 캐싱/재시도/로딩 자동화 | ⭐⭐ |
| 3 | 코드 스플리팅 | 14페이지를 `React.lazy()`로 → 초기 로딩 개선 | ⭐⭐ |
| 4 | 가상 스크롤 (react-window) | 세포 수천 개 썸네일 격자 성능 | ⭐⭐⭐ |
| 5 | WebSocket 진행률 | 추출이 몇 분 걸리는데 진행률 표시 없음 | ⭐⭐⭐ |
| 6 | i18n 한국어 추가 | 현재 영어/일본어만 → 한국 시장 | ⭐⭐ |
| 7 | 모바일 라벨링 UI | 아이패드 라벨링 | ⭐⭐⭐ |

> 1번은 30분 작업이고 첫 PR로 적합. 원본 저장소 기여 이력을 만들 수 있음.

### PHP → 가능하지만 전체 재작성은 비추천

| 문제 | 상세 | 심각도 |
| --- | --- | --- |
| 과학 계산 생태계 부재 | NumPy 대응물 없음. 배열 연산 10~100배 느림 | 🔴🔴🔴 |
| OpenCV 바인딩 부실 | `php-opencv` 유지보수 불안정, 기능 제한 | 🔴🔴🔴 |
| ND2 파서 없음 | 니콘 독점 바이너리 포맷. 파이썬 라이브러리 수천 줄을 PHP로 재작성해야 함 | 🔴🔴🔴 |
| ML 불가 | `autoannotator.pkl`은 scikit-learn 형식. PHP에서 못 읽음 | 🔴🔴🔴 |
| Cellpose 불가 | PyTorch 모델. PHP 추론 불가 | 🔴🔴🔴 |

예시 — Python 3줄이 PHP에서는 130줄+:

```python
# Python
cov = np.cov(contour.T)
eigenvalues, eigenvectors = np.linalg.eig(cov)
aligned = eigenvectors.T @ contour.T
```
→ PHP: 공분산 직접 구현(20줄) + 고유값 분해 Jacobi 회전법(100줄+) + 행렬곱(15줄), 그리고 10배 느림.

### 권장: 하이브리드 아키텍처

```
┌────────────────────────────────────┐
│  🐘 PHP (Laravel / Symfony)        │
│  ✅ 회원가입 / 로그인 / 권한          │
│  ✅ 조직·팀·프로젝트 관리             │
│  ✅ 결제 (Stripe / 토스페이먼츠)      │
│  ✅ 관리자 대시보드                  │
│  ✅ 이메일 알림 / 초대               │
│  ✅ 감사 로그                        │
│  ✅ 업로드 수신 + 큐 등록             │
└────────────────────────────────────┘
              ↓ HTTP / 메시지큐
┌────────────────────────────────────┐
│  🐍 Python (FastAPI = PhenoPixel)  │
│  ✅ ND2 파싱 / OpenCV / ML          │
│  ✅ Cellpose / 통계 / 그래프          │
└────────────────────────────────────┘
```

Laravel 예시:

```php
class PhenoPixelClient
{
    public function extractCells(string $nd2Path, array $params): array
    {
        return Http::timeout(600)
            ->attach('file', file_get_contents($nd2Path), basename($nd2Path))
            ->post(config('services.phenopixel.url').'/api/v1/extract-cells', [
                'layer_mode' => $params['layer_mode'] ?? 'dual',
                'param1'     => $params['param1']     ?? 130,
                'crop_size'  => $params['crop_size']  ?? 200,
                'objective'  => $params['objective']  ?? '100x',
            ])->throw()->json();
    }
}
```

```php
// 큐로 비동기 처리 (몇 분 걸리는 작업)
class ProcessNd2Job implements ShouldQueue
{
    public function handle(PhenoPixelClient $client): void
    {
        $result = $client->extractCells($this->upload->path, $this->upload->params);
        $this->upload->update(['status' => 'done', 'db_name' => $result['db_name']]);
        UsageLog::create(['team_id' => $this->upload->team_id, 'cells' => $result['cell_count']]);
    }
}
```

### 언어별 비교

| 스택 | 난이도 | 추천도 | 이유 |
| --- | --- | --- | --- |
| React 프론트 개선 | ⭐ | 🟢🟢🟢🟢🟢 | 이미 React. 바로 시작 |
| PHP 래퍼 (하이브리드) | ⭐⭐ | 🟢🟢🟢🟢 | PHP 잘하면 최선 |
| Next.js 프론트 재작성 | ⭐⭐⭐ | 🟢🟢🟢 | SSR + API Routes, 백엔드는 Python 유지 |
| Node.js 백엔드 재작성 | ⭐⭐⭐⭐ | 🟡🟡 | ND2/ML이 여전히 문제 |
| PHP 전체 재작성 | ⭐⭐⭐⭐⭐ | 🔴 | ND2+ML+Cellpose 전부 재구현. 비추 |
| Rust/Go 백엔드 | ⭐⭐⭐⭐⭐ | 🔴 | 과학 생태계가 파이썬만 못함 |

**핵심 원칙**: 이미 잘 만들어진 건 그대로 쓰고, **없는 것(인증·결제·멀티테넌시·감사로그·협업)을 만들어서 수익화.**

---

## 14. 수익화 아이디어

### 법적 기반

MIT 라이선스이므로 상업 판매 / 수정 재배포 / **소스 비공개** / 특허 출원 / 브랜드 변경 모두 허용.
의무는 저작권 고지 + 라이선스 사본 포함뿐. (GPL이 아니라 수정 코드 공개 의무 없음)

의존성 라이선스도 전부 안전:

| 패키지 | 라이선스 |
| --- | --- |
| FastAPI, Pydantic, NumPy, Pillow, SQLAlchemy | MIT / BSD |
| opencv-python-headless, fastmcp | Apache 2.0 (고지 필요) |
| cellpose | BSD-3 |
| nd2 / nd2reader | BSD / MIT |
| matplotlib | PSF-like |

→ 배포 시 `THIRD_PARTY_LICENSES.txt`로 고지 모으기 권장.

**윤리적 권장**: README에 원저자 명시 · 원저자에게 이메일 통보 · 개선사항은 upstream PR로 기여 · 논문 인용 형식 유지. (원저자가 학회에서 추천해주면 최고의 마케팅)

---

### 아이디어 1 🥇 온프레미스 엔터프라이즈 라이선스

**핵심**: PhenoPixel에 인증·권한·감사 기능이 없음. 기업·병원·제약사는 이게 없으면 규정 위반으로 사용 불가.

```
PhenoPixel (무료) + 인증/SSO + 팀권한 + 감사로그 + 암호화 + SLA = 엔터프라이즈 제품
```

**타겟**

| 고객 | 이유 | 지불능력 |
| --- | --- | --- |
| 대학병원 / 임상검사실 | 환자 데이터 = 개인정보, 감사로그 법적 필수 | 💰💰💰💰 |
| 제약사 R&D | GxP, 21 CFR Part 11 준수 | 💰💰💰💰💰 |
| CRO (수탁연구기관) | 고객사별 데이터 격리 필수 | 💰💰💰💰 |
| 정부 연구기관 | 폐쇄망(에어갭) 운영 | 💰💰💰 |
| 바이오 스타트업 | 투자자 실사 보안 체크 | 💰💰 |

**개발 항목**

- P0 (필수): 인증(JWT + SAML/OIDC + MFA) · RBAC(Admin/Researcher/Viewer + 데이터셋별 권한) · 감사로그(append-only, CSV/JSON 내보내기) · **git-pull 엔드포인트 제거** · CORS 제한
- P1: 저장·전송 암호화(SQLCipher) · 세션 관리(유휴 로그아웃) · 백업/복구 · 오프라인 검증 라이선스 키(에어갭 대응)
- P2: 전자서명(21 CFR Part 11) · 무결성 체크섬 · LDAP/AD 연동 · 오프라인 업데이트 패키지

**가격**

| 플랜 | 가격 | 포함 |
| --- | --- | --- |
| Community | $0 | 오픈소스 그대로 (마케팅용) |
| Team | $3,000/년 | 사용자 10명, 인증+RBAC, 이메일 지원 |
| Enterprise | $15,000/년 | 무제한 사용자, SSO, 감사로그, SLA 99.9% |
| Regulated | $40,000+/년 | GxP 검증(IQ/OQ/PQ), 전자서명, 온사이트 교육 |
| 초기 구축비 | $5,000~20,000 | 일회성 |

**근거**: 대학원생 1명이 라벨링에 주 20시간 쓰는 인건비가 연 수천만원. 연 $15,000로 그걸 없애면 충분히 저렴.

**매출 시나리오**

```
1년차:  Team 5 × $3,000 + Enterprise 1 × $15,000 + 구축 3 × $8,000 ≈ $54,000
3년차:  Team 20 + Enterprise 8 + Regulated 2 + 구축/컨설팅        ≈ $320,000
```

**기간**: MVP 2~3개월 / v1.0 +2개월 / Regulated +4개월. 개발자 1~2명.

**장점**: 진입장벽 낮음(코드 존재) · 명확한 수요 · 높은 마진 · 반복 매출 · 락인 · **온프레미스라 인프라 비용 0**

---

### 아이디어 2 🥈 관리형 SaaS

**핵심**: 설치가 어려움(Python 3.14, OpenCV 시스템 라이브러리, legacy-peer-deps, Docker+Traefik+도메인+인증서). 연구자는 이걸 못 함.

**추가 개발 항목**

1. 멀티테넌시 (Organization → Project → Dataset, 테넌트별 격리 — SQLite 파일 분리 방식이 오히려 유리)
2. 오브젝트 스토리지 (ND2가 수백MB~수GB. S3/R2/GCS. 현재는 로컬 FS → 추상화 필요)
3. 작업 큐 + 워커 오토스케일링 (Celery/RQ/Dramatiq)
4. 사용량 측정 + 과금 (세포 수 / GB·시간 / 작업 수, Stripe 또는 토스페이먼츠)
5. 실시간 진행률 (WebSocket / SSE)
6. 결과 공유 링크 (읽기전용, 만료 설정)

**가격**

| 플랜 | 가격/월 | 포함 |
| --- | --- | --- |
| Free | $0 | 월 5 ND2, 500MB, 세포 1,000개, 워터마크 |
| Academic | $39 | 월 50 ND2, 20GB, 세포 50,000개, 사용자 3 |
| Lab | $149 | 월 300 ND2, 200GB, 무제한 세포, 사용자 10, 우선 큐 |
| Institution | $599 | 무제한, 사용자 50, SSO, 전용 워커 |
| Pay-as-you-go | $0.50/ND2 | 간헐적 사용자 |

**손익**

```
스토리지 (R2 200GB)  $3
컴퓨팅 (워커 2대)    $80
DB (관리형 PG)       $25
전송 (R2 egress 무료) $0
기타                  $20
─────────────────────────
고정비 ≈ $130/월
→ Lab 플랜 1곳($149)만 있으면 흑자
→ 고객 50곳: 매출 $7,450 - 비용 $800 = 순익 약 $6,650/월
```

> Cloudflare R2 권장: egress 무료 → ND2 다운로드 트래픽 비용 0

**리스크 & 대응**

| 리스크 | 대응 |
| --- | --- |
| 데이터 민감성 거부감 | 리전 선택, 암호화, SOC 2, "학습에 사용 안 함" 명시 |
| 대용량 업로드 | 멀티파트 + 재시도 + tus 프로토콜 |
| GPU 비용 (Cellpose) | GPU 워커 분리 + 추가 요금 |
| 콜드스타트 | 최소 1대 상시 가동 |

---

### 아이디어 3 🥉 분석용 MCP 서버 — 가성비 1등 💎

**핵심**: 현재 MCP는 "코드 읽기"만. "분석 실행" MCP를 만들면 AI 에이전트가 전체 분석을 자동화.

```
연구자: "이 폴더의 ND2 다 분석해서 대조군 대비 세포 길이가 유의하게 변한 조건 찾아줘"
   ↓
AI: list_nd2_files() → run_extraction() × N → get_cell_lengths()
    → 통계 검정(t-test/ANOVA) → 그래프 생성 → 리포트
   ↓
"조건 C에서 평균 길이 3.2μm → 4.8μm 증가 (p<0.001). A, B는 유의차 없음."
```

킬러 시나리오: **"지난주 실험 데이터 전부 분석해서 랩미팅 슬라이드 만들어줘"**

**만들 도구**

```python
# 파일 관리
list_nd2_files() / get_nd2_metadata(filename)

# 추출 (비동기)
run_extraction(filename, layer_mode="dual", param1=130, crop_size=200,
               objective="100x", auto_annotation=True) -> {"job_id": ...}
get_job_status(job_id)

# 데이터 조회
list_databases() / get_label_counts(db_name)

# 분석
get_cell_lengths(db_name, label="1")
get_cell_areas(db_name, label="1")
get_normalized_medians(db_name, label, channel)
get_heatmap_vectors(db_name, label)          # 35차원 벡터

# 비교 (최고 가치)
compare_populations(db_names, metric, label="1")   # 통계 검정 포함
```

**안전장치**

```python
# ① 파괴적 작업은 확인 필요
def delete_database(db_name, confirm=False):
    if not confirm: return {"error": "Set confirm=True"}

# ② 리소스 제한
MAX_CONCURRENT_JOBS = 2
MAX_CELLS_PER_QUERY = 10_000

# ③ 읽기/쓰기 분리 — guards.py 패턴 재사용
READ_ONLY_TOOLS = (...) ; MUTATING_TOOLS = (...)

# ④ 데이터 반환 제한 — raw 픽셀은 요약 통계만 (컨텍스트 폭발 방지)
def get_raw_data_summary(db_name, label):
    return {"mean": ..., "std": ..., "percentiles": {...}}
```

**수익 모델**: 오픈소스 + 유료 호스팅 $29~99/월 · 엔터프라이즈 번들 포함 · 사용량 과금 $0.10/작업 · 커스텀 MCP 구축 $5,000~15,000

**개발 기간: 2~4주** (REST API 55개가 이미 존재 → 얇은 래퍼 / FastMCP 패턴 존재 / guards.py 재사용 / 테스트 구조 존재)

→ **"생물이미지 분석 분야 최초의 분석 실행 MCP"** 포지션 선점 가능.

---

### 아이디어 4 도메인 확장 플러그인

현재는 박테리아(막대형) + ND2 특화. 확장 여지:

```
효모 (타원형, 출아 감지) / 고세균 / 동물세포 배양 / 조직 절편(Watershed) /
식물세포 / 바이오필름(3D)
```

**플러그인 아키텍처**

```python
class SegmentationBackend(Protocol):
    def segment(self, image: np.ndarray, params: dict) -> list[Contour]: ...

class CannyRodBacteria(SegmentationBackend): ...   # 현재 구현
class CellposeGeneric(SegmentationBackend): ...    # mother_machine에 이미 있음
class YeastBudDetector(SegmentationBackend): ...   # 신규 💰
class WatershedTissue(SegmentationBackend): ...    # 신규 💰
```

**파일 포맷 확장**: `.czi`(Zeiss, 수요 최대 — `aicspylibczi`/`pylibCZIrw`) · `.lif`(Leica) · `.oib/.oif`(Olympus) · OME-TIFF · `.lsm`

**가격**: Yeast Pack $1,500 / Mammalian $2,500 / Tissue $3,500 / Format Pack $2,000 / 번들 $8,000

---

### 아이디어 5 컨설팅 + 커스텀 파이프라인 — 즉시 시작 가능

소프트웨어가 아니라 "시간"을 판매. 연구실마다 실험이 달라 커스텀 수요가 항상 존재.

| 서비스 | 가격 | 기간 |
| --- | --- | --- |
| 설치 + 세팅 | $1,500 | 1~2일 |
| 온라인 교육 (2시간) | $500 | 반일 |
| 온사이트 워크샵 (1일) | $2,500 | 1일 |
| 파라미터 튜닝 + 검증 | $2,500 | 3~5일 |
| 커스텀 분석 모듈 1개 | $5,000~12,000 | 2~4주 |
| 논문용 분석 대행 | $3,000~10,000 | 2~6주 |
| 커스텀 ML 모델 학습 | $8,000~20,000 | 4~8주 |
| 유지보수 계약 | $1,000/월 | 연간 |

**장점**: 자본 0원 · 즉시 시작 · 빠른 현금흐름 · **시장 학습**(고객이 원하는 것을 직접 파악) · 제품 아이디어 발굴(3번 반복 요청 = 제품화 신호)

**고객 발굴**: GitHub Issues 답변 → 원본 저장소 기여 → bioRxiv 논문 저자 콜드메일 → ResearchGate/X 커뮤니티 → 학회 부스(ASM Microbe, 한국미생물학회) → 대학 산학협력단

---

### 아이디어 6 교육 콘텐츠 & 강의

| 상품 | 가격 |
| --- | --- |
| 전자책 "생물이미지 분석 실전" | $39 |
| 온라인 강의 (8시간) | $199 |
| Udemy/Coursera 코스 | $50 |
| 대학 강의 라이선스 | $3,000/학기 |
| YouTube (광고/스폰서) | 마케팅 겸용 |
| 부트캠프 (5일) | $2,500/인 |

**커리큘럼**: 현미경 이미지 기초 → OpenCV 윤곽선 → PCA 정렬 → 중심선 피팅 → 고정차원 특징 벡터 → ML 자동 라벨링 → FastAPI → React 뷰어 → **MCP로 AI 에이전트 연결** → Docker 배포/재현성

Module 9(MCP)가 차별점. "AI 에이전트 + 과학 분석"을 가르치는 강의는 거의 없음.

---

### 아이디어 7 논문 그림 자동 생성 (Figure Builder)

**페인포인트**: PNG 다운로드 → Illustrator → 폰트/축/패널 배치 → 저널 규정 확인 → 리비전 오면 처음부터.

```
Figure Builder
├── 드래그앤드롭 패널 배치 (A, B, C, D)
├── 저널 프리셋 (Nature/Science/Cell/PLOS/eLife) → 폰트·크기·여백·DPI 자동
├── 통계 표시 자동화 (*, **, ***, n.s.)
├── 스케일바 자동 삽입 (μm 정확)
├── 색맹 친화 팔레트 (Viridis, Cividis)
├── 벡터 내보내기 (SVG/EPS/PDF)
└── 버전 관리 (리비전 대응) ← 킬러 기능
```

**가격**: Free(워터마크/PNG만) · Pro $19/월 · Lab $99/월 · 제작 대행 $49/figure

---

### 아이디어 8 데이터셋 & 사전학습 모델 마켓

현재 AutoAnnotation은 520개 데이터 기반. 확장 판매 가능:

| 상품 | 가격 |
| --- | --- |
| 대장균 대형 데이터셋 (50,000 라벨) | $5,000 |
| 효모 데이터셋 (20,000) | $8,000 |
| 다배율 대응 모델 (40x/60x/100x) | $3,000 |
| 다양한 현미경 대응 모델 | $4,000 |
| 세포 분열 단계 분류 | $6,000 |
| 생사 판별 (PI 염색) | $5,000 |
| 종(species) 분류 | $10,000 |

**플랫폼**: PhenoModel Hub (허깅페이스 스타일) — 업로드/다운로드, 벤치마크 리더보드, 유·무료 혼합, 원클릭 배포, 수익 배분 70/30

**리스크**: 데이터 저작권 확인 필수 · 라벨링 비용(전문가 시간당 $50~100) → **Active Learning으로 1/5 절감** · 일반화 문제 → 도메인 적응/파인튜닝 기능 제공

---

### 아이디어 9 장비 제조사 OEM 파트너십

현미경 회사의 기본 소프트웨어(NIS-Elements, ZEN, LAS X)는 무겁고 UI가 구식.
"우리 현미경 사면 최신 웹 분석 도구도 함께" = 차별화.

**접근 경로**: 본체 제조사(Nikon/Zeiss/Leica/Evident) → 카메라 제조사(Hamamatsu/Andor/Photometrics/Teledyne) → **국내 대리점(우선 공략)** → 시약 회사(Thermo Fisher/Invitrogen) → 라이브셀 이미징(타임랩스 = Mother Machine 궁합)

| 계약 방식 | 금액 |
| --- | --- |
| OEM 라이선스 | $50,000~200,000/년 |
| 장비당 로열티 | $500~2,000/대 |
| 공동 개발 | $100,000+ |
| 화이트라벨 | 매출의 20~30% |
| 국내 대리점 리셀러 | 마진 30~40% |

→ 본사보다 **국내 대리점부터** 시작하는 것이 현실적. 성공 사례 후 본사로.

---

### 아이디어 10 구독형 "랩 OS" (장기 비전)

```
LabOS
├── ELN (전자 실험 노트)
├── 시료/균주 관리 (LIMS)
├── 장비 예약
├── 이미지 분석 ← PhenoPixel
├── 데이터 저장소 + 버전관리
├── 통계 분석 + 시각화
├── 논문 초안 AI 도우미
├── 협업 (코멘트/리뷰/승인)
└── AI 에이전트 (MCP로 전부 연결)
```

가격: Lab Starter $299/월 · Lab Pro $999/월 · Department $4,999/월 · Institution $20,000+/월

**리스크 높음**: Benchling(유니콘)·LabArchives·Dotmatics·eLabNext 등 강력한 경쟁자 · 팀 10명+ 2~3년 · 시리즈A($5M+) 필요 · 기관 단위 영업 난이도

→ 첫 사업이 아니라 아이디어 1·2·3을 하면서 수렴할 방향성.

---

### 종합 비교

| # | 아이디어 | 개발기간 | 초기비용 | 난이도 | 예상 ARR | 리스크 | 추천 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 온프레미스 엔터프라이즈 | 3~5개월 | 낮음 | 중 | $50K~500K | 낮음 | 🟢🟢🟢🟢🟢 |
| 2 | 관리형 SaaS | 6~9개월 | 중 | 높음 | $30K~500K | 중 | 🟢🟢🟢🟢 |
| 3 | 분석 MCP 서버 | **2~4주** | 매우낮음 | 낮음 | $10K~100K | 낮음 | 🟢🟢🟢🟢🟢 |
| 4 | 도메인 플러그인 | 2~4개월/개 | 낮음 | 중 | $20K~150K | 중 | 🟢🟢🟢 |
| 5 | 컨설팅 | **즉시** | **0원** | 낮음 | $50K~200K | 매우낮음 | 🟢🟢🟢🟢🟢 |
| 6 | 교육 콘텐츠 | 2~3개월 | 낮음 | 낮음 | $10K~80K | 낮음 | 🟢🟢🟢 |
| 7 | Figure Builder | 3~5개월 | 낮음 | 중 | $20K~200K | 중 | 🟢🟢🟢🟢 |
| 8 | 모델 마켓 | 6~12개월 | 높음 | 높음 | $20K~300K | 높음 | 🟢🟢 |
| 9 | OEM 파트너십 | 6~18개월 | 낮음 | 중 | $50K~500K | 중 | 🟢🟢🟢 |
| 10 | 랩 OS | 24~36개월 | 매우높음 | 매우높음 | $100K~5M | 매우높음 | 🟢 |

---

## 15. 실행 로드맵

### Phase 0 — 지금 당장 (0~1개월), 자본 0원

1. **원본 저장소 기여** — `App.tsx` 죽은 코드 정리 PR(30분) · 보안 이슈 리포트(git-pull, CORS) · 한국어 README 추가
   → 목적: 이름 알리기 + 신뢰 + 컨트리뷰터 이력
2. **GitHub Issues 답변** (원본에 3개 열려있음)
   → 목적: 전문성 인식
3. **컨설팅 영업 시작** (아이디어 5) — bioRxiv 논문 저자 리스트업 → 콜드메일 20통(설치 지원 무료 제공)
   → 목적: 첫 고객 + 시장 학습

### Phase 1 — 첫 제품 (1~3개월)

4. **분석용 MCP 서버** (아이디어 3) — 2~4주, 오픈소스 공개, 포지션 선점
   → 이유: 개발 빠름(REST 재활용) · 화제성 · 마케팅 효과 · 컨설팅 영업 무기
5. **보안 패치 버전** — 인증 + RBAC + 감사로그 MVP (아이디어 1의 씨앗)

### Phase 2 — 수익화 본격화 (3~9개월)

6. **온프레미스 엔터프라이즈 출시** (아이디어 1) — Phase 1에서 만난 고객에게 먼저 제안
7. **교육 콘텐츠** (아이디어 6) — YouTube 무료 → 유료 강의 유도

### Phase 3 — 확장 (9~24개월)

8. SaaS 또는 Figure Builder (고객 피드백으로 선택)
9. 플러그인 / OEM 파트너십

---

## 16. 핵심 조언 3가지

### ① 제품보다 고객이 먼저

```
❌ 1년간 완벽한 제품 → 출시 → 아무도 안 씀
✅ 컨설팅으로 고객 만나기 → 진짜 고통 파악 → 그것만 만들기
```

Phase 0의 컨설팅이 가장 중요. 여기서 배운 것으로 제품 방향을 결정.

### ② 오픈소스는 적이 아니라 마케팅

```
Community 무료 버전 = 무한한 마케팅 채널
  → 써보고 좋으면 → "우리 회사는 인증이 필요한데..." → Enterprise 구매
```

원본을 폐쇄적으로 포크하지 말고, 기여하면서 신뢰를 쌓을 것.

### ③ Compliance(규정 준수)가 진짜 돈

```
기술적으로 어려운 것 ≠ 비싸게 팔리는 것

세포 추출 알고리즘 (어려움)  → 이미 무료로 존재
감사 로그 (기술적으로 쉬움)  → 없으면 병원이 사용 불가 → 💰💰💰
```

**"어렵지만 돈 안 되는 것"보다 "쉽지만 없으면 안 되는 것"을 만들 것.**

---

## 17. 기술 스택 요약

**백엔드**: FastAPI 0.128 · Uvicorn 0.40 · Pydantic 2.12 · SQLAlchemy 2.0 · SQLite ·
NumPy 2.2 · opencv-python-headless 5.0 · Pillow 12.1 · Matplotlib 3.10 ·
nd2 0.11 / nd2reader 3.3 · cellpose 4.2.1.1 · **fastmcp 3.2.4** ·
aiofiles / aiohttp / aiosqlite / greenlet / python-multipart

**프론트엔드**: React 19 · React Router 7 · TypeScript 5.9 · Vite 7 · Chakra UI 3 ·
Emotion 11 · Framer Motion 12 · lucide-react · fflate ·
Storybook 8.6 · Playwright · Docusaurus 3.10 (+ mermaid, katex) · ESLint 9

**인프라**: Docker · Traefik (Let's Encrypt HTTPS 자동) · nginx.conf · Slack Webhook

---

## 18. 참고: 연구 인용 형식

```text
Ikeda, Y. PhenoPixel: microscopy single-cell extraction and batch phenotype analysis software.
GitHub repository. URL: https://github.com/ikeda042/PhenoPixel
(accessed <YYYY-MM-DD>), commit <commit-hash>.
```

Mother Machine 워크플로우 사용 시 Cellpose-SAM도 함께 인용:

> Pachitariu, M., Rariden, M., & Stringer, C. (2025). Cellpose-SAM:
> superhuman generalization for cellular segmentation. *bioRxiv*,
> 2025.04.28.651001. https://doi.org/10.1101/2025.04.28.651001

재현성을 위해 보고할 항목: OS/Python/Node 버전 · 의존성 스냅샷 · ND2 메타데이터/획득 조건 ·
추출 파라미터 · 라벨링 기준 · Bulk Engine 모드 · 임계값 · 내보내기 설정 · 정확한 커밋 해시

---

*이 문서는 PhenoPixel 저장소 전수조사(파일 1,176개 / 백엔드 15,835줄 / 프론트엔드 15,271줄) 결과입니다.*
