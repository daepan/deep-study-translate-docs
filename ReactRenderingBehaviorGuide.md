# React 렌더링 동작에 관한 (거의) 완벽한 가이드

>이 글은 Marks's Dev Blog의 [Blogged Answers: A (Mostly) Complete Guide to React Rendering Behavior](https://blog.isquaredsoftware.com/2020/05/blogged-answers-a-mostly-complete-guide-to-react-rendering-behavior)를 번역한 글입니다.
>
>*(역자가 첫 번역인지라 미숙할 수 있음에 유의해주세요.)*

___

 React가 컴포넌트를 언제, 왜 재렌더링하게 되는 것인지, Context와 React-Redux의 사용이 재렌더링의 타이밍과 범위에 어떻게 영향을 끼치는지 이해하는 건 참 복잡합니다.
 
 이 글을 수십번 고쳐써서, 이제야 사람들에게 추천할만하다 싶은 통합 가이드라인이 되었습니다.

 이 글의 모든 정보는 이미 온라인에서 찾을 수 있는 내용들이며, 수많은 훌륭한 블로그 글들이나 기사에서 찾아볼 수 있습니다. 그 중 몇몇은, 그런 글들의 마지막 "추가 정보"란에 링크되어 있을 겁니다. 그럼에도, 참 많은 사람들이 여기저기서 부분부분 정보를 모아 이해하려고 애씁니다. 

이 글이 누군가에겐, 명확한 이해를 도울 수 있길 바랍니다.

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
   - [Render 일괄 처리 및 타이밍](#render-일괄-처리-및-타이밍)
   - [Render 동작의 예외](#render-동작의-예외)
 - [렌더링 성능 향상시키기](#렌더링-성능-향상시키기)
   - [컴포넌트 Render 최적화 기법](#컴포넌트-render-최적화-기법)
   - [새로운 props 레퍼런스가 Render 최적화에 끼치는 영향](#새로운-props-레퍼런스가-render-최적화에-끼치는-영향)
   - [Props 레퍼런스 최적화하기](#props-레퍼런스-최적화하기)
   - [전부 메모이제이션해야할까요?](#전부-메모이제이션해야할까요)
   - [불변성과 렌더링](#불변성과-렌더링)
   - [React 컴포넌트 렌더링 성능 측정하기](#react-컴포넌트-렌더링-성능-측정하기)
 - [Context와 렌더링 동작](#context와-렌더링-동작)
	- [Context 기초](#context-기초)
	- [Context 값 업데이트하기](#context-값-업데이트하기)
	- [State 업데이트, Context, 그리고 재렌더링](#state-업데이트-context-그리고-재렌더링)
	- [Context 업데이트와 Render 최적화](#context-업데이트와-render-최적화)
 - [React-Redux와 렌더링 동작](#react-redux와-렌더링-동작)
    - [React-Redux의 구독](#react-redux의-구독)
    - [`connect`와 `useSelector`의 차이](#connect와-useselector의-차이)
  - [요약](#요약)
  - [마무리](#마무리)
  - [추가 정보](#추가-정보)


# 렌더링이란 무엇인가?
**렌더링**은  여러분의 모든 컴포넌트에게 현재 state와 props값의 조합을 기반으로 **각각의 UI를 어떻게 화면에 띄우고 싶어하는지 설명**하도록 물어보는 React의 프로세스입니다.


## 렌더링 과정 몰아보기
렌더링 과정동안, React는 컴포넌트 트리의 최상단으로부터 업데이트가 필요하다고 표시된 모든 컴포넌트를 찾아 아래로 내려옵니다. React는 각각의 표시된 컴포넌트에게 `classComponentInstance.render()`(class 컴포넌트) 또는`FunctionComponent()`(function 컴포넌트) 중 하나를 호출하고, render 결과물을 저장합니다.

컴포넌트의 render 결과물은 기본적으로 JSX문법으로 작성되어있고, `React.createElement()`로 전환되어, 배포될 땐 JS로 컴파일됩니다. `createElement`는 **의도한 UI 구조를 설명하는 JS 일반 객체**인 *React elements*를 반환합니다.

예시:
```js
// JSX
return <SomeComponent a={42} b="testing">Text here</SomeComponent>

// 위 JSX 문법은 아래처럼 전환됩니다.
return React.createElement(
  SomeComponent, 
  {a: 42, b: "testing"},
  "Text here"
)

// 그리고 element 객체가 됩니다.
{
  type: SomeComponent, 
  props: {a:42, b:"testing"}, 
  children: ["Text here"]
}
```
이러한 render 결과물(ReactElement)을 모든 컴포넌트 트리로부터 모았다면, React는 새로운 객체들의 트리(흔히 말하는 "가상 DOM")와 비교합니다. 그리고 실제 DOM이 해당 render 결과물처럼 변하기 위해 바뀌어야 하는 변경점들의 리스트를 만듭니다. 이렇게 비교하고 계산하는 과정이 "[재조정](https://reactjs.org/docs/reconciliation.html)"입니다.

그러고 나서, React는 단 한번의 동기적인 과정으로 DOM을 바꾸기 위해 계산된 모든 변경점들을 적용시킵니다.

> **_NOTE:_**  React 팀은 최근 몇 년간 "가상 DOM"이라는 표현을 경시했습니다. _[Dan Abramov(Redux 개발자)의 트윗](https://twitter.com/dan_abramov/status/1066328666341294080?lang=en)_
>
> 저는 "가상 DOM"이라는 용어를 버리길 바랍니다. 2013년에는 말이 되는 용어였습니다. 이걸 쓰지 않으면 사람들은 React가 모든 render마다 DOM노드를 만들 것이라고 믿었기 때문이죠. 그런데 요즘 사람들은 이렇게 생각하지 않습니다. "가상 DOM"은  DOM문제의 해결책으로 들릴 수 있겠지만, React는 그렇지 않습니다.
> React는 "value UI *(의역: 값으로 UI를 만드는 방식)*"입니다. 핵심 원리는, **UI는 그저 문자열이나 배열로 이루어진 값**이라는 점입니다. **값**이기에, 변수에 저장하고, 전달하고, JS의 제어 흐름을 따라갈 수 있습니다. 이 표현이 정말 중요한 요점입니다. 약간의 DOM 변경을 피하기 위해 비교하는 과정이 아닙니다. 
> React는 언제나 DOM을 표현하는 것이 아닙니다. 예를 들면, `<Message recipientId={10} />` 은 DOM이 아닙니다. 개념적으로, `Message.bind(null, {recipientId: 10})` 이라는 function을 추후에 호출할 뿐입니다.


## Render Phase와 Commit Phase
React팀은 이를 두 단계로 구분합니다.
 - "Render Phase"는 컴포넌트를 렌더링하고, 필요한 변화를 계산하는 작업을 모두 포함합니다.
 - "Commit Phase"는 해당 변화를 DOM에 적용시키는 과정입니다.

React가 Commit Phase에 DOM을 업데이트한 뒤엔, 바뀐 DOM 노드와 컴포넌트 인스턴스를 가리키도록 모든 ref를 업데이트합니다. 그 후에 동기적으로 class 라이프사이클에선 `componentDidMount`, `componentDidUpdate`를 실행하고, function의 경우 `useLayoutEffect`를 실행합니다.

React는 짧은 제한시간을 정하고, 이 시간이 지나면, 모든 `useEffect` hook을 실행시킵니다. 이 과정은 "Passive Effect" phase라고도 불립니다.

이 class 라이프사이클의 시각화를 [훌륭한 React lifecycle method diagram](https://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/)에서 확인할 수 있습니다. (아직 effect hook을 보여주진 않지만, 추가되면 좋겠네요)

> 곧 [출시될 Concurrent Mode](https://17.reactjs.org/docs/concurrent-mode-intro.html) *(역자: React 18버전에 추가되었죠! 아직 공식 문서엔 구 문서만 남아있고, 업데이트가 안되었습니다.)* 에서는, 렌더링 페이즈 중 일시 정지할 수 있습니다! 브라우저가 이벤트를 처리할 수 있도록 말이죠. 그 후  React는 추후 적절하게 작업을 재개하거나, 버리거나, 다시 계산할 수 있습니다. 만약 render 과정이 완료되어 넘어가면, React는 여전히 한번에 동기적으로 commit phase를 실행합니다. 

가장 중요한 한 가지를 이해하고 갑시다. **"렌더링"은 "DOM을 업데이트하는 것"과 동일하지 않고, 컴포넌트는 결과적으로 눈에 보이는 어떠한 변화가 발생하지 않고도 렌더링 될 수 있습니다.**
React가 컴포넌트를 렌더링 할 때:
 - 컴포넌트는 이전과 같은 render 결과물을 반환할 수 있기에, 아무런 변화가 필요없을 수도 있습니다.
 - Concurrent Mode에서는, React는 최종적으로 컴포넌트를 수차례 렌더링하지만, 동시에 수행되는 다른 업데이트가 현재의 작업을 무효화시킨다면, render 결과물을 버릴 수 있습니다.


# React는 Render를 어떻게 다룰까?

## Render 대기열에 담기
최초 렌더링이 완료된 후에, React에게 re-render를 대기열에 담으라고 요청하는 몇 가지 방식들이 있습니다.
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

이제, 컴포넌트 트리에 있는 대부분의 컴포넌트들은 이전과 똑같은 render 결과물을 나타낼 가능성이 매우 높고, 그렇기에 React는 DOM에 어떠한 변화도 줄 필요가 없습니다. 그렇지만, React는 여전히 모든 컴포넌트에게 렌더링을 요청하고 render 결과물과의 차이점을 확인해야 합니다. 두 작업 전부 시간과 자원이 필요한데도 말이죠.

기억하세요, **렌더링은 나쁜 것이 아닙니다. 렌더링은 React가 실제 DOM의 변화를 일으켜야 할 지 안할지 알아내는 방법입니다.**


## React 렌더링 규칙
React의 주요 규칙 중 하나는 **렌더링은 "순수"해야하고, 어떠한 사이드 이펙트도 일으키면 안된다는 것입니다!** 이는 까다롭고 혼동될만한 내용인데, 수많은 사이드 이펙트는 명확하지 못하고, 결과적으로 아무 것도 막지 않기 때문입니다. 예를 들어, 엄밀히 말하면 `console.log()`는 사이드 이펙트입니다만, 아무 것도 막지 않습니다. prop을 변경하는 것은 사이드 이펙트고, 아무 것도 막지 않을 *수도* 있습니다. AJAX(비동기 서버 요청)을 하는 것은 분명 사이드 이펙트면서, request의 종류에 따라 예상하지 못했던 애플리케이션의 동작을 분명히 야기할 수 있습니다.

Sebastian Markbage는 [The Rules of React](https://gist.github.com/sebmarkbage/75f0838967cd003cd7f9ab938eb1958f)라는 제목의 훌륭한 문서를 작성했습니다. 여기서 필자는 서로 다른 React의 라이프사이클에서, `render`와 같은 예상된 동작을 정의하고, "순수"하다고 고려할 만한 표현들이 어떤 것인지, 어떤 것이 안전하지 못한지 정의했습니다. 이 글은 전부 읽을 가치가 있습니다만, 키 포인트만 좀 요약해보자면 아래와 같습니다:

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

렌더링 과정동안, React는 새로운 렌더링 결과를 계산할 때, 트리의 Fiber 객체를 반복적으로 확인함으로써 업데이트된 트리를 구성합니다.

**"Fiber" 객체들은 *실제* 컴포넌트의 props와 state값을 저장합니다.**
여러분이 컴포넌트의 props와 state를 사용하면, React는 실제로 Fiber 객체에 저장되어진 값에 접근시킵니다. 특히 class 컴포넌트에선, [React는 명시적으로 컴포넌트를 렌더링하기 직전에 `componentInstance.props = newProps`를 복사합니다.](https://github.com/facebook/react/blob/v17.0.0/packages/react-reconciler/src/ReactFiberClassComponent.new.js#L1038-L1042) 그렇기에`this.props`는 존재하나, 오직 React가 Fiber 객체의 내부에서 레퍼런스를 복사했기에 존재합니다. 그런 의미에서 컴포넌트는 Fiber 객체의 일종의 외관일 뿐입니다.

비슷하게, function 컴포넌트에서 hook은 [React가 연결된 모든 hook을 연결 리스트로 만들어, 컴포넌트의 Fiber 객체에 저장하기 때문에](https://www.swyx.io/hooks/) 작동할 수 있습니다. React가 function 컴포넌트를 렌더링하면, Fiber 객체로부터 연결된 hook 관련 리스트를 가져오고, hook을 호출할 때마다, [해당 hook 객체에 저장되어있는 적절한 값(state나 useReducer의 dispatch 같은)을 반환합니다.](https://github.com/facebook/react/blob/v17.0.0/packages/react-reconciler/src/ReactFiberHooks.new.js#L795)

부모 컴포넌트가 주어진 자식 컴포넌트를 최초로 렌더링할때, React는 컴포넌트의 "인스턴스"를 추적하기 위해 Fiber 객체를 생성합니다. class 컴포넌트에선, [`const instance = new YourComponentType(props)` 를 그대로 호출하고,](https://github.com/facebook/react/blob/v17.0.0/packages/react-reconciler/src/ReactFiberClassComponent.new.js#L653) 실제 컴포넌트 인스턴스를 Fiber 객체에 저장합니다. [React는 단지 `YourComponentType(props)`를 함수로써 호출할 뿐입니다.](https://github.com/facebook/react/blob/v17.0.0/packages/react-reconciler/src/ReactFiberHooks.new.js#L405)


## 컴포넌트 타입과 재조정
["재조정" 공식 문서](https://ko.reactjs.org/docs/reconciliation.html)에서 설명하듯, React는 재렌더링 과정에서 현재 DOM구조를 가능한 만큼 재사용하는 방식을 통해 효율성을 증대시킵니다. **React에게 같은 타입의 컴포넌트나 HTML 노드를 같은 위치에 렌더링하도록 하면, React는 DOM을 새로 만들기보단, 적절하다면 업데이트를 함으로써 기존 DOM을 재사용할것입니다.** 이 말은 즉, React는 컴포넌트의 인스턴스를 같은 위치에서 렌더링하는 한, 최대한 살려둔다는 것입니다.  class 컴포넌트에선, 실제로 같은 컴포넌트 인스턴스를 사용합니다. function 컴포넌트에선, class처럼 실제 "인스턴스"는 존재하지 않지만, `<MyFunctionComponent />`가 컴포넌트의 "인스턴스"처럼 표현되어서 최대한 사라지지 않고 유지됩니다. 

그래서, React는 언제, 그리고 어떻게 render 출력 결과가 변경되었음을 알아차릴까요?
> 역) 여기서 말하는 "render 출력 결과"는 렌더링 과정에서 컴포넌트가 반환한, JSX를 변환시켜 모아둔 fiber 객체들의 집합을 의미합니다.

React의 렌더링 로직은 요소의 `===` 레퍼런스 비교를 통해 `type`필드를 먼저 비교합니다. 만약 요소가 `<div>`에서 `<span>`이나, `<ComponentA>` 에서  `<ComponentB>`로 변한 것과 같이 새로운 타입으로 변경되었다면,  React는 모든 트리가 변경된다고 추정해, 비교 과정의 속도를 올립니다. 결과적으로, **React는 해당 위치부터 존재하는 모든 DOM 노드를 포함해, 컴포넌트 트리를 전부 지웁니다.** 그리고 컴포넌트 인스턴스를 처음부터 새로 만들어냅니다. 

이는 **렌더링중에 새로운 타입의 컴포넌트를 절대 만들면 안된다**는 뜻입니다. 새로운 컴포넌트 타입을 생성할 때마다, 새로운 레퍼런스이므로, React는 반복적으로 하위 컴포넌트들을 없애고 다시 만들겁니다.

다르게 말해서 이러면 **안됩니다**:
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
React가 컴포넌트의 "인스턴스를" `key`라는 pseudo-prop *(역: 가상의 prop)*을 통해서도 식별합니다. `key`는 React의 지침이며, 실제 컴포넌트에겐 절대 전달되지 않습니다. React는 `key`를 컴포넌트 타입의 특정한 인스턴스를 구별지을때 사용하는 유일한 식별자로 간주합니다. 

`key`는 주로 리스트를 렌더링할때 사용합니다. `key`는 렌더링하는 과정에서 리스트에 데이터 추가, 제거, 수정하는 등 바뀔 수 있을 때 특히나 중요합니다.  **key는 가능한 모든 데이터 중 유일한 ID여야 한다는 점**은 특히나 중요합니다. **배열의 인덱스는 정말 최후의 수단으로 사용해야 합니다.**

위 문제가 발생하는 예시를 보여드리겠습니다. 10개의 `<TodoListItem>`을 렌더링했다고 치고, idx값을 key에 담았습니다. React는 `0...9`의 key로 이루어진 10개의 아이템을 확인합니다. 이제, 6번과 7번 아이템을 삭제합니다. 그러고 맨 뒤에 3개의 아이템을 추가하면, 결국 `0...10`의 key값을 가진 리스트가 렌더링됩니다. 그러므로, React는 제가 딱 1개의 아이템을 추가한 것처럼 이해할 겁니다. 10개였던 아이템이 11개가 되었기 때문입니다. React는 기분좋게 이미 존재하는 DOM 노드와 컴포넌트 인스턴스를 재사용할 겁니다. 하지만, 여기서 `<TodoListItem key={6}>`을 렌더링하면, 아마 `8`번째의 아이템을 불러올 겁니다. 그렇기에, 컴포넌트 인스턴스는 (지워졌어야 함에도) 아직 살아있고, 이전과 다른 객체를 데이터로 불러올 것입니다. 이게 작동하긴 할 테지만, 그러면서도 예상치 못한 동작을 불러올겁니다. 또한, React는 텍스트나 DOM 콘텐츠를 수정하려 할 때, 아이템들은 이전과 다른 데이터를 보여주어야 하기 때문에 여러 개의 아이템들을 업데이트할 것입니다. 아이템들이 바뀐 게 아니기 때문에 이러한 업데이트는 정말 쓸모없습니다.

대신 각각의 아이템마다 `key={todo.id}`를 담았다면, React는 삭제된 2개의 아이템과 추가된 3개의 아이템을 맞게 확인할 것입니다. React는 2개의 삭제된 아이템의 컴포넌트 인스턴스와 관련 DOM 요소들을 없애고, 3개의 새로운 아이템의 컴포넌트 인스턴스와 DOM요소를 생성합니다. 이러한 변경이 실제로 변화되지 않은 아이템들을 불필요하게 변경하는 것보다 훨씬 낫습니다.

Key는 list 말고도 컴포넌트 인스턴스 id에 유용합니다. **여러분은 `key`값을 id를 나타내는 데에 아무 React 컴포넌트에 아무 때나 추가할 수 있습니다. 해당 key를 변경하게 되면, React는 이전 컴포넌트 인스턴스를 버리고, 새로운 것을 만듭니다.** 이러한 일반적인 사용 예시는 리스트에서 현재 선택된 데이터의 정보를 나타내는 양식입니다. `<DetailForm key={selectedItem.id}>`를 렌더링하면, 선택된 아이템이 바뀌었을때, React는 양식을 없애고 다시 만들어서 내부의 상태 문제를 방지합니다.

## Render 일괄 처리 및 타이밍
기본적으로, 각각의 `setState()`호출은 React가 새로운 render를 진행하게 만들고, 동기적으로 실행하며, 반환하게 합니다. 그런데, React는 자동적으로 렌더링 일괄 처리의 방식으로 일종의 최적화도 자동으로 적용시킵니다. 렌더링 일괄 처리는 여러 가지의 `setState()`호출이 들어왔을 때 결과적으로 조금의 딜레이만으로 단 하나의 렌더링만 대기열에 담아 실행되도록 만들어줍니다. 

React 문서에선 ["state의 업데이트는 비동기적일 수 있다"](https://reactjs.org/docs/state-and-lifecycle.html#state-updates-may-be-asynchronous)라고 언급합니다. 이 말은 앞으로 설명할 렌더링 일괄 처리 동작과 연관되어있습니다.  부분적으로, React는 React의 이벤트 핸들러에서 발생하는 state 업데이트를 알아서 일괄 처리합니다. React 이벤트 핸들러는 전형적인 React 앱 코드상의 매우 큰 부분을 차지하기 때문에, 대부분의 state 변경은 실제로 일괄 처리됩니다.

React는 이벤트 핸들러를 위한 렌더링 일괄 처리를 `unstable_batchedUpdates`라고 알려진 내부 함수로 감싸서 구현합니다. React는 대기열에 담긴 모든 state 업데이트를 `unstable_batchedUpdates`가 실행되는 동안 추적하며, 후에 단일 render 과정으로 모든 변화 사항을 적용시킵니다. 이벤트 핸들러의 경우, React가 주어진 이벤트에 대해 어떤 핸들러를 호출해야 할 지 이미 정확히 알고있기 때문에 잘 작동합니다.

개념적으로, React가 내부적으로 하는 작업을 이러한 의사 코드로 상상할 수 있습니다:
```js
// 의사 코드는 실제가 아닙니다만, 약간의 아이디어 정도는 줄 수 있습니다.
function internalHandleEvent(e) {
  const userProvidedEventHandler = findEventHandler(e);
  
  let batchedUpdates = [];
  
  unstable_batchedUpdates( () => {
    // 여기에 대기중인 업데이트들은 `batchedUpdates = []`에 들어갑니다.
    userProvidedEventHandler(e);
  });
  
  renderWithQueuedStateUpdates(batchedUpdates);
}
```

그렇지만, 이는 외부의 실제 call stack에 대기중인 state 변화는 함께 일괄 처리되지 않는다는 의미이기도 합니다.

여기 구체적인 예시를 들어보죠.
```js
const [counter, setCounter] = useState(0);

const onClick = async () => {
  setCounter(0);
  setCounter(1);
  
  const data = await fetchSomeData();
  
  setCounter(2);
  setCounter(3);
}
```
이는 총 3번의 렌더링 과정을 실행합니다.  처음엔 `setCounter(0)`,`setCounter(1)`둘 다 일반적인 이벤트 핸들러 call stack에서 발생하기 때문에 동시에 일괄 처리되며, `unstable_batchedUpdates()` 호출 내부에서 발생합니다.

그러나, `setCounter(2)` 호출은 `await`이후에 발생합니다. 이는 기본적인 동기적 call stack이 끝난 뒤, 함수의 나머지 절반은 한참이 지난 후에야 완벽히 분리된 이벤트 루프 call stack에서 호출됨을 의미합니다. 그렇기 때문에, React는 `setCounter(2)` 직전까지 동기적으로 실행하고, `await`처리가 끝나면, `setCounter(2)`로 돌아옵니다.

그러면 `setCounter(3)`또한 원래의 이벤트 핸들러 외부에서 실행되는데다가, 일괄 처리 밖에서 처리되는 같은 일이 일어납니다.

commit phase 내부의 라이프사이클엔 `componentDidMount`, `componentDidUpdate`, `useLayoutEffect` 세가지의 추가적인 예외 케이스가 있습니다. 이들은 렌더링 과정을 지나고 추가적인 로직을 수행할 수 있게 해주며, 브라우저가 화면을 그리기 이전에 시행됩니다. 보통 사용하는 예시는,
 - 부분적이지만 불완전한 데이터를 포함하는 컴포넌트를 최초로 렌더링할 때
 - commit phase 라이프사이클에서, ref를 사용한 페이지에 존재하는 실제 DOM 노드의 진짜 사이즈를 측정해야 할 때.
 - 이러한 측정을 기반으로 하는 state를 정해야 할 때
 - 업데이트된 데이터로 즉시 화면을 재렌더링 할 때

이런 예시에서, 우리는 초기 "부분적으로" 렌더링되는 UI를 유저에게 보여주는 것보단, "최종적으로" 완성된 UI를 사용자들에게 보여주기를 원할 겁니다. 브라우저는 수정된 DOM의 구조를 다시 계산하고 만들지만, JS가 이벤트 루프를 막고 여전히 실행되고 있는 동안에는 실제 화면에는 아무것도 그려지지 않을 것입니다. `div.innerHTML = "a"; div.innerHTML = "b"`같은 코드에서 `"a"`는 절대 보이지 않는 것처럼 말이죠.

이러한 이유로, React는 언제나 commit phase 동안 동기적으로 렌더링을 실행할 겁니다. 그러면, 만약 "부분적 -> 최종적" 변경과 같은 업데이트를 수행한다면 오직 "최종적인" 내용만 화면에 표시됩니다.

마지막으로, 제가 아는 한, `useEffect` 내에서의 state 변화는 대기열에 들어가서, "Passive Effect" phase의 마지막에 모든 `useEffect` 콜백이 완료되는 순간 한번에 수행됩니다.

`unstable_batchedUpdates` API가 공개적으로 export되지만,  다음 내용은 주목할 가치가 있습니다:
 - 이름마다, "unstable"이라는 라벨이 붙어 React API의 공식적으로 지원되는 부분이 아닙니다.
 - 하지만, React 팀은 "이는 'unstable' API중 가장 안정적인 API이며, Facebook 코드의 절반은 이 함수에 의존합니다" 라고 말했습니다.
 - `react`에 배포된 나머지 핵심 API들과 달리, `unstable_batchedUpdates`는  `react` 패키지의 항목이 아닌, reconciler-specific(재조정-명세) API입니다. 대신, `react-dom`과 `react-native`에는 배포되어 있습니다. 이는 `react-three-fiber`나 `ink`같은 다른 재조정 관련 도구들도 `unstable_batchedUpdates` 함수를 export하지 않을 것이라는 점을 의미합니다.

React-Redux 7버전에서는, [`unstable_batchedUpdates`를 내부적으로 사용하기 시작했습니다.](https://blog.isquaredsoftware.com/2018/11/react-redux-history-implementation/#use-of-react-s-batched-updates-api) 그렇지만 React DOM과 React Native에 둘 다 적용하는 것은 조금 까다로운 세팅이 필요했습니다.(이용 가능한 패키지의 따라 효과적으로 부분만 가져오는 방식)

곧 나올 React의 Concurrent Mode에선, React는 *항상, 언제나, 어디서든지* 업데이트를 일괄 처리합니다.

## Render 동작의 예외

React는 **개발 중에 사용하는 `<React.StrictMode>` 태그의 내부에서 컴포넌트를 두 번 렌더링 할 것입니다.** 이는 당신이 렌더링 로직을 실행하는 횟수가 커밋되는 렌더 과정의 횟수와 같지 *않으며*, 렌더링 횟수를 계산하기 위해 `console.log()`에 의존할 수 없다는 것을 의미합니다. *(역 추가: [React 17부터 React는 자동으로 `console.log()` 같은 콘솔 메서드를 수정해서 생명주기 함수의 두 번째 호출에서 로그를 찍지 않습니다.](https://ko.reactjs.org/docs/strict-mode.html#detecting-unexpected-side-effects))* 대신, React DevTools Profiler를 사용해 전체적인 커밋된 렌더 횟수를 추적하거나, `useEffect` hook이나 `componentDidMount/Update` 라이프사이클에 로깅을 추가할 수 있습니다. 로그는 React가 완전히 렌더 과정을 완료하고 커밋되었을 때만 출력될 것입니다. 
> 역) 복습하건데, useEffect는 Commit Phase가 끝난 뒤 Passive Effect에 동작하기 때문이죠.

일반적으로, 실제 렌더링 로직에서 *절대* 상태 업데이트를 대기열에 담아선 안됩니다. 다른 말로, click이 발생할 때 `setSomeState()`를 호출하는 click callback을 만드는 것은 괜찮습니다만,`setSomeState()`를 실제 렌더링 동작의 한 파트로 호출해선 안됩니다.

하지만, 여기엔 한 가지 예외가 있습니다. **Function 컴포넌트는 렌더링 도중에 `setSomeState()`를 직접적으로 호출할 수 있으며**, 해당 컴포넌트가 렌더링될 때마다 실행하지 않습니다. 이는 [함수 컴포넌트가 클래스 컴포넌트의 `getDerivedStateFromProps`와 동일한 것](https://reactjs.org/docs/hooks-faq.html#how-do-i-implement-getderivedstatefromprops)처럼 동작합니다. 만약 함수 컴포넌트가 state 업데이트를 렌더링 중 대기열에 담았다면, React는 즉시 해당 state 업데이트를 적용하고, 계속 진행하기 전에 동기적으로 해당 컴포넌트를 재렌더링 합니다. 만약 컴포넌트가 state 업데이트를 무한히 대기열에 유지하고, 재렌더링하도록 강제된다면, React는 몇 번(지금은 50번)의 재도전을 한 후, 루프를 중단하고 에러를 발생시킵니다. 이 기술은 재렌더링 요청 없이 props 변경에 따라 state 값을 즉시 강제로 업데이트 할 때, +`useEffect` 내에서 `setSomeState()`를 호출할 때 사용됩니다. 

# 렌더링 성능 향상시키기

렌더링은 React의 작동 방법에서 일반적으로 예상되는 부분이지만, 때때로 "낭비"될 수 있다는 것도 사실입니다. 만약 컴포넌트의 렌더링 결과값(output)이 변하지 않는다면, 해당 부분의 DOM 또한 바뀔 필요가 없으므로, 해당 컴포넌트를 렌더링 하는 것 자체가 일종의 시간낭비입니다.

React 컴포넌트의 렌더링 결과물은 언제나 완전히 현재 props와 state를 기반으로*해야 합니다.* 그러므로, 만약 우리가 컴포넌트의 props와 state가 변경되지 않는다는 것을 미리 알고 있다면, 컴포넌트의 render 결과물 도 같을 것이고, 결국 컴포넌트의 변화도 필요 없기 때문에 결국 안전하게 렌더링 과정을 건너뛸 수 있을 겁니다.

일반적으로 소프트웨어의 성능을 향상시키려 할 때, 기본적으로 두가지 접근 방식이 있습니다.
 1. 같은 작업을 빠르게 하거나
 2. 작업을 덜 하거나

React 렌더링을 최적화하는 것은 주로 **2번**에 해당됩니다. 적절할 때, 컴포넌트의 렌더링을 건너뜀으로써 작업량을 줄이는 것이죠.

## 컴포넌트 Render 최적화 기법
React는 잠재적으로 컴포넌트의 렌더링을 건너뛸 수 있도록 세 가지 주요 API를 제공합니다.
 - [`React.Component.shouldComponentUpdate`](https://reactjs.org/docs/react-component.html#shouldcomponentupdate): 렌더링 과정 초기에 호출되는 선택적인 class component lifecycle method입니다. 만약 `false`를 반환하면, React는 해당 컴포넌트의 렌더링을 건너뜁니다. 아마 이 방식은 boolean 결과값을 계산하는 로직이 포함될 수 있지만, 그런 접근보다는 대부분 지난번 렌더링과 비교했을 때 props나 state의 변화가 있었는지 확인하고 변화가 없다면 `false`를 반환합니다.
 - [`React.PureComponent`](https://reactjs.org/docs/react-api.html#reactpurecomponent): props와 state의 비교가 가장 흔한 `shouldComponentUpdate`의 구현 방식이기에, `PureComponent` 클래스는 기본적으로 앞 방식(props와 state의 비교)을 구현하고, `Component` + `shouldComponentUpdate` 대신 사용될 수 있습니다.
 - [`React.memo()`](https://reactjs.org/docs/react-api.html#reactmemo): 내장된 ["고차 컴포넌트(High Order Component)"](https://ko.reactjs.org/docs/higher-order-components.html#gatsby-focus-wrapper) 타입입니다. 해당 컴포넌트를 인자로 삼아, 새로운 Wrapper 컴포넌트를 반환합니다. Wrapper 컴포넌트의 기본적인 동작은 props가 하나라도 바뀌었거나, 바뀌지 않은지 확인해서 재렌더링을 방지하는 것입니다. 함수형 컴포넌트와 클래스형 컴포넌트 둘 다 `React.memo()`를 사용해 감싸질 수 있습니다. (`React.memo()`는 두 번째 인자로 커스텀 비교 콜백을 담아 비교할 수 있습니다만, 이는 이전 props와 새로운 props만 비교할 수 있으므로, 이를 사용하는 주된 방법은 모든 props에 대해서 비교하는 것이 아닌, 특정 필드만 비교하는 것입니다.)

이러한 비교 기술을 이용한 모든 접근은 **"shallow equality(얕은 동등성)"** 이라고 불립니다. 이는 두개의 다른 객체의 모든 필드를 각각 검사하고, 서로 다른 값을 가진 내용물이 있는지 확인하는 방식입니다. 대충 `obj1.a === obj2.a && obj1.b === obj2.b && ..........` 이렇게 확인하는 과정입니다. JS엔진은 `===`연산을 아주 간단하게 비교하기 때문에, 일반적으로 빠릅니다. 그렇기에, 이러한 세가지 접근법은 `const shouldRender = !shallowEqual(newProps, prevProps)` 처럼 동작합니다.

여기엔 또한 덜 알려진 기술이 있는데, **만약 React 컴포넌트가 마지막으로 반환했던 것과 똑같은 요소의 레퍼런스를 렌더링 결과로 반환한다면, React는 특정 하위 요소들의 재렌더링을 생략합니다.** 이 기술은 두가지 구현 방식이 있습니다:
 - 만약 `props.children`을 출력에 포함한다면, 해당 요소는 컴포넌트의 state를 업데이트해도 동일한 레퍼런스를 가집니다.
 - 만약 `useMemo()`로 몇몇 요소를 감쌌다면, 의존성이 변경될 때 까지 동일한 레퍼런스를 가집니다.
예시:
```jsx
// `props.children` 요소는 state가 바뀌어도 재렌더링이 발생하지 않습니다.
function SomeProvider({children}) {
  const [counter, setCounter] = useState(0);

  return (
    <div>
      <button onClick={() => setCounter(counter + 1)}>Count: {counter}</button>
      <OtherChildComponent /> 
      {children}
    </div>
  )
}

function OptimizedParent() {
  const [counter1, setCounter1] = useState(0);
  const [counter2, setCounter2] = useState(0);

  const memoizedElement = useMemo(() => {
    // 이 요소는 counter2가 업데이트되어도 동일한 레퍼런스를 유지합니다.
    // 그렇기에 이 비용이 큰 컴포넌트는 counter1이 변하지 않는 이상 재렌더링되지 않습니다.
    return <ExpensiveChildComponent />
  }, [counter1]) ;

  return (
    <div>
      <button onClick={() => setCounter1(counter1 + 1)}>Counter 1: {counter1}</button>      
      <button onClick={() => setCounter1(counter2 + 1)}>Counter 2: {counter2}</button>
      {memoizedElement}
    </div>
  )
}
```
이러한 기술들 덕에, 컴포넌트의 렌더링을 생략하는 것은 React가 모든 하위 트리의 렌더링도 생략하는 것을 의미합니다. "자식 요소들을 재귀적으로 렌더링하는" 기본적인 동작을 멈추기 위한 정지 신호를 효과적으로 설정하기 때문이죠.

## 새로운 props 레퍼런스가 Render 최적화에 끼치는 영향
**React는 기본적으로 중첩된 모든 컴포넌트를, 컴포넌트들의 props가 변하지 않았더라도 재렌더링한다는 것**을 확인했습니다. 이것은 도한 새로운 **레퍼런스**가 props로 자식 컴포넌트에게 들어가는 것은 문제가 되지 않는다는 것을 의미합니다. **같은 props 값을 전달했는지는 상관 없이 그냥 렌더링하기 때문입니다.** 아래와 같은 코드는 전혀 문제가 없는 코드입니다: 
```jsx
function ParentComponent() {
    const onClick = () => {
      console.log("Button clicked")
    }
    
    const data = {a: 1, b: 2}
    
    return <NormalChildComponent onClick={onClick} data={data} />
}
```
> 역) 왜 문제가 없을까요? 최적화를 시도한 더 밑의 코드와 비교하면 이해할 수 있습니다.

`ParentComponent`가 렌더링될 때마다, 새로운 `onClick`함수 레퍼런스와 `data`객체 레퍼런스가 만들어져서, `NormalChildComponent`의 props로 들어갑니다. (`onClick`함수를 정의할 때 `function`키워드를 사용할지, 화살표 함수를 사용할지는 중요하지 않습니다 - 어느 쪽이던 새로운 함수 레퍼런스가 생깁니다.)

그리고 하위 요소가 없는`div`나 `button`같은 단일 컴포넌트를 `React.memo()`로 감싸서 최적화하는건 의미가 없습니다. 컴포넌트의 하위 요소가 없기 때문에, 결국 렌더링 프로세스는 마지막까지 도달해 중지됩니다.

그렇지만, **만약 자식 컴포넌트가 props가 변경되었는지를 확인해서 최적화를 하려고 한다면,  새로운 레퍼런스를 props로 담는 것이 자식 컴포넌트를 렌더링하게 합니다.** 새로운 prop의 레퍼런스가 실제로 새로운 데이터라면, 좋은 방법입니다. 그렇지만, 부모 컴포넌트가 콜백 함수를 전달해준다면 어떻게 될까요?
```jsx
const MemoizedChildComponent = React.memo(ChildComponent)

function ParentComponent() {
    const onClick = () => {
      console.log("Button clicked")
    }
    
    const data = {a: 1, b: 2}
    
    return <MemoizedChildComponent onClick={onClick} data={data} />
}
```
이제, `ParentComponent`가 렌더링되면, 새로운 레퍼런스들이 `MemoizedChildComponent`에게 들어가고, props 값이 새로운 레퍼런스로 변경된 것이 확인되어, 재렌더링이 발생하게 될 겁니다. `onClick` 함수와 `data`는 매 호출마다 *기본적으론* 동일함에도 불구하고 말이죠!

그러므로:
 - `MemoizedChildComponent`의 렌더링을 생략하고 싶었음에도, 언제나 재렌더링됩니다.
 - 이전 props와 새로운 props를 비교하는 작업은 그저 낭비입니다.

비슷하게, `<MemoizedChild> <OtherComponent/> </MemoizedChild>`처럼 렌더링하는 것 *또한* 자식 컴포넌트를 항상 렌더링하도록 만듭니다. `props.children`이 언제나 새로운 레퍼런스를 갖기 때문이죠.

## Props 레퍼런스 최적화하기
클래스 컴포넌트는 인스턴스의 메소드가 매번 같은 레퍼런스를 갖기 때문에, 의도치 않게 새로운 콜백함수 레퍼런스가 만들어지는 것에 대해 걱정할 필요가 적습니다. 그렇지만, 하위 리스트의 요소마다 특수한 콜백을 만들어줘야 할 때나, 익명 함수의 값을 저장하고 자식 요소에게 넘겨주어야 할 때가 있을겁니다. 그런 경우에는 새 참조가 생성되고, 자식 props로써 새로운 객체들이 렌더링 과정에 생성됩니다. React는 이러한 경우를 최적화하는 방법이 없습니다.

함수형 컴포넌트는, React는 같은 레퍼런스를 재사용할 수 있도록 두 가지 hook을 제공합니다. `useMemo`는 객체를 생성하거나 복잡한 연산같은 모든 종류의 일반적인 데이터에, `useCallback`은 특히 콜백 함수를 생성할 때 쓰입니다.
 
## 전부 메모이제이션해야할까요?
위에서 언급했듯, `useMemo`나 `useCallback`을 prop으로 담아주는 모든 단독 함수나 객체에 쓸 필요가 없습니다. 오직 자식 요소의 변화를 만드는 경우에만 사용합니다. (이 말은, `useEffect`의 의존성 배열이 또 다른 use case가 있는 경우인데, 자식 요소가 일관된 props 레퍼런스로 해당 배열을 받으려 한다면, 구조가 더 복잡해집니다.)
> (That said, the dependency array comparisons for `useEffect`  _do_ add another use case where the child might want to receive consistent props references, which does make things more complicated.)

"왜 React는 기본적으로 _모든_ 컴포넌트를 `React.memo()`에 담지 않는 것입니까?" 라는 질문은 매번 나옵니다.

Dan Abramov(Redux 개발자)는 [메모이제이션은 여전히 props를 비교하는 비용이 발생한다는 것](https://twitter.com/dan_abramov/status/1095661142477811717)과, 컴포넌트는 대부분 새로운 props를 받기 때문에 수많은 경우에 메모이제이션 과정이 절대 재렌더링을 방지할 수 없다고 반복적으로 지적해왔습니다. 예시로, [Dan의 트위터 쓰레드를 확인해주세요.](https://twitter.com/dan_abramov/status/1083897065263034368)
> _왜 React는 기본적으로 모든 컴포넌트를 memo()로 담지 않는것인가요? 그게 더 빠르지 않나요? 우리가 체크를 위한 벤치마크를 직접 만들어야 하나요?_
> _스스로 답해보세요:_
> _왜 모든 함수에 Lodash 의 memoize() 를 쓰지 않는거죠? 그게 더 빠르지 않나요?_

또한, 이것에 대한 특정한 링크는 없지만, 아마 모든 컴포넌트에 기본적으로 메모이제이션을 적용시킨다면, 사람들이 데이터를 새로 갱신하는 것이 아닌 변화시키는 경우에, 아마 버그가 발생할 것입니다.
> 역) mutating rather than updating it immutably 라는데, 전자가 값의 변화, 후자는 레퍼런스의 변화로 해석했습니다.

저는 Twitter에서 Dan과 이 트윗과 관련된 몇몇 토론을 했습니다. 저는 개인적으로 `React.memo()`를 광범위하게 사용하는 것이 전반적인 렌더링 성능의 향상으로 이루어질 것이라고 보았습니다. [작년에 트위터에서 한 말:](https://twitter.com/acemarke/status/1141755698948165632)
> React 커뮤니티는 "성능"에 지나치게 집착하는 것 같지만, 대부분의 토론이 Medium이나 Twitter 게시물을 기반의 오래된 "구식 지혜"를 중심으로 진행됩니다. 구체적인 사용법을 기반으로 하지 않고 말이죠.
> "렌더링"과 성능의 영향이라는 주제엔 명백히 집단적인 오해가 있습니다. React는 전부 다 렌더링하고, 렌더링은 아무 일도 일으키지 않을 수 있습니다. 대부분의 렌더링은 비용이 크지도 않습니다.
> 렌더링 "낭비"를 한다고 해서 뭐 망하고 그러는 게 아닙니다. root부터 앱 전체를 전부 렌더링하는 것도 아닙니다. DOM 업데이트를 포함하지 않는 "낭비된" 렌더링은 CPU 사이클을 끌어올리지 않는다는 점도 사실입니다. 대부분의 앱들이 문제가 있나요? 아마 아닐겁니다. 그걸 향상시킬 수 있나요? 아마도요.
>  "전부 다 렌더링하는" 기본적인 접근을 하는 앱들 중 충분하지 않은 앱이 있나요? 당연하죠, 그렇기에 shouldComponentUpdate, PureComponent, React.memo() 가 존재합니다.
>  사용자들이 기본적으로 모두 memo()에 담아야 하나요? 여러분의 앱의 성능 요구사항에 맞춰 생각해야하기 때문에 아마 아닐 수 있습니다. 그럼 앱에 피해를 줄까요? 아뇨, 아마 성능의 순 이익이 있을겁니다 (Dan의 비교 낭비 지적에도 불구하고 말이죠.)
>  벤치마크에 결함이 있고, 앱과 시나리오마다 결과가 다양하게 나옵니까? 맞아요. 확실히 이런 어려운 점들을 지적하고 이런 토론을 하는 것이 "제가 댓글에서 보기론..." 같은거보단 **정말 정말 훨씬** 낫습니다.
>  React팀과 큰 커뮤니티들의 벤치마크들을 보고 시나리오를 측정해서, 이 논쟁이 하루빨리 끝나길 기원합니다. 함수 생성, 렌더링 비용, 최적화... **정확한 증거, 제발!!**

하지만, [그 누구도 이게 맞았는지 틀렸는지 증명하는 벤치마크를 만들어오지 못했습니다](https://twitter.com/acemarke/status/1229083161646305280):
> Dan의 표준적인 답변은 앱마다 구조와 업데이트 패턴이 다양하기에, 대표적인 벤치마크를 만들기 힘들다는 점입니다.
> 저는 여전히 몇몇 수치가 토론에 도움이 될 것이라고 생각합니다.

React issue란에 ["언제 React.memo를 쓰면 **안**되나요?"](https://github.com/facebook/react/issues/14463) 에 대한 좀 더 확장된 토론이 있습니다.
(이 블로그 게시물은 해당 트윗 스레드의 오래 지연되고, 확장된 버전입니다.)

## 불변성과 렌더링
React에서 **State 변화는 언제나 불변적(immutably)으로 행해져야 합니다.** 여기엔 두 가지 주된 이유가 있습니다:
 - 무엇을 어디서 바꿨는지에 따라, 렌더링 될 것이라 예상했던 컴포넌트가 결과적으로 렌더링되지 않을 수 있습니다.
 - 언제, 왜 데이터가 실제로 업데이트되었는지에 혼란을 일으킵니다.

좀 더 구체적인 예시를 들어봅시다.

앞서 보았듯, `React.memo / PureComponent / shouldComponentUpdate`는 전부 현재 props와 이전 props의 얕은 동등성 검사에 의존합니다. 그러므로, 우리는 prop이 새로운 값인지 `props.someValue !== prevProps.someValue`를 행함으로써 알 수 있다고 예상합니다.

만약 변화시킨다면, `someValue`는 같은 레퍼런스를 가지고 있으므로, 컴포넌트는 변화가 없다고 가정합니다.
> 여기서 말하는 *변화(mutate)*는 state의 레퍼런스를 유지한 채 값을 갱신하는 것을 말합니다.
> ex) props.someValue = "newValue"

구체적으로 우리가 불필요한 재렌더링을 피함으로써 성능 최적화를 시도한다는 점에 집중하세요. 만약 props가 변하지 않았다면 렌더링은 "불필요"하거나 "낭비"입니다. 만약 변화시킨다면, 컴포넌트는 아무것도 변하지 않았다고 잘못 생각할 것이고, 여러분은 컴포넌트가 재렌더링되지 않는 것에 의문을 품게 될 겁니다.

또 다른 문제는, `useState`와 `useReducer` 훅입니다. `setCounter()`나 `dispatch()`함수를 호출할 때마다, React는 재렌더링을 대기열에 담습니다. 그렇지만, React는 어떤 hook이던 state의 변경은 반드시 새로운 state 값으로써 새로운 레퍼런스를 반환/전달해야합니다. 그것이 새로운 객체/배열의 레퍼런스던, 혹은 새로운 원시타입이던간에 말이죠. (string/number 등)
> 역) 실제로 useReducer의 dispatch action이나, setState로 원시타입인 경우 같은 값을, 객체인 경우 같은 레퍼런스를 담아보세요! 아무 변화도 일어나지 않습니다.

React는 모든 state의 변화를 render phase에 수용합니다. React가 hook으로부터 state변화를 수용하려 할 때, 새로운 값이 같은 레퍼런스인지 확인합니다. React는 _언제나_ 업데이트 대기열에 담아둔 컴포넌트의 렌더링을 완료합니다. 그렇지만, 만약 그 값이 이전과 같은 레퍼런스라면, 그리고 렌더링을 지속해야하는 또 다른 이유가 없더라면(부모 컴포넌트가 렌더링되었다던가 하는), React는 컴포넌트에 대한 렌더링 결과를 버리고, 렌더 과정에서 완전히 빠져나갑니다. 따라서, 이렇게 배열을 변화시키면:
```js
const [todos, setTodos] = useState(someTodosArray);

const onClick = () => {
  todos[3].completed = true;
  setTodos(todos);
}
```
이러면 컴포넌트는 재렌더링되지 않습니다.

새로운 레퍼런스를 만들어 기존 레퍼런스의 불변성을 지켜야합니다. 예시를 이렇게 바꿔볼 수 있습니다:
```js
const onClick = () => {
  const newTodos = todos.slice();
  newTodos[3].completed = true;
  setTodos(newTodos);
}
```
이러면 우리는 새로운 배열 레퍼런스를 만들어 담아주었고, 그렇기에 컴포넌트는 재렌더링 될 것입니다.

클래스형 컴포넌트의 `this.setState()`와 함수형 컴포넌트의 `useState`와 `useReducer` 훅 사이의 동작이 구분된다는 점에 주의해주세요. 
`this.setState()`는 값을 변화시켜도 상관하지 않습니다. 이는 호출될 때마다 재렌더링을 진행합니다. 그렇기에 아래 코드도 재렌더링이 발생합니다:
```js
const {todos} = this.state;
todos[3].completed = true;
this.setState({todos});
```
그래서 사실, `this.setState({})`처럼 빈 객체를 담아도 마찬가지입니다.

모든 실제 렌더링 동작을 넘어, 변화는 React의 단방향 데이터 흐름에 혼란을 줍니다. 변화는 전혀 바뀌지 않았다고 예상될 때도 다른 코드들이 값들을 확인하도록 만듭니다. 이는 실제로 업데이트를 하기로 되어 있는지, 어디서 변화가 발생한 것인지 파악하기 더 힘들게 만듭니다.

결론: **React나, 다른 React환경은 immutable한 업데이트를 가정합니다. 값을 변화시키면, 버그가 발생할 위험을 안고 가는 것이니, 그러지 마십시오.**

## React 컴포넌트 렌더링 성능 측정하기
[React DevTools Profiler](https://reactjs.org/blog/2018/09/10/introducing-the-react-profiler.html)를 사용해서 어떤 컴포넌트들이 렌더링되는지 확인해보세요. 예상과 다르게 렌더링되는 컴포넌트들을 찾고, DevTools를 이용해서 *왜* 그 컴포넌트들이 렌더링되었는지 확인하고, 수정하세요.(아마 `React.memo()`로  감싸거나, 부모 컴포넌트에서 내려주는 props값을 메모이제이션 하는 등의 방법으로 수정될겁니다.)

또한, React는 개발 빌드중에 정말 느리게 동작한다는 점을 기억하세요. 개발 모드에서 어떤 컴포넌트가 왜 렌더링되었는지 확인하고, 컴포넌트마다 렌더링하는데 소요되는 *상대적인* 시간을 측정하여 애플리케이션의 성능을 분석할 수 있습니다.("컴포넌트 B는 컴포넌트 A보다 렌더링을 마치기까지 시간이 3배나 걸리더라.." 처럼요.) 하지만, 개발 빌드중에 렌더링에 걸리는 절대적인 시간을 측정하는 것은 절대 안됩니다. - 절대적인 시간 측정은 프로덕션 빌드에서만! 실제로 개발모드의 프로파일러를 사용해 프로덕션 빌드와 유사한 시간 데이터를 측정하고 싶다면 [특별한 React의 "프로파일링" 빌드](https://kentcdodds.com/blog/profile-a-react-app-for-performance)를 사용해야 합니다.

# Context와 렌더링 동작
 React의 Context API는 컴포넌트의 하위트리에서 사용할 수 있는 단일 값을 만드는 메커니즘입니다. 주어진 `<MyContext.Provider>` 내부의 어떤 컴포넌트던, Context 인스턴스로부터 해당 값을 읽을 수 있습니다. prop을 명시적으로 매번 내려줄 필요 없이 말이죠.

Context는 "상태 관리" 도구가 아닙니다. Context에 전달된 state는 직접 관리해야합니다. Context는 명시적으로 데이터를 유지할 뿐이고, 그 데이터를 기반으로 context 값을 구성합니다.

## Context 기초
context의 provider는 `<MyContext.Provider value={42}>` 처럼  `value`라는 단일 prop을 받습니다. 자식 컴포넌트들은 context의 consumer 컴포넌트를 만들어 prop을 받아 값을 활용할 수 있습니다:
`<MyContext.Consumer>{(value) => <div>{value}</div>} <MyContext.Consumer/>`

함수형 컴포넌트라면 `useContext` 훅을 호출함으로써 쓸 수 있습니다:
`const value = useContext(MyContext)`

## Context 값 업데이트하기
React는 주변 컴포넌트가 provider를 렌더링할 때, provider가 새로운 값을 받았는지 확인합니다. 만약 provider의 value가 새로운 레퍼런스라면, React는 값이 바뀌었다고 인식하고, 해당 context를 사용하고 있는 컴포넌트들은 업데이트되어야 한다고 확인합니다.

새로운 객체를 provider에게 전달하는 것은, 업데이트를 유발할 것입니다:
```jsx
// const MyContext = React.createContext({}); 처럼 생성합니다!

function GrandchildComponent() {
    const value = useContext(MyContext);
    return <div>{value.a}</div>
}

function ChildComponent() {
    return <GrandchildComponent />
}

function ParentComponent() {
    const [a, setA] = useState(0);
    const [b, setB] = useState("text");

    const contextValue = {a, b};

    return (
      <MyContext.Provider value={contextValue}>
        <ChildComponent />
      </MyContext.Provider>
    )
}
```
이 예시에서, ParentComponent가 렌더링될 때마다, React는 `MyContext.Provider`에 새로운 값이 주어졌다고 확인하고, 자식 컴포넌트를 내려가며 `MyContext`를 소비중인 컴포넌트를 찾아갑니다. context provider가 새로운 값을 받으면, 해당 context를 소비중인 *모든* 중첩된 컴포넌트의 재렌더링을 강제합니다.

React의 관점에서, 모든 context provider는 단일 값을 가져야 합니다. 그것이 object던, array던, 원시 타입이던, 한가지 context value면 됩니다. 현재, 새로운 context 값의 갱신에 의한 이를 소비중인 컴포넌트들의 재렌더링이 발생하는 것을 건너뛰는 방법은 없습니다. 컴포넌트들이 해당 context의 *일부만* 사용한다고 하더라도 말이죠.

## State 업데이트, Context, 그리고 재렌더링
여기까지 알아본 내용들을 모아서 정리해보겠습니다:
 - `setState()`의 호출은, 컴포넌트의 렌더링을 대기열에 담는다.
 - React는 기본적으로, 중첩된 컴포넌트를 재귀적으로 렌더링한다.
 - Context의 Provider는 컴포넌트가 Provider를 렌더링함으로써 값을 부여받는다.
 - 그 값은 보통 부모 컴포넌트의 state다.

이는 기본적으로, 어떠한 context provider를 렌더링하는 부모 컴포넌트의 모든 state 업데이트는 어쨌든 자식 컴포넌트들의 제랜더링을 발생시킵니다. 확인한 context 값이 변경되었던, 그대로던 상관없이 말이죠.

다시 바로 위의 `Parent/Child/Grandchild` 예시를 보면, `GrandchildComponent`는 재렌더링될거라는 사실을 알 수 있는데, context 값이 바뀌기 때문이 아닙니다. - **그저 `ChildComponent`가 렌더링되었기 때문**입니다! 이 예시에서, 여기엔 "불필요한" 렌더링을 최적화하기 위한 어떠한 노력도 하지 않았고, 그렇기에 React는 `ChildComponent`와 `GrandchildComponent`를 `ParentComponent`가 렌더링될 때마다 렌더링합니다. 만약 부모 컴포넌트에서 `MyContext.Provider`에게 새로운 context값을 준다면, `GrandchildComponent`가 렌더링되고 그 값을 사용하려 할 때, 값이 바뀐 것을 확인할 것입니다. 하지만 context 값의 변경이 `GrandchildComponent`의 렌더링을 **발생시킨 것이 아닙니다.** - 어찌됐던 발생하긴 하지만요.

## Context 업데이트와 Render 최적화
이제 해당 예시를 바꿔서 최적화를 좀 해보겠습니다만, 조금 다르게 `GreatGrandchildComponent`를 추가해서 보여줄 것입니다.
```jsx
function GreatGrandchildComponent() {
  return <div>Hi</div>
}

function GrandchildComponent() {
    const value = useContext(MyContext);
    return (
      <div>
        {value.a}
        <GreatGrandchildComponent />
      </div>
}

function ChildComponent() {
    return <GrandchildComponent />
}

const MemoizedChildComponent = React.memo(ChildComponent);

function ParentComponent() {
    const [a, setA] = useState(0);
    const [b, setB] = useState("text");

    const contextValue = {a, b};

    return (
      <MyContext.Provider value={contextValue}>
        <MemoizedChildComponent />
      </MyContext.Provider>
    )
}
```
여기서, `setA(42)` 를 호출하게 된다면:
 - `ParentComponent`가 렌더링됩니다.
 - 새로운 `contextValue`의 레퍼런스가 생성됩니다.
 - React는 `MyContext.Provider`가 새로운 context값을 받았다고 확인하고, `MyContext`를 사용하고 있는 컴포넌트들은 업데이트되어야 한다고 확인합니다.
 - React는 `MemoizedChildComponent`를 렌더링하려고 시도하지만, `React.memo()`로 감싸져 있습니다. 컴포넌트에게 담아둔 props는 없으므로, props의 변화 또한 없기에 React는 `ChildComponent`의 렌더링을 전부 건너뜁니다.
 - 그렇지만, `MyContext.Provider`의 업데이트가 남아있는데, 더 하위 요소를 업데이트해야 할 수도 있습니다.
 - React는 `GrandchildComponent`에 도달할때까지 자식 요소로 내려갑니다. `GrandchildComponent`에서 `MyContext`를 확인하고는, 새로운 context값이 주어졌으니 해당 컴포넌트를 재렌더링합니다.
 - `GrandchildComponent`가 렌더링되었기때문에, React는 다시 내부 요소를 순회하며 렌더링을 진행합니다. 결론적으로, React는 `GreatGrandchildComponent`를 재렌더링합니다.

즉, [Sophie Alpert(React 엔지니어링 관리자)는 이렇게 말했습니다](https://twitter.com/sophiebits/status/1228942768543686656):
> Context Provider 바로 밑에 있는 React 컴포넌트에는 아마 `React.memo`를 써야 할 겁니다.

이렇게 하면, 부모 컴포넌트의 state 업데이트가 모든 컴포넌트를 재렌더링하지 않고, 해당 context를 사용하는 부분만 재렌더링할 것입니다. (혹은 기본적으로 `ParentComponent`를 `<MyContext.Provider>{props.children}</MyContext.Provider>`처럼 "같은 요소의 레퍼런스"를 사용하는 기술로 하위 요소의 재렌더링을 방지할 수 있습니다. `<ParentComponent><ChildComponent /></ParentComponent>`를 한 단계 위에서 호출합니다.)

그렇지만, `GrandchildComponent`가 새로운 context값에 의해 한 번 렌더링되고 나면, React는 다시 해당 요소부터 모든 자식컴포넌트들을 재귀적으로 다시 렌더링했습니다. 그래서 `GreatGrandchildComponent`가 렌더링되었고, 그 밑에 뭐가 있던 똑같이 렌더링되었을겁니다.

# React-Redux와 렌더링 동작
다양한 형태의 **"CONTEXT VS REDUX?!?!?!??"** 라는 질문은 지금도 React 커뮤니티에서 가장 많이 받는 질문이 아닐까 싶습니다. ([Context와 Redux는 전혀 다른 일을 하는 도구기 때문에, 이런 이분적인 질문 자체가 잘못된 것입니다.](https://blog.isquaredsoftware.com/2018/03/redux-not-dead-yet/))

즉 사람들이 반복적으로 지적하는 점은, "React-Redux는 실제로 렌더링이 필요한 곳만 렌더링해서, context보다 낫냐?"입니다.

이 말은 어느 정도 사실이지만, 그보다 훨씬 미묘한 차이가 있습니다.

## React-Redux의 구독
많은 사람들이 "React-Redux는 내부 구현에 context를 사용합니다." 라고 말하는 것을 보았습니다. 기술적으로 맞습니다만, [React-Redux는 context를 상태 값을 저장하는데 쓰는 것이 아니라, Redux store의 인스턴스를 유지하는 데 사용합니다.](https://blog.isquaredsoftware.com/2020/01/blogged-answers-react-redux-and-context-behavior/) 이는 언제나 같은 context 값을 `<ReactReduxContext.Provider>`에게 제공한다는 뜻입니다.

Redux의 store는 action이 dispatch될 때마다 구독자 알림 콜백을 실행한다는 것을 기억하세요. [Redux를 사용하는 UI계층은 언제나 Redux store를 구독하고, 구독자 알림 콜백으로부터 가장 최신 상태를 읽어서, 값을 비교하고, 관련된 데이터가 바뀌었다면, 재렌더링하게 합니다.](https://blog.isquaredsoftware.com/2018/11/react-redux-history-implementation/) 구독자 콜백 프로세스는 완전히 React *바깥에서* 동작하고, React는 오직 React-Redux가 특정한 React 컴포넌트에 필요한 데이터가 변경되었음을 아는 경우에만 관여합니다. (`mapState` 혹은 `useSelector`의 반환값을 기반으로 합니다)

이 결과는 context와 매우 다른 일련의 성능을 보입니다. 매번 더 적은 컴포넌트가 렌더링된다는 사실은 맞습니다만, React-Redux는 언제나 store state가 업데이트 될 때마다 전체 컴포넌트 트리를 위해 `mapState/useSelector` 함수를 실행해야 합니다. 대부분의 경우, 이러한 selector들을 실행하는 것이 React가 새로운 렌더링 과정을 진행하는 비용보다 싸기에, 일반적으로 이득을 봅니다만, 일단은 해야 하는 작업입니다. 그렇기에 selector 함수들을 실행하는 비용이 크거나, 필요하지 않을 때 새 값이 반환되면, 속도 저하를 일으키게 됩니다.

## `connect`와 `useSelector`의 차이
`connect`는 고차 컴포넌트입니다. 컴포넌트에게 담을 수 있고, `connect`는 store를 구독하고, `mapState`와 `mapDispatch` 를 실행하며, 결합된 props를 컴포넌트에게 내려주는 모든 일을 하는 wrapper 컴포넌트를 반환합니다.

`connect`의 wrapper컴포넌트는 언제나 `PureComponent/React.memo()`와 동일하게 작동합니다. 그렇지만, 조금 다른 중요한 점이 있는데: `connect`는 오직 결합된 props가 컴포넌트를 바꾸어야할 때만 렌더링을 진행합니다. 일반적으로, 최종적으로 결합된 props는 `{...ownProps, ...stateProps, ...dispatchProps}`의 결합이기에, 부모 컴포넌트의 모든 새로운 prop 레퍼런스가 컴포넌트를 렌더링하게 만듭니다. `PureComponent`나 `React.memo`처럼 말이죠. 부모 컴포넌트의 props 뿐만 아니라, [`mapState`로부터 받은 모든 새로운 레퍼런스도 컴포넌트를 렌더링하게 만듭니다.](https://react-redux.js.org/using-react-redux/connect-mapstate#mapstatetoprops-and-performance) (`ownProps/stateProps/dispatchProps`가 어떻게 병합되는지 커스터마이징해서 해당 동작을 바꿀 수 있습니다.)

반면에, `useSelector`는, 함수형 컴포넌트 내부에서 호출되는 hook입니다. 그렇기 때문에, `useSelector`는 부모 컴포넌트가 렌더링될 때 함께 렌더링되는 것을 막을 수 없습니다!

이는 [`connect`와 `useSelector`의 핵심적인 성능 차이입니다.](https://react-redux.js.org/api/hooks#performance) `connect`를 사용하면, 모든 연결된 컴포넌트가 `PureComponent`처럼 동작하고, React의 기본적인 전체 컴포넌트 트리를 위에서부터 내려오며 렌더하는 동작을 막아주는 방화벽입니다. 일반적인 React-Redux 애플리케이션은 다양한 컴포넌트들과 연결되어있는데, 이는 대부분의 재렌더링이 컴포넌트 트리의 꽤 작은 부분에 한정된다는 것을 의미합니다. React-Redux는 연결된 컴포넌트를 데이터 변화에 맞게 렌더링하도록 만들고, 그 밑에 있는 다음 두~세개의 컴포넌트도 렌더링할 수 있으며, 그러고 나서 React는 업데이트할 필요가 없고, 내려가는 과정을 중지시키는 또다른 연결된 컴포넌트를 실행합니다.
>역) React-Redux는 이런 불필요한 렌더링을 실행할 수 있다고 말하는 것일까요?

추가적으로 더 많은 연결된 컴포넌트를 갖는다는 것은 각각의 컴포넌트가 store로부터 아마도 더 작은 데이터 조각을 읽을 것이고, action이 주어진 이후에 앞으로 렌더링 할 가능성이 적다는 것을 의미합니다.

만약 함수형 컴포넌트와 `useSelector`를 독점적으로 사용중이라면, `connect`를 쓸 때보다 Redux store의 업데이트를 기반으로 더 큰 컴포넌트 트리의 재렌더링을 유발할 수 있습니다. 트리를 지속해서 내려오며 진행되는 렌더링을 멈춰줄 다른 연결된 컴포넌트들이 없기 때문이죠.

만약 성능이 문제된다면, 불필요한 재렌더링을 방지하기 위해 `React.memo()`로 컴포넌트들을 필요한 만큼 감싸세요. 

# 요약
 - React는 언제나 재귀적으로 렌더링을 진행하고, 기본적으로 부모 컴포넌트를 렌더링하면 모든 자식 컴포넌트도 렌더링합니다.
 - 렌더링 자체는 괜찮습니다 - DOM의 변화가 필요한 지 알아내는 방법일 뿐입니다.
 - 그렇지만, 렌더링도 시간이 소요되고, UI출력엔 아무런 변화가 없는 "낭비된 렌더링"이 발생할 수 있습니다.
 - 대부분의 경우 props로 콜백함수나 객체처럼 새로운 레퍼런스를 담아주어도 괜찮습니다.
 - `React.memo()`와 같은 API들을 활용해 props가 변하지 않았을 때의 불필요한 렌더링을 건너뛸 수 있습니다.
 - 하지만 `React.memo()`로 감싼 컴포넌트에게 새로운 레퍼런스를 props로 담아준다면, 렌더링을 절대 건너뛰지 못합니다. 그러니 넣어줄 값들의 메모이제이션도 필요합니다.
 - Context는 깊게 중첩된 소비자 컴포넌트가 값에 접근할 수 있도록 만들어줍니다.
 - Context Provider는 레퍼런스로 value의 변화를 확인합니다.
 - 새로운 Context value는 중첩된 모든 소비자들의 재렌더링을 유발합니다.
 - 그렇지만, 대부분의 자식 컴포넌트들은 어차피 부모->자식 간 발생하는 기본적인 재렌더링 과정때 렌더링이 됩니다.
 - 그러므로 context provider의 자식컴포넌트를 `React.memo()`로 묶거나 `{props.children}`을 사용해서 context value가 업데이트될 때 전체 트리가 갱신되는 것을 막을 수 있습니다.
 - 자식 컴포넌트가 새로운 context value에 의해 렌더링되면, React는 평소처럼 해당 자식 컴포넌트로부터 재귀적인 렌더링을 시행합니다.
 - React-Redux는 context별 value를 전달하는 대신, Redux store의 값이 갱신되는지를 확인하는 구독을 사용해서 업데이트를 확인합니다.
 - 이러한 구독은 Redux의 모든 store 업데이트에서 실행되므로, 가능한 한 빠르게 행해져야합니다.
 - React-Redux는 데이터가 변경된 컴포넌트만을 재렌더링할 수 있도록 많은 일을 합니다.
 - `connect`는 `React.memo()`처럼 동작하며, 연결된 컴포넌트들을 많이 가지면 컴포넌트가 한번에 렌더링되는 총 횟수를 줄일 수 있습니다.
 - `useSelector`는 hook이기에, 부모 컴포넌트의 재렌더링으로 인한 렌더링을 멈출 수 없습니다. 애플리케이션이 모든 곳에 오직 `useSelector`만으로 관리한다면, 아마 `React.memo()`로 몇몇 컴포넌트의 재렌더링을 방지해야할 겁니다.

# 마무리
분명히, 전체적인 상황은 "context는 모든 것을 렌더링하지만, Redux는 그렇지 않아요, Redux 쓰세요." 라는 말보다 훨씬 복잡합니다. 오해하지 마세요, 저는 사람들이 Redux를 썼으면 좋겠고, 또한 사람들이 다양한 도구와 관련된 동작과 절충안을 명확하게 이해하여 자신의 사용 사례에 가장 적합한 것이 무엇인지를 알고 정할 수 있기를 원합니다.

사람들이 매번 "언제 Context를 쓰고, 언제 (React-)Redux를 써야하나요?" 라고 물어봐서 몇가지 표준적인 규칙을 요약해봤습니다:

 - Context를 써야 할 때:
	 - 자주 바뀌지 않는 단순한 값을 전달해주어야 할 때
	 - 앱의 끝까지 내려가면서 몇몇 state나 함수를 props로 전달하고싶지 않을 때
	 - 추가 라이브러리가 싫고, React의 내장된 함수만을 사용하고 싶을 때

 - (React-)Redux를 써야 할 때:
	 - 애플리케이션의 다양한 곳에 너무 많은 state가 존재해야 할 때
	 - 앱의 상태가 시간에 따라 자주 업데이트될 때
	 - 상태를 업데이트하는 로직이 복잡할 때
	 - 애플리케이션이 중간 / 큰 사이즈의 코드베이스를 갖고 있고, 많은 사람들이 작업할 때

이것들이 어렵고 배타적인 규칙이 아닙니다 - 이 도구들이 의미가 있을 수 있는 가이드라인을 제안한 것 뿐입니다! 언제나, 시간을 좀 가지고 스스로 자신이 다룰 상황에 가장 좋은 도구가 무엇일 지 생각해서 결정해 보세요.

전반적으로, 이 설명이 사람들에게 다양한 상황에서 React의 렌더링 동작이 실제로 무슨 일이 일어나고 있는건지 더 큰 그림을 이해할 수 있도록 도움이 되기를 바랍니다.

# 추가 정보
-   **General**
    -   [Dave Ceddia: A Visual Guide to References in JavaScript](https://daveceddia.com/javascript-references/)
-   **React Render Behavior**
    -   [React docs: Reconciliation](https://reactjs.org/docs/reconciliation.html)
    -   [React lifecycle methods diagram](https://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/)
    -   [React hooks lifecycle diagram](https://github.com/donavon/hook-flow)
    -   [React issues: bailing out of context and hooks](https://github.com/facebook/react/issues/14110)
    -   [React issues: why  `setState`  is async](https://github.com/facebook/react/issues/11527#issuecomment-360199710)
    -   [Seb Markbage: "Context is good for low-frequency updates, not Flux-like state propagation"](https://github.com/facebook/react/issues/14110#issuecomment-448074060)
    -   [Ryan Florence: React, Inline Functions, and Performance](https://cdb.reacttraining.com/react-inline-functions-and-performance-bdff784f5578)
    -   [James K Nelson: React context and performance](https://frontarm.com/james-k-nelson/react-context-performance/)
-   **Optimizing Render Performance**
    -   [React docs: Optimizing Performance](https://reactjs.org/docs/optimizing-performance.html)
    -   [Kent C Dodds: Fix the slow render before you fix the re-render](https://kentcdodds.com/blog/fix-the-slow-render-before-you-fix-the-re-render)
    -   [Kent C Dodds: When to  `useMemo`  and  `useCallback`](https://kentcdodds.com/blog/usememo-and-usecallback)
    -   [Kent C Dodds: One simple trick to optimize React re-renders](https://kentcdodds.com/blog/optimize-react-re-renders)
    -   [React issues: When should you NOT use React.memo?](https://github.com/facebook/react/issues/14463)
-   **Profiling React Components**
    -   [React docs: Introducing the React DevTools Profiler](https://reactjs.org/blog/2018/09/10/introducing-the-react-profiler.html)
    -   [React DevTools profiler interactive tutorial](https://react-devtools-tutorial.now.sh/)
    -   [Kent C Dodds: Profile a React App for Performance](https://kentcdodds.com/blog/profile-a-react-app-for-performance)
    -   [Shawn Wang: Using the React DevTools Profiler to Diagnose React App Performance Issues](https://www.netlify.com/blog/2018/08/29/using-the-react-devtools-profiler-to-diagnose-react-app-performance-issues/)
    -   [Use the React Profiler for Performance](https://scotch.io/tutorials/use-the-react-profiler-for-performance)
-   **React-Redux Performance**
    -   [Practical Redux, Part 6: Connected Lists and Performance](https://blog.isquaredsoftware.com/2017/01/practical-redux-part-6-connected-lists-forms-and-performance/)
    -   [Idiomatic Redux: The History and Implementation of React-Redux](https://blog.isquaredsoftware.com/2018/11/react-redux-history-implementation/)
    -   [React-Redux docs:  `mapState`  Usage Guide - Performance](https://react-redux.js.org/using-react-redux/connect-mapstate#mapstatetoprops-and-performance)
    -   [High-Performance Redux](http://somebody32.github.io/high-performance-redux/)
    -   [React-Redux links: React/Redux Performance](https://github.com/markerikson/react-redux-links/blob/master/react-performance.md)