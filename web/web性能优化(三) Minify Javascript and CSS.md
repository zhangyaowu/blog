####系列文章包括：
[web性能优化(一) 使用压缩传输](https://github.com/zhangyaowu/blog/blob/master/web/web%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E4%BD%BF%E7%94%A8%E5%8E%8B%E7%BC%A9%E4%BC%A0%E8%BE%93.md)  
[web性能优化(二) 合理利用cache-control](https://github.com/zhangyaowu/blog/blob/master/web/web%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96(%E4%BA%8C)%20%E5%90%88%E7%90%86%E5%88%A9%E7%94%A8%E6%B5%8F%E8%A7%88%E5%99%A8%E7%BC%93%E5%AD%98.md  
"web性能优化(二)合理利用cache-control")  
web性能优化(三) Minify Javascript and CSS  
web性能优化(四) 合并、删除js和样式表&利用chrome developer tools做页面性能分析(待写)   
***
精简(minification)js和css包括，移除不必要的字符（空白、换行、制表符）以减小其大小。更深入和复杂一点的是混淆(obfuscation)，在源码层面上进行的另外一种优化方式，它也会移除多余字符，除此之外还会改写函数和变量名，替换为更短的字符串。混淆的过程很复杂，而且极易引入错误。  

压缩传输和精简都是提升性能的途径，有好事者*(他们才是好人!)系统的测试过gzip传输和精简的性能提升，数据如下：  
* 只精简javascript，大小平均减小21%
* 只进行gzip传输，大小平均减小73%
* 同时进行精简javascript和gzip传输，大小平均减小78%  

所以在项目中，投入收益比权衡中选择什么技术手段非常明显了。


***
*这个好事者是Steve Souders, Yahoo!的前端工程师，数据来自他的书High Performance Web Sites
