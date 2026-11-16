# 전력 사용량 예측 모니터링 대시보드

전국 17개 시도의 **다음 달 전력 사용량 예측치를 지역끼리 비교해 보는** 대시보드입니다.
"다음 달 어느 지역을 더 유심히 봐야 하는가" 를 한 화면에서 판단하는 것이 목표였습니다.

저는 그중 **상세 데이터 페이지의 좌측 — 지역별 상세 표, 지역 선택, CSV, 백엔드 API** 를 맡았습니다.

![상세 데이터 페이지](docs/detail-page.png)

*왼쪽이 제가 맡은 영역입니다. 서울(A)·울산(B)을 고른 상태 — 표의 하이라이트, 상단 칩, 오른쪽 차트가 같은 선택을 따라갑니다.*

> 4인 팀 프로젝트 **HelloWorld** (2026.09.18 ~ 10.16 · **10.16 발표**) · 그린AI캠퍼스 평생교육원
> 상세 페이지 담당은 **09.30 ~ 10.12** 이고, 이후는 발표 준비입니다.
> 이 저장소는 팀 저장소 [`ung-Jin/Predict`](https://github.com/ung-Jin/Predict) 의 fork 입니다.

---

## 담당 범위

**상세 페이지의 좌측 절반과 그 백엔드를 통째로 맡았습니다.**

📁 **[`src/main/java/com/green/Predict/detail`](https://github.com/rlawns2848-hue/Predict/tree/dev/src/main/java/com/green/Predict/detail)**

| 파일 | 하는 일 |
|---|---|
| `controller/DetailController.java` | `/detail` 페이지 렌더링 (데이터는 안 싣는다) |
| `controller/DetailApiController.java` | `/detail-api/table` — 표 17행을 JSON 으로 |
| `service/DetailService.java` | 표 조회 |
| `mapper/DetailMapper.java` | DB 접근 인터페이스 |
| `dto/RegionDetailRow.java` | 표 한 행 (식별 3 + 예측 2 + 정확도 3) |
| `resources/mapper/DetailMapper.xml` | `region` · `forecast` · `backtest` 세 테이블을 한 쿼리로 |
| `templates/detail.html` | 페이지 틀 (좌 표 + 우 차트 배치) |
| `templates/fragments/detail_filter.html` | 상단 필터 바 + 지역 선택 모달 |
| `templates/fragments/detail_table.html` | 좌측 표의 머리글 |
| `static/js/detail.js` | **649줄** — 표 그리기 · 모달 · 선택 상태 · CSV |
| `static/css/detail.css` | 표 · 칩 · 모달 · 좌우 레이아웃 |

🔗 **[팀 저장소에서 제 커밋 보기](https://github.com/ung-Jin/Predict/commits/dev?author=rlawns2848-hue)**

**손대지 않은 파일** — `fragments/detail_charts.html`, `js/detail-charts.js`, `common.css`, 공통 사이드바·헤더. 팀원 소유입니다.

---

## 주요 기능

### 1. 지역별 상세 표

17개 시도를 **예측량이 큰 순으로** 한 줄씩 보여줍니다.

```
시도 │ 8월 예측(GWh) │ 전년 대비 증감률 │ MAPE │ RMSE │ MAE
```

정렬은 화면이 아니라 **SQL 에서 `ORDER BY predictedGwh DESC`** 로 끝냅니다. 받아온 순서가 곧 표시 순서라 JS 가 다시 정렬할 일이 없습니다.

숫자만 있으면 크기 비교가 잘 안 돼서, 예측량과 MAPE 두 열에는 **그 열의 최댓값 대비 비율만큼 길이가 늘어나는 막대**를 숫자 옆에 붙였습니다.

```javascript
// 최댓값을 먼저 구해 두고, 각 행의 길이를 비율(%)로 환산
let widthPct = (fc / maxFc) * 100;
if (widthPct > 100) widthPct = 100;
```

셀 종류마다 함수를 하나씩 뒀습니다 (`makeSidoCell` · `makeForecastCell` · `makeYoyCell` · `makeMapeCell`). 한 함수가 길어지는 것보다, **고칠 셀의 함수만 찾아가면 되는 쪽**이 나중에 읽기 쉬웠습니다.

### 2. 지역 선택 — A·B 두 자리

비교는 **최대 두 개**까지입니다. 표의 행을 눌러도, 모달에서 골라도 같은 자리에 들어갑니다.

세 번째를 고르면 **먼저 고른 쪽이 밀려납니다.**

```
A 자리 비었으면        → A 로
A 만 차 있으면         → B 로
둘 다 차 있으면        → 기존 A 를 버리고 · 기존 B 가 A 로 · 새 지역이 B 로
이미 선택된 걸 또 누르면 → 해제
```

"가장 최근에 고른 둘" 이 항상 남게 되는 규칙입니다. 선택을 지우라고 따로 요구하지 않아도 계속 비교할 수 있습니다.

모달에는 가나다 정렬, 시도 검색, 선택 해제, `Esc` 닫기를 넣었습니다.

### 3. 선택 상태를 다른 영역과 나눠 쓰기

같은 페이지 오른쪽의 차트 3종은 **팀원이 만든 다른 파일**입니다. 제 쪽에서 지역을 바꾸면 차트도 따라 바뀌어야 하는데, 서로의 함수를 직접 부르면 **한쪽을 고칠 때마다 다른 쪽이 깨집니다.**

그래서 저장소와 신호만 공유했습니다.

```javascript
function setSelection(nextA, nextB) {
  // (1) 팀이 약속한 키로 저장 — 지도·대시보드도 같은 키를 읽는다
  if (nextA) sessionStorage.setItem(DETAIL_KEY_A, nextA);
  else       sessionStorage.removeItem(DETAIL_KEY_A);
  // ...

  // (2) "바뀌었다" 고 알리기만 한다. 누가 듣는지는 모른다
  document.dispatchEvent(new CustomEvent('region:selected',
    { detail: { a: nextA, b: nextB } }));
}
```

**선택을 바꾸는 모든 경로가 이 함수 하나를 지나갑니다.** 저장과 알림이 한 자리에 있어서, "저장은 됐는데 화면이 안 바뀌는" 경우가 생기지 않습니다.

> `setItem(key, null)` 을 하면 `null` 이 아니라 **`"null"` 이라는 문자열이 저장됩니다.** 해제할 때 `removeItem` 을 쓰는 이유입니다.

### 4. 상세 API — 세 테이블을 한 번에

표 한 줄에 들어가는 값이 **서로 다른 세 테이블**에 흩어져 있었습니다.

```
region     시도 이름 (풀네임 · 짧은 이름)
forecast   예측 전력량 · 전년 동월 값
backtest   검증 기간의 예측값과 실제값 (여러 달치)
```

쿼리를 세 번 날려 자바에서 합칠 수도 있었지만, **한 번에 받아오는 쪽**을 골랐습니다. 화면이 표 하나를 그리려고 서버를 세 번 부를 이유가 없었습니다.

까다로웠던 건 두 가지입니다.

**① "8월" 을 어떻게 고르나** — `WHERE month = 8` 로 박으면 다음 달 예측이 들어왔을 때 쿼리를 고쳐야 합니다. **예측 테이블에서 가장 이른 달**을 서브쿼리로 찾아 거기에 맞췄습니다.

```sql
JOIN (SELECT year AS y, month AS m
      FROM forecast
      ORDER BY year, month
      LIMIT 1) first
    ON fc.year = first.y AND fc.month = first.m
```

**② 정확도 3종은 여러 행을 하나로 줄여야 한다** — `backtest` 는 지역마다 검증 기간만큼 행이 있습니다. MAPE·RMSE·MAE 를 자바에서 계산하지 않고 **SQL 집계로 지역당 한 줄까지 줄여서** 가져옵니다.

```sql
ROUND(AVG(ABS(bt.predicted_kwh - bt.actual_kwh) / bt.actual_kwh) * 100, 2)  AS mape,
ROUND(SQRT(AVG(POW(bt.predicted_kwh - bt.actual_kwh, 2))) / 1000000, 1)     AS rmseGwh,
```

이 집계는 `LEFT JOIN` 으로 붙였습니다. **검증 데이터가 없는 지역이라도 표에서 사라지면 안 되기** 때문입니다. 정확도 칸만 `-` 로 비고 나머지 값은 그대로 보입니다.

단위 환산(`kWh → GWh`)도 SQL 에서 끝냅니다. 화면이 GWh 로 보여 주므로 **표시 단위로 맞춰서 내보내는 쪽**이 중간에 실수할 여지가 적었습니다.

### 5. CSV 내보내기 — 서버 없이

표에 이미 있는 데이터를 다시 받으려고 엔드포인트를 만들 필요는 없다고 봤습니다. **브라우저 안에서 문자열을 조립해 파일로 만듭니다.**

```javascript
const blob = new Blob([BOM + csvText], { type: 'text/csv;charset=utf-8;' });
const url  = URL.createObjectURL(blob);     // 메모리 안의 임시 주소

const link = document.createElement('a');   // 보이지 않는 링크를 만들어
link.href = url;
link.download = 'detail_regions.csv';
document.body.appendChild(link);
link.click();                                // 자동으로 누른다

document.body.removeChild(link);
URL.revokeObjectURL(url);                    // 임시 주소 해제
```

맨 앞에 붙는 것은 **BOM** 입니다. 이게 없으면 엑셀이 UTF-8 인 줄 모르고 한글을 깨뜨립니다. 처음엔 글자가 다 깨져 나와서, **인코딩이 아니라 "엑셀에게 인코딩을 알려 주는 표시" 가 빠진 것**이라는 걸 한참 뒤에 알았습니다.

---

## 설계에서 신경 쓴 것

- **서버는 틀만, 본문은 JS 가.** 표의 머리글은 Thymeleaf 가 서버에서 넣고, 17행은 `/detail-api/table` 응답으로 그립니다. 선택이 바뀌어도 **서버를 다시 부르지 않고** 받아 둔 배열(`detailTableRows`)로 하이라이트만 다시 그립니다.
- **상태를 한 함수에 모았습니다.** 고를 때·지울 때·바꿀 때가 전부 `setSelection()` 을 지납니다.
- **이름 앞에 `detail` 을 붙였습니다.** 한 페이지에 팀원 4명의 스크립트가 같이 로드되기 때문에, 전역 이름이 겹치면 조용히 덮어씁니다.
- **`fetch` 대신 `axios`.** 팀의 다른 영역이 이미 쓰고 있었고, 200 이 아니면 알아서 던져 줘서 `catch` 하나로 처리됩니다.

---

## 트러블슈팅 — 서버는 200 OK 인데 표가 비어 있었다

표가 안 보여서 먼저 네트워크 탭을 봤습니다. `/detail-api/table` 은 **17행을 200 으로 정상 응답**하고 있었습니다. DOM 에도 `<tr>` 17개가 멀쩡히 들어 있었습니다. 그런데 화면에는 아무것도 없었습니다.

데이터가 사라진 게 아니라 **놓일 자리가 사라져** 있었습니다.

```
.detail-table (좌)    flex-basis: 0      ← 내용과 무관하게 0 에서 시작
.detail-charts (우)   flex-basis: auto   ← Chart.js 캔버스가 내용만큼 다 가져감
```

`flex` 는 **콘텐츠 크기를 보고** 공간을 나눕니다. 우측 캔버스가 자기 폭을 주장하는 동안 좌측은 0 에서 늘어나지 못하고 찌그러졌습니다.

비율로만 나누는 Grid 로 바꿔서 해결했습니다.

```css
.detail-body {
  display: grid;
  grid-template-columns: 1.35fr 1fr;   /* 내용이 뭐든 비율은 지켜진다 */
  gap: 16px;
}
```

> **"응답이 정상이면 화면도 정상" 이 아니었습니다.** 데이터 흐름에만 문제가 있을 거라고 보고 네트워크와 바인딩만 들여다봤는데, 정작 원인은 CSS 에 있었습니다. 이제는 "오는데 안 보인다" 싶으면 **그 요소의 실제 계산된 폭부터** 봅니다.

### 덧 — 담당이 바뀌었을 때 무엇을 지울지 (10/7)

우측 차트 3종 담당이 팀원에게 넘어갔습니다. 제가 짜 둔 차트 코드를 어디까지 지울지 정해야 했습니다.

**소유 파일은 그대로 두고, 제가 쓴 차트 전용 코드만 지웠습니다.** `detail_charts.html` 과 Chart.js 로드는 팀원이 이어서 쓸 자리라 건드리지 않고, `/charts` 엔드포인트·차트 전용 DTO·쿼리·호출부만 걷어냈습니다.

지금 코드의 주석에 그 경계가 남아 있습니다.

```java
/**
 * [4번 소유] 상세 데이터 페이지 좌측 표가 axios로 호출하는 JSON API.
 *
 * 우측 차트 3종은 1번 팀원이 담당하므로 이 컨트롤러에는 관련 엔드포인트가 없다.
 */
```

---

## 작업 기록

작업일지를 Notion 에 남겼습니다.

| 날짜 | 내용 |
|---|---|
| 09/21 | [전력 데이터 수집과 병합 전 고민](https://scratched-ceramic-15c.notion.site/9-21-3e490bcc7b07813f8ac4d14a34b1269d) |
| 09/22 | [지도학습 방향 확정과 상관관계 분석의 벽](https://scratched-ceramic-15c.notion.site/9-22-3e490bcc7b078165bdc7c22960ddce34) |
| 09/23 | [결측치 처리부터 4개 모델 비교까지](https://scratched-ceramic-15c.notion.site/9-23-4-3e890bcc7b07812984c9ce13bdb110ff) |
| 09/30 | [엑셀 데이터 DB 적재와 MyBatis 복합키 매핑](https://scratched-ceramic-15c.notion.site/9-30-DB-MyBatis-3f290bcc7b07819ea717d9c291d7c2df) |
| **10/06** | **[상세 데이터 페이지 처음부터 끝까지](https://scratched-ceramic-15c.notion.site/10-6-3f290bcc7b0781b29fcccea2057e615a)** |
| 10/07 | [역할 분담 변경과 가독성 중심 리팩토링](https://scratched-ceramic-15c.notion.site/10-7-3f290bcc7b078187a84bd2ca7f03a4a8) |
| **10/12** | **[상세 페이지를 마무리하며 — 남긴 것과 남은 것](https://scratched-ceramic-15c.notion.site/10-12-41a90bcc7b07811d9bdbe0b7350f647d)** |

> 앞 세 편(9/21~9/23)은 **데이터와 모델** 이야기이고, 9/30 부터가 **웹 담당** 이야기입니다.
> 팀이 예측 방식을 정하는 동안 같이 데이터를 들여다본 기록이라 함께 남겨 뒀습니다.

---

## 알려진 제약 · 보완할 점

**고치지 못한 채로 남아 있는 것들입니다.**

- **CSV 에 이스케이프 처리가 없습니다.** `row.join(',')` 으로 그냥 잇기 때문에, 값 안에 콤마나 따옴표가 들어가면 열이 밀립니다. 지금 데이터는 시도명과 숫자뿐이라 드러나지 않을 뿐, 막아 둔 것은 아닙니다
- **CSV 는 항상 17행 전체입니다.** A·B 를 골라 둬도 선택과 무관하게 전체가 나갑니다
- **반응형이 없습니다.** `1.35fr : 1fr` 고정이라 좁은 화면에서는 두 영역이 같이 좁아집니다
- **표 정렬이 예측량 내림차순으로 고정입니다.** 머리글을 눌러 MAPE 순으로 보는 것이 안 됩니다
- **상단 "기간" 은 표시용입니다.** `2022.01 ~ 2026.07` 이 버튼처럼 보이지만 실제 피커가 없습니다
- **표 로드에 실패하면 콘솔에만 남습니다.** 화면에는 "데이터가 없습니다" 만 뜨고 다시 시도할 방법이 없습니다
- **행을 `innerHTML` 문자열로 조립합니다.** 값이 DB 에서 오는 시도명이라 지금은 문제가 없지만, 사용자 입력이 섞이는 자리였다면 그대로 두면 안 되는 방식입니다

> 첫 번째가 가장 신경 쓰입니다. **"지금 데이터로는 안 터진다" 와 "안 터지게 막았다" 는 다릅니다.**

---

## 기술 스택

| | |
|---|---|
| **Backend** | Java 17 · Spring Boot · MyBatis 4.0.1 · Thymeleaf (fragment 분할) · Lombok · Gradle |
| **Database** | MariaDB |
| **Frontend** | Vanilla JavaScript (ES6+) · axios 1.7.7 · HTML5 / CSS3 (Grid · Flexbox · 커스텀 프로퍼티) |
| **브라우저 API** | sessionStorage · CustomEvent · Blob / URL.createObjectURL |
| **협업** | Git / GitHub (feature → PR → dev → main) · Notion |

팀 공통으로 Chart.js 4.4.1 · amCharts 5 를 썼습니다.

**데이터 정리와 모델 학습은 팀이 함께 했습니다.** Python (pandas · scikit-learn · XGBoost · LightGBM) 으로 세 모델을 비교해 Random Forest 를 골랐고(MAPE 4.11%), 화면은 모델을 돌리지 않고 **미리 계산해 DB 에 적재한 값을 읽습니다.** 제가 만든 표의 MAPE·RMSE·MAE 열이 이 검증 결과(`backtest`)입니다.

---

## 실행 방법

### 1. 접속 정보

`src/main/resources/application.yaml` 의 `spring.datasource` 를 본인 환경에 맞게 바꿔 주세요.

### 2. 실행

```bash
./gradlew bootRun
```

`http://localhost:8080` · 상세 데이터 페이지는 `/detail` 입니다.

> DB 스키마(`region` · `forecast` · `backtest`)와 적재 데이터는 저장소에 들어 있지 않습니다.

---

## 팀 구성

**화면 영역으로 나눴습니다.**

| 담당 | 범위 |
|---|---|
| 이웅진 (팀장) | **공통 기반** — 사이트 뼈대와 DB · Git 관리 · 레이더와 상세 차트 |
| 박성인 | **지도와 지역 비교** — 지도와 지역 선택 · 선택 표시 · 도넛과 비교 기능 |
| 최준영 | **요약과 예측** — 요약 카드 · 3개월 예측 · 월별 추이와 순위 |
| **김성준** | **상세 데이터** — 지역별 상세 표 · 지역 선택 창 · 상세 API와 CSV |

화면 기능은 넷으로 나눴지만, **데이터 정리와 모델 학습은 팀 공동 작업**이었습니다. 9/21~9/23 작업일지가 그 기간의 기록입니다.

상세 페이지는 **좌우를 둘이 나눠** 맡았습니다. 같은 화면을 두 사람이 건드리게 되므로, 파일 소유를 먼저 정하고 선택 상태만 `sessionStorage` 로 주고받는 방식으로 경계를 뒀습니다.
