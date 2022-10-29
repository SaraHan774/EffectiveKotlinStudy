# Effective Kotlin 1-5



1. **가변성을 제한하라**

읽고 쓸 수 있는 프로퍼티 (read-write property) `var` 를 사용하거나, mutable 객체를 사용하면 상태를 가질 수 있다. 상태를 가질 경우, 해당 요소의 동작은 사용 방법뿐만 아니라, 그 이력에도 의존하게 된다. 

   ```kotlin
   var a = 10 
   var list: MutableList<Int> = mutableListOf()
   ```

   ```kotlin
   class BankAccount {
     var balance = 0.0 
     	private set 
     
     fun deposit(depositAmount: Double) {
       balance += depositAmount 
     }
     
     @Throws(InsufficientFunds::class)
     fun withdraw(withdrawAmount: Double) {
       if (balance < withdrawAmount) {
         throw InsufficientFunds() 
       }
       balance -= withdrawAmount 
     }
   }
   
   class InsufficientFunds : Exception() 
   val account = BankAccount() 
   
   println(account.balance) // 0.0 
   account.deposit(100.0)
   println(account.balance) //100.0 
   account.withdraw(50.0) 
   println(account.balance) // 50.0 
   
   /**
   * 시간의 변화에 따라서 변하는 요소를 표현할 수 있다는 것은 유용하지만, 상태를 적절하게 관리하는 것이 생각보다 어렵다. 
   * 1. 프로그램을 이해하고 디버그하기 힘들어진다 
   * 2. 가변성이 있으면 코드의 실행을 추론하기 어렵다
   * 3. 멀티스레드 프로그램일 경우 적절한 동기화가 필요하다. 
   * 4. 모든 상태를 테스트해야 하므로 테스트 조합이 많아진다. 
   * 5. 상태 변경이 일어날 때 이런 변경을 다른부분에 알려야한다. 가령, 정렬되어 있는 리스트에 가변요소를 추가하면 요소 변경이 일어날 때마다 리스트를 다시 정렬해야함. 
   */
   ```

   

   멀티스레드 활용해서 프로퍼티 수정 시 충돌에 의해 일부 연산이 이루어지지 않는다. 
   ```kotlin
   fun countThread() {
       var num = 0
       for (i in 1..1000) {
           thread {
               Thread.sleep(10)
               num += 1
               println(Thread.currentThread().hashCode()) // 매 번 다른 스레드
           }
           println("outer" + Thread.currentThread().hashCode()) // 동일한 스레드
       }
   
       Thread.sleep(5000)
       print(num) // 1000 이 나오지 않는다
   }
   
   suspend fun countCoroutine() {
       var num = 0
       coroutineScope {
           for (i in 1..1000) {
               launch {
                   delay(10)
                   num += 1
                   // 더 적은 수의 스레드로 작업되는 것을 확인할 수 있다.
                   println(Thread.currentThread().hashCode())
               }
           }
       }
       println(num) // 실행할 때마다 다른 숫자가 나온다.
   }
   
   fun lockCount() {
       val lock = Any()
       var num = 0
       for (i in 1 .. 1000) {
           thread { // creates new thread and runs this block of code
   //            Thread.sleep(10)
               synchronized(lock) {
                   num += 1
                   println("$num : " + Thread.currentThread().hashCode())
               }
           }
       }
       Thread.sleep(1000) // 하지 않으면 1000번째 스레드 결과가 한박자 늦게 출력된다 
       println(Thread.activeCount())
   }
   ```

   가변성은 생각보다 단점이 많아서, 순수 함수형 프로그래밍 언어 Haskell 처럼 이를 완전하게 제한하는 언어도 있다. 


   **코틀린에서 가변성 제한하기**

   - 읽기 전용 프라퍼티 val 
   - 가변 컬렉션과 읽기 전용 컬렉션 구분하기 
   - 데이터 클래스의 copy 


   var 은 게터와 세터를 모두 제공하지만 val 은 변경이 불가능하므로 게터만 제공합니다. 그래서 val 을 var 로 오버라이딩할 수 있습니다. 
   ```kotlin
   interface Element {
     val active: Boolean 
   }
   
   class ActualElement : Element {
     override var active : Boolean = false 
   }
   ```

   ```kotlin
   val name: String? = "Martin"
   val surname: String = "Baron"
   
   val fullName: String? 
       get() = name?.let { "$it $surname" }
   val fullName2: String? = name?.let { "$it $surname" }
   
   fun fullName() {
       // Smart cast to 'String' is impossible, because 'fullName' is a property that has open or custom getter
       // if (fullName != null) println(fullName.length)
       if (fullName2 != null) println(fullName2.length)
   }
   ```

   다운캐스팅은 하지 말것 

   ```kotlin
   val list = listOf(1, 2, 3)
   // 이렇게 하지 마세요 ! 
   if (list is MutableList) {
     list.add(4)
   }
   /**
   * 컬렉션 다운캐스팅은 읽기전용 계약을 위반하고 추상화를 무시하는 행위 
   * 안전하지 않고, 예측하지 못한 결과를 초래. 
   * 읽기 전용에서 mutable 로 변경해야 한다면, 복제(copy) 를 통해 새로운 mutable 컬렉션을 만드는 list.toMutableList() 를 활용해야 함. 
   */
   
   val list = listOf(1, 2, 3) 
   val mutableList = list.toMutableList() 
   mutableList.add(4) 
   ```

   **데이터클래스의 copy** 
   
   String, Int 처럼 내부적인 상태를 변경하지 않는 immutable 객체를 사용하면 다음과 같은 장점이 있다. 

   - 한 번 정의된 상태가 유지되므로 코드 이해 쉬움 
   - immutable 객체는 공유했을때도 충돌이 따로 이루어지지 않으므로, 병렬 처리 안전 
   - **immutable 객체에 대한 참조는 변경되지 않으므로 쉽게 캐시할 수 있음** 
   - immutable 객체는 방어적 복사본 (defensive copy) 를 만들 필요가 없음, 객체 복사시 deep copy 를 해야할 필요가 없음 
   - immutable 객체는 set, map 등의 키로 사용할 수 있음. 해시 테이블은 처음 요소를 넣을때 요소의 값을 기반으로 버킷을 결정. 따라서 요소에 수정이 일어나면 해시테이블 내부에서 요소를 찾을 수 없음. 

   ```kotlin
   val names : SortedSet<FullName> = TreeSet() 
   val person = FullName("AAA", "AAA")
   names.add(person)
   names.add(FullName("adsf", "qwer"))
   names.add(FullName("adsf2", "qwer2"))
   
   print(names) 
   print(person in names) // true 
   
   person.name = "ZZZ"
   print(names) // ZZZ 로 바뀐채로 출력 
   print(person in names) // false 
   ```

   자신을 수정한 새로운 객체를 만들어내야 한다 

   ```kotlin
   class User(val name: String, val surname: String){
     fun withName(surname: String) = User(name, surname)
   }
   
   // data class 를 활용하면 위와 같은 기능을 copy 를 통해 달성할 수 있다 
   data class User(
   	val name:String, 
     val surname:String
   )
   
   var user = User("sdf", "sdfsdf")
   user = user.copy(surname = "qweqwe")
   print(user) // User(name="sdf", surname = "qweqwe")
   ```

   변경가능지점 
   ```kotlin
   class Mutable {
       val list1: MutableList<Int> = mutableListOf() // 리스트 구현 내부에 변경가능지점이 있다 
       var list2: List<Int> = listOf() // 프로퍼티 자체가 변경 가능 지점이다. 
       
       fun count() {
           var list = listOf<Int>() 
           
           for (i in 1 .. 1000) {
               thread { 
                   list = list.plus(i)
               }
           }
           Thread.sleep(1000)
           print(list.size)
       }
       
       // mutable 리스트 대신 mutable 프라퍼티를 사용하는 형태는 사용자 정의 세터
       // 또는 이를 사용하는 델리게이트를 활용해서 변경을 추적할 수 있다. 
       // 예를 들어, Delegates.observable 을 사용하면 리스트에 변경이 있을때 로그 출력 가능 
       var names by Delegates.observable(listOf<String>()) { _, old, new -> 
           println("name changed from $old to $new")
       }
       
       fun changeName() {
           names += "Fabio" //'+=' creates new list under the hood 
           names += "Bill"
       }
   }
   ```

   컬렉션은 객체를 읽기전용으로 업캐스트하여 가변성을 제한할 수 있다. 
   ```kotlin
   data class User(val name: String)
   
   class UserRepository {
     private val storedUsers: MutableMap<Int, String>  = mutableMapOf() 
     
     fun loadAll() : Map<Int, String> {
       return storedUsers
     }
   }
   ```

   

### 2. **변수의 스코프를 최소화하라**

   ```kotlin
   /**
   * 상태를 정의할때는 변수와 프로퍼티의 스코프를 최소화하는게 좋다. 
   * 1. 프로퍼티보다는 지역 변수를 사용하는 것이 좋다 
   * 2. 최대한 좁은 스코프를 갖게 변수를 사용한다. 예를 들어, 반복문 내부에서만 변수가 사용된다면 변수를 반복문 내부에 작성하는 것이 좋다. 
   */
   
   // scope 를 최소화한 버전 
   for((i, user) in users.withIndex()) {
     print("User at $i is $user")
   }
   
   // 변수는 읽기 전용, 읽기 쓰기 전용인것과 상관없이 변수를 정의할 때 초기화되는게 좋다. 
   val user : User = if(hasValue) getValue() else User() 
   
   //여러 프로퍼티 한꺼번에 설정하는 경우 destructuring declaration 을 활용 
   fun updateWeather(degrees: Int) {
     val (description, color) = when {
       degress < 5 -> "cold" to Color.BLUE
       degress < 23 -> "mild" to Color.YELLOW
       else -> "hot" to Color.RED
     }
   }
   
   class Eratos {
       fun simple() {
           var numbers = (2..100).toList()
           val primes = mutableListOf<Int>()
           while (numbers.isNotEmpty()) {
               val prime = numbers.first()
               primes.add(prime)
               numbers = numbers.filter { it % prime != 0 } // 조건을 만족하는 요소들만 남는다
               println(numbers)
           }
           print(primes)
       }
   
       fun sequence() {
           val primes : Sequence<Int> = sequence {
               var numbers = generateSequence(2) { it + 1}
   
               while (true) {
                   val prime = numbers.first()
                   println(prime)
                   yield(prime)
                   numbers = numbers.drop(1)
                       .filter {
                           println("filter $it")
                           it % prime != 0
                       }
               }
           }
           print(primes.take(10).toList())
       }
   
       fun captured() {
           val primes: Sequence<Int> = sequence {
               var numbers = generateSequence(2){ it + 1 }
   
               var prime: Int
               while (true) { // sequence 가 들고있을 값들을 생성하는 생성자
                   prime = numbers.first() // 첫 숫자 2 생성
                   println("prime = $prime")
                   yield(prime) // 시퀀스가 무엇을 뱉어낼건지
                   numbers = numbers.drop(1).filter { 
                       // 시퀀스를 활용하므로 필터링이 지연된다. 
                       println("filtering ... $it % $prime")
                       it % prime != 0
                   }
               }
           }
           print(primes.take(10).toList())
           // 시퀀스를 10개까지만 생성한다. 리스트로 변환한다.
       }
   
       // 시퀀스 다큐먼트 참고 
       fun fibo() {
           val fib: Sequence<Int> = sequence {
               var terms = Pair(0, 1)
   
               while (true) { // this sequence is infinite
                   // yields a value to the Iterator being build and suspends
                   // until the next value is requested
                   yield(terms.first)
                   terms = Pair(terms.second, terms.first + terms.second)
               }
           }
           println(fib.take(10).toList())
       }
   }
   ```

   

### 3. **최대한 플랫폼 타입을 사용하지 말라** (라이브러리 사용시 주의해야 할것같음)


   ```kotlin
   // java 등 다른 프로그래밍 언어에서 넘어온 타입들을 특수하게 다룬다. 이런 타입들을 플랫폼 타입이라 한다. 
   
   //자바 
   public class UserRepo {
     public User getUser() { ... }
   }
   
   // kotlin 
   val repo = UserRepo() 
   val user1 = repo.user // User! 
   val user2: User = repo.user // User 
   val user3: User? = repo.user // User? 
   
   val users: List<User> = UserRepo().users 
   val users: List<List<User>> = UserRepo().groupedUsers // 별도로 null check 할 필요 없다 
   
   // null 이 아니라고 생각되는 것이 null 일 가능성이 있으므로, 여전히 위험하다. 따라서 플랫폼 타입을 사용할때는 항상 조심해야한다. 
   
   // 자바를 코틀린과 함께 사용할 때, 자바 코드를 직접 조작할 수 있다면, 가능한 @Nullable 과 @NotNull 어노테이션을 붙여서 사용하라. 
   import org.jetbrains.annotaions.NotNull; 
   public class UserRepo{
     public @NotNull User getUser() { ... }
   }
   
   // statedType vs platformType 
   // java 
   public class JavaClass {
     public String getValue() {
       return null; 
     }
   }
   
   // kotlin 
   fun statedType() {
     val value: String = JavaClass().value // 값을 가져오는 위치에서 NPE 발생 - null 값이 들어옴
     println(value.length)
   }
   
   fun platformType() {
     val value = JavaClass().value // 값을 활용할 때 NPE가 발생 nullable 일수도, 아닐수도 있음. 이후 다른사람이 사용할때 발견할것 
     println(value.length)
   }
   
   // 인터페이스에서 플랫폼 타입 사용 
   interface UserRepo {
     fun getUserName() = JavaClass().value 
   }
   
   class RepoImpl : UserRepo {
     override fun getUserName() : String? {
       return null 
     }
   }
   
   fun main() {
     val repo : UserRepo = RepoImpl() 
     val text: String = repo.getUserName() // runtime 에 NPE 발생 
     print("User name length is ${text.length}")
   }
   
   // 이처럼 플랫폼 타입이 전파되는 일은 매우 위험하다. 
   ```

   

### 4. **inferred 타입으로 리턴하지 말라**


   ```kotlin
   /**
   * 코틀린의 타입추론은 JVM 세계에서 가장 널리 알려진 코틀린의 특징. 
   */
   
   interface CarFactory {
     fun produce() : Car 
   } 
   
   val DEFAULT_CAR : Car = Fiat126P() 
   
   // 코드를 작성하다보니, DEFAULT_CAR 는 Car 로 명시적으로 지정되어 있으므로, 따로 필요 없다고 판단해서 함수의 리턴타입을 제거하면 
   interface CarFactory {
     fun produce() = DEFAULT_CAR 
   }
    
   // 이후에 다른 사람이 다음과 같이 코드를 변경하면 .. interface 는 Fiat126P 타입의 차만 만들어낼수있다.
   val DEFAULT_CAR = Fiat126P() 
   ```

   

### 5. 예외를 활용해 코드에 제한을 걸어라


    ```kotlin    
    /**
    * 확실하게 어떤 형태로 동작해야 하는 코드가 있다면 예외를 활용해 제한을 걸자. 
    * require, check, assert, return/throw 와 함께 활용하는 elvis 연산자 
    */
    
    // 예시1 
    fun pop(num: Int = 1): List<T> {
      require(num <= size) {
        "Cannot remove more elements than current size" 
      }
      check(isOpen) { "Cannot pop from closed stack" }
      val ret = collection.take(num)
      collction = collection.drop(num)
      assert(ret.size == num)
      return ret 
    }
    
    /**
    * Argument
    * 함수를 정의할 때 타입 시스템을 활용해서 아규먼트에 제한을 거는 코드를 많이 사용한다. 
    * 숫자를 아규먼트로 받아서 팩토리얼을 계산하면 숫자는 양의 정수여야 한다, 
    * 좌표들을 아규먼트로 받아서 클러스터를 찾을 때는 비어있지 않은 좌표 목록이 필요하다, 
    * 사용자로부터 이메일 주소를 입력받을 때는 값이 입력되어 있는지, 이메일 형식이 올바른지 확인이 필요하다. 
    * ===> 이러한 제한을 걸 때는 require 함수 사용 / 제한 확인 후 만족 못할 경우 throw exception 함 
    */
    fun factorial(n : Int) : Long {
      require(n >= 0) 
      return if(n <= 1) 1 else factorial(n - 1) * n
    }
    
    fun findClusters(points: List<Point>) : List<Cluster> {
      require(points.isNotEmpty())
      // ... 
    }
    
    fun sendEmail(user: User, message: String) {
      requireNotNull(user.email)
      require(isValidEmail(user.email))
      // ... 
    }
    
    // IllegalArgumentException 던지도록 되어있음. 
    // 람다로 메시지 재정의 할수도 
    fun factorial(n: Int) : Long {
      require(n >= 0) { "Cannot calculate factorial of $n" }
      return ... 
    }
    
    /**
    * 상태 
    * 어떤 구체적인 조건을 만족할때만 함수를 사용할 수 있도록 
    * IllegalStateException 던짐 
    */
    fun speak(text: String) {
      check(isInitialized)
    } 
    
    fun getUserInfo() : UserInfo {
      checkNotNull(token)
    }
    
    fun next() : T {
      check(isOpen)
    }
    
    ```
  
