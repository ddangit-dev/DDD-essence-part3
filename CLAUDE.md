# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 저장소 개요

"도메인 주도 설계의 사실과 오해" 파트 3 — *더 심층적인 통찰력을 향한 리팩터링* 학습용 코드 모음.
각 모듈은 동일한 도메인을 점진적으로 리팩터링한 **스냅샷**이며, 모듈 사이를 비교하면서 변화의 의도를 추적하는 것이 핵심이다.

- **Java 17**, **Maven 멀티 모듈** (`pom.xml` packaging=`pom`)
- 의존성: Lombok 1.18.38, JUnit **4** (`org.junit.Test`), AssertJ 3.27.3
- 루트 패키지는 모두 `org.eternity` 하위
- 빌드 산출물(`target/`)과 IntelliJ 모듈 파일(`.idea/modules.xml` 등)은 `.gitignore` 처리

## 자주 쓰는 명령

```bash
# 전체 빌드(테스트 포함)
mvn verify

# 컴파일만
mvn compile

# 전체 테스트
mvn test

# 단일 모듈 테스트 (예: 01-syndicated-loan)
mvn -pl 01-syndicated-loan test

# 단일 테스트 클래스 실행
mvn -pl 03-breakthrough test -Dtest=FacilityTest

# 단일 테스트 메서드(한글 메서드명은 따옴표 권장)
mvn -pl 01-syndicated-loan test -Dtest='FacilityTest#회사별_퍼실러티_대출_금액'

# 특정 모듈 빌드 (의존성 모듈 포함하려면 -am)
mvn -pl 08-loan-refactoring -am package
```

## 모듈 구성과 학습 흐름

모듈은 두 개의 도메인 트랙으로 나뉘며, 번호 순서가 곧 리팩터링 단계다.

### 신디케이트 대출 트랙 (`org.eternity.loan`)
| 모듈 | 핵심 변화 |
|---|---|
| `01-syndicated-loan` | 출발점. `Facility`(애그리거트 루트) → `Loan` → `LoanInvestment(Investment)`. 대출 분배는 `Investment.percentage` 비율로 수행 |
| `02-refinement` | `LoanAdjustment extends LoanInvestment` 도입 — 조정된 대출에 *원래 정보*를 보존 |
| `03-breakthrough` | 돌파. `Investment`/`LoanInvestment`를 버리고 `Share`(VO)와 `Map<Company, Share>` 기반으로 재설계. `Loan.increase`가 `Map<Company, Share>`을 받음 |
| `08-loan-refactoring` | 위 결과를 다시 4단계로 정련 (아래) |

`08-loan-refactoring` 내부 단계 (소스 트리에서 `step01_*` ~ `step04_*` 패키지로 분리되어 같은 모듈에 공존):
- `step01_start_loan` — 03 결과를 출발점으로 재구성
- `step02_side_effect_free` — 부수효과 없는 함수로 `Loan` 변경 메서드 정리
- `step03_sharepie` — **`SharePie`** 1급 추상 도입 (`Map<Company, Share>` 캡슐화). `prorate`/`plus`/`minus`가 새 `SharePie`를 반환
- `step04_conceptual_contour` — 개념적 윤곽. `Facility`가 `SharePie`로 직접 대화하고, `Loan`은 `SharePie`를 합/차로 다룸. 상환(`calculateRepayments`/`applyRepayments`/`distributeChange`) 책임이 도메인 객체로 들어옴

### 페인트 트랙 (`org.eternity.paint`)
| 모듈 | 핵심 변화 |
|---|---|
| `04-paint` | 출발점. `Paint`가 `v/r/y/b` 필드와 절차적 `paint(Paint)` 메서드를 가짐 |
| `05-intention-revealing-interface` | 의도를 드러내는 인터페이스. `v→volume`, `paint→mixin` 등 이름만 바꿔도 의미가 살아남 |
| `06-side-effect-free-function` | `PigmentColor` 분리. `mixedWith`는 새 객체를 반환(부수효과 제거) |
| `07-assertion` | `Paint`를 **인터페이스**로 추상화하고 `StockPaint`/`MixedPaint`로 분리. `MixedPaint.getVolume`/`getColor`가 빈 컬렉션에 대해 단언(`IllegalStateException`)을 명시 |

## 아키텍처 핵심: 레이어 슈퍼타입

각 모듈의 `org.eternity.shared.domain` 패키지에는 **동일한 3개의 추상 클래스**가 사본으로 존재한다 (모듈은 독립적이라 공유 라이브러리가 아님 — 학습 단계별 스냅샷이기 때문).

- `DomainEntity<T, TID>` — `equals`/`hashCode`를 `getId()` 기반으로 강제. 추상 메서드는 `getId()` 하나
- `AggregateRoot<T, TID> extends DomainEntity<T, TID>` — 애그리거트 루트 표식 (현재는 추가 동작 없는 마커성 슈퍼타입)
- `ValueObject<T>` — 리플렉션으로 선언 필드 전체를 모아 `equals`/`hashCode` 구현 (`getEqualityFields()` 오버라이드 가능)

`Money`(`org.eternity.shared.monetary.Money`)는 `ValueObject<Money>`를 상속하지만 `compareTo` 기반의 자체 `equals`를 다시 정의해 `BigDecimal` scale 차이를 흡수한다. 따라서 같은 금액의 `won(100)`과 `won(100L)`은 동등.

## 작업 시 유의점

- **모듈 간 코드 공유 금지가 의도된 설계다.** 같은 이름의 클래스(`Loan`, `Facility`, `Paint`, `Money` 등)가 여러 모듈에 존재하지만 각자 독립적으로 진화한다. 이전 모듈을 수정해 다른 모듈을 "고치는" 식의 변경은 학습 의도를 깨뜨리므로 피한다.
- `08-loan-refactoring`은 한 모듈 안에서 `step01_*` ~ `step04_*` 패키지가 **병렬로 공존**한다. 한 step의 클래스가 다른 step을 참조하지 않도록 import 경로를 항상 확인.
- 테스트 메서드명은 **한글**(`회사별_퍼실러티_대출_금액`, `지분_더하기` 등). 새 테스트 추가 시 동일 컨벤션을 따른다.
- `Facility`의 정적 팩토리 메서드 이름은 `crate`(오타가 아니라 코드 그대로) — 기존 모듈의 명명을 함부로 바로잡지 말 것.
- JUnit **4**가 의도적 선택이다 (`@org.junit.Test`). JUnit 5(`Jupiter`)로 임의 마이그레이션하지 않는다.
