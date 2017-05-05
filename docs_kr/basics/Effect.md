# 이펙트 추상화


일반적으로, Saga 내부에서 사이드 이펙트를 일으키는 것은 항상 서술적인 이펙트를 yield 합니다. (물론 직접 Promise 를 yield 할 수도 있습니다. 하지만 첫번째 섹션에서 봤던것처럼 테스트를 어렵게 만들겁니다.)
<!--To generalize, triggering Side Effects from inside a Saga is always done by yielding some declarative Effect. (You can also yield Promise directly, but this will make testing difficult as we saw in the first section.)-->

Saga 가 실제로 하는 일은, 훌륭한 제어 흐름을 구현하기 위해 모든 이펙트들을 통합하는것 입니다. yield 들을 차례차례 넣음으로써 yield 된 이펙트들의 순서를 지키는것 처럼요. 물론 (`if`, `while`, `for`) 같은 친숙한 흐름 연산자를 사용하여 더 세련된 제어 흐름을 구현할 수도 있습니다.
<!--What a Saga does is actually compose all those Effects together to implement the desired control flow. The simplest example is to sequence yielded Effects by just putting the yields one after another. You can also use the familiar control flow operators (`if`, `while`, `for`) to implement more sophisticated control flows.-->


`takeEvery` 처럼 고레벨 API들 이 결합된 `call` 과 `put` 같은 이펙트들을 사용하는것을 알아보았습니다. `redux-thunk` 와 결과는 비슷하지만 쉽게 테스트 할 수 있다는 장점이 있죠.
<!--We saw that using Effects like `call` and `put`, combined with high-level APIs like `takeEvery` allows us to achieve the same things as `redux-thunk`, but with the added benefit of easy testability.-->

그러나 `redux-saga` 는 `redux-thunk` 의 다른 장점들도 제공합니다. 고급 개념 섹션에선, 테스트에 대한 장점을 동일하게 제공하면서 더 복잡한 제어 흐름을 가능하게 하는 강력한 이펙트들에 대해 알아보겠습니다.
<!--But `redux-saga` provides another advantage over `redux-thunk`. In the Advanced section you'll encounter some more powerful Effects that let you express complex control flows while still allowing the same testability benefit.-->
