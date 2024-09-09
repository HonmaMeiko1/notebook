**这个项目是我自己的github笔记本，以及对github学习使用的开端**

2024-8-28  
“Spark环境配置”  
花费了三天左右时间将本机基于 ubuntu20.04 的 spark-yarn 环境集群环境搭建完成，搭建中遇到的主要问题在于 WEB UI 中查看 historyLog 失败，具体解决办法已经写在了spark环境配置这个md文件中  

2024-9-3  
“Hibench配置”  
又是三天左右时间，在服务器上配置并测试了基于maven3.9.9的Hibench(7.1.1)，现按照该项目中的md进行配置能够解决  
1. 在HDFS中完成指定量数据的生成
2. 使用集群运行application后的结果能分区存储在HDFS中
3. 配置完成了monitor.html，能够拉取到本地，在作业运行完成之后对其运行过程中资源使用的可视化监测

2024-9-9
“集群监测指标”  
通过使用intel RAPL(Running Average Power Limit)实现检测处理器的功率消耗，为后续实验中不同算法的能源消耗监测提供支持  
