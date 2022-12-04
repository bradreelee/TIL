- promise all은 정말 병렬처리 인가?
  - parallel? concurrent?
    - https://stackoverflow.com/questions/30823653/is-node-js-native-promise-all-processing-in-parallel-or-sequentially
  - 근데 일단 Promise all 쓰는 것이 for 돌면서 매번 await 하는 것보다 훨씬 빨랐다.

# forEach와 비동기 함수

사이드 프로젝트를 진행하던 중 아래 코드와 같이 배열의 각 원소에 대하여 비동기함수를 실행할 일이 있었다.

```javascript
export const findCoords = async (arr) => {
  let result = [];
  arr.forEach(async (item, idx) => {
    const place = item["PLACE"];
    // place가 장소명일 경우
    try {
      const address = await findAddressWithPlace(place);
      try {
        const coord = await findCoordsWithAddress(address);
        result.push(coord);
      } catch (err) {}
    } catch (err) {
      // place가 주소일 수 있으므로 바로 좌표 찾기
      try {
        const coord = await findCoordsWithAddress(place);
        result.push(coord);
      } catch (err) {}
    }
  });

  console.log(resultWithCoords);
  return resultWithCoords;
};
```

위 코드를 실행했을 때, 예상과 달리 마지막 return문 전의 `console.log(resultWithCoords)`는 빈 배열을 찍어냈다.

## 원인 - forEach의 콜백함수는 동기함수여야 한다.

[mdn docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach)에서는 다음과 같이 forEach 함수를 설명한다.

> **_"forEach() expects a synchronous function — it does not wait for promises. Make sure you are aware of the implications while using promises (or async functions) as forEach callbacks. To run a series of asynchronous operations sequentially or concurrently, see promise composition."_**

그러면서 다음과 같은 예시코드를 보여준다:

```javascript
const ratings = [5, 4, 5];
let sum = 0;

const sumFunction = async (a, b) => a + b;

ratings.forEach(async (rating) => {
  sum = await sumFunction(sum, rating);
});

console.log(sum);
// Naively expected output: 14
// Actual output: 0
```

그런데 위와 같은 코드는 왜 예상과 다르게 작동할까?

### async/await 함수에 대한 오해

`await` 키워드를 사용하면 다음과 같이 promise를 다루는 것이 마치 동기 함수를 다루는 것과 비슷해진다:

```javascript
function resolveAfter2Seconds() {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve("resolved");
    }, 2000);
  });
}

async function asyncCall() {
  console.log("calling");
  const result = await resolveAfter2Seconds();
  console.log(result);
  // expected output: "resolved"
}

asyncCall();
```

[코드 출처 - mdn docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await)

그런데 나는 바로 이 "promise를 다루는 것이 마치 동기함수를 다루는 것과 비슷해진다는 것"을 오해하였다.

### async 함수는 Promise를 return한다.

[mdn docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await)에서는 `await`에 대해서 이렇게 설명된 부분이 있다.

> **_"The await operator is used to wait for a Promise and get its fulfillment value. It can only be used inside an async function or at the top level of a module. (...) Because await is only valid inside async functions and modules, which themselves are asynchronous and return promises, the await expression never blocks the main thread and only defers execution of code that actually depends on the result, i.e. anything after the await expression."_**

즉, `await`는 프로미스가 fulfilled 될때까지 기다렸다가, 그 결과값을 (변수에 할당하는 식으로) 사용할 수 있게 해준다. **_즉, 프로미스를 동기적으로 처리할 수 있게 해준다._**

하지만 이 `await`는 `async`로 선언된 비동기 함수 내에서만 사용될 수 있다. 그리고 `async` 함수는 비동기로 실행되며 그 실행상태와 결과값에 Promise를 통해 사용자가 접근할 수 있게 해준다. **_즉, `await`로 인해 프로미스가 동기적으로 처리되건 말건, `await`가 사용되는 컨텍스트 자체가 전체 애플리케이션의 메인 실행 흐름에서 벗어난 실행 흐름이라는 것이다._**

그래서 위의 코드들처럼 `forEach` 함수에 파라미터로 `async` 함수로 정의된 콜백함수를 넘겨준다면, 그저 _배열의 각 원소마다 비동기 함수를 호출하는 것_ 이 된다. 그러니 당연히 `forEach` 함수 호출 이후에 실행된 코드들이 예상과 다른 결과를 보여준 것이다.

## 수정 코드

1차적으로 맨위에 나왔던 코드는 아래와 같이 수정가능 하다.

```javascript
export const findCoords = async (arr) => {
  const result = [];

  for (let i = 0; i < arr.length; i++) {
    const item = arr[i];
    const place = item["PLACE"].trim();
    // place가 장소명일 경우
    try {
      const address = await findAddressWithPlace(place);
      try {
        const coord = await findCoordsWithAddress(address);
        result.push(coord);
      } catch (err) {
        throw err;
      }
    } catch (err) {
      // place가 주소일 수 있으므로 바로 좌표 찾기
      try {
        const coord = await findCoordsWithAddress(place);
        result.push(coord);
      } catch (err) {
        console.log(i, err);
      }
    }
  }

  console.log(resultWithCoords);
  return resultWithCoords;
};
```

하지만 사이드 프로젝트에서 위 코드가 실제 실행됬을 때 `result` 안에는 4000개 정도의 아이템이 있었다. 즉, 4000 번 이상의 api 호출이 순차적으로 호출 됬고, 이로 인해 `findCoords` 함수를 실행하는데는 굉장히 많은 시간이 소요됬다.

## 성능 개선 - `Promise.allSettled()` 사용

`findCoords` 함수의 성능을 개선시키기 위해 `Promise.allSettled` 함수를 사용하였다. `Promise.all` 함수를 사용하지 않은 이유는, `findCoords`내에서 실행되는 api 호출의 결과가 에러를 발생시킬 수 있기 때문이었다.

```javascript
export const findCoords = async (result) => {
  const promises = result.map(async function (item) {
    const place = item["PLACE"].trim();
    // place가 장소명일 경우
    try {
      const address = await findAddressWithPlace(place);
      try {
        return findCoordsWithAddress(address);
      } catch (err) {
        return new Error("error finding coords with address:", address);
      }
    } catch (err) {
      // place가 주소일 수 있으므로 바로 좌표 찾기
      try {
        return findCoordsWithAddress(place);
      } catch (err) {
        return new Error("error finding coords with place:", place);
      }
    }
  });

  return Promise.allSettled(promises);
};
```

참고로 이 `findCoord`함수는 `redux toolkit`의 `createAsyncThunk`함수 내에서 쓰여야 되서 프로미스를 리턴하게 작성하였다.

이렇게 성능개선을 하고 다음과 같이 `builder`에서 간단하게 실행시간을 해보았다.

```javascript
builder
  .addCase(fetchCoords.pending, (state) => {
    start = Date.now();
  })
  .addCase(fetchCoords.fulfilled, (state, action) => {
    end = Date.now();
    console.log(end - start);
  });
```

그 결과, 배열의 크기가 1000일 때, 즉, promise가 1000개일때, `Promise.allSettled`의 경우에는 평균적으로 약 18초, 개선전에는 약 121초의 실행시간이 측정되었다. **_비약적인 발전이다_**.

## `Promise.allSettled`는 parallel인가 concurrent인가?

위 처럼 실행시간이 짧아질 수 있었던 이유는 `Promise.allSettled` 때문이다. [stackoverflow를 찾아보니](https://stackoverflow.com/questions/30823653/is-node-js-native-promise-all-processing-in-parallel-or-sequentially) `Promise.allSettled`는 parallel은 아니고 concurrent라고 한다. 이는 더 공부해봐야할 내용이다.
