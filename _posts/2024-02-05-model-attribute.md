---
layout: post
title: ModelAttribute의 바인딩 과정
date: 2024-02-05 23:11:23 +09:00
author: Kyong
categories: Spring
banner:
  image: "/assets/images/post/banners/moon_night.jpg"
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: Spring Springboot
top: 1
sidebar: []
---
## @ModelAttribute은 어떻게 요청데이터를 바인딩할까?

클라이언트에서 전달되는 Request 데이터를 객체에 바인딩해주는 `@ModelAttribute`를 사용하는 중, 값이 바인딩이 되지 않아 null로 입력되는 현상이 발생하였습니다.

> java.lang.IllegalStateException: Cannot resolve parameter names for constructor private
> org.example.blancstudy.controller.request.
> Review(java.lang.String,java.lang.String)


#### 1. 문제상황
- Springboot 3.2.1로 프로젝트를 진행하였습니다.
- 빌드 방식은 IntelliJ IDEA 방식을 사용하였습니다.
- 멘토링에 대한 리뷰를 수정할때, POST method로 받는 데이터는 다음과 같습니다.
  - `id`, `contents`, `ratings`
- `@ModelAttribute`로 바인딩하기 위한 객체의 구조는 다음과 같습니다.

```Java
 @Getter
 public class RequiredEditReview{
    
    private Long id;
    
    @NotBlank(message= "상세내용을 입력해주세요.")
    @Size(min = 1, max = 200, message = "1자 이상 200자 이하로 작성해 주세요.")
    private String contents;
    
    private Integer ratings;
    
    @Builder
    private RequiredEditReview(Long id, String contents, Integer ratings){
        this.id = id;
        this.contents = contents;
        this.ratings = ratings;   
    }

 }
```
- form POST 요청을 받는 Controller의 메서드는 다음과 같습니다.
```Java
  @PostMapping("/me/review/edit")
  public String editReview(@Validated @ModelAttribute RequiredEditReview requiredEditReview,
      BindingResult bindingResult, Model model) {
    if (bindingResult.hasErrors()) {
      model.addAttribute("requiredEditReview", requiredEditReview);
      return viewResolver.getViewPath("user", "edit-review");
    }
    userService.editReview(requiredEditReview);
    return "redirect:/users/me/applied-mentorings";
  }
```

- `editReview` 메서드의 파라미터로 `RequiredEditReview` 객체에 값이 제대로 바인딩 되지 않는 상황이 발생하였습니다.

#### 2. 문제분석
`@ModelAttribute`에 대해 Spring 공식문서에서는 다음과 같이 설명하고 있습니다.
> - Instantiated through a default constructor.
> - Instantiated through a “primary constructor” with arguments that match to Servlet request parameters. Argument names are determined through runtime-retained parameter names in the bytecode.

`@ModelAttribute`는 기본 생성자를 통해 인스턴스화됩니다.

서블릿 요청 매개변수와 일치하는 인수를 사용하여 `primary constructor`를 통해 인스턴스화됩니다. 인수 이름은 바이트코드의 런타임 유지 매개변수 이름을 통해 결정됩니다.

즉, `@ModelAttribute`에 Request 데이터를 바인딩 하기위해서는 기본생성자와 setter 메서드를 통해 객체를 생성한 후 바인딩하거나,
Request Parameter와 일치하는 생성자를 찾아서 사용하게 됩니다.

하지만 `RequiredEditReview`에는 왜 데이터가 바인딩되지 않은걸까요? 문제해결을 위해 저는 ModelAttribute의 바인딩 과정을 담당하는 `ModelAttributeMethodProcessor`를 디버깅 하였습니다.

<img src="/assets/images/post/modelattribute/modelAttribute.png" alt="model-attribute">

우선, `resolveArgument` 메서드에서 출발하고, 내부에서 `constructAttribute`를 호출합니다.

<img src="/assets/images/post/modelattribute/modelAttribute2.png" alt="model-attribute-2">

![model-attribute-3](/assets/images/post/modelattribute/modelAttribute3.png)

`WebRequestDataBinder`의 `construct`는 `ValueResolver`를 매개변수로 갖는 `construct`를 호출합니다.
![request-data-binder](/assets/images/post/modelattribute/requestDataBinder.png)

`DataBinder`의 `construct`는 `createObject`를 통해 객체를 생성하게됩니다.

![constructor](/assets/images/post/modelattribute/construct.png)

`createObject`에서 객체를 생성하는데, `BeanUtils.getResolvableConstructor`를 호출해 생성자를 조회합니다.

![create-object](/assets/images/post/modelattribute/createObject.png)

조회 순서는 다음과 같습니다.
1. findPrimaryConstructor(clazz)를 호출하여 주요 생성자(primary constructor)를 찾습니다.
  - 해당 메서드는 kotlin 전용이므로, 현재는 신경쓰지 않습니다.
2. 클래스의 모든 생성자 배열을 얻습니다.
3. 생성자가 하나라면, 해당 생성자를 반환합니다.
4. 명시된 생성자가 없다면 기본생성자를 반환합니다.

생성자를 조회한 다음엔 `BeanUtils.getParameterNames`를 호출해 생성자 파라미터 이름을 배열로 반환받아야합니다.

![getParameterNames](/assets/images/post/modelattribute/getParameterNames.png)
1. 생성자에 `@ConstructorProperties` 어노테이션이 붙어있는지 조회합니다.
2. 존재하지 않는다면, 생성자의 파라미터 명을 조회합니다.

`2번` 상황에서 `paramNames == null` 이 되어 `IllegalStateException`이 발생한 것입니다.

그렇다면 해결방법은 무엇이 존재할까요?

#### 3. 문제해결
##### 기본생성자와 setter 메서드
가장 기본적으로, 기본생성자와 setter 메서드를 사용해 값을 바인딩해줄 수 있습니다.
하지만 Request 데이터를 바인딩 하는 Dto 객체는 데이터의 불변성을 유지해야하기때문에, setter 메서드를 사용한 바인딩은 올바르지 못한 방법이라고 생각하였습니다.

##### `@ConstructorProperties`
두번째 방법으로는 `BeanUtils.getParameterNames`의 첫번째 줄에서 검증하는 `@ConstructorProperties`를 사용하는 것입니다.
해당 어노테이션은 생성자 파라미터와 프로퍼티 간의 명시적인 매핑을 제공합니다.

```Java
 @Getter
 public class RequiredEditReview{
    
    private Long id;
    
    @NotBlank(message= "상세내용을 입력해주세요.")
    @Size(min = 1, max = 200, message = "1자 이상 200자 이하로 작성해 주세요.")
    private String contents;
    
    private Integer ratings;
    
    @Builder
    @ConstructorProperties({"id","contents","ratings"})
    private RequiredEditReview(Long id, String contents, Integer ratings){
        this.id = id;
        this.contents = contents;
        this.ratings = ratings;   
    }

 }
```
### 4. 추가 조사
#### 그렇다면 왜? 생성자 파라미터 이름을 조회하지 못하는가?
Gradle로 빌드하거나 스프링부트 3.2 이하 버전에서는 데이터 바인딩이 `@ConstructorProperties`없이도 잘 이뤄진다는 사실을 확인하였습니다.


왜 3.2버전부터 바인딩이 되지않는가에 대한 궁금증이 생겨 검색 하던 중, 스프링 6 업데이트 노트를 찾게 되었습니다.

[Upgrading to Spring Framework 6.x](https://github.com/spring-projects/spring-framework/wiki/Upgrading-to-Spring-Framework-6.x#parameter-name-retention)

>LocalVariableTableParameterNameDiscoverer has been removed in 6.1. Consequently, code within the Spring Framework and Spring portfolio frameworks no longer attempts to deduce parameter names by parsing bytecode.

스프링 6부터는 바이트코드에서 파라미터이름을 추론하는 방식을 사용하지 않는다고 합니다.

Gradle로 빌드를 하게된다면 자바 컴파일러 옵션인 `-parameters`가 활성화 되어있어서 파라미터 이름을 바이트코드에 포함시킵니다.

하지만 IntelliJ IDEA 빌드는 해당 옵션이 비활성화 상태이고, Spring6부터 `LocalVariableTableParameterNameDiscoverer`가 삭제되었기 때문에 메서드 파라미터에서 이름을 추론하지 못해 발생한 문제였습니다.
