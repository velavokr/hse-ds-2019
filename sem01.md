## Орг. часть
* Чат группы https://t.me/joinchat/A08TVFjhH1r602Yi0Nl5FA

## Литература
* [Introduction to Reliable and Secure Distributed Programming 2nd Edition](https://www.amazon.com/Introduction-Reliable-Secure-Distributed-Programming/dp/3642152597)
* [Notes on Theory of Distributed Systems](http://www.cs.yale.edu/homes/aspnes/classes/465/notes.pdf)
* [Designing Data-Intensive Applications](https://dataintensive.net/)
* [Google SRE books](https://landing.google.com/sre/books/)

## Надежная передача по ненадежному каналу, протокол TCP
### Абстракция
* Каждая пара процессов соединена двунаправленным каналом
* У каждого сообщения известен отправитель и получатель
* В случае arbitrary / byzantine fault считаем, что можем положиться на криптографические методы валидации
* Реальная топология сети может быть неоднородной, там важен performance, congestion / flow control
* Если важна связь между сообщениями, используем идентификаторы (уникальные счётчики или uuid)
  * uuid - типично timestamp + много random bits ([A brief history of the UUID](https://segment.com/blog/a-brief-history-of-the-uuid/))
* Получено vs Доставлено
#### FairLossLink
Сообщения могут теряться, но eventually что-то послать получится
* Свойства
  1. **Fair-loss**. Если корректный процесс `p` бесконечно долго перепосылает сообщение процессу `q`, рано или поздно оно будет доставлено.
  1. **Finite duplication**. Если корректный процесс `p` послал сообщение процессу `q` конечное число раз, доставлено оно будет тоже конечное число раз.
  1. **No creation**. Если `q` получил сообщение от `p`, значит ранее `p` посылал его `q`
* Интерфейс
  ```go
  type Link interface {
    SendMessage(dst Addr, msg Message)
  }
  type LinkHandler interface {
    HandleMessage(src Addr, msg Message)
  }
  ```
#### StubbornLink
Абстракция над перепосылками
* Свойства
  1. **Stubborn delivery**. Если корректный процесс `p` послал сообщение процессу `q`, то оно будет доставлено бесконечное число раз.
  1. **No creation**.
* Реализация - появляется таймер  
#### PerfectLink
Абстракция над гарантированной доставкой без дубликатов
* Свойства
  1. **Reliable delivery**. Если корректный процесс `p` отправил сообщение корректному процессу `q`, то рано или поздно оно будет доставлено.
  1. **No dupliction**. Ни одно сообщение не будет доставлено больше одного раза.
  1. **No creation**.
* Важно - порядок сообщений никто не гарантировал!  
* Реализация - StubbornLink + детектор дубликатов на принимающей стороне  
#### PerfectFifoLink
Абстрация над порядком
* Свойства
  1. **Reliable delivery**.
  1. **No dupliction**.
  1. **No creation**.
  1. **FIFO delivery**. Процесс `q` получит сообщения от процесса `p` в том же порядке, в котором `p` их отправил.
#### Failure detectors
TBD
### Реализация
#### UDP
* Состоит из 
  * 16 bit `src port`
  * 16 bit `dst port`
  * 16 bit `length`
  * 16 bit `checksum` - никаких гарантий *no creation*!
    * Особенно часто случается в памяти сетевого оборудования, когда не защищен чексуммой eth (32 bit)
* Почти fifo. [Реальное телекоммуникационное оборудование очень старается](https://linkmeup.ru/blog/312.html).
#### TCP
* Состоит из
  * 16 bit `src port`
  * 16 bit `dst port`
  * 32 bit `seq`
  * 32 bit `ack` - если есть флаг `ACK`
  * 4 bit `data offset`
  * 3 bit `reserved`
  * 9 bit `flags`:
    * `NS` - реально не используется
    * `CWR` - Congestion Window Reduced
    * `ECE` - ECN-Echo
    * `URG` - реально почти никогда не используется
    * `ACK` - см `ack` поле
    * `PSH` - доставить приложению
    * `RST` - ошибка соединения
    * `SYN` - первый пакет
    * `FIN` - последний пакет
  * 16 bit `window size`
  * 16 bit `checksum` - никаких гарантий *no creation*!
  * 16 bit `urgent pointer` - см `URG`
  * 0-320 bit `options`
    * `MSS`
    * `window scaling` (0-14)
    * `selective acknowledgments`
    * `timestamp` + `echo timestamp`
