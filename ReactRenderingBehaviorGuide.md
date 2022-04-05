# React 렌더링 동작에 관한 (거의) 완벽한 가이드

 이 글은 Marks's Dev Blog의 [Blogged Answers: A (Mostly) Complete Guide to React Rendering Behavior](https://blog.isquaredsoftware.com/2020/05/blogged-answers-a-mostly-complete-guide-to-react-rendering-behavior/#standard-render-behavior)를 번역한 글입니다.
 (옮긴이가 첫 번역인데다가 미숙한 영어와 markdown실력을 겸비했으니, 수정사항은 언제나 환영입니다..ㅎ)

___

 React가 컴포넌트를 언제, 왜 재렌더링 하게 되는 것인지, Context와 React-Redux의 사용이 재랜더링 타이밍과 범위에 어떻게 영향을 끼치는지 이해하는 건 참 복잡합니다.
 
 이 글을 수십번 고쳐써서, 이제야 사람들에게 추천할 만 하다 싶은 통합 가이드라인이 되었습니다.

 이 글의 모든 정보는 이미 온라인에서 찾을 수 있는 내용들이며, 수많은 훌륭한 블로그 글들이나 기사에서 찾아볼 수 있습니다. 그 중 몇몇은, 그런 글들의 마지막 "추가 정보" 란에 링크되어 있을겁니다. 그럼에도, 참 많은 사람들이 정보를 여기저기서 부분부분 모아 이해하려고 애씁니다. 

이 글이 누군가에겐, 명확한 이해를 도와주길 바랍니다.

## 목차 
 
 - [렌더링이란 무엇인가?](#렌더링이란-무엇인가)
   - [렌더링 과정 몰아보기](#렌더링-과정-몰아보기)
   - [Render Phase와 Commit Phase](#render-phase와-commit-phase)
 - [React는 Render를 어떻게 다룰까?](#react는-render를-어떻게-다룰까)
   - [Render 대기열에 담기](#render-대기열에-담기)
   - [기초적인 Render 동작](#기초적인-render-동작)
   - [React 렌더링 규칙](#react-렌더링-규칙)
   - [컴포넌트 메타데이터와 Fibers](#컴포넌트-메타데이터와-fibers)
   - [컴포넌트 타입과 재조정](#컴포넌트-타입과-재조정)
   - [Key와 재조정](#key와-재조정)
   - Render 일괄 처리 및 타이밍
   - Render 동작의 예외
 - 렌더링 성능 향상시키기
   - 컴포넌트 Render 최적화 기법
   - 새로운 props 참조가 Render 최적화에 끼치는 영향
   - Props 참조 최정화하기
   - 모든 것을 메모이제이션하고 계신가요?
   - 불변성과 렌더링
   - React 컴포넌트 렌더링 성능 측정하기
 - Context와 렌더링 동작
	- Context 기초
	- Context 값 업데이트하기
	- State 업데이트, Context, 그리고 재렌더링
	- Context 없데이트와 Render 최적화
 - React-Redux와 렌더링 동작
    - React-Redux의 구독
    - `connect`와 `useSelector`의 차이
  - 요약
  - 마무리
  - 추가 정보


# 렌더링이란 무엇인가?
**렌더링**은  여러분의 모든 컴포넌트에게 현재 state와 props값의 조합을 기반으로 **각각의 UI를 어떻게 화면에 띄우고 싶어하는지 설명**하도록 물어보는 React의 프로세스입니다.


## 렌더링 과정 몰아보기
렌더링 과정동안, React는 컴포넌트 트리의 최상단으로부터 업데이트가 필요하다고 표시된 모든 컴포넌트를 찾아 아래로 내려옵니다. React는 각각의 표시된 컴포넌트에게 `classComponentInstance.render()`(class 컴포넌트의 경우) 또는`FunctionComponent()`(function 컴포넌트의 경우) 중 하나를 호출하고, render 결과물을 저장합니다.

컴포넌트의 render 결과물은 기본적으로 JSX문법으로 작성되어있고, `React.createElement()`로 전환되어, 배포될 땐 JS로 컴파일됩니다. `createElement`는 **의도한 UI 구조를 설명하는 JS 일반 객체**인 *React elements*를 반환합니다.

예시:
```js
// JSX
return <SomeComponent a={42} b="testing">Text here</SomeComponent>

// 위 JSX 문법은 아래처럼 전환됩니다.
return React.createElement(SomeComponent, {a: 42, b: "testing"}, "Text here")

// 그리고 element 객체가 됩니다.
{type: SomeComponent, props: {a:42, b:"testing"}, children: ["Text here"]}
```
이러한 render 결과물을 모든 컴포넌트 트리로부터 모았다면, React는 새로운 객체들의 트리(흔히 말하는 "가상 DOM")와 비교합니다. 그리고 실제 DOM이 해당 render 결과물처럼 변하기 위해 바뀌어야하는 변경점들의 리스트를 만듭니다. 이렇게 비교하고 계산하는 과정이 "[재조정](https://reactjs.org/docs/reconciliation.html)"입니다.

그러고 나서, React는 단 한번의 동기적인 과정으로 DOM을 바꾸기 위해 계산된 모든 변경점들을 적용시킵니다.

> **_NOTE:_**  React 팀은 최근 몇 년간 "가상 DOM"이라는 표현을 경시했습니다. _[Dan Abramov(Redux 개발자)의 트윗](https://twitter.com/dan_abramov/status/1066328666341294080?lang=en)_
>
> 저는 "가상 DOM"이라는 용어를 버리길 바랍니다. 2013년에는 말이 되는 용어였습니다. 이걸 쓰지 않으면 사람들은 React가 모든 render마다 DOM노드를 만들 것이라고 믿었기 때문이죠. 그런데 요즘 사람들은 이렇게 생각하지 않습니다. "가상 DOM"은  DOM문제의 해결책으로 들릴 수 있겠지만, React는 그렇지 않습니다.
> React는 "value UI(의역: 값으로 UI를 만드는 방식)"입니다. 핵심 원리는, **UI는 그저 문자열이나 배열로 이루어진 값**이라는 점입니다. **값**이기에, 변수에 저장하고, 전달하고, JS의 제어 흐름을 따라갈 수 있습니다. 이 표현이 정말 중요한 요점입니다. 약간의 DOM 변경을 피하기 위해 비교하는 과정이 아닙니다. 
> React는 언제나 DOM을 표현하는 것이 아닙니다. 예를 들면, `<Message recipientId={10} />` 은 DOM이 아닙니다. 개념적으로, `Message.bind(null, {recipientId: 10})` 이라는 function을 추후에 호출할 뿐입니다.


## Render Phase와 Commit Phase
React팀은 이를 두 단계로 구분합니다.
 - "Render Phase"는 컴포넌트를 렌더링하고, 필요한 변화를 계산하는 작업을 모두 포함합니다.
 - "Commit Phase"는 해당 변화를 DOM에 적용시키는 과정입니다.

React가 Commit Phase에 DOM을 업데이트한 뒤엔, 바뀐 DOM 노드와 컴포넌트 인스턴스를 가리키도록 모든 ref를 업데이트합니다. 그 후에 동기적으로 class 라이프사이클에선 `componentDidMount`, `componentDidUpdate`를 실행하고, function의 경우 `useLayoutEffect`를 실행합니다.

React는 짧은 제한시간을 정하고, 이 시간이 지나면, 모든 `useEffect` hook을 실행시킵니다. 이 과정은 "Passive Effect" phase라고도 불립니다.

이 class 라이프사이클의 시각화를 [훌륭한 React lifecycle method diagram](https://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/)에서 확인할 수 있습니다. (아직 effect hook을 보여주진 않지만, 추가되면 좋겠네요)

> 곧 [출시될 Concurrent Mode](https://17.reactjs.org/docs/concurrent-mode-intro.html) (역자: React 18버전에 추가되었죠! 아직 공식 문서엔 구 문서만 남아있고, 업데이트가 안되었습니다.) 에서는, 렌더링 페이즈 중 일시 정지할 수 있습니다! 브라우저가 이벤트를 처리할 수 있도록 말이죠. 그 후  React는 추후 적절하게 작업을 재개하거나, 버리거나, 다시 계산할 수 있습니다. 만약 render 과정이 완료되어 넘어가면, React는 여전히 한번에 동기적으로 commit phase를 실행합니다. 

가장 중요한 한 가지를 이해하고 갑시다. **"렌더링"은 "DOM을 업데이트하는 것"과 동일하지 않고, 컴포넌트는 결과적으로 눈에 보이는 어떠한 변화가 발생하지 않고도 렌더링 될 수 있습니다.**
React가 컴포넌트를 렌더링 할 때:
 - 컴포넌트는 이전과 같은 render 결과물을 반환할 수 있고, 그렇기에 아무런 변화도 필요없을 수도 있습니다.
 - Concurrent Mode에서는, React는 최종적으로 컴포넌트를 수 차례 렌더링하지만, 동시에 수행되는 다른 업데이트가  현재의 작업을 무효화시킨다면, render 결과물을 버릴 수 있습니다.


# React는 Render를 어떻게 다룰까?

## Render 대기열에 담기
초기 render가 완료된 후에, React에게 re-render를 대기열에 담으라고 요청하는 몇 가지 방식들이 있습니다.
 - Class 컴포넌트
	 - `this.setState()`
	 - `this.forceUpdate()`
 - Function 컴포넌트
	 - `useState`의 setter
	 - `useReducer`의 dispatch
 - 그 외
	 - `ReactDOM.render(<App>)`을 다시 호출할 때 (루트 컴포넌트에서 `forceUpdate()`를 호출하는 것과 동일)


## 기초적인 Render 동작
이것을 이해하는 것은 정말로 중요합니다:

React는 기본적으로 **부모 컴포넌트가 렌더링되면, React는 모든 내부의 자식 컴포넌트를 재귀적으로 렌더링하도록** 동작한다는 점입니다.

예를 들어, `A > B > C > D` 로 내려가는 컴포넌트 트리가 있다고 해보죠, 그리고 이미 페이지에 띄운 상태입니다. 여기서 유저가 `B`의 버튼을 클릭해 카운터를 +1 시킵니다.
 - 우리는 `B`에서 `setState()`를 호출했으므로, `B`의 재렌더링을 대기열에 담습니다.
 - React는 트리의 꼭대기부터 render하기 시작합니다.
 - React는 `A`를 확인하고, 업데이트가 필요하다고 표시되지 않은 것을 확인해서, 넘어갑니다.
 - React는 `B`를 확인하고, 업데이트가 필요하다고 표시된 것을 확인했기에, render합니다. `B`는 최초 렌더링때와 같이 `<C/>`를 반환합니다.
 - `C`는 업데이트가 필요하다고 표시되어있지 **않습니다**.  그렇지만, 부모 컴포넌트인 `B`가 렌더링되었기에, React는 `C`또한 밑으로 내려오며 렌더링합니다. `C`는 `<D/>`를 반환합니다.
 - `D` 또한 표시되어있지 않지만, 부모인 `C`가 렌더링되었기에 React는 내려오며 렌더링합니다.

이는 이렇게도 표현할 수 있습니다:
**한 컴포넌트를 렌더링하는 것은 기본적으로, 내부의 모든 컴포넌트를 렌더링하게 될 것입니다.**

그리고 또 다른 요점은:
**기본적인 렌더링 과정에서, React는 "props의 변화"를 확인하지 않습니다. - 부모 컴포넌트가 렌더링되었기에 무조건적으로 자식 컴포넌트가 렌더링되는 것입니다.**

이 말은, 루트의 `<App>`컴포넌트에서 `setState()`를 호출하는 것은 다른 어떠한 동작상 변화가 없더라도, React는 컴포넌트 트리에 포함되는 개별 컴포넌트를 전부 재렌더링하게 만든다는 것입니다. 결국, React의 가장 기초 판매 전략중 하나는 ["모든 업데이트마다 전체 앱을 다시 그리는 것처럼 행동하기"](https://www.slideshare.net/floydophone/react-preso-v2)였던 겁니다.

이제, 컴포넌트 트리에 있는 대부분의 컴포넌트들은 이전과 똑같은 render 결과물을 나타낼 가능성이 매우 높고, 그렇기에 React는 DOM에 어떠한 변화도 줄 필요가 없습니다. 그렇지만, React는 여전히 모든 컴포넌트에게 렌더링을 요청하고 render 결과물과의 차이점을 확인하도록 물어야만 합니다. 두 작업 전부 시간과 자원이 필요한데도 말이죠.

기억하세요, **렌더링은 나쁜 것이 아닙니다. 렌더링은 React가 실제 DOM의 변화를 일으켜야 할 지 안할지 알아내는 방법입니다.**


## React 렌더링 규칙
React의 주요 규칙 중 하나는 **렌더링은 "순수"해야하고, 어떠한 사이드 이펙트도 일으키면 안된다는 것입니다!** 이는 까다롭고 혼동될 만한 내용인데, 수많은 사이드 이펙트는 명확하지 못하고, 결과적으로 아무 것도 막지 않기 때문입니다. 예를 들어, 엄밀히 말하면 `console.log()`는 사이드 이펙트입니다만, 아무 것도 막지 않습니다. prop을 변경하는 것은 사이드 이펙트고, 아무 것도 막지 않을 *수도*있습니다. AJAX(비동기 서버 요청)을 하는 것은 분명 사이드 이펙트면서, request의 종류에 따라 예상하지 못했던 애플리케이션의 동작을 분명히 야기할 수 있습니다.

Sebastian Markbage는 [The Rules of React](https://gist.github.com/sebmarkbage/75f0838967cd003cd7f9ab938eb1958f)라는 제목의 훌륭한 문서를 작성했습니다. 여기서 필자는 서로 다른 React의 라이프사이클에서, `render`와 같은 예상된 동작을 정의하고, "순수"하다고 고려할 만한 표현들이 어떤 것인지, 어떤 것이 안전하지 못한지 정의했습니다. 이 글은 전부 읽을 가치가 있습니다만, 키 포인트만 좀 요약해보자면:

 - Render 로직이 해선 안될 것
   - 존재하는 변수나 객체를 변화시킬 수 없습니다.
   - `Math.random()`이나`Date.now()`와 같은 랜덤한 값을 만들 수 없습니다.
   - 네트워크 요청을 만들어선 안됩니다.
   - state 업데이트를 요청해서도 안됩니다.
 - Render 로직이 해도 되는 것
   - 렌더링 도중 만들어진 새로운 객체를 변화시키는 것
   - Error Throw
   - 캐싱된 값 처럼 아직 만들어지지 않은 데이터를 "지연 초기화"하는 것


## 컴포넌트 메타데이터와 Fibers
React는 애플리케이션에 존재하는 모든 현재 컴포넌트 인스턴스를 추적하는 내부 데이터 구조를 저장합니다. 이 데이터 구조의 가장 핵심 부분은 "Fiber"라 불리는 객체입니다. Fiber는 아래 내용들을 표현하는 메타데이터를 포함합니다:

 - 컴포넌트 트리에서, 이 시점에서 렌더링되어야 하는 컴포넌트의 type
 - 현재 컴포넌트와 연관된 props와 state
 - 부모 컴포넌트, 형제 컴포넌트, 자식 컴포넌트를 향한 포인터
 - React가 렌더링 과정에서 추적하기 위해 사용하는 다른 내부 메타데이터

[여기 React 17에서 `Fiber` 객체를 확인할 수 있습니다.](https://github.com/facebook/react/blob/v17.0.0/packages/react-reconciler/src/ReactFiber.new.js#L47-L174)

렌더링 과정동안, React는 새로운 렌더링 결과를 계산할 때, 트리의 Fiber 객체를 반복해서확인함으로써 업데이트된 트리를 구성합니다.

**"Fiber" 객체들은 *실제* 컴포넌트의 props와 state값을 저장합니다.**
여러분이 컴포넌트의 props와 state를 사용하면, React는 실제로 Fiber 객체에 저장되어진 값에 접근시킵니다. 특히 class 컴포넌트에선, [React는 명시적으로 컴포넌트를 렌더링하기 직전에 `componentInstance.props = newProps`를 복사합니다.](https://github.com/facebook/react/blob/v17.0.0/packages/react-reconciler/src/ReactFiberClassComponent.new.js#L1038-L1042) 그렇기에`this.props`는 존재하나, 오직 React가 Fiber 객체의 내부에서 참조를 복사했기에 존재합니다. 그런 의미에서 컴포넌트는 Fiber 객체의 일종의 외관일 뿐입니다.

비슷하게, function 컴포넌트에서 hook은 [React가 연결된 모든 hook을 연결 리스트로 만들어, 컴포넌트의 Fiber 객체에 저장하기 때문에](https://www.swyx.io/hooks/) 작동할 수 있습니다. React가 function 컴포넌트를 렌더링하면, Fiber 객체로부터 연결된 hook 관련 리스트를 가져오고, hook을 호출할 때마다, [해당 hook 객체에 저장되어있는 적절한 값(state나 useReducer의 dispatch 같은)을 반환합니다.](https://github.com/facebook/react/blob/v17.0.0/packages/react-reconciler/src/ReactFiberHooks.new.js#L795)

부모 컴포넌트가 주어진 자식 컴포넌트를 최초로 렌더링할때, React는 컴포넌트의 "인스턴스"를 추적하기 위해 Fiber 객체를 생성합니다. class 컴포넌트에선, [`const instance = new YourComponentType(props)` 를 그대로 호출하고,](https://github.com/facebook/react/blob/v17.0.0/packages/react-reconciler/src/ReactFiberClassComponent.new.js#L653) 실제 컴포넌트 인스턴스를 Fiber 객체에 저장합니다. [React는 단지 `YourComponentType(props)`를 함수로써 호출할 뿐입니다.](https://github.com/facebook/react/blob/v17.0.0/packages/react-reconciler/src/ReactFiberHooks.new.js#L405)


## 컴포넌트 타입과 재조정
["재조정" 공식 문서](https://ko.reactjs.org/docs/reconciliation.html)에서 설명하듯, React는 재렌더링 과정에서 현재 DOM구조를 가능한 만큼 재사용하는 방식을 통해 효율성을 증대시킵니다. React에게 같은 타입의 컴포넌트나 HTML 노드를 같은 위치에 렌더링하도록 시키면, React는 DOM을 새로 만들기보단, 적절하다면 업데이트를 함으로써 기존 DOM을 재사용할것입니다. 이 말은 즉, React는 컴포넌트의 인스턴스를 같은 위치에서 렌더링하는 한, 최대한 살려둔다는 것입니다.  class 컴포넌트에선, 실제로 같은 컴포넌트 인스턴스를 사용합니다. function 컴포넌트에선, class처럼 실제 "인스턴스"는 존재하지 않지만, `<MyFunctionComponent />`가 컴포넌트의 "인스턴스"처럼 표현되어서 최대한 사라지지 않고 유지됩니다. 

그래서, React는 언제, 그리고 어떻게 render 출력 결과가 변경되었음을 알아차릴까요?

React의 렌더링 로직은 요소의 `===`참조 비교를 통해 `type`필드를 먼저 비교합니다. 만약 요소가 `<div>`에서 `<span>`이나, `<ComponentA>` 에서  `<ComponentB>`로 변한 것과 같이 새로운 타입으로 변경되었다면,  React는 모든 트리가 변경된다고 추정해, 비교 과정의 속도를 올립니다. 결과적으로, **React는 존재하는 모든 DOM 노드를 포함해, 컴포넌트 트리를 전부 지웁니다.** 그리고 컴포넌트 인스턴스를 처음부터 새로 만들어냅니다. 

이는 **렌더링중에 새로운 타입의 컴포넌트를 절대 만들면 안된다**는 뜻입니다. 새로운 컴포넌트 타입을 생성할 때마다, 새로운 참조이므로, React는 반복적으로 하위 컴포넌트들을 없애고 다시 만들겁니다.

다르게 말해서 이러면 **안됩니다.**:
```js
function ParentComponent() {

  // 여기서 새로운 `ChildComponent`의 레퍼런스가 렌더링마다 생성됩니다.
  function ChildComponent() {}
  
  return <ChildComponent />
}
```
대신, 컴포넌트를 분리해서 정의하는 것은 괜찮습니다:
```js
// 여기서 'ChildComponent' 타입 레퍼런스가 한 번 생성됩니다.
function ChildComponent() {}
  
function ParentComponent() {
  return <ChildComponent />
}
```


## Key와 재조정
React가 컴포넌트의 "인스턴스를" `key`라는 pseudo-prop(역: 가상의 prop)을 통해서도 식별합니다. `key`는 React의 지침이며, 실제 컴포넌트에겐 절대 전달되지 않습니다. React는 `key`를 컴포넌트 타입의 특정한 인스턴스를 구별지을때 사용하는 유일한 식별자로 간주합니다. 

`key`는 주로 리스트를 렌더링할때 사용합니다. `key`는 렌더링하는 과정에서 리스트에 데이터 추가, 제거, 수정하는 등 바뀔 수 있을 때 특히나 중요합니다. 

---
작성중