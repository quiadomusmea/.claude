---
name: verifying-code-functionality
description: Use when asked to verify, test, or check if code works correctly. Use when completing a feature and needing to confirm it meets requirements. Use when code needs functional validation beyond just reading it.
---

# Verifying Code Functionality

## Overview

코드 검증은 **읽기가 아니라 실행**이다. 코드를 읽고 "문제 없어 보인다"는 검증이 아니다. 모든 요구사항을 실제로 실행하고, 결과를 pass/fail로 보고한다.

## When to Use

- 사용자가 "잘 동작하는지 확인해줘", "테스트해봐", "검증해줘" 요청 시
- 기능 구현 완료 후 동작 확인 필요 시
- 기존 코드가 요구사항을 충족하는지 판단 필요 시
- NOT: 코드 리뷰 (스타일, 설계 피드백) → 그건 별도 리뷰 스킬

## Core Principle

> **코드를 읽는 것은 검증이 아니다. 실행해서 결과를 확인하는 것이 검증이다.**

```
읽기만 함 → "문제 없어 보입니다" (검증 아님)
실행 + 결과 확인 → "요구사항 4개 중 4개 통과" (검증)
```

## 검증 플로우

```dot
digraph verification_flow {
    rankdir=TB;
    node [shape=box];

    read [label="1. 코드 읽기\n+ 요구사항 파악"];
    map [label="2. 요구사항 매핑\n각 요구사항 → 테스트 케이스"];
    decide [label="3. 검증 방법 결정" shape=diamond];
    run_direct [label="직접 실행\n(CLI, 스크립트, API 호출)"];
    unit_test [label="단위 테스트 작성\n→ writing-python-unit-tests 스킬"];
    both [label="둘 다\n(직접 실행 + 단위 테스트)"];
    edge [label="4. 엣지 케이스 테스트"];
    report [label="5. 결과 보고\n(pass/fail 테이블)"];

    read -> map -> decide;
    decide -> run_direct [label="간단한 코드\nCLI/GUI"];
    decide -> unit_test [label="복잡한 로직\n순수 함수"];
    decide -> both [label="복잡 + 외부 I/O"];
    run_direct -> edge;
    unit_test -> edge;
    both -> edge;
    edge -> report;
}
```

## Step 1: 코드 읽기 + 요구사항 파악

코드를 읽되, **검증할 항목을 추출하는 것이 목적**이다.

```
읽을 때 체크:
- 함수/메서드의 입출력 시그니처
- 외부 의존성 (DB, API, 파일시스템)
- 에러 처리 경로
- 설정/환경 변수
```

**복잡한 코드 (3+ 파일, 모듈 간 의존성):** code-explorer 에이전트에 위임하여 코드 구조를 먼저 파악.

```
Task(subagent_type="feature-dev:code-explorer",
     prompt="이 코드의 실행 흐름과 모듈 간 의존성을 분석해줘: [파일 목록]")
```

## Step 2: 요구사항 매핑 (필수)

**모든 요구사항을 테이블로 명시하고, 각각에 대해 테스트 케이스를 작성한다.**

```markdown
| # | 요구사항 | 테스트 방법 | 입력 | 기대 결과 | 상태 |
|---|---------|-----------|------|----------|------|
| 1 | URL에서 제목 추출 | 직접 실행 | example.com | "Example Domain" | - |
| 2 | --output CSV 저장 | 직접 실행 + 파일 확인 | -o test.csv | CSV 파일 생성 | - |
| 3 | --limit 링크 제한 | 직접 실행 | --limit 3 | 3개 이하 링크 | - |
| 4 | 잘못된 URL 에러 처리 | 직접 실행 | "not-a-url" | 에러 메시지, 크래시 없음 | - |
```

> 이 테이블을 먼저 만들지 않고 테스트를 시작하지 않는다.

## Step 3: 검증 방법 결정

| 코드 유형 | 검증 방법 | 이유 |
|-----------|----------|------|
| CLI 도구 | `uv run` / `python` 직접 실행 | 실제 사용 환경과 동일 |
| 순수 함수 (계산, 변환) | 단위 테스트 | 다양한 입력을 빠르게 검증 |
| GUI 앱 | 직접 실행 + 스크린샷/스냅샷 | 시각적 확인 필요 |
| API/서버 | 요청 보내서 응답 확인 | 실제 통신 검증 |
| 외부 의존성 있는 로직 | mock + 단위 테스트 | 격리 테스트 |

**단위 테스트가 필요한 경우:** `writing-python-unit-tests` 스킬을 활용한다.

## Step 4: 실행 및 검증

### 직접 실행 검증

```bash
# 요구사항 1: 기본 동작
uv run python main.py https://example.com

# 요구사항 2: CSV 출력
uv run python main.py https://example.com -o test.csv
# → 파일 생성 확인 + 내용 확인 (Read tool)

# 요구사항 3: 옵션 동작
uv run python main.py https://example.com --limit 3

# 요구사항 4: 에러 처리
uv run python main.py "not-a-url"
# → 크래시 없이 에러 메시지 출력되는지 확인
```

**각 실행 결과를 요구사항 테이블의 "상태" 열에 기록한다.**

### 단위 테스트 검증

```python
# 순수 함수는 직접 호출하여 검증
def test_calculate_total():
    result = calculate_total([{"price": 10, "quantity": 2}])
    assert result["subtotal"] == 20
    assert result["tax"] == 2.00
    assert result["shipping"] == 5.99  # under $50
```

## Step 5: 엣지 케이스 체크리스트

**항상 확인할 엣지 케이스:**

| 카테고리 | 테스트 케이스 |
|---------|-------------|
| 빈 입력 | 빈 리스트, 빈 문자열, None |
| 경계값 | 최소값, 최대값, 경계값 ±1 |
| 특수 문자 | 유니코드, 이모지, 줄바꿈 |
| 대용량 | 매우 긴 입력, 많은 항목 |
| 네트워크 | 타임아웃, 잘못된 URL, 연결 거부 |
| 파일 | 존재하지 않는 경로, 권한 없음, 디스크 풀 |

**전부 할 필요 없다.** 해당 코드에 관련 있는 것만 선택.

## Step 6: 결과 보고 (필수 형식)

```markdown
## 검증 결과

| # | 요구사항 | 상태 | 비고 |
|---|---------|------|------|
| 1 | URL에서 제목 추출 | PASS | "Example Domain" 정상 추출 |
| 2 | --output CSV 저장 | PASS | 헤더 + 데이터 정상 |
| 3 | --limit 링크 제한 | PASS | 3개 제한 동작 |
| 4 | 잘못된 URL 에러 처리 | FAIL | ConnectionError 크래시 |

**결과: 4개 중 3개 통과 (75%)**

### FAIL 항목 상세
- #4: `requests.exceptions.ConnectionError` 미처리.
  `except requests.RequestException` 추가 필요.
```

> **"문제 없어 보입니다"는 결과 보고가 아니다.** 반드시 pass/fail 테이블을 제시한다.

## 복잡한 검증: 서브에이전트 활용

| 상황 | 활용 방법 |
|------|----------|
| 3+ 파일 코드 구조 파악 | `feature-dev:code-explorer` 에이전트 |
| 다중 독립 테스트 | `Bash` 병렬 실행 |
| 단위 테스트 작성 | `writing-python-unit-tests` 스킬 |
| 보안/품질 검토 | `feature-dev:code-reviewer` 에이전트 |

## Red Flags - 검증이 아닌 것

- "코드를 읽어봤는데 문제 없어 보입니다" → 실행하지 않았음
- "이 함수는 올바르게 구현되어 있습니다" → 실행 결과 없음
- 요구사항 테이블 없이 "전반적으로 잘 동작합니다" → 무엇을 확인했는지 불분명
- "테스트 코드를 작성했습니다" (실행 안 함) → 작성 ≠ 검증

**이 중 하나라도 해당하면, 멈추고 실제로 실행한다.**

## Common Mistakes

| 실수 | 올바른 방법 |
|------|------------|
| 코드만 읽고 "잘 동작한다" 결론 | 반드시 실행하고 결과 확인 |
| 요구사항 일부만 테스트 | 요구사항 테이블로 전수 매핑 |
| happy path만 테스트 | 엣지 케이스 체크리스트 적용 |
| 결과를 서술형으로 보고 | pass/fail 테이블 필수 |
| 단위 테스트만 작성하고 실행 안 함 | 작성 + 실행 + 결과 확인 |
| 복잡한 코드를 혼자 분석 | 서브에이전트 활용 |
