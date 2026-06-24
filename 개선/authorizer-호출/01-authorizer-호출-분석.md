# Authorizer 호출 과다 분석

[2026.06.12]



## 0. 요약

- `jariitnayo-authorizer`가 전체 Lambda 중 **호출량 1위** (일평균 446회, 피크 676회). 2위 `get-points-balance`(215회)의 2배 이상.
- 원인은 구조적. 보호 라우트(12개) 모든 요청이 타겟 Lambda 도달 전 authorizer를 거치고, 홈 화면 3종(`points/balance`, `level/me`, `seats/me`) 버스트가 보호 트래픽의 ~95%.
- API Gateway Authorizer **결과 캐싱** 적용. 같은 토큰의 단기 재호출을 캐시 히트로 병합하여 호출 자체를 줄임.



## 1. 문제 상황

CloudWatch 주간 호출 추이에서 `jariitnayo-authorizer`가 호출량 1위로 확인.

**최근 7일 일별 Invocations** (Sum/1d, KST):

| 날짜 | 06-04 | 06-05 | 06-06 | 06-07 | 06-08 | 06-09 | 06-10 | 06-11 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 호출 | 669 | 283 | 223 | 582 | 676 | 327 | 228 | 584 |

→ 합계 3,572 / **일평균 446회**. 2위 `get-points-balance`(일평균 215회)의 2배 이상.



## 2. 원인 분석

### 구조적 원인

보호 라우트(12개)로 들어오는 모든 요청이 타겟 Lambda 도달 전 authorizer Lambda를 통과:

```
[보호 API 1회 호출]
API GW → authorizer Lambda invoke → JWT 검증 + Users get_item → Allow → 타겟 Lambda
```

즉 인증이 필요한 모든 요청이 **authorizer Lambda invoke 1회 + Users get_item 1회**를 항상 추가 부담. 호출량 자체가 보호 트래픽 수에 그대로 비례하는 구조.

### 앱 사용 패턴이 호출량 증폭

홈 화면 진입 한 번에 `points/balance` + `level/me` + `seats/me` 3개 보호 요청이 연달아 발사. 최근 7일 보호 라우트 호출량 상위 3개가 정확히 이 3개:

| routeKey | 건수 |
| --- | --: |
| GET /api/v1/points/balance | 1,161 |
| GET /api/v1/level/me | 1,018 |
| GET /api/v1/seats/me | 663 |

→ 합계 2,842건 = 보호 트래픽의 약 95%. **홈 진입 1회당 3건 버스트 패턴**. 같은 토큰으로 1초 안에 3건이 발사되는 형태라 캐싱의 적중 대상과 정확히 일치.



## 3. 해결

### 검토한 대안

| 방안 | 효과 | 기각 사유 | 결정 |
| --- | --- | --- | --- |
| Provisioned Concurrency | authorizer cold 제거 | 추가 비용 발생. 호출량 자체는 그대로 (진짜 문제 해결 안 됨) | 기각 |
| 인증 클라이언트 측 캐싱 | 호출 일부 감소 | 토큰 만료/취소 처리 복잡, 보안 정책 변경 시 일관성 깨짐 | 기각 |
| **API Gateway Authorizer 결과 캐싱** | **invoke 자체 병합으로 호출량 직접 감소** | - | **채택** |

### 채택

API Gateway에 판정 결과(통과/거절) 캐싱 설정. 

같은 토큰(Authorization 헤더 = 캐시 키)은 TTL 동안 직전 판정(Allow + userId 컨텍스트)을 재사용 
→ authorizer Lambda 호출과 Users 조회 건너뜀.

```hcl
identity_sources                 = ["$request.header.Authorization"]
authorizer_result_ttl_in_seconds = 60   # 카나리. 검증 후 300으로 상향
```

**추가 변경**: `authorizer/handler.py`의 Allow 정책 Resource를 라우트별 ARN에서 스테이지 wildcard(`.../$default/*`)로 변경. 캐싱 시 한 토큰의 판정이 모든 라우트에서 재사용되기 위한 필수 조건.



### 추가 효과

캐싱은 호출량 감소가 목적이지만, 다음 효과도 따라옴:

- **응답시간 단축**: 기존엔 보호 요청 1회 = authorizer Lambda 시간 + 타겟 Lambda 시간이 합산됐는데, 캐시 히트 시 authorizer 부분이 통째로 빠져 타겟 Lambda 시간만 남음.
- **cold 스파이크 제거**: authorizer가 cold일 때 응답에 가산되던 ~950ms(Init 649ms + 실행 301ms)도 캐시 히트 시 사라짐. "오랜만에 앱 열었을 때 첫 화면이 이상하게 느린" 케이스 해소.
- **Users 테이블 부하 감소**: 캐시 히트 1건마다 Users get_item 1회가 줄어듦. 캐시 미스 시에만 조회 발생.

→ 실측 결과: [`02-개선결과.md`](./02-개선결과.md)
