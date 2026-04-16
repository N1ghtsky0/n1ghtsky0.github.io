---
title: X-Forwarded-For을 로그에 찍기까지
description: 클라이언트 IP가 확인되지 않는 상황에서 나의 생각과 해결까지
date: 2026-04-15 20:00:00 +0900
categories: [일상]
render_with_liquid: false
---

SI 프로젝트를 진행하며 사용자의 행동을 추적하기 위한 로깅용 AOP를 구현하게 되었다. Spring Boot 환경이었기에 AOP 자체는 금방 만들었지만, **클라이언트 IP가 모두 내부망(LB) IP로 출력되는 현상**이 발생했다.

우리 프로젝트의 인프라 구조는 다음과 같았다.
```text
클라이언트 <> 고객사 LB(제어 불가) <> Apache (L7 LB) <> Spring Boot (WAS)
```

고객사 측에서는 이미 LB에서 클라이언트 IP를 X-Forwarded-For(XFF) 헤더에 담아 보내도록 설정을 마쳤다고 했다. 하지만 Apache의 access_log나 WAS의 HttpServletRequest을 이용한 로그에서는 외부 IP가 아닌 내부 IP만 출력될뿐이었다.

처음에는 고객사의 설정을 의심했다. 우리가 확인할 수 있는 부분을 모두 확인했을 때, 내부 IP가 출력되고 있었기에 나왔던 생각이었다. 하지만 그 때의 나는 어플리케이션 (스프링부트) 수준에서의 로그와 설정만 확인한 상태였고 이번 기회에 Apache에 대해 공부할 수 있는 기회라 생각하고 설정을 차근차근 살펴보기로 했다.

```text
apache/conf
├── httpd.conf
└── extra/
    ├── 00-common.conf
    └── vhost.conf
```
httpd.conf에서는 extra 폴더의 설정들을 include하고 있었고, vhost.conf에는 가상 호스트 설정이 되어 있었다. 헤더 설정을 위한 모듈과 IfModule 설정도 httpd.conf 내에 되어 있었지만 access_log와 was 에서는 내부 IP만이 출력되고 있었다. 

때문에 고객사의 설정에 대해 문의를 하기로 내부적으로 얘기가 되었지만 여기서 나는 한 단계 더 깊이 들어가기로 했다. Apache 로그에 찍히기 전, 요청되는 모든 HTTP 헤더를 확인해보고 싶었다. **실제로 고객사에서 Apache로 요청을 전달할 때 XFF 가 없는 것인지 Apache 설정이 잘 못된 것인지 확실하게 원인을 발견하고 싶었기 때문이다.** 그렇게 찾아낸 방법이 **mod_log_forensic** 모듈이었다.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> `mod_log_forensic`은 아파치 웹 서버에서 발생한 요청의 **"모든 것"을 가감 없이 기록하여 수사(Forensic) 수준의 추적을 가능하게 하는 모듈**. 일반적인 access_log는 요청이 **완료된 후**에 기록되지만, mod_log_forensic은 요청이 **시작되는 시점**과 **종료되는 시점**을 모두 기록한다는 점이 가장 큰 차이점입니다.  
> (출처: 제미나이)
{: .prompt-info }
<!-- markdownlint-restore -->

해당 모듈을 활성화하고 요청에 대한 모든 헤더를 확인했을 때, `X-Forwarded-For: (클라이언트 IP), (LB IP)` 가 찍히는 것을 확인했다. 이 근거를 토대로 Apache 설정이 잘못되었음을 확정지을 수 있었고 다시 conf 파일들을 하나씩 살펴보았다.

원인은 대수롭지 않게 생각하고 미처 확인하지 못했던 00-common.conf 파일에 있었다. 해당 파일 내에서 XFF 헤더를 REMOTE_ADDR 로 덮어씌우는 설정이 되어있어 XFF 가 LB의 IP로 출력되는 현상이 발생했던 것이었다.
```text
...
RequestHeader set X-Forwarded-For %{REMOTE_ADDR}e
...
```
{: file='apache/conf/extra/00-common.conf'}

해당 설정을 주석 처리하고 Apache를 재시작하자마자, 그토록 원하던 클라이언트 IP가 로그에 정상적으로 찍히기 시작했다.

이번 경험을 통해서 얻은 것이 생각보다 많았다. 우선 Apache와 linux라는 개발자에게 있어서 필수에 가까운 기술 지식에 대해 배울 수 있었다. 그리고 **확실한 기술적 근거를 바탕으로 원인을 좁혀가며 문제를 해결한 경험**을 얻었다.

1. was에서 클라이언트 IP가 찍히지 않는 문제 발생
2. Apache 설정확인
3. 검증을 위해 가공되지 않은 요청에 대한 모든 헤더를 확인할 수 있는 방법 탐색
4. mod_log_forensic 을 통해 원인 확정
5. 문제 해결

이라는 과정을 통해 문제를 바라볼 수 있는 내 시야가 조금 더 넓어지는 느낌이 든다.
