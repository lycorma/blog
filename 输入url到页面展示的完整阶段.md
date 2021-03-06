### 前置基础

1. 浏览器是多进程的，在浏览器中新开一个tab，至少会同步创建如下四个进程，（可以在浏览器 更多工具-任务管理器查看）：

   - 浏览器进程 - 控制管理其他进程/地址栏等公共部分，文中会称为主进程；
   - GPU进程 - 控制帧(frame)的合成；
   - 网络进程 - 控制网络请求；
   - 渲染进程- 页面渲染进程；

   *note：开第二个tab的时候，前三个进程是共用的，渲染进程大部分时候是独立的，某些时候比如同站点时是共用的。*

2. 进程之间是通过进程间通信 - IPC来交流的；
3. 渲染进程内的资源是从网络中获取的，安全性不可控，所以运行在沙箱里；



### 打开浏览器，在地址栏输入好内容，点击回车

1. 浏览器同步创建4个进程；

2. **主进程**在点回车时分析输入内容；

   - 如果是一个query，就用默认搜索网址的地址拼接上字符串
   - 如果是url，根据默认策略拼接上协议头http:// 或https://

3. 如果是在一个渲染好的页面上输入的url，触发当前页面beforeunload事件；

   - If 如果beforeunload取消了路由跳转，则无事发生，留在原来的页面
   - Else，继续往下走

4. 主进程修改标签上的UI为loading状态；

5. 将拼接好的url发送给**网络进程**，网络进程查找该域名有没有缓存；

   - If 有缓存，并且还处于缓存有效期，就返回缓存内容
   - Else，继续往下走

6. 开始准备IP地址，查找DNS缓存；

   - IP地址：每台连入互联网的计算机地址，IP协议：数据包在互联网上传输必须遵守的协议

   - If DNS有缓存，返回缓存中域名对应的IP地址
   - Else，发起DNS请求，根据域名拿回IP地址

7. 准备端口号；

   -  一般如果没有特别指明，HTTP协议默认端口是80，HTTPS端口是443

   - IP只负责将数据包传给对方电脑，但对方不知道要将数据包给哪个程序，所以需要UDP协议。UDP传输速度快，但不能校验数据包的正确性，也没有重传机制，所以采用TCP协议
   - TCP协议 - 面向连接的、可靠的、基于字节流的传输层通信协议。有重传机制和数据包排序机制。

8. 开始建立TCP连接；

   - Chrome同域名下同时最多只能建立6个TCP连接，如果同时同域多于6个，就会进入TCP队列，等前面的请求完成
   - 三次握手

9. 构建请求行、请求头、请求体信息，并将与该域名相关的 Cookie 等数据附加到请求头；

   ``` 
   GET /index.html HTTP1.1
   ```

10. 发送请求；

    - ？ 先发送一次请求行，如果是post请求，再发送一次请求体

    - 在**应用层**，数据包被根据HTTP协议加上HTTP头
    - 在**传输层**，数据包被加上TCP头，包含目标端口、本机端口和排序序列号
    - IP地址有了以后，在**网络层**，数据包被附加上IP头信息，包括IP版本、源IP地址、目标IP地址、生存时间等

11. 拿到服务器返回的响应行和响应头，并开始解析；

    - 经过两层拆包，拆IP头拆TCP头后拿到的HTTP数据
    - 数据传输完成后经过四次挥手断开TCP连接

    - If 状态码301/302，则根据响应头中的Location字段重定向，一切从头开始
    - If 状态码是304，说明缓存可用，网络进程会把对应的缓存内容返回出去（后面说）
    - If content-type是application/octet-stream这种下载类型，网络进程会把请求丢给下载管理器
    - If content-type是html，继续往下走

12. **网络进程**向浏览器主进程发送通知，**主进程**接到通知后，向**渲染进程**发送一条commitNavigation的通知；

13. **渲染进程**接收到commitNavigation通知后，与**网络进程**建立一个通道，开始接收html等数据，接收完成后，向**主进程**发送‘确认提交’通知；

14. **主进程**收到确认提交后，移除页面上的旧文档（如有），更新浏览器界面状态（如安全提示），更新地址栏前进后退信息；

    - 此时意味着：导航阶段结束，文档加载阶段开始

14. （两个14没错，因为两个进程是并行的）**渲染进程**开始解析html；

    - domLoading事件：浏览器即将开始解析第一批收到的 HTML 文档字节

    - 渲染进程有多个线程，核心任务是将html，css， js转化为用户可交互的网页
      - 主线程
      - 合成线程 compositor
      - 栅格线程 raster

    1. 主线程构建DOM Tree，如果遇到link/img等需下载资源，通过IPC交给网络进程去下载；

       - 读取原始字节，根据文件的指定编码（如UTF-8）将字节转换为字符；
       - 根据HTML 5标准，将字符串转换为token(令牌)，如<html> <body>等，每个令牌都有特殊含义和规则
       - 词法分析：将令牌转换为含属性和规则的对象
       - 构建DOM Tree
       - domInteractive： 浏览器完成所有 HTML 的解析且构建完 DOM 

    2. 构建好DOM Tree 后，将css 转换为统一标准单位的styleSheets；

       - 也会经历构建DOM Tree那一套

       - 将em、rem、px等统一
       - 生成的styleSheets可以通过js调用

    3. 计算每个节点的样式和几何位置，生成layout Tree；

       - domContentLoaded： DOM 准备就绪且没有样式表阻止 JavaScript 执行，这意味着现在可以构建渲染树了

    4. 根据layout Tree生成paint records，决定绘制顺序；

    5. 根据layout Tree和paint records，找出每个元素应该在哪一层，分割layer，生成layer Tree；

    6. 当layer tree 和绘制记录已生成，**主线程**会向**合成线程**提交信息；

    7. **合成线程**将每个layer分割为图块(tiles)后（因为整个结构太多太大了），发送到**栅格线程**进行栅格化；

       - 合成线程可以控制栅格线程的优先级，在视口或附近的图块先栅格化

    8. **栅格线程**维护了一个栅格化线程池，所有的图块会依次被栅格化，这个过程会使用**GPU进程**来加速生成；

       - 栅格化就是将图块转换为位图
       - 使用GPU生成的过程叫快速栅格化，生成的位图会直接被保存在GPU内存中

    9. 所有图块被栅格化后，**合成线程**会收集名为‘draw quads’的图块信息创建合成帧，并通过IPC将该帧发送到**浏览器进程**；

       - draw quads: 包含图块在内存中的信息，及考虑到页面合成，在页面哪个位置绘制图块

    10. **浏览器进程**中的viz组件接收到帧信息后，将内容绘制到内存中，最后再将内存显示到屏幕上。

        - domComplete：所有处理完成，网页上的所有资源（图像等）都已下载完毕，tab loading已停止



至此，在浏览器地址栏输入好内容，点击回车到页面展示的所有流程就结束了。



### 问题与答案

1. js和css会不会阻塞DOM解析？
   - 在<body>前的同步js一定会，async和defer的<script>不会；因为js可能会修改DOM节点，为了不浪费构建事件，会等同步js执行完。
   - 一般css不会，但如果同步js在执行时，访问了某元素的css样式，就会去下载css资源，这时会阻塞。
2. css会不会阻塞页面渲染
   - 会。构建好DOM Tree后构建CSSOM，二者合到一起形成布局树后才会生成绘制记录，继而渲染页面。
3. DOM树和渲染树节点一一对应吗？
   - 不是。被css display none的节点，只存在DOM树中，不存在渲染树中；有content的伪元素，只存在渲染树中，不存在DOM树里。
4. 真正的白屏时间，是从什么时间点开始到什么时间点结束？
   - 从渲染进程接收完html发送‘确认提交’给主进程开始，到合成线程将图块信息合成帧并发送到浏览器进程，浏览器进程绘制到内存，在显示到屏幕上结束。
5. 关键渲染路径（CRP）是什么，如何优化？
   - 从收到 HTML、CSS 和 JavaScript 字节到构建渲染树的中间步骤
   - 怎么优化就三个思路
     - 减少请求的资源数量（async，defer，代码分割）
     - 减小请求资源的大小（压缩，代码分割）
     - 缩短路径长度（减少重排重绘）
6. passive：true为什么会让浏览器滚动更流畅？
   - 太长了，单独写吧





### 参考资料

https://developers.google.com/web/updates/2018/09/inside-browser-part1

https://developers.google.com/web/fundamentals/performance/critical-rendering-path/constructing-the-object-model

https://time.geekbang.org/column/intro/216
