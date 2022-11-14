---
marp: true
---

# [ITEM 33] 생성자 대신 팩토리 함수를 사용하라

- 생성자의 역할을 대신 해주는 함수를 Factory 함수라고 한다. 

---

### 장점 

- 생성자와 다르게 함수에 이름을 붙일 수 있다. 

```kotlin 
ArrayList(3) 
ArrayList.withSize(3) // 3 이 무슨 역할인지 명확함 
```

---

- 인터페이스 뒤에 실제 객체의 구현을 숨길 때 유용하다. 
    - Kotlin/JVM, Kotlin/JS, Kotlin/Native 에 따라서 각 플랫폼의 빌트인 컬렉션으로 만들어진다. 
- `listOf` 
```kotlin
public fun <T> listOf(vararg elements: T): List<T> 
    = if (elements.size > 0) elements.asList() else emptyList()
```

---

- 호출될 때마다 새 객체를 만들 필요가 없다. 객체를 만들 수 없는 경우엔 null 을 리턴하도록 할수도 있다. 

```kotlin 
Connection.createOrNull() 

public fun <T> List<T>.getOrNull(index: Int): T? {
    return if (index >= 0 && index <= lastIndex) get(index) else null
}
```

---

- 객체 외부에 팩토리 함수를 만들면 가시성을 원하는대로 제어할 수 있다. 
- 팩토리 함수는 `inline`으로 만들 수 있고, 그 파라미터들을 `reified` 로 만들 수 있다. 
- 생성자로 만들기 복잡한 객체도 만들어 낼 수 있다. 

--- 

- `inline` : 고차 함수(higher-order function) 는 런타임에 오버헤드가 발생. 
    - 각 함수는 객체이며 closure 를 가짐
    - closure 란, 함수의 바디 안에서 접근 가능한 변수들에 대한 범위 설정임 
    - 이런 작업들을 위한 메모리 할당이 런타임 오버헤드
- `inline` 을 사용해서 다음과 같은 코드 블럭을 컴파일러가 복붙할 수 있다 
- "The lock() function could be easily inlined at call-sites." 
```kotlin
inline fun <T> lock(lock: Lock, body: () -> T): T { 
    lock.lock() 
    try {
        return body()
    } finally {
        lock.unlock() 
    }
}

// 활용 
lock(l) { foo() }
```
---

- `reified` : `inline` 함수에서는 `reified` type parameter 를 지원함
    - 제네릭 타입 T 를 함수 안에서 일반 클래스인 것 처럼 접근할 수 있도록 함 
    - `!is`, `as` 와 같은 키워드 사용 가능 

```kotlin
inline fun <reified T> membersOf() = T::class.members

fun main(s: Array<String>) {
    println(membersOf<StringBuilder>().joinToString("\n"))
}
```

> 출처 : https://kotlinlang.org/docs/inline-functions.html#inline-properties

---

### 팩토리 함수의 종류 

1. companion 객체 
2. 확장 함수 
3. Top-level 함수 
4. 가짜 생성자 
5. 팩토리 클래스의 메서드 

---

### 1. Companion 객체 

```kotlin 
class MyLinkedList<T>(
    val head: T,
    val tail: MyLinkedList<T>?
) {

    companion object {
        fun <T> of(vararg elements: T): MyLinkedList<T>? {
            if (elements.isEmpty()) return null
            val head = elements.first()
            // 첫 요소를 제외한 나머지 요소들
            val elementsTail = elements.copyOfRange(fromIndex = 1, toIndex = elements.size)
            val tail = of(*elementsTail) // 나머지 요소들을 새로운 리스트로 재귀적으로 생성
            return MyLinkedList(head, tail)
        }
    }
}
```

---

- Java 의 정적 팩토리 함수 (static factory method) 와 같음 
- C++ 에서는 이름을 가진 생성자 (named constructor idiom) 이라 부름 
- 생성자와 같은 역할을 하면서 다른 이름이 있음 

---

### Companion 객체 생성 예시 

```kotlin 
// 현재 시각으로 날짜 객체 생성
val date: Date = Date.from(Instant.now()) 

// BigInteger.valueOf(IntArray)
// BigInteger.valueOf(Long)
val prime: BigInteger = BigInteger.valueOf(10000L) 

val dialogFragment = DialogFragment.newInstance(title, onConfirm, onDismiss)
```

---

### CoroutineContext.Key 
```kotlin 
open class BaseElement : CoroutineContext.Element {

    companion object Key : CoroutineContext.Key<BaseElement>
 
    override val key: CoroutineContext.Key<*> get() = Key
    // It is important to use getPolymorphicKey and minusPolymorphicKey
 
    override fun <E : CoroutineContext.Element> get(key: CoroutineContext.Key<E>): E? = getPolymorphicElement(key)
 
    override fun minusKey(key: CoroutineContext.Key<*>): CoroutineContext = minusPolymorphicKey(key)
}
```
- https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-abstract-coroutine-context-key/

--- 

### 2. 확장 함수 

- 이미 companion 객체가 존재할 때 팩토리 함수 만들기 
- companion 객체를 직접 수정할 수는 없을 때 확장 함수 활용 
- 다만 companion 객체를 확장하려면 (적어도 비어있는) 컴패니언 객체가 필요함 

--- 

### 확장 함수 예시 

```kotlin 
interface Tool {
    companion object
}

class BigTool : Tool {
    init {
        println("initializing BigTool")
    }
}

fun MyTool.Companion.createBigTool(): BigTool {
    return BigTool()
}
```

---

### ViewHolder 생성 
```kotlin 
inline fun <reified T: RecyclerView.ViewHolder> ViewGroup.create(
    createHolder: (View) -> T, 
    @LayoutRes res: Int
): T {
    val inflater = LayoutInflater.from(context)
    val itemView = inflater.inflate(res, this, false)
    return createHolder(itemView)
}
```
- 활용 
```kotlin
val viewHolder = ViewGroup.create(::MyViewHolder, R.layout.example_layout)
```

---

### 3. 톱 레벨 함수 

- 대표적으로 `listOf` `setOf` `mapOf` 와 같은 함수 
- Activity 를 시작하기 위해 Intent 만드는 함수 정의해서 사용할수도 

```kotlin 
class MainActivity: AppCompatActivity {

    companion object {
        fun getIntent(context: Context): Intent {
            return Intent(context, MainActivity::class.java)
        }
    }
}
```

---

- Anko 라이브러리를 사용하면 reified 타입을 활용해서 `intentFor` 이라는 톱 레벨 함수를 사용할 수 있음 

```kotlin
intentFor<MainActivity>() 
```

- But, Anko 는 현재 deprecate 상태이며, Android KTX, Compose 등을 대신 사용해야 함 
- https://github.com/Kotlin/anko/blob/master/GOODBYE.md

--- 

- List.of(1, 2, 3) 보다는 listOf() 가 읽기도 쓰기도 편함 
- 하지만 public 톱레벨 함수는 모든 곳에서 사용 가능하므로 신중히 사용하기 
- 톱레벨 함수 이름을 클래스 메서드처럼 지으면 혼란스러우므로 이름을 신중하게 지을 것 

---

### 4. 가짜 생성자 

- 생성자처럼 보이고, 생성자처럼 작동하는 함수 
- 하지만 클라이언트 입장에서는 원래 생성자처럼 보임 
- 인터페이스를 위한 생성자를 만들고 싶을 때 활용 
- reified 타입 아규먼트를 갖게 하고 싶을 때 활용 

---

```kotlin 
// 내부적으로는 MutableList로 생성하나, 불변인 List로 리턴  
public inline fun <T> List(
    size: Int,
    init: (index: Int) -> T
): List<T> = MutableList(size, init)

// ArrayList 를 사용함 
public inline fun <T> MutableList(
    size:Int,
    init: (index: Int) -> T
): MutableList<T> {
    val list = ArrayList<T>(size)
    repeat(size) { idx ->
        list.add(init(idx))
    }
    return list
}
```

---

### 5. 팩토리 클래스의 메서드 

- 팩토리 함수와 달리 클래스이기 때문에, 클래스의 상태를 가질 수 있다. 

---

 ```kotlin 
 data class Student(
    val id: Int, 
    val name: String, 
    val surname: String 
 )

 class StudentsFactory {
    var nextId = 0

    fun next(name: String, surname: String): Student {
        // nextId 라는 클래스 상태 값을 활용해 객체 생성 
        return Student(nextId++, name, surname)
    }
 }
 ```

 ---

 ```kotlin 
 fun main() {
    val factory = StudentsFactory() 
    val s1 = factory.next("Marcin", "Moskala")
    println(s1) // id = 0

    val s2 = factory.next("Igor", "Wojda") 
    println(s2) // id = 1
 }
 ```

 ---

# [ITEM 34] 기본 생성자에 이름 있는 옵션 아규먼트를 사용하라 

- 자바 디자인 패턴 중 
    - 점층적 생성자 패턴 (telescoping constructor pattern)
    - 빌더 패턴 (builder pattern)
- 들은 코틀린에서 필요 없음 

--- 

### 점층적 생성자 패턴 : Java 스타일 
```kotlin 
class Pizza {
    val size: String 
    val cheese: Int 
    val olives: Int 
    val bacon: Int 
    
    constructor(size: String, cheese: Int, olives: Int, bacon: Int) {
        this.size = size 
        this.cheese = cheese 
        this.olives = olives
        this.bacon = bacon 
    }
    
    constructor(size: String, cheese: Int, olives: Int) : this(size, cheese, olives, 0)
    constructor(size: String, cheese: Int): this(size, cheese, 0)
    constructor(size: String): this(size, 0)
}
```

--- 

### 빌터 패턴 : Java 스타일 

```kotlin
class PizzaBD private constructor(
    val size: String, 
    val cheese: Int, 
    val olives: Int,  
    val bacon: Int
) {
    class Builder(private val size: String) {
        private var cheese: Int = 0 
        private var olives: Int = 0 
        private var bacon: Int = 0 
        
        fun setCheese(value: Int): Builder = apply { 
            cheese = value 
        }
        
        fun setOlives(value: Int): Builder = apply { 
            olives = value 
        }
        
        fun setBacon(value: Int): Builder = apply { 
            bacon = value 
        }
        
        fun build() = PizzaBD(size, cheese, olives, bacon)
    }
}
```

---

### 빌더 패턴, 점층적 생성자 패턴 : Kotlin 스타일 

```kotlin
class PizzaKT(
    val size: String,
    val cheese: Int = 0,
    val olives: Int = 0,
    val bacon: Int = 0
)

val p = PizzaKT(
    size = "L", 
    cheese = 1, 
    olives = 3, 
    bacon = 1
)
```

- Named argument 사용으로 훨씬 명확해짐 
- 파라미터 값 순서 지키지 않아도 됨 
- **동시성 관련 문제가 없음** : 코틀린의 함수 파라미터는 항상 immutable 이므로 thread-safe 함  

---
### 빌더 패턴이 더 좋은 경우 
- 다음 예시들을 단순 `data class` 로 변경하는 경우를 떠올려보자 

```kotlin 
// 값의 의미를 묶어서 지정하는 케이스 
val dialog = AlertDialog.Builder(context)
    .setMessage(R.string.message)
    .setPositiveButton(R.string.confirm, {dialog, id -> 
        // set positivie 
    })
    .setNegativeButton(R.string.calcel, {dialog, id -> 
        // set negative 
    })
    .create() 
```

---

```kotlin
// 특정 값을 누적하는 형태로 사용하는 케이스 
val router = Router.Builder()
    .addRoute(path="/home", ::showHome)
    .addRoute(path="/users", ::showUsers)
    .build() 

// 빌더 사용하지 않는 경우: 
val router = Router(
    routes = listOf(Route("/home", ::showHome), Route("/users", ::showUsers))
) // BAD: Route 이라는 새로운 데이터 타입 생성비용
```

--- 

### 빌더 패턴의 클래스를 팩토리 함수로 래핑 가능 

```kotlin 
fun Context.makeDefaultDialogBuilder() = 
    AlertDialog.Builder(this)... 
```

---

# [ITEM 35] 복잡한 객체를 생성하기 위한 DSL 을 정의하라 
- DSL : Domain Specific Language 
- Boilerplate 와 복잡성을 숨길 수 있음 
- Kotlin HTML DSL 
--- 

### 사용자 정의 DSL 

- 리시버를 가진 함수 타입 
    - 파라미터 앞에 리시버 타입이 추가되고 점(.) 기호로 구분됨 

```kotlin
// 리시버를 가진 함수 타입 - this 의 참조 대상을 변경할 수 있다.
val receiverPlus: Int.(Int) -> Int = fun Int.(other: Int) = this + other

fun main() {
    receiverPlus.invoke(1, 2)
    receiverPlus(1, 2)
    1.receiverPlus(2)
}
```
- `apply`, `also` 
---

- HTML 테이블 표현하는 DSL 만들어보기 

```kotlin 

class TableBuilder {
    var trs = listOf<TrBuilder>()
    fun tr(init: TrBuilder.() -> Unit) {
        trs = trs + TrBuilder().apply(init)
    }

    override fun toString(): String {
        val sb = StringBuffer()
        trs.forEach {
            it.tds.forEach { tdBuilder ->
                sb.append(tdBuilder.text)
                sb.append('\n')
            }
        }
        return sb.toString()
    }
}
```

---

```kotlin
class TrBuilder {
    var tds = listOf<TdBuilder>()
    fun td(init: TdBuilder.() -> Unit) {
        tds = tds + TdBuilder().apply(init)
    }
}

class TdBuilder {
    var text = ""

    operator fun String.unaryPlus() {
        text += this
    }
}
```

---

```kotlin
fun table(init: TableBuilder.() -> Unit): TableBuilder = TableBuilder().apply(init)
```
- init의 타입 == TableBuilder 리시버를 가진 함수 타입
- TableBuilder().apply(init) 을 통해 TableBuilder 객체 수정 

---

```kotlin
fun main() {
    val table = table {
        tr {
            for (i in 1..2) {
                td { +"This is column $i" }
            }
        }
    }
    println(table)
}
```

--- 

- 단순한 기능까지 DSL을 사용할 경우 유지보수가 어려움 
- 복잡한 자료구조, 계층적 구조, 거대한 양의 데이터를 표현할 경우 유용함 

--- 
###
```kotlin 
BulletSection {
    BulletTitle("공지사항 1") {
        BulletItem("수익률의 계산식은 다음과 같습니다")
        BuelletItem("투자 결과에 대한 책임은 투자자 본인이")
    }
}
```
##### 다음과 같이 표현될 수 있음: 
- 공지사항 1 
    - 수익률의 계산식은 다음과 같습니다
    - 투자 결과에 대한 책임은 투자자 본인이