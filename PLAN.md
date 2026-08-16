# local-knowledge-graph 계획

> 작성일: 2026-08-11 · 상태: **설계 확정, 착수 전(미결정 2건 있음)**

비정형 한국어 문서를 로컬 LLM으로 읽어 지식 그래프를 만들고, 브라우저에서 탐색하는 도구.
문서가 로컬 장비 밖으로 나가지 않는 것이 전제다.

---

## 0. 결정 사항

| 항목 | 결정 |
|---|---|
| 용도 | **문서 탐색·이해** (질의응답/RAG는 범위 밖. 단, 나중에 올릴 수 있게 스키마만 열어둠) |
| 구현 방식 | **직접 구현.** 업스트림([ai-knowledge-graph](https://github.com/robert-mcdermott/ai-knowledge-graph))은 참고만 |
| 추론(inference) 패스 | **넣지 않는다.** 원문에 없는 추정이라 탐색 용도와 상충 |
| 그래프 저장소 | SQLite. 문서 수천 개 규모까지 충분하고, 별도 그래프 DB는 과함 |
| 대상 언어 | 한국어 우선 |

---

## 1. 환경 실측 (2026-08-11 기준)

```
하드웨어    Apple M2 Pro / 32GB
Ollama      0.32.3 (서버 실행 중, 설치된 모델 0개)
uv          설치됨 (/opt/homebrew/bin/uv)
python3     3.9.6 (시스템) → uv로 3.12 환경 별도 구성 필요
```

**함의**: 32GB이므로 12~14B급으로 타협할 이유가 없다. `qwen3:30b-a3b-instruct`(19GB, MoE라 30B급 품질에 3B급 속도)가 1순위. 아직 아무 모델도 받지 않았으므로 첫 작업은 `ollama pull`.

---

## 2. 업스트림 조사 결과

`robert-mcdermott/ai-knowledge-graph` (Apache 2.0, ★2833, 마지막 푸시 **2025-12-28** — 8개월 정체, 오픈 이슈 13).

### 2-1. 가져올 것

- 3단 패스 구조(추출 → 표준화 → 추론) 중 **추출·표준화 개념**
- 트리플(SPO) 스키마와 프롬프트 구성 방식
- 시각화 레이아웃 아이디어 (`src/knowledge_graph/templates/graph_template.html`)

### 2-2. 코드 대조로 확인한 사실 (인계 문서에 틀리게 적혀 있던 것들)

| 떠도는 설명 | 실제 코드 |
|---|---|
| 표준화 패스가 고유 엔티티 **전체**(201개)를 한 번에 LLM에 넘긴다 | `entity_standardization.py`가 **100개 초과 시 빈도 상위 100개만** 추려 보낸다. 입력이 무한정 커지지 않음 |
| `config.toml` 기본값 `chunk_size=200`, `temperature=0.2` | 저장소 실제 파일은 `chunk_size=100`, `temperature=0.8`. 게다가 `[visualization]` 섹션이 별도로 있음 (200/0.2는 README 예시값) |
| 프롬프트 위치가 "버전에 따라 `prompts.py` 또는 `prompts/`" | 현재 main은 `src/knowledge_graph/prompts/` 디렉토리 확정 (`main_prompts.py`, `entity_prompts.py`, `inference_prompts.py`) |

`num_ctx` 확장이 필요하다는 결론 자체는 맞지만 **이유가 다르다.** 진짜 이유는 `max_tokens = 8192`인데 Ollama 기본 `num_ctx`가 4096이고, Ollama는 입력+출력이 같은 창을 공유하므로 잘린다는 점이다. OpenAI 호환 엔드포인트로는 `num_ctx`를 넘길 수 없어 Modelfile 또는 `OLLAMA_CONTEXT_LENGTH` 환경변수를 써야 한다는 것도 맞다.

**확인된 통계** (README, 산업혁명 예제): 초기 216 트리플 / 13 청크 → 표준화 후 엔티티 201→160 → 최종 564 트리플(**원본 209, 추론 355**), 161 노드 9 커뮤니티. 추론이 원본보다 많다.

**추출 프롬프트에 하드코딩된 제약** (`main_prompts.py`):
- `"All relationships (predicates) MUST be no more than 3 words maximum. Ideally 1-2 words."`
- `"Make all the text of S-P-O text lower-case, even Names of people and places."`

→ 한국어에 lower-case는 무의미하지만 **영어 출력을 유도하는 압력**으로 작용한다. 그대로 쓰면 한국어 문서에서 영어 엔티티가 섞여 나와 표준화가 깨진다.

### 2-3. 그대로 쓰지 않는 이유

1. **출처 추적(provenance)이 없다.** 트리플이 원문 어느 문장에서 나왔는지 역추적 불가. 로컬 LLM 추출은 환각이 섞이는 게 정상이고, 근거를 못 짚으면 그래프를 신뢰할 수 없다. **가장 큰 이유.**
2. **증분 갱신이 없다.** 문서 하나 추가마다 전체 재계산. 로컬 LLM 호출 비용상 실사용 불가로 직결.
3. **표준화가 상위 100개 휴리스틱.** 문서가 늘수록 롱테일이 방치된다. 한국어 표기 변이(띄어쓰기·약칭·한자 병기)가 롱테일에 몰릴수록 효과가 떨어진다.
4. **질의 불가.** 결과가 HTML 한 장이라 눈으로 보는 것만 된다.
5. 8개월 정체 상태라 포크해도 유지보수는 결국 우리 몫.

---

## 3. 설계

### 3-1. 파이프라인

```
문서 로드 → 청킹 → 추출(structured output) → 정규화·표준화 → SQLite → 시각화 HTML
                      ↑ 청크 해시로 기처리분 skip (증분)
```

### 3-2. 직접 구현으로 얻는 것 (설계 핵심 3가지)

**(a) 파싱 실패율 문제를 구조적으로 제거**
업스트림은 LLM 응답 텍스트에서 JSON을 정규식으로 긁어내므로 작은 모델에서 실패가 잦다. Ollama 0.32.3은 **structured outputs**(JSON 스키마 강제)를 지원하므로, 스키마를 넘기면 디코딩 단계에서 형식이 보장된다. "파싱 실패 20% 넘으면 큰 모델로 교체" 같은 검증 항목 자체가 사라진다.

**(b) provenance를 1급 데이터로**
트리플마다 `doc_id / chunk_id / 근거 문장`을 함께 저장. 그래프에서 엣지를 클릭하면 근거 문장이 뜨는 구조. 탐색·이해 용도에서는 시각화 완성도보다 이쪽이 중요하다.

**(c) 증분 갱신**
`문서 해시 + 청크 해시`를 키로 이미 추출한 청크는 건너뛴다.

### 3-3. 한국어 기본값

| 항목 | 방침 |
|---|---|
| 청킹 | 단어(공백) 수가 아니라 **문자 수 + 문장 경계 유지**. 한국어는 공백 기준이 정보량을 왜곡함 |
| 출력 언어 | 프롬프트에 "엔티티·술어는 입력 문서의 언어를 유지" 명시 |
| 술어 제약 | "3단어 이내" 제거 → 한국어 서술형 어구 허용 (조사·어미 때문에 원 기준이 부적합) |
| lower-case 강제 | 제거 |
| 표준화 | 상위 N개 휴리스틱 대신, 표기 변이를 정면으로 다루는 방식 검토 (3-5 참고) |

### 3-4. SQLite 스키마 (초안)

```
documents (doc_id, path, sha256, ingested_at)
chunks    (chunk_id, doc_id, seq, text, sha256)
triples   (triple_id, chunk_id, subject, predicate, object, evidence_sentence, created_at)
entities  (entity_id, canonical_name)
aliases   (alias, entity_id)        -- 표기 변이 → 정규 엔티티 매핑
```

`entities`/`aliases`를 분리해 두면 표준화를 되돌릴 수 있고, 원본 트리플은 손대지 않은 채 유지된다. 나중에 질의 계층을 올릴 때도 이 형태가 유리하다.

### 3-5. 미확정 설계 항목

- 엔티티 표준화 알고리즘: LLM 일괄 처리 vs 임베딩 유사도 vs 규칙(띄어쓰기 정규화) 조합. **3단계 실측 이후 결정.**
- 시각화에서 근거 문장 패널 UI 형태.

---

## 4. 의존성 (**승인 대기**)

전역 지침상 라이브러리 추가는 사전 확인이 필요하다. 최소 구성:

| 패키지 | 용도 | 대안 |
|---|---|---|
| `ollama` | Ollama 파이썬 클라이언트 (structured outputs) | `httpx` 직접 호출 — 의존성 하나 줄지만 코드가 늘어남 |
| `networkx` | 커뮤니티 탐지, 중심성 등 그래프 분석 | 없으면 시각화만 가능 |
| `pyvis` | 인터랙티브 HTML 생성 | 자체 HTML 템플릿 + vis-network. **근거 문장 패널을 붙이려면 오히려 이쪽이 편할 수 있음** |

`sqlite3`, `tomllib`는 표준 라이브러리라 추가 없음.

---

## 5. 실행 계획

- [ ] **1. 모델 준비** — `ollama pull qwen3:30b-a3b-instruct`. 필요 시 Modelfile로 `num_ctx` 상향
- [ ] **2. 스캐폴딩** — uv로 python 3.12 환경, 프로젝트 구조, SQLite 스키마
- [ ] **3. 추출 패스만 구현 + 실측** — 한국어 샘플 문서 1개로 트리플 품질·처리 시간·엔티티 언어 일관성 확인
- [ ] **4. 조정** — 3의 결과로 청크 크기·프롬프트 확정
- [ ] **5. 표준화 패스** — 알고리즘은 3-5에서 미확정. 4 이후 결정
- [ ] **6. 시각화** — 엣지 클릭 시 근거 문장 노출
- [ ] **7. 증분 갱신** — 해시 기반 skip

3번까지 가서 **실제 숫자를 보고** 나머지를 정한다. 한 번에 다 만들지 않는다.

---

## 6. 미결정 — 착수 전 확인 필요

1. **의존성 3개**(`ollama`, `networkx`, `pyvis`) 그대로 진행할지
2. **3단계 실측용 한국어 샘플 문서** — 직접 제공할지, 임의 생성할지.
   실제 사용할 문서 종류(회의록 / 조사 보고서 / 위키 등)와 비슷해야 청크 크기·프롬프트 조정이 의미가 있다.

---

## 7. 참고

- 업스트림 저장소: https://github.com/robert-mcdermott/ai-knowledge-graph
- 라이브 데모(산업혁명 그래프): https://robert-mcdermott.github.io/ai-knowledge-graph/
- 국내 소개 글: https://discuss.pytorch.kr/t/ai-knowledge-graph/11583
