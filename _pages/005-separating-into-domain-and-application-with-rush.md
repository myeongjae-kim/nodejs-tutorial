---
title: 5. 도메인과 애플리케이션의 분리 (with Rush)
author: Myeongjae Kim
date: 2022-03-12
category: Tutorial
layout: post
---

## Lerna와 Rush

node 기반 프로젝트를 모노레포로 구성할 수 있게 도와주는 도구가 여럿 있습니다. 저는 실무에서 2020년에 [Lerna](https://lerna.js.org)를 사용해
모노레포로 프로젝트를 구성한 경험이 있습니다. JVM 진영의 [gradle](https://gradle.org) 멀티모듈 구조에 익숙해 있던 터라 모노레포의 개념을 이해하기는
쉬웠지만, Lerna로 프로젝트를 구성해보니 설정이 복잡했고 `npm`만으로는 해결할 수 없는 문제가 있어서
[직접 심볼릭 링크를 업데이트하는 쉘 스크립트를 작성해서 사용](https://myeongjae.kim/blog/2020/06/12/copy-symbolic-link-of-relative-path)했습니다(이 문제는 Lerna와 yarn workspace를 함께 사용하는 방식으로도 해결이 가능합니다).

에듀테크 스타트업 밀당의 정성대님이 작성하신 [Rush로 프론트엔드 모노레포 도입기](https://medium.com/mildang/rush로-프론트엔드-모노레포-도입기-5da0c5bc9b30)
에서는 제가 말씀드릴 수 있는 것보다 훨씬 풍부하고 깊은 이야기가 있습니다. 정성대님의 글에서는 여러 개의 모노레포 관리 도구들을 비교하고
[Rush](https://rushjs.io)를 선택해서 React 버전을 점진적으로 업데이트합니다. 저도 2020년에 Lerna와 Rush중에 고민하다가 Rush가 더 나아보여서
Rush로 프로젝트 구성을 시도해봤는데, 이 때는 Next.js와 Rush가 호환이 되지 않아 Lerna를 사용했었습니다. 정성대님의 예시에서는 Rush에서
Next.js가 잘 작동합니다(그냥 그 때 제가 설정을 제대로 못한 것 같습니다). 

### PNPM?

pnpm은 npm같은 패키지 매니저입니다. Rush는 [npm](https://docs.npmjs.com), [pnpm](https://pnpm.io), [yarn](https://yarnpkg.com) 모두 사용할 수 있지만 pnpm을
권장합니다 ([NPM vs PNPM vs Yarn \| Rush](https://rushjs.io/pages/maintainer/package_managers/)). npm은 옛날 버전인 4.5.0을
권장하고, yarn은 사용은 가능하지만 [왠지 자신없어 하는 모습](https://rushjs.io/pages/maintainer/package_managers/#considerations-for-yarn)입니다.
우리의 게시판 애플리케이션 정도는 아마 npm 최신 버전에서도 잘 작동할 것 같아서 일단 프로젝트는 Rush와 npm을 사용한 뒤, 모노레포 구성 후에
npm에서 pnpm으로 마이그레이션을 해보겠습니다.

pnpm은 npm을 사용하면서 겪을 수 있는 [Phantom dependencies](https://rushjs.io/pages/advanced/phantom_deps/) 문제와
[NPM doppelgangers](https://rushjs.io/pages/advanced/npm_doppelgangers/) 문제를 해결했습니다. 이 문제들은 npm이 모든 의존성을
`node_modules`에 평평flat하게(=의존성의 의존성, 의존성의 의존성의 의존성... 이 모두 `node_modules` 디렉토리로 올라오는hoisted 것) 구성하기
때문에 발생합니다. 우리 프로젝트도 `dependencies`와 `devDependencies` 합쳐서 18개의 의존성을 사용하지만 `node_modules`에는 435개의 디렉토리가
있습니다.

<figure>
    <img alt="Super Massive node_modules" src="/res/11-super-massive-node-modules.webp" />
    <figcaption>초 거대 <del>블랙홀</del>node_modules<br />출처: <a target="_blank" href="https://dev.to/leoat12/the-nodemodules-problem-29dc">https://dev.to/leoat12/the-nodemodules-problem-29dc</a></figcaption>
</figure>

```json-doc
{
  ...,
  "dependencies": {
    "@reduxjs/toolkit": "^1.8.0",
    "inversify": "^6.0.1",
    "mobx": "^6.4.2",
    "node": "^17.5.0",
    "reflect-metadata": "^0.1.13"
  },
  "devDependencies": {
    "@johanblumenberg/ts-mockito": "^1.0.32",
    "@types/jest": "^27.4.1",
    "@typescript-eslint/eslint-plugin": "^5.14.0",
    "@typescript-eslint/parser": "^5.14.0",
    "dependency-cruiser": "^11.4.1",
    "eslint": "^8.11.0",
    "eslint-config-prettier": "^8.5.0",
    "jest": "^27.5.1",
    "nodemon": "^2.0.15",
    "prettier": "2.5.1",
    "ts-jest": "^27.1.3",
    "ts-node": "^10.7.0",
    "typescript": "^4.6.2"
  }
}
```

```
$ tree node_modules -L 1
node_modules
├── @ampproject
├── @babel
...
├── yargs
├── yargs-parser
└── yn

435 directories, 0 files
```

pnpm은 `node_modules` 디렉토리를 평평하지 않게non-flat 구성해서 [Phantom dependencies](https://rushjs.io/pages/advanced/phantom_deps/)
문제와 [NPM doppelgangers](https://rushjs.io/pages/advanced/npm_doppelgangers/) 문제를 해결합니다. `node_modules/.pnpm`에
의존성들을 설치하고, `node_modules`이하에는 심볼릭 링크를 생성해서 `node_modules/.pnpm`이하의 의존성에 연결합니다:
[https://pnpm.io/motivation#creating-a-non-flat-node_modules-directory](https://pnpm.io/motivation#creating-a-non-flat-node_modules-directory)

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">🟢Node.js Tip<br><br>Why store separate copies of packages if you can have a `single` referencable one?<br><br>That&#39;s the idea behind PNPM which makes it a performant, space-efficient, and faster alternative to NPM/Yarn.<br><br>Trust me, your hard disk will thank you for this weight loss exercise <a href="https://t.co/mltti8CFAX">pic.twitter.com/mltti8CFAX</a></p>&mdash; Hem (@HemSays) <a href="https://twitter.com/HemSays/status/1434921646083563525?ref_src=twsrc%5Etfw">September 6, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## Rush를 사용해보자

아래 내용은 Rush의 [Maintainer Tutorials](https://rushjs.io/pages/maintainer/setup_new_repo/)를 참고해서 작성했습니다.

### 디렉토리 구조 변경

새로운 디렉토리를 만들어 `rush init`을 합니다. 비어있는 디렉토리가 아니면 `rush init`은 불가능합니다.

```
~/nodejs-tutorial-example$ npm install -g @microsoft/rush
~/nodejs-tutorial-example$ cd .. && mkdir nodejs-tutorial-example-rush && cd nodejs-tutorial-example-rush
~/nodejs-tutorial-example-rush$ rush init
```

아래처럼 파일들이 생성됩니다.

```
nodejs-tutorial-example-rush
├── .gitattributes
├── .gitignore
├── .travis.yml
├── common
│   ├── config
│   │   └── rush
│   │       ├── .npmrc
│   │       ├── .npmrc-publish
│   │       ├── .pnpmfile.cjs
│   │       ├── artifactory.json
│   │       ├── build-cache.json
│   │       ├── command-line.json
│   │       ├── common-versions.json
│   │       ├── experiments.json
│   │       ├── rush-plugins.json
│   │       └── version-policies.json
│   └── git-hooks
│       └── commit-msg.sample
└── rush.json
```

npm을 사용하므로 `rush.json`을 열어서 `pnpmVersion` 대신 `npmVersion`을 사용하도록 변경합니다.

```
$ npm --version
6.14.16
```

```json-doc
// rush.json
{
  ...,
  // "pnpmVersion": "6.7.1",

  "npmVersion": "6.14.16",
  ...
}
```

그리고 `nodejs-example-tutorial` 프로젝트 전체를 `app/board-cli`(명렁줄 인터페이스로 작동하는 게시판 애플리케이션이라는 의미) 디렉토리 이하로
복사합니다.

```
~/nodejs-tutorial-example-rush$ mkdir app && cd app
~/nodejs-tutorial-example-rush/app$ cp -R ../../nodejs-tutorial-example board-cli
~/nodejs-tutorial-example-rush/app$ cd board-cli
```

그리고 `~/nodejs-tutorial-example-rush/app/board-cli/package.json` 파일을 열어서 `name` 속성을 `nodejs-tutorial-example`에서
`app-board-cli`로 변경합니다.

```json-doc
// ~/nodejs-tutorial-example-rush/app/board-cli/package.json
{
  "name": "nodejs-tutorial-example",
  ...
}
```

`~/nodejs-tutorial-example-rush/app/board-cli/.git` 디렉토리를 `~/nodejs-tutorial-example-rush/.git`으로 옮겨서 `git` 설정과
히스토리를 보존하고 변경사항을 커밋합니다.

```
~/nodejs-tutorial-example-rush/app/board-cli$ mv .git ../../
~/nodejs-tutorial-example-rush/app/board-cli$ cd ../../
~/nodejs-tutorial-example-rush$ git add .
~/nodejs-tutorial-example-rush$ git commit
```

### 애플리케이션 빌드 및 실행

`rush.json`의 `projects` 배열에 아래와 같이 프로젝트를 추가합니다.

```json-doc
{
  ...,
  "projects": [
    {
      "packageName": "app-board-cli",
      "projectFolder": "app/board-cli"
    }
  ]
}
```

이후에 `rush update`를 실행합니다.

```
~/nodejs-tutorial-example-rush$ rush update
```

아래와 같은 메시지가 나오면 성공입니다.

```
LINKING: app-board-cli
Purging ../nodejs-tutorial-example-rush/app/board-cli/node_modules

Linking finished successfully. (9.18 seconds)

Next you should probably run "rush build" or "rush rebuild"

Rush update finished successfully. (52.95 seconds)
```

`app/board-cli/dist/node_modules`를 확인해보면 의존성이 심볼릭 링크로 연결된 것을 확인할 수 있습니다.

```
~/nodejs-tutorial-example-rush$ ls -al app/board-cli/node_modules
...
lrwxr-xr-x 36 mj  2 Apr 15:57 ws -> ../../../common/temp/node_modules/ws
lrwxr-xr-x 45 mj  2 Apr 15:57 xdg-basedir -> ../../../common/temp/node_modules/xdg-basedir
lrwxr-xr-x 52 mj  2 Apr 15:57 xml-name-validator -> ../../../common/temp/node_modules/xml-name-validator
lrwxr-xr-x 42 mj  2 Apr 15:57 xmlchars -> ../../../common/temp/node_modules/xmlchars
lrwxr-xr-x 38 mj  2 Apr 15:57 y18n -> ../../../common/temp/node_modules/y18n
lrwxr-xr-x 41 mj  2 Apr 15:57 yallist -> ../../../common/temp/node_modules/yallist
lrwxr-xr-x 39 mj  2 Apr 15:57 yargs -> ../../../common/temp/node_modules/yargs
lrwxr-xr-x 46 mj  2 Apr 15:57 yargs-parser -> ../../../common/temp/node_modules/yargs-parser
lrwxr-xr-x 36 mj  2 Apr 15:57 yn -> ../../../common/temp/node_modules/yn
```

빌드 결과물이 담기는 디렉토리인 `app/board-cli/dist` 디렉토리를 제거한 뒤 `rush build`를 하고 애플리케이션을 실행해봅시다.

```
~/nodejs-tutorial-example-rush$ rm -rf app/board-cli/dist
~/nodejs-tutorial-example-rush$ rush build
~/nodejs-tutorial-example-rush$ app/board-cli
~/nodejs-tutorial-example-rush$ rushx start
1) 목록 조회
2) 쓰기
x) 종료

선택: 
```

실행이 잘 됩니다.

이제부터 `npm`의 커맨드는 잊으시는게 좋습니다. [`npm run` 대신 `rushx`](https://rushjs.io/pages/developer/everyday_commands/#rushx)를, 
[`npm install`대신 `rush add`](https://rushjs.io/pages/developer/modifying_package_json/)를 사용합니다. Rush로 관리하는
프로젝트에서 어떻게 개발하는지는 [Developer tutorials](https://rushjs.io/pages/developer/new_developer/)를 읽어보세요.

지금까지 작성한 코드는 [nodejs-tutorial-example:chapter-5-rush-setup](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-5-rush-setup)에서 확인할 수 있습니다.

## 도메인 패키지 분리

### 도메인 패키지 디렉토리 설계

챕터 2에서 육각형 아키텍처의 목적은 [비즈니스 로직을 순수하게 유지하는 것](/pages/002-hexagonal-architecture/#비즈니스-로직을-순수하게)이라고
말씀드렸습니다.

> `domain` 디렉토리와 `application` 디렉토리를 합쳐서 광의(廣義)의 도메인 영역이라고 할 수 있습니다. 오직 비즈니스만을 위한 코드가 존재하는,
> 세부사항에 오염되지 않도록 순수하게 유지해야 하는 영역입니다.
> 
> \- [2.육각형 아키텍처/Application](/pages/002-hexagonal-architecture/#application)

지금까지는 하나의 패키지 안에 사용자 인터렉션을 처리하는 `View`부터 순수한 도메인 영역까지 모두 관리했습니다. 이제 Rush로 프로젝트를 구성하도록
변경했으므로 도메인 영역을 다른 패키지로 분리해서 의존성의 방향을 이전보다 더 엄격하게 관리할 수 있습니다. 도메인 영역만 분리된 패키지로 구성할 수 있으므로
도메인 패키지를 재사용해서 여러 개의 애플리케이션을 만들 수도 있습니다.

`domain/board-domain` 패키지를 추가해서 `app/board-cli/src/article`의 `domain`과 `application` 디렉토리를 옮겨 도메인 패키지를
만든다면 디렉토리 구조를 어떻게 만들어야 할까요?

- `domain/board-domain/src/article/domain`
- `domain/board-domain/src/article/application`

그냥 그대로 옮기면 이렇게 두 개의 디렉토리가 생길텐데, 도메인 패키지의 `domain` 디렉토리라니.. 디렉토리 경로에서 중복이 발생하는 것 같아 영 찜찜하니까
`domain` 디렉토리를 `model`로 변경하겠습니다.

- `domain/board-domain/src/article/model`
- `domain/board-domain/src/article/application`

`model` 디렉토리는 '도메인 모델'을 의미하게 됩니다. 즉 엔티티만 있는 디렉토리입니다. `application` 디렉토리는 엔티티를 바탕으로 수행하는 비즈니스
로직이 들어있는 디렉토리가 됩니다. 그리고 디렉토리 구조에서 `application`이라는 단어는 이미 `app`이라는 디렉토리로 `app/board-cli`처럼 실행 가능한
애플리케이션이 있는 패키지의 부모 디렉토리로 사용하고 있으니, 도메인 패키지에서 `application` 디렉토리가 존재하는 것도 영 찜찜합니다.

도메인패키지의 `application` 디렉토리에는 `incoming` 포트와 `outgoing` 포트, 그리고 `incoming` 포트 인터페이스를 구현하는 서비스들이
들어있습니다. 외부 의존성 없이 엔티티와 포트로만 수행하는 비즈니스 로직은 도메인의 범주 안에 있기 때문에 굳이 `application` 디렉토리 아래로 넣어놓지 말고
그냥 밖으로 꺼내놓으면 어떨까요? 아래와 같은 트리 구조도 괜찮아 보입니다.

```
domain/board-domain/src/article
├── ArticleCommandService.ts
├── ArticleQueryService.ts
├── port
│    ├── incoming
│    │   ├── ArticleCreateUseCase.ts
│    │   ├── ArticleGetUseCase.ts
│    │   ├── ArticleListUseCase.ts
│    │   ├── ArticleRequest.ts
│    │   └── ArticleResponse.ts
│    └── outgoing
│        ├── ArticleLoadPort.ts
│        └── ArticleSavePort.ts
└── model
    ├── Article.ts
    └── ArticleImpl.ts
```

### 도메인 패키지 추가

어떤 구조로 도메인 패키지를 구성할지는 정해졌으니 `rush.json`에 `domain/board-domain` 디렉토리를 `board-domain`이라는 패키지 이름으로
추가합니다.

```json-doc
{
  ...,
  "projects": [
    {
      "packageName": "app-board-cli",
      "projectFolder": "app/board-cli"
    },
    {
      "packageName": "board-domain",
      "projectFolder": "domain/board-domain"
    }
  ]
}
```

`domain/board-domain` 디렉토리를 생성합니다.

```
~/nodejs-tutorial-example-rush$ mkdir -p domain/board-domain && cd domain/board-domain
```

`npm init`을 입력해 `domain/board-domain/package.json` 파일을 생성합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ npm init
```

`entry-point`(`main` 속성)만 `dist/index.js`로 지정합니다.

```json-doc
// domain/board-domain/package.json
{
  "name": "board-domain",
  "version": "1.0.0",
  "description": "",
  "main": "dist/index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC"
}
```

`rush.json`에 프로젝트를 추가하고 `domain/board-domain/package.json`을 추가했으니 `rush update`를 실행합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush update
```

`typescript`, `eslint`, `prettier`, `jest` 설정은 공통으로 사용하는 방법이 있을 것 같기도 하지만, 일단 `app/board-cli` 디렉토리에서 그대로
복사해오고 `rush add`로 `devDependencies` 의존성도 설치합니다. (개발 의존성을 여러 패키지에서 공통으로 사용하는 방법이 있는지 찾아봤는데 없다고
합니다: [https://stackoverflow.com/a/69803173/14659782](https://stackoverflow.com/a/69803173/14659782) 수동으로 개발 의존성을
모듈마다 설치하면 해당 패키지의 `package.json` 만 보고 의존성을 명확하게 파악할 수 있다는 장점이 있습니다.)

```
~/nodejs-tutorial-example-rush/domain/board-domain$ cd ../../app/board-cli
~/nodejs-tutorial-example-rush/app/board-cli$ cp .eslintignore .eslintrc.js .prettierignore .prettierrc.json jest.config.js tsconfig.json ../../domain/board-domain
~/nodejs-tutorial-example-rush/app/board-cli$ cd ../../domain/board-domain
~/nodejs-tutorial-example-rush/domain/board-domain$ rush add -p @johanblumenberg/ts-mockito -p @types/jest -p @typescript-eslint/eslint-plugin -p @typescript-eslint/parser -p eslint -p eslint-config-prettier -p jest -p prettier -p ts-jest -p ts-node -p typescript --dev --caret
```

`domain/board-domain/package.json`의 `devDependencies`에 아래와 같이 의존성이 추가되었습니다.

```json-doc
{
  ...,
  "devDependencies": {
    "@johanblumenberg/ts-mockito": "^1.0.32",
    "@types/jest": "^27.4.1",
    "@typescript-eslint/eslint-plugin": "^5.17.0",
    "@typescript-eslint/parser": "^5.17.0",
    "eslint": "^8.12.0",
    "eslint-config-prettier": "^8.5.0",
    "jest": "^27.5.1",
    "prettier": "^2.6.1",
    "ts-jest": "^27.1.4",
    "ts-node": "^10.7.0",
    "typescript": "^4.6.3"
  }
}
```

프로젝트가 특정 의존성의 과거 버전을 사용해야만 한다거나, 여러 프로젝트가 사용하는 의존성의 버전이 프로젝트마다 다른 경우 예상치 못한 문제가 발생할 수 있기
때문에 Rush에서는 [`common/config/rush/common-versions.json`](https://rushjs.io/pages/configs/common-versions_json/)을
제공해서 의존성 버전을 통일할 수 있게 도와줍니다. `dependencies`에서 사용하는 의존성은 `range` 없이 특정한 버전을, `devDependencies`에서 사용하는
의존성은 자동으로 `MINOR`패치까지 업데이트 하는 caret(`^`)을 사용하도록 지정합니다.

```json-doc
// common/config/rush/common-versions.json
{
  ...,
  "preferredVersions": {
    // dependencies
    "@reduxjs/toolkit": "1.8.0",
    "inversify": "6.0.1",
    "mobx": "6.4.2",
    "node": "17.5.0",
    "reflect-metadata": "0.1.13",

    // devDependencies
    "@johanblumenberg/ts-mockito": "^1.0.32",
    "@types/jest": "^27.4.1",
    "@typescript-eslint/eslint-plugin": "^5.14.0",
    "@typescript-eslint/parser": "^5.14.0",
    "dependency-cruiser": "^11.4.1",
    "eslint": "^8.11.0",
    "eslint-config-prettier": "^8.5.0",
    "jest": "^27.5.1",
    "nodemon": "^2.0.15",
    "prettier": "2.5.1",
    "ts-jest": "^27.1.3",
    "ts-node": "^10.7.0",
    "typescript": "^4.6.2"
  }
}
```

지금보니 `app/board-cli/package.json`에서 `dependencies`에도 caret을 사용하고 있었네요. 동일하게 `dependencies`에서만 caret을 제거합니다.

저는 `dependencies`에서는 caret(`^`)이나 tilde(`~`)를 사용하지 않고 특정한 버전을 지정해서 사용하는 편이 좋다고 생각합니다. 관련된 더 깊은 이야기는
[Should you Pin your JavaScript Dependencies?](https://docs.renovatebot.com/dependency-pinning/)를 읽어보시길 추천드립니다.

`common-versions.json`을 변경했으니 `rush update --full`를 실행합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush update --full
```

`src/index.ts` 파일에 `foo` 함수를 추가해서 로그를 출력하도록 합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ mkdir src
~/nodejs-tutorial-example-rush/domain/board-domain$ echo 'export const foo = () => console.log("Hello, board-domain!")' > src/index.ts
```

`domain/board-domain/package.json`의 `scripts` 속성에 아래와 같이 커맨드를 추가합니다.

```json-doc
{
  ...,
  "scripts": {
    "build": "npm run prettier && npm run lint && tsc",
    "test": "jest",
    "report": "jest unit --collect-coverage && open coverage/lcov-report/index.html",
    "lint": "eslint . --ext .ts,.tsx",
    "prettier": "prettier --write .",
    "clean": "rm -rf dist"
  }
}
```

`rush build`를 입력해서 빌드가 잘 되는지 확인합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush build
==[ board-domain ]=================================================[ 1 of 2 ]==
"board-domain" completed successfully in 4.86 seconds.

==[ app-board-cli ]================================================[ 2 of 2 ]==
"app-board-cli" completed successfully in 6.46 seconds.



==[ SUCCESS: 2 operations ]====================================================

These operations completed successfully:
  app-board-cli    6.46 seconds
  board-domain     4.86 seconds


rush build (6.59 seconds)
```

두 개의 모듈이 모두 빌드에 성공했습니다. 이제 `app-board-cli`에서 `board-domain`을 사용할 수 있도록 `app/board-cli/package.json`의
`dependencies`에 `"board-domain": "1.0.0"`을 추가합니다.

```json-doc
// app/board-cli/package.json
{
  ...,
  "dependencies": {
    "@reduxjs/toolkit": "1.8.0",
    "inversify": "6.0.1",
    "mobx": "6.4.2",
    "node": "17.5.0",
    "reflect-metadata": "0.1.13",
    "board-domain": "1.0.0"
  }
}
```

패키지간 의존성을 추가했으므로 `app/board-cli/node_modules`를 업데이트하기 위해 `rush update --full`을 합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush update --full
...
Linking local projects

LINKING: app-board-cli
Purging /Users/mj/projects/nodejs-tutorial-example-rush/app/board-cli/node_modules

LINKING: board-domain
Purging /Users/mj/projects/nodejs-tutorial-example-rush/domain/board-domain/node_modules

Linking finished successfully. (2.05 seconds)

Next you should probably run "rush build" or "rush rebuild"


Rush update finished successfully. (9.28 seconds)
```

`app/board-cli/node_modules/board-domain` 심볼릭 링크가 생성된 것을 확인할 수 있습니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ ls -al ../../app/board-cli/node_modules | grep board-domain
lrwxr-xr-x 28 mj  2 Apr 20:25 board-domain -> ../../../domain/board-domain
```

`rush build`를 합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush build
```

`app/board-cli/src/index.ts`에서 `board-domain`의 `foo`함수를 가져올 수 있는지 확인해봅니다.

<figure>
    <img alt="No exported types" src="/res/12-no-exported-types.png" />
    <figcaption>타입이 없어요!</figcaption>
</figure>

타입이 없다고 나오는군요. `domain/board-domain/dist`를 확인해보면 `index.js`만 있고 타입 관련 파일은 없습니다. 이전까지는 컴파일한 결과물이
자바스크립트로서 실행만 되면 충분했기 때문에 타입스크립트 컴파일러가 타입 정보는 출력하지 않도록 하고 있었습니다. 하지만 `board-domain`패키지는
`board-cli` 패키지에서 `import`해서 사용하기 때문에 컴파일 할 때 타입정보까지 출력하도록 변경해야 합니다.

`domain/board-domain/tsconfig.json`에서 `declaration`과 `declarationMap` 속성을 `true`로 변경합니다.

```json-doc
{
  ...,
  "declaration": true,
  "declarationMap": true,
  ...
}
```

다시 빌드합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush build
```

`dist` 디렉토리에 `.d.ts`와 `.d.ts.map` 파일도 생성됩니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ tree dist
dist
├── index.d.ts
├── index.d.ts.map
└── index.js
```

`index.d.ts`에는 `foo()`함수에 대한 타입이 선언되어 있습니다. `index.d.ts.map`은 `index.ts`에서 `index.js`로 컴파일할 때 타입스크립트 소스가
자바스크립트의 어느 부분으로 변환되었는지를 알려줍니다. 실행만을 위해서는 필요 없는 파일이지만 디버깅할 때 유용하게 쓸 수 있으므로 같이 포함해서 내보냅니다.

`app/board-cli/src/index.ts`를 열었던 소스 코드 편집기(제 경우 VSCode)를 재시작하면 `board-domain` 패키지의 타입 관련 에러가 사라진 것을
확인할 수 있습니다.

<figure>
    <img alt="Type found" src="/res/13-type-fixed.png" />
    <figcaption>타입이 생겼어요!</figcaption>
</figure>

`index.d.ts.map` 덕분에 `foo()` 함수의 선언을 VSCode에서 추적하면 `index.d.ts`가 아니라 원본 소스인 `index.ts`로 추적을 해줍니다.

<figure>
    <img alt="Jumping to function declaration" src="/res/14-go-to-definition.png" />
    <figcaption>F12를 누르면 원본 소스로 이동합니다.</figcaption>
</figure>

`applicationByStateManager.run()` 대신 `foo()`를 호출하면 `Hello, board-domain!` 문자열이 출력이 되는 것을 확인할 수 있습니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ cd ../../app/board-cli 
~/nodejs-tutorial-example-rush/app/board-cli$ rush build && rushx start
```

<figure>
    <img alt="Hello, board-domain!" src="/res/15-log-printed.png" />
    <figcaption><code>board-cli</code>패키지에서 <code>board-domain</code>패키지의 코드를 사용할 수 있습니다.</figcaption>
</figure>

`domain/board-domain/package.json` 패키지를 보면 `devDependencies` 속성만 있고 `dependencies`는 없습니다. 도메인 패키지에서는 최대한
보수적으로 외부 의존성을 추가해야 합니다. 아직은 `console.log`밖에 없지만, `board-cli`의 도메인 관련 코드를 옮겨올 때도 외부 의존성이 없을 수
있을지... 만들어봅시다.

일단 그 전에 `.gitignore`파일을 수정하고 커밋해야 합니다. `app/board-cli/.gitignore`의 내용을 프로젝트 root의 `.gitignore`에 추가하고
`app/board-cli/.gitignore`를 제거한 뒤에 커밋합니다. 저는 아래와 같이 `.gitignore`에 내용을 추가했습니다.

```
#Custom
*.js
dist

!jest.config.js
!.eslintrc.js

dependencies.svg
```

지금까지 작성한 코드는 [nodejs-tutorial-example:chapter-5-add-domain-package](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-5-add-domain-package)에서 확인할 수 있습니다.

### `rush clean` 커맨드 추가

이전까지는 소스코드가 하나의 패키지 안에 있었고 빌드를 하면 `dist` 디렉토리 안에 빌드 결과물이 생성되었습니다. 그러나 패키지가 두 개로 나뉜 순간부터는
빌드 결과물이 두 군데로 흩어지기 때문에 이전의 빌드 결과물을 완전히 제거하려면 `app/board-cli/dist`와 `domain/board-domain/dist` 두 군데를
모두 제거해야 합니다.

[Rush는 사용자가 Custom commands를 추가할 수 있도록 허용](https://rushjs.io/pages/maintainer/custom_commands/)하고 있습니다.
빌드 결과물의 제거를 위해 `package.json`에 `clean` 스크립트를 정의해놨으니 `rush clean` 커맨드를 추가해서 두 패키지의 빌드 결과물을 한 번에
제거하겠습니다.

`common/config/rush/command-line.json`의 `commands` 속성에 아래와 같이 추가합니다.

```json-doc
{
  ...,
  "commands": [
    {
      "commandKind": "bulk",
      "name": "clean",
      "summary": "Delete build results of each project.",
      "description": "Delete build results of each project by removing dist directory",
      "enableParallelism": true
    }
  ]
}
```

`commandKind`는 `bulk`와 `global`이 있습니다. `bulk`는 커맨드를 모든 프로젝트마다 한 번씩 실행하고, `global`은 저장소 전체에 대해서 한 번만
커맨드를 실행합니다. `rush clean`을 하면 `dist` 디렉토리 이하의 모든 파일과 디렉토리를 제거합니다.

```
~/nodejs-tutorial-example$ rush clean

Starting "rush clean"

Executing a maximum of 16 simultaneous processes...

==[ board-domain ]=================================================[ 1 of 2 ]==
"board-domain" completed successfully in 0.13 seconds.

==[ app-board-cli ]================================================[ 2 of 2 ]==
"app-board-cli" completed successfully in 0.06 seconds.



==[ SUCCESS: 2 operations ]====================================================

These operations completed successfully:
  app-board-cli    0.06 seconds
  board-domain     0.13 seconds


rush clean (0.22 seconds)

~/nodejs-tutorial-example$ ls app/board-cli/dist       # 아무것도 출력하지 않는다.
~/nodejs-tutorial-example$ ls domain/board-domain/dist # 아무것도 출력하지 않는다.
```

### `board-cli`에서 도메인 분리하기

[이전에 설계했던 대로](/pages/005-separating-into-domain-and-application-with-rush/#도메인-패키지-디렉토리-설계) `board-cli`에 있는
도메인 영역을 `board-domain`패키지로 가져옵니다. 음.. 어떻게 하면 좋을까요? 파일을 이동하면 IDE에서 `import`를 자동으로 변경하면서 뭔가 꼬일 것
같으니 `application`과 `domain` 디렉토리를 복사해서 `board-domain` 패키지에 붙여넣고 설계한 대로 디렉토리를 변경하겠습니다: [https://github.com/myeongjae-kim/nodejs-tutorial-example/commit/6e9642e87566dc9637f2cbaf6340f04bbbfcad35](https://github.com/myeongjae-kim/nodejs-tutorial-example/commit/6e9642e87566dc9637f2cbaf6340f04bbbfcad35)

기존에 작성했던 테스트들은 `board-domain`에서도 통과합니다. 도메인 영역을 외부 의존성 없이 순수한 타입스크립트로만 작성했기 떄문에 `board-domain`에
의존성을 추가하지 않아도 됩니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rushx test
Test Suites: 5 passed, 5 total
Tests:       7 passed, 7 total
Snapshots:   0 total
Time:        3.41 s, estimated 4 s
Ran all test suites.
```

`rush build`를 실행해서 `domain/board-domain/dist` 디렉토리를 업데이트한 뒤에 `app/board-cli/src/application`,
`app/board-cli/src/domain` 디렉토를 과감하게 제거하고 오류가 발생하는 `import`에 대해서 `board-domain`을 의존하도록 변경합니다.
`board-domain/src/index.ts`의 `foo()` 함수는 더 이상 필요 없으니 제거합니다: [https://github.com/myeongjae-kim/nodejs-tutorial-example/commit/145a603e565294b357834a083bd711c5610ec554](https://github.com/myeongjae-kim/nodejs-tutorial-example/commit/145a603e565294b357834a083bd711c5610ec554)

`rush clean`을 해서 기존 빌드를 지우고 `rush build`가 잘 되는지 확인합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush build

src/article/view/cli/MenuPrinter.ts(2,33): error TS2307: Cannot find module 'board-domain/dist/article/port/incoming/ArticleResponse' or its corresponding type declarations.

Operations failed.

rush build (5.36 seconds)
```

안되네요... `app/board-cli`에서 `domain/board-domain`의 빌드 결과물을 찾지 못합니다. 도메인 패키지의 빌드 결과를 확인해보니 소스 코드와 디렉토리
구조가 약간 다릅니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ tree . -I node_modules -I __test__ -L 3 -d
.
├── dist
│   ├── model
│   └── port
│       ├── incoming
│       └── outgoing
└── src
    └── article
        ├── model
        └── port

12 directories
```

`src/article`의 내용물이 `dist/article`이 아니라 그냥 `dist`로 올라와버렸네요. `src` 디렉토리가 비어있기 때문에 이런 현상이 발생합니다.
`tsconfig.json`에서 `rootDir`를 `./src`로 지정해주면 `src`의 구조 그대로 `dist`에 컴파일 결과물을 생성합니다.

```json-doc
// doamin/board-domain/tsconfig.json
{
  "compilerOptions": {
    ...,
    "rootDir": "./src",
    ...
  }
}
```

빌드를 합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush build

==[ board-domain ]=================================================[ 1 of 2 ]==
"board-domain" completed successfully in 4.39 seconds.

==[ app-board-cli ]================================================[ 2 of 2 ]==
"app-board-cli" completed successfully in 5.91 seconds.



==[ SUCCESS: 2 operations ]====================================================

These operations completed successfully:
  app-board-cli    5.91 seconds
  board-domain     4.39 seconds


rush build (10.34 seconds)
```

빌드가 잘 되네요. 트리를 출력해보면 확인해보면 `src`와 `dist`의 구조가 동일합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ tree . -I node_modules -I __test__ -L 3 -d
.
├── dist
│   └── article
│       ├── model
│       └── port
└── src
    └── article
        ├── model
        └── port

8 directories
```

`app/board-cli` 디렉토리로 이동해서 애플리케이션 실행이 잘 되는지 `rushx start`로 확인합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ cd ../../app/board-cli
~/nodejs-tutorial-example-rush/app/board-cli$ rushx start
1) 목록 조회
2) 쓰기
x) 종료

선택: 
```

실행도 잘 됩니다.

지금까지 작성한 코드는 [nodejs-tutorial-example:chapter-5-use-domain-package](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-5-use-domain-package)에서 확인할 수 있습니다.

### 배포용 `tar` ball 만들기

로컬에서 빌드하고 실행까지 잘 되는건 확인했습니다. 배포용 압축 파일은 어떻게 만들 수 있을까요? `node_modules`의 디렉토리가 심볼릭 링크로 연결되어 있기
때문에 `app/board-domain/node_modules`가 압축 파일에 포함되면 `app/board-domain/node_modules/board-domain`도 포함되고, 다시
`app/board-domain/node_modules/board-domain/node_modules`도 포함되고... 압축파일의 크기가 무진장 늘어나게 됩니다. 그렇다고 심볼릭 링크만
압축해버리면 원본을 찾을수가 없으니, 원본을 포함해서 압축하려면 프로젝트 전체를 압축해야 합니다. 세상에.. 아무도 그걸 원하진 않을거에요. 그리고 개발용
의존성을 제거하고 실행용 의존성만 `node_modules`에 포함해서 배포해야 할 코드의 양도 줄여야 합니다.

이 모든것을 위해서 [Rush는 `rush deploy`라는 커맨드를 준비](https://rushjs.io/pages/maintainer/deploying/)해놨습니다. Rush가 Lerna에
비해서 편한 부분이 많지만 정말 배포용 빌드를 만드는 과정은 Lerna에 비해서 훨씬 편합니다.

먼저 `rush init-deploy`로 배포 설정을 추가합니다.

```
~/nodejs-tutorial-example-rush$ rush init-deploy --project app-board-cli

Starting "rush init-deploy"

Creating scenario file: .../nodejs-tutorial-example-rush/common/config/rush/deploy.json

File successfully written. Please review the file contents before committing.
```

`common/config/rush/deploy.json`파일이 생성되었습니다. 이 파일을 따로 수정해주진 않아도 되고, 배포용 빌드에 `app/board-cli/dist/index.js`
뿐만 아니라 `app/board-cli/dist` 디렉토리 전체가 포함되도록 `app/board-cli/package.json`에 `files` 속성을 추가합니다. 

```json-doc
// app/board-cli/package.json
{
  ...,
  "files": [
    "dist"
  ],
  ...
}
```

`rush deploy`를 입력하면 `common/config/rush/deploy.json`을 바탕으로 배포용 빌드를 `common/deploy/app/board-cli`에 생성합니다.

```
~/nodejs-tutorial-example-rush$ rush deploy

Starting "rush deploy"

Loading deployment scenario: /Users/mj/projects/nodejs-tutorial-example-rush/common/config/rush/deploy.json
Deploying to target folder:  /Users/mj/projects/nodejs-tutorial-example-rush/common/deploy
Main project for deployment: app-board-cli

Analyzing project: app-board-cli

Copying folders...
Writing deploy-metadata.json
Creating symlinks...

The operation completed successfully.

~/nodejs-tutorial-example-rush$ node common/deploy/app/board-cli/dist/index.js
1) 목록 조회
2) 쓰기
x) 종료

선택: 
```

실행이 잘 됩니다. `node_modules`를 확인해보면 `devDependencies`의 개발 의존성 없이 `dependencies`의 의존성만 포함되어 있음을 확인할 수 있습니다.

```
common/deploy
├── app
│   └── board-cli
│       ├── dist
│       └── node_modules
│           ├── @reduxjs
│           ├── board-domain -> ../../../domain/board-domain
│           ├── inversify -> ../../../common/temp/node_modules/inversify
│           ├── mobx -> ../../../common/temp/node_modules/mobx
│           └── reflect-metadata -> ../../../common/temp/node_modules/reflect-metadata
├── common
│   └── temp
│       └── node_modules
│           ├── @babel
│           ├── @reduxjs
│           ├── immer
│           ├── inversify
│           ├── mobx
│           ├── redux
│           ├── redux-thunk
│           ├── reflect-metadata
│           ├── regenerator-runtime
│           └── reselect
└── domain
    └── board-domain
        ├── dist
        └── src
```

`board-domain`은 `dependencies`가 없어서 `node_modules` 디렉토리 자체가 없군요. `board-cli/node_modules`를 보면 심볼릭 링크로
`common/temp/node_modules`이하의 디렉토리와 `domain/board-domain` 디렉토리를 참조하는 것을 확인할 수 있습니다. 결국 `common/deploy`
디렉토리는 그 자체로 완결성을 가지므로 빌드 결과물과 호환되는 버전의 node가 설치되어 있는 환경이라면 어디서든지 `app/board-cli/dist/index.js`를
실행해서 우리의 게시판 애플리케이션을 실행할 수 있게 됩니다.

```
~/nodejs-tutorial-example-rush$ tar -czvf bundle.tar common/deploy
~/nodejs-tutorial-example-rush$ mv bundle.tar ~/Downloads && cd ~/Downloads
~/Downloads$ tar -xzvf bundle.tar
~/Downloads$ node common/deploy/app/board-cli/dist/index.js # 이하 4개의 커맨드는 모두 동일한 의미를 가진다.
~/Downloads$ node common/deploy/app/board-cli/dist/index    # index.js 파일을 실행한다.
~/Downloads$ node common/deploy/app/board-cli/dist          # dist 디렉토리의 index.js 파일을 실행한다.
~/Downloads$ node common/deploy/app/board-cli               # package.json의 `main` 속성 값인 dist/index.js 파일을 실행한다.
1) 목록 조회
2) 쓰기
x) 종료

선택: 
```

빌드 결과물을 압축해서 `~/Downlaods` 디렉토리에 배포하고 실행해봤습니다.

모노레포에서 복수의 애플리케이션을 관리해야 한다면, 예를들어 게시판 애플리케이션을 `app/board-react` 디렉토리에 리액트로 구현했다면
[기존의 `deploy.json`의 `"deploymentProjectNames"`에 프로젝트 이름을 추가](https://rushjs.io/pages/maintainer/deploying/#multiple-deployments-using-the-same-config-file)할 수도 있고,
[`deploy-app-board-api.json`같이 새로운 배포 설정용 json을 추가](https://rushjs.io/pages/maintainer/deploying/#multiple-deployments-using-different-config-files)해서 빌드를 할 수 도 있습니다.

### 현실세계의 모노레포

현실에선 지금보다 더 많은 패키지가 필요합니다. 프로젝트 전체에서 공통으로 쓰이는 `core` 패키지, 데이터베이스나 메시지 큐등 외부 의존성에 대한 설정이
담겨 있는 `configruation/xxx` 패키지, 혹은 원격 API를 호출하기 위한 `client/xxx-client` 등 필요에 따라 추가로 패키지를 구성하면 됩니다. 

노드 진영에서는 하나의 저장소 안에 여러 개의 프로젝트가 있는 구조를 모노레포라 부르지만 JVM 진영에서는 멀티모듈이라고 부릅니다. 권용근님의 글
[멀티모듈 설계 이야기 with Spring, Gradle](https://techblog.woowahan.com/2637/)은 제가 실무에서 프로젝트를 구성하면서 모듈을 나누는 기준을
세울 때 정말 많은 도움을 받았던 글입니다.

클래스든 패키지든 모듈이든 아키텍처의 목표는 동일합니다: **높은 응집도와 낮은 결합도를 가지도록 코드를 나누고 비즈니스 로직이 세부사항에 오염되지 않도록 보호하기.**
아키텍처가 목표를 제대로 달성한다면 우리는 [**비즈니스 요구사항에 빠르게 대응할 수 있는 유연한 프로그램**](/#결국엔-비즈니스)이라는 이상에 한 걸음 다가갈 수 있게 됩니다.

지금까지 작성한 코드는 [nodejs-tutorial-example:chapter-5-deploy](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-5-deploy)에서 확인할 수 있습니다.

## PNPM 사용하기

다행히 npm을 사용해도 우리에게 필요한 Rush의 기능은 정상작동을 합니다. 하지만 [속도도 빠르고 디스크 용량도 적게 차지하고 Rush에서 권장하는 pnpm](#pnpm)을 써보면
어떨까요? [https://pnpm.io/installation](https://pnpm.io/installation)을 참고해서 pnpm을 설치합니다. 저는 macOS를 사용하고 있으므로
`brew install pnpm`으로 설치하겠습니다.

```
~/nodejs-tutorial-example-rush$ brew install pnpm
...
~/nodejs-tutorial-example-rush$ pnpm --version
6.32.4
```

프로젝트 root에 있는 `rush.json`에서 `"npmVersion": "6.14.16"` 대신 `"pnpmVersion": "6.32.4"`를 사용하도록 변경합니다.

```json-doc
// rush.json
{
  ...,
  "pnpmVersion": "6.32.4",
  // "npmVersion": "6.14.16",
  ...
}
```

[`rush update --full --purge`를 입력](https://rushjs.io/pages/maintainer/package_managers/#specifying-your-package-manager)합니다.

```
~/nodejs-tutorial-example-rush$ rush update --full --purge

ERROR: An unrecognized file "npm-shrinkwrap.json" was found in the Rush config folder:
/Users/mj/projects/nodejs-tutorial-example-rush/common/config/rush
```

음.. `npm-shrinkwrap.json` 때문에 실패하는군요. 이 파일은 뭘까요?

> What is this "shrinkwrap file"?
> 
> Most projects don't specify an exact version such as 1.2.3 for a dependency, but instead specify SemVer range such as 1.x or ^1.2.3. By itself, this would mean that what gets installed depends on the latest version at the time. Such nondeterminism is bad: It would be maddening for a Git branch that built on Monday to mysteriously be failing on Tuesday because of a new release of a library. The shrinkwrap file solves this problem by storing a complete installation plan in a large file that is tracked by Git.
> 
> The shrinkwrap file has different names depending on the package manager that your repo is using: shrinkwrap.yaml, npm-shrinkwrap.json, or yarn.lock
> 
> \- [Everyday commands \| Rush](https://rushjs.io/pages/developer/everyday_commands/)


의존성 버전에 caret(`^`)이나 tilde(`~`)등을 사용하면 동일한 버전이라도 언제 `npm install`을 하는가에 따라서 다른 버전이 설치될 수 있기 때문에
앱이 제대로 작동하지 않을 수 있습니다. 이 문제를 해결하기 위해 npm은 `package-lock.json`을 추가해서 `package-lock.json`이 변경되지 않는 이상
같은 버전이 설치되도록 제한합니다. Rush에서는 `npm-shrinkwrap.json`이 동일한 역할을 합니다. pnpm을 사용하면 `pnpm-lock.yaml`을 사용합니다.
`rush.json`은 pnpm을 사용하도록 변경했지만 `npm-shrinkwrap.json`파일이 이미 있으니까 `rush update --full --purge`가 실패했습니다.

저희는 [도메인 패키지를 추가](#도메인-패키지-추가)하면서 `dependencies`의 버전이 여러 개로 해석될 수 없게끔 변경했으니 마음놓고
`npm-shrinkwrap.json`을 삭제해도 됩니다.

```
~/nodejs-tutorial-example-rush$ rm common/config/rush/npm-shrinkwrap.json
~/nodejs-tutorial-example-rush$ rush update --full --purge

The command failed:
 /Users/mj/projects/nodejs-tutorial-example-rush/common/temp/pnpm-local/node_modules/.bin/pnpm install --store /Users/mj/projects/nodejs-tutorial-example-rush/common/temp/pnpm-store --no-prefer-frozen-lockfile --recursive --link-workspace-packages falseERROR: Error: The command failed with exit code 1

Giving up after 3 attempts
```

그래도 저는 뭐가 잘 안되네요... 기존에 설치되어있던 node_modules 디렉토리 때문에 충돌이 발생하는 것 같으니 찾아서 모두 제거하겠습니다.

```
~/nodejs-tutorial-example-rush$ find . -type d -name "node_modules" -prune
./app/board-cli/node_modules
./common/temp/node_modules
./common/deploy/app/board-cli/node_modules
./common/deploy/common/temp/node_modules
./domain/board-domain/node_modules

# node_modules를 잘 찾는 것을 확인했으니 과감히 rm -rf로 지워준다.
~/nodejs-tutorial-example-rush$ find . -type d -name "node_modules" -prune | xargs rm -rf
~/nodejs-tutorial-example-rush$ rush update --full --purge
...
 WARN  Issues with peer dependencies found
../../app/board-cli
└─┬ ts-node
  └── ✕ missing peer @types/node@"*"
Peer dependencies that should be installed:
  @types/node@"*"  

../../domain/board-domain
└─┬ ts-node
  └── ✕ missing peer @types/node@"*"

Peer dependencies that should be installed:
  @types/node@"*"  

Copying "/Users/mj/projects/nodejs-tutorial-example-rush/common/temp/pnpm-lock.yaml"
  --> "/Users/mj/projects/nodejs-tutorial-example-rush/common/config/rush/pnpm-lock.yaml"


Rush update finished successfully. (27.10 seconds)
```

드디어 `rush update`가 성공을 했습니다. 하지만 Peer dependency 관련 경고가 발생했습니다. `ts-node`가 `@types/node`를 사용하지만 저희가
명시적으로 의존성을 추가하지 않았기 때문에 발생하는 경고입니다. [Rush는 pnpm을 사용하는 경우 `rush.json`에 `"strictPeerDependencies"` 옵션을 켜도록 권장](https://rushjs.io/pages/maintainer/recommended_settings/#strictpeerdependencies)하고 있습니다.
옵션을 켜고 다시 `rush update --full`을 해보겠습니다.

```json-doc
// rush.json
{
  ...,
  "pnpmOptions": {
    "strictPeerDependencies": true
  }
}
```

```
~/nodejs-tutorial-example-rush$ rush update --full

...
 ERR_PNPM_PEER_DEP_ISSUES  Unmet peer dependencies

../../app/board-cli
└─┬ ts-node
  └── ✕ missing peer @types/node@"*"
Peer dependencies that should be installed:
  @types/node@"*"
  
../../domain/board-domain
└─┬ ts-node
  └── ✕ missing peer @types/node@"*"
Peer dependencies that should be installed:
  @types/node@"*"
```

이번에는 경고가 아니라 에러가 발생하면서 `rush update --full`이 실패합니다. 의존성 관리를 좀 더 빡빡하게 할 수 있어서 아주 만족스럽습니다.
`app/board-cli`와 `domain/board-domain`의 `devDependencies`에 `@types/node`를 추가한 뒤 `rush update --full`을 입력합니다. 
`npm show @types/node version`를 입력하면 `@types/node`의 최신버전을 알 수 있습니다. 저는 `"@types/node": "^17.0.23"`를 추가하겠습니다.


```
~/nodejs-tutorial-example-rush$ rush update --full

...

Scope: all 3 workspace projects
.                                        | +559 ++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Packages are hard linked from the content-addressable store to the virtual store.
  Content-addressable store is at: .../nodejs-tutorial-example-rush/common/temp/pnpm-store/v3
  Virtual store is at:             node_modules/.pnpm
Progress: resolved 559, reused 559, downloaded 0, added 559, done
node_modules/.pnpm/nodemon@2.0.15/node_modules/nodemon: Running postinstall script, done in 179ms

Copying ".../nodejs-tutorial-example-rush/common/temp/pnpm-lock.yaml"
  --> ".../nodejs-tutorial-example-rush/common/config/rush/pnpm-lock.yaml"

Rush update finished successfully. (22.03 seconds)
```

이제 에러 없이 잘 되는군요! `rush build && rush deploy`로 빌드까지 잘 되는지 확인해보겠습니다.

```
~/nodejs-tutorial-example-rush$ rush build && rush deploy

> app-board-cli@1.0.0 lint /Users/mj/projects/nodejs-tutorial-example-rush/app/board-cli
> eslint . --ext .ts,.tsx

src/article/view/cli/state-modules/redux/MyStore.ts(1,42): error TS2307: Cannot find module 'redux' or its corresponding type declarations.


Operations failed.

rush build (10.35 seconds)
```

빌드 과정에서 문제가 발생했습니다. `MyStore.ts`에서 사용하는 `redux` 모듈을 찾을 수 없다고 나옵니다. `app/board-cli/package.json`의 의존성을
보면 `@reduxjs/toolkit`만 있을 뿐 `redux`는 없는데, 지금까지 어떻게 `redux`를 사용하고 있었을까요?

```typescript
import { ActionFromReducer, Store } from "redux"; // pnpm 덕분에 잘못된 import를 발견!
import * as reduxModule from "./redux-module";

export type MyStore = Store<
  reduxModule.State,
  ActionFromReducer<typeof reduxModule.reducer>
>;
```

```json-doc
// app/board-cli/package.json
{
  ...,
  "dependencies": {
    "@reduxjs/toolkit": "1.8.0", // 이 의존성만 있고 redux는 없다.
    "inversify": "6.0.1",
    "mobx": "6.4.2",
    "reflect-metadata": "0.1.13",
    "board-domain": "workspace:*" // pnpm을 사용하도록 변경하고 rush update --full --purge를 하면서 "1.0.0"대신 "workspace:*"로 변경되었다.
  },
  "devDependencies": {
    "@johanblumenberg/ts-mockito": "^1.0.32",
    "@types/jest": "^27.4.1",
    "@types/node": "^17.0.23",
    "@typescript-eslint/eslint-plugin": "^5.14.0",
    "@typescript-eslint/parser": "^5.14.0",
    "dependency-cruiser": "^11.4.1",
    "eslint": "^8.11.0",
    "eslint-config-prettier": "^8.5.0",
    "jest": "^27.5.1",
    "nodemon": "^2.0.15",
    "prettier": "2.5.1",
    "ts-jest": "^27.1.3",
    "ts-node": "^10.7.0",
    "typescript": "^4.6.2"
  }
}
```

[`@reduxjs/toolkit`의 의존성](https://www.npmjs.com/package/@reduxjs/toolkit?activeTab=dependencies)을 확인해보면 `redux`를
찾을 수 있습니다. npm을 사용중이었다면 의존성(`@reduxjs/toolkit`)의 의존성(`redux`)까지 `node_modules` 디렉토리에 평평하게 펼쳐지기 때문에
이제까지는 직접 `redux`를 의존하지 않더라도 `redux` 패키지를 사용할 수 있었습니다. 이것이 [Phantom Dependencies](https://rushjs.io/pages/advanced/phantom_deps/)
문제 입니다.

pnpm을 사용하게 되면 `app/board-cli/node_modules`에는 `redux` 없이 `@reduxjs/toolkit`만 설치가 됩니다.

```
~/nodejs-tutorial-example-rush$ tree app/board-cli/node_modules -d
app/board-cli/node_modules
├── @johanblumenberg
│   └── ts-mockito -> ../../../../common/temp/node_modules/.pnpm/...
├── @reduxjs
│   └── toolkit -> ../../../../common/temp/node_modules/.pnpm/...
├── @types
│   ├── jest -> ../../../../common/temp/node_modules/.pnpm/...
│   └── node -> ../../../../common/temp/node_modules/.pnpm/...
├── @typescript-eslint
│   ├── eslint-plugin -> ../../../../common/temp/node_modules/.pnpm/...
│   └── parser -> ../../../../common/temp/node_modules/.pnpm/...
├── board-domain -> ../../../domain/board-domain
├── dependency-cruiser -> ../../../common/temp/node_modules/.pnpm/...
├── eslint -> ../../../common/temp/node_modules/.pnpm/...
├── eslint-config-prettier -> ../../../common/temp/node_modules/.pnpm/...
├── inversify -> ../../../common/temp/node_modules/.pnpm/...
├── jest -> ../../../common/temp/node_modules/.pnpm/...
├── mobx -> ../../../common/temp/node_modules/.pnpm/...
├── nodemon -> ../../../common/temp/node_modules/.pnpm/...
├── prettier -> ../../../common/temp/node_modules/.pnpm/...
├── reflect-metadata -> ../../../common/temp/node_modules/.pnpm/...
├── ts-jest -> ../../../common/temp/node_modules/.pnpm/...
├── ts-node -> ../../../common/temp/node_modules/.pnpm/...
└── typescript -> ../../../common/temp/node_modules/.pnpm/...

23 directories
```

`node_modules`에 23개의 디렉토리밖에 없다니... [이전에 확인했을 때는 435개의 디렉토리 있었는데](#pnpm) 참 대조적이네요. 디렉토리들도 모두 심볼릭
링크로 연결되어 있기 때문에 모노레포 상황이더라도 동일한 의존성이 여러 `node_modules`에 설치되는 현상을 해결할 수 있게 되었습니다. 디스크 사용량도
줄어들고 의존성을 설치할 때도 한 곳(`common/temp/node_modules/.pnpm`)에만 설치하기 때문에 속도도 빠릅니다. IDE에서도 `import` 문을 자동으로
추가할 때 인덱싱해야 할 `node_modules` 의존성 개수가 확 줄어들기 때문에 개발 환경도 쾌적해집니다.

`MyStore.ts`에서 `redux`대신 `@reduxjs/toolkit`을 사용하도록 변경하고 다시 빌드합니다.

```typescript
import { ActionFromReducer, Store } from "@reduxjs/toolkit"; // redux 대신 @reduxjs/toolkit을 사용
import * as reduxModule from "./redux-module";

export type MyStore = Store<
  reduxModule.State,
  ActionFromReducer<typeof reduxModule.reducer>
>;
```

```
~/nodejs-tutorial-example-rush$ rush build && rush deploy

Starting "rush deploy"

Loading deployment scenario: /Users/mj/projects/nodejs-tutorial-example-rush/common/config/rush/deploy.json
Deploying to target folder:  /Users/mj/projects/nodejs-tutorial-example-rush/common/deploy
Main project for deployment: app-board-cli


ERROR: The deploy target folder is not empty. You can specify "--overwrite" to recursively delete all folder contents.
```

빌드 과정은 성공했지만 배포 디렉토리가 비어있지 않다고 에러가 발생했습니다. `--overwrite` 옵션을 추가해서 배포하고 애플리케이션을 실행해보겠습니다.

```
~/nodejs-tutorial-example-rush$ rush deploy --overwrite

...
The operation completed successfully.
~/nodejs-tutorial-example-rush$ node common/deploy/app/board-cli
1) 목록 조회
2) 쓰기
x) 종료

선택: 
```

실행이 잘 됩니다.

지금까지 작성한 코드는 [nodejs-tutorial-example:chapter-5-use-pnpm](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-5-use-pnpm)에서 확인할 수 있습니다.

## 중복 설정 제거하기 (with Heft)

모노레포를 먼저 적용하기 위해 중복으로 작성했던 설정은 다음과 같습니다.

- `.eslintignore`
- `.eslintrc.js`
- `.prettierignore`
- `.prettierrc.json`
- `.jest.config.js`
- `tsconfig.json`

지금처럼 프로젝트마다 중복으로 설정파일을 작성하면 작동은 가능하지만 영 찜찜한 것은 어쩔 수 없습니다. Rush는 중복 설정의 문제를 해결하기 위해 Heft를
사용합니다.

- [https://rushstack.io/pages/heft_tasks/eslint/](https://rushstack.io/pages/heft_tasks/eslint/)
- [https://rushstack.io/pages/heft_tasks/typescript/](https://rushstack.io/pages/heft_tasks/typescript/)

Heft는 Rush Stack에 포함되어 있습니다. Rush Stack은 또 뭘까요? 이렇게 작은 게시판 프로젝트에서도 중복 설정의 문제가 발생하는걸 보면, 프로젝트가
커질수록 모노레포 상황에서만 발생하게 되는 문제들을 마주칠 것 같은 불안한 기분이 듭니다. Rush Stack은 대규모 모노레포 프로젝트를 관리할 때 마주치게 되는
문제를 해결할 수 있는 도구 모음집니다: [https://rushstack.io/#what-is-rush-stack](https://rushstack.io/#what-is-rush-stack)

Heft에서는 설정 파일의 중복을 ['rig packages'](https://rushstack.io/pages/heft/rig_packages/)로 해결합니다.
일단 [Heft부터 설치](https://rushstack.io/pages/heft_tutorials/getting_started/)해서 사용해볼까요?

```
~/nodejs-tutorial-example-rush$ cd app/board-cli
~/nodejs-tutorial-example-rush/app/board-cli$ rush add -p @rushstack/heft --dev --caret
```

Heft를 명령줄에서도 실행할 수 있게 `--global`로 설치도 해줍니다.

```
~/nodejs-tutorial-example-rush/app/board-cli$ pnpm install --global @rushstack/heft
```

그리고 `app/board-cli`에서 `heft build`로 빌드가 잘 되는지 테스트합니다.

```
~/nodejs-tutorial-example-rush/app/board-cli$ heft build

...
[typescript] Using TypeScript version 4.6.3
[eslint] Using ESLint version 8.12.0
 ---- Compile finished (3452ms) ---- 
 ---- Bundle started ---- 
 ---- Bundle finished (1ms) ---- 
 ---- Post-build started ---- 
 ---- Post-build finished (0ms) ---- 
-------------------- Finished (3.802s) --------------------
Project: app-board-cli@1.0.0
Heft version: 0.44.5
Node version: v14.19.1

~/nodejs-tutorial-example-rush/app/board-cli$ rushx start
1) 목록 조회
2) 쓰기
x) 종료

선택: 
~/nodejs-tutorial-example-rush/app/board-cli$ echo 'temp/' >> ../../.gitignore
```

빌드가 잘 되는군요. 실행도 잘 됩니다. `temp` 디렉토리도 생성되는데 저장소에 추가할 필요는 없으니 `.gitignore`에 `temp/`를 추가합니다.

방금은 `package.json`의 `scripts`의 `build`를 사용하지 않고 빌드를 했습니다. `package.json`에서 `build` 스크립트를 삭제하고 다시
`heft build`를 입력해도 동일하게 빌드가 잘 되는 것을 확인할 수 있습니다. `heft build`를 실행하면 Heft가 `.eslintrc`와 `tsconfig.json`를
보고 알아서 lint를 수행하고 컴파일을 합니다. 우리가 원하는 빌드 과정은

1. `prettier`로 코드 스타일 정리
2. `eslint`로 규칙에 어긋나는 코드가 있는지 확인
3. `typescript`로 컴파일

하는 것입니다. 운이 좋게 2번과 3번은 Heft가 알아서 해주지만, `prettier`는 추가로 설정해서 `heft build` 과정에 집어넣어야 합니다. 그리고 위 3개의
과정과 관련된 모든 설정파일도 여전히 패키지별로 존재하고 있습니다. 일단
[메뉴얼을 따라서 `package.json`의 `build` 스크립트에서 `heft build --clean`을 실행하도록 변경](https://rushstack.io/pages/heft_tutorials/heft_and_rush/#how-heft-gets-invoked)합니다.

```json-doc
// app/board-cli/package.json과 domain/board-domain/package.json 모두 동일하게 변경
{
  ...,
  "scripts": {
    "build": "heft build --clean"
  },
  ...
}
```

그리고 최종 빌드 결과물이 실행이 잘 되는지 확인합니다.

```
~/nodejs-tutorial-example-rush/app/board-cli$ cd ../../
~/nodejs-tutorial-example-rush$ rush build && rush deploy --overwrite

...
Deleting target folder contents because "--overwrite" was specified...                                                  
                                                                                                                        
Analyzing project: app-board-cli                                                                                        
                                                                                                                        
Copying folders...                                                                                                      
Writing deploy-metadata.json                                                                                            
Creating symlinks...                                                                                                    
                                                                                                                        
The operation completed successfully. 
~/nodejs-tutorial-example-rush$ node common/deploy/app/board-cli
1) 목록 조회
2) 쓰기
x) 종료

선택:
```

Heft로 빌드를 수행하더라도 빌드 결과물이 제대로 나옵니다. Rush와 Heft가 잘 맞물려서 작동하는 것도 확인했습니다. 이제는 중복된 설정 파일을 제거할
차례입니다.

### rig package로 `tsconfig.json` 중복 없애기

Heft는 ['rig packages'라는 설정용 패키지를 도입](https://rushstack.io/pages/heft_tutorials/heft_and_rush/#sharing-configuration-using-rig-packages)해서 설정 파일의 중복을 없앱니다.
`tsconfig.json`이나 `.eslintrc`같이 IDE에서 사용해야 하는 설정파일의 경우는 개별 패키지 디렉토리에서 이 파일들을 완전히 없앨 수는 없지만,
`jest.config.json`같은 경우는 'rig packages'를 도입해서 완전히 없앨 수 있습니다.

`core-rig`라는 우리만의 rig package를 만들어서 사용해봅시다. `rig/core-rig`라는 디렉토리를 만들고 `rush.json`의 `projects` 속성에 추가합니다.

```
~/nodejs-tutorial-example-rush$ mkdir -p rig/core-rig && cd rig/core-rig
~/nodejs-tutorial-example-rush/rig/core-rig$ npm init # package.json 생성. 계속 엔터만 눌러줍니다.
```

생성한 `package.json`을 열어서 `build`와 `clean` 스크립트를 추가합니다. 없으면 `rush` 커맨드 실행시 에러가 발생하기 때문에 빈 문자열만 넣어줍니다.

```json-doc
// rig/core-rig/package.json
{
  "name": "core-rig",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "author": "",
  "scripts": {
    "build": "",
    "clean": ""
  },
  "license": "ISC"
}
```

`rush.json`에 프로젝트를 추가합니다.

```json-doc
// rush.json
{
  ...,
  "projects": [
    {
      "packageName": "app-board-cli",
      "projectFolder": "app/board-cli"
    },
    {
      "packageName": "board-domain",
      "projectFolder": "domain/board-domain"
    },
    {
      "packageName": "core-rig",
      "projectFolder": "rig/core-rig"
    }
  ]
}
```

다시 프로젝트 root으로 돌아와서 `rush update`를 합니다.

```
~/nodejs-tutorial-example-rush/rig/core-rig$ cd ../../
~/nodejs-tutorial-example-rush$ rush update

...
Rush update finished successfully. (1.82 seconds)
```

우리는 이전에 `domain/board-domain` 패키지를 추가하면서 `tsconfig.json`에 `declaration`과 `declarationMap` 옵션을 켜주었습니다.
`tsconfig.json` 옵션을 `rig/core-rig` 패키지로 옮기더라도 `domain/board-domain`빌드 결과물에 `.d.ts`와 `.d.ts.map`이 생성된다면
`tsconfig.json` 설정이 잘 적용됐다는 것을 증명할 수 있습니다.

`doamin/board-domain`으로 이동해서 `core-rig` 의존성을 추가합니다.

```
~/nodejs-tutorial-example-rush$ cd domain/board-domain
~/nodejs-tutorial-example-rush/domain/board-domain$ rush add -p core-rig
~/nodejs-tutorial-example-rush/domain/board-domain$ cat package.json
{
  ...,
  "dependencies": {
    "core-rig": "workspace:*"
  }
}
~/nodejs-tutorial-example-rush/domain/board-domain$ ls -al node_modules | grep core-rig
lrwxr-xr-x  21 mj  4 Apr 21:47 core-rig -> ../../../rig/core-rig
```

`core-rig` 의존성이 추가되었고 `rig/core-rig` 디렉토리를 가리키는 것을 확인할 수 있습니다. `domain/board-domain/tsconfig.json`을
`rig/core-rig/tsconfig.json`으로 복사합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ cp tsconfig.json ../../rig/core-rig
```

그리고 `domain/board-domain/tsconfig.json`이 `rig/core-rig`의 `tsconfig.json`을 상속하도록 변경합니다.

```json-doc
// domain/board-domain/tsconfig.json
{
  "extends": "./node_modules/core-rig/tsconfig.json"
}
```

마지막으로 `domain/board-domain/config/rig.json`을 추가합니다. `rigPackageName`은 `core-rig`입니다.

```
// The "rig.json" file directs tools to look for their config files in an external package.
// Documentation for this system: https://www.npmjs.com/package/@rushstack/rig-package
{
  "$schema": "https://developer.microsoft.com/json-schemas/rig-package/rig.schema.json",

  /**
   * (Required) The name of the rig package to inherit from.
   * It should be an NPM package name with the "-rig" suffix.
   */
  "rigPackageName": "core-rig"

  /**
   * (Optional) Selects a config profile from the rig package.  The name must consist of
   * lowercase alphanumeric words separated by hyphens, for example "sample-profile".
   * If omitted, then the "default" profile will be used."
   */
  // "rigProfile": "your-profile-name"
}
```

`dist` 디렉토리를 제거하고 다시 빌드를 합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rm -rf dist && rush build

--[ FAILURE: board-domain ]----------------------------------[ 0.54 seconds ]--

Error: The rig profile "default" is not defined by the rig package "core-rig"


Operations failed.

rush build (0.57 seconds)
```

`default` profile이 `core-rig`에 없다고 에러가 발생합니다.
[rig pakcage의 예시를 보면 tsconfig-base.json이 profiles/default 디렉토리 안에 있는 것을 확인](https://github.com/microsoft/rushstack/tree/master/rigs/heft-node-rig/profiles/default)할 수 있습니다.
저희도 동일하게 `rig/core-rig/tsconfig.json`을 `rig/core-rig/profiles/default` 이하로 옮겨주고 다시 빌드를 합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ cd ../../rig/core-rig
~/nodejs-tutorial-example-rush/rig/core-rig$ mkdir -p profiles/default && mv tsconfig.json profiles/default
~/nodejs-tutorial-example-rush/rig/core-rig$ rush build

...
  [typescript] src/article/view/cli/MenuPrinter.ts:2:33 - (TS2307) Cannot find module 'board-domain/dist/article/port/incoming/ArticleResponse' or its corresponding type declarations.
Error: Encountered TypeScript errors


Operations failed.

rush build (7.94 seconds)
```

이번에도 실패합니다. rig package 설정까지는 잘 된 것 같은데 `app/board-cli`를 컴파일 할 때 `domain/board-domain`의 코드를 찾지 못합니다.
디렉토리를 확인해보니 `domain/board-domain/dist`가 생성되어 있지 않습니다. `domain/board-domain` 패키지의 빌드는 성공했는데 빌드 결과물은
어디에 있는걸까요? 두구두구... `rig/core-rig/profiles/default/dist`에 있습니다.

`rig/core-rig/profiles/default/tsconfig.json`에 `"outDir": "./dist"`로 옵션이 들어가있어서 이를 상속하는
`domain/board-domain/tsconfig.json`의 빌드 결과물이 `rig/core-rig/profiles/default/dist`로 가버린 것이었습니다.`"outDir"`과
`"rootDir"` 속성은 `core-rig`의 `tsconfig.json`에 있을 필요가 없으니 지워주고 `domain/board-domain/tsconfig.json`에 추가해줍니다.

```json-doc
// domain/board-domain/tsconfig.json
{
  "extends": "./node_modules/core-rig/profiles/default/tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
  }
}
```

오랜만에 `rush clean`을 하고 `rush build`를 실행합니다.

```
~/nodejs-tutorial-example-rush/rig/core-rig$ rush clean && rush build

...
==[ SUCCESS: 2 operations ]====================================================

These operations completed successfully:
  app-board-cli    4.27 seconds
  board-domain     3.13 seconds


rush build (7.42 seconds)
```

빌드가 잘 되네요. `domain/board-domain/dist`를 확인해보면 `.d.ts`와 `.d.ts.map` 파일을 확인할 수 있습니다. 이로써 `core-rig`의
`tsconfig.json`의 설정이 `domain/board-domain`의 컴파일에 영향을 주고 있음을 알 수 있습니다.

```
~/nodejs-tutorial-example-rush/rig/core-rig$ tree ../../domain/board-domain/dist -P '*.d.ts|*.d.ts.map' -I __test__
../../domain/board-domain/dist
└── article
    ├── ArticleCommandService.d.ts
    ├── ArticleCommandService.d.ts.map
    ├── ArticleQueryService.d.ts
    ├── ArticleQueryService.d.ts.map
    ├── model
    │   ├── Article.d.ts
    │   ├── Article.d.ts.map
    │   ├── ArticleImpl.d.ts
    │   └── ArticleImpl.d.ts.map
    └── port
        ├── incoming
        │   ├── ArticleCreateUseCase.d.ts
        │   ├── ArticleCreateUseCase.d.ts.map
        │   ├── ArticleGetUseCase.d.ts
        │   ├── ArticleGetUseCase.d.ts.map
        │   ├── ArticleListUseCase.d.ts
        │   ├── ArticleListUseCase.d.ts.map
        │   ├── ArticleRequest.d.ts
        │   ├── ArticleRequest.d.ts.map
        │   ├── ArticleResponse.d.ts
        │   └── ArticleResponse.d.ts.map
        └── outgoing
            ├── ArticleLoadPort.d.ts
            ├── ArticleLoadPort.d.ts.map
            ├── ArticleSavePort.d.ts
            └── ArticleSavePort.d.ts.map

5 directories, 22 files
```

`app/board-cli`에서도 `rig/core-rig`를 의존하고 `tsconfig.json`의 중복을 없앱니다.

```
~/nodejs-tutorial-example-rush/rig/core-rig$ cd ../../app/board-cli
~/nodejs-tutorial-example-rush/app/board-cli$ rush add -p core-rig
```

```
// app/board-cli/tsconfig.json
{
  "extends": "./node_modules/core-rig/profiles/default/tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  }
}
```

다시 빌드하고 실행까지 잘 되는지 확인합니다.

```
~/nodejs-tutorial-example-rush/app/board-cli$ rush clean && rush rebuild && rushx start
1) 목록 조회
2) 쓰기
x) 종료

선택:
```

아래 설정파일들 중에서 `tsconfig.json`의 중복을 해결했습니다.

- [ ] `.eslintignore`
- [ ] `.eslintrc.js`
- [ ] `.prettierignore`
- [ ] `.prettierrc.json`
- [ ] `.jest.config.js`
- [x] `tsconfig.json`

다음은 `.eslintrc.js`와 `.eslitignore` 차례입니다.

지금까지 작성한 코드는 [nodejs-tutorial-example:chapter-5-rig](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-5-rig)에서 확인할 수 있습니다.

### `.eslintrc.js` 중복 없애기

Rush Stack에서는 `.eslintrc.js`에 대해서 아래와 같이 말하고 있습니다.

> 모노레포의 root 폴더에 중앙화된 `.eslintrc.js` 파일을 놓기를 추천하지 않는다. 이는 프로젝트들이 독립적이어야 하고 모노레포간에 프로젝트들을
> 옮기기 쉬워야 한다는 Rush의 원칙과 맞지 않는다.
> 
> It's not recommended to place a centralized .eslintrc.js in the monorepo root folder. This violates Rush's principle
> that projects should be independent and easily movable between monorepos.
> 
> \- ["eslint" \| Rush Stack](https://rushstack.io/pages/heft_tasks/eslint/#config-files)

그리고 이어지는 코드를 보면 `tsconfig.json`과 마찬가지로 rig package에 설정을 보관하고 개별 프로젝트의 `.eslintrc`에서 상속해서 사용합니다.

```javascript
// This is a workaround for https://github.com/eslint/eslint/issues/3458
require('@rushstack/eslint-config/patch/modern-module-resolution');

module.exports = {
  extends: ['@rushstack/eslint-config/profile/node'],
  parserOptions: { tsconfigRootDir: __dirname }
};
```

저희도 동일하게 해보겠습니다. `.eslintrc.js`를 `core-rig` 프로젝트로 옮기고 `app/board-cli`와 `domain/board-domain` 프로젝트에
`@rushstack/eslint-config`를 개발 의존성으로 추가합니다 (위 코드의 `require` 부분을 사용하기 위함).

```
~/nodejs-tutorial-example-rush/app/board-cli$ cp .eslintrc.js ../../rig/core-rig/profiles/default/
~/nodejs-tutorial-example-rush/app/board-cli$ rush add -p @rushstack/eslint-config --dev --caret
~/nodejs-tutorial-example-rush/app/board-cli$ cd ../../domain/board-domain
~/nodejs-tutorial-example-rush/domain/board-domain$ rush add -p @rushstack/eslint-config --dev --caret
```

그리고 `rig/core-rig` 프로젝트에 `eslint` 관련 의존성을 개발 의존성으로 설치합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ cd ../../rig/core-rig
~/nodejs-tutorial-example-rush/rig/core-rig$ rush add -p @typescript-eslint/eslint-plugin -p @typescript-eslint/parser -p typescript -p eslint -p eslint-config-prettier --dev --caret
```

`app/board-cli/.eslintrc.js`와 `domain/board-domain/.eslintrc.js`를 아래처럼 변경합니다.

```javascript
// This is a workaround for https://github.com/eslint/eslint/issues/3458
require('@rushstack/eslint-config/patch/modern-module-resolution');

module.exports = {
  extends: ['core-rig/profiles/default/.eslintrc'],
  parserOptions: { tsconfigRootDir: __dirname }
};
```

`app/board-cli/index.ts`에서 `eslint` 규칙을 위배해보겠습니다.

<figure>
    <img alt="Eslint works well" src="/res/16-eslint.png" />
    <figcaption>VSCode의 eslint 플러그인이 잘 작동합니다.</figcaption>
</figure>

```
~/nodejs-tutorial-example-rush/domain/board-domain$ cd ../../app/board-cli
~/nodejs-tutorial-example-rush/app/board-cli$ rushx lint

...
Rush Multi-Project Build Tool 5.64.0 - Node.js 14.19.1 (LTS)
> "eslint . --ext .ts,.tsx"


/Users/mj/projects/nodejs-tutorial-example-rush/app/board-cli/src/index.ts
  7:7  warning  'applicationByStateManager' is assigned a value but never used  @typescript-eslint/no-unused-vars

✖ 1 problem (0 errors, 1 warning)
```

작동이 잘 됩니다. `.eslintignore`는 root으로 올려줍니다.

```
~/nodejs-tutorial-example-rush/app/board-cli$ mv .eslintignore ../../
~/nodejs-tutorial-example-rush/app/board-cli$ rm ../../domain/board-domain/.eslintignore
```

[Rush Stack 문서의 Riggable Dependencies 항목](https://rushstack.io/pages/heft/rig_packages/#3-riggable-dependencies)을 보면
`typescript`와 `eslint` 의존성은 rig package로 제공할 수 있다고 합니다.

> The rig package can also provide NPM dependencies, to avoid having to specify them as "devDependencies" for your project. The following tool packages can be provided by the rig:
> 
> - typescript
> - @microsoft/api-extractor
> - eslint
> - tslint
> 
> Today, only these packages can be provided via a rig. Providing dependencies via a rig is optional. Your local project's devDependencies take precedence over the rig.
> 
> \- [Using rig packages \| Rush Stack](https://rushstack.io/pages/heft/rig_packages/#3-riggable-dependencies)

`typescript`, `eslint`, `@typescript-eslint/eslint-plugin`, `@typescript-eslint/parser`,  `eslint-config-prettier`
이 5개의 의존성은 `app/board-cli`, `domain/board-domain`, `rig/core-rig` 3개의 패키지에서 모두 의존하고 있습니다. rig package에만
남겨놔도 될 것 같으니 `app/board-cli`, `domain/board-domain`에서 위 의존성들을 제거합니다.

`typescript`, `eslint`는 `rig/core-rig/package.json`에서 `devDependencies`가 아니라 `dependencies`에 있어야 합니다.

> Heft resolves each riggable tool independently, using the following procedure:
> 
> 1. If the tool package is listed in the devDependencies for the local project, then the tool is resolved from the current project folder. (This step does NOT consider dependencies or peerDependencies.)
> 2. OTHERWISE, if the current project has a rig.json file, and if the rig's package.json lists the tool in its dependencies, then the tool is resolved from the rig package folder. (This step does NOT consider devDependencies or peerDependencies.)
> 3. OTHERWISE, the tool is resolved from the current project folder. If it can't be found there, then an error is reported.
> 
> \- [Using rig packages \| Rush Stack](https://rushstack.io/pages/heft/rig_packages/#3-riggable-dependencies)

의존성을 제거하는 [`rush remove`같은 커맨드는 아직 없습니다.](https://github.com/microsoft/rushstack/issues/1457)
`app/board-cli/pakcage.json`, `domain/board-domain/package.json`에서 수동으로 의존성을 제거하고 `rig/coer-rig/package.json`에서
`typescript`와 `eslint`를 `dependencies`로 옮긴 뒤 `rush update`를 입력하면...

```
~/nodejs-tutorial-example-rush/app/board-cli$ rush update
...
 ERR_PNPM_PEER_DEP_ISSUES  Unmet peer dependencies                                                                      
                                                                                                                        
../../app/board-cli
├─┬ ts-jest
│ └── ✕ missing peer typescript@">=3.8 <5.0"
├─┬ ts-node
│ └── ✕ missing peer typescript@>=2.7
└─┬ @rushstack/eslint-config
  ├── ✕ missing peer typescript@>=3.0.0
  ├── ✕ missing peer eslint@"^6.0.0 || ^7.0.0 || ^8.0.0"
...
```

`pnpm`의 강력한 peer dependency 정책때문에 잘 안됩니다. `ts-jest`와 `ts-node`에서 `typescript`를 필요로 하는군요. 어쩔 수 없이
`typescript`는 다시 추가하고, `@rushstack/eslint-config`는 `eslint`를 필요로 하기때문에 `eslint`도 다시 추가합니다.
`@typescript-eslint/eslint-plugin`, `@typescript-eslint/parser`는 `app/board-cli`와 `domain/board-domain`에서 직접 사용하는게
아니라 `.eslintrc.js`에서 사용하기 때문에 이 둘은 `rig/core-rig/package.json`에만 남겨도 됩니다.

pnpm때문에 문서에 나와있는 것과는 달리 `typescript`와 `eslint`를 rig package로 옮기진 못했지만 `@typescript-eslint/eslint-plugin`,
`@typescript-eslint/parser`는 `rig/core-rig`에서만 의존하고 나머지 프로젝트에서는 제거했습니다. 뭔가 많이 바꿨으니 `rush update --full`을
하고 빌드와 배포까지 잘 되는지 확인합니다.

```
~/nodejs-tutorial-example-rush/app/board-cli$ rush update --full --purge && rush build && rush deploy --overwrite
...
The operation completed successfully. 
~/nodejs-tutorial-example-rush/app/board-cli$ cd../../
~/nodejs-tutorial-example-rush$ node common/deploy/app/board-cli
1) 목록 조회
2) 쓰기
x) 종료

선택: 
```

실행이 잘 됩니다.

- [x] `.eslintignore`
- [x] `.eslintrc.js`
- [ ] `.prettierignore`
- [ ] `.prettierrc.json`
- [ ] `.jest.config.js`
- [x] `tsconfig.json`

이제 `prettier`와 `jest` 설정의 중복만 없애면 됩니다.

지금까지 작성한 코드는 [nodejs-tutorial-example:chapter-5-remove-eslintrc-dup](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-5-remove-eslintrc-dup)에서 확인할 수 있습니다.

### `prettierrc.json` 중복 없애기

Rush Stack에서는 Prettier에 대해서 아래와 같이 말하고 있습니다.

> [Prettier](https://rushjs.io/pages/maintainer/enabling_prettier/): This tool manages trivial syntax aspects such as spaces, commas, and semicolons. Because these aspects
> normally don't affect code semantics, we never bother the developer with error messages about it, nor is it part of
> the build. Instead, Prettier reformats the code automatically via a git commit hook. To se this up, see the [Enabling Prettier](https://rushjs.io/pages/maintainer/enabling_prettier/)
> tutorial on the Rush website.
> 
> \- ["eslint" task \| Rush Stack](https://rushstack.io/pages/heft_tasks/eslint/#when-to-use-it)

요약하자면, Prettier는 코드의 의미(semantics)에 영향을 주지 않으니 `git`의 commit hook 정도로만 사용해도 충분하다고 합니다. commit hook에서
수행하는 작업은 최대한 빨라야 합니다. 만약 commit을 할 때마다 10초 이상씩 걸리게 된다면 커밋 한 번 한 번이 부담스러워지게 되고 커밋 하나 하나의 덩치가
커지게 됩니다. 작은 단위로 커밋을 하게되면 커밋 히스토리만으로도 작업자의 의도를 알 수 있고 커밋당 변경사항도 많지 않아 코드 리뷰때도 편합니다.
`pretty-quick` 패키지는 변경이 있는 파일에 대해서만 Prettier를 실행하기 때문에 속도가 빨라서 Rush에서도 사용을 권장하고 있습니다.

[Enabling Prettier](https://rushjs.io/pages/maintainer/enabling_prettier/) 가이드를 따라서 진행해보겠습니다. 일단
`.prettierrc.json`과 `.prettierignore`를 프로젝트 root로 옮깁니다.

```
~/nodejs-tutorial-example-rush$ mv app/board-cli/.prettierrc.json app/board-cli/.prettierignore .
~/nodejs-tutorial-example-rush$ rm domain/board-domain/.prettierrc.json domain/board-domain/.prettierignore
```

프로젝트를 세팅할 때 commit hook을 자동으로 설정하도록 [`rush init-autoinstaller`](https://rushjs.io/pages/commands/rush_init-autoinstaller/)
를 사용해서 `common/autoinstallers/rush-prettier/package.json` 파일을 생성합니다.

```
~/nodejs-tutorial-example-rush$ rush init-autoinstaller --name rush-prettier

Starting "rush init-autoinstaller"

Creating package: /Users/mj/projects/nodejs-tutorial-example-rush/common/autoinstallers/rush-prettier/package.json

File successfully written. Add your dependencies before committing.
```

`common/autoinstallers/rush-prettier`로 이동해서 `prettier`와 `pretty-quick` 의존성을 추가하고 autoinstaller를 업데이트합니다.

```
~/nodejs-tutorial-example-rush$ cd common/autoinstallers/rush-prettier
~/nodejs-tutorial-example-rush/common/autoinstallers/rush-prettier$ pnpm install prettier pretty-quick
...
dependencies:
+ prettier 2.6.2
+ pretty-quick 3.1.3

~/nodejs-tutorial-example-rush/common/autoinstallers/rush-prettier$ rush update-autoinstaller --name rush-prettier
```

지금까지의 변경사항은 아래와 같습니다. 모두 추가하고 커밋합니다.

```
~/nodejs-tutorial-example-rush/common/autoinstallers/rush-prettier$ git status                                                                                                         ─╯
On branch main
Your branch is up to date with 'origin/main'.

Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        deleted:    ../../../app/board-cli/.prettierignore
        deleted:    ../../../app/board-cli/.prettierrc.json
        deleted:    ../../../domain/board-domain/.prettierignore
        deleted:    ../../../domain/board-domain/.prettierrc.json

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        ../../../.prettierignore
        ../../../.prettierrc.json
        ../

no changes added to commit (use "git add" and/or "git commit -a")
~/nodejs-tutorial-example-rush/common/autoinstallers/rush-prettier$ cd ../../../
~/nodejs-tutorial-example-rush$ git add .
~/nodejs-tutorial-example-rush$ git commit
```

그리고 [이전에 `rush clean` 커맨드를 추가](#rush-clean-커맨드-추가)했던 것처럼 `rush prettier` 커맨드를 추가합니다. 이번에 `commandKind`는
`global`입니다.

```json-doc
// common/config/rush/command-line.json
{
  "commands": [
    ...,
    {
      "name": "prettier",
      "commandKind": "global",
      "summary": "Used by the pre-commit Git hook. This command invokes Prettier to reformat staged changes.",
      "safeForSimultaneousRushProcesses": true,

      "autoinstallerName": "rush-prettier",

      // This will invoke common/autoinstallers/rush-prettier/node_modules/.bin/pretty-quick
      "shellCommand": "pretty-quick --staged"
    }
  ]
}
```

커맨드를 추가했으니 `rush prettier`를 입력합니다. 두 번 입력합니다.

```
~/nodejs-tutorial-example-rush$ rush prettier # 첫 번째
...

dependencies:
+ prettier 2.6.2
+ pretty-quick 3.1.3
Auto install completed successfully

🔍  Finding changed files since git revision 55aae42.
🎯  Found 0 changed files.
✅  Everything is awesome!
~/nodejs-tutorial-example-rush$ rush prettier # 두 번째
...

Autoinstaller folder is already up to date

🔍  Finding changed files since git revision 55aae42.
🎯  Found 0 changed files.
✅  Everything is awesome!
```

첫 번째 실행해서는 관련 의존성을 설치하고 `pretty-quick --staged`를 실행했습니다. 두 번째 실행에서는 이미 의존성이 설치되어 있으므로
`pretty-quick --staged`를 실행했습니다. 최신 커밋과 비교해서 변경된 파일에만 Prettier를 적용하는 것을 알 수 있네요. `.ts`파일을 무작위로 선택해서
변경한 뒤 `git add`로 파일을 추가하고 `rush prettier`를 실행하면 Prettier가 파일을 검사하고 변경합니다.

```
# app/board-cli/src/index.ts를 변경하고 rush prettier를 실행
~/nodejs-tutorial-example-rush$ rush prettier
...

Autoinstaller folder is already up to date

🔍  Finding changed files since git revision 55aae42.
🎯  Found 3 changed files.
✍️  Fixing up app/board-cli/src/index.ts.
✅  Everything is awesome!
```

마지막으로 `common/git-hooks/pre-commit` 파일을 생성해서 아래 내용을 붙여넣어줍니다.

```sh
#!/bin/sh
# Called by "git commit" with no arguments.  The hook should
# exit with non-zero status after issuing an appropriate message if
# it wants to stop the commit.

# Invoke the "rush prettier" custom command to reformat files whenever they
# are committed. The command is defined in common/config/rush/command-line.json
# and uses the "rush-prettier" autoinstaller.
node common/scripts/install-run-rush.js prettier || exit $?
```

`rush install`을 입력해서 commit hook을 등록합니다. `.git/hooks/pre-commit` 파일이 위 내용으로 잘 생성됐는지 확인합니다.

```
~/nodejs-tutorial-example-rush$ rush install # autoinstaller가 hook을 설치한다.
...
Rush install finished successfully. (2.00 seconds) 

~/nodejs-tutorial-example-rush$ cat .git/hooks/pre-commit # 설치한 pre-commit hook 확인
#!/bin/sh
# Called by "git commit" with no arguments.  The hook should
# exit with non-zero status after issuing an appropriate message if
# it wants to stop the commit.

# Invoke the "rush prettier" custom command to reformat files whenever they
# are committed. The command is defined in common/config/rush/command-line.json
# and uses the "rush-prettier" autoinstaller.
node common/scripts/install-run-rush.js prettier || exit $?
```

`git` hook은 `.git/hooks`에 넣어야 동작하는데, `.git` 디렉토리는 `git` 자체에 대한 디렉토리라 이 디렉토리를 `git`으로 관리할 수는 없습니다.
그래서 Rush는 autoinstaller를 사용하는 방법으로 모든 개발자가 `pre-commit` hook을 사용할 수 있도록 자동화했습니다. 다른 방법으로는 `git`이
사용하는 hook 디렉토리를 `.git/hooks`가 아니라 `.git` 디렉토리 바깥으로 변경하는 방법이 있는데, 어쨌든 `git clone`을 한 뒤에 특정한 작업을
수행해줘야 한다는 점은 변하지 않습니다.

※ 저는 [Prettier - Code formatter](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode) VSCode
플러그인으로 VSCode에서 파일을 저장할 때마다 해당 파일에 대해서 자동으로 Prettier를 실행하도록 했습니다.

```json-doc
// .vscode/settings.json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll": true,
    "source.fixAll.eslint": true
  },
  ...
}
```

지금까지 작성한 코드는 [nodejs-tutorial-example:chapter-5-remove-jest-dup](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-5-remove-jest-dup)에서 확인할 수 있습니다.

- [x] `.eslintignore`
- [x] `.eslintrc.js`
- [x] `.prettierignore`
- [x] `.prettierrc.json`
- [ ] `.jest.config.js`
- [x] `tsconfig.json`

이제 `jest.config.js`만 남았군요.

### `jest.config.js` 중복 없애기

Jest의 경우는 약간 복잡합니다. 타입스크립트와 Jest를 함께 사용하는 방법은 `babel-jest`를 사용하거나 `ts-jest`를 사용하는 방법이 있습니다.
그러나 [`babel-jest`와 `ts-jest`가 모두 마음에 들지 않았던 Heft팀은 `heft-jest`를 만들었습니다.](https://rushstack.io/pages/heft_tasks/jest/#differences-from-ts-jest)
`heft-jest`는 기존의 Jest가 mocking하는 방식에서 몇 가지 주의해야 할 점이 생기고, 이에 대해서 개발자가 실수하지 않도록 [`eslint` plugin](https://www.npmjs.com/package/@rushstack/eslint-plugin#rushstackhoist-jest-mock)을
만들어서 제공합니다. 제가 말씀드린 내용 모두 ["jest" task 문서](https://rushstack.io/pages/heft_tasks/jest/)에 자세하게 나와있습니다.

["Riggable" config files](https://rushstack.io/pages/heft/rig_packages/#2-riggable-config-files)는 개별 프로젝트에 설정 파일이
없을 때 rig package의 config의 설정을 참고하게 되는 파일들입니다. `jest.config.json`도 riggable config이므로 설정 파일을 rig package로
옮겨서 중복을 제거할 수 있습니다.

문서를 따라서 설정해보겠습니다.

`rig/core-rig` 프로젝트에 `@rushstack/heft`, `@rushstack/heft-jest-plugin`, `@types/heft-jest` 의존성을 설치합니다.

```
~/nodejs-tutorial-example-rush$ cd rig/core-rig
~/nodejs-tutorial-example-rush/rig/core-rig$ rush add -p @rushstack/heft -p @rushstack/heft-jest-plugin --caret --dev
~/nodejs-tutorial-example-rush/rig/core-rig$ rush add -p @types/heft-jest --exact --dev
```

그리고 `rig/core-rig/profiles/default/tsconfig.json`의 `types`에 `"node"`와 `"heft-jest"`를 추가합니다. `"sourceMap"` 옵션도
켜줍니다. `"sourceMap"`은 컴파일 결과물에 `.js.map` 파일도 출력합니다. 이 파일이 있어야 `heft-jest`가 테스트를 수행할 수 있습니다.

```json-doc
{
  "compilerOptions": {
    ...,
    "types": ["node", "heft-jest"],
    "sourceMap": true,
    ...,
  }
}
```

프로젝트 root에 `rig/core-rig/profiles/default/config/heft.json` 파일을 아래와 같이 추가합니다.

```json-doc
// config/heft.json
{
  "$schema": "https://developer.microsoft.com/json-schemas/heft/heft.schema.json",
  "heftPlugins": [{ "plugin": "@rushstack/heft-jest-plugin" }]
}
```

`rig/core-rig/profiles/default/config/jest.config.json` 파일도 추가합니다.

```json-doc
// config/jest.config.json
{
  "extends": "@rushstack/heft-jest-plugin/includes/jest-shared.config.json",
  "collectCoverageFrom": ["src/**/*.{ts,tsx}", "!src/**/__test__/**"],
  "testPathIgnorePatterns": ["/node_modules/", "/dist/"]
}
```

이전에 추가했던 `domain/board-domain/config/rig.json`을 복사해서 `app/board-cli/config/rig.json`을 생성합니다.

```
~/nodejs-tutorial-example-rush/rig/core-rig$ cd ../../
~/nodejs-tutorial-example-rush$ mkdir app/board-cli/config
~/nodejs-tutorial-example-rush$ cp domain/board-domain/config/rig.json app/board-cli/config
```

더 이상 필요없어진 `app/board-cli/jest.config.js`, `domain/board-domain/jest.config.js` 파일을 제거하고 테스트를 실행합니다.

```
~/nodejs-tutorial-example-rush$ rm app/board-cli/jest.config.js domain/board-domain/jest.config.js
~/nodejs-tutorial-example-rush$ cd app/board-cli
~/nodejs-tutorial-example-rush/app/board-cli$ heft test
Project: app-board-cli@1.0.0
Heft version: 0.44.5
Node version: v14.19.1
Error: The transpiler output folder does not exist:
  /Users/mj/projects/nodejs-tutorial-example-rush/app/board-cli/lib
Was the compiler invoked? Is the "emitFolderNameForTests" setting correctly specified in config/typescript.json?
```

`config/typescript.json` 파일이 없다고 에러가 발생합니다. `rig/core-rig/profiles/default/config/typescript.json` 파일을 생성해
아래 내용을 넣어줍니다.

```json-doc
// rig/core-rig/profiles/default/config/typescript.json
{
  "$schema": "https://developer.microsoft.com/json-schemas/heft/typescript.schema.json",
  "emitFolderNameForTests": "./dist"
}
```

다시 `heft test`를 입력하면 테스트가 모두 성공합니다.

```
~/nodejs-tutorial-example-rush/app/board-cli$ heft test
...

Tests finished:
  Successes: 38
  Failures: 0
  Total: 38
 ---- Test finished (10772ms) ---- 
-------------------- Finished (14.993s) --------------------
Project: app-board-cli@1.0.0
Heft version: 0.44.5
Node version: v14.19.1
```

`domain/board-domain` 프로젝트에서도 테스트가 성공하는지 확인합니다.

```
~/nodejs-tutorial-example-rush/app/board-cli$ cd ../../domain/board-domain
~/nodejs-tutorial-example-rush/domain/board-domain$ heft test
...

Tests finished:
  Successes: 7
  Failures: 0
  Total: 7
 ---- Test finished (2620ms) ---- 
-------------------- Finished (6.316s) --------------------
Project: board-domain@1.0.0
Heft version: 0.44.5
Node version: v14.19.1
```

마찬가지로 모두 성공하는군요. 모든 프로젝트의 테스트를 한 번에 실행할 수 있는 `rush test` 커맨드를 추가합니다.

```json-doc
// common/config/rush/command-line.json
{
  ...,
  "commands": [
    ...,
    {
      "commandKind": "bulk",
      "name": "test",
      "summary": "Run tests of each project.",
      "description": "Run tests of each project.",
      "enableParallelism": true
    }
  ]
}
```

`app/board-cli/package.json`과 `domain/board-domain/package.json`의 `"test"` 스크립트를 `"heft test --clean"`으로 변경합니다.
`rig/core-rig/package.json`은 `"test": ""`로 입력합니다. `rush test`가 잘 되는지 확인합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush test
...

==[ SUCCESS: 2 operation ]=====================================================                                     
                                                                                                                    
These operations completed successfully:                                                                            
  board-domain    5.31 seconds 
  app-board-cli   7.98 seconds 
```

잘 되는군요.

- [x] `.eslintignore`
- [x] `.eslintrc.js`
- [x] `.prettierignore`
- [x] `.prettierrc.json`
- [x] `.jest.config.js`
- [x] `tsconfig.json`

모든 설정의 중복을 Rush Stack에서 권장하는 방식으로 제거했습니다 🎉🎊🎉⭐️🔥☀️🎉
 
## 기타 설정

### 빌드 과정에 테스트 포함하기

이전에 [우리가 원하는 빌드 과정](#중복-설정-제거하기-with-heft)은 아래와 같다고 했습니다.

1. `prettier`로 코드 스타일 정리
2. `eslint`로 규칙에 어긋나는 코드가 있는지 확인
3. `typescript`로 컴파일

1번은 `pre-commit` hook으로 대체했고, 개별 프로젝트에서 `heft build`를 입력하면 `eslint`로 `lint` 과정을 수행한 뒤에 `typescript`로
컴파일을 해서 결과물을 `dist` 디렉토리에 생성합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ heft clean
~/nodejs-tutorial-example-rush/domain/board-domain$ heft build
Project build folder is ".../nodejs-tutorial-example-rush/domain/board-domain"
Using rig configuration from ./node_modules/core-rig/profiles/default
Starting build ---- Pre-compile started ---- 
 ---- Pre-compile finished (1ms) ---- 
 ---- Compile started ---- 
[typescript] The TypeScript compiler version 4.6.3 is newer than the latest version that was tested with Heft (4.5); it may not work correctly.
[typescript] Using TypeScript version 4.6.3
[eslint] Using ESLint version 8.12.0
 ---- Compile finished (3267ms) ---- 
 ---- Bundle started ---- 
 ---- Bundle finished (1ms) ---- 
 ---- Post-build started ---- 
 ---- Post-build finished (2ms) ---- 
-------------------- Finished (3.952s) --------------------
Project: board-domain@1.0.0
Heft version: 0.44.5
Node version: v14.19.1

~/nodejs-tutorial-example-rush/domain/board-domain$ ls dist
article
```

빌드 과정을 보면 컴파일 과정에서 `eslint`를 실행합니다. `eslint` 규칙을 위배한 후에 빌드를 하면 규칙에 대한 경고까지 출력이 됩니다. 빌드 이후에
테스트까지 수행하려면 어떻게 해야할까요? `heft build && heft test`를 입력하면 될까요? 그럴 필요 없이 그냥 `heft test`만 입력해도 됩니다.

[heft 커맨드의 설명서](https://rushstack.io/pages/heft/cli/#heft)를 보면 아래처럼 나와있습니다.

```
...
Positional arguments:
  <command>
    clean        Clean the project
    build        Build the project.
    start        Run the local server for the current project
    test         Build the project and run tests.
...
```

이미 `heft test`에 빌드 과정이 포함되어 있습니다. `ts-jest`는 타입스크립트 파일을 컴파일 없이 그대로 입력받아서 테스트를 진행하지만 `heft-jest`를
수행하려면 타입스크립트를 자바스크립트로 컴파일해야 하기 때문에 테스트 과정에 빌드가 포함되는 것 같습니다. 저는 매번 빌드를 할 때마다 테스트를 실행하기를
선호하기 때문에 `package.json`의 `build` 스크립트에서 `heft build --clean` 대신 `heft test --clean`을 사용하도록 하겠습니다. 직전에
추가했던 `rush test` 커맨드는 다시 제거해도 되겠습니다. 나중에 빌드 과정에서 테스트를 제거하고 싶을 때 다시 `rush test` 커맨드를 추가하면 될 것
같습니다.

`app/board-cli/package.json`, `domain/board-domain/package.json`의 `"build"` 스크립트를 `heft build --clean`으로 변경하고
`app/board-cli/package.json`, `domain/board-domain/package.json`, `rig/core-rig/package.json`에서 `"test"` 스크립트를
제거합니다. `common/config/rush/command-line.json`에서 다시 `test` 커맨드를 제거합니다.

그리고 `rush deploy`까지 성공하고 애플리케이션 실행이 잘 되는지 확인합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ cd ../../
~/nodejs-tutorial-example-rush$ rush clean && rush rebuild && rush deploy --overwrite
~/nodejs-tutorial-example-rush$ node common/deploy/app/board-cli
1) 목록 조회
2) 쓰기
x) 종료

선택: 
```

잘 되는군요.

지금까지 작성한 코드는 [nodejs-tutorial-example:chapter-5-build-with-test](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-5-build-with-test)에서 확인할 수 있습니다.

### 의존성 버전 통일

#### `ensureConsistentVersion` 활성화

Rush 문서에서는 `ensureConsistentVersion` 옵션을 켜놓기를 권장하고 있습니다. 편한 진행을 위해서 일단은 끄고 튜토리얼을 진행했지만 이제는 켜야 할
때가 온 것 같습니다. 프로젝트 root의 `rush.json`에서 해당 옵션을 켜줍니다.

```json-doc
// rush.json
{
  ...,
  "ensureConsistentVersions": true,
  ...
}
```

그리고 `rush update`를 입력합니다.

```
~/nodejs-tutorial-example-rush$ rush check

Starting "rush update"                                                                                              
                                                                                                                    
eslint                                                                                                              
  ^8.12.0                                                                                                           
   - app-board-cli                                                                                                  
   - board-domain                                                                                                   
   - core-rig                                                                                                       
  ^8.11.0                                                                                                           
   - preferred versions from common-versions.json
   
prettier                                                                                                            
  2.5.1                                                                                                             
   - app-board-cli                                                                                                  
   - preferred versions from common-versions.json                                                                   
  ^2.6.1                                                                                                            
   - board-domain                                                                                                   

ts-jest
  ^27.1.3
   - app-board-cli
   - preferred versions from common-versions.json
  ^27.1.4
   - board-domain

typescript
  ^4.6.3
   - app-board-cli
   - board-domain
   - core-rig
  ^4.6.2
   - preferred versions from common-versions.json

@typescript-eslint/eslint-plugin
  ^5.18.0
   - core-rig
  ^5.14.0
   - preferred versions from common-versions.json
 
@typescript-eslint/parser
  ^5.18.0
   - core-rig
  ^5.14.0
   - preferred versions from common-versions.json
```

이전에 작성했던 `common-version.json`와 버전이 안 맞아서 경고가 나는군요. `common-version.json`의 `perferredVersions`는 특정 의존성을
과거 버전에 고정해야 할 때 사용합니다. 프로젝트간의 의존성 버전을 맞추기 위해서는 `ensureConsistentVersion`만으로 충분하니
`common-version.json`의 `preferredVersions`에 작성했던 버전들을 지워주고 다시 `rush update`를 합니다.

```
~/nodejs-tutorial-example-rush$ rush update
Starting "rush update"

prettier
  2.5.1
   - app-board-cli
  ^2.6.1
   - board-domain

ts-jest
  ^27.1.3
   - app-board-cli
  ^27.1.4
   - board-domain

Found 2 mis-matching dependencies!
```

`prettier`와 `ts-jest`의 버전이 맞지 않는군요. `prettier`는 `common/autoinstallers/rush-prettier`에서 관리하고 `ts-jest`는
`heft-jest` 때문에 필요가 없어졌으니 `app/board-cli`와 `domain/board-domain`에서 두 의존성을 모두 지워버리고 `rush update`를 합니다. 

```
~/nodejs-tutorial-example-rush$ rush update

...
Rush update finished successfully. (2.47 seconds)
```

잘 되네요. `@types/jest`도 `@types/heft-jest` 때문에 필요가 없어졌으니 지워줍니다.

### `@types/*` 의존성 버전 고정

Rush 문서를 보다보면 `@types/*` 의존성([DefinitelyTyped 프로젝트](https://github.com/DefinitelyTyped/DefinitelyTyped))에 대해서는
caret(`^`)이나 tilde(`~`)를 사용하지 말고 정확한 버전을 지정하라고 합니다. 타입 관련된 것은 빡빡할수록 좋긴 합니다.

<figure>
    <img alt="--save-exact 1" src="/res/17-exact-type-1.png" />
    <figcaption><a href="https://rushstack.io/pages/heft_tutorials/getting_started/"><code>--save-exact</code>로 정확한 버전을 명시하도록 한다</a></figcaption>
</figure>

<figure>
    <img alt="--save-exact 1" src="/res/18-exact-type-2.png" />
    <figcaption><a href="https://rushstack.io/pages/heft_tutorials/adding_tasks/">여기서도 마찬가지</a></figcaption>
</figure>

저희는... `@types/node`만 사용하는군요. caret과 tilde 없이 정확한 버전을 사용하도록 변경합니다. 저는 `17.0.23` 버전을 사용합니다.

### VSCode로 테스트 디버깅하기

`heft-jest`관련 문서를 보면 [VSCode로 테스트를 실행해서 디버깅할 수 있도록 설정](https://rushstack.io/pages/heft_tasks/jest/#debugging-jest-tests)하는
부분이 나옵니다. 저희도 해봅시다.

[https://github.com/microsoft/rushstack](https://github.com/microsoft/rushstack) 프로젝트에는 `apps` 디렉토리 이하에 여러 개의
프로젝트가 있는데요, 프로젝트마다 `.vscode/launch.json`을 갖습니다. VSCode로 프로젝트를 열 때 모노레포의 root이 아니라 `app/board-cli`나
`domain/board-domain`을 직접 열어야 `.vscode/launch.json` 설정이 적용됩니다.

`app/board-cli/.vscode/launch.json`과 `domain/board-domain/.vscode/launch.json`에 아래와 같이 작성합니다.

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Jest tests",
      "program": "${workspaceFolder}/node_modules/@rushstack/heft/lib/start.js",
      "cwd": "${workspaceFolder}",
      "args": ["--debug", "test", "--clean"],
      "console": "integratedTerminal",
      "sourceMaps": true
    }
  ]
}
```

그리고 VSCode로 `app/board-cli` 디렉토리를 열어서 `src/index.test.ts`에 브레이크 포인트를 찍고 메뉴의 View -> Run을 선택해서 좌측 상단의
초록색 ▷버튼(F5)로 테스트를 실행하면 디버거가 브레이크 포인트에서 멈추는 것을 확인할 수 있습니다.

<figure>
    <a href="/res/19-debug.png" target="_blank">
      <img alt="Debugging" src="/res/19-debug.png" style="max-width: initial !important; width:100%" />
    </a>
    <figcaption>라인 넘버 좌측을 누르면 빨간 점(브레이크 포인트)이 찍힌다.<br/>이 상태에서 F5로 테스트를 실행하면 브레이크 포인트에서 테스트가 멈춘다.</figcaption>
</figure>

### Rush Stack 권장 eslint 설정 사용

[`@rushstack/eslint-config`](https://www.npmjs.com/package/@rushstack/eslint-config)은 대규모 모노레포를 위한 `eslint` 설정
모음집입니다. 링크의 Philosophy와 Implementation은 꼭 읽어보세요!

`.eslintrc.js`에서 `extends`할 수 있는 플러그인 목록은 아래와 같습니다. 3개의 Profile중에는 하나만 상속하고 필요에 따라 여러 개의 Mixins를
선택해서 상속할 수 있습니다.

- Profile
  - `@rushstack/eslint-config/profile/node`
    - 기본. 제일 빡빡한 설정입니다.
  - `@rushstack/eslint-config/profile/node-trusted-tool`
    - 기본 설정보다 덜 빡빡합니다. 외부로 공개되지 않는 상황에서만 사용합니다.
  - `@rushstack/eslint-config/profile/web-app`
    - 웹 브라우저 관련 취약점을 예방하기 위한 규칙이 포함되어 있습니다. Node.js와 웹 브라우저에서 모두 실행되는 프로젝트인 경우에도 선택합니다.
- Mixins
  - `@rushstack/eslint-config/mixins/friendly-locals`
    - 로컬 변수에도 타입을 명시하도록 강제합니다.
  - `@rushstack/eslint-config/mixins/packlets`
    - 모노레포의 패키지로 분리하지는 않으면서도 하나의 패키지 안에서 엄격한 `import` 정책을 설정하기 위해 사용합니다. [`@rushstack/eslint-plugin-packlets`](https://www.npmjs.com/package/@rushstack/eslint-plugin-packlets)를 참고하세요
  - `@rushstack/eslint-config/mixins/tsdoc`
    - 주석이 [TSDoc](https://github.com/Microsoft/tsdoc) 표준을 지키도록 강제합니다. [API Extractor](https://api-extractor.com/)같은 툴을 사용할 때 도움이 됩니다.
  - `@rushstack/eslint-config/mixins/react`
    - 리액트를 사용할 때 추가합니다.

뷔페에 온 느낌이네요. 기본 Profile인 `@rushstack/eslint-config/profile/node`과
`@rushstack/eslint-config/mixins/friendly-locals` 믹스인을 선택해서 상속해보겠습니다.

`rig/core-rig` 프로젝트로 이동해서 `@rushstack/eslint-config` 의존성을 추가합니다.

```
~/nodejs-tutorial-example-rush$ cd rig/core-rig
~/nodejs-tutorial-example-rush/rig/core-rig$ rush add -p @rushstack/eslint-config --dev --caret
```

`rig/core-rig/profiles/default/.eslintrc.js`의 `"extends"` 속성에 `@rushstack/eslint-config/profile/node`과
`@rushstack/eslint-config/mixins/friendly-locals`를 추가합니다.

```javascript
// rig/core-rig/profiles/default/.eslintrc.js
module.exports = {
  root: true,
  parser: "@typescript-eslint/parser",
  plugins: ["@typescript-eslint"],
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "prettier",
    "@rushstack/eslint-config/profile/node",
    "@rushstack/eslint-config/mixins/friendly-locals"
  ],
};
```

`rush build`를 입력하면 아래와 같이 에러가 발생합니다.

```
~/nodejs-tutorial-example-rush/rig/core-rig$ rush build

...
--[ FAILURE: board-domain ]----------------------------------[ 3.55 seconds ]--

Error: Plugin "@typescript-eslint" was conflicted between
"--config » ./node_modules/core-rig/profiles/default/.eslintrc » @rushstack/eslint-config/profile/node" and
"--config » ./node_modules/core-rig/profiles/default/.eslintrc » plugin:@typescript-eslint/recommended » ./configs/base".

Operations failed.
```

`@typescript-eslint`와 충돌이 있습니다. `.eslintrc.js`에서 `@typescript-eslint` 관련 설정을 제거하고 `git`의 `pre-commit` hook 덕분에
필요없어진 `"perttier"`도 제거합니다. `@rushstack/eslint-config/profile/node`이 충분히 고난을 가져다 줄 것 같으니 문서에 나와있는 것처럼
`eslint:recommended`도 제거하고 `@rushstack/eslint-config/profile/node`, `@rushstack/eslint-config/mixins/friendly-locals`
만 남겨봅시다...


```javascript
// rig/core-rig/profiles/default/.eslintrc.js
module.exports = {
  root: true,
  extends: [
    "@rushstack/eslint-config/profile/node",
    "@rushstack/eslint-config/mixins/friendly-locals"
  ],
};
```

`rig/core-rig/package.json`에서 필요없어진 `@typescript-eslint/eslint-plugin`, `@typescript-eslint/parser`,
`eslint-config-prettier`를 제거하고 `rush update`를 한 뒤에 `rush build`를 실행합니다.

```
~/nodejs-tutorial-example-rush/rig/core-rig$ rush update
~/nodejs-tutorial-example-rush/rig/core-rig$ rush build
...
  [eslint] src/index.ts:11:1 - (@typescript-eslint/no-floating-promises) Promises must be awaited, end with a call to .catch, end with a call to .then with a rejection handler or be explicitly marked as ignored with the `void` operator.


Operations failed.
```

세상에... 다 적을 수 없지만 한 2억개 정도 경고와 에러가 발생한 것 같습니다. 경고는 일단 놔두고 에러라도 처리해야 할 것 같네요. 다행히 에러 종류는
[`no-floating-promises`](https://github.com/typescript-eslint/typescript-eslint/blob/v5.6.0/packages/eslint-plugin/docs/rules/no-floating-promises.md)
밖에 없어서 [`void` operator](https://github.com/typescript-eslint/typescript-eslint/blob/v5.6.0/packages/eslint-plugin/docs/rules/no-floating-promises.md#ignorevoid)를
사용하는 방식으로 해결했습니다: [https://github.com/myeongjae-kim/nodejs-tutorial-example/commit/2ffa40a6c94faf01ea05634db98f19c97108ebb7](https://github.com/myeongjae-kim/nodejs-tutorial-example/commit/2ffa40a6c94faf01ea05634db98f19c97108ebb7)

```javascript
// rig/core-rig/profiles/default/.eslintrc.js
module.exports = {
  root: true,
  extends: [
    "@rushstack/eslint-config/profile/node",
    "@rushstack/eslint-config/mixins/friendly-locals",
  ],
  rules: {
    "no-void": ["error", { allowAsStatement: true }],
  },
};
```

```
~/nodejs-tutorial-example-rush/rig/core-rig$ rush build

...
Operations succeeded with warnings.
```

이제 경고가 1억개정도 남아있습니다. `rush build`는 `eslint` 경고가 발생하지만 성공하긴 합니다. `rush deploy --overwrite`까지 해서 배포용
빌드도 실행이 잘 되는것을 확인했습니다.

### 험난한 모노레포의 길

프로젝트 구성하면서 설정에 실패하고 문제를 해결하는 과정까지 모두 자세하게 담았습니다. 생각보다 글이 길어졌지만 어느정도 만족스러운 모노레포를 구성할 수
있어서 좋네요.

새로운 비즈니스 로직이 추가되면 `domain` 디렉토리 이하에 프로젝트를 추가합니다. 새로운 애플리케이션이 필요하다면 `app`이하에 프로젝트를 추가하면 됩니다.
새로운 설정용 프로젝트가 필요하다면 `rig`이하에 rig package 역할의 프로젝트를 추가하면 됩니다. 이 3가지의 구분 외에도 다양한 레이어가 만들어질 수
있습니다. Remote 요청을 보내는 `client/xxx-client` 프로젝트라든지, 데이터베이스나 캐시같은 외부 의존성을 위한 `infrastructure` 라든지...
일관성있게 관리하면서 중복을 제거한다면 모노레포는 엄청난 크기로 커지면서도 일정 수준의 복잡도를 유지할 수 있게 됩니다.

[Rush는 대규모 모노레포를 관리하기 위한 정책을 강제하는 기능도 제공합니다.](https://rushjs.io/pages/maintainer/setup_policies/)
Lerna는 '뼈대만 제공할 테니 나머지는 알아서 하라'는 느낌이었다면, Rush는 '우리가 해보니까 이런게 필요하더라 그래서 준비해뒀어'의 느낌입니다.
정성대님은 ['Rush로 프론트엔드 모노레포 도입기'](https://medium.com/mildang/rush로-프론트엔드-모노레포-도입기-5da0c5bc9b30)에
'오픈소스 저장소가 아닌 애플리케이션용 저장소를 위해서는 Lerna보다 Rush가 낫다'고 했는데 그 말에 충분히 공감을 합니다.

모노레포의 길을 쉽지 않지만, 소수의 인원이 고생해서 틀을 만들어 놓으면 그 이득은 조직 전체에 복리로 돌아온다고 생각합니다.
