---
title: 1. 명령줄 인터페이스 애플리케이션
author: Myeongjae Kim
date: 2022-03-12
category: Tutorial
layout: post
---

다짜고자 코드부터 작성해봅시다! Node.js로 아래의 요구사항을 만족하는 명령줄 인터페이스 애플리케이션을 구현하면 됩니다.

데이터는 메모리상에만 존재해도 됩니다. 즉, 애플리케이션을 재시작 할 때마다 초기화되어도 상관 없습니다.

## 비기능적 요구사항

프로젝트를 진행하기 위해 필요한 의존성을 설치합니다.

- TypeScript: [https://www.typescriptlang.org/download](https://www.typescriptlang.org/download)
- typescript-eslint: [https://typescript-eslint.io/docs/linting/](https://typescript-eslint.io/docs/linting/)
- Prettier: [https://prettier.io/docs/en/install.html](https://prettier.io/docs/en/install.html)
- ts-node: [https://github.com/TypeStrong/ts-node#installation](https://github.com/TypeStrong/ts-node#installation)
- nodemon: [https://github.com/remy/nodemon#installation](https://github.com/remy/nodemon#installation)
- ts-jest: [https://github.com/kulshekhar/ts-jest](https://github.com/kulshekhar/ts-jest)
- @johanblumenberg/ts-mockito: [https://github.com/johanblumenberg/ts-mockito](https://github.com/johanblumenberg/ts-mockito)
  - [jest.mock 보다 ts-mockito 사용하기 (feat. Node.js) - 기억보단 기록을](https://jojoldu.tistory.com/638)

설정 완료한 프로젝트 예시: [https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/setup](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/setup)

## 기능적 요구사항

목록 조회, 상세 조회, 쓰기가 가능한 게시판을 명령줄 인터페이스 애플리케이션.

### 첫 화면

```
1) 목록 조회
2) 쓰기
x) 종료

선택: <cursor> 
```

- `<cursor>`는 문자열이 아니라 사용자 입력을 기다리는 커서를 의미합니다.
- 1, 2, x 중에 하나를 입력하고 엔터를 누르면 다음 화면으로 넘어갑니다.
- 입력값이 올바르지 않을 경우 사용자에게 아래처럼 다시 입력을 요구합니다.

```
입력이 올바르지 않습니다. 아래 선택지의 맨 앞 문자를 입력해주세요.

1) 목록 조회
2) 쓰기
x) 종료

선택: <cursor> 
```

### 목록 조회

목록 조회에서는 존재하는 모든 게시글의 제목을 출력합니다.

```
1) 물고기는 존재하지 않는다
2) 호킹의 빅 퀘스천에 대한 간결한 대답
3) 나와 너
4) 상자 밖에 있는 사람
x) 뒤로가기

선택: <cursor> 
```

- 게시글이 하나도 없다면 `x) 뒤로가기` 선택지만 출력합니다.
- 숫자를 입력하면 해당 게시글의 상세 조회 화면으로 넘어갑니다.
- x를 입력하고 엔터를 누르면 이전 화면으로 돌아갑니다.
- 입력값이 올바르지 않을 경우 사용자에게 아래처럼 다시 입력을 요구합니다.

```
입력이 올바르지 않습니다. 아래 선택지의 맨 앞 문자를 입력해주세요.

1) 물고기는 존재하지 않는다
2) 호킹의 빅 퀘스천에 대한 간결한 대답
3) 나와 너
4) 상자 밖에 있는 사람
x) 뒤로가기

선택: <cursor> 
```

### 상세 조회

상세 조회 화면에서는 선택한 게시글의 제목과 내용을 출력합니다.

```
제목: 물고기는 존재하지 않는다
내용: ‘물고기는 존재하지 않는다’ 읽고 있는데 과학 에세이에서 갑자기 그것이 알고싶다로 장르 전환함..

엔터 키를 누르면 이전 화면으로 되돌아갑니다.
```

- 제목과 내용을 출력합니다.
- 엔터 키를 누르면 이전 화면으로 되돌아갑니다.

### 쓰기

제목과 내용을 입력하면 게시글을 저장하고 id를 출력합니다. 줄바꿈은 지원하지 않습니다.

#### 제목 입력 화면

```
제목: <cursor>
```

#### 내용 입력 화면

```
제목: 물고기는 존재하지 않는다
내용: <cursor>
```

#### 내용 입력 완료

```
제목: 물고기는 존재하지 않는다
내용: ‘물고기는 존재하지 않는다’ 읽고 있는데 과학 에세이에서 갑자기 그것이 알고싶다로 장르 전환함..

게시글을 저장했습니다. id: 1

엔터 키를 누르면 이전 화면으로 되돌아갑니다.
```

## 너무 간단한 애플리케이션 아니에요?

맞아요.. 간단한 비즈니스 로직을 가진 애플리케이션입니다. 이 애플리케이션의 기능적 요구사항은 튜토리얼 내내 바뀌지 않습니다. 하지만 프로젝트를 둘러싼 비즈니스의 환경이 바뀝니다.

명령줄 대신 웹브라우저에서 동작해야 한다든지, 애플리케이션을 재시작해도 데이터가 남아있어야 한다든지, 애플리케이션을 프론트엔드와 백엔드로 나눈다든지, 웹브라우저가 아니라 모바일 앱에서 작동하게 한다든지.

애플리케이션이 작동하는 환경이 바뀌더라도 기능적 요구사항은 그대로이기 때문에 비즈니스 로직은 변경이 없는것이 가장 이상적입니다.

하지만 유연한 아키텍처라는 환상을 좇다가 일정이 늘어지면 안됩니다! 가장 중요한 것은 약속한 시간 내에 제품을 전달하는 것입니다. 현실 세계에선 수많은 이유로 일정이 늦어지는게 다반사지만, 어찌됐든 개발자는 일정을 산정해서 고객과 약속을 해야하니까요.

요구사항을 보고 시간이 얼마나 걸릴지 가늠해보세요, 그리고 실제 시간이 얼마나 걸렸는지도 측정해보세요(+ 일정 산정은 경력과 상관없이 모든 개발자들에게 어렵습니다).

어떤 작업을 해야 할지 상세하게 나눌 수록 정확한 일정 산정을 할 수 있습니다. Jira를 활용해도 좋고, 노션도 좋고, GitHub의 Projects도 좋습니다. 기능 하나를 여러 개의 작은 티켓으로 쪼개서 작업해보세요.
