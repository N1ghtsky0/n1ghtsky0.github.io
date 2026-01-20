---
title: Java, 왜 NullPointerException 일까
description: Java의 NullPointerException 에 대해 갑자기 든 생각을 정리한 글
date: 2026-01-20 20:10:00 +0900
categories: [cs]
tags: [java]
render_with_liquid: false
---

Java 는 개발자가 직접적으로 포인터를 선언하고 사용하는 기능이 없다. 그런데 왜 Null 로 인해 발생하는 예외의 클래스명은 Null**Pointer**Exception일까? 라는 의문이 생겼고 얕게나마 알고있는 지식들로 생각해보았다. 

먼저 NPE는 언제 발생하는가? NPE는 코드내에서 null 인 객체를 참조하려할 때 발생한다. 
```java
String sample = null;
System.out.println(sample.toString());
```
위의 코드와 같이 같이 null 에서 toString() 을 호출할 때, NPE 가 발생한다. 

자바에는 두가지 데이터 타입이 존재한다. **기본타입(primitive type)**과 **참조타입(reference type)** 이 있으며, NPE 는 참조타입의 동작방식으로 인해 발생하는 예외이다. 참조 타입의 경우 변수 선언 후 초기화 시 값이 직접 담기는게 아니라 생성된 객체의 메모리 주소가 저장된다. 그리고 변수를 참조할 때 저장된 메모리 주소로 접근하여 생성된 객체를 참조하는 방식으로 동작한다. 그럼 위의 코드를 다시 살펴봤을 때 sample 이 선언되고 null 로 초기화가 되었다. 그 의미는 바라보는 메모리 주소가 null 이라는 것이고 가리킬 주소가 없다는 뜻이 된다. 그 상황에서 참조를 시도하면 존재하지 않는 메모리 주소를 참조하려 했기에 Null**Pointer**Exception 이 발생하게 된다. 즉, 자바를 사용할 때 직접적으로 메모리 주소를 제어하거나 사용하지는 않지만 내부 동작방식의 경우 포인터를 사용하는 식으로 되어 있기에 클래스명을 그에 맞춰 작성한 것이라고 생각한다.
