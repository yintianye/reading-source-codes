# ServletContext和ApplicationContext（IOC容器）

严格意义上说servlet容器包含IOC容器，因此，ApplicationContext最终是ServletContext的一个【attribute】。

* ServletContext的创建和销毁
  * ServletContext的创建时间为web应用（服务器）启动时。
  * ServletContext的销毁时间为web应用（服务器）关闭时。

* ServletContext创建时的操作

  * 读取<context-param></context-param>中的节点内容
  * 触发ServletContextEvent事件，ServletContextListener【接口】去监听这个事件

* Spring中对于ApplicationContext的创建操作

  * 因为ContextLoaderListener(spring-web中的)就是ServletContextListener的实现，因此在这个ContextLoaderListener中，去初始化IOC容器

    ```java
    @Override
    public void contextInitialized(ServletContextEvent event) {
    	initWebApplicationContext(event.getServletContext());
    }
    ```

    

  * 先判断是否已经创建ApplicationContext

  * 然后创建WebApplicationContext

  * 转换成ConfigurableWebApplicationContext

  * 执行configureAndRefreshWebApplicationContext（）方法，封装ApplicationContext的上下文数据，其中会从ServletContext中读取applicationContext.xml中的配置，调用refresh（）方法完成所有bean的解析初始化创建。

  * 最后把创建好的configureAndRefreshWebApplicationContext放到ServletContext中

    ```java
    servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
    ```

    是一种setAttribute的方式。ApplicationContext最终是ServletContext的一个【attribute】。