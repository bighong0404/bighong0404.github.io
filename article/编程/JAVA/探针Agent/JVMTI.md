https://developer.ibm.com/zh/articles/j-lo-jpda2/

https://zhuanlan.zhihu.com/p/135872794

https://www.cnblogs.com/aspirant/p/8796974.html

JDK6 Instrumentation  https://www.ibm.com/developerworks/cn/java/j-lo-jse61/





JVMTI 

JVMTI（JVM Tool Interface）是 Java 虚拟机所提供的 native 编程接口. JVMTI 提供了可用于 debug 和 profiler 的接口；同时，在 Java 5/6 中，虚拟机接口也增加了监听（Monitoring），线程分析（Thread analysis）以及覆盖率分析（Coverage Analysis）等功能。正是由于 JVMTI 的强大功能，它是实现 Java 调试器，以及其它 Java 运行态测试与分析工具的基础。



Java Agent 

Java Agent 所使用的 `Instrumentation `依赖 JVMTI 实现，当然也可以绕过 Instrumentation 直接使用 JVMTI 实现 Agent。因此，JVMTI 与 JDI 组成了 Java 平台调试体系（JPDA）的主要能力。如果想要深入了解 Java Agent,就得需要了解 JVMTI.