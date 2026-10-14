# Multi-DB RAG 시스템 (Supabase + LangChain + Gemini)

PDF, TXT, CSV 등 서로 다른 형태의 데이터를 **Supabase 한 곳에 세 가지 DB(RDB, VectorDB, GraphDB)로 나누어 저장**하고,
질문이 들어오면 **Router(LLM)가 질문에 맞는 DB를 골라** 그 DB에서만 근거를 가져와 답변하는 RAG 시스템입니다.

## 시스템 구조

```
[Step 1. Data Indexing]
PDF / TXT ──(분할 → 임베딩)──▶ document_vectors   (VectorDB, pgvector 384차원)
CSV       ───────────────────▶ structured_data    (RDB)
Triples   ───────────────────▶ graph_nodes / graph_edges (GraphDB)

[Step 2. Retrieval & Generation]
질문 ─▶ Router (rdb | vectordb | graphdb) ─▶ 해당 DB 검색 ─▶ LLM(Gemini / Groq) ─▶ 답변
```

| 데이터 종류 | 저장 위치 | 질문 예시 |
|---|---|---|
| 문서 (PDF, TXT) | VectorDB | "Supabase의 장점을 설명해줘" |
| 표 (CSV) | RDB | "프로젝트들의 value 합계는?" |
| 관계 (Triples) | GraphDB | "Project Alpha와 연결된 기술은?" |

## 사용 기술

- **Supabase** (PostgreSQL + pgvector): RDB, VectorDB, GraphDB를 한 곳에서 관리
- **LangChain**: 문서 로더, 텍스트 분할 (`RecursiveCharacterTextSplitter`), LLM 연결
- **Hugging Face 임베딩**: `paraphrase-multilingual-MiniLM-L12-v2` (한국어 지원, 384차원)
- **Google Gemini** (`gemini-3.5-flash`), **Groq** (선택): 라우팅과 답변 생성
- **Google Colab**: 실행 환경

## 실행 방법

1. [Google Colab](https://colab.research.google.com)에서 **File → Upload notebook**으로 `RAG_Supabase_Topshiriq.ipynb`를 엽니다.
2. **Runtime → Run all**을 누릅니다.
3. 입력창이 나오면 다음을 차례로 입력합니다. (입력한 키는 노트북 파일에 저장되지 않습니다.)
   - Supabase Project URL
   - Supabase `service_role` (또는 secret) 키
   - Gemini API 키 (Groq 키는 선택)
4. 테이블이 없으면 노트북이 **SQL과 링크를 출력하고 멈춥니다.** 링크를 열어 SQL을 Supabase SQL Editor에서 한 번 실행한 뒤, **Run all**을 다시 누릅니다.
5. 인덱싱이 끝나면 3개의 테스트 질문이 자동으로 실행됩니다. 마지막 셀의 `QUESTION`을 바꿔 직접 질문할 수도 있습니다.

## 노트북 구성

| 단계 | 내용 |
|---|---|
| 1 | 라이브러리 설치 |
| 2 | 키 입력 (URL 자동 정리 포함) |
| 3 | Supabase 연결 및 테이블 확인 (없으면 SQL 안내) |
| 4 | 임베딩 모델과 LLM 준비 |
| 5 | 샘플 데이터 생성 (CSV, TXT, PDF) |
| 6 | 인덱싱: 3개 DB에 저장 (재실행 시 중복 방지를 위해 먼저 초기화) |
| 7 | Router + Retrieval + Generation |
| 8 | 테스트 3종 |
| 9 | 자유 질문 |

## 실행 결과

| 테스트 | Router 선택 | 결과 |
|---|---|---|
| Supabase 장점 (문서) | **VECTORDB** | 문서 내용(pgvector, VectorDB와 RDB 역할 겸용)에 근거해 답변 |
| 프로젝트 목록과 value 합계 | **RDB** | Alpha 15000, Beta 23000, Gamma 8500, 합계 **46500** |
| Project Alpha의 기술과 Supabase 확장 | **GRAPHDB** | LangChain, Supabase 연결, Supabase → pgvector |

## n8n 워크플로우와의 관계

수업의 n8n RAG 워크플로우(Google Drive → Supabase Vector Store → AI Agent)를 Python으로 옮기고 확장한 구현입니다.

| n8n | 이 프로젝트 |
|---|---|
| Download file / Default Data Loader | `PyPDFLoader`, `TextLoader`, `pd.read_csv` |
| Recursive Character Text Splitter | `RecursiveCharacterTextSplitter` |
| Embeddings Google Gemini | Hugging Face 다국어 임베딩 (384차원) |
| Supabase Vector Store | `document_vectors` 테이블 + `match_documents` 함수 |
| AI Agent + Tool | **Router**가 3개 DB 중 하나를 선택 |
| Gemini Chat Model | Gemini (`gemini-3.5-flash`) |

n8n 워크플로우는 벡터 DB 하나만 사용하지만, 이 프로젝트는 RDB, VectorDB, GraphDB 세 가지를 Router로 연결한 점이 확장된 부분입니다.

## 보안

- API 키는 코드나 파일에 저장하지 않고 실행할 때 입력합니다.
  
