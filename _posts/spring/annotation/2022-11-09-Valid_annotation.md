---
title:  "Spring @Valid Annotation"

excerpt: "Spring @Valid Annotation"

categories:

- Spring

tags:

- Spring annotation
- Valid

toc: true

toc_sticky: true

toc_label: "Spring valid annotation"

last_modified_at: 2022-11-09T09:59:00-05:00

---

## @Valid Annotation?

> JSR-303 표준 스펙으로 객체의 필드에 어노테이션을 다는 것만으로 필드값 검증 가능
- 예시

~~~java
@Data
public class Order {
    @NotNull
    private String issueName;
    @IssuesCode
    private String issueCode;
    @OrderPrice
    private Long price;
}
~~~
- 검증하고 싶은 필드에 어노테이션을 달아두면 검증 오류시 아래와 같이 응답

![](/assets/images/spring/valid/response_example.png)

---

## 동작원리

- Dispatcher Servlet -> Controller 로 요청이 전달될 때, 컨트롤러 메소드의 객체를 만들어주는 ArgumentResolver가 동작한다.
- 이 때, @Valid로 시작하는 어노테이션이 있을 경우 유효성을 검증한다.
- 따라서 Controller 계층에서만 동작하며 다른 계층(ex. Service)에서 검증을 수행하고 싶다면 @Validated를 사용해야한다.

---

## 호가 검증

- 검증 항목
  - 종목명
  - 종목코드
  - 호가가격

> 본 예시에서는 Custom Annotation과 Validator 까지 생성해본다.

---

## 컨트롤러 설정

- 검증을 수행하고자 하는 Controller에 @Valid 어노테이션 설정
~~~java
@PostMapping("/order/new")
Orderbook newOrder(@RequestBody @Valid Order order) {
    return orderService.newOrder(order);
}
~~~
---

## 검증 대상 필드 설정

- 검증하고자 하는 필드에 어노테이션 설정
- @NotNull, @NotBlank와 같은 Annotation은 기본 제공된다.
- [기본 제공 어노테이션 확인 링크](https://reflectoring.io/bean-validation-with-spring-boot/)
~~~java
@Data
public class Order {
  @NotNull
  private String issueName;
  @IssuesCode
  private String issueCode;
  @OrderPrice
  private Long price;
  private String orderType;
  private String orderCondition;
  private String buySell;

  public Orderbook toOrderbook(Order order) {
    return new Orderbook(order.issueName, order.issueCode, order.price, order.orderType, order.orderCondition, order.buySell);
  }
}
~~~
---

## Custom Annotation

- 기본제공 되는 Annotation 외에 검증로직을 만들고 싶다면 Custom Annotation을 만들고, Validator에 검증 로직을 구현할 수 있다.
~~~java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = IssuesCodeValidator.class)
public @interface IssuesCode {
    String message() default "종목 코드 오류";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
~~~
- @Target : 어노테이션 적용가능한 타입 지정 (Class, Field 등)
- @Retention : 어노테이션 사용 가능한 시점 (Annotation이 메모리에 올려지는 시점)
- @Constraint : 검증을 수행할 Validator 지정

---

## Validator

- isValid 함수를 오버라이딩하여 검증로직 작성
~~~java
public class IssuesCodeValidator implements ConstraintValidator<IssuesCode, String> {
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null || value == "") {
            return false;
        }
        return value.startsWith("KR") || value.startsWith("FU");
    }
}
~~~
- Not null과 Not blank 검증
- 현물인 경우 "KR" / 선물인 경우 "FU" 로 시작하는지 검증

---

## 테스트

~~~json
{
  "buySell": "buy",
  "issueCode": "KF001",
  "issueName": "삼성전자",
  "orderCondition": "FAS",
  "orderType": "LIMIT",
  "price": 50000
}
~~~
- 종목 코드를 KR이 아닌 KF로 시작하게 작성

---

## 결과

![](/assets/images/spring/valid/response_ugly.png)
- 어떤 값이 오류가 났는지 모르니까 답답하다.
- Controller Advice로 직접 리스폰스를 생성해야겠다.

---

## Controller Advice

- 검증이 실패하면 ConstraintValidator는 MethodArgumentNotValidException을 던진다.
- MethodArgumentNotValidException에 대한 예외처리를 만들어주자

~~~java
package com.moongle.minimatchingengine.advice;

import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@ControllerAdvice
public class exceptionController {
    @ExceptionHandler({MethodArgumentNotValidException.class})
    public ResponseEntity badRequestHandler(MethodArgumentNotValidException e) {
        List<ErrorObject> errors = new ArrayList<>();
        for(FieldError result : e.getBindingResult().getFieldErrors())
            errors.add(new ErrorObject(result.getField(), result.getDefaultMessage()));

        return ResponseEntity.badRequest().body(errors);
    }
}
~~~
- e.getBindingResult().getFieldErrors()의 필드 에러 리스트를 모아 해시맵 생성
- Key : 오류 필드명
- Value : 오류 필드 메세지

---

## 테스트

![](/assets/images/spring/valid/response_ugly.png)

- 마음이 편안해졌다.
