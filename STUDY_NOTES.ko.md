# translate-book 학습 노트 (한국어)

원본: https://github.com/deusyu/translate-book (MIT). 이 노트는 소스를 읽고 정리한 학습 기록이며 원저작물이 아닙니다.

## 한 줄 요약
PDF/DOCX/EPUB 책을 Markdown 청크(약 6,000자)로 쪼개고, 청크마다 독립 서브에이전트가 병렬 번역한 뒤 검증·병합하여 HTML/DOCX/EPUB/PDF로 만드는 에이전트 스킬.

## 파이프라인
1. `convert.py`: Calibre(HTMLZ) → Markdown → 청크 분할. `manifest.json`(청크별 SHA-256)과 `source_fingerprint.json`(원본 정체성) 기록. `.md` 입력은 Calibre와 pypandoc을 건너뜀.
2. `glossary.py`: 5개 청크를 샘플링해 `glossary.json`(v2) 작성. 청크별로 (해당 청크에 등장하는 용어 ∪ 책 전체 빈도 상위 N개, 최대 50개)를 3열 표로 프롬프트에 주입.
3. `run_state.py plan`: 번역할 청크 / 기록만 할 청크 / 변경 없음으로 분류. 용어집 수정 시 영향받는 청크만 재번역.
4. 병렬 번역: 청크 1개 = 서브에이전트 1개, 기본 8개씩 배치. 앞뒤 청크 발췌 300자를 읽기 전용으로 제공(`chunk_context.py`).
5. `merge_meta.py`: 서브에이전트의 `*.meta.json` 관찰을 배치마다 용어집에 병합. `prepare-merge` → 메인 에이전트 판단 → `apply-merge`(전부 성공해야 반영되는 트랜잭션).
6. `merge_and_build.py`: 검증(1:1 대응, 해시, 빈 출력, 이미지 참조 보존) → 병합 → pandoc HTML → 목차 삽입 → Calibre로 DOCX/EPUB/PDF.

## 설계 원칙
- 스크립트는 기록·검증(결정적), LLM은 의미 판단.
- 공유 상태는 메인 에이전트만 쓴다(락 불필요). 서브에이전트는 청크별 meta 파일만 쓴다.
- 병합은 보수적: 새 개체엔 근거 필요, 성별은 `unknown`에서 시작, 충돌 시 조용히 덮어쓰지 않음.
- 상태 파일 분리: `glossary.json`(정본) / `*.meta.json`(관찰) / `run_state.json`(오케스트레이션).

## 핵심 구현 메모
- 청크 분할: 구조 블록(코드·수식·제목·인용·표·목록·이미지·문단)을 쪼개지 않고 합침. 제목 경계 우선. 12,000자 초과 블록만 강제 분할(코드는 펜스 재부착).
- 페이지 번호 제거: 단독 숫자 줄에서 LNDS(최장 비감소 부분수열)로 단조 수열을 찾아 제거. 연도·장 번호는 보존.
- 용어 매칭: ASCII는 단어 경계 정규식, CJK는 부분 문자열, 한 글자 CJK는 제외. 같은 표면형을 두 용어가 공유할 수 없음.
- 신뢰도: 증거 청크 1개 low, 2개 medium, 3개 이상 high(내려가지 않음). 증거는 최대 5개(FIFO).
- 이미지 검증: 원문과 출력의 `![](..)`·`<img src>` 개수 비교, alt 속성 안 따옴표로 태그가 깨졌는지 검사.
- HTML 변환 폴백: pandoc → python-markdown → 정규식.

## 환경 요구사항
- Calibre(`ebook-convert`), Pandoc, Python 3.12+, `pypandoc`(입력 변환 시 필수), `beautifulsoup4`, `markdown`.
- 단위 테스트는 표준 라이브러리만으로 실행됨: `python3 -m unittest discover -s tests -p 'test_*.py'` (245개 통과 확인).

## 한국어 번역 시 참고
- `ko` 출력 설정 있음(폰트 Nanum Myeongjo/Batang, 목차 라벨 "목차").
- 번역 프롬프트 본문은 중국어. 한국어 번역 품질이 아쉬우면 SKILL.md의 지시문을 한국어로 바꾸는 것을 검토.
- 신학·고전 번역은 `glossary.json`을 번역 전에 직접 다듬는 것이 품질의 핵심.
