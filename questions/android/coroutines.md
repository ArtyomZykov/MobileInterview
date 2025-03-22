# Короутины

### MustHave

<details>
<summary>[cor-1] Что такое Dispatchers</summary>
Контекст корутины включает себя такой элемент как диспетчер корутины. Диспетчер корутины определяет какой поток или какие потоки будут использоваться для выполнения корутины.
Рассмотрим доступные типы диспетчеров:

Dispatchers.Default: применяется по умолчанию, если тип диспетчера не указан явным образом. Этот тип использует общий пул разделяемых фоновых потоков и подходит для вычислений, которые не работают с операциями ввода-вывода (операциями с файлами, базами данных, сетью) и которые требуют интенсивного потребления ресурсов центрального процессора.

Dispatchers.IO: использует общий пул потоков, создаваемых по мере необходимости, и предназначен для выполнения операций ввода-вывода (например, операции с файлами или сетевыми запросами).

Dispatchers.Main: применяется в графических приложениях, например, в приложениях Android или JavaFX.

Dispatchers.Unconfined: корутина не закреплена четко за определенным потоком или пулом потоков. Она запускается в текущем потоке до первой приостановки. После возобновления работы корутина продолжает работу в одном из потоков, который сторого не фиксирован. Разработчики языка Kotlin в обычной ситуации не рекомендуют использовать данный тип.

newSingleThreadContext и newFixedThreadPoolContext: позволяют вручную задать поток/пул для выполнения корутины
</details>

<details>
<summary>[cor-2] Чем отличаются друг от друга билдеры корутин launch и async?</summary>

launch возвращает Job. Мы можем использовать его для отмены короутины или чтобы дождаться ее окончания через join

async возвращает Deferred, и предполагает наличия результата. Мы можем испоьзовать Deferred для отмены короутины или await для ожидания результата
</details>

### Лишним не будет

<details>
<summary>[cor-3] При отмена короутины, что произойдет?</summary>
Корутины обрабатывают отмену, создавая специальное исключение: CancellationException
Используя это исключение возможно корректно обработать остановку короутины, например освободить ресурсы которые были в ней использованны. Корутина в состоянии отмены не может быть приостановлена!
Чтобы иметь возможность вызывать функции suspend при отмене корутины, нам нужно будет переключить работу очистки, которую мы должны выполнить в NonCancellableCoroutineContext

```kotlin
// предположим, что у нас есть область, определенная для этого слоя приложения
val job1 = scope.launch { … }
val job2 = scope.launch { … }
scope.cancel()
```
Отмена сферы действия корутины отменяет ее дочерние элементы
```kotlin
// предположим, что у нас есть область, определенная для этого слоя приложения
val job1 = scope.launch { … }
val job2 = scope.launch { … }
// Первая корутина будет отменена, а вторая нет
job1.cancel()
```
Отмена конкретной короутины не отменит остальные

Отмена короутины не означает мгновенное прекращение ее работы, поэтому нужно периодически проверять отмену перед началом любой длительной работы
Все функции приостановки работы от kotlinx.coroutines могут быть отменены: withContext, delay и т. д. Поэтому, если вы используете любой из них, не нужно проверять отмену и останавливать выполнение или создавать исключение CancellationException. Но, если вы их не используете, то, чтобы сделать ваш код корутины совместимым, есть два варианта:

- Проверка job.isActive или ensureActive()
- Позволить другой работе проходить через yield()
</details>

### Нюансы

<details>
<summary>[cor-4] Как работают короутины?</summary>
Короутины работают с помощью сгенрированной машины состояний (то что вы знаете о ее существовании уже даст плюсик в карму)

Suspend функции взаимодействуют друг с другом с помощью `Continuation` объектов. `Continuation` объект - это простой generic интерфейс с дополнительными данными. Позже мы увидим, что сгенерированная машина состояний для suspend функции будет реализовывать этот интерфейс.
Сам интерфейс выглядит так:
```kotlin
interface Continuation<in T> {
  public val context: CoroutineContext
  public fun resumeWith(value: Result<T>)
}
```
`context` - это экземпляр `CoroutineContext`, который будет использоваться при возобновлении.
`resumeWith` - возобновляет выполнение корутины с `Result`, он может либо содержать результат вычисления, либо исключение.

Компилятор заменяет ключевое слово suspend на дополнительный аргумент `completion` (тип `Continuation`) в функции, аргуемнт используется для передачи результата suspend функции в вызывающую корутину:
```kotlin
fun loginUser(userId: String, password: String, completion: Continuation<Any?>) {
  val user = userRemoteDataSource.logUserIn(userId, password)
  val userDb = userLocalDataSource.logUserIn(user)
  completion.resume(userDb)
}
```
Компилятору нужно знать:
Функция вызывается первый раз или
Функция была возобновлена из предыдущего состояния Для этого проверяется тип аргумента `Continuation` в функции:
```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {
  /* ... */
  val continuation = completion as? LoginUserStateMachine ?: LoginUserStateMachine(completion)
  /* ... */
```
Байткод suspend функций фактически возвращает Any? так как это объединение (union) типов T | COROUTINE_SUSPENDED. Что позволяет функции возвращать результат синхронно, когда это возможно.
Если suspend функция не вызывает другие suspend функции, компилятор добавляет аргумент `Continuation`, но не будет с ним ничего делать, байткод функции будет выглядеть как обычная функция.

Компилятор Kotlin определяет, когда функция может остановится внутри. Каждая точка прерывания представляется как отдельное состояние в конечной машине состояний. Такие состояния компилятор помечает метками:

```kotlin 
fun loginUser(userId: String, password: String, completion: Continuation<Any?>) {
/* ... */
// Label 0 -> first execution
val user = userRemoteDataSource.logUserIn(userId, password)

// Label 1 -> resumes from userRemoteDataSource
val userDb = userLocalDataSource.logUserIn(user)

// Label 2 -> resumes from userLocalDataSource
completion.resume(userDb)
}
```

При этом компилятор создает приватный класс, который:

- хранит нужные данные

- вызывает функцию `loginUser` рекурсивно для возобновления вычисления

- Ниже представлен примерный вид такого сгенерированного класса:

```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {

  class LoginUserStateMachine(
    // completion parameter is the callback to the function
    // that called loginUser
    completion: Continuation<Any?>
  ): CoroutineImpl(completion) {

    // Local variables of the suspend function
    var user: User? = null
    var userDb: UserDb? = null

    // Common objects for all CoroutineImpls
    var result: Any? = null
    var label: Int = 0

    // this function calls the loginUser again to trigger the
    // state machine (label will be already in the next state) and
    // result will be the result of the previous state's computation
    override fun invokeSuspend(result: Any?) {
      this.result = result
      loginUser(null, null, this)
    }
  }
  /* ... */
}
```
Поскольку invokeSuspend вызывает `loginUser` только с аргументом `Continuation`, остальные аргументы в функции `loginUser` будут нулевыми. На этом этапе компилятору нужно только добавить информацию как переходить из одного состояния в другое.

Для смены состояний генерируется код
```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {
    /* ... */

    val continuation = completion as? LoginUserStateMachine ?: LoginUserStateMachine(completion)

    when(continuation.label) {
        0 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Next time this continuation is called, it should go to state 1
            continuation.label = 1
            // The continuation object is passed to logUserIn to resume
            // this state machine's execution when it finishes
            userRemoteDataSource.logUserIn(userId!!, password!!, continuation)
        }
        1 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Gets the result of the previous state
            continuation.user = continuation.result as User
            // Next time this continuation is called, it should go to state 2
            continuation.label = 2
            // The continuation object is passed to logUserIn to resume
            // this state machine's execution when it finishes
            userLocalDataSource.logUserIn(continuation.user, continuation)
        }
          /* ... leaving out the last state on purpose */
    }
}
```
- Появилась переменная `label` из `LoginUserStateMachine`, которая передается в when.

- Каждый раз при обработке нового состояния проверяется есть ли ошибка.

- Перед вызовом следующей suspend функции (logUserIn), LoginUserStateMachine обновляет переменную label.

- Когда внутри машины состояний вызывается другая suspend функция, экземпляр `Continuation` (с типом `LoginUserStateMachine`) передается как аргумент. Вложенная suspend функция также была преобразована компилятором со своей машиной состояний. Когда эта внутренняя машина состояний завершит свою работу, она возобновит выполнение “родительской” машины состояний.

Последнее состояние должно возобновить выполнение `completion` через вызов `continuation.completion.resume` (очевидно что входной аргумент `completion`, сохраняется в переменной `continuation.completion` экземпляра `LoginUserStateMachine`):
```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {
    /* ... */

    val continuation = completion as? LoginUserStateMachine ?: LoginUserStateMachine(completion)

    when(continuation.label) {
        /* ... */
        2 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Gets the result of the previous state
            continuation.userDb = continuation.result as UserDb
            // Resumes the execution of the function that called this one
            continuation.completion.resume(continuation.userDb)
        }
        else -> throw IllegalStateException(/* ... */)
    }
}
```
Компилятор Kotlin делает много работы “под капотом”. Из suspend функции:
```kotlin
suspend fun loginUser(userId: String, password: String): User {
  val user = userRemoteDataSource.logUserIn(userId, password)
  val userDb = userLocalDataSource.logUserIn(user)
  return userDb
}
```
Генерируется большой кусок кода:
```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {

    class LoginUserStateMachine(
        // completion parameter is the callback to the function that called loginUser
        completion: Continuation<Any?>
    ): CoroutineImpl(completion) {
        // objects to store across the suspend function
        var user: User? = null
        var userDb: UserDb? = null

        // Common objects for all CoroutineImpl
        var result: Any? = null
        var label: Int = 0

        // this function calls the loginUser again to trigger the
        // state machine (label will be already in the next state) and
        // result will be the result of the previous state's computation
        override fun invokeSuspend(result: Any?) {
            this.result = result
            loginUser(null, null, this)
        }
    }

    val continuation = completion as? LoginUserStateMachine ?: LoginUserStateMachine(completion)

    when(continuation.label) {
        0 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Next time this continuation is called, it should go to state 1
            continuation.label = 1
            // The continuation object is passed to logUserIn to resume
            // this state machine's execution when it finishes
            userRemoteDataSource.logUserIn(userId!!, password!!, continuation)
        }
        1 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Gets the result of the previous state
            continuation.user = continuation.result as User
            // Next time this continuation is called, it should go to state 2
            continuation.label = 2
            // The continuation object is passed to logUserIn to resume
            // this state machine's execution when it finishes
            userLocalDataSource.logUserIn(continuation.user, continuation)
        }
        2 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Gets the result of the previous state
            continuation.userDb = continuation.result as UserDb
            // Resumes the execution of the function that called this one
            continuation.completion.resume(continuation.userDb)
        }
        else -> throw IllegalStateException(/* ... */)
    }
}
```
Компилятор Kotlin преобразовывает каждую suspend функцию в машину состояний, с использованием обратных вызовов.

Уточнение: Приведенный код не полностью соответствует байткоду сгенерированному компилятором. Это код на Kotlin, достаточно точный, для понимания того, что в действительности происходит внутри.
</details>
