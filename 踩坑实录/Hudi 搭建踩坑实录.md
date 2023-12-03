# Hudi 搭建踩坑实录

* 编译Hudi 0.13.0 ，这个版本已经依赖了 spark3，但是还没有依赖 Hadoop3.x 和 hive3.x。
* 参考 [(79条消息) Hudi编译适配hadoop3.2.4_YoungerChina的博客-CSDN博客](https://blog.csdn.net/younger_china/article/details/127484478) 修改 hadoop 的版本号，并且因为 0.13.0 已经有依赖了 spark3 ，所以编译过程意外的顺利。就是不知道以后用的时候会不会发现翻车
