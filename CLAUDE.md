# CLAUDE.md — 매뉴얼 적재 규칙 (자동 실행)

이 저장소에서 사용자가 매뉴얼 파일(MD·PDF 등)을 주며 **"적재해줘"** 라고만 하면,
**확인 질문 없이 한 번에** 아래 규칙 그대로 Supabase DB에 적재한다.
매번 같은 기준(0단계 준비 → 6단계 벡터화)으로 동작해야 한다.

---

## 대상 인프라 (고정값)

- **Supabase 프로젝트**: `seoil-manual-chatbot`
  - project_id: `jokshzpthxiyweoyrawt` (region: ap-northeast-2)
- **대상 테이블**: `public.manual_chunks`
- **적재 수단**: Supabase MCP (`mcp__Supabase__execute_sql`)
- **검색 함수(참고)**: `match_manual_chunks(query_embedding vector, match_count int)` — 코사인 거리(`<=>`) 기반

### manual_chunks 스키마 (그대로 채운다)

| 컬럼 | 타입 | 채우는 값 |
|------|------|-----------|
| `id` | bigint identity | 자동 (넣지 않음) |
| `chunk_order` | integer | 같은 문서 내 조각 순서(1,2,3…) |
| `section_path` | text | 기존 분류 체계에 맞춰 부여 (아래 규칙 참고) |
| `chunk_type` | text | **`policy` / `note` / `table` 중 하나만** (CHECK 제약) |
| `content` | text | 구조화된 조각 본문 |
| `embedding` | vector(768) | Gemini 임베딩 768차원 (**반드시 함께 채운다**) |
| `created_at` | timestamptz | 자동(now()), 넣지 않음 |

---

## 절대 규칙 (매번 지킬 것)

1. **기존 행 삭제·수정 금지.** 오직 **INSERT(이어서 추가)**만 한다. UPDATE/DELETE/TRUNCATE 금지.
2. **임베딩 없이 텍스트만 넣지 않는다.** 모든 INSERT는 `content` + `embedding`(768차원)을 함께 채운다. 임베딩 생성이 안 되면 그 조각은 넣지 않고, 이유를 보고한다.
3. **사실·수치·고유명사 변경·창작 금지.** 허용되는 것은 **다듬기·묶기·나누기**뿐이다. 없는 내용을 만들거나, 숫자/명칭/조건을 바꾸지 않는다.
4. **표(table)는 통째로.** 표는 쪼개지 말고 하나의 조각으로 넣고 `chunk_type='table'`로 둔다.
5. **중복이면 건너뛴다.** 이미 DB에 있는 내용(같은 의미의 조각)은 넣지 않는다. (아래 중복 판정 참고)
6. **확인 질문 없이 한 번에** 끝까지 진행한다. 단, 아래 "중단 조건"에 해당하면 그때만 안내한다.

---

## 0단계 — 입력 파일 정규화(전처리)

들어온 파일 확장자에 따라 처리한다.

- **`.md`, `.pdf`** → 그대로 직접 읽어 1단계로 간다. (PDF는 Read 도구의 `pages`로 직접 읽는다.)
- **`.docx` / `.pptx` / `.xlsx` 등** →
  1. `markitdown` 설치 여부를 먼저 확인한다: `python -m markitdown --help` (또는 `markitdown --help`).
  2. 없으면 **필요한 확장자만** 지정해 설치한다 (`[all]` 전체 설치는 일부 환경에서 실패 가능하므로 쓰지 않는다):
     - 예) `pip install "markitdown[docx]"` · `pip install "markitdown[pptx]"` · `pip install "markitdown[xlsx]"`
  3. `markitdown 파일 -o 출력.md` 로 md 변환 후, 그 md를 1단계로 넘긴다.
- **한글 파일형식(`.hwp` / `.hwpx`)** → 변환 도구가 지원하지 않는다. **적재하지 말고**,
  "한글 파일은 지원되지 않으니 **Word(.docx) 또는 PDF**로 먼저 바꿔서 다시 주세요"라고 안내하고 멈춘다.

---

## 1단계 — 의미 단위로 구조화 (기계적 분할 금지)

- 파일을 **직접 읽어** 내용을 이해하고, **한 조각 = 한 주제**가 되도록 나눈다.
- **글자 수로 자르지 않는다.** 챗봇이 의미 검색(코사인 유사도)으로 정확히 찾아올 수 있는 자기완결적 단위로 만든다.
- 각 조각에 `chunk_type` 부여:
  - `policy` — 규정·절차·기준·지침 등 "규칙"에 해당하는 본문
  - `note` — 보충 설명·주의사항·안내·팁 등
  - `table` — 표. 표는 통째로 하나의 조각(마크다운 표로 보존).
- `chunk_order` 는 문서 내 등장 순서대로 1부터 부여한다.

## 2단계 — section_path 부여 (기존 체계에 이어서)

1. 먼저 **DB의 기존 분류 체계를 조회**한다:
   ```sql
   select distinct section_path from public.manual_chunks order by section_path;
   ```
2. 새 조각의 `section_path`는 위에서 나온 **기존 분류에 맞춰 이어서** 부여한다.
   (예: 기존이 `설치/전원`, `설치/배선` 형태면 같은 계층 표기법을 따른다.)
3. 정말 맞는 상위 분류가 없을 때만 같은 표기 규칙으로 새 경로를 만든다.

## 3단계 — 중복 판정

- 넣기 전, 같은 `section_path` 및 유사 `content`가 이미 있는지 확인한다.
  - 텍스트 동일/거의 동일하거나, 임베딩 코사인 유사도가 매우 높으면(≈0.97 이상) **중복으로 보고 건너뛴다.**
- 건너뛴 조각 수는 마지막 보고에 포함한다.

## 6단계 — Gemini 임베딩 생성 (벡터화)

- 각 조각의 `content`를 **6단계와 동일한 Gemini 임베딩 모델**로 벡터화해 **768차원** 벡터를 얻는다.
  - 모델: Gemini 임베딩 모델 중 **출력 768차원**인 것을 사용한다
    (`text-embedding-004`, 또는 `gemini-embedding-001` 에 `output_dimensionality=768`).
    `embedding vector(768)` 컬럼과 차원이 반드시 일치해야 한다.
  - 문서 적재이므로 검색용 문서 임베딩(task type: retrieval document)으로 생성한다.
- API 키는 환경변수(`GEMINI_API_KEY` 또는 `GOOGLE_API_KEY`)에서 읽는다. **키를 코드/커밋/CLAUDE.md에 하드코딩하지 않는다.**
- 호출 예 (Google Generative Language REST):
  ```bash
  curl -s "https://generativelanguage.googleapis.com/v1beta/models/text-embedding-004:embedContent?key=$GEMINI_API_KEY" \
    -H 'Content-Type: application/json' \
    -d '{"content":{"parts":[{"text":"<조각 본문>"}]},"outputDimensionality":768}'
  # 응답의 embedding.values (768개 float) 를 사용
  ```

## 7단계 — 적재 (Supabase MCP, INSERT only)

- 조각별로 `mcp__Supabase__execute_sql` 로 INSERT 한다. 벡터는 pgvector 리터럴 `'[v1,v2,...,v768]'::vector` 로 넣는다.
  ```sql
  insert into public.manual_chunks (chunk_order, section_path, chunk_type, content, embedding)
  values (1, '설치/전원', 'policy', $$...본문...$$, '[...768개...]'::vector);
  ```
- `content` 는 `$$ ... $$` 달러 인용으로 넣어 따옴표/개행 문제를 피한다.

## 8단계 — 보고

- 끝나면 **몇 조각이 추가됐는지**(그리고 중복으로 건너뛴 수, 파일별 요약)를 한 줄로 보고한다.

---

## 중단 조건 (이때만 사용자에게 안내)

- 한글 파일형식(.hwp/.hwpx)이 들어옴 → Word/PDF 변환 요청 안내 후 중단.
- Gemini API 키가 환경에 없음 → 임베딩을 만들 수 없으므로, 키 설정을 요청하고 해당 파일은 적재하지 않음.
- 그 외에는 질문 없이 규칙대로 끝까지 진행한다.
