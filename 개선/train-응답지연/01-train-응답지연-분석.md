# 서울시 API 호출 람다 응답지연 분석

[2026.06.02 / 최종 수정 2026.06.07]

<br>

## 0. 요약

- `get_train`, `get_nearby_arrival` 등 서울시 API를 호출하는 람다가 cold start 시 평균 **2,443ms** 응답지연 
- 원인은 **`httpx` 클라이언트 초기화 오버헤드**
- httpx -> http.client 로 교체. 대상: `get_train`, `get_nearby_arrival`, `pass_seat`

<br>

## 1. 문제 상황

사용자로부터 열차 조회가 느리다는 보고 접수.
 `jariitnayo-get-train` Duration을 Average / Maximum / p99로 조회.

- Average: 정상 범위
- p99: 최대 3초까지 튀는 구간 확인 → 일부 호출이 극단적으로 느림

CloudWatch Logs Insights로 cold/warm 분리하여 분포 확인:

```
filter @type = "REPORT"
| parse @message "Duration: * ms" as duration
| parse @message "Init Duration: * ms" as initDuration
| fields ispresent(initDuration) as isColdStart
| stats avg(duration) as avgDuration, max(duration) as maxDuration,
        avg(initDuration) as avgInitDuration, count() as count
  by isColdStart
```

지난 3일 결과:

| 함수 | 상태 | 평균 Duration | 최대 Duration | 평균 Init | 건수 |
| --- | --- | --- | --- | --- | --- |
| `get_train` | Warm | 192ms | 3,000ms\* | - | 210 |
| `get_train` | Cold | 2,397ms | 3,000ms\* | 653ms | 95 |
| `get_nearby_arrival` | Warm | 183ms | 368ms | - | 178 |
| `get_nearby_arrival` | Cold | 2,399ms | 2,614ms | 649ms | 144 |

\* `get_train` maxDuration 3,000ms는 Lambda 기본 타임아웃(3s)으로 잘린 값.

**결과: cold start에서만 ~2,400ms 추가 지연 발생, warm 상태는 정상.**

<br>

## 2. 원인 분석

<br>

### 초기 가설 - TCP 소켓 재초기화

Cold start 시 Python 람다의 네트워크 소켓이 새로 초기화되면서, 첫 번째 외부 HTTP 연결(서울시 API)이 TCP 핸드셰이크부터 시작되어 응답 지연이 발생하는 것으로 추정.

<br>

### 가설 검증 도구 선택

HTTP 요청의 어느 구간(DNS / TCP / TLS / TTFB / transfer)에서 시간이 소요되는지 알아야 검증 가능.

| 측정 방법 | 평가 | 결정 |
| --- | --- | --- |
| `httpx` | `response.elapsed`로 전체 왕복만 알 수 있음. 구간 분리 불가 | 부적합 |
| `pycurl` | 구간별 타이밍 직접 제공. 가장 정밀 | PyPI에 manylinux 휠 없어 Lambda 배포 불가 |
| `http.client` (표준 라이브러리) | `socket` + `http.client`에 `time.perf_counter()`로 구간별 직접 측정 | **채택** |

테스트 람다 제작, `http.client`로 구간별 측정.

<br>

### 측정 결과 - 가설 전환

테스트 람다(http.client)와 운영 중인 `get_nearby_arrival`(httpx)을 동시간대 같은 역으로 호출하여 비교.

| 측정 회차 | network_probe (http.client) | get_nearby_arrival (httpx) |
| --- | --- | --- |
| 1회 Cold HTTP 구간 | 156ms | 2,449ms |
| 2회 Cold HTTP 구간 | 54ms | 2,448ms |
| Init Duration | ~630ms | ~670ms |

Init Duration은 거의 동일한데 **HTTP 구간만 매번 ~2,400ms 차이**. 두 함수의 차이는 HTTP 라이브러리뿐 
→ **가설 전환: TCP 소켓 재초기화가 아니라 `httpx` 패키지 자체의 초기화 오버헤드가 원인.**

<br>

### httpx vs http.client 차이

`httpx.get(url)`은 호출할 때마다 내부적으로 HTTP 클라이언트 인스턴스, Connection Pool, Transport 레이어, 타임아웃/헤더/리다이렉트 기본값을 **매번 새로 생성**한다. Warm 상태에서는 라이브러리가 메모리에 있어 이 비용이 작지만, cold start에서는 Python 프로세스가 새로 뜨면서 httpx 임포트 + 클라이언트 초기화가 합쳐져 ~2,400ms가 발생.

`http.client`는 Python 표준 라이브러리로 임포트 비용이 거의 없고, 클라이언트 객체 없이 소켓을 직접 열어 요청하는 구조라 초기화 비용이 거의 없음.

<br>

## 3. 테스트

<br>

### 최종 검증 - get_train 로직 + http.client

테스트 람다에 `get_train`과 동일한 비즈니스 로직(노선·방향 필터링, retry, 응답 포맷)을 그대로 적용하고 HTTP 라이브러리만 `http.client`로 교체하여 배포. 실제 열차 조회 파라미터로 cold start invoke.

```
INIT_START Runtime Version: python:3.12.mainlinev2.v11
START RequestId: b38bcf7c  timestamp: 1780843656801
{"message":"열차 조회 요청 수신","line":"수인분당선","direction":"인천 방면","station":"달월"}  timestamp: 1780843656802
{"message":"서울시 API 응답","httpStatus":200,"attempt":1}  timestamp: 1780843656868
{"message":"도착 목록 수신","count":3}  timestamp: 1780843656868
{"message":"노선/방향 필터링 결과","matched":1}  timestamp: 1780843656868
{"message":"응답 반환","trains":1}  timestamp: 1780843656868
REPORT Duration: 68.42 ms  Init Duration: 588.39 ms
```

<br>

### 결과 비교

| 항목 | get_train (httpx) | network_probe (http.client) |
| --- | --- | --- |
| Cold Start HTTP 구간 | ~2,397ms (평균) | **66ms** |
| Init Duration | ~653ms | ~588ms |
| 정상 응답 여부 | ✓ | ✓ |

→ Cold Start임에도 HTTP 구간이 66ms로 warm 수준과 동일. 필터링·응답 로직도 정상 동작.
→ **가설 확정: `httpx` 클라이언트 초기화 오버헤드가 cold start 응답지연의 실질적 원인.**

<br>

## 4. 해결

<br>

### 검토한 대안

| 방안 | 효과 | 기각 사유 | 결정 |
| --- | --- | --- | --- |
| EventBridge 주기적 핑 (Warm-up) | 일부 cold 감소 | Lambda 실행 환경 수거 시점을 AWS가 보장하지 않아 cold 완전 제거 불가. "줄이는" 수준 | 기각 |
| Provisioned Concurrency | cold 완전 제거 | 추가 비용 발생 (~$0.015/GB-시간). 대안 4로 해결 가능해 불필요 | 기각 |
| Lambda SnapStart | Init Duration 단축 | httpx 초기화 오버헤드(~2,400ms)는 해결 안 됨. 핵심 문제 미해결 | 기각 |
| **`http.client` 교체** | **httpx 초기화 ~2,400ms 제거. 추가 비용 0** | - | **채택** |

<br>

### 채택

`httpx.get()` 호출을 Python 표준 라이브러리 `http.client.HTTPConnection`으로 교체. 외부 의존성 없이 cold start 비용 제거.

**교체 범위**

| 함수 | 비고 |
| --- | --- |
| `get_train` | 서울시 API 호출 |
| `get_nearby_arrival` | 서울시 API 호출 |
| `pass_seat` | 서울시 API 호출 |

미교체: `auth`, `delete_user` - HTTPS 처리도 `http.client.HTTPSConnection`으로 가능하지만, 이번 범위는 서울시 API 호출 람다로 한정 (이후 별도 안건으로 처리).



**교체 가능 여부 사전 조사**

- 프로토콜: 서울시 API는 `http://`이므로 `HTTPConnection`으로 충분
- 사용 기능 (전부 대체 가능):

| 기능 | httpx | http.client |
| --- | --- | --- |
| GET 요청 | `httpx.get(url)` | `conn.request("GET", path)` |
| 상태코드 | `resp.status_code` | `response.status` |
| JSON 파싱 | `resp.json()` | `json.loads(response.read())` |
| 타임아웃 | `httpx.get(url, timeout=5)` | `HTTPConnection(host, port, timeout=5)` |

httpx의 async, HTTP/2, 쿠키, 인증, 리다이렉트 등 고급 기능은 해당 함수에서 사용하지 않음.

→ 실측 결과: [`02-개선결과.md`](./02-개선결과.md)
