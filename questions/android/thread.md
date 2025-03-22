# Многопоточность



### MustHave

<details>
<summary>[th-1] Что такое ThreadPool, зачем он нужен?</summary>
ThreadPool - пул потоков, где определенное число потоков создается для выполнения целого ряда задач, которые обычно организуются в очереди.

Они используются в CoroutineDispatcher, например
```kotlin 
launch(Dispatchers.Default) { // will get dispatched to DefaultDispatcher
println("Default               : I'm working in thread ${Thread.currentThread().name}")
}
```
Выведет:

Default               : I'm working in thread DefaultDispatcher-worker-1

Работа которую мы выполняем в короутине будет выполняться на пуле потоков который есть у Dispatchers.Default. Мы можем единовременно выполнять несколько короутин на одном пуле потоков, тогда из задач будет выстренна очередь, задачи из которой и будут передаваться освободившимся потокам пула.
</details>

<details>
<summary>[th-2] Что такое Handler’ы и Looper’ы, зачем нужны?</summary>
Рассмотрим класс HandlerThread, произошедший от Thread. 

![IMAGE](http://javaway.info/wp-content/uploads/2016/10/Looper.png)

Единственное существенное отличие между HandlerThread и Thread заключается в том что первый содержит внутри себя Looper, Thread и MessageQueue.

Looper трудится, обслуживая looperMessageQueue для текущего потока.

MessageQueue это очередь которая содержит в себе задачи, называющиеся сообщениями, которые нужно обработать.

Looper перемещается по этой очереди и отправляет сообщения в соответствующие обработчики для выполнения.

Любой поток может иметь единственный уникальлный Looper, это ограничение достигается с помощью концепции ThreadLocal хранилища.

Связка Looper+MessageQueue выглядит как конвейер с коробками.

Задачи в очередь помещают Handler‘ы.

Более детально можно посмотреть в [статье](http://javaway.info/mnogopotochnost-v-android-looper-handler-handlerthread-chast-1/)
</details>

### Лишним не будет

<details><summary>[th-3] Как запустить задачу на отдельном потоке?</summary></details>

<details><summary>[th-4] Как остановить запущенный поток?</summary></details>


