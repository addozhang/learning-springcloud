# Zuul
嵌入式的zuul代理

使用了Netfilx OSS的其他组件:
- Hystrix   熔断
- Ribbon    负责发送外出请求的客户端, 提供软件负载均衡功能
- Trubine   实时地聚合细粒度的metrics数据
- Archaius  动态配置

## 介绍

### 特性
- Authentication 认证
- Insights 洞察
- Stress Testing 压力测试
- Canary Testing 金丝雀测试
- Dynamic Routing 动态路由
- Multi-Region Resiliency 多区域弹性
- Load Shedding 负载脱落
- Security 安全
- Static Response handling 静态响应处理
- Multi-Region Resiliency 主动/主动流量管理

### Zuul核心架构

#### 过滤器加载器
从文件目录定时的监控文件, 编译成Class并加载到过滤器链中.

#### 贯穿整个请求的RequestContext
将Servlet的请求和响应初始化成`RequestContext`, 保存在ThreadLocal中贯穿整个请求.

以及添加Netfix库的指定概念和数据的扩展对象`NFRequestContext`, 如`Eureka`

#### 四种过滤器:

- preRoute
- route
- postRoute
- error

![Zuul Core Architecture](https://cdn-images-1.medium.com/max/1000/1*j9iGkeQ7bPK2nC1a7BgFOw.png)

### Zuul请求生命周期

![Request Lifecycle](https://camo.githubusercontent.com/4eb7754152028cdebd5c09d1c6f5acc7683f0094/687474703a2f2f6e6574666c69782e6769746875622e696f2f7a75756c2f696d616765732f7a75756c2d726571756573742d6c6966656379636c652e706e67)

### Zuul Netflix
使用Netflix的其他组件

![Netflix OSS libraries in Zuul](https://cdn-images-1.medium.com/max/800/1*pz6sv69la9ek6yWNTPqymQ.png)

### zull在Netfilx的应用
#### 精确路由
创建一个过滤器是特定的用户或者设备的请求重定向到独立的API集群达到调试的目的. 

#### 多区域弹
Zuul是我们称为地峡(Isthmus)的多地区ELB弹性项目的核心. 作为Isthmus的一部分, Zuul被用来将请求从西海岸数据中心传送到东海岸, 以帮助我们在我们的关键领域的ELB中实现多区域冗余.

#### 压力测试
在`Zuul`过滤器中使用动态`Archaius`配置逐步提升进入一部分服务器的流量, 自动实现压力测试. 

## 原理

### 如何工作

#### StartServer初始化
实现了ServletContextListener接口, 如果需要与netflix oss其他组件集成(如Eureka, Archaius)实例化的时候启动一个Karyon服务器. 

在ServletContext初始化完成后调用`initGroovyFilterManager`和`initJavaFilters`.

##### initGroovyFilterManager
向过滤器注册表中添加Groovy过滤器.

```java
private void initGroovyFilterManager() {
    //设置GroovyCompiler
    //GroovyCompiler是DynamicCompiler的实现类
    FilterLoader.getInstance().setCompiler(new GroovyCompiler());

    //从配置中是获取过滤器源文件的根目录
    String scriptRoot = System.getProperty("zuul.filter.root", "");
    if (scriptRoot.length() > 0) scriptRoot = scriptRoot + File.separator;
    try {
        //设置文件名过滤器, 这里只过滤`.groovy`类型文件.
        FilterFileManager.setFilenameFilter(new GroovyFileFilter());
        //初始化过滤器文件管理器
        //第一个参数是扫描目录的间隔时间, 单位为秒
        //后面跟要扫描的子目录
        //1. 初始化的时候会扫描各个子目录, 使用文件名过滤器获取到所有的过滤器源文件. 
        //2. 遍历这些文件, 使用`FilterLoader.getInstance().putFilter(file)`, compiler编译之后使用FilterFactory进行实例化, 并添加到过滤器注册表中. 是否实例化的逻辑判断是否在上次修改且文件最后修改时间是否相同. 如果是上次修改之后又有改动, 要重建改类型过滤器的列表. 如果没有修改, 对改文件不做任何处理.
        //3. 启动线程, 每个5秒执行一个1和2的操作.
        FilterFileManager.init(5, scriptRoot + "pre", scriptRoot + "route", scriptRoot + "post");
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```
##### initJavaFilters
向过滤器注册表中添加Java过滤器.

*官方没有提供从java源代码到classs的编译器. 

#### ZuulServlet
核心zuul servlet, 初始化和卸掉zullFilter的运行.
使用ZuulRunner将Servlet的请求和响应初始化成`RequestContext`, 并将`FilterProcessor`的调用包装成`preRoute()`, `route()`, `postRoute()`和`error()`方法. 初始化时可以选择将请求包装成`HttpServletRequestWrapper`并缓冲请求消息体. 

初始化后的`RequestContext`会放在`ThreadLocal`中, 供后续的filter访问.

**Service方法**

```java
public void service(javax.servlet.ServletRequest servletRequest, javax.servlet.ServletResponse servletResponse) throws ServletException, IOException {
    try {
        init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);

        // Marks this request as having passed through the "Zuul engine", as opposed to servlets
        // explicitly bound in web.xml, for which requests will not have the same data attached
        RequestContext context = RequestContext.getCurrentContext();
        context.setZuulEngineRan();

        try {
            preRoute();
        } catch (ZuulException e) {
            error(e);
            postRoute();
            return;
        }
        try {
            route();
        } catch (ZuulException e) {
            error(e);
            postRoute();
            return;
        }
        try {
            postRoute();
        } catch (ZuulException e) {
            error(e);
            return;
        }

    } catch (Throwable e) {
        error(new ZuulException(e, 500, "UNHANDLED_EXCEPTION_" + e.getClass().getName()));
    } finally {
        RequestContext.getCurrentContext().unset();
    }
}
```

通过`FilterProcessor.getInstnace()`调用`FilterProcessor`的`preRoute()`, `route()`, `postRoute()`和`error()`方法.

四个方法都是通过`FilterLoader.getInstance()`获取对应类型的filter列表.

遍历filter列表, 调用filter的`runFilter()`方法.

```java
/**
 * runFilter checks !isFilterDisabled() and shouldFilter(). The run() method is invoked if both are true.
 *
 * @return the return from ZuulFilterResult
 */
public ZuulFilterResult runFilter() {
    ZuulFilterResult zr = new ZuulFilterResult();
    //动态获取`zuul.filerClassName.filterType.disable`的值
    //动态获取使用Archaius的DynamicPropertyFactory获取*, 通过这个可实现动态配置
    if (!isFilterDisabled()) {
        //调用filter类的校验逻辑
        if (shouldFilter()) {
            Tracer t = TracerFactory.instance().startMicroTracer("ZUUL::" + this.getClass().getSimpleName());
            try {
                //执行filter的逻辑处理
                Object res = run();
                //执行成功
                zr = new ZuulFilterResult(res, ExecutionStatus.SUCCESS);
            } catch (Throwable e) {
                t.setName("ZUUL::" + this.getClass().getSimpleName() + " failed");
                //执行失败
                zr = new ZuulFilterResult(ExecutionStatus.FAILED);
                zr.setException(e);
            } finally {
                t.stopAndLog();
            }
        } else {
            //filter不适用, 直接跳过
            zr = new ZuulFilterResult(ExecutionStatus.SKIPPED);
        }
    }
    return zr;
}
```

#### ContextLifecycleFilter
清空`ThreadLocal`中的`RequestContext`

```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
    throws IOException, ServletException {
    try {
        chain.doFilter(req, res);
    } finally {
        RequestContext.getCurrentContext().unset();
    }
}
```

### 调试
调试信息信息中的民资

- ZUUL_DEBUG    输出zuul的诊断信息
- REQUEST_DUBG  输出Http请求的信息. REQUEST -> ZUUL -> ORIGIN_RESPONSE -> OUTBOUND
    - REQUEST   进入zuul的请求
    - ZUUL  zuul转发给原目标的请求
    - ORIGIN_RESPONSE   原目标返回的原始响应
    - OUTBOND   zuul返回给客户端的响应

## 接口和类

### 接口

#### DynamicCodeCompiler
从源代码编译成Classes的接口, 目前只有一个`GroovyCompiler`实现类

#### FilterFactory
生成给定的过滤器类实例的接口, 实现类`DefaultFilterFactory`

#### FilterUsageNotifier
注册过滤器使用时的回调的接口

### 类

#### DefaultFilterFactory
使用反射实现

```java
public ZuulFilter newInstance(Class clazz) throws InstantiationException, IllegalAccessException {
    return (ZuulFilter) clazz.newInstance();
}
```

#### FilterFileManager
从过滤器目录中获取修改和新增的Groovy过滤器文件. 

#### FilterLoader
持有过滤器注册表, 加载过滤器.
