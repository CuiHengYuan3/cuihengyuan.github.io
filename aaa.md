意料之中，由于多线程造成的并发，sharedVar++并非原子操作，是先读取值，在加上1成为一个新的值再写入。

如果在一个线程读的时候另一个线程也在读，最后会得到一样的值写入，那么就相当于浪费了一次循环。

![1568881047962](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1568881047962.png)
