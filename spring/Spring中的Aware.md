# Spring中的Aware

Spring提供aware接口，能够让Bean感知Spring容器的存在，即让Bean可以使用Spring容器所提供的资源。

举个简单的小例子，BeanNameAware这个接口，哪个类A实现了它，在通过getBean()方法获取时，BeanNameAware中的setBeanName(String name)方法就会生效。例如：

```java
	public interface BeanNameAware extends Aware {
    	void setBeanName(String name);
	}
```

```java
    public class User implements BeanNameAware {
    	private String id;
        private String name;
        
        //getter、setter
        
        public void setBeanName(String beanName) {
            this.id = beanName;
        }
    }
```

```xml
<bean id="zhangsan"  class="User">
	<property name="name" value="zhangsan"></property>
	<property name="address" value="火星"></property>
</bean>
```

```java
public static void main(String[] args) {
    ApplicationContext context = 
        new ClassPathXmlApplicationContext("classpath:application-beanaware.xml");
    User user=context.getBean(User.class);
    System.out.println(String.format("实现了BeanNameAware接口的信息BeanId=%s,所有信息=%s",
                                         user.getId(),user.toString()));
}
```

最终结果如下：

```java
实现了BeanNameAware接口的信息BeanId=zhangsan,所有信息=User{id='zhangsan', name='zhangsan', address='火星'}
```

可见通过ApplicationContext.getBean()获取到的对象，是调用了aware中的set方法的，也就使得Bean能够感知到BeanFactory的存在，使得Bean能够使用Spring容器所提供的资源!!



几种常用的Aware接口

* ApplicationContextAware
* ApplicationEventPublisherAware，应用事件发布器，可以用来发布事件。
* BeanClassLoaderAware
* BeanFactoryAware
* BeanNameAware
* EnvironmentAware
* MessageSourceAware，能够获取I18N文本信息。
* ResourceLoaderAware
* ServletConfigAware
* ServletContextAware，能够获取到ServletContext。



这样其实有个好处，就不用在这里面注入这些东西了。有一些还不是很容易找。