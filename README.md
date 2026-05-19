# 📁 project-configs

> 📖 [중앙 Wiki에서 보기](https://github.com/9oogle/.github/wiki/project-configs)

Spring Cloud Config Server에서 사용하는 각 서비스의 설정 파일 저장소입니다.

---

## 📂 디렉토리 구조

```
configs/
├── common/                  # 모든 서비스에 공통 적용되는 설정
│   ├── application.yaml         # 공통 기본 설정
│   └── application-test.yaml    # 공통 테스트 설정
├── gateway-server/
│   ├── application.yaml
│   └── application-local.yaml
├── mentoring-service/
│   ├── application.yaml
│   └── application-local.yaml
├── order-service/
│   ├── application.yaml
│   └── application-local.yaml
└── user-service/
    ├── application.yaml
    └── application-local.yaml
```

---

## 🗄️ DB 구조 (논리적 스키마 분리)

모든 서비스는 **같은 PostgreSQL DB(`goggles`)를 공유하고, 스키마로 논리적으로 분리**합니다.

| 서비스 | 스키마 |
|--------|--------|
| `user-service` | `user` |
| `mentoring-service` | `mentoring` |
| `order-service` | `order` |

DB URL 형식은 모든 서비스에서 동일합니다.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:${DB_PORT:5432}/${DB_NAME:goggles}
  jpa:
    properties:
      hibernate:
        default_schema: {서비스별 스키마 이름}
```

> ⚠️ 서비스별 스키마가 DB에 미리 생성되어 있어야 합니다. (`CREATE SCHEMA IF NOT EXISTS user;` 등)

---

## ⚙️ 공통 설정 (`common/application.yaml`)

`common/application.yaml`은 **모든 서비스에 자동으로 적용**됩니다.
각 서비스 파일에서 동일한 키를 선언하면 서비스 설정이 우선 적용됩니다.

### 공통으로 관리되는 항목

| 항목 | 내용 |
|------|------|
| **Spring Cloud LoadBalancer** | Ribbon 비활성화 |
| **OpenFeign Circuit Breaker** | 전체 활성화 |
| **Feign 타임아웃** | connectTimeout: 5000ms / readTimeout: 2000ms |
| **Eureka 클라이언트** | `${EUREKA_SERVER_URL1}`, `${EUREKA_SERVER_URL2}` 환경변수 기반 |
| **Actuator** | `refresh, health, info, prometheus` 노출 |
| **Swagger(SpringDoc)** | `/v3/api-docs`, `/swagger-ui.html` 공통 경로 설정 |
| **로깅** | root 레벨 INFO |

---

## ✅ 서비스별 설정 작성 규칙

### 1. 공통 설정은 각 서비스 파일에 중복 작성하지 마세요

아래 항목들은 **`common/application.yaml`에서 이미 관리**되므로 각 서비스 파일에 다시 쓸 필요 없습니다.

- `spring.cloud.loadbalancer`, `spring.cloud.openfeign.circuitbreaker`
- `feign.client.config.default` (기본 타임아웃)
- `eureka.client.service-url.defaultZone`
- `management.endpoints.web.exposure.include`
- `springdoc.api-docs.path`, `springdoc.swagger-ui.path`
- `logging.level.root`

### 2. Feign 타임아웃 커스터마이징

서비스별로 특정 클라이언트의 타임아웃을 바꾸고 싶다면, **공통 설정을 건드리지 말고** 서비스 파일에 해당 클라이언트만 추가하세요.

```yaml
# 예: order-service/application.yaml
feign:
  client:
    config:
      lecture-service:        # 이 클라이언트만 별도 설정
        connectTimeout: 3000
        readTimeout: 8000
```

> ⚠️ `feign.client.config.default`를 서비스 파일에서 재정의하면 공통 설정 전체가 덮어써집니다.

### 3. Eureka 인스턴스 설정

모든 서비스는 아래 형식으로 통일합니다.

```yaml
eureka:
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${random.value}
```

> `hostname` 기반 설정(`prefer-ip-address: false`)은 사용하지 않습니다.

### 4. Zipkin(분산 추적) 엔드포인트 설정

반드시 `management` 하위 키를 사용하세요. (Spring Boot 3.x 표준)

```yaml
# ✅ 올바른 방식
management:
  zipkin:
    tracing:
      endpoint: http://${MONITORING_HOST:localhost}:9411/api/v2/spans

# ❌ 잘못된 방식 (Spring Boot 2.x 방식, 적용 안 됨)
zipkin:
  tracing:
    endpoint: ...
```

### 5. JPA ddl-auto 설정

| 환경 | 권장 값 |
|------|---------|
| 운영 (`application.yaml`) | `validate` |
| 로컬 (`application-local.yaml`) | `create` 또는 `update` |
| 테스트 (`application-test.yaml`) | `create-drop` |

> ⚠️ 운영 프로파일에서 `create`, `update`를 사용하면 스키마가 의도치 않게 변경될 수 있습니다.

### 6. Swagger operationsSorter

`common`의 기본값은 `alpha`(알파벳순)입니다.
서비스별로 다른 정렬 방식이 필요하면 해당 서비스 파일에서만 재정의하세요.

```yaml
springdoc:
  swagger-ui:
    operationsSorter: method  # 서비스 파일에서만 재정의
```

---

## 🌍 환경변수 목록

서비스 운영 시 아래 환경변수를 주입해야 합니다.

| 환경변수 | 설명 | 기본값 |
|----------|------|--------|
| `EUREKA_SERVER_URL1` | Eureka 서버 1 호스트 | `localhost` |
| `EUREKA_SERVER_URL2` | Eureka 서버 2 호스트 | `localhost` |
| `DB_HOST` | DB 호스트 | `localhost` |
| `DB_PORT` | DB 포트 | `5432` |
| `DB_NAME` | DB 이름 | `goggles` |
| `DB_USERNAME` | DB 사용자 | - |
| `DB_PASSWORD` | DB 비밀번호 | - |
| `KAFKA_1_HOST` ~ `KAFKA_3_HOST` | Kafka 브로커 호스트 | `localhost` |
| `MONITORING_HOST` | Zipkin 호스트 | `localhost` |
| `ZIPKIN_ENDPOINT` | Zipkin 전체 URL | `http://localhost:9411/api/v2/spans` |

---

## 🔒 주의사항

- 이 저장소에는 **실제 시크릿(비밀번호, API 키 등)을 절대 커밋하지 마세요.**
- 민감한 값은 반드시 환경변수(`${VAR_NAME}`)로 주입받아야 합니다.
- 기본값(`${VAR:defaultValue}`)은 **로컬 개발 편의용**으로만 사용하고, 운영 환경에서는 반드시 환경변수를 주입하세요.
