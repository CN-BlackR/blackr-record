# 本笔录采用Docsify + Github Pages + DNS加速构建

> 除了域名，斥巨资，其他的均为白嫖......
> 
> 所以在笔录首页最前面还是给他们冠个名😁😁😁......

# 社交网站
> 本笔记汇集了其他网站的文章（暂时还没有整理完...）
>
> 本笔录[源码](https://github.com/Ch-BlackR/blackr-record)在github上面（码云的要实名认证😑......）
- [简书](https://www.jianshu.com/u/ee3350ee4593)
- [码云](https://gitee.com/BlackT_JAVA)
- [Github](https://github.com/Ch-BlackR)

# 笔录目录

- 设计模式与SpringSecurity的那些事
    - [责任链模式](zh-cn/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F.md)
    - [观察者模式](zh-cn/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F.md)
    - [代理模式](zh-cn/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E4%BB%A3%E7%90%86%E6%A8%A1%E5%BC%8F.md)
- 时光过客
    - Redis&Lua
        - [redis备份还原](zh-cn/Redis%26Lua/redis%E5%A4%87%E4%BB%BD%E8%BF%98%E5%8E%9F.md)
        - [lua脚本](zh-cn/Redis%26Lua/lua%E8%84%9A%E6%9C%AC.md)
    - Nginx
        - [lua脚本](zh-cn/Nginx/lua%E8%84%9A%E6%9C%AC.md)
        - [负载均衡](zh-cn/Nginx/%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1.md)
        - [代理转发](zh-cn/Nginx/%E4%BB%A3%E7%90%86%E8%BD%AC%E5%8F%91.md)
    - MySQL
        - [用户权限](zh-cn/MySQL/%E7%94%A8%E6%88%B7%E6%9D%83%E9%99%90.md)
    - Docker
        - [docker-compose](zh-cn/Docker/docker-compose.md)
        - [Dockerfile](zh-cn/Docker/Dockerfile.md)
    - Linux
        - [常用命令](zh-cn/Linux/%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4.md)
- 从那（log4j）以后....
    - [CVE-2022-22947(spring-cloud-gateway router)](zh-cn/CVE复现/CVE-2022-22947.md)
    - [CVE-2022-22965(spring-framework class)](zh-cn/CVE复现/CVE-2022-22965.md)

# 引用
- [emoji：😁😁😁](https://emojixd.com/)
- [Docsify：一个神奇的文档网站生成器](https://docsify.js.org/#/zh-cn/)
- [github Pages：Websites for you and your projects.](https://pages.github.com/)
- [Cloudflare：DNS加速 ](https://dash.cloudflare.com/)


# Docsify的使用教程
> （主要是我记不住，忽略不计......）
- 安装docsify工具
```shell
npm i docsify-cli -g
```

- 创建新的docsify
```shell
docsify init ./docs
```

- 启动docsify
```shell
docsify serve docs
```
