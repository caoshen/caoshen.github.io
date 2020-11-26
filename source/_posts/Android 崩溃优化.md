# Android 崩溃优化

## Android 的两种崩溃

1. Java 崩溃
2. Native 崩溃

Java 崩溃就是在 Java 代码中，出现了未捕获异常，导致程序异常退出。

Native 崩溃一般都是因为在 Native 代码中访问非法地址，也可能是地址对齐出现了问题，或者发生了程序主动 abort，这些都会产生相应的 signal 信号，导致程序异常退出。