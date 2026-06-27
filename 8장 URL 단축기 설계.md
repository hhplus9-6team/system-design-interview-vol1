# 8장. URL 단축기 설계

## 1. 개요

긴 URL을 짧은 URL로 변환해 주는 **URL 단축기**를 설계했다.

예를 들어 다음과 같은 긴 URL이 있을 때,

```text
https://example.com/products/category/item?id=12345&utm_source=google
```

사용자에게는 아래처럼 짧은 URL을 제공한다.

```text
https://short.ly/abc123
```

사용자가 단축 URL에 접근하면 서버는 원래 URL을 찾아 리다이렉트한다.

겉보기에는 단순한 기능이지만, 실제로는 다음 문제를 고려해야 한다.

* 짧은 URL을 어떻게 생성할 것인가
* 중복되지 않는 값을 어떻게 보장할 것인가
* 대량의 읽기 요청을 어떻게 처리할 것인가
* 데이터베이스와 캐시는 어떻게 구성할 것인가
* 리다이렉트 상태 코드는 무엇을 사용할 것인가

---

## 2. 요구사항 정리

### 기능 요구사항

URL 단축기가 제공해야 할 핵심 기능은 두 가지다.

1. 긴 URL을 입력받아 짧은 URL을 생성한다.
2. 짧은 URL로 접근하면 원래 URL로 리다이렉트한다.

추가적으로 다음 기능도 고려할 수 있다.

* 사용자가 직접 단축 키 지정
* 단축 URL 만료 시간 설정
* 클릭 횟수 통계
* 악성 URL 차단
* 단축 URL 삭제 및 비활성화

### 비기능 요구사항

* 높은 가용성
* 빠른 리다이렉트 응답
* 단축 URL의 중복 방지
* 대규모 트래픽 처리
* 생성된 URL이 쉽게 예측되지 않도록 구성
* 장애가 발생해도 기존 단축 URL 조회 가능

URL 단축기는 일반적으로 **쓰기보다 읽기가 훨씬 많은 서비스**다.

URL은 한 번 생성되지만, 생성된 URL은 여러 번 조회될 수 있다.

```text
쓰기 요청: URL 생성
읽기 요청: URL 리다이렉트
```

따라서 읽기 성능을 높이는 것이 중요하다.

---

## 3. API 설계

### 단축 URL 생성

```http
POST /api/v1/urls
Content-Type: application/json
```

요청:

```json
{
  "longUrl": "https://example.com/products/123"
}
```

응답:

```json
{
  "shortUrl": "https://short.ly/abc123",
  "shortKey": "abc123"
}
```

서버 내부 처리 흐름은 다음과 같다.

```text
클라이언트
   ↓
긴 URL 전달
   ↓
단축 키 생성
   ↓
단축 키와 원본 URL 저장
   ↓
단축 URL 반환
```

### 원본 URL 조회 및 리다이렉트

```http
GET /abc123
```

응답:

```http
HTTP/1.1 302 Found
Location: https://example.com/products/123
```

서버는 `abc123`을 이용해 원본 URL을 찾고, 해당 URL로 사용자를 이동시킨다.

---

## 4. 301과 302 리다이렉트

URL 단축기에서는 보통 `301` 또는 `302` 상태 코드를 사용할 수 있다.

### 301 Moved Permanently

영구적인 리다이렉트를 의미한다.

브라우저나 검색 엔진이 리다이렉트 결과를 캐싱할 수 있다.

```http
HTTP/1.1 301 Moved Permanently
Location: https://example.com
```

장점:

* 동일 요청이 반복될 때 서버 요청을 줄일 수 있다.
* 서버 부하가 감소한다.

단점:

* 브라우저가 캐싱하면 이후 요청이 URL 단축 서버를 거치지 않을 수 있다.
* 클릭 통계를 정확하게 집계하기 어렵다.
* 원본 URL 변경 사항이 바로 반영되지 않을 수 있다.

### 302 Found

임시적인 리다이렉트를 의미한다.

```http
HTTP/1.1 302 Found
Location: https://example.com
```

장점:

* 요청이 URL 단축 서버를 계속 거친다.
* 클릭 통계 수집이 가능하다.
* 원본 URL 변경을 반영하기 쉽다.

단점:

* 매번 서버 요청이 발생한다.
* 301보다 서버 부하가 증가할 수 있다.

클릭 통계나 URL 변경 기능이 중요하다면 일반적으로 `302`가 적합하다.

---

## 5. 데이터 모델

가장 단순한 데이터 모델은 다음과 같다.

| 컬럼         | 설명             |
| ---------- | -------------- |
| id         | 내부 식별자         |
| short_key  | 단축 URL에 사용되는 키 |
| long_url   | 원본 URL         |
| created_at | 생성 시각          |
| expires_at | 만료 시각          |
| status     | 활성화 여부         |

예시:

```text
id: 10001
short_key: abc123
long_url: https://example.com/products/123
created_at: 2026-06-27 10:00:00
expires_at: null
status: ACTIVE
```

URL 조회 시에는 다음과 같이 검색한다.

```sql
SELECT long_url
FROM short_url
WHERE short_key = 'abc123'
  AND status = 'ACTIVE';
```

`short_key`로 조회가 자주 발생하므로 인덱스나 유니크 제약조건이 필요하다.

```sql
CREATE UNIQUE INDEX uk_short_url_short_key
ON short_url(short_key);
```

---

## 6. 단축 키 생성 방법

URL 단축기 설계에서 가장 중요한 부분은 **짧고 중복되지 않는 키를 생성하는 것**이다.

예를 들어 다음 문자열이 단축 키다.

```text
abc123
```

### 방법 1. 해시 함수 사용

원본 URL을 해시 함수에 입력한다.

```text
long URL
   ↓
Hash Function
   ↓
e92a84d97c...
```

결과 전체를 사용하면 너무 길기 때문에 앞의 일부만 잘라 사용한다.

```text
e92a84
```

하지만 일부 문자열만 사용할 경우 충돌이 발생할 수 있다.

```text
URL A → e92a84
URL B → e92a84
```

따라서 충돌이 발생하면 다른 문자를 추가하거나 다시 해싱해야 한다.

예:

```text
hash(longUrl)
hash(longUrl + salt)
hash(longUrl + retryCount)
```

해시 방식의 문제점은 다음과 같다.

* 해시 충돌 처리 필요
* 같은 URL 입력 시 같은 키가 생성될 수 있음
* 키 길이를 줄일수록 충돌 가능성이 증가함

---

### 방법 2. 숫자 ID를 Base62로 변환

데이터베이스의 고유 숫자 ID를 발급받은 뒤 Base62 문자열로 변환한다.

Base62는 다음 문자를 사용한다.

```text
0-9
a-z
A-Z
```

총 62개의 문자를 사용할 수 있다.

```text
10 + 26 + 26 = 62
```

예를 들어 숫자 ID가 `125`라면 Base62로 변환한 결과를 단축 키로 사용한다.

```text
125 → 21
```

실제 결과는 구현한 문자 테이블에 따라 달라질 수 있다.

장점:

* 동일한 숫자 ID에서 항상 동일한 값 생성
* 충돌이 발생하지 않음
* 짧은 문자열 생성 가능
* 구현과 검증이 비교적 단순함

단점:

* 숫자 ID 생성 시스템이 필요함
* 생성 순서를 추측할 수 있음
* 단일 데이터베이스의 자동 증가 값을 사용하면 병목이 발생할 수 있음

---

## 7. Base62의 저장 공간

길이가 6인 Base62 문자열로 표현 가능한 URL 수는 다음과 같다.

```text
62^6 = 56,800,235,584
```

약 568억 개의 단축 URL을 표현할 수 있다.

길이가 7이면 다음과 같다.

```text
62^7 = 3,521,614,606,208
```

약 3조 5천억 개를 표현할 수 있다.

따라서 6~7자리의 Base62 문자열만으로도 매우 많은 URL을 생성할 수 있다.

다만 모든 문자열을 사용할 수 있는 것은 아니다.

* 금지어
* 욕설
* 오해를 일으킬 수 있는 문자열
* 예약된 경로
* 시스템 API 경로

예를 들어 다음 값들은 제외해야 할 수 있다.

```text
admin
login
api
health
fuck
```

---

## 8. ID 생성 전략

Base62 방식을 사용하려면 먼저 중복되지 않는 숫자 ID가 필요하다.

### 데이터베이스 Auto Increment

가장 단순한 방식이다.

```text
INSERT 수행
   ↓
DB에서 ID 생성
   ↓
ID를 Base62로 변환
```

장점:

* 구현이 단순함
* 중복이 발생하지 않음

단점:

* 단일 DB에 요청이 몰릴 수 있음
* 데이터베이스가 ID 생성의 병목이 됨
* 데이터베이스 장애 시 URL 생성이 불가능함

소규모 서비스에서는 충분히 사용할 수 있지만, 대규모 서비스에서는 별도 ID 생성 전략이 필요하다.

### 분산 ID 생성기

다음과 같은 방식을 고려할 수 있다.

* Snowflake 방식
* Redis `INCR`
* 데이터베이스 ID 구간 할당
* 전용 키 생성 서비스

예를 들어 서버마다 ID 범위를 미리 할당할 수 있다.

```text
서버 A: 1 ~ 1,000,000
서버 B: 1,000,001 ~ 2,000,000
서버 C: 2,000,001 ~ 3,000,000
```

각 서버는 할당받은 범위 내에서 ID를 생성하기 때문에 매 요청마다 중앙 데이터베이스에 접근하지 않아도 된다.

---

## 9. 전체 시스템 구조

기본적인 URL 단축기 구조는 다음과 같다.

```text
                  ┌──────────────┐
                  │    Client    │
                  └──────┬───────┘
                         │
                         ▼
                  ┌──────────────┐
                  │ Load Balancer│
                  └──────┬───────┘
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
       ┌─────────────┐       ┌─────────────┐
       │ URL Server  │       │ URL Server  │
       └──────┬──────┘       └──────┬──────┘
              │                     │
              └──────────┬──────────┘
                         │
                  ┌──────▼───────┐
                  │    Cache     │
                  │    Redis     │
                  └──────┬───────┘
                         │ Cache Miss
                         ▼
                  ┌──────────────┐
                  │   Database   │
                  └──────────────┘
```

### URL 생성 흐름

```text
1. 클라이언트가 긴 URL을 전달한다.
2. 서버가 고유 ID를 생성한다.
3. ID를 Base62 문자열로 변환한다.
4. shortKey와 longUrl을 DB에 저장한다.
5. 단축 URL을 클라이언트에 반환한다.
```

### URL 조회 흐름

```text
1. 클라이언트가 단축 URL에 접근한다.
2. 서버가 Redis에서 shortKey를 조회한다.
3. 캐시에 값이 있으면 즉시 원본 URL을 반환한다.
4. 캐시에 없으면 DB에서 조회한다.
5. 조회 결과를 Redis에 저장한다.
6. 클라이언트를 원본 URL로 리다이렉트한다.
```

---

## 10. 캐시가 중요한 이유

URL 단축기는 동일한 URL이 반복해서 조회될 가능성이 크다.

예를 들어 SNS에서 하나의 단축 URL이 공유되면 수십만 명이 같은 URL을 요청할 수 있다.

매 요청마다 DB에 접근하면 데이터베이스 부하가 커진다.

따라서 다음 데이터를 Redis에 캐싱한다.

```text
Key: short:url:abc123
Value: https://example.com/products/123
```

조회 과정:

```text
GET short:url:abc123
```

캐시에 값이 없으면 DB를 조회하고 캐시에 저장한다.

```text
Cache Miss
   ↓
DB 조회
   ↓
Redis SET
   ↓
302 Redirect
```

대표적인 캐시 전략은 Cache Aside 패턴이다.

```java
String longUrl = redis.get(shortKey);

if (longUrl == null) {
    longUrl = repository.findLongUrlByShortKey(shortKey)
        .orElseThrow(NotFoundException::new);

    redis.set(shortKey, longUrl);
}

return longUrl;
```

---

## 11. 캐시 만료 정책

URL에 만료 시간이 있다면 Redis TTL도 URL 만료 시간과 맞춰야 한다.

예를 들어 단축 URL이 24시간 뒤 만료된다면 다음과 같이 저장할 수 있다.

```text
SET short:url:abc123 https://example.com EX 86400
```

영구 URL이라도 캐시에는 적절한 TTL을 두는 것이 좋다.

이유는 다음과 같다.

* URL이 삭제되거나 수정될 수 있음
* 오래 사용되지 않는 데이터를 캐시에서 제거해야 함
* 캐시 메모리는 제한적임

모든 URL을 Redis에 저장하기보다는 자주 조회되는 URL을 중심으로 유지하는 것이 효율적이다.

---

## 12. 데이터베이스 확장

URL 개수가 증가하면 하나의 데이터베이스에 모든 데이터를 저장하기 어려워질 수 있다.

이 경우 `short_key`를 기준으로 샤딩할 수 있다.

```text
hash(shortKey) % shardCount
```

예:

```text
hash("abc123") % 4 = 2
```

따라서 해당 데이터는 2번 샤드에 저장한다.

```text
Shard 0
Shard 1
Shard 2 ← abc123
Shard 3
```

이 방식은 데이터를 여러 데이터베이스에 비교적 균등하게 분배할 수 있다.

다만 샤드 수가 변경되면 기존 데이터의 위치가 대량으로 변경될 수 있다. 이를 완화하기 위해 Consistent Hashing을 고려할 수 있다.

---

## 13. 동일한 URL 요청 처리

같은 긴 URL이 여러 번 입력될 때 두 가지 정책을 선택할 수 있다.

### 항상 새로운 단축 URL 생성

```text
https://example.com
→ abc123

https://example.com
→ xyZ789
```

장점:

* 사용자별 URL 관리 가능
* 캠페인별 통계 수집 가능
* 만료 시간이나 권한을 다르게 설정 가능

단점:

* 동일 URL이 여러 번 저장됨
* 저장 공간이 증가함

### 항상 같은 단축 URL 반환

```text
https://example.com
→ abc123

https://example.com
→ abc123
```

장점:

* 중복 저장 감소
* 저장 공간 절약

단점:

* 사용자별 통계 구분이 어려움
* URL 옵션을 개별적으로 설정하기 어려움
* 긴 URL을 기준으로 별도의 인덱스나 해시가 필요함

실제 서비스에서는 일반적으로 요청마다 새로운 단축 URL을 생성하는 방식이 더 유연하다.

---

## 14. 동시성 문제

여러 서버가 동시에 단축 키를 생성하더라도 중복이 발생하면 안 된다.

다음 두 요청이 동시에 같은 키를 생성할 수 있다.

```text
서버 A → abc123 생성
서버 B → abc123 생성
```

애플리케이션에서 중복 여부를 확인하는 것만으로는 충분하지 않다.

```text
1. 중복 확인
2. 저장
```

두 요청이 동시에 중복 확인을 통과할 수 있기 때문이다.

따라서 데이터베이스의 유니크 제약조건을 최종 방어선으로 사용해야 한다.

```sql
ALTER TABLE short_url
ADD CONSTRAINT uk_short_key UNIQUE (short_key);
```

충돌이 발생하면 새로운 키를 생성해 재시도한다.

```text
키 생성
   ↓
INSERT
   ↓
중복 오류 발생?
   ├─ 아니오 → 성공
   └─ 예 → 새 키 생성 후 재시도
```

Base62와 고유 숫자 ID를 사용한다면 기본적으로 키 충돌 가능성을 제거할 수 있다.

---

## 15. 잘못된 요청 처리

존재하지 않는 단축 URL로 요청이 들어올 수 있다.

```http
GET /unknown
```

이 경우 다음과 같이 응답할 수 있다.

```http
HTTP/1.1 404 Not Found
```

만료된 URL이라면 `410 Gone`을 사용할 수도 있다.

```http
HTTP/1.1 410 Gone
```

차이는 다음과 같다.

* `404 Not Found`: 존재 여부를 알 수 없거나 찾을 수 없음
* `410 Gone`: 과거에는 존재했지만 현재는 제거됨

서비스 정책에 따라 모든 오류를 404로 통일할 수도 있다.

---

## 16. 보안과 악용 방지

URL 단축기는 피싱이나 악성 사이트를 숨기는 용도로 악용될 수 있다.

사용자는 단축 URL만 보고 최종 목적지를 확인하기 어렵기 때문이다.

따라서 다음 기능이 필요할 수 있다.

* 악성 URL 검사
* 허용되지 않은 도메인 차단
* 사용자별 생성 횟수 제한
* IP 기반 Rate Limit
* 신고 기능
* 단축 URL 미리보기 기능
* 위험 URL 접근 전 경고 화면
* 생성 API 인증

예를 들어 사용자별 생성 요청을 제한할 수 있다.

```text
사용자당 1분에 20개 생성 가능
IP당 1시간에 100개 생성 가능
```

URL 조회는 공개적으로 허용하되, 생성 API는 더 엄격하게 제한하는 방식이 적절하다.

---

## 17. 통계 처리

클릭 수를 저장하기 위해 리다이렉트 요청마다 데이터베이스를 갱신하면 성능이 저하될 수 있다.

```sql
UPDATE short_url
SET click_count = click_count + 1
WHERE short_key = 'abc123';
```

인기 URL 하나에 트래픽이 집중되면 하나의 행에 지속적인 쓰기 경합이 발생한다.

따라서 클릭 이벤트는 비동기로 처리하는 것이 좋다.

```text
사용자 요청
   ↓
원본 URL 조회
   ↓
302 Redirect
   ↓
클릭 이벤트 발행
   ↓
Kafka
   ↓
통계 처리 서버
   ↓
통계 DB 저장
```

리다이렉트는 빠르게 처리하고, 통계 데이터는 이벤트 큐를 통해 별도로 처리한다.

통계 데이터로는 다음 정보를 수집할 수 있다.

* 클릭 시간
* 사용자 IP
* User-Agent
* Referrer
* 국가 또는 지역
* 브라우저
* 운영체제

---

## 18. 장애 상황 고려

### Redis 장애

Redis 조회에 실패하면 DB에서 원본 URL을 조회한다.

```text
Redis 장애
   ↓
DB Fallback
   ↓
302 Redirect
```

캐시 장애가 전체 서비스 장애로 이어지지 않도록 해야 한다.

### 데이터베이스 장애

읽기 복제본을 사용하면 조회 트래픽을 분산할 수 있다.

```text
Write → Primary DB
Read  → Read Replica
```

단, URL 생성 직후 바로 조회하는 경우 복제 지연 문제가 발생할 수 있다.

이를 해결하기 위해 다음 방식을 고려할 수 있다.

* 생성 직후 Redis에 값을 저장
* 일정 시간 동안 Primary DB에서 조회
* 읽기 일관성이 필요한 요청만 Primary 사용

### 인기 URL 트래픽 집중

특정 단축 URL 하나에 트래픽이 몰리는 Hot Key 문제가 발생할 수 있다.

```text
short:url:abc123
```

대응 방법:

* CDN 또는 Edge Cache 사용
* Redis Replica 활용
* 로컬 캐시 추가
* 인기 URL 사전 캐싱
* 캐시 키 분산

---

## 19. 최종 설계 흐름

### URL 생성

```text
Client
  ↓ POST /api/v1/urls
Load Balancer
  ↓
URL Service
  ↓
Unique ID Generator
  ↓
Base62 Encoding
  ↓
Database 저장
  ↓
Redis 저장
  ↓
Short URL 반환
```

### URL 리다이렉트

```text
Client
  ↓ GET /abc123
Load Balancer
  ↓
URL Service
  ↓
Redis 조회
  ├─ Cache Hit
  │      ↓
  │   302 Redirect
  │
  └─ Cache Miss
         ↓
      Database 조회
         ↓
      Redis 저장
         ↓
      302 Redirect
```

### 클릭 통계

```text
URL Service
   ↓
Click Event 발행
   ↓
Message Queue
   ↓
Analytics Consumer
   ↓
통계 Database
```

---

## QnA
1. URL 단축기의 핵심 기능은 무엇인가?
2. URL 단축 서비스가 10년간 운영될 경우 저장해야 할 URL 개수는 어떻게 계산하는가?