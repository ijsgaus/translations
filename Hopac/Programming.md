Использование Hopac
====================
1. [Вступление](#Вступление)
2. [Модель разработки Hopac](#the-hopac-programming-model)
  1. [Где и когда использовать Hopac](#potential-applications-for-hopac) 
3. [Несколько ступительных примеров](#a-couple-of-introductory-examples)
  1. [Пример: Updatable Storage Cells](#example-updatable-storage-cells)
  2. [Пример: Storage Cells Using Alternatives](#example-storage-cells-using-alternatives)
  3. [Пример: Kismet](#example-kismet)
4. [Starting and Waiting for Jobs](#starting-and-waiting-for-jobs)
  1. [Fork-Join Parallelism](#fork-join-parallelism)  
5. [Programming with Alternatives](#programming-with-alternatives)
  1. [Just what is an alternative?](#just-what-is-an-alternative)
  2. [Primitive Alternatives](#primitive-alternatives)
  3. [Binding an Alternative](#binding-an-alternative)
  4. [Choose and after](#choose-and-after)
  5. [Prepare](#prepare)
  6. [Negative Acknowledgments](#negative-acknowledgments)
  7. [On the Semantics of Alternatives](#on-the-semantics-of-alternatives)
6. [Channels, Mailboxes, IVars, MVars, ...](#channels-mailboxes-ivars-mvars-)
7. [Going Further](#going-further)

Вступление 
------------
Hopac предлагает модель программирования, вдохновленную идеями, использованными 
[John Reppy](http://people.cs.uchicago.edu/~jhr/) в языке программирования **Concurrent ML**.
Похожую  или близкую по духу модель используют и другие языки и библиотеки, например 
[Racket](http://docs.racket-lang.org/reference/concurrency.html), Clojure
[core.async](https://github.com/clojure/core.async/) и
[Go](https://golang.org/).  Модель программирования Racket вобрала в себя большинство идей CML, в то время как
библиотека core.async в Clojure и Go предоставляют некоторое подмножество его возможностей, опуская прежде всего 
события высшего порядка и механизм "знания об отказе", реализованного в CML.  Если вы разбираетесь в core.async или Go,
то можете ознакомится с наброском проекта реализации событий в стиле CML поверх core.async по ссылке 
на этот интересный проект [poc.cml](https://github.com/VesaKarvonen/poc.cml).

Книга
[Concurrent Programming in ML](http://www.cambridge.org/us/academic/subjects/computer-science/distributed-networked-and-mobile-computing/concurrent-programming-ml) - 
наиболее полное введение в стиль программирования Concurrent ML, которое можно рекомендовать как базовый курс для изучения. 
Этот документ - первоначальное описание и примеры использования этой техники программирования с использованием Hopac.
Возможно в будущем он вырастет до настоящего введения в эту библиотеку. Отзывы привествуются!

Файл
[Hopac.fsi](https://github.com/Hopac/Hopac/blob/master/Libs/Hopac/Hopac.fsi)
содержит прокомметированные сигнатуры примитивов Hopac, используемых в этом документе.  По ссылке
[Hopac Library Reference](http://hopac.github.io/Hopac/Hopac.html) можно найти автоматически генерируемое из исходных
текстов и комментариев руководство по примитивам Hopac.  Рекомендуется открыть решение (solution) Hopac в Visual Studio и запустить 
среду F# interactive, тогда вы сможете быстро попробовать примеры из этого руководства. Используйте скрипт
[Hopac.fsx](https://github.com/Hopac/Hopac/blob/master/Hopac.fsx) для настройки окружения, что бы иметь возможность сразу
запускать примеры.

Модель программирования Hopac
---------------------------

Есть два базовых аспекта, определяющих основу программирования с использованием Hopac.

Первый из них -  крайне леговесные нити (threads), которые мы называем *jobs*, представленные типом
`Job<'x>`[*](http://hopac.github.io/Hopac/Hopac.html#def:type%20Hopac.Job).  На современном компьютере можно запустить десятки миллионов новых *jobs* за секунду, а, так как
*job*  занимает очень маленький объем памяти, начиная с десятков байт, программа может содержать 
миллионы запущенных *jobs* на современном железе.  (Естественно, в любой момент времени большинство этих *jobs* приостановлены, ведь  даже последние процессоры 
имеют все же ограниченный набор ядер.)  Поэтому, используя Hopac, вы можете запустить новый *job* в ситуации, когда даже подумать не могли бы о запуске настоящей тяжеловесной нити.

Другой аспект - в Hopac примитивы, передающие сообщения для координации и коммуникации между отдельными *jobs*,
являются избирательными, сихронными и легковесными объектами высшего порядка первого класса. 
Это  *channels* (каналы), представленый типом `Ch<'x>`[*](http://hopac.github.io/Hopac/Hopac.html#def:type%20Hopac.Ch),
и *alternatives* (варианты), представленный типом `Alt<'x>`[*](http://hopac.github.io/Hopac/Hopac.html#def:type%20Hopac.Alt).  
Надеюсь, у Вас уже потекли слюнки! Сейчас кое что еще.

* **Объект первого класса** означает, что *channels* и *alternatives* - это обычные значения, которым можно присвоить имя. Их можно 
  передать в функцию и вернуть из нее или передать из одной *job* в другую.
* **Высшего порядка** означает, что из примитивных *alternatives*, путем комбинирования и
  расширения[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%5E=%3E)
  разработчиком своими процедурами, можно строить[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Alt.prepareJob) более сложные, 
  комлексные[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Alt.withNackJob) *alternatives*,
  в том числе и конкурентные клиент-серверные протоколы.
* **Изберательный** означает поддержку возможности выбора варианта исполнения[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Alt.choose)
  или разделения потока событий между *alternatives*.  Альтернативу, например, можно определить так, что отдавать[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Ch.give)
  сообщение или принимать[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Ch.take) сообщение от другой *job* она будет 
  в зависимости от ситуации, возникшей во время исполнения.
* **Синхронный** означает, что вместо использования очереди для передачи сообщения из одной *job* в другую, *jobs*
  могут обмениваться сообщениями с помощью *rendezvous* (рандеву). Две *jobs* могут синхронно, напрямую передать сообщение друг другу.
* **Легковесный** означает, что создание синхронного *channel* занимает очень мало времени (одно отведение памяти) 
  и *channel* требует очень небольшого количества памяти для работы.

Все это сводится к тому, что Hopac в основе своей предоставляет определенного рода язык для выражения 
параллельного потока управления.

### Потенциальный сферы использования Hopac

Hopac - это не панацея на все случаи.  Как сказано выше, основа Hopac - легковесные нити, называемые *jobs*, 
и гибкая легкая синхронная передача сообщений через *channels* (и через другие примитивы сообщений).  Hopac спроектирован и 
оптимизирован для масштабирования большого нарастающего количества относительно независимых небольших процессов. Его можно рассматривать
как форму *параллелизма данных*, в которой данные - это программные элементы, реализованные в виде *jobs* и примитивов коммуникации.

Проблемные области, которые более или менее естественно могут быть выражены в терминах большого количества простых
нитей (одна и более нити на программный элемент) и сообщений между ними - это то место, где Hopac может блистать в плане
производительности и легкости написания программ. Системы параллельной сборки проектов, моделирования процессов, web-серверы и GUI -
примеры подобных областей.

С другой стороны, там где задача выражается в малом количестве независимых нитей, вы вряд ли сможете получить
преимущества от использования Hopac. Например, если вашу задачу можно представить как  фиксированный набор конвейеров обработки
данных с несколькими нитями, которые ожидают прихода сообщений, то системы, подобные [LMAX Disruptor](http://lmax-exchange.github.io/disruptor/),
покажут, скорее всего, лучшую производительность, нежели Hopac.

Несколько вступительных примеров
---------------------------------

Вместо того, чтобы скрупулезно изучать примитивы Hopac, давайте сначала пробежимся по нескольким примерам. 
Эти примеры просты и вы можете быстро их просмотреть. Мы чуть позже детально рассмотрим примитивы, используемые в них.

### Пример: Изменяемая ячейка памяти

В книге
[Concurrent Programming in ML](http://www.cambridge.org/us/academic/subjects/computer-science/distributed-networked-and-mobile-computing/concurrent-programming-ml),
[John Reppy](http://people.cs.uchicago.edu/~jhr/) эта задача - первый пример использования каналов
Concurrent ML и зеленых нитей. Конечно на практике никто не будет использовать данную реализацию, поскольку в FSharp уже есть
родная реализация изменяемых ячеек (ref cells), но это хороший пример, иллюстрирующий некоторые основные особенности
модели. Давайте воспроизведем этот пример, используя Hopac.

Вот сигнатура изменяемой ячейки:

```fsharp
type Cell<'a>
val cell: 'a -> Job<Cell<'a>>
val get: Cell<'a> -> Job<'a>
val put: Cell<'a> -> 'a -> Job<unit>
```

Функция `cell` создает
job[*](http://hopac.github.io/Hopac/Hopac.html#def:type%20Hopac.Job), генерерующий новую ячейку.  Функция
`get` создает `job`, возвращающий ее содержимое, а функция `put` - `job`, изменяющий ячейку.

Основная идея реализации заключается в том, что ячейка рассматривается как конкуретный сервер, способный обслуживать запросы на
чтение и запись значения в ячейку. Тип запроса мы определим, используя размеченнное объединение `Request`:

```fsharp
type Request<'a> =
 | Get
 | Put of 'a
```

Для общения с внешним миром сервер предоставляет два канала - один для приема запросов, 
по другому он посылает ответы. Тип `Cell` - запись из этих 
каналов[*](http://hopac.github.io/Hopac/Hopac.html#def:type%20Hopac.Ch):

```fsharp
type Cell<'a> = {
  reqCh: Ch<Request<'a>>
  replyCh: Ch<'a>
}
```

Функция `put` - `job`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Hopac.job), 
отдающий (give[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Ch.give)) 
запрос на сервер через канал запроса:

```fsharp
let put (c: Cell<'a>) (x: 'a) : Job<unit> = job {
  return! Ch.give c.reqCh (Put x)
}
```

Функция `get` отправляет запрос `Get` на канал запроса сервер и забирает
ответ (take[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Ch.take))
с канала ответа:

```fsharp
let get (c: Cell<'a>) : Job<'a> = job {
  do! Ch.give c.reqCh Get
  return! Ch.take c.replyCh
}
```

И последняя необходимая функция `cell` действительно создает каналы и запускает
(start[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Job.start))
конкуррентный сервер:

```fsharp
let cell (x: 'a) : Job<Cell<'a>> = job {
  let c = {reqCh = Ch (); replyCh = Ch ()}
  let rec server x = job {
        let! req = Ch.take c.reqCh
        match req with
         | Get ->
           do! Ch.give c.replyCh x
           return! server x
         | Put x ->
           return! server x
      }
  do! Job.start (server x)
  return c
}
```

Конкуретный сервер - это `job`, который в бесконечном рекурсивном цикле забирает сообщения из 
канала запроса и, в случае `Get` сообщения , возвращает значение своего параметра через канал ответа,
перезапуская себя с текущим параметром, или, в случае
`Put` сообщения, запускает себя с новым параметром.

Вот пример вывода, полученный в интерактивной сессии:

```fsharp
> let c = run (cell 1) ;;
val c : Cell<int> = ...
> run (get c) ;;
val it : int = 1
> run (put c 2) ;;
val it : unit = ()
> run (get c) ;;
val it : int = 2
```

#### Сборка мусора

Запуская предыдущий пример, вы могли заинтересоваться тем, что же происходит с
`job`, запущенной внутри ячейки. Будет ли она когда нибудь остановлена? Действительно,
один из важных моментов для понимания - это то, что Hopac `job`s и `channel`s всего
навсего обычные объекты .Net и могут быть собраны сборщиком мусора. К тому же, они
не содержат ресурсов, требующих непосредственного управления жизненным циклом. Это
отличает их, например, от  [MailboxProcessor]'а(http://msdn.microsoft.com/en-us/library/ee370357.aspx),
требущего прямого освобождения через IDisposable(http://msdn.microsoft.com/en-us/library/system.idisposable.aspx)
интерфейс. `Job`, ожидающий сообщений
от канала, не имеющего более ссылок, может быть (и будет) собран сборщиком. Лишь
`job`ы, непосредственно владеющие ресурсами, троебующими освобождения, должны поддерживать
протокол уничтожения для уверенного их, ресурсов, освобождения. 

Посмотрите на такой тест:

```fsharp
> GC.GetTotalMemory true ;;
val it : int64 = 39784152L
> let cs = ref (List.init 100000 <| fun i -> run (cell i)) ;;
// ...
> GC.GetTotalMemory true ;;
val it : int64 = 66296336L
> cs := [] ;;
val it : unit = ()
> GC.GetTotalMemory true ;;
val it : int64 = 39950064L
```

Он показывает, что после сборки списка, все ячейки были тоже собраны. (Пример
использует список во избежание возможности войти в последнее поколение кучи больших объектов (LOH), так
как .Net runtime сложно заставить его (LOH) собирать)

#### Об используемой памяти

Другой важной особенностью Hopac jobs и синхронных каналов, является прогнозируемое выделение
памяти под них, а именно, **m** jobs, обменивающихся сообщениями через **n** каналов, будут 
использовать **&Theta;(m + n)** памяти.  

Возможно это очевидно, но многие конкурентные системы, например [Erlang](http://www.erlang.org/) 
и F#' [MailboxProcessor](http://msdn.microsoft.com/en-us/library/ee370357.aspx), построены поверх
асинхронных примитивов передачи сообщений и собирают приходящие сообщения в очереди, если
подписчик обрабатывает их медленнее, чем производит публикатор. Синхронные каналы работают
по другому. У них нет буфера сообщений, поэтому если публикатор пытается отправить сообщение, которое подписчик
еще не может обработать, публикатор останавливается и ждет освобождения подписчика. Синхронные каналы
предоставляют механизм простого рандеву (*simple rendezvous*), который не является по сути буфером для сообщения,
а скорее механизмом управления исполнением программы, подобно вызову процедуры, невозвращающей результат. Такое свойство
упрощает понимание поведения конкурентных програм.

Конечно, граница **&Theta;(m + n)** не учитывает место занимамаемое собственными данными `job`, кроме, собственно, самих каналов.

#### Способы описания

Существует два способа описания job в Hopac. Первый использует монадную нотацию `job`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Hopac.job),
как и было показано в прошлом примере. Второй напрямую использут набор монадных
комбинаторов, `result`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Job.result) и 
bind, `>>=`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%3E%3E=), которые нотация скрывает под абстракцией.
Лично я предпочитаю использовать комбинаторы и только иногда переходить на монадную нотацию. У меня есть для этого 
ряд причин:

* использование комбинаторов напрямую порождает более лаконичный код
* мне чаще проще понять код, написанный с использованием комбинаторов
* есть много часто используемых комбинаторов, например `>>-`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%3E%3E-)
  и
  `>>-.`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%3E%3E-.),
  не имеющих аналогов в монадной нотации, и их использование ускоряет программу.  
* используя комбинаторы напрямуя, я часто могу избежать необязательных операций
  `delay`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Job.delay),
  которые в монадной нотации введены из соображений безопасности.

Я боюсь, что полное объяснение моих побудетельных мотивов потребует ни одного листа текста, а есть
много гораздо более интересных вещей в Hopac, о которых хотелось бы рассказать. В дальнейшем я буду
использовать тот стиль написания программ на Hopac, который мне больше нравится. Если вы предпочитаете монадную нотацию,
вы можете попробовать переписать примеры с ее помощью - это будет хорошим упражнением в изучении библиотеки. Если вы решитесь на это,
обратите внимание на особенности хвостовой рекурсии  оригинальных примеров.

Перед тем как мы продолжим, вот пример обновляемой ячейки, записанный в стиле монадных комбинаторов:
 
```fsharp
let put (c: Cell<'a>) (x: 'a) : Job<unit> =
  Ch.give c.reqCh (Put x)

let get (c: Cell<'a>) : Job<'a> = Ch.give c.reqCh Get >>=. Ch.take c.replyCh

let create (x: 'a) : Job<Cell<'a>> = Job.delay <| fun () ->
  let c = {reqCh = Ch (); replyCh = Ch ()}
  let rec server x =
    Ch.take c.reqCh >>= function
     | Get ->
       Ch.give c.replyCh x >>=. server x
     | Put x -> server x       
  Job.start (server x) >>-. c
```

Как вы видете, я использовал 
`delay`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Job.delay)
лишь один раз. А если вы подсчитаете количество слов и строк, вы обнаружите, что код более лаконичен. Лично я нахожу код, 
написанный с использованием монадных комбинаторов, как минимум, настолько же читаемым, как и код в стиле монадной нотации.

В дополнение ук монадным комбинаторам, Hopac предоставляет набор операторов дл многих примитивов передачи сообщений.
Некоторые из них - это обычный upcast в цепочке наследования и может быть пропуч\щен полностью или заменен на
родную операцию  F# `upcast`. Более того, Hopac предоставляет так же набор кратких операций для часто используемых
примитивов передачи сообщений и основных методов программирования. Используя эти операторы и опустив необязательные декларации типов,
пример с ячейкой можно записать так:

```fsharp
let put c x = c.reqCh *<- Put x

let get c = c.reqCh *<- Get >>=. c.replyCh

let create x = Job.delay <| fun () ->
  let c = {reqCh = Ch (); replyCh = Ch ()}
  Job.iterateServer x <| fun x ->
        c.reqCh >>= function
          | Get -> c.replyCh *<- x >>-. x
          | Put x -> Job.result x
  >>-. c
```

Однако в этом документе мы будем использовать декларации типов, что бы увидеть их без компиляции кода и стараться
избегать использования специфичных операторов для концептуально синтаксически более ясного кода. Например,
хотя можно вять сообщения из канала просто через bind, и это допустимо в рабочем коде, мы постараемся не делать этого
в примерах.

**Упражнение:** Как альтернатива использования канала ответов, можно отводить свежий канал для ответа в каждой операции get. 
Попробуйте изменить исходный текст, используя эту технику. Подумайте о плюсах и минусах в производительности у такого подхода.

### Пример: Ячейка памяти с использованием альтернатив

Изменяемая ячейка памяти в предыдущем разделе была построена с использованием 
только каналов и *jobs*. Чтобы позволить использовать два типа запросов, на чтение и запись,
ьы использовали `union` *Request* и сопоставление с образцом. В этом разделе посмотрим на другой способ
реализации ячейки памяти с использованием избирательной коммуникации (selective communication).

Напомню вам сигнатуру, которую мы хотим реализовать:

```fsharp
type Cell<'a>
val cell: 'a -> Job<Cell<'a>>
val get: Cell<'a> -> Job<'a>
val put: Cell<'a> -> 'a -> Job<unit>
```

Идея оприрается на реализацию серверного цикла, который создает альтернативу,
способную принять как новое значение для опереции записи, так и отдать текущее значение
для операции чтения. Тип ячейки состоит из двух каналов [*](http://hopac.github.io/Hopac/Hopac.html#def:type%20Hopac.Ch):

```fsharp
type Cell<'a> = {
  getCh: Ch<'a>
  putCh: Ch<'a>
}
```
Операция чтения после этого просто забирает [*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Ch.take)
значение из `getCh` канала сервера:

```fsharp
let get (c: Cell<'a>) : Job<'a> = Ch.take c.getCh
```
А операция записи [*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Ch.give)
отдает значение серверу через канал `putCh`:

```fsharp
let put (c: Cell<'a>) (x: 'a) : Job<unit> = Ch.give c.putCh x
```

Конструктор ячейки создает каналы и запускает серверный цикл:

```fsharp
let cell x = Job.delay <| fun () ->
  let c = {getCh = Ch (); putCh = Ch ()}
  let rec server x =
    Alt.choose [Ch.take c.putCh   ^=> fun x -> server x
                Ch.give c.getCh x ^=> fun () -> server x]
  Job.start (server x) >>-. c
```

В серверном цикле эта реализация использует избирательную[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Alt.choose)
коммунникацию. Выбор представляется из двух примитивный альтернатив (`alternatives`)[*](http://hopac.github.io/Hopac/Hopac.html#def:type%20Hopac.Alt):

* Первая альтернатива забирает[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Ch.take)
  от клиенте значение из канала `putCh`, после чего[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%5E=%3E)
  продолжает цикл.
* Вторая альтернатива отдает[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Ch.take)
  текущее значение через `getCh` канал клиенту, после чего[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%5E=%3E)
  продолжает цикл.

Получается, что сервер предлагает клиенту контракт для выбора операции. Из двух возможных
выборов будет исполнена та, которая будет предложена первой. Следущая будет работать с текущим значением,
так как значение уйдет в замыкание. 

Такой шаблон переноса некоторого значениея между итеррациями серверного цикла настолько общий,
что вынесен в специальный комбинатор `iterate`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Job.iterate).
Использование `iterate`:

```fsharp
let cell x = Job.delay <| fun () ->
  let c = {getCh = Ch (); putCh = Ch ()}
  Job.server << Job.iterate x <| fun x ->
        Alt.choose [Ch.take c.putCh
                    Ch.give c.getCh x ^->. x]
  >>-. c
```

Эта реализация использует так же функцию
`Job.server`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Job.server)
вместо
`Job.start`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Job.start).
`Job.server` оптимизирует использование, опираясь на знание того, что `job` является бесконечным циклом.

Вы можете убедиться, используя F# interactive, что такая реализация ячейки работает точно так же, как и 
предыдущий вариант.

Обратите внимание на программу-бенчмарк [Cell](https://github.com/Hopac/Hopac/tree/master/Benchmarks/Cell),
использующую эту реализацию, которая создает большое количество `job` и ячеек и случайным образом работате с ними.
Это хорошо подтверждает легковесность Hopac, описанную в самом начале статьи.

**Упражнение**: Возможно использование двух канадов излишне для реализации такого простого
протокола и достаточно одного двунаправленного канала (это разрешено библиотекой). `Job` посылвет сам себе сообщение
в дной синхронной операции. Исследуйте, что может пойти не так при использовании одного канала вместо двух.
Подсказка: Рассмотрите ситуацию с несколькими клиентами.

### Пример: Kismet

Пример с изменяемой ячейкой возможно выглядеть слегка далеким от практики. Ее серверный `job` 
делает совсем немного, поЭтому вряд ли кто либо из вас захочет запускать для этого отдельную нить, пусть 
даже настолько легкую, насколько это вообще возможно. Давайте набросаем 
более убедительный пример, хотя то же далекий от реальности.


[UnrealScript](http://en.wikipedia.org/wiki/UnrealScript) - скриптовый язык 
[Unreal Engine](http://en.wikipedia.org/wiki/Unreal_Engine), используемый для создания игра.

[Kismet](http://en.wikipedia.org/wiki/UnrealEd#Kismet)  - инструмент, позволяющий создавать программы на 
UnrealScript, использую визуальный интерфейс. Работая с Kismet, дизайнер создает игру, используя блоки, написанные программистами.
Эти блоки можно рассмотреть как черные ящики, имеющие несколько входов и выходов и обладающие поведением, отражающим
входы на выходы.

В Wikipedia усть страничка, [UnrealEd](http://en.wikipedia.org/wiki/UnrealEd), где можно увидеть
скриншот простой системы, построенной с помощью Kismet.  Найдите минутку и посмотрите на этот скрниншот:
[Roboblitz](http://upload.wikimedia.org/wikipedia/en/e/e6/Kismet_Roboblitz.PNG).
Как вы можете видеть, есть базовые преиспользуемые блоки, такие как `Bool`, `Compare Bool`,
`Delay`, and `Matinee`, у которых есть входы, выходы и какое то поведение.

Kismet, UnrealScript and the Unreal Engine, в основе, состоят из компонентов и семантики, спроектированной 
для разработки игр. Фактически, я (автор) никогда по настоящему на программировал на UnrealScript и не использовалKismet, 
но мне было крайне любопытно, как создавать подобные черные ящики? Можно ли построить что либо похожее,
опираясь на возможности Hopac?

Сначала рассмотриьм блок `Compare Bool`.  Глядя на скриншот, можно сделат обоснованное предлоположение,
что есть вход `In` и два выхода - `True` и `False`, и еще чтение некоей `Bool` переменной. Выглядит так, что 
при получении входа блок выставляет один зи сигналов `True` или `False` в зависимости от значения этой переменной.
Что то вроде этого может быть выражено такой Hopac `job`:

```fsharp
let CompareBool (comparand: ref<bool>)
                (input: Alt<'x>)
                (onTrue: 'x -> Job<unit>)
                (onFalse: 'x -> Job<unit>) : Job<unit> =
  input >>= fun x ->
  if !comparand then onTrue x else onFalse x
```

Функция `CompareBool` создает `job`, который сначала привязывается к входной альтернативе `input`,
а потом вызывает одну из `job's` `onTrue` или `onFalse` в зависимости от значения `comparand`.  
Как видите, `CompareBool` не задумывается о типе значения на входе, а копирует его в зависимости от 
`comparand` на тот или иной выход.

Теперь рассмотри блок `Delay`.  Сделав еще одно предположение и слегка упрощая, у него есть два входных события - 
`Start` и `Stop` (Я оставлю событие `Pause` как упражнение для читателя) и два выходных события - 
`Finished` и `Aborted`, а так же некоторое значение времени `Duration`. Выглядит так, что получив событие старт,
блок запускает таймер на время `Duration`, по истечении которого вызывает событие `Finished`. Но, если во время активности таймера поступит сигнал `Stop`, то вызвано будет
событие `Aborted`. Вот как это выглядит в виде Hopac `job`:

```fsharp
let Delay (duration: ref<TimeSpan>)
          (start: Alt<'x>)
          (stop: Alt<'y>)
          (finished: 'x -> Job<unit>)
          (aborted: 'y -> Job<unit>) : Job<unit> =
  start >>= fun x ->
  Alt.choose [stop                ^=> fun y -> aborted y
              timeOut (!duration) ^=> fun () -> finished x]
```

Функция `Delay` создает `job`, который привязывается к альтернативе `start`. Потом 
выбирает[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Alt.choose) один из двух вариантов продолжения.
Первый - это получение `stop` альтернативы, после чего выбор заканчивается и занчение полученной из альтернативы `stop`
передается на выход `aborted`.  Вторая альтернатива запускает 
`timeOut`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Hopac.timeOut) альтернативу с параметром времени,
равным текущему значению `duration` и в случае его завершения значение, переданное на старте отправляется на выход `finished`.
Побеждает событие выбора, произошедшеево время исполнения первым.

Эти строительные блоки выглядят обманчиво простыми. Важно отметить, что эта протота опираетс яна способность
Hopac's jobs ожидать альтернативу, находясь вв блокировке. Без подобной возможности ожидание таймаута и 
сброс его превратилось бы в довольно сложную задачу.

Предыдущие примеры - только два из необходимых строительных блоков. Представим, что у нас есть все необходимые блоки, созданные в одинаковом стиле,
как тогда перевести сценарий их соединения, показанный на скриншоте, в код?
Вот маленький набросок того, как это могло бы выглядеть:

```fsharp
let ch_1 = Ch ()
let ch_2 = Ch ()
let ch_3 = Ch ()
// ...
let bMoved = ref false
// ...
do! CompareBool bMoved
                (Ch.take ch_1)
                (Ch.give ch_2)
                (fun _ -> Job.unit ())
    |> Job.forever |> Job.server
do! Delay (ref (TimeSpan.FromSeconds 3.14))
          (Ch.take ch_2)
          (Alt.never ())
          (Ch.give ch_3)
          (fun _ -> Job.unit ())
    |> Job.forever |> Job.server
// ...
```

Код инициализации создает общие каналы и переменные. Потом создаются соответствующие `job` с правильно соединенными входами и 
выходами, каждый (`job`) из которых запускается как бесконечно работающий сервер.

Конесно, игры часто используют свое собственное представление времени, отличное от часов на вашей руке,
поэтому в реальной игре реализацию `Delay` придется делать по другому (но и это можно реализовать в рамках Hopac).
Да и использование переменных в виде изменяемых ссылок на ячейки памяти слегка наивно, но, надеюсь, этот пример
даст вам пищу для размышлений. 


Запуск и ожидание `Jobs`
-----------------------------

Посмотрев вступительные примеры, вернемся на шаг назад и немного поиграем с `job's`.
Djn ghjcnjq `job`, который в цикле сначала спит [*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Hopac.timeOut)
в течении секунды, а потом печатает сообщение:

```fsharp
let hello what = job {
  for i=1 to 3 do
    do! timeOut (TimeSpan.FromSeconds 1.0)
    do printfn "%s" what
}
```

Запустим двеа таких `jobs` с разницей в полсекунды:

```fsharp
> run <| job {
  do! Job.start (hello "Hello, from a job!")
  do! timeOut (TimeSpan.FromSeconds 0.5)
  do! Job.start (hello "Hello, from another job!")
} ;;
val it : unit = ()
> Hello, from a job!
Hello, from another job!
Hello, from a job!
Hello, from another job!
Hello, from a job!
Hello, from another job!
```

К несчастью, проограмма из этого примера вернется немедленно, а `job` останутся запущенными в фоне.
`Job.start`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Job.start)
примитив напрямую не предоставляет никакого способа дождаться окончания запущенных `job`.  
Это сделано намеренно, так как самое распространненое поведение `job` - запуститься и никогда не останавливаться. 
`Promise`[*](http://hopac.github.io/Hopac/Hopac.html#def:type%20Hopac.Promise)
позволяет родительскому job дождаться завершения потомка:

```fsharp
> run <| job {
  let! j1 = Promise.start (hello "Hello, from a job!")
  do! timeOut (TimeSpan.FromSeconds 0.5)
  let! j2 = Promise.start (hello "Hello, from another job!")
  do! Promise.read j1
  do! Promise.read j2
} ;;
Hello, from a job!
Hello, from another job!
Hello, from a job!
Hello, from another job!
Hello, from a job!
Hello, from another job!
val it : unit = ()
>
```

Теперь программа явно ждет завершения своих потомков и вывод программы стал правильным. 
Есть еще одна неудачная вещь в этой программе. Promises читаются строго последовательно.
В следующей программе это не имеет особого смысла, но хорошо демонстрирует гибкость Hopac,
позволяющую избежать упорядоченной зависимости  с использованием 
избирательной[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Alt.choose)
связи, предоставляемой механизмом альтернатив[*](http://hopac.github.io/Hopac/Hopac.html#def:type%20Hopac.Alt):

```fsharp
> run <| job {
  let! j1 = Promise.start (hello "Hello, from a job!")
  do! timeOut (TimeSpan.FromSeconds 0.5)
  let! j2 = Promise.start (hello "Hello, from another job!")
  do! Alt.choose
       [Promise.read j1 ^=> fun () ->
          printfn "First job finished first."
          Promise.read j2
        Promise.read j2 ^=> fun () ->
          printfn "Second job finished first."
          Promise.read j1]
} ;;
Hello, from a job!
Hello, from another job!
Hello, from a job!
Hello, from another job!
Hello, from a job!
First job finished first.
Hello, from another job!
val it : unit = ()
```
Если вы запустите последнюю программу, то увидите, что сообщение `First job
finished first.` напечатается примерно на полсекундыi раньше , чем последнее сообщение `Hello, from
another job!` перед тем как программа завершится и F# interactive напечатает выведенный тип.

Работа с многими `job` на таком уровне довольно обременительна, поэтому Hopac предоставляет функции
`Job.conCollect`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Job.conCollect)
и
`Job.conIgnore`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Job.conIgnore)
для запуска и ожидания списков `jobs`.  В нашем случае результат исполнения `jobs` нам не интересен,
поэтому воспользумся функцией `Job.conIgnore`:

```fsharp
> [timeOut (TimeSpan.FromSeconds 0.0) >>-. hello "Hello, from first job!" ;
   timeOut (TimeSpan.FromSeconds 0.3) >>-. hello "Hello, from second job!" ;
   timeOut (TimeSpan.FromSeconds 0.6) >>-. hello "Hello, from third job"]
|> Job.conIgnore |> run ;;
Hello, from first job!
Hello, from second job!
Hello, from third job
Hello, from first job!
Hello, from second job!
Hello, from third job
Hello, from first job!
Hello, from second job!
Hello, from third job
val it : unit = ()
>
```
Эта программа запускает три параллельных `jobs`, печатающих сообщения с разницей в 0.3 секунды и ожидает их окончания.

### Fork-Join Parallelism

Только что мы научились нескольким способам запуска и ожидания `jobs`, но между индивидуальными задачами 
в примерах не было никакого обмена данными. Тем временем, сила Hopac в предоставлении высокоуровневых примитивов коммуникации
между конкурентными `jobs`, реализующих в том числе и стиль программирования, основанный на запуске и объединении результатов
нескольких потоков, известном как *fork-join
parallelism* и являющимся распространенной парадигмой многих параллельных алгоритмов.

Одна из целей Hopac - это ускорение вычислений на многоядерных машинах. 
Примитивы, такие как каналы, `jobs` (threads в CML) и альтернативы (events в CML), вдохновленные Concurrent ML, первично разработаны
для конкуррентного программирования в различных потоках исполнения. Для достижения ускорения от распараллеливания наличие независимых
потоков может быть не настолько существенным, как создание нового тпотока исполнения для каждого индивидуального действия, когда сразу несколько `jobs`
могут выполняться параллельно, то есть есть некий пул `jobs`, которые готовы к исполнению и процессы могут исполнять любые из них.


#### Фунция Фибоначчи и параллелизм

Начнем с такой наивной реализации функции Фибоначчи в виде `job`:

```fsharp
let rec fib n = Job.delay <| fun () ->
  if n < 2L then
    Job.result n
  else
    fib (n-2L) <&> fib (n-1L) >>- fun (x, y) ->
    x + y
```

Эта реализация выполнена с помощью комбинаторов
`<&>`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%3C%26%3E)
и
`>>-`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%3E%3E-)
чей смысл может быть выражен в терминах
`result`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Job.result)
и
`>>=`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%3E%3E=)
следующим образом:

```fsharp
let (<&>) xJ yJ = xJ >>= fun x -> yJ >>= fun y -> result (x, y)
let (>>-) xJ x2y = xJ >>= fun x -> result (x2y x)
```

Заметьте, что семантика `<&>` послностью последовательна и такая `fib` job не использует паралеллизма.

Запустим эту реализацию в F# Interactive:

```fsharp
> run (fib 38L) ;;
val it : int64 = 39088169L
```

Если вы действительно исполнили код, то заметили, что получение результата заняло заметное время. Дейсьвительно,
это экстремально неэффективный алгоритм.

Давайте сделаем небольшое изменение, заменив последовательный комбинатор
`<&>`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%3C%26%3E)
на параллельный
`<*>`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%3C%2A%3E):

```fsharp
let rec fib n = Job.delay <| fun () ->
  if n < 2L then
    Job.result n
  else
    fib (n-2L) <*> fib (n-1L) >>- fun (x, y) ->
    x + y
```

Паралелльный комбинатор `<*>` позволяет запускать 2 jobs как последовательно, так и параллельно, в зависимости от текущих возможностей.
Иными словами, приналичии возможности и свободного потока исполнение будет параллельным или последовательным, любым. 
Для того, чтобы это было возможно, нужно, чтобы `jobs` безопасно могли работать одновременно. В примере это так, 
но вот пример вызывающий взаимную блокировку: 

```fsharp
let notSafe = Job.delay <| fun () ->
  let c = Ch ()
  Ch.take c <*> Ch.give c ()
```

Проблема в том, что этот `job` одновременно использует и 
`take`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Ch.take)
и
`give`[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Ch.give),
и нет гарантии, что они будут выполняться в отдельных нитях, а одна нить не может коммуницировать сама с собой с помощью
этих операций по одному каналу. Поэтому первая операция заблокируется в ожэидании второй, которая не сможет быть выполнена.

##### Speedup?

Did you already try to run the parallel version of the naive Fibonacci function
in the F# interactive?  If you did, the behavior may have not been what you'd
expect&mdash;that the parallel version would run about `N` times faster than the
sequential version where `N` is the number of processor cores your machine has.
Now, there are a number of reasons for this and one of the possible reasons is
that, by default, .Net uses single-threaded workstation garbage collection.  If
garbage collection is single-threaded, it becomes a sequential bottleneck and an
application cannot possibly scale.  So, you need to make sure that you are using
multi-threaded server garbage collection.  See
[&lt;gcServer&gt; Element](http://msdn.microsoft.com/en-us/library/ms229357%28v=vs.110%29.aspx)
for some details.  I have modified the configuration files of the F# tools on my
machine to use the server garbage collection.  I also use the 64-bit version of
F# interactive and run on 64-bit machines.  Once you've made the necessary
adjustments to the tool configurations, you should see the expected speedup from
the parallel version.

##### About the Fibonacci Example

This example is inspired by the parallel Fibonacci function used traditionally
as a Cilk programming example.  See
[The Implementation of the Cilk-5 Multithreaded Language](http://supertech.csail.mit.edu/papers/cilk5.pdf)
for a representative example.  Basically, the naive, recursive, exponential time
Fibonacci algorithm is used.  Parallelized versions simply run recursive calls
in parallel.

Like is often the case with cute programming examples, this is actually an
extremely inefficient algorithm for computing Fibonacci numbers and that seems
to be a recurring source of confusion.  Indeed, the naively parallelized version
of the Fibonacci function is still hopelessly inefficient, because the amount of
work done in each parallel job is an order of magnitude smaller than the
overhead costs of starting parallel jobs.

The main reason for using the Fibonacci function as an example is that it is a
simple example for introducing the concept of optional parallel execution, which
is employed by the `<*>` combinator.  The parallel Fibonacci function is also
useful and instructive as a benchmark for measuring the overhead costs of
starting, running and retrieving the results of parallel jobs.  Indeed, there is
a
[benchmark program](https://github.com/Hopac/Hopac/tree/master/Benchmarks/Fibonacci)
based on the parallel Fibonacci function.

**Exercise:** Write a basic sequential Fibonacci function (not a job) and time
it.  Then change the parallelized version of the Fibonacci function to call the
sequential function when the **n** is smaller than some constant.  Try to find a
constant after which the new parallelized version actually gives a speedup on
the order of the number of cores on your machine.

#### Parallel Merge Sort

Let's consider a bit more realistic example of fork-join parallelism: a parallel
[merge sort](http://en.wikipedia.org/wiki/Merge_sort).  This example is still a
bit of toy, because the idea here isn't to show how to make the fastest merge
sort, but rather to demonstrate fork-join parallelism.

The two building blocks of merge sort are the functions `split` and `merge`.
The `split` function simply splits the given input sequence into two halves.
The `merge` function, on the other hand, merges two sequences into a new sorted
sequence containing the elements of both of the given sequences.

Here is a simple implementation of `split`:

```fsharp
let split xs =
  let rec loop xs ys zs =
    match xs with
     | []    -> (ys, zs)
     | x::xs -> loop xs (x::zs) ys
  loop xs [] []
```

And here is a simple implementation of `merge`:

```fsharp
let merge xs ys =
  let rec loop xs ys zs =
    match (xs, ys) with
     | ([], ys)       -> List.rev zs @ ys
     | (xs, [])       -> List.rev zs @ xs
     | (x::xs, y::ys) ->
       if x <= y
       then loop xs (y::ys) (x::zs)
       else loop (x::xs) ys (y::zs)
  loop xs ys []
```

It is left as an exercise for the reader to implement `merge` in a more
efficient form.

Merge sort then simply recursively splits, sorts and then merges the lists:

```fsharp
let rec mergeSort xs =
  match split xs with
   | ([], ys) -> ys
   | (xs, []) -> xs
   | (xs, ys) -> merge (mergeSort xs) (mergeSort ys)
```

We can now test that our `mergeSort` works:

```fsharp
> mergeSort [3; 1; 4; 1; 5; 9; 2] ;;
val it : int list = [1; 1; 2; 3; 4; 5; 9]
```

This is not a very good implementation of merge sort, but it should be easy
enough to understand&mdash;perhaps even without having seen merge sort before.
One particular problem with this implementation is that it is not
[stable](http://en.wikipedia.org/wiki/Sorting_algorithm#Stability).  We'll leave
it as an exercise for the reader to tune this example into something more
realistic.

Let's then write a fork-join parallel version of merge sort:

```fsharp
let rec mergeSortJob xs = Job.delay <| fun () ->
  match split xs with
   | ([], ys) -> Job.result ys
   | (xs, []) -> Job.result xs
   | (xs, ys) ->
     mergeSortJob xs <*> mergeSortJob ys >>- fun (xs, ys) ->
     merge xs ys
```

We can also test this version:

```fsharp
> run (mergeSortJob [3; 1; 4; 1; 5; 9; 2]) ;;
val it : int list = [1; 1; 2; 3; 4; 5; 9]
```

Like suggested in an exercise in the previous section, to actually get
speed-ups, the work done in each parallel job needs to be significant compared
to the cost of starting a parallel job.  One way to do this is to use the
sequential version of merge sort when the length of the list becomes shorter
than some threshold.  That threshold then needs to be chosen in such a way that
the work required to sort a list shorter than the threshold is significant
compared to the cost of starting parallel jobs.  In practice, this often means
that you run a few experiments to find a good threshold.  Here is a modified
version of `mergeSortJob` that uses a given threshold:

```fsharp
let mergeSortJob threshold xs = Job.delay <| fun () ->
  assert (threshold > 0)
  let rec mergeSortJob n xs = Job.delay <| fun () ->
    if n < threshold then
      Job.result (mergeSort xs)
    else
      let (xs, ys) = split xs
      let n = n/2
      mergeSortJob n xs <*> mergeSortJob n ys >>- fun (xs, ys) ->
      merge xs ys
  mergeSortJob (List.length xs) xs
```

For simplicity, the above computes the length of the input list just once and
then approximates the lengths of the sub-lists resulting from the split.  It
also assumes that `threshold` is greater than zero.  Using a function like the
above you can experiment, perhaps by writing a simple driver program, to find a
threshold that gives the best speed-ups.

Programming with Alternatives
-----------------------------

The alternative mechanism (events in CML) allows the definition of first-class
synchronous operations.  In previous sections we have already seen some simple
uses of alternatives.  In this section we'll take a closer look at alternatives.

### Just what is an alternative?

There are many ways to characterize alternatives.  Here is one.  An alternative,
`Alt<'x>`, represents the possibility of communicating a value of type `'x` from
one concurrent entity to another.  How that value is computed and when that
value is available are details encapsulated by the alternative.  Alternatives
can be created and combined in many ways allowing alternatives to encapsulate
complex communication protocols.

### Primitive Alternatives

The basic building blocks of alternatives are primitive alternatives provided by
message passing primitives like channels.  For example, the channel primitive
provides the following primitive alternatives:

```fsharp
module Ch =
  val give: Ch<'x> -> 'x -> Alt<unit>
  val take: Ch<'x> -> Alt<'x>
```

The `Ch.give`
[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Ch.give) alternative
represents the possibility of giving a value on a channel to another concurrent
job and the `Ch.take`
[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Ch.take) alternative
represents the possibility of taking a value from another concurrent job on a
channel.

It is important that primitive alternatives such as these only represent the
*possibility* of performing the operations.  As we will see shortly, we can form
a disjunction of alternatives, whether primitive or complex, and commit to
perform exactly one of those alternatives.

### Binding an Alternative

To actually perform an operation made possible by an alternative, one can simply
bind the operation.  So, for example, to indeed offer to give a value on a
channel to another job, a job might run the following code:

```fsharp
do! Ch.give aChannel aValue
```

Likewise, to offer to take a value from another job, the following code could be
run:

```fsharp
let! aValue = Ch.take aChannel
```

Conceptually, binding an alternative operation *instantiates* the given
alternative, *waits until* the alternative becomes *available* and then
*commits* to the alternative and returns the value communicated by the
alternative.  In the instantiation phase the computation encapsulated by the
alternative is started.  In the case of the `Ch.give` operation, for example, it
means that the job basically registers an offer to give a value on a channel.
If the alternative cannot be performed immediately, e.g. no other job has
offered to take a value on the channel, the job is blocked until the alternative
becomes available.

### Choose and after

If all we had was primitive alternatives there would be no point in the whole
mechanism.  What makes alternatives useful is that they can be composed in
various ways.

Let's motivate the introduction of selective communication with a sketch of a
simplistic GUI system.  Suppose our GUI framework represents buttons as simple
concurrent objects that communicate using alternatives:

```fsharp
type Button =
  val Pressed: Alt<unit>
  // ...
```

A simple Yes/No -dialog could then contain two such buttons:

```fsharp
type YesNoDialog =
  val Yes: Button
  val No: Button
  val Show: Job<unit>
  // ...
```

A job could then have a dialogue with the user using a `YesNoDialog` and code
such as:

```fsharp
do! dialog.Show
let! answer = Alt.choose [
       dialog.Yes.Pressed ^=> fun () -> Job.result true
       dialog.No.Pressed  ^=> fun () -> Job.result false
     ]
if answer then
  // Perform action on Yes.
else
  // Perform action on No.
```

The operations `Alt.choose` and `^=>`, called *after*, and also known as *wrap*
in Concurrent ML, have the following signatures:

```fsharp
val choose: seq<#Alt<'x>> -> Alt<'x>
val (^=>): Alt<'x> -> ('x -> #Job<'y>) -> Alt<'y>
```

The `Alt.choose`
[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Alt.choose)
operation forms a disjunction of the sequence of alternatives given to it.  When
such a disjunction is bound, the alternatives involved in the disjunction are
instantiated one-by-one.  Assuming no alternative is immediately available, the
job is blocked, waiting for any one of the alternatives to become available.
When one of the alternatives in the disjunction becomes available, the
alternative is committed to and the other alternatives are canceled.

The after `^=>`
[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%5E=%3E)
operation is somewhat similar to the bind `>>=`
[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%3E%3E=)
operation on jobs and allows one to extend an alternative so that further
operations are performed after the alternative has been committed to.  Similarly
to corresponding operations on jobs, several commonly useful operators, such as
`^->`
[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%5E-%3E)
and `^->.`
[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%5E-%3E.),
are provided in addition to `^=>` on alternatives.

In this case we use the ability to simply map the button messages to a boolean
value for further processing.  We could also just continue processing in the
after operation:

```fsharp
do! Alt.choose [
      dialog.Yes.Pressed ^=> fun () ->
        // Perform action on Yes.
      dialog.No.Pressed  ^=> fun () ->
        // Perform action on No.
    ]
```

Using selective communication in this way feels and works much like using
ordinary conditional statements.

A key point in the types of the `choose` and `^=>` operations is that they
create new alternatives and those alternatives are first-class values just like
the primitive `give` and `take` alternatives on channels.  For the common case
of simply combining just two
alternatives the operation `<|>`
[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%3C%7C%3E)
is provided.  Its semantics can be described as follows:

```fsharp
let (<|>) a1 a2 = choose [a1; a2]
```

The binary choice `<|>` operation can be, and is, implemented internally as a
slightly more efficient special case (avoiding the construction of the
sequence).

It is also worth pointing out that `choose` allows synchronizing on a sequence
of alternatives that is computed dynamically.  Languages that have a special
purpose `select` statement typically only allow the program to synchronize on a
set of events that is specified statically in the program text.

### Prepare

The after combinator `^=>`
[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Infixes.%5E=%3E)
allows post-commit actions to be added to an alternative.  Hopac also provides
the `prepareJob`
[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Alt.prepareJob)
combinator that allows an alternative to be computed at instantiation time.

```fsharp
val prepareJob: (unit -> #Job<#Alt<'x>>) -> Alt<'x>
```

The idea of the `prepareJob` combinator is that it allows one to encapsulate a
protocol for interacting with a concurrent server as an abstract selective
operation.  The way a client and a server typically interact is that the client
sends the server a message and then waits for a reply from the server.  What is
necessary is that the prepare combinator allows one to package the operations of
constructing the message, sending it to the server and then waiting for the
reply in a form that can then be invoked an arbitrary number of times.

Recall in the Kismet sketch it was mentioned that simulations like games often
have their own notion of time and that the wall-clock time provided by `timeOut`
[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Hopac.timeOut)
probably doesn't provide the desired semantics.  A simple game might be designed
to update the simulation of the game world 60 times per second to match with a
60Hz display devices.  Rather than complicate all the calculations done in the
simulation with a variable time step, such a simulation could be advanced in
fixed length time steps or *ticks*.  Simplifying things to a minimum, the main
loop of a game could then look roughly like this:

```fsharp
while !runGame do
  tick ()   // Advance simulation
  render () // Render new view to back buffer
  flip ()   // Wait for display refresh and switch front and back buffers
```

Now, the idea is that the `tick ()` call runs all the simulation logic for one
step and that the simulation is implemented using Hopac jobs.  More specifically
we don't want any simulation code to run after the `tick ()` call returns.  This
is so that the `render ()` call has one consistent view of the world.

A clean way to achieve this is to create a *local scheduler* for running the
Hopac jobs that implement the game logic.  This way, after we've triggered all
the jobs waiting for the next game tick, we can simply wait until the local
scheduler becomes idle.  This means that all the jobs that run under the local
scheduler have become blocked waiting for something&mdash;possibly waiting for a
future game tick.

Enough with the motivation.  We'll represent time using a 64-bit integer type.

```fsharp
type Ticks = int64
```

And we have a variable that holds the current time.


```fsharp
let mutable currentTime : Ticks = 0L
```

Now, to integrate this concept of time with Hopac, we'll have a time server with
which we communicate through a timer request channel.

```fsharp
let timerReqCh : Ch<Ticks * Ch<unit>> = Ch ()
```

Via the channel, a client can send a request to the server to send back a
message on a channel allocated for the request at the specified time.  To send
the request and allocate a new channel for the server's reply, we'll use the
`prepareJob` combinator.  The following `atTime` function creates an alternative
that *encapsulates the whole protocol* for interacting with the time server:

```fsharp
let atTime (atTime: Ticks) : Alt<unit> =
  Alt.prepareJob <| fun () ->
  let replyCh : Ch<unit> = Ch ()
  Ch.send timerReqCh (atTime, replyCh) >>-.
  Ch.take replyCh
```

A detail worth pointing out above is the use of the `Ch.send`
[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Ch.send)
operation to send requests to the server asynchronously.  We already have the
client synchronously taking a reply from the server, so there is no need to have
the client synchronously waiting for the time server to take the request.  Using
`atTime` we can implement the `timeOut` alternative constructor used in the
earlier Kismet example:

```fsharp
let timeOut (afterTicks: Ticks) : Alt<unit> =
  assert (0L <= afterTicks)
  Alt.prepareFun <| fun () ->
  atTime (currentTime + afterTicks)
```

What remains is to implement the time server itself.  Taking advantage of
existing data structures, we'll use a simple dictionary that maps ticks to lists
of reply channels to represent the queue of pending requests:

```fsharp
let requests = Dictionary<Ticks, ResizeArray<Ch<unit>>> ()
```

The time request server takes messages from the request channel and deals with
them, either responding to them immediately or adding them to the queue of
pending requests:

```fsharp
let timeReqServer =
  Ch.take timerReqCh >>= fun (atTime, replyCh) ->
  if currentTime <= atTime then
    Ch.send replyCh ()
  else
    let replyChs =
      match requests.TryGetValue atTime with
       | (true, replyChs) -> replyChs
       | _ ->
         let replyChs = ResizeArray<_>()
         requests.Add (atTime, replyChs)
         replyChs
    replyChs.Add replyCh
    Job.unit ()
```

The time request server also uses an asynchronous send to reply to requests.
This time it is not only an optimization, but required, because it is possible
that within a selective communication the timer request is abandoned and there
is no client waiting for the request.  If the server would try to give the reply
synchronously, the server would be blocked indefinitely.

The above only implements a single iteration of the time request server.  We
need to start the server after we have created the local scheduler:

```fsharp
do! Job.server (Job.forever timeReqServer)
```

One final part of the implementation of time is a routine to advance time.  The
following `tick` job increments the current time and then sends replies to all
the requests at that time:

```fsharp
let tick = Job.delay <| fun () ->
  currentTime <- currentTime + 1L
  match requests.TryGetValue currentTime with
   | (true, replyChs) ->
     requests.Remove currentTime |> ignore
     replyChs
     |> Seq.iterJob (fun replyCh -> Ch.send replyCh ())
   | _ ->
     Job.unit ()
```

That concludes the implementation of the time server itself.

### Negative Acknowledgments

In the previous section the `prepareJob`
[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Alt.prepareJob)
combinator was used to encapsulate the protocol for interacting with the custom
timer server.  This worked because the service provided by the time server is
idempotent.  If a client makes a request to the time server and later aborts the
request, that is, doesn't wait for the server's reply, it causes no harm.
Sometimes things are not that simple and a server needs to know whether client
actually committed to a transaction.  Hopac, like CML, supports this via the
`withNackJob`
[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Alt.withNackJob)
combinator:

```fsharp
val withNackJob: (Promise<unit> -> #Job<#Alt<'x>>) -> Alt<'x>
```

The `withNackJob` combinator is like `prepareJob` in that it allows an
alternative to be computed at instantiation time.  Additionally, `withNackJob`
creates a *negative acknowledgment alternative*, which is actually represented
as a promise, that it gives to the encapsulated alternative constructor.  If the
constructed alternative is ultimately not committed to, the negative
acknowledgment alternative becomes available.  Consider the following example:

```fsharp
let verbose alt = Alt.withNackJob <| fun nack ->
  printf "Instantiated and "
  Job.start (nack >>- fun () -> printfn "aborted.") >>-.
  alt ^-> fun x -> printfn "committed to." ; x
```

The above implements an alternative constructor that simply prints out what
happens.  Let's consider three interactions using a `verbose` alternative.

```fsharp
> run <| Alt.choose [verbose <| Alt.always 1; Alt.always 2] ;;
Instantiated and committed to.
val it : int = 1
```

In the first case above, a verbose alternative is instantiated and committed to.
The negative acknowledgment is created, but does not become enabled.

```fsharp
> run <| Alt.choose [verbose <| Alt.never (); Alt.always 2] ;;
Instantiated and aborted.
val it : int = 2
```

In the second case above, a verbose alternative is instantiated and aborted as
the second alternative is committed to.

```fsharp
> run <| Alt.choose [Alt.always 1; verbose <| Alt.always 2] ;;
val it : int = 1
```

In the third case above, the first alternative is immediately committed to and
no verbose alternative is instantiated.  No code within the verbose alternative
constructor was executed and no negative acknowledgment alternative was created.

Negative acknowledgments can be useful as both as a mechanism that allows one to
cancel expensive operations for performance reasons (when the request is
idempotent) and also as a mechanism that allows one to encapsulate protocols
that otherwise couldn't be properly encapsulated as alternatives.

#### Example: Lock Server

An example that illustrates how `withNackJob`
[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Alt.withNackJob) can
be used to encapsulate a non-idempotent request as an alternative is the
implementation of a *lock server*.  Here is a signature of a lock server:

```fsharp
type Server
type Lock

val start: Job<Server>
val createLock: Server -> Lock
val acquire: Server -> Lock -> Alt<unit>
val release: Server -> Lock -> Job<unit>
```

The idea is that a lock server allows a client to acquire a lock as an
alternative within a selective communication.  A client could, for example, try
to obtain one of several locks and proceed accordingly:

```fsharp
Alt.choose [acquire server lockA ^=> fun () ->
              (* critical section A *)
              release server lockA
            acquire server lockB ^=> fun () ->
              (* critical section B *)
              release server lockB]
```

Or a client could use a timeout to avoid waiting indefinitely for a lock:

```fsharp
Alt.choose [acquire server lock ^=> (* critical section *)
            timeOut duration ^=> (* do something else *)]
```

What is important here is that the `acquire` alternative must work correctly
even in case that the operation is ultimately abandoned by the client.

Let's then describe the lock server example implementation.  We represent a lock
using a unique integer:

```fsharp
type Lock = Lock of int64
```

A request for the lock server is either an `Acquire` or a `Release`:

```fsharp
type Req =
 | Acquire of lock: int64 * replyCh: Ch<unit> * abortAlt: Alt<unit>
 | Release of lock: int64
```

An `Acquire` request passes both a reply channel and an abort alternative for
the server.  The server record just contains an integer for generating new locks
and the request channel:

```fsharp
type Server = {
  mutable unique: int64
  reqCh: Ch<Req>
}
```

The `release`

```fsharp
let release s (Lock lock) = Ch.give s.reqCh (Release lock)
```

and `createLock`

```fsharp
let createLock s =
  Lock (Interlocked.Increment &s.unique)
```

operations are entirely straightforward.  Note that in this example we simply
use an interlocked increment to allocate locks.

Note that this example implementation is not entirely type safe, because two
different lock servers might have locks by the same integer value.  The reason
for leaving the server exposed like this is that it is now easier to run the
code snippets of this example in an interactive session.  As this type safety
issue is not an essential aspect of the example, we leave it as an exercise for
the reader to consider how to plug this typing hole.

The `acquire` operation is where we'll use `withNackJob`:

```fsharp
let acquire s (Lock lock) = Alt.withNackJob <| fun abortAlt ->
  let replyCh = Ch ()
  Ch.send s.reqCh (Acquire (lock, replyCh, abortAlt)) >>-.
  Ch.take replyCh
```

Using `withNackJob` a negative acknowledgment alternative, `abortAlt`, is
created and then a reply channel, `replyCh`, is allocated and a request is
created and sent to the lock server `s`.  An asynchronous `send`
[*](http://hopac.github.io/Hopac/Hopac.html#def:val%20Hopac.Ch.send) operation
is used as there is no point in waiting for the server at this point.  Finally
the alternative of taking the server's reply is returned.

Note that a new pair of a negative acknowledgment alternative and reply channel
is created each time an alternative constructed with `acquire` is instantiated.
This means that one can even try to acquire the same lock multiple times with
the same alternative and it will work correctly:

```fsharp
let acq = acquire s l
do! Alt.choose [acq ^=> /* ... */
                acq ^=> /* ... */]
```

What remains is the implementation of the server itself.  We again make use of
readily available data structures to hold the state, that is pending requests to
active locks, of the lock server.

```fsharp
let start = Job.delay <| fun () ->
  let locks = Dictionary<int64, Queue<Ch<unit> * Alt<unit>>>()
  let s = {unique = 0L; reqCh = Ch ()}
  (Job.server << Job.forever)
   (Ch.take s.reqCh >>= function
     | Acquire (lock, replyCh, abortAlt) ->
       match locks.TryGetValue lock with
        | (true, pending) ->
          pending.Enqueue (replyCh, abortAlt)
          Job.unit ()
        | _ ->
          Alt.choose [Ch.give replyCh () ^-> fun () ->
                        locks.Add (lock, Queue<_>())
                      abortAlt]
     | Release lock ->
       match locks.TryGetValue lock with
        | (true, pending) ->
          let rec assign () =
            if 0 = pending.Count then
              locks.Remove lock |> ignore
              Job.unit ()
            else
              let (replyCh, abortAlt) = pending.Dequeue ()
              Alt.choose [Ch.give replyCh ()
                          abortAlt ^=> assign]
          assign ()
        | _ ->
          // We just ignore the erroneous release request
          Job.unit ()) >>-. s
```

As usual, the above server is implemented as a job that loops indefinitely
taking requests from the server's request channel.  The crucial bits in the
above implementation are the uses of `choose`.  In both cases, the server
selects between giving the lock to the client and aborting the transaction using
the reply channel and the abort alternative, which was implemented by the client
using a negative acknowledgment alternative created by the `withNackJob`
combinator.

You probably noticed the comment in the above server implementation in the case
of an unmatched release operation.  We could also have a combinator that
acquires a lock, executes some job and then releases the lock:

```fsharp
let withLock (s: Server) (l: Lock) (xJ: Job<'x>) : Alt<'x> =
  acquire s l ^=> fun () ->
  Job.tryFinallyJob xJ (release s l)
```

This ensures that an acquire is properly matched by a release.

### On the Semantics of Alternatives

The alternatives of Hopac are heavily inspired by the events of Concurrent ML,
but the two are not precisely the same.  Whether or not you are familiar with
the semantics of CML, it is important to understand how alternatives are
evaluated in Hopac.  If you are not familiar with CML, you can ignore the
comparison made to CML in this section and just concentrate on the description
of how alternatives in Hopac behave.

The semantics of Concurrent ML events and Hopac alternatives are slightly
different.  Concurrent ML emphasizes *fairness* and *non-determinism*, while
Hopac emphasizes *performance* and *co-operation*.  In CML, when two or more
events are immediately available, the choice between them is made in a
*non-deterministic* fashion.  In Hopac, the first alternative that is available
will be chosen *deterministically*.  Consider the following expression:

```fsharp
Alt.choose
 [Alt.always 1
  Alt.always 2]
```

In Hopac, the above alternative will *deterministically* evaluate to 1, because
it is the first available alternative.  In CML, the similar event would
*non-deterministically* choose between the two events.  In this case, we could
get the same behavior in Hopac given a function `shuffle` that would reorder the
elements of a sequence randomly:

```fsharp
Alt.prepareFun <| fun () ->
 Alt.choose
  (shuffle
    [Alt.always 1
     Alt.always 2])
```

The choice of the simpler deterministic semantics in the case of multiple
immediately available alternatives is motivated by performance considerations.
In order to provide the non-determinism, considerably more processing would need
to be performed.  Consider the following example:

```fsharp
Alt.choose
 [Alt.prepareFun <| fun () -> printfn "A" ; Alt.always 1
  Alt.prepareFun <| fun () -> printfn "B" ; Alt.always 2]
```

In Hopac, binding the above alternative prints `A` and nothing else.  In CML,
the similar event would print both `A` and `B`.  In other words, in the initial
phase, Hopac evaluates alternatives *lazily*, while CML evaluates events
*eagerly*.  Hopac can therefore run more efficiently in cases where an
alternative happens to be immediately available.

The above examples are contrived.  In real programs, choices are made over
*communications between separate jobs* that may run in parallel.  This means
that in many cases the initial lazy and deterministic evaluation of alternatives
makes no difference except for performance.  In cases where none of the
alternatives is immediately available, the behavior of Hopac and CML is
essentially the same.  However, it is obviously possible to write programs that
rely on either the Hopac style lazy and deterministic or the CML style eager and
non-deterministic initial choice.

Channels, Mailboxes, IVars, MVars, ...
--------------------------------------

In this document we have mostly used channels in our examples.  The Hopac
library, like CML, also directly provides other communication primitives such as
`Mailbox`[*](http://hopac.github.io/Hopac/Hopac.html#def:type%20Hopac.Mailbox),
`IVar`[*](http://hopac.github.io/Hopac/Hopac.html#def:type%20Hopac.IVar),
`MVar`[*](http://hopac.github.io/Hopac/Hopac.html#def:type%20Hopac.MVar)
and
`Lock`[*](http://hopac.github.io/Hopac/Hopac.html#def:type%20Hopac.Lock).
These other primitives are optimized for the particular communication patterns
they support, but most of them could be implemented using only jobs and channels
as shown in the book
[Concurrent Programming in ML](http://www.cambridge.org/us/academic/subjects/computer-science/distributed-networked-and-mobile-computing/concurrent-programming-ml),
for example.  When programming with Hopac, it, of course, makes sense to use the
optimized primitives where possible.  So, for example, rather than allocating a
channel and starting a job for a one-shot communication, it makes sense to use
an
`IVar`[*](http://hopac.github.io/Hopac/Hopac.html#def:type%20Hopac.IVar),
which implements the desired semantics more efficiently.  On the other hand, it
is reassuring that these optimized primitives, and many others, can be
implemented using only jobs and channels.  This means that there is no need for
the Hopac library to be continuously extended with new communication primitives.

Going Further
-------------

For learning more about Concurrent ML style programming, I highly recommend
[John Reppy](http://people.cs.uchicago.edu/~jhr/)'s book
[Concurrent Programming in ML](http://www.cambridge.org/us/academic/subjects/computer-science/distributed-networked-and-mobile-computing/concurrent-programming-ml).
