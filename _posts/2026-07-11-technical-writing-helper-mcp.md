---
layout: post
title: "빈 화면 앞의 막막함을 MCP로 옮기기 — Technical Writing Helper 개발기"
tags:
  - MCP
  - Python
  - 테크니컬라이팅
  - docs-as-code
  - 프로젝트
---

테크니컬 라이팅에서 가장 어려운 순간은 문장을 다듬는 때가 아닙니다. **빈 화면 앞에서 구조를 잡는 순간**입니다. 어떤 타입의 문서인지, 어떤 섹션이 필요한지, 각 섹션이 무엇에 답해야 하는지 — 이 뼈대가 서기 전까지는 한 줄도 나아가지 못합니다. 반대로 뼈대만 서면 초안은 놀랄 만큼 빨리 채워집니다.

요즘은 초안을 LLM이 곧잘 씁니다. 그런데 LLM에게 "가이드 문서 하나 써줘"라고 하면, 그럴듯하지만 타입이 애매하고 구조가 매번 다른 글이 나옵니다. LLM은 글은 잘 쓰지만 **검증된 문서 구조·팀 고유 서식·결정적(deterministic) 판정** 같은 건 잘 못합니다. 그 간극을 메우려고 MCP 서버를 하나 만들어 깃허브에 올렸습니다. 이름은 [Technical Writing Helper](https://github.com/annayoon/Technical-Writing-Helper)(`twh`)입니다.

포지셔닝을 한 문장으로 정리하면 이렇습니다. **"글은 Claude가 쓰고, 구조·재료·출처 추적은 서버가 한다."** 이 역할 분담이 이 프로젝트의 거의 모든 설계 결정을 지배했습니다.

## 왜 프롬프트가 아니라 MCP 서버인가

"좋은 목차 잡아주는 프롬프트" 하나면 되지 않나 싶을 수 있습니다. 실제로 해보면 세 가지가 안 됩니다.

- **결정성**: 같은 입력에 매번 다른 목차가 나옵니다. 문서 타입 판정이 LLM의 그날 기분에 좌우됩니다.
- **상태**: 작년 보고서를 참조로 등록해두고 올해 초안에 쓰는 흐름이, 대화가 끊기면 매번 리셋됩니다.
- **재사용**: 개인 Claude에서 잡은 규칙을 CI(docs-as-code) 파이프라인에서 똑같이 돌릴 방법이 없습니다.

MCP(Model Context Protocol)는 이 셋을 정면으로 해결합니다. 규칙은 서버 코드에 결정적으로 박아두고, 상태는 서버가 파일로 들고 있고, 같은 코어를 여러 입구로 서빙할 수 있습니다. 그래서 프롬프트가 아니라 서버로 갔습니다.

## 아키텍처: 코어 하나, 얇은 프론트엔드 여럿

가장 먼저 정한 뼈대는 **코어 라이브러리 + 얇은 프론트엔드**입니다. 판정·목차·검색 로직은 전부 `outline` / `references` / `doc_types` / `storage` 모듈에 있고, MCP 서버(`server.py`)와 CLI(`cli.py`)는 그걸 얇게 노출하는 껍데기입니다.

```
                    ┌─────────────┐
 Claude (MCP) ─────▶│  server.py  │─┐
                    └─────────────┘ │
                    ┌─────────────┐ │   ┌──────────── core ───────────┐
 CI / docs-as-code ▶│   cli.py    │─┼──▶│ outline · references ·      │
                    └─────────────┘ │   │ doc_types · storage · extract│
                    ┌─────────────┐ │   └──────────────────────────────┘
 DocPortal 위키   ─▶│ import twh  │─┘
                    └─────────────┘
```

덕분에 같은 규칙이 세 입구에서 똑같이 돕니다. 개인은 Claude Desktop/Code의 MCP로, CI는 CLI로, 전사 [문서 포털(DocPortal)](https://github.com/annayoon/docportal)은 코어를 직접 `import`해서 위키 에디터의 "구조 잡기" 패널로. 나중에 원격 HTTP 서버로 전환할 때도 코어는 그대로 두고 껍데기만 바꾸면 됩니다.

`server.py`가 얼마나 얇은지는 툴 정의만 봐도 보입니다. 로직이 없습니다.

```python
@mcp.tool()
def suggest_outline(doc_type: str, topic: str, audience: str, reader_goal: str) -> dict:
    types = _types()
    if doc_type not in types:
        return {"ok": False, "error": f"알 수 없는 문서 타입: {doc_type}",
                "available": sorted(types.keys())}
    return outline_mod.suggest(types[doc_type], topic=topic,
                               audience=audience, reader_goal=reader_goal)
```

## 인터뷰를 프롬프트가 아니라 '툴 시그니처'로 강제하기

이 프로젝트에서 가장 마음에 드는 결정입니다. LLM에게 "먼저 독자를 물어보고 쓰세요"라고 지시문으로 부탁하면, 급할 때 건너뜁니다. MCP 서버는 대화를 주도할 수 없으니, 인터뷰를 **파라미터로 강제**했습니다.

`classify_document`는 `reader_purpose`와 `audience`를 필수로 받습니다. 그리고 `reader_purpose`가 정해진 enum이 아니면, 판정 대신 **되물을 질문 목록**을 돌려줍니다.

```python
def classify(doc_types, reader_purpose, audience, topic=""):
    if reader_purpose not in VALID_PURPOSES:
        return {
            "ok": False,
            "error": "reader_purpose가 유효하지 않습니다. 독자에게 "
                     "'이 문서를 읽고 나서 무엇을 할 수 있어야 하나요?'를 물어본 뒤 고르세요.",
            "valid_purposes": [...],  # learn-by-doing | accomplish-task | ...
        }
    primary = [t for t in doc_types.values() if reader_purpose in t.reader_purposes]
    ...
```

즉 인터뷰를 건너뛰고 바로 `classify_document`를 부르면 서버가 "그건 유효한 값이 아니니 사용자에게 물어보라"고 되받아칩니다. LLM 입장에서는 인터뷰 없이는 다음 단계로 갈 방법이 없습니다. **하지 말라고 부탁하는 대신, 안 하면 진행이 안 되게** 만든 것입니다.

## 판정은 규칙 기반, 목차는 스키마 자산

문서 타입 판정은 LLM의 감이 아니라 서버의 규칙으로 합니다. 기준 축은 딱 하나 — **"독자가 읽고 나서 무엇을 할 수 있어야 하는가"**(`reader_purpose`)입니다. `learn-by-doing`이면 tutorial, `accomplish-task`면 how-to, `report-official`이면 공문·보고서 식으로, 목적 → 타입 매핑을 서버가 들고 있습니다. 같은 입력엔 항상 같은 판정이 나옵니다.

내장 타입은 Diátaxis 4종(tutorial / how-to / reference / explanation)에 README, 릴리스 노트, ADR, 트러블슈팅, API 레퍼런스, 그리고 한국 실무에 꼭 필요한 **공문·보고서(관공서/대내)**까지 얹었습니다. 각 타입은 YAML 한 장으로 정의되는데, 사실상 이 스키마가 이 프로젝트의 핵심 자산입니다. 섹션마다 제목뿐 아니라 **"이 섹션이 답해야 할 질문"**과 분량 가이드가 붙습니다.

```yaml
- title: 소요 예산 및 일정
  goal: 자원과 시간 계획을 명시한다.
  questions:
    - 예산의 산출 근거는 무엇인가?
    - 단계별 일정과 책임 주체는 명확한가?
  length_hint: 표 권장
  optional: true
```

`suggest_outline`은 이 섹션들을 그대로 목차로 돌려주고, 각 질문이 곧 라이터에게 던지는 체크리스트가 됩니다. 팀 고유 타입은 `save_team_template`로 같은 스키마로 추가하면 내장 타입과 똑같이 동작합니다(같은 `id`면 팀 템플릿이 우선). 내장 타입은 패키지 안에, 팀 템플릿은 프로젝트의 `.twmcp/`에 — 코드와 데이터를 갈라놓은 것도 의도입니다.

## 한국어 검색: 형태소 분석기 없이 BM25 + 바이그램

참조 문서 기반 초안이 두 번째 축입니다. 여기서 서버의 몫은 명확합니다 — **수집·검색·출처 추적**. 글은 여전히 Claude가 씁니다.

검색은 임베딩 대신 BM25를 자체 구현했습니다. MVP에서 무거운 의존성 없이 실용적인 정확도를 내는 게 목표였거든요. 문제는 한국어입니다. 공백 단위로 토큰을 자르면 조사가 붙은 "예산의"와 "예산을"이 서로 다른 토큰이 되어 매칭이 무너집니다. 형태소 분석기(KoNLPy 등)를 쓰면 정확하지만, 무거운 설치 의존성이 따라옵니다.

타협점은 **문자 바이그램**이었습니다. 영문은 단어 토큰으로, 한글은 2글자씩 슬라이딩하며 토큰으로 만듭니다.

```python
_WORD_RE = re.compile(r"[a-z0-9_]+")
_HANGUL_RE = re.compile(r"[가-힣]+")

def tokenize(text: str) -> list[str]:
    text = text.lower()
    tokens = _WORD_RE.findall(text)
    for run in _HANGUL_RE.findall(text):
        if len(run) == 1:
            tokens.append(run)
        else:
            # "예산안" → "예산", "산안"
            tokens.extend(run[i:i+2] for i in range(len(run) - 1))
    return tokens
```

"예산안"은 `예산`·`산안`으로 쪼개지므로 "예산"이 들어간 질의와 겹칩니다. 형태소 분석기만큼 정교하진 않지만, 조사 변화에 상당히 강인하고 의존성이 0입니다. 그 위에 정석 BM25 랭킹을 얹었습니다.

```python
idf = math.log(1 + (n - df + 0.5) / (df + 0.5))
denom = tf + k1 * (1 - b + b * len(doc) / avg_len)  # k1=1.5, b=0.75
scores[i] += idf * tf * (k1 + 1) / denom
```

중요한 건 인터페이스입니다. 검색은 `ReferenceLibrary.search()` 뒤에 숨어 있어서, 나중에 임베딩 검색으로 갈아끼워도 호출부는 안 바뀝니다. `draft_section`이 반환하는 근거 발췌의 단위는 `ref_id#chunk_index`인데, 이게 그대로 **출처 표기 단위**가 됩니다. 초안 문단 끝에 `[ref: 작년보고서#3]`처럼 남기라고 지침을 함께 돌려줘서, LLM이 근거 없이 지어내는 걸 구조적으로 억제합니다.

## HWPX를 의존성 0으로 읽기

한국 공공·기업 문서의 현실은 한글 파일입니다. 작년 보고서(`.hwpx`)를 참조로 등록하고 올해 초안을 뽑는 흐름이 자연스럽게 성립하려면 hwpx를 읽어야 합니다. 그런데 여기서 발견이 하나 있었습니다 — **hwpx는 개방 포맷(OWPML, KS X 6101)이라 그냥 ZIP + XML**입니다. 한컴오피스도, 외부 라이브러리도 필요 없습니다.

```python
def _extract_hwpx(path: Path) -> str:
    paragraphs = []
    with zipfile.ZipFile(path) as zf:
        section_names = sorted(
            n for n in zf.namelist()
            if re.fullmatch(r"Contents/section\d+\.xml", n)
        )
        for name in section_names:
            root = ElementTree.fromstring(zf.read(name))
            # 네임스페이스 URI가 버전마다 달라서 로컬 이름(<hp:p>, <hp:t>)으로 매칭
            for para in root.iter():
                if para.tag.rsplit("}", 1)[-1] != "p":
                    continue
                runs = [t.text or "" for t in para.iter()
                        if t.tag.rsplit("}", 1)[-1] == "t"]
                text = "".join(runs).strip()
                if text:
                    paragraphs.append(text)
    return "\n\n".join(paragraphs)
```

한 가지 함정은 네임스페이스였습니다. hwpx 버전에 따라 XML 네임스페이스 URI가 달라져서 `{네임스페이스}p`로 정확히 매칭하면 어떤 파일은 잡히고 어떤 파일은 안 잡힙니다. 그래서 태그의 **로컬 이름**(`}` 뒤 부분)만 보고 `p`·`t`를 찾도록 했습니다. 버전이 뭐든 문단(`p`)과 텍스트 런(`t`)만 긁어오면 되니까요.

`.md/.txt/.rst`와 `.hwpx`는 이렇게 표준 라이브러리만으로 처리하고, PDF·DOCX·구형 `.hwp`만 선택 의존성(`pip install 'twh[docs]'`)으로 뺐습니다. 설치 마찰을 최소화하는 방향입니다. 참고로 **쓰기는 하지 않습니다** — 바이너리 `.hwp` 생성은 오픈소스로 불안정하고, 관공서 서식은 백지 생성보다 "기존 서식의 빈칸 채우기"가 실무와 맞기 때문입니다. 안 되는 걸 되는 척하지 않는 게 도구의 신뢰성이라고 생각합니다.

## 상태는 왜 `.twmcp/` 파일로 두는가

MCP 서버는 재시작되면 메모리가 날아갑니다. 그래서 세션을 넘겨 유지할 상태 — 참조 인덱스, 추출된 본문, 팀 템플릿 — 는 전부 프로젝트 루트의 `.twmcp/`에 파일로 저장합니다. 루트는 `TWH_PROJECT_ROOT` 환경변수로 지정합니다.

```
.twmcp/
  references/
    index.json       # 참조 메타데이터
    <id>.txt         # 추출된 본문
  templates/*.yaml   # 팀 문서 타입
```

`.twmcp/`에는 회사·개인 데이터가 들어가므로 **`.gitignore`로 커밋을 막습니다.** 레포에는 범용 코드만, 서식·용어집·참조 인덱스 같은 조직 데이터는 프로젝트 상태로. 전사 사용 시 회사 자료가 개인 레포로 새어 들어가는 사고를 구조적으로 막는 장치입니다. 참조 `id`를 만들 때도 한글은 유지하되 파일명으로 안전하게 슬러그화하고, 같은 이름이 있으면 `-2`를 붙여 충돌을 피합니다.

## 세 입구, 하나의 코어 — 실제 통합

이 구조의 값어치는 통합에서 드러났습니다. DocPortal 위키 에디터의 "구조 잡기" 패널은 MCP를 거치지 않고 코어를 직접 `import`합니다(`GET /wiki/outline` → `twh.outline`). 문서 타입·주제·독자·목표를 입력하면 작성 가이드 주석이 달린 목차 뼈대가 본문에 채워집니다. CLI로는 CI에서 이렇게 씁니다.

```bash
twh outline how-to --topic "SSO 연동" \
    --audience "사내 개발자" --goal "SSO 로그인 붙이기"   # 목차 스켈레톤
twh ref add ./작년보고서.hwpx --title "작년 보고서"       # 참조 등록
twh ref search "예산 산출 근거"                          # BM25 검색
```

같은 판정·같은 목차·같은 검색이 Claude에서도, 포털에서도, CI에서도 동일하게 나옵니다. 코어를 한 번 잘 만들어두니 입구는 얼마든지 늘릴 수 있었습니다.

## 마치며

돌아보면 이 프로젝트에서 코드보다 오래 붙든 건 두 가지였습니다. 하나는 **"글은 LLM, 구조와 근거는 서버"라는 역할 분담을 어디서도 흐리지 않는 것**. 다른 하나는 **템플릿 스키마 — 섹션마다 무엇을 물어야 하는가**를 설계하는 일이었습니다. 후자는 사실 20년 된 테크니컬 라이팅 원칙을 YAML로 옮겨 적는 작업에 가까웠습니다.

라이터의 무게중심은 초안을 쓰는 데서, 무엇이 참인지 판별하고 그 판별 기준을 도구로 굳혀두는 쪽으로 옮겨가고 있습니다. 이 MCP 서버는 그 이동을 제 손으로 코드에 새겨본 결과물입니다. 빈 화면 앞의 막막함은 사라지지 않지만, 이제 그 막막함을 잘 설계된 질문 몇 개로 바꿔둘 수는 있게 됐습니다.

로드맵은 아직 남았습니다 — Fumadocs MDX 스캐폴딩, Pandoc 한글화 레이어, HWPX 서식 채우기, 목차 대비 섹션 검토. 하지만 Phase 1(인터뷰 → 판정 → 질문 달린 목차)만으로도 매일 쓰고 있습니다. 도구는 완성돼서 쓰는 게 아니라, 쓰면서 완성되는 것 같습니다.
