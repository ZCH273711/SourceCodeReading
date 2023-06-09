# Redis7.0源码阅读笔记——listpack

listpack在redis7.0中也是一种底层的数据结构，quiklist的实现、stream的实现以及redis向外提供的六种数据结构之一的hash也需要依赖listpack的实现。这一文章将对listpack进行分析。



