# TypeScript 치트시트 번역본
> 타입스크립트를 공부하면서 치트시트로 공부하는 속성공부가 있는데
역시나 영어로 되어있다. 조금은 영어실력을 늘리기 위해 한번 번역해 보곘다.

## 클래스

### 키포인트
>TypeScript 클래스는 ES2015 자바스크립트 클래스에 대한 몇 가지 유형별 확장 기능과 하나 또는 두 개의 런타임 추가 기능을 가지고 있습니다.

* 클래스 객체 생성하기
```tsx
class ABC{...}
const abc = new ABC()
```
new ABC에 대한 `parameter`는 생성자 함수로부터 온다.

* private X vs #private

private X
형식 전용 추가이며 런타임에는 적용되지 않습니다. 
다음 경우 클래스 외부의 코드가 항목에 도달할 수 있습니다.
```tsx
class Bag{
    private item : any
}
```  
#private
#private는 런타임 비공개이며 Javascript 엔진 내부에 클래스 내에서만 액세스할 수 있는 강제성을 가지고 있습니다.
```tsx
class Bag{ #item: any }
```
클래스 안에서의 `this`

함수 내부의 'this' 값은 함수를 호출하는 방법에 따라 달라집니다.  
다른 언어에서 익숙할 수 있는 클래스 인스턴스가 항상 보장되는 것은 아닙니다.

'this parameter'를 사용하거나, 바인딩 함수 또는 화살표 함수를 사용하여 문제가 발생할 때 해결할 수 있습니다.

* Type과 Value
  
클래스를 `type` 또는 `value`으로 모두 사용할 수 있습니다.
```tsx
    const a:Bag = new Bag()
    //여기서 a:Bag로 사용된 Bag은 type
    //뒤에 new Bag으로 사용된 Bag은 value로 사용되었다. 
    /*
        주의 사항!
    */
   class C implement Bag{} // 이와 같이 사용하는 것은 지양!
```

## 기본 문법

```tsx
    //User클래스에 하위클래스 Account       
    //클래스가 인터페이스 또는 유형 집합을 준수하는지 확인합니다.
    class User extends Accout implements Updatable, Serializable{
        id: string;               //field: 클래스에서 사용이 가능한 필수 공용 속성 
        displayName?: boolean;    //optional field: 클래스에서 사용이 가능한 선택 공용 속성  
        name!: string;            //identify field(존재 주장 field): 따로 정의하지 않아도 괜찮은 공용속성
        #attributes: Map<any, any>;//private field
        roles=["user"];            //field 기본값 선언
        readonly createAt = new Date()//readonly 속성 + 기본값 선언
        
        constructor(id: string, email: string){//코드가 new 함수를 호출한다.
            super(id);
            this.email = email; 
            //In strict : true 이 코드가 올바르게 설정되었는지 확인하기 위해 field에 대해 검사됩니다.
        }

        setName(name: string) { this.name = name }
        verifyName = (name: string) =>  Promuse<{...}>
        //클래스 메서드를 설명하는 방법(및 화살표 함수 fieds)

        sync(): Promise<{...}>
        sync(cb:((result: string) => void)): void
        sync(cb?:((result: string) => void)): void |  Promise<{...}>  Promise<{...}>
        //overload 정의가 두 개 있는 함수

        get accountID(){}
        set accountID(value: string) {}
        //getter. setter

        private makeRequest() { ... }
        protected handleRequest() { ... }
        //이 클래스는 private 접근만 허용되고 하위 클래스에 대한 protected가 됩니다. 
        //유형 확인에만 사용되며, public이 기본값입니다.
        
        static #userCount = 0;
        static registerUser(user: User) { ... }//static fields와 methods

        static { this.#userCount = -1 }
        //Static blocks는 static var로 세팅된다. this는 클래스를 참고한다.
    }
    
```
Generics
TypeScript는 클래스 메서드에서 변경할 수 있는 유형을 선언합니다.

```tsx
class Box<Type>{//클래스 타입 파라미터
    contents: Type
    constructor(value: Type) {
        this.contents = value;
    }
    const stringBox = new Box(" a package")
}

```

Generics 기능은 현재 구문으로는 JavaScript에 들어가지 못할 수 있는 TypeScript 특정 언어 확장입니다.

Parameter Properties

TypeScript의 확장된 기능으로 class들에서의 지정된 객체 field를 자동적으로 input parameter로 설정하는 기능입니다.
```tsx
class Location {
    constructor(public x: number, public y: number) {}
}
const loc = new Location(20, 40);
loc.x //20;
loc.y //40;
```