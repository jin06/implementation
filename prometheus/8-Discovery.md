### Discovery

prometheus自动发现监控节点的方式。 

就是动态的获取需要监控的节点，然后reload，重新配置抓取的scrape的。

例如k8s的注解，可以解析工作负载的注解，如果工作负载配置有相关的注解，那么prometheus就可以reload去抓取这个新的工作负载中的pod。

不重要，快速浏览一下。


