---
layout: post
title: Map을 사용한 Enum 조회성능 개선
date: 2024-02-13 22:10:25 +09:00
author: Kyong
categories: Java
banner:
  image: "/assets/images/post/banners/moon_night.jpg"
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: Java
sidebar: []
---
# Enum 조회 성능 개선기
팀 프로젝트를 진행하며 Enum을 사용해 값을 정의하는 경우가 많았습니다.

저는 은행이름과 은행코드를 Enum으로 사용하였는데요,
```java
@Getter
public enum BankCode{

  HANA("하나은행", "081"),
  KDB("산업은행", "002"),
  IBK("기업은행", "003"),
  KB("국민은행", "004"),
  NH("농협은행", "011"),
  WOORI("우리은행", "020"),
  SC_KOREA("제일은행", "023"),
  CITY("시티은행", "027"),
  DAEGU("대구은행", "032"),
  KWANGJU("광주은행", "034"),
  JEJU("제주은행", "035"),
  JEONBUK("전북은행", "037"),
  BNK_KYONGNAM("경남은행", "039"),
  MG("새마을금고", "045"),
  SHINHAN("신한은행", "088"),
  KAKAO("카카오뱅크", "090");
  
  private final String bankName;
  private final String code;
  
  BankCode(String bankName,String code){
    this.bankName = bankName;
    this.code = code;
  }
  
}
```

클라이언트로부터 `bankName`을 전달받아 DB에 저장합니다. 정산 작업시 예금주 조회나 입금이체를 진행할때에는 `code`가 필요하기때문에 은행명을 사용해서 Enum 값을 조회하는 메서드가 필요했습니다.

```java
public BankCode find(String bankName) {
  for (BankCode bankCode : values()) {
    if(bankCode.bankName.equals(bankName)){
      return bankCode;
    }
  }
  throw new BankNameNotFoundException();
}
```

간단하게 위와같은 메서드를 작성해서 사용하였는데, 매번 은행코드를 조회하기위해 Enum 배열을 순회하게 되어 시간복잡도 O(n)으로 효율적이지 못한 코드라는 생각이 들었습니다.

어떻게 개선을 할 지 고민하던 중, `Map` 자료구조를 사용해 `bankName`을 key로 설정하면 효율적이지 않을까? 라는 생각이 들어 메서드를 수정하고 성능 테스트를 진행하였습니다.
```java

private static final Map<String,BankCode> BANK_CODE_MAP = 
  Arrays.stream(values()).collect(Collectors.toMap(BankCode::getBankName),Function.identity()); 

public BankCode find(String bankName){
  return Optional.ofNullable(BANK_CODE_MAP.get(bankName)).orElseThrow(BankNameNotFoundException::new);
}
```

```java
@Slf4j
class BankCodeTest {

  @Test
  @DisplayName("3억건 forEach BankCode 조회 테스트")
  void findBankCodeByForEachTest() {
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    for (int i = 0; i < 200_000_000; i++) {
      BankCode.findBy("카카오뱅크");
    }
    stopWatch.stop();
    log.info("ForEach ElapsedTime :: {} ms", stopWatch.getTotalTimeMillis());
  }

  @Test
  @DisplayName("3억건 HashMap BankCode 조회 테스트")
  void findBankCodeByHashMapTest() {
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    for (int i = 0; i < 200_000_000; i++) {
      BankCode.find("카카오뱅크");
    }
    stopWatch.stop();
    log.info("Map ElapsedTime :: {} ms", stopWatch.getTotalTimeMillis());
  }

}
```

![TestResult](https://kihong1602.github.io/storage/images/enumsearch/enum_search_testpng.png)

테스트결과는 forEach 탐색이 22929ms, HashMap 탐색이 964ms로, 약 95%의 성능 개선이 이뤄졌습니다.
