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
  * 32 bit `seq` - номер последовательности: все байты данных + 1 для SYN и 1 для FIN.
  * 32 bit `ack` - если есть флаг `ACK`. 1 + последний полученный `seq`.
  * 4 bit `data offset` - размер в `uint32`
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
    * `selective acknowledgments` - 1-4 интервала.
    * `timestamp` + `echo timestamp` - рандомное начальное значение, инкремент примерно 1ms.
* Стейт-машина
  ![TCP стейт-машина](https://upload.wikimedia.org/wikipedia/en/5/57/Tcp_state_diagram.png)
* Пример
```bash
[console 1]$ tcpdump -n -i lo ip6 and host ::1 and tcp and port 8080
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lo, link-type EN10MB (Ethernet), capture size 262144 bytes
[console 2]$ nc -l ::1 8080
[console 3]$ nc ::1 8080
```
```
12:29:30.839127 IP6 ::1.41120 > ::1.8080: Flags [S], seq 1966290566, win 43690, options [mss 65476,sackOK,TS val 7157002 ecr 0,nop,wscale 7], length 0
12:29:30.839175 IP6 ::1.8080 > ::1.41120: Flags [S.], seq 2764958068, ack 1966290567, win 43690, options [mss 65476,sackOK,TS val 7157002 ecr 7157002,nop,wscale 7], length 0
12:29:30.839210 IP6 ::1.41120 > ::1.8080: Flags [.], ack 1, win 342, options [nop,nop,TS val 7157003 ecr 7157002], length 0
12:29:33.025852 IP6 ::1.41120 > ::1.8080: Flags [P.], seq 1:7, ack 1, win 342, options [nop,nop,TS val 7159189 ecr 7157002], length 6: HTTP
12:29:33.025892 IP6 ::1.8080 > ::1.41120: Flags [.], ack 7, win 342, options [nop,nop,TS val 7159189 ecr 7159189], length 0
12:29:37.385552 IP6 ::1.8080 > ::1.41120: Flags [P.], seq 1:7, ack 7, win 342, options [nop,nop,TS val 7163556 ecr 7159189], length 6: HTTP
12:29:37.385591 IP6 ::1.41120 > ::1.8080: Flags [.], ack 7, win 342, options [nop,nop,TS val 7163556 ecr 7163556], length 0
12:29:41.478883 IP6 ::1.41120 > ::1.8080: Flags [F.], seq 7, ack 7, win 342, options [nop,nop,TS val 7167657 ecr 7163556], length 0
12:29:41.479034 IP6 ::1.8080 > ::1.41120: Flags [F.], seq 7, ack 8, win 342, options [nop,nop,TS val 7167657 ecr 7167657], length 0
12:29:41.479081 IP6 ::1.41120 > ::1.8080: Flags [.], ack 8, win 342, options [nop,nop,TS val 7167657 ecr 7167657], length 0
```
* Почему обман:  
  * `NAT`
  * `keepalive`
  * таймауты
  * буферы на отправку и приём
  
#### Полезные ссылки
* [tcpdump](https://danielmiessler.com/study/tcpdump/)
* [tshark](https://hackertarget.com/tshark-tutorial-and-filter-examples/)
* [socat](https://medium.com/@copyconstruct/socat-29453e9fc8a6)
* [netcat](https://kapeli.com/cheat_sheets/Netcat.docset/Contents/Resources/Documents/index)
* [curl](https://devhints.io/curl)
