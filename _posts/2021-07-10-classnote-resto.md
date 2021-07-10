# 웹 에디터 모듈 개발 회고

6개월 가량 웹 에디터를 제작하며 경험한 react 모듈화에 대한 정리

# 🤮 요구 사항

에디터 개발에는 많은 요구사항이 있었지만, 세 가지 까다로운 요구사항이 있었다.

1. 하나는 모듈화 해서 어느 서비스에서든 기능을 이용할 수 있고,
2. 다른 하나는 동적으로 생성되는 에디터 박스를 하나의 툴바로 제어할 수 있으며,
3. 마지막으로 생성된 에디터 데이터는 앱과도 같은 포맷으로 연동되어야 한다.

## 에디터 모듈화

에디터 모듈화는 리액트 기반에서 모듈만 호출하면 사용할 수 있어야 한다. 라는 조금 추상적인 조건 이었다.

에디터 모듈을 호출하는 모 웹과의 통신용 인터페이스를 내부에 설계하고, 기능들을 외부에서 제어할 수 있도록 기초값을 포함한 파라미터를 외부에서 입력 받도록 구현 했다.

내부 설계는 되었지만, 이 모듈을 어떻게 편하게 가져다 쓸까? 잠깐 고민을 하다 NPM에 올리기로 결정했다. 방법이 없었다. 모듈을 여러곳에서 편하게 가져다 쓸 방법은 NPM이 최선이라 생각하고 초기 라이브러리화 작업을 시작했다.

## NPM 패키지

NPM 패키지를 만드는 건 배포되는 패지키의 기본 구조만 이해하면 어렵지 않다.

다만 패키지를 빌드 하는 방법에서 다양성을 보이는데, 여러면에서 유리한 [rollup](https://rollupjs.org/guide/en/)을 통해서 패키지 빌드를 구현했다. ([모듈러에 대한 정리](https://wormwlrm.github.io/2020/08/12/History-of-JavaScript-Modules-and-Bundlers.html))

rollup을 선택한건 실수가 아니었지만, 개발이 어느정도 진행 된 뒤 문제가 발생했다. 모듈 구현이 어느정도 갖춰진 후 패키지 빌드를 진행 했는데 이때는 문제가 없었다. 1차 개발완료를 해야 하는 시점에서 빌드에 문제가 생겼고, 다급한 일정중에 해결이 되지 않아 TSC를 이용한 방식으로 변경 했다.

NPM 패키지를 빌드하는덴 몇 가지 주의사항이 따른다. 이는 다른 패키지 제작 글에서 좀 더 자세하게 다루겠다.

- package.json 을 명확하게 설정해야 한다. 모듈을 사용하는 쪽의 패키지들과 충돌을 일으킬 수 있어 불필요한 패키지를 제거하고, peerDefendencies를 통해 의존성 패키지를 정확하게 명세해줘야 한다.
- asset 파일의 경우 정확한 위치에 옮겨줘야 한다. 따라서 흩어져있지 않고 한 군데 모여있어야 불필요한 에러를 줄일 수 있다. rollup을 사용해서 빌드시에 import 되는 여러 파일들의 데이터가 직접 주입되는 경우가 있다. 이를 주의해서 불필요한 파일들이 파일에 직접 포함되지 않도록 해야 한다.

## 데이터 연동

에디터 모듈은 웹에서만 사용되지 않고, 오히려 앱이 주력인 어플리케이션의 일부이다. 따라서 앱과의 데이터 연동이 필수적이었다.

무난한 json을 통해 작성된 데이터를 포장하고 불러오는 작업을 수행 했다. 다른 부분은 단순한 데이터의 전달이었는데 HTML데이터를 전달하는 부분이 문제였다.

### HTML 포맷

텍스트 박스에서 사용한 Draft는 데이터를 글자 단위로 제어하는 라이브러리이기 때문에 데이터를 내보낼때는 공통 포맷을 맞출 필요가 있었다. 모든 라이브러리에서 HTML로의 전환 및 입력을 지원 했기 때문에 자연스럽게 HTML로 결정 되었다.

IOS/안드로이드에서 사용하는 에디터 라이브러리를 통일할 수 없는 상황이었기 때문에 포맷을 최종 통일하기까지는 오랜 시간이 걸렸다. 중간에 클라이언트의 요구사항이 변경되어 HTML의 태그 중 span태그만 사용할 수 있도록 변경되었다.

---

# 🤸 동적인 에디터 박스 제어

에디터 박스는 여러 형태로 생성 될 수 있었다.

텍스트 입력, 맞춤법 검사, 개별 알림장, 지도, 숙제(할일 목록), 일정, 사진, 동영상, 파일 

9가지 형태를 띄고 있었으며, 가장 신경을 써야 할 부분은 텍스트 입력과 여러개의 파일 업로드 처리 였다.

## 텍스트 입력

텍스트 입력도 단순하진 않았다. 단순 텍스트 외에 글자별 색상, 배경 색상, 일반적인 폰트 효과등이 필요했으며 맞춤법 검사와도 연동 돼서 전체 또는 필요한 부분만 맞춤법 검사를 수행해야 한다.

더 어려운점은 상단에 고정된 텍스트 툴바 하나로 현재 선택된 글씨에 대해서만 제어를 하게 해야했다.

이 부분을 해결하기 위해 많은 고민과 테스트를 수행했다. 

### 구독/발행 패턴

여러 방식을 고민하다 구독/발행 패턴(publish/subscribe pattern)을 사용하기로 결정했다.

앵귤러에 기본 채용된 rx/JS를 사용하며 얻은 경험으로 가장 적합하다 판단 했으며, 이는 개발하며 했던 여러 선택중 가장 좋은 선택이었다고 판단한다.

구독/발행 패턴을 Redux를 통해 구현했다. 이벤트 리듀서가 하나 존재하고, 각 박스 컴포넌트 들은 생성되며 이벤트 리듀서를 구독하는 방식을 취했다. 에디터 툴 박스나, 텍스트 툴 박스에서 발행되는 데이터를 이벤트 리듀서가 취합하고 각 컴포넌트는 생성된 이벤트를 캐치하여 본인의 역할을 수행 했다. 개발하며 추가 된 기능들에 대해서도 유연하게 대응 할 수 있어 유용한 전략 이었다.

### Draft.js

일반적인 html 형태의 접근으로는 맞춤법 검사 해결이 어려울 듯 하여 여러 글쓰기 라이브러리들을 검색 했고, Draft.js 를 최종적으로 선택했다.

Draft는 에디터에 입력되는 문자열을 글자 단위로 나누고, 각 글자의 속성에 스타일을 주입하고 최종 화면에 결과물(HTML)이 보여지는 형태였다. 따라서 맞춤법 검사나 여러 스타일이 복잡하게 겹쳐져도 데이터 관리에 큰 문제가 없고, 앱의 포맷이 변경되어도 유연하게 대응할 수 있다고 판단 했다.

결과적으로 보면 잘못된 선택이었는데, Draft가 한글 정확히는 조합형 문자에 대해 몇 가지 버그가 있었고 해결하지 못한 몇 가지 문제들은 우회적인 눈가림을 통해서 기능 구현을 해야 했다. 조합형 문자가 지닌 문제점에 대해서는 다른 글에서 다루겠다.

## 맞춤법 검사

맞춤법 검사는 다음에서 제공되는 API를 그대로 사용했다. 모듈에서 직접 호출하진 못하고 인터페이스를 통해 호출을 의뢰하고 리턴 된 데이터를 텍스트 에디터에 적용 했다.

아래는 맞춤법 검사의 일부 데이터 구조이다.

```jsx
const APISampleData = [
  {
    sentence: '첫번째 빨간색 먹었읍니다.',
    result: [
      {
        input: '첫번째',
        output: '첫 번째',
        etype: 'space',
        di: [
          {
            word: '첫',
            cword: '',
            etype: 'no_error',
            etyped: '',
            offset: '0',
            eoffset: '2',
          },
          {
            word: '번째',
            cword: '',
            etype: 'space',
            etyped: 'cit',
            offset: '3',
            eoffset: '8',
          },
        ],
      },
      {
        //...생략
      }
    ]
  }, {
    // ...두번째 문장 데이터
  }
]
```

입력된 텍스트와 맞춤법이 수정된 결과 데이터를 일치 시키고 수정하는데는 어려움이 존재 했다.

총 글자 수가 바뀌는 점이 가장 큰 문제였다. 맞춤법 적용 대상의 단어의 글자가 늘어난 경우와 드물지만 글자수가 줄어드는 경우까지 고려해야 했다. 많은 고민을 하다 텍스트의 offset을 기준으로 작업했는데, offset은 글자가 아닌 글자의 위치를 나타내기 때문에 보정을 해서 수정 작업을 진행 했다.

```jsx
0 1  2 3    0 1  2 3 4
 첫 번 째      첫 _ 번 째

이 경우 수정 대상은 1-3 offset이며,
수정데이터의 1-4 offset 데이터를 적용해야 한다.
```

추가적으로 고려해야 될 내용은 맞춤법 검사 결과에서 유저가 선택한 검사내용만 적용돼야 했고, 변경되는 데이터의 스타일이 변경 된 후에도 그대로 남아있어야 한다는 점 이었다. 

## 파일 업로드

파일 업로드는 3가지 형태였는데 가장 괴로운 부분은 여러개의 업로드를 동시에 진행하고, 실패한 케이스는 재업로드나 실패처리를 해줘야 했다. 

개발 초기에는 모 웹의 인터페이스가 적용되지 않아 직접적으로 API를 호출해서 업로드를 적용하고 결과값만 갱신해서 사용했는데, 나중에 인터페이스를 적용하며 문제가 생겼다.

전송되는 파일 데이터들은 form으로 인터페이스를 통해 전달 했는데, 전송하던 폼 데이터를 그대로 전송해서 redux에 문제가 생겼다. 파일 같은 binary 데이터는 리덕스의 [Non-Serializable value](https://redux.js.org/style-guide/style-guide#do-not-put-non-serializable-values-in-state-or-actions) 원칙을 위배한다. (지원하는 미들웨어도 있다.)

[URL.createObjectURL](https://developer.mozilla.org/ko/docs/Web/API/URL/createObjectURL)을 이용해서 url만 redux에 전달해도 해결 될 문제였는데 파일 업로드 컴포넌트 구조를 바꾸는 바보짓을 했었다. 이 얼마나 멍청한 일인가.

이때는 여러 일이 동시에 터지고 있어서 이런 단순한 문제 해결도 하지 못했었던 것 같다. (라고 변호해본다.)

---

# 📲 통신 인터페이스

독립적으로 동작하고 결과를 외부에 빠르게 제대로 전달을 해야 하기 때문에 인터페이스 설계에 공을 들였다. 

엑셀 시트 10개에 달하는 사용자의 요구사항 때문.. 만은 아니다.

우스운 얘기지만, 에디터 모듈의 1차 개발이 완료될 때 까지도 모듈을 사용할 어플리케이션의 개발이 시작도 안 된 상태였다. 뇌피셜로 테스트할 수는 없어 나름의 노력을 다 했다.

사용자측에서 실제 개발이 시작되며 개발해둔 인터페이스의 80%를 없앤 건 안 비밀이다.

## Module Wrapper

에디터 모듈은 다른 서비스 없이 단독으로는 동작하지 못하는 라이브러리다. 아직 개발도 되지 않은 실제 서비스를 기다리고 싶지 않았다. 실제 서비스가 이미 개발해서 있다고 해도 다른 서비스에 올려진 상태로 테스트 하는건 무리가 있다고 판단됐다.

그래서 모듈을 감싸서 웹 어플리케이션으로 동작하는 래퍼 어플리케이션을 추가 개발했다.

이 래퍼는 인터페이스로 정의된 많은 기본 데이터를 에디터 모듈에 주입하고, 에디터에서 외부에 요청하는 많은 함수들의 대리 수행을 해줘야 했다.

## Sample Data

대부분 에디터 옵션을 조정하는 값으로 많은 기본 데이터를 지정해서 사용 했다. 

- 툴바 버튼 각각의 사용 유무
- 폰트사이즈, 패밀리, 줄간격 등의 텍스트 옵션 데이터
- 파일 종류별 스펙 명세(용량, 길이, 확장자 등)
- 그 외 기능들 상세 옵션 등

## Dummy Listener

정의된 인터페이스의 행동을 흉내내는 더미 함수들을 구현 해서 테스트를 진행했다.

기능별로 함수를 만들고 각 함수별로 동작을 수행해서 지정된 결과값을 리턴 하도록 개발했다.

파일 업로드의 경우 여러개를 동시에 처리해야 했고, 프로그레스 완료 후 다중 실패건에 대응하기 위해 랜덤으로 성공/실패 처리하도록 수정 했다.

```jsx
const dummyListener = (event: IListenerEventEvent): void => {
  if (ListenerBodyTypes.FileUpload === event.type) {
    dummyHandleFileUpload({ ...event.payload });
  } else if (ListenerBodyTypes.EditorAdd === event.type) {
    dummyHandleEditorStateAdd({ ...event.payload });
  } {
     // ...생략
  }

const dummyHandleFileUpload = (props: FileUploadRequest) => {
  //.. handle upload progress
  // return random success
}
```

---

# ⚙️ 프로젝트 구조

컴포넌트를 얼마나 어떻게 나누는가는 모든 개발자들의 고민일것이다. 

이전 프로젝트에서는 [Atomic Design](https://atomicdesign.bradfrost.com/chapter-2/)에 심취한 나머지 너무 세세하게 작업해서 모두가 힘든 프로젝트가 됐었다. 

컴포넌트를 나누는 과정에서 빠지기 쉬운 함정이 지나친 세분화다. 아토믹 디자인은 아주 좋은 개념이지만 그대로 개발에 적용하기엔 문제가 있었다. 원자와 분자단위로 세세하게 나누는 과정에서 또다른 중복 작업이 발생하고, 이를 관리하기 위한 또다른 룰이 발생했다.

개발에 참여하는 모두가 완전히 이해 한 상태에서 정확하게 진행되면 괜찮을 수 있지만, 그런 프로젝트는 세상에 없다. 현실과 이상 사이에서 어딘가 적절한 지점을 찾는 것은 인생에서도 항상 필요한 일이다.

src/
  /app - core모음, 전역적으로 사용됨
  /components - 컴포넌트
  /features - 피쳐
    /domain - 페이지의 단위 모음
  /utils - 유틸
  /hooks - 커스텀 훅

## Component

컴포넌트의 가장 중요한 목적을 어디에서도 사용할 수 있어야 한다고 정했다.

쓰기 편하되 마구잡이로 사용되지 않도록 제한장치를 걸었다.

### Theme

테마는 컴포넌트의 유형을 지정하는 필수 옵션값이다. 필요한경우 size나 다른 옵션을 넣도록 했지만, 테마는 그 컴포넌트의 형태를 결정짓는 필수 옵션으로 정했다.

예로 들면 버튼의 테마를 Round, Square, Default 등으로 주고 필요에 맞춰 곡선을 띈 형태는 Round Theme, 사각형태는 Square Theme등으로 사용할 수 있다. 쓰는 입장에서는 하나의 유형을 반드시 선택해야 하고, 이는 어플리케이션 전체의 통일성으로 이어진다.

### Props

컴포넌트의 데이터와 동작방식은 항상 컴포넌트를 사용하는 상위에서 전달하도록 했다. 타입스크립트로 전달되는 Props를 규격화 해서 의도한 동작만 수행할 수 있도록 강제했다. 

컴포넌트 제작시 특정 Redux를 호출하지 않도록 한 점은 컴포넌트의 범용성을 위해서 였다. 반드시 redux의 dispatch나 selector를 호출해야 한다면 컴포넌트가 아닌 Feature의 범주에 넣도록 했다.

## **Feature**

특정 페이지에서 사용되는 컴포넌트들은 Feature로 만들도록 가이드를 잡았다.

피쳐는 작은 페이지 조각부터 페이지까지를 전부 포함 했다.(지금은 조금 생각이 바뀌었다.)

### Domain

피쳐는 하나의 폴더안에 모두 모아뒀다. 이 domain은 경로상 feature아래에 파일들을 모아두는 개념 단위였는데, 이 안에는 작은 단위의 조각부터 페이지 전체, 그리고 이 페이지에서 사용되는 slice가 같이 있었다. 페이지에서 사용되는 모든 자원을 모아뒀다.

### Props

컴포넌트와는 다르게 Props를 거의 사용하지 않았다. 성능 문제를 제외한 대부분의 데이터는 Redux를 통해 주고 받았다.

## Container

컨테이너 개념은 사용하지 않았다. custom hook을 통해서 필요한 비즈니스 로직은 분리할 수 있었고, 피쳐에서 필요한 만큼 리덕스에서 데이터를 불러오고 전송할 수 있었기에 불필요한 단계는 줄이고 싶었다.

---

# 🌐 상태관리(Store)

리액트에 리덕스를 적용하는 건 어려운 일이다. 정확히는 리덕스는 쉽지만 리덕스를 편하게 쓰기 위해 추가로 딸려오는 부가적인 요소들이 너무 많고, 이 부가요소들로 인해 리덕스가 리액트를 도입하는데 가장 큰 걸림돌이 됐다는 건 모든 FE개발자들이 공감하는 내용일거다.

여러 고민끝에 Redux/Toolkit을 도입했는데 성공적이었다. 접근 난이도가 사가에 비해서 낮지는 않았지만, 사가의 복잡한 구조가 가져오는 혼란을 줄여주는데는 큰 역할을 했다.

### [Redux-saga](https://github.com/redux-saga/redux-saga-beginner-tutorial)

나는 익숙하다 해도 투입되는 다른 개발자들이 잘 모른다면 얘기가 되질 않는다. 이 프로젝트의 다른 목적인 퍼블리셔의 개발자화를 생각해보면 더욱 안될 일이다.

사가의 구조는 여전히 불편하고 불만스럽다. 제너레이터 문법에 대한 내 거부감도 한몫 했다.

### [Mobx](https://mobx.js.org/react-integration.html)

Mobx는 어떤가, 구미는 당기지만 항상 최종 후보에서 탈락이다. 데코레이터(@)의 적용이 간편하지만 예전의 자바처럼 개발하고 싶지 않았고, 취향에 맞지 않는다.

대세의 흐름에서 벗어나고 싶지 않은 것도 한 몫 했다. 아무리 그래도 Redux를 사용하는게 낫지 않을까 라는 고민은 항상 따라온다. 

mobx6 이후부터는 데코레이터가 제거 됐다. observer를 사용하며, rx/JS와 비슷한 형태로 흘러가는 느낌이다.

### [Context API](https://ko.reactjs.org/docs/context.html)

이전 프로젝트에서 써드파티 라이브러리를 사용하지 않고 기본만으로 해보자 하는 잘못된 의욕으로 ContextAPI만 이용해서 프로젝트를 구축한 적이 있다.

ContextAPI를 사용하면 프로젝트 구조가 많이 복잡해진다. 프로바이더로 항상 감싸고 데이터를 주입하는 과정이 원활하지 않고 아토믹 디자인을 같이 사용해서 난장판이 됐었다.

같이 프로젝트를 진행한 인원 모두가 괴로운 프로젝트로 기억된다. (참여한 모든 분에게 미안하다.)

아주 작은 토이 프로젝트를 제외하곤 상태관리 라이브러리를 도입하길 추천한다.

### [Redux/Toolkit](https://redux-toolkit.js.org/tutorials/typescript#application-usage)

이런 고민을 나만 하는건 아닌지 redux/toolkit이 등장했다. 사가에서 action, actionTypes, reducer, saga로 나뉘어서 적용되는 불편함을 Slice라는 새로운 개념으로 통일 시켰다.

아래는 아주 단순화 시킨 Slice의 예이다.

```jsx
// slice 선언
const testSlice = createSlice({
  name: 'testSlice',
  initialState: testAdapter.getInitialState(),
  reducer: {
    testEvent(state, action: PayloadAction<TestState>) {
      // ... update state
    }
  }
});

// 액션의 선언
export const testAction = testSlice.action;
```

```jsx
import { testAction } from './testSlice'
// 실제 액션의 사용
function Test() {
  const dispatch = Redux.useDispatch();
  dispatch(testAction.testEvent('some data'));
}
```

상태에 대한 접근 및 갱신의 통일성을 위해 [Entity Adapter](https://redux-toolkit.js.org/api/createEntityAdapter#createentityadapter)를 사용, 데이터의 정규화([Normalize](https://redux.js.org/recipes/structuring-reducers/normalizing-state-shape))를 수행했다. 초기 접근 비용은 높아지지만 Adapter에서 제공되는 여러 함수를 통해 불필요한 코드들이 많이 제거된다. 툴킷을 제대로 사용하기 위해선 필수적으로 판단됐다.

툴킷의 사용성을 높이기 위해 하나 더 필수적으로 사용해야되는 것이 있었는데, 바로 TypeScript다.

---

# 📝 TypeScript

에디터 모듈에서 다루는 박스의 종류가 다양하고, 내/외부에서 사용되는 데이터 규격을 정확히 맞추기 위해 [타입스크립트](https://www.typescriptlang.org/ko/docs/handbook/react.html)를 선택했다. 컴포넌트나 스토어에서 사용되는 매우 다양한 데이터의 형태를 지정하고 규격을 맞추는 과정에서 많은 오류 발생 가능성이 줄어든다. 

결과적으로 타입 스크립트는 좋은 선택이었다. 인터페이스(type) enum을 통해 컴포넌트에 주입되는 데이터를 정확하게 관리할 수 있었다. 다만 프로젝트에서 사용된 몇 몇 패키지가 type을 제대로 지원하지 않아 사용하기 힘들었다. 리덕스 에서 액션 타입을 지정해서 사용할 때나 상속을 받아 사용하는 과정에서 불편한 점이 있었다.(이는 내 경험 부족으로 초기 설계를 잘못 한 점이라 생각된다.)

## Enum

[enum](https://www.typescriptlang.org/ko/docs/handbook/enums.html)은 형태를 지정한 자료를 열거한 형태로 사용 될 자료를 미리 나열해서 사용할 수 있다. 다양한 형태로 사용할 수 있지만, 이번 컴포넌트 전략에서 가장 중요한 부분은 테마를 이용한 컴포넌트의 제한된 사용이었다. 

따라서 enum을 통해 컴포넌트의 테마 등을 정의하고 범위 내에서 컴포넌트 옵션을 사용하도록 했다. 

컴포넌트 외에 리덕스에서 사용되는 데이터에도 enum을 부분적으로 사용하고, 강제 했다.

## Type(interface)

사용전에 미리 타입을 알 수 있다는 건 사용자 입장에선 축복이다.

enum이 컴포넌트의 데이터의 폭을 제한 했다면, type의 지정으로 컴포넌트에 전달되는 props의 데이터 형태를 제어해서 사용했다. React에서 제공되는 PropTypes와 비슷하지만, TSC(타입스크립트 컴파일러)가 개발시에 아예 에러로 체크해주는 점이 가장 매력적이다. 이 에러 체크 때문에 초기에는 조금 힘들지만(esLint와 함께 엄청 괴롭힌다.) 결국 어딘가에서 거쳐야 할 과정이니 부지런히 타입 지정을 해야 한다.

다른 장점들이 많지만 가장 골치아픈 에러를 미리 예방할 수 있다는 점이 타입스크립트를 쓰는 최고의 강점 아닐까

---

# 📖 Storybook

스토리북은 개발된 컴포넌트를 시각화 할 수 있는 도구다. 단순 시각화 외에 Props를 바로 조정해서 결과를 확인할 수 있고, 데코레이터를 통해 필요한 기능들을 덧붙일 수 있다. 이제 FE개발을 제대로 하는 곳이면 필수적으로 사용하고 있으며, 퍼블리셔에게 개발코드를 활용한 스타일링도 가능케 한다.

장점이 많고 많지만 개발자의 피부로 와닿는 명확한 장점은 두가지다.

## 시각화

필요한 컴포넌트들만 바로 확인할 수 있다. 스타일을 바로 적용하여 시각적인 테스트도 가능하고, 실제 주입되는 데이터의 형태를 바꿔가며 데이터가 컴포넌트에 미치는 영향을 바로 확인할 수 있다.

자동 제작된 문서를 통해서 해당 컴포넌트가 어떤 Props를 지니고 있는지, 어떤 방식으로 호출하는지 쉽게 확인할 수 있다.

추가로 스토리북 빌드를 통해 html페이지 자체를 만들 수 있다. netlify나 s3등에 올려 웹서비스를 할 수 있고, 단순히 압축 전달하는 것만으로도 필요한 대상에게 디자인 및 동작여부를 확인하게 할 수 있다.

동작하는 방식은 [이 링크](https://next--storybookjs.netlify.app/official-storybook/?path=/story/addons-a11y-basebutton--default)를 통해 확인하자.

## 개발 편의성

페이지 프로세스를 거치지 않고 원하는 컴포넌트를 바로 확인할 수 있다. 현재 개발중인 페이지가 특정 프로세스의 5번째 페이지라고 한다면, 테스트를 위해 1,2,3,4번 페이지를 거치는 일이 잦을것이다.

혹은 특정 데이터를 넣어야 동작하는 기능이라고 한다면 매번 데이터를 입력해야 하는 번거로움도 있다. 개발하며 항상 인풋에 ㅁㄴㅇㄹ를 넣는 모습을 생각해보면 금방 이해가 간다. 스토리북을 통해 이 불편한 과정들을 많이 건너뛰거나 테스트 과정으로 바꿀 수 있다.

---

# 정리하며

회고라 하고 프로젝트에 대한 설명만 늘어논 것 같다. 힘들기도 했고 아쉬웠던 부분이 많았던 만큼 하고싶은 말이 많았다.  

## 아쉬운 점

테스트를 제대로 하지 못했다. 아쉬운점은 항상 변명으로 시작한다.

초기에 간단한 테스트를 구현해 두었지만, 중간에 요구사항이 급 범람하며 설계 된 부분을 많이 손댔고, 그때마다 구현해둔 테스트 모듈이 걸림돌로 작용했다. 일정에 쫓겨 개발하다보니 테스트는 뒷전이었고, 갈수록 복잡해지는 구조를 풀어내느라 급급했다.

돌이켜보면 항상 그랬다. 테스트 초반에는 항상 불필요한 일처럼 여겨지지만 지나고 보면 테스트만 제대로 했어도 막을 수 있는 일들이 많다.

## 좋았던 점

오랫동안 고민하던 마크업의 코드화, 퍼블리셔의 개발자화에 큰 진전이 있었다.

Theme로 명명한 컴포넌트의 구조 강제화, Emotion-Styled 컴포넌트를 통해 css를 제대로 제어할 수 있었다.

또한 개발을 제대로 이해 못하고 있는 퍼블리셔를 개발자로 바꾸는데 성공하며, 그동안 실패하고 고민했던 내가 가지고 있던 개발 가이드에 대해 많은 확인을 얻기도 했다.

### 지나고 보니

개발은 항상 그랬다. 지나고 나면 아쉬움이 남고, 오히려 조금 더 아는 지금의 아쉬움이 더 큰 것 같다.

지나고 나면 후회와 아쉬움이 먼저 다가오지만, 제대로 노력했다면 뒤이어 오는 결실을 통해 한 걸음 더 나아갈 수 있다. 인생도 그렇다. 

회고를 쓰며 나는 조금 더 나아갈 수 있는 에너지를 얻었다.

이 글을 보신 분들이 비슷한 에너지를 얻어갈 수 있길 바라며 글을 마무리 한다.

# 프로젝트 스펙

이 프로젝트에는 다음과 같은 주요 라이브러리들이 사용 되었다.

- [Typescript](https://www.typescriptlang.org/)
- [React](https://ko.reactjs.org/)
- [Redux Toolkit](https://redux-toolkit.js.org/)
- [Emotion / styled component](https://emotion.sh/docs/styled)
- [Storybook](https://storybook.js.org/)
- [Draft.js](https://draftjs.org/)