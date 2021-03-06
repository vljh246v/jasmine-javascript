# CHAPTER 4. 제너레이터

## 4.1. 완전-실행을 타파하다.

- 저자가 말하는 완전-실행이란. 

  자바스크립트 내에서 일단 함수가 시작하면 완료될 때까지 계속 실행되며 도중에 다른 코드가 끼어들어 실행되지 않는다.
              
- ES6 제너레이터는 이러한 완전-실행 법칙을 따르지 않는 새로운 종류의 함수로 등장한다.

```javascript
var x = 1;
function foo() {
  x++
  bar()
  console.log("x:",x)
};

function bar() {
  x++
};

foo();
```
 - 제너레이터를 이용하면 bar()가 없는데도 bar()를 실행할 수 있다.
 - 선점형 멀티스레드 언어의 Context Change와는 다른 의미이다.

 ```javascript
var x = 1;
function *foo() {
  x++
  yield()
  console.log("x:",x)
};

function bar() {
  x++
};

var it = foo()

it.next()
console.log(x) //2
bar()
console.log(x) //3
it.next()
```
 - foo() function이 실행되었지만 '완전-실행'되지 않고 yield 문에서 Stop
 - 제너레이터는 반드시 끝까지 실행하지 않아도 된다.
 - 이 책의 목표 : '비동기 흐름을 제어하는 제너레이터' 패턴 완성.


### 4.1.1 입력과 출력

- 함수 기본 체계인, 입출력이 존재한다. 인자를 받고 어떤 값을 반환한다.
- 일반 함수와 실행하는 방식이 다르다.
- 제너레이터 함수를 제어하는 '이터레이터' 객체를 만들어 실행해야한다.

```javascript
function *foo(x,y) {
  return x * y
};

var it = foo(6,7) // Input 6,7
var res = it.next() // Output 42
console.log(res.value) 
```
 - 제너레이터? 이터레이터?
 - 제너레이터 함수를 설계하면서 이를 제어할 것으로 Symbol.iterator로 설계
 - https://www.bsidesoft.com/5494

```javascript
//Generator
//Symbol.iterator
function *foo(){}
var it = foo()
```

 - 제너레이터는 단방향이 아닌 양방향 메시징이 가능하다.
 ```javascript
function *foo(x) {
  var y = x * (yield "hello");
  return y;
};

var it = foo(6) // x 6
var res = it.next() // yield hello Out
console.log(res.value)

res = it.next(7) // yield 7 In
console.log(res.value)
```
-> 마지막 it.next(7) yield문이 없으며 return으로 끝남.
   return이 없을 때는 return undefined가 생략됨.

### 4.1.2. 다중 이터레이터

- 이터레이터를 생성할 때마다 해당 이터레이터가 제어할 제너레이터의 인스턴스 역시 암시적으로 생성된다.

```javascript
function *foo(){}
var it1 = foo()
var it2 = foo()
```
- 인터리빙(interleaving) - 끼워넣기.
- 같은 변수를 공유한 상태에서 두 제너레이터의 이터레이터를 인터리빙 함으로써 이론적 스레드 경합 조건이 발생하는 환경을 실제로 재현가능하다.

```javascript
//study.js
var a = 1
var b = 2

function *foo() {
  a++
  yield
  b = b * a
  a = (yield b) + 3
}

function *bar() {
  b--
  yield
  a = (yield 8) + b
  b = a * (yield 2)
}

function step(gen) {
  var it = gen()
  var last
  //closure
  return function() {
    last = it.next(last).value
  }
}

var s1 = step(foo)
var s2 = step(bar)

s2() // b--;
s2() //           yield 8
s1() // a++
s2() // a = 8 + b yield 2
s1() // b = b * a yield b
s1() // a = b + 3
s2() // b = a * 2
```
-> Homework 1 
   왜 결과가 a=12,b=18 인가?
   
## 4.2. 값을 제너레이터링

- 제너레이터가 값을 만들어내는 용도로 쓸 수 있다 / 제너레이터 Name 기원
- 이터레이터와 제너레이터의 관계. / 제너레이터로 어떻게 값을 만들어내는지.

### 5.2.1. 제조기와 이터레이터

- 값을 생산하려고 할 때, 자신의 전 상태를 알아야하므로 상태성 생산기가 필요하다.

>Closure로 구현. 
```javascript
var gimmeSomething = (function() {
  var nextVal
  return function() {
    if(nextVal === undefined) {
      nextVal = 1
    }
    else {
      nextVal = (3 * nextVal) + 6
    }
    return  nextVal
  }
})()
gimmeSomething() // 1
gimmeSomething() // 9
gimmeSomething() // 33
gimmeSomething() // 105
```

> Symbol.iterator 구현.
> - Symbol.iterator는 ES6에서 사전 정의한 특수한 Symbol 값이다.
> - next()를 호출하면 프로퍼티가 2개인 객체가 반환됨.
> - done : Iterator 완료 상태 / value : 순회값.
> - ES6 / for-of Loop 표준 이터레이터를 자동으로 루프 구문 형태로 사용가능.
```javascript
var something = (function() {
  var nextVal
  return {
    // 'for-of' Loop <- Iterator로 Loop 순회함.
    [Symbol.iterator]: function(){return this},
    next: function() {
      if(nextVal === undefined) {
        nextVal = 1
      }
      else {
        nextVal = (3 * nextVal) + 6
      }
      return { done:false, value:nextVal}
    }
  }
})()

/*
something.next().value // 1
something.next().value // 9
something.next().value // 33
something.next().value // 105
*/

for(var v of something) {
  console.log(v)

  if(v>500) break
}
```
> ES6 Array.
> - ES6부터 배열 같은 자바스크립트 내장 자료 구조 대부분에는 기본 이터레이터가 장착되어 있다.
> - 일반 객체에는 존재하지 않기 때문에. Object.keys(obj)로 순회를 대체함.
```javascript
var a = [1,3,5,7,9]
for(var v of a) {
  console.log(v)
}
```

### 4.2.2. 이터러블

- '이터러블(iterable)' , 순회 가능한 이터레이터를 포괄한 객체.
- ES6부터 이터러블은 ES6 Symbol.iterator라는 이름을 가진 함수를 지니고 있어야 이터레이터를 가져올 수 있다.
- for-of Loop는 자동으로 Sybole.iterator 함수를 호출하여 이터레이터를 생성하여 사용한다.
   
```javascript
var a = [1,3,5,7,9]
// 직접 이터레이터를 생성하여 사용할 수 있다.
// for-of Loop는 자동으로 생성해서 Loop를 돌림.
var it = a[Symbol.iterator]()
it.next().value
it.next().value
it.next().value
```

- 이 객체가 Symbol.iterator 함수를 가진다.
- 객체를 이터러블하게 만들어 사용가능하다.
```javascript
[Symbol.iterator]: function(){return this}
```

### 4.2.3. 제너레이터 이터레이터

- 제너레이터 자체는 이터러블이 아니다. -> 실행해야 이터러블.
- 제너레이터를 실행하면 이터레이터를 돌려받게 된다.

> 무한 수열 생성기
> - 전에 Closure를 쓴 이유는 nextVal 상태 값을 유지하기 위하여..
> - 이렇게 yield로 시작/멈춤을 통해 nextVal 값을 유지할 수 있으므로 closure를 쓰지 않아도 된다.
> - 호출할 때마다 변수 상태값을 보존하기 위해 습관적으로 클로저 구문을 남발할 필요가 없다.
```javascript
function *something() {
  var nextVal
  while(true) {
    if(nextVal === undefined) {
      nextVal = 1
    }
    else {
      nextVal = (3 * nextVal) + 6
    }
    yield nextVal
  }
}

//something
for(var v of something) {
  console.log(v)
  if(v>500) break
}
```

### 4.2.3. 제너레이터 이터레이터 / 제너레이터 멈춤
- 루프가 완료(정상/비정상)될 경우 for-of Loop는 Iterator에게 신호한다.
- 제너레이터가 외부적으로 완료된 다음에도 내부에서 try-finally 절을 사용하면 실행할 수 있다. / 저자 Ex) DB 자원 정리.
- it.return() 시, 제너레이터 실행은 즉시 끝나고 finally 절로 옮겨감.

```javascript
function *something() {
  try {
    var nextVal
    while(true) {
      if(nextVal === undefined) {
        nextVal = 1
      }
      else {
        nextVal = (3 * nextVal) + 6
      }
      yield nextVal
    }
  }
  finally {
    console.log('Resource Free.')
  }
}

//something
for(var v of something()) {
  console.log(v)
  if(v>500) {
    it.return()
  }
}
```

## 4.3. 제너레이터를 비동기적으로 순회

- 이제 찐이다.
- 비동기 코딩 패턴과 제너레이터는 무슨 상관이 있을까?
- 어떻게 콜백의 문제점을 해결할 수 있는가?

```javascript
function foo(x,y,cb) {
  ajax("http://jaeyeop.url.1/?x+y"),
  cb
}

foo(11,31,function(err,text) {
  if(err) {
    console.error(err)
  }
  else {
    console.log(text)
  }
})
```
- foo()를 호출하고 다시 이 함수는 AJAX를 호출하여 text를 받음.
- ajax 요청을 하고 다 끝나면 응답을 출력하라!
- ajax는 비동기 함수. -> 제너레이터를 이용해 동기적으로 콜백 처리가능.
- 비동기성을 순차적/동기적 방향으로 표현할 수 없었던, 콜백의 단점을 완벽하게 보완함.
```javascript
//Generator Code
function foo(x,y) {
  ajax(
    "http://jaeyeop/x+y",
    function (err,data) {
      if(err) it.throw(err)
      else it.next(data)
    }
  )
}

function *main() {
  try {
    //data가 정상적으로 수신되면 it.next(data) 발동.
    //data가 정상적으로 수신될때까지 멈춤.
    var text = yield foo(11,31)
    console.log(text)
  }
  catch(err) {
    //err 발생 시, it.throw(err) 발동.
    //catch.
    console.log(err)
  }
}
var it = main()
it.next()
```

### 4.3.1. 동기적 에러 처리

- foo()는 ajax를 실행하는 비동기 함수이기에 try-catch로 비동기 에러를 잡을 수 없었음.
- 제너레이터는 가능하다. yield 멈춤. 콜백 처리.
- 전 예제는 제너레이터 안에서 catch.
- 지금 예제는 제너레이터 밖에서 catch. 모두 가능하다.
- 비동기 코드에서 난 에러를 동기적으로 처리할 수 있다.

```javascript
function *main() {
  var x = yield "Hello World"
  yield x.toLowerCase()
}
var it = main()
it.next().value
try {
  it.next(42)
}
catch(err) {
  console.error(err)
}
```

## 4.4. 제너레이터 + 프라미스

- 제너레이터를 비동기적으로 순회할 수 있다. 앞에서 OK!
- ES6의 백미가 제너레이터(동기적 형태의 비동기 코드) + 프라미스(믿음성과 조합성)의 만남이다.

> 프라미스 Ajax
> - foo()는 Ajax 호출 이후 프라미스를 반환한다.
> - foo()로 프라미스를 생성하고 제너레이터에서 yield해서 이터레이터 제어 코드가 이 프라미스를 받게 해보자.
```javascript
function foo(x,y) {
  //ajax 대체
  return new Promise(function(resolve, reject) {
    setTimeout(resolve, 3000)
  })
}
foo()
  .then(
    function(text) {
      console.log(text)
    },
    function(err) {
      console.error(err)
    }
  )
```

> 프라미스 Ajax + 제너레이터 yield + 이터레이터 제어코드.
> - 아주 많은 단계가 있을 때, 순회 제어를 반복하면서 프라미스가 나올 때마다 귀결되길 기다렸다가 진행하면 개꿀..
```javascript
function foo(x,y) {
  //ajax 대체
  return new Promise(function(resolve, reject) {
    setTimeout(resolve, 3000)
  })
}

function *main() {
  try {
    var text = yield foo()
    console.log(text)
  }
  catch(err) {
    console.error(err)
  }
}

var it = main()
var p = it.next.value
p.then(
  function(text) {
    it.next(text)
  },
  function(err) {
    it.throw(err)
  }
)
```


### 4.4.1. 프라미스-인식형 제너레이터 실행기

- 제너레이터 순회 + 프라미스 귀결을 기다림을 반복하면 좋음.
- 저자가 Util을 만들었습니다만, p322. 갠적으로 공부 ㄱ
- 이 책이 오래되어서 그런데 현재 이미 이러한 Util이 만들어져있음. ES7.
- https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Statements/async_function
- async function, await는 프라미스가 귀결될 때까지 이 함수를 멈추게 할 거란 사실을 자동으로 인식하게 함.
```javascript
//MDN
function resolveAfter2Seconds() {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve('resolved');
    }, 2000);
  });
}

async function asyncCall() {
  console.log('calling');
  const result = await resolveAfter2Seconds();
  console.log(result);
  // expected output: "resolved"
}

asyncCall();
```

### 4.4.2. 제너레이터에서의 프라미스 동시성

- 고민고민 -> 최적(Optimal) 성능 패턴에 주먹하라.
> Step1.
```javascript
function *foo() {
  var r1 = yield request("http://1/")
  var r2 = yield request("http://1/")
  var r3 = yield request("http://1/" + r1 + r2)

  console.log(r3)
}
// 제너레이터 순회 + 프로미스 귀결 / 반복.실행. Util함수. 가정.
run(foo)
```
>Step2
```javascript
function *foo() {
  //p1,p2는 서로 독립적이어서 동시 실행을 하는 편이 성능에 유리하다.
  var p1 = request("http://1/")
  var p2 = request("http://1/")

  var r1 = yield p1
  var r2 = yield p2
  var r3 = yield request("http://1/" + r1 + r2)

  console.log(r3)
}
// 제너레이터 순회 + 프로미스 귀결 / 반복.실행. Util함수. 가정.
run(foo)
```

>Step3
> - 순차적 비동기 흐름 제어 이상의 무언가가 필요할 때 '프라미스'
> - 믿음성, 조합성
```javascript
function *foo() {
  // 병렬 실행 + 모두 귀결
  var results = yield Promise.all([
    request("http://1/"),
    request("http://1/")
  ])
  var r1 = results[0]
  var r2 = results[1]
  var r3 = yield request("http://1/" + r1 + r2)

  console.log(r3)
}
// 제너레이터 순회 + 프로미스 귀결 / 반복.실행. Util함수. 가정.
run(foo)
```

### 4.4.2. 제너레이터에서의 프라미스 동시성 / 프라미스 숨김

- 가능한 한 비동기 관련 세부분은 감추고자 해야한다.
- 비동기성, 사실상 프라미스를 하나의 구현 상세로 봐야한다.
- 세부 로직은 반드시 제너레이터 코드로부터 멀찌감치 떨어뜨리고 의도적으로 추상화시켜 놓아야 더 상위 수준의 작업을 표현하기 편하다.
```javascript
function bar(ur1,ur2) {
  return Promise.all([
    request(url1),
    request(url2)
  ])
}

function *foo() {
  var results = yield bar(
    "http://1/", "http://2/"
  )
  var r1 = results[0]
  var r2 = results[1]
  var r3 = yield request("http://3/" +r1+r2)
  console.log(r3)
}
```