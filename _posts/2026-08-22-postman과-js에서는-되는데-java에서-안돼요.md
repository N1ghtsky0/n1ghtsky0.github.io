---
title: postman과 js에서는 되는데 java에서 안돼요
description: 외부 api 연동과정에서 발생한 문제 원인 분석과 해결과정
date: 2026-08-22 20:00:00 +0900
categories: [cs]
tags: [java]
render_with_liquid: false
---

## RestTemplate으로 외부 공공 API 호출 시 발생한 401 오류, 원인은 URL 인코딩이었다

**한 줄 요약**: 외부 공공 API를 RestTemplate으로 호출했을 때 Postman·JS(ajax)에서는 정상 응답이 오는데, Java RestTemplate에서만 계속 `401 [no body]` 오류가 발생했다. 원인은 API 키가 아니라, RestTemplate이 URL을 처리하는 과정에서 발생한 **URL 인코딩 문제**였다. 정확히는 인코딩이 두 번 적용되는 문제(Double Encoding)와, 그 다음엔 반대로 특수문자가 아예 인코딩되지 않는 문제가 순차적으로 발생했다.

---

### 문제 상황

동료로부터 외부 공공 API 연동 코드에서 발생한 오류를 함께 분석해달라는 요청을 받았다. 상황은 다음과 같았다.

- 공공 데이터포털에서 제공하는 REST API를 연동하는 과정에서 발생
- Postman과 JavaScript의 ajax로 호출하면 정상적으로 응답이 오는데, Java의 `RestTemplate`으로 호출하면 `401 [no body]` 에러가 발생

동료는 Postman과 JS 요청 시 만들어진 HTTP 요청을 참고해서 Java 코드를 구현했다고 했다. 공공 데이터포털 측에 문의한 결과, 401은 **API 사용 키(서비스 키)에 문제가 있을 때 반환되는 응답**이라는 답을 받았다. 문제는 세 가지 방식(Postman, JS, Java) 모두 **동일한 서비스 키**를 사용하고 있었다는 점이었다.

즉, 키 자체는 문제가 아닐 가능성이 높았고, 오히려 RestTemplate이 요청을 만드는 과정에서 뭔가 달라지고 있다고 의심하는 게 자연스러운 방향이었다.

### 원인 파악: 인터셉터로 실제 요청 로깅하기

RestTemplate은 기본적으로 요청·응답 내용을 로그로 남기지 않는다. 그래서 실제로 어떤 요청이 나가는지 확인하기 위해 `ClientHttpRequestInterceptor`를 추가했다.

```java
restTemplate.getInterceptors().add((request, body, execution) -> {
    log.info("========== REQUEST ==========");
    log.info("Method : {}", request.getMethod());
    log.info("URI : {}", request.getURI());
    log.info("Headers : {}", request.getHeaders());
    log.info("Body : {}", new String(body, StandardCharsets.UTF_8));
    log.info("=============================");
    return execution.execute(request, body);
});
```

이 로그 덕분에 실제로 RestTemplate이 어떤 URI를 만들어서 보내는지 눈으로 확인할 수 있었고, 여기서부터 진짜 원인을 찾기 시작했다.

### 1차 시도: Double Encoding 문제

공공 데이터포털에서 발급하는 서비스 키는 URL 쿼리 스트링에 들어가야 했다. 그런데 기존 코드는 이렇게 되어 있었다.

```java
String url = UriComponentsBuilder
        .fromHttpUrl(baseUrl)
        .queryParam("serviceKey", serviceKey)
        .queryParam("returnType", "JSON")
        .encode()
        .toUriString();
```

인터셉터 로그의 `request.getURI()` 값을 확인해보니 원인이 보였다. 공공 데이터포털의 서비스 키는 발급 시점에 **이미 URL 인코딩이 되어있는 문자열**이다. 그런데 위 코드에서 `.encode()`를 호출하면서 이 문자열을 다시 한 번 인코딩해버리고 있었다.

```
// 원래 서비스 키에 포함된 문자
=

// 1차 인코딩(발급 시점에 이미 적용됨)
%3D

// RestTemplate에서 .encode()로 한 번 더 인코딩된 결과 (Double Encoding)
%253D
```

즉 `=`가 `%3D`로, 그리고 이미 `%3D`였던 부분이 다시 `%253D`로 이중 인코딩되면서 서버가 인식하는 키 값 자체가 원본과 완전히 달라진 것이다.

해결을 위해 서비스 키를 먼저 디코딩한 뒤 `queryParam`에 넣도록 수정했다.

### 2차 시도: 이번엔 인코딩이 안 되는 문제

1차 수정 후 다시 테스트했는데, 여전히 동일하게 `401 [no body]` 오류가 발생했다. 로그를 확인해보니 이번엔 반대 방향의 문제였다. **인코딩이 되어야 할 특수문자가 인코딩되지 않고** 있었다.

서비스 키 원문에는 `+`, `/`, `=` 이렇게 세 개의 특수문자가 포함되어 있었다. 그런데 수정 후 로그를 보면:

```
// 수정 전 서비스 키 원문 (예시, 실제 값 아님)
abc+de/f=gh

// 수정 후 실제 요청 URI에 찍힌 값
abc+de/f%3Dgh
```

`=`만 `%3D`로 정상 인코딩되었고, `+`와 `/`는 문자 그대로 URI에 들어가고 있었다. `queryParam`에 값을 그대로 넘기면 UriComponentsBuilder가 이를 **URI 문법의 일부로 해석**해버려서, `+`나 `/`처럼 URI 구조에서 특별한 의미를 가질 수 있는 문자는 인코딩 대상에서 제외되는 것이 원인이었다.

### 최종 해결: URI Template Variable + Strict Encoding

두 문제를 종합하면 결국 필요한 건 "서비스 키의 특수문자를 **URI의 문법 구조가 아니라, 외부에서 주입된 순수한 데이터 값**으로 취급해서 인코딩하는 것"이었다. 이를 위해 쿼리 파라미터에 값을 직접 넣는 대신, URI 템플릿 변수를 사용하도록 수정했다.

```java
String url = UriComponentsBuilder
        .fromHttpUrl(baseUrl)
        .queryParam("serviceKey", "{serviceKey}") // 값 대신 템플릿 변수 사용
        .queryParam("returnType", "JSON")
        .encode()
        .buildAndExpand(serviceKey) // 여기서 실제 값을 주입하며 인코딩
        .toUriString();
```

이렇게 하면 `serviceKey`에 담긴 값은 URI의 문법적 요소가 아니라 단순한 데이터로 취급되기 때문에, `+`, `/`, `=` 등 모든 특수문자가 누락 없이 한 번만 정확하게 인코딩된다. 수정 후 테스트에서는 정상적으로 API가 호출되었고, 원하는 응답도 받을 수 있었다.

### 정리

결과만 놓고 보면 단순한 URL 인코딩 문제였지만, 원인은 두 단계에 걸쳐 있었다.

| 단계 | 문제                                                     | 증상                                                            |
| ---- | -------------------------------------------------------- | --------------------------------------------------------------- |
| 1차  | 이미 인코딩된 서비스 키를 그대로 `.encode()` 처리        | Double Encoding (`=` → `%3D` → `%253D`)                         |
| 2차  | `queryParam`에 값을 직접 대입                            | `+`, `/` 등 특수문자가 인코딩되지 않고 그대로 전달              |
| 해결 | URI Template Variable(`{serviceKey}`) + `buildAndExpand` | 서비스 키를 opaque한 데이터 값으로 취급해 정확히 한 번만 인코딩 |

이번 경험에서 얻은 교훈은 두 가지다.

1. **Postman·브라우저는 성공하는데 Java 클라이언트만 실패한다면, API 서버를 의심하기 전에 두 클라이언트가 실제로 동일한 HTTP 요청을 만들고 있는지부터 비교해봐야 한다.** RestTemplate 같은 Java HTTP 클라이언트 라이브러리는 URL을 다루는 과정에서 자체적으로 인코딩/디코딩 로직을 한 번 더 태우는 경우가 많고, 이게 겉보기엔 똑같아 보이는 요청을 실제로는 다르게 만든다.
2. **RestTemplate은 기본적으로 요청·응답 로그를 남기지 않으므로, 문제 상황에서는 `ClientHttpRequestInterceptor`로 실제 나가는 요청을 눈으로 확인하는 것이 가장 확실한 디버깅 방법이다.**
