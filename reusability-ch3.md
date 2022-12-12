---
marp: true
---

# 제 3장 : 재사용성 

--- 

- "프로젝트에서 이미 있던 코드를 복사해서 붙여넣고 있다면, 무언가가 잘못된 것"
- "함께 변경될 가능성이 높은가? 따로 변경될 가능성이 높은가?" 

--- 

### 단일 책임 원칙 

- 코드를 추출해도 되는지를 확인할 수 있는 원칙 


---

### 일반적인 알고리즘은 반복을 피하자 

- 함수 : 상태를 유지하지 않으므로 행위를 나타내기 좋다. 부가작용 없는 경우에는 더 좋다. 
- 톱레벨 함수와 비교해서, 확장 함수는 구체적인 타입이 있는 객체에만 사용을 제한할 수 있으므로 좋다. 
- 수정할 객체를 argument 로 받기보다는 확장 리시버로 사용하는 것이 가독성 측면에서 좋다. 
- `TextUtils.isEmpty("Text")` 보다는 `"Text".isEmpty()` 가 더 사용하기 쉽다. 
    - isEmpty() 가 어디에 있는지 찾지 않아도 됨 (auto completion 활용 가능)

---

### 일반적인 프로퍼티 패턴은 프로퍼티 위임으로 만들어라 

- 프로퍼티 위임을 사용하면 일반적인 프로퍼티의 행위를 추출해서 재사용할 수 있음 
- 대표적으로 지연 프로퍼티 `by lazy` 

---

- by 는 어떻게 컴파일 되는가 ? 

```kotlin 
@JvmField
private val 'token$delegate' = LoggingProperty<String?>(null)
var token: String?
    get() = 'token$delegate'.getValue(this, ::token)
    set(value) {
        'token$delegate'.setValue(this, ::token, value)
    }
```

- getValue, setValue 는 단순하게 값만 처리하게 바뀌는 것이 아니라, 컨텍스트(this)와 프로퍼티 레퍼런스의 경계도 함께 사용하는 형태로 바뀜 

---

- kotlin stdlib 에 정의된 프로퍼티 델리게이터 
    - lazy 
    - Delegates.observable 
    - Delegates.vetoable
    - Delegates.notNull

--- 

### 일반적인 알고리즘을 구현할 때 제네릭을 사용해라 

- 타입 아규먼트를 갖는 함수, 즉 타입 파라미터를 갖는 함수를 제네릭 함수라고 부른다. 

```kotlin 
inline fun <T> Iterable<T>.filter(
    predicate: (T) -> Boolean
): List<T> {
    val dest = ArrayList<T>()
    for (e in this) {
        if(predicate(e)) {
            dest.add(e)
        }
    }
    return dest
}
```

---

```
// Java
interface Collection<E> ... {
    void addAll(Collection<E> items);
}

// Java
void copyAll(Collection<Object> to, Collection<String> from) {
    to.addAll(from);
    // !!! Would not compile with the naive declaration of addAll:
    // Collection<String> is not a subtype of Collection<Object>
}

// Java
interface Collection<E> ... {
    void addAll(Collection<? extends E> items);
}
```

---
### covariance

- The wildcard type argument `? extends E` indicates that this method accepts a collection of objects of E or a subtype of E, not just E itself. 
- This means that you can safely `read E's from items` (elements of this collection are instances of a subclass of E), 
- but `cannot write` to it as you don't know what objects comply with that unknown subtype of E. 
- `Collection<String>` is a subtype of `Collection<? extends Object>`.
- In other words, the wildcard with an `extends-bound` (upper bound) makes the type `covariant`.

--- 
### contravariance

- if you can only `put` items into the collection, it's okay to take a collection of Objects and put Strings into it: in Java there is `List<? super String>`, a `supertype` of `List<Object>`.
-  you can only call methods that take `String` as an argument on `List<? super String>`
    - for example, you can call `add(String)` or `set(int, String)`
- If you call something that returns `T in List<T>`, you don't get a `String`, but rather an `Object`.


---

- "For maximum flexibility, use wildcard types on input parameters that represent producers or consumers", and proposes the following mnemonic:

- PECS stands for Producer-Extends, Consumer-Super.
