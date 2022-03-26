---
title: 4. 제어의 역전 컨테이너 (with InversifyJS)
author: Myeongjae Kim
date: 2022-03-12
category: Tutorial
layout: post
---

https://tc39.es/ecma262/multipage/reflection.html#sec-reflect-object

리플렉션으로 `AppliationByStateManager` 생성하는 코드

메타프로그래밍이라고 불린다. 프로그래밍을 프로그래밍한다.

https://ko.wikipedia.org/wiki/메타프로그래밍





제어의 역전과 의존성 주입이라는 단어에 대해서 정리. IoC 컨테이너? IoC 프레임워크? DI 프레임워크?

https://stackoverflow.com/a/6551303/14659782

https://martinfowler.com/articles/injection.html#InversionOfControl

IoC의 좀 더 특정한 경우를 지칭하는 DI를 쓰자.

IoC 단어 탄생: https://martinfowler.com/bliki/InversionOfControl.html 아래의 어원학 참고

우리 예제에서도 InversifyJS는 의존성 주입의 역할로 사용한다. 하지만 InversifyJS 소개 페이지에는 'A powerful and lightweight inversion of control container
for JavaScript & Node.js apps powered by TypeScript.' 라고 적혀있으므로 챕터 이름을 제어의 역전 컨테이너라고 지었다.




라이브러리와 프레임워크의 차이. 토비 글 인용. Angular, Vue는 프레임워크, React는 라이브러리

의존성 주입에 필요한 정보가.. 이름만 있으면 되네?

InversifyJS

To Be Developed...

inversify로 의존성 엮을 때 인터페이스 만들어서 하자. 클래스로 만들어서 모조리 생성한 다음에 메서드 하나씩 실행하는 식으로 의존성 등록하기.

Symbol도 사용해서. Symbol이 무엇인지, 원시값. 동일한 문자열이 들어간 Symbol은 같은거냐? 이것도 테스트.




자바같은 강타입 언어들은 런타임에도 타입 정보에 접근할 수 있기 때문에 이 작업들을 더 간편하게 수행할 수 있다. 따로 이름을 지정해주지 않아도 프레임워크에서
클래스와 인터페이스 정보를 바탕으로 이름을 정해서 의존성 주입을 수행한다.

node나 웹브라우저는 자바스크립트를 실행하지만, deno같이 타입스크립트를 바로 실행하는 플랫폼에서는 Inversify보다 더 진보된 프레임워크가 탄생할 수 있습니다.

https://deno.land/x/di@v0.1.1#newable-services

deno의 di 모듈 사용법을 보면, InversifyJS와는 다르게 타입과 인젝터블을 수동으로 `bind`하지 않아도 잘 작동하는 것을 볼 수 있다.


