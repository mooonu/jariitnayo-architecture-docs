# auth Lambda 응답지연 분석

[2026.06.19]



## 0. 요약

- auth 람다 함수 cold 실행시간이 Apple 8초, 카카오 6.5초 수준
- 원인은 httpx 의 클라이언트 초기화 + https 엔드포인트로 인해 5초정도의 병목이 발생
- httpx -> http.client 로 변경. `_verify_apple_token` / `_verify_kakao_token` / `_fetch_apple_refresh_token` 3곳 모두 교체하고 `import httpx` 제거



## 1. 문제 상황

CloudWatch 대시보드의 람다 Duration-Max  위젯에서 `jariitnayo-auth`가 3초에서 점점 늘어나 
4일만에 **8초로 관찰**됨. 

정확한 빈도와 패턴을 확인하기 위해 Logs Insights로 다음 쿼리를 실행

```
SOURCE "arn:aws:logs:ap-northeast-2:415464488514:log-group:/aws/lambda/jariitnayo-auth"
START=-604800s END=0s
| filter @type = "REPORT"
| filter @duration >= 8000
| fields @timestamp, @requestId, @duration
| sort @timestamp desc
```

결과: **지난 1주일간 8초 이상 케이스 4건. 모두 Apple 신규 회원가입 요청.**

### 추가 조사

Kakao 신규 회원가입 요청에 대한 실행시간도 조사하였음.

결과: **Kakao 신규 회원가입 요청도 6.5초 수준의 지연 관찰**



## 2. 원인 분석

실행 로그를 분석하여 병목이 발생하는 구간 특정

```
10:57:57.081Z  애플 로그인 프로세스 시작
10:58:03.238Z  애플 토큰 검증 완료           ← 6,157ms
...
10:58:05.471Z  애플 refresh_token 보관 완료  ← 1,415ms
...
10:58:05.557Z  REPORT Duration: 8,476ms  Init Duration: 909ms
```

- **6,157ms - 코드 상 `_verify_apple_token` 구간**
- 1,415ms - 코드 상 `_fetch_apple_refresh_token` 구간 (신규 가입에만 발생)

### 병목 구간 코드 분석

lambda/auth/handler.py 조사

```python
def _verify_apple_token(identity_token: str, user_identifier: str):
    resp = httpx.get("https://appleid.apple.com/auth/keys", timeout=5.0) # 유력
    jwks = resp.json()                                                 
    payload = jwt.decode(identity_token, jwks, algorithms=["RS256"], ...)
```

외부 HTTPS를 호출하는 유일한 네트워크 I/O인 `httpx.get()` 을 **원인으로 의심**

### 가설

#### 비슷한 응답지연 분석 사례

2026-06-07 에 비슷한 cold start 응답지연 분석이 있었고, 외부 API를 호출하는 람다 함수가 cold start 시 평균 2,443ms 응답 지연을 보였다. 원인은 httpx 클라이언트 초기화 오버헤드였다.
따라서 이번 원인도 **동일하게 클라이언트 초기화 오버헤드로 의심**된다



## 3. 테스트

가설 검증을 위해 두 라이브러리를 격리하여 테스트를 진행한다.

### 테스트 결과

요약: **http.client로 변경 시 Duration이 4.7s 가량 감소**한다.

#### httpx

```
attempt 1 (cold)  httpx_elapsed 5,554ms  response_elapsed 508ms  jwt_decode 394ms
attempt 2 (warm)  httpx_elapsed   587ms  response_elapsed 487ms  jwt_decode   0.4ms
attempt 3 (warm)  httpx_elapsed   566ms  response_elapsed 494ms  jwt_decode   0.4ms
REPORT  Duration 7,108ms  Init 863ms
```

| 호출       | 외부 elapsed | httpx 내부 elapsed | **차이 (클라이언트 초기화 비용)** | jwt.decode |
| ---------- | ------------ | ------------------ | --------------------------------- | ---------- |
| 1회 (cold) | 5,554ms      | 508ms              | **5,046ms**                       | 394ms      |
| 2회 (warm) | 587ms        | 487ms              | 100ms                             | 0.43ms     |
| 3회 (warm) | 566ms        | 494ms              | 72ms                              | 0.41ms     |

#### http.client

```
attempt 1 (cold)  DNS 21 + TCP 149 + TLS 200 + TTFB 301 + transfer 0 = total 844ms  jwt_decode 237ms
attempt 2 (warm)  DNS 19 + TCP 141 + TLS 148 + TTFB 295 + transfer 0 = total 783ms  jwt_decode   0.9ms
attempt 3 (warm)  DNS 33 + TCP 143 + TLS 145 + TTFB 287 + transfer 0 = total 767ms  jwt_decode   5.3ms
REPORT  Duration 2,664ms  Init 790ms
```

| 호출       | DNS  | TCP   | TLS   | TTFB  | Transfer | **Total** | jwt.decode |
| ---------- | ---- | ----- | ----- | ----- | -------- | --------- | ---------- |
| 1회 (cold) | 21ms | 149ms | 200ms | 301ms | 0ms      | **844ms** | 237ms      |
| 2회 (warm) | 19ms | 141ms | 148ms | 295ms | 0ms      | 783ms     | 0.89ms     |
| 3회 (warm) | 33ms | 143ms | 145ms | 287ms | 0ms      | 767ms     | 5.29ms     |

#### httpx vs http.client 비교

| 구간              | httpx       | http.client | 차이                           |
| ----------------- | ----------- | ----------- | ------------------------------ |
| **Cold 1회 JWKS** | **5,554ms** | **844ms**   | **-4,710ms**                   |
| Warm 평균 JWKS    | 575ms       | 775ms       | +200ms                         |
| jwt.decode cold   | 394ms       | 237ms       | -157ms (변동성 범위)           |
| Init Duration     | 862ms       | 790ms       | -72ms (httpx 임포트 비용 추정) |

-> **원인은 cold start 시 httpx 클라이언트 초기화로 인한 병목**이 발생하는 것.



## 4. 해결

### 검토한 대안

| 방안 | 효과 | 기각 사유 | 결정 |
| --- | --- | --- | --- |
| SSM Parameter Store 캐싱 | Apple JWKS 호출 자체 제거 | boto3 cold 비용 ~1초로 효과 미미 (test 람다 측정 결과 **0.1초 차이**) | 기각 |
| 모듈 메모리 캐시 | 첫 호출 외 fetch 0 | auth는 사실상 100% cold start (유저당 최초 1회 또는 30일 만료 후 1회 호출) | 기각 |
| **`http.client` 교체** | 클라이언트 초기화 제거. 코드 변경 최소. | - | **채택** |

### 채택

`httpx.get()` / `httpx.post()` 호출을 Python 표준 라이브러리 `http.client.HTTPSConnection`으로 교체.

**교체 범위 (auth 람다 3곳)**

| 호출부 | 변환 |
| --- | --- |
| `_verify_apple_token` | `httpx.get` → `http.client GET` |
| `_verify_kakao_token` | `httpx.get` + Authorization 헤더 → `http.client GET` |
| `_fetch_apple_refresh_token` | `httpx.post(data=...)` → `http.client POST` + `urlencode` |

→ 실측 결과: [`03-개선결과.md`](./03-개선결과.md)
