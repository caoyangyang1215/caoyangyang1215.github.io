---
title: 用 java 实现 spring 的 IOC 容器
date: 2022-04-19 15:53:45
tags: 
- java
- ioc
---



> 用 java 实现 spring 的 IOC 容器，阅读 [tiny-spring](https://github.com/code4craft/tiny-spring) 之后整理，从 step-1 到 step-6 由简入繁

<!-- more -->

## step1

IOC 最基本的两个角色，容器 和 bean 本身，bean 被存放到容器中，需要使用的时候容器将 bean 提供给我们
那么我们需要一个类，作为 容器，存放 bean
最简单的做法是，定义一个 `BeanFactory` 类作为容器，一个 map 属性以键值对的形式存放 bean name 和 bean 对象

```java
public BeanFactory {
	private Map<String,Object> beanMap = new HashMap<String,Object>();
	
	public Object getBean(String name) {
		return beanMap.get(name);
	}

	public void registerBean(String name, Object bean) {
		beanMap.put(name,bean);
	}
}
```

现在就可以调用 registerBean 方法把 bean 放入到容器中，getBean 获取 bean了



将上面的方法优化一下

定义一个 `BeanDefinition` 用来封装 bean，使用对象去封装 bean 的好处是，除了存放 bean 本身，还可以定义其他的属性去存放一些 bean 的其他信息

```java
public class BeanDefinition {
	private Object bean;

    public BeanDefinition(Object bean) {
        this.bean = bean;
    }

    public Object getBean() {
        return bean;
    }
}
```



再将前面的容器对应修改，逻辑还是一样的

```java
public class BeanFactory {

	private Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>();

	public Object getBean(String name) {
		return beanDefinitionMap.get(name).getBean();
	}

	public void registerBeanDefinition(String name, BeanDefinition beanDefinition) {
		beanDefinitionMap.put(name, beanDefinition);
	}

}
```



容器完成了，定义一个类作为 bean，用作测试

```java
public class HelloWorldServiceImpl implements HelloWorldService {
    @Override
    public void helloWorld() {
        System.out.println("hello world");
    }
}
```



example main 示例
初始化 BeanFactory 容器
初始化一个 `HelloServiceImpl` 对象，存放进 BeanDefinition 的 bean 属性中
以 helloServiceImpl 作为 键(bean name)，beanDefinition 作为值存放进 BeanFactory 的 map 中
然后就可以调用 getBean 方法，通过 bean name 得到对应的 bean 对象

```java
public class ExampleMain {
    public static void main(String[] args) {
       
        BeanFactory beanFactory = new BeanFactory();
       
        BeanDefinition beanDefinition = new BeanDefinition(new HelloWorldServiceImpl());
        beanFactory.registerBeanDefinition("helloWorldServiceImpl",beanDefinition);
   
        HelloWorldService helloWorldService = (HelloWorldService)beanFactory.getBean("helloWorldServiceImpl");
        helloWorldService.helloWorld();

    }
}
```



## step2

在第一步中，做到了 定义 `BeanFactory` 容器，定义 `BeanDefinition` 用来封装 bean，然后初始化 bean 放入容器，最后可以通过 bean name 去获取 bean
我们在初始化 bean 的时候是这么做的：把 HelloWorldServiceImpl 初始化好之后再 set 进属性

```java
BeanDefinition beanDefinition = new BeanDefinition(new HelloWorldServiceImpl());
```



IOC 的原则是 bean 的初始化应该交给容器去做，这显然不是我们想要的，
如何让容器去初始化 bean 呢？
首先，容器肯定需要知道要初始化哪一个类
step-1 中，在初始化 BeanDefinition 的时候就已经把 bean 实例存放在 BeanDefinition 中了

```java
BeanDefinition beanDefinition = new BeanDefinition(new HelloWorldServiceImpl());
```



而下面的一步，beanFactory 登记 bean，除了存放进 map，就没有做任何事情

```java
beanFactory.registerBeanDefinition("helloWorldServiceImpl",beanDefinition);
```



要让容器去初始化 bean，那么在 new BeanDefinition 的时候就不初始化 bean
但是要存放一些信息，后面 BeanFactory 根据这些信息去初始化 bean

BeanDefinition 新增属性
beanClassName 存放 bean 的全路径名
beanClass 存放 bean 的 calss 对象
setBeanClassName 的时候，根据全路径，使用 java 反射得到类对象存放到 beanClass 属性中

```java
public class BeanDefinition {

	private Object bean;

	private Class beanClass;

	private String beanClassName;

	public BeanDefinition() {
	}

	public void setBean(Object bean) {
		this.bean = bean;
	}

	public Class getBeanClass() {
		return beanClass;
	}

	public void setBeanClass(Class beanClass) {
		this.beanClass = beanClass;
	}

	public String getBeanClassName() {
		return beanClassName;
	}

	public void setBeanClassName(String beanClassName) {
		this.beanClassName = beanClassName;
		try {
			this.beanClass = Class.forName(beanClassName);
		} catch (ClassNotFoundException e) {
			e.printStackTrace();
		}
	}

	public Object getBean() {
		return bean;
	}

}
```



step-1 的 BeanFactory 是一个 class 对象，为了保证扩展性
把容器特性抽象出来成为一个接口
定义两个必要方法 获取 bean 和 注册 bean

```java
public interface BeanFactory {

    Object getBean(String name);

    void registerBeanDefinition(String name, BeanDefinition beanDefinition);
}
```



抽象类 `AbstractBeanFactory` 实现接口，存放 bean 的 map 在此处定义
这里面定义了一个抽象方法 doCreateBean
这个方法就是我们用来初始化 bean 的
参数 BeanDefinition 封装了类的信息，容器得到类的信息去初始化 bean

```java
public abstract class AbstractBeanFactory implements BeanFactory {
    private Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>();

    @Override
    public Object getBean(String name) {
        return beanDefinitionMap.get(name).getBean();
    }

    @Override
    public void registerBeanDefinition(String name, BeanDefinition beanDefinition) {
       
        Object bean = doCreateBean(beanDefinition);
        
        beanDefinition.setBean(bean);
       
        beanDefinitionMap.put(name, beanDefinition);
    }

    protected abstract Object doCreateBean(BeanDefinition beanDefinition);

}
```



定义 `AutowireCapableBeanFactory` 做为一个实现

```java
public class AutowireCapableBeanFactory extends AbstractBeanFactory {

    @Override
    protected Object doCreateBean(BeanDefinition beanDefinition) {
        try {
            Object bean = beanDefinition.getBeanClass().newInstance();
            return bean;
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```



example main

```java
public class ExampleMain {
    public static void main(String[] args) {
        // 初始化 BeanFactory
        BeanFactory beanFactory = new AutowireCapableBeanFactory();
        // 初始化 BeanDefinition
        BeanDefinition beanDefinition = new BeanDefinition();

        /**
         *  step1 是直接实例化出来一个 bean，这里把这一步交给了容器，也就是让 BeanFactory 去初始化
         *  给 BeanDefinition 中的 类名赋值，全路径名
         *  BeanDefinition 根据类全路径名称，反射出来一个 Class 对象
         */
        beanDefinition.setBeanClassName("common_step2.HelloWorldService");

       /**
        * 这一步会从 BeanDefinition 中拿到之前存放的 Class 对象，通过 newInstance() 方法实例化出来一个 Bean
        * 存放进 BeanDefinition 的 属性中
        * 然后把 BeanName 和 BeanDefinition 通过键值对的方式存放进 BeanFactory 的 ConcurrentHashMap 中去
        * **/
        beanFactory.registerBeanDefinition("helloWorldService",beanDefinition);

        /**
         * 获取 bean
         */
        HelloWorldService helloWorldService = (HelloWorldService)beanFactory.getBean("helloWorldService");
        helloWorldService.helloWorld();
    }
}
```



## step3

将容器逐步完善，第二步中把实例化 bean 的操作交给了 BeanFactory 容器
实现了 spring 的控制反转，接下来考虑一些另一个特性：依赖注入

一般情况 bean 都是会带有自己的属性的，如何去注入属性呢

在前几步的基础上，新建 `PropertyValue` 对象和 `PropertyValues` 对象

PropertyValue
name属性对应 bean 的属性名称
value 属性对应 bean 的属性值
一个 PropertyValue 对象即对应一个 bean 的属性

```java
public class PropertyValue {

    private final String name;

    private final Object value;

    public PropertyValue(String name, Object value) {
        this.name = name;
        this.value = value;
    }

    public String getName() {
        return name;
    }

    public Object getValue() {
        return value;
    }
}
```



PropertyValues 存放多个 PropertyValue
此对象会被放进 BeanDefinition

```java
public class PropertyValues {

	private final List<PropertyValue> propertyValueList = new ArrayList<PropertyValue>();

	public PropertyValues() {
	}

	public void addPropertyValue(PropertyValue pv) {
        //TODO:这里可以对于重复propertyName进行判断，直接用list没法做到
		this.propertyValueList.add(pv);
	}

	public List<PropertyValue> getPropertyValueLists() {
		return this.propertyValueList;
	}

}
```



BeanDefinition 中新增一个 PropertyValues 属性

```java
public class BeanDefinition {

	private Object bean;

	private Class beanClass;

	private String beanClassName;
	// 新增
        private PropertyValues propertyValues;

	public BeanDefinition() {
	}

	public void setBean(Object bean) {
		this.bean = bean;
	}

	public Class getBeanClass() {
		return beanClass;
	}

	public void setBeanClass(Class beanClass) {
		this.beanClass = beanClass;
	}

	public String getBeanClassName() {
		return beanClassName;
	}

	public void setBeanClassName(String beanClassName) {
		this.beanClassName = beanClassName;
		try {
			this.beanClass = Class.forName(beanClassName);
		} catch (ClassNotFoundException e) {
			e.printStackTrace();
		}
	}

	public Object getBean() {
		return bean;
	}

    public PropertyValues getPropertyValues() {
        return propertyValues;
    }

    public void setPropertyValues(PropertyValues propertyValues) {
        this.propertyValues = propertyValues;
    }
}
```



## step4

经过前三步的逐步优化，已经实现容器实例化 bean 并注入属性值

![image.png](image-67276738a6c34b8eb421c6a9bb0dbd8a.png)



如果一个 bean 有很多属性值的话，代码会写的非常长
所以这一步实现类似 sping 的 xml 配置方式
在 xml 里面配置 bean 名称，bean 路径，bean 属性名，bean 属性值
按照 spring 的方法，新建一个 xml 文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
	http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
	http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.5.xsd
	http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-2.5.xsd
	http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-2.5.xsd">

    <bean name="helloWorldService" class="us.codecraft.tinyioc.HelloWorldService">
        <property name="text" value="Hello World!"></property>
    </bean>

</beans>
```



既然有了 xml 文件，那肯定需要去解析 xml 文件，获得各个节点的内容
在前面的基础上，定义 `Resource` 接口，`UrlResource`实现接口
此对象的作用是根据 xml 文件的 URL 对象得到 xml 文件的 输入流

```java
/**
 * Resource是spring内部定位资源的接口。
 * @author yihua.huang@dianping.com
 */
public interface Resource {

    InputStream getInputStream() throws IOException;
}
public class UrlResource implements Resource {

    private final URL url;

    public UrlResource(URL url) {
        this.url = url;
    }

    @Override
    public InputStream getInputStream() throws IOException{
        URLConnection urlConnection = url.openConnection();
        urlConnection.connect();
        return urlConnection.getInputStream();
    }
}
```



定义一个 `ResourceLoader` 类，一个 getResource() 方法
在根路径下根据文件名找到对应的 xml 文件
返回一个 UrlResource 对象

```java
public class ResourceLoader {

    public Resource getResource(String location){
        /**
         * location 为 xml 文件的 路径+名称
         * this.getClass().getClassLoader() 表示 classpath 根目录
         * 返回一个 URL 对象
         */
        URL resource = this.getClass().getClassLoader().getResource(location);
        return new UrlResource(resource);
    }
}
```



定义解析 xml 文件的接口`BeanDefinitionReader`
`AbstractBeanDefinitionReader`实现接口
一个 map 属性 registry
从 xml 中解析出来，封装好的 BeanDefinition 对象
通过键值对的方式存放在 map 中
一个 ResourceLoader 属性

```java
public interface BeanDefinitionReader {

    void loadBeanDefinitions(String location) throws Exception;
}
public abstract class AbstractBeanDefinitionReader implements BeanDefinitionReader {
    /**
     * 从 xml 中读取到的文件封装到 BeanDefinition 对象之后
     * 以 bean name 为键，BeanDefinition 为值得方式放入 此 map 中
     */
    private Map<String,BeanDefinition> registry;
    /**
     * ResourceLoader 对象
     * 主要作用为读取 xml 文件，得到 xml 文件的 URl 对象
     */
    private ResourceLoader resourceLoader;

    protected AbstractBeanDefinitionReader(ResourceLoader resourceLoader) {
        this.registry = new HashMap<String, BeanDefinition>();
        this.resourceLoader = resourceLoader;
    }

    public Map<String, BeanDefinition> getRegistry() {
        return registry;
    }

    public ResourceLoader getResourceLoader() {
        return resourceLoader;
    }
}
```



`XmlBeanDefinitionReader` 继承 AbstractBeanDefinitionReader 实现方法
loadBeanDefinitions 方法中，得到 xml 文件的 输入流，然后去解析
doLoadBeanDefinitions 方法，通过输入流得到 xml 的 Document 对象
registerBeanDefinitions 方法，通过 Document 对象得到 xml 根节点
parseBeanDefinitions 方法，通过根节点得到所有的子节点
在循环中，解析每一个子节点
processBeanDefinition 方法，得到 bean 标签的 name 和 class 的值
processProperty 方法，再去解析 bean 标签下面的 property 标签的内容
将得到的 nama 和 value 的值，封装到 BeanDefinition 中
再把 BeanDefinition 和 bean name 以键值对的形式存放到 map 属性中

```java
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {

	public XmlBeanDefinitionReader(ResourceLoader resourceLoader) {
		super(resourceLoader);
	}

	@Override
	public void loadBeanDefinitions(String location) throws Exception {
		/**
		 * 调用继承的 ResourceLoader 的 getResource 方法
		 * 该方法返回一个 Resource 类型的对象 UrlResource
		 * 得到 根目录下面的 名为 location 文件的 URL 对象
		 * 将 URL 对象存放进 UrlResource 中返回
		 * 调用 UrlResource 对象的 getInputStream 方法得到一个 inputStream 对象
		 */
		InputStream inputStream = getResourceLoader().getResource(location).getInputStream();
		/**
		 * 解析 xml
		 */
		doLoadBeanDefinitions(inputStream);
	}

	/**
	 * 通过 xml 文件输入流
	 * 得到 xml 文件的 Document 对象
	 * @param inputStream
	 * @throws Exception
	 */
	protected void doLoadBeanDefinitions(InputStream inputStream) throws Exception {
		DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
		DocumentBuilder docBuilder = factory.newDocumentBuilder();
		Document doc = docBuilder.parse(inputStream);
		/**
		 * 解析 bean
		 */
		registerBeanDefinitions(doc);
		inputStream.close();
	}

	/**
	 * 通过 Document 对象解析 xml
	 * @param doc
	 */
	public void registerBeanDefinitions(Document doc) {
		/**
		 * 得到根节点
		 */
		Element root = doc.getDocumentElement();

		parseBeanDefinitions(root);
	}

	/**
	 * 遍历根节点 <beans></beans> 的子节点 NodeList
	 * @param root
	 */
	protected void parseBeanDefinitions(Element root) {
		NodeList nl = root.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (node instanceof Element) {
				Element ele = (Element) node;
				processBeanDefinition(ele);
			}
		}
	}

	/**
	 * 此处的 Element 表示 <bean></bean>
	 * 获取 bean 标签里的 name 和 class 属性值
	 * 初始化 BeanDefinition
	 * @param ele
	 */
	protected void processBeanDefinition(Element ele) {
		String name = ele.getAttribute("name");
		String className = ele.getAttribute("class");
        BeanDefinition beanDefinition = new BeanDefinition();
		/**
		 * 解析 bean 标签下面的 property 标签，遍历 name value
		 * 将解析完成的数据封装成 PropertyValue 放进 BeanDefinition 的 PropertyValues 属性中
		 */
		processProperty(ele,beanDefinition);
		/**
		 * BeanDefinition className 属性赋值，同时反射出 class 对象给 class 属性赋值
		 */
        beanDefinition.setBeanClassName(className);
		/**
		 * 将 bean name 和对应的 BeanDefinition 对象以键值对的形式存放进本类中的 HashMap 中(继承自抽象类)
		 * 后面初始化 BeanFactory 并 注册 bean 的时候从这里取值
		 */
		getRegistry().put(name, beanDefinition);
	}

	/**
	 * 解析 <bean></bean> 下面的 <property></property> 中的 name 和 value
	 * 放进 PropertyValue 中
	 * 给 BeanDefinition 中的 list 赋值
	 * @param ele
	 * @param beanDefinition
	 */
    private void processProperty(Element ele,BeanDefinition beanDefinition) {
        NodeList propertyNode = ele.getElementsByTagName("property");
        for (int i = 0; i < propertyNode.getLength(); i++) {
            Node node = propertyNode.item(i);
            if (node instanceof Element) {
                Element propertyEle = (Element) node;
                String name = propertyEle.getAttribute("name");
                String value = propertyEle.getAttribute("value");
                beanDefinition.getPropertyValues().addPropertyValue(new PropertyValue(name,value));
            }
        }
    }
}
```



上面方法已经将 xml 文件解析完成，BeanDefinition 已经存放进了 map 中
同样的，此时的 BeanDefinition 中的 bean 属性还是空的，bean 未被实例化
接下来只需要从 map 取出数据，再使用 BeanFactory 容器把 bean 放如工厂就行了
AutowireCapableBeanFactory 与 step-3 相同，没有做改动



example main

```java
public class ExampleMain {
    public static void main(String[] args) {
        // 初始化 BeanFactory
        BeanFactory beanFactory = new AutowireCapableBeanFactory();
        // 初始化 BeanDefinition
        BeanDefinition beanDefinition = new BeanDefinition();
        beanDefinition.setBeanClassName("common_step3.HelloWorldService");
        // 初始化 PropertyValues 对象
        PropertyValues propertyValues = new PropertyValues();
        /**
         * 实例化一个 PropertyValue 对象，赋值 name = ”text“ value = ”Hello World“
         * 对应的 Bean text 属性，值 “Hello World”
         * 把 PropertyValue 放进 PropertyValues 的 List 中
         */
        propertyValues.addPropertyValue(new PropertyValue("text","Hello World"));
        /**
         * 把 PropertyValues 放进 BeanDefinition 的 属性中
         */
        beanDefinition.setPropertyValues(propertyValues);
        /**
         * 注入 bean 到容器中，与上一步相同，多了一步给 bean 注入属性值的方法
         */
        try {
            beanFactory.registerBeanDefinition("helloWorldService",beanDefinition);
        } catch (Exception e) {
            e.printStackTrace();
        }
        // 获取 Bean
        HelloWorldService helloWorldService = (HelloWorldService)beanFactory.getBean("helloWorldService");
        helloWorldService.helloWorld();



    }
}
```



## step5

上一步实现了 xml 配置的形式，去解析配置文件，实现 bean 的自动装配
但是有一个问题，无法处理 bean 和 bean 之间的依赖
例如
HelloWorldService 中有一个 OutputService 对象

```java
public class HelloWorldService {

        private String text;

        private OutputService outputService;

        public void helloWorld(){
        outputService.output(text);
    }

        public void setText(String text) {
        this.text = text;
    }

        public void setOutputService(OutputService outputService) {
        this.outputService = outputService;
    }

}
```

OutputService 中有一个 HelloWorldService 对象

```java
public class OutputService {
    private HelloWorldService helloWorldService;

    public void output(String text){

        System.out.println(text);
    }

    public void setHelloWorldService(HelloWorldService helloWorldService) {
        this.helloWorldService = helloWorldService;
    }
```

这种形式的话，使用上一步的方式就无法实现 bean 中注入 另一个 bean
为了处理 baen 和 bean 之间的依赖关系
定义一个 `BeanReference` 类，当一个类中出现对另一个类的引用时候使用它

```java
public class BeanReference {

    private String name;

    private Object bean;

    public BeanReference(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Object getBean() {
        return bean;
    }

    public void setBean(Object bean) {
        this.bean = bean;
    }
}
```

xml 配置文件按照 sprin 的写法修改
ref 表示对其他 bean 的引用

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:tx="http://www.springframework.org/schema/tx" xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
	http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
	http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.5.xsd
	http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-2.5.xsd
	http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-2.5.xsd">

    <bean name="outputService" class="common_step5.OutputService">
        <property name="helloWorldService" ref="helloWorldService"></property>
    </bean>

    <bean name="helloWorldService" class="common_step5.HelloWorldService">
        <property name="text" value="Hello World!"></property>
        <property name="outputService" ref="outputService"></property>
    </bean>

</beans>
```

相应的解析 xml 文件的类 XmlBeanDefinitionReader 需要调整
类中解析 property 标签的方法 processProperty 需要对 有 ref 这个属性的标签做相应的操作

之前的解析方式是解析 property 标签的 name 和 value 属性的值对应的存放到 BeanDefinition propertyValues 属性中
这里添加了 如果 property 标签中有 ref 这个属性在
就表示，是对另一个 bean 的引用，ref 的值 就为引用的 bean 的 name
创建一个 BeanReference 对象，把 bean name 存放进去
然后把 BeanReference 对象存放到 PropertyValue 对象的 value 属性中
后面就一样的把 PropertyValue 存放到 BeanDefinition 中去

```java
private void processProperty(Element ele, BeanDefinition beanDefinition) {
		NodeList propertyNode = ele.getElementsByTagName("property");
		for (int i = 0; i < propertyNode.getLength(); i++) {
			Node node = propertyNode.item(i);
			if (node instanceof Element) {
				Element propertyEle = (Element) node;
				String name = propertyEle.getAttribute("name");
				String value = propertyEle.getAttribute("value");
				if (value != null && value.length(P) > 0) {
					beanDefinition.getPropertyValues().addPropertyValue(new PropertyValue(name, value));
				} else {
					String ref = propertyEle.getAttribute("ref");
					if (ref == null || ref.length() == 0) {
						throw new IllegalArgumentException("Configuration problem: <property> element for property '"
								+ name + "' must specify a ref or value");
					}
					BeanReference beanReference = new BeanReference(ref);
					beanDefinition.getPropertyValues().addPropertyValue(new PropertyValue(name, beanReference));
				}
			}
		}
	}
```

完成之后，XmlBeanDefinitionReader 中的 registry map 中就存放了所有的 需要放进 容器的 bean name 和对应的 BeanDefinition 对象
BeanDefinition 对象中的 PropertyValues 属性里的 PropertyValue 对象
value 属性有两种类型：String 或者 BeanReference 对象
下面解析的时候对这两种类型做判断，分别做不同的操作

对 AbstractBeanFactory 做一些调整
getBean 方法从 map 中拿到 bean，如果 bean 不存在，调用 doCreateBean 方法去初始化 bean
`同时为了解决循环依赖的问题，我们使用lazy-init的方式，将createBean的事情放到`getBean`的时候才执行，是不是一下子方便很多？这样在注入bean的时候，如果该属性对应的bean找不到，那么就先创建！因为总是先创建后注入，所以不会存在两个循环依赖的bean创建死锁的问题。`
新增一个 list 集合 beanDefinitionNames 存放 bean 的 name
在这里 registerBeanDefinition 所做的只是将
bean name 和对应的 BeanDefinition 存放进 map 里面
bean name 存放进 list 里面

preInstantiateSingletons 方法用来初始化所有的 bean 到容器中

```java
public abstract class AbstractBeanFactory implements BeanFactory {

	private Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>();

	private final List<String> beanDefinitionNames = new ArrayList<String>();

	@Override
	public Object getBean(String name) throws Exception {
		BeanDefinition beanDefinition = beanDefinitionMap.get(name);
		if (beanDefinition == null) {
			throw new IllegalArgumentException("No bean named " + name + " is defined");
		}
		Object bean = beanDefinition.getBean();
		if (bean == null) {
			bean = doCreateBean(beanDefinition);
		}
		return bean;
	}

	@Override
	public void registerBeanDefinition(String name, BeanDefinition beanDefinition) throws Exception {
		beanDefinitionMap.put(name, beanDefinition);
		beanDefinitionNames.add(name);
	}

	public void preInstantiateSingletons() throws Exception {
		for (Iterator it = this.beanDefinitionNames.iterator(); it.hasNext();) {
			String beanName = (String) it.next();
			getBean(beanName);
		}
	}

	/**
	 * 初始化bean
	 *
	 * @param beanDefinition
	 * @return
	 */
	protected abstract Object doCreateBean(BeanDefinition beanDefinition) throws Exception;
```

AutowireCapableBeanFactory 中 遍历 BeanDefinition 中 PropertyValue 并将值赋值给 bean 中对应的属性方法
applyPropertyValues 也要做调整
PropertyValue 的 value 可能是一个字符串，也可能是一个 BeanReference 对象
如果对应的是一个 BeanReference 对象，表示 bean 中的该属性是一个其他 bean 的对象
那么就从 容器 中，根据 bean name 名称去得到该对象的实例赋值给 bean
实例还不存在？调用 doCreateBean 方法去创建

```java
protected void applyPropertyValues(Object bean, BeanDefinition mbd) throws Exception {

		/*n++;
		System.out.println("NO."+n+" ==> applyPropertyValues()");*/

		for (PropertyValue propertyValue : mbd.getPropertyValues().getPropertyValues()) {
			Field declaredField = bean.getClass().getDeclaredField(propertyValue.getName());
			declaredField.setAccessible(true);
			Object value = propertyValue.getValue();
			if (value instanceof BeanReference) {
				BeanReference beanReference = (BeanReference) value;
				/**
				 * 为 bean 注入 bean 的操作，在这一步
				 * 如果 为 bean1 注入 bean2
				 * 两个 bean 都需要在放入到容器中
				 * 等查找到 某一个 bean 中依赖了另一个 bean
				 * 只需要在容器中找到被依赖的 bean 放入
				 */
				value = getBean(beanReference.getName());
			}
			declaredField.set(bean, value);
		}
	}
```

example main

```java
public class ExampleMain {

    public static void main(String[] args) throws Exception {

        /**
         * step-5 处理了 bean 和 bean 之间的注入
         * 为 bena 注入 bean
         */

        // 1.读取配置
        XmlBeanDefinitionReader xmlBeanDefinitionReader = new XmlBeanDefinitionReader(new ResourceLoader());
        xmlBeanDefinitionReader.loadBeanDefinitions("tinyioc.xml");

        // 2.初始化BeanFactory并注册bean
        AbstractBeanFactory beanFactory = new AutowireCapableBeanFactory();
        for (Map.Entry<String, BeanDefinition> beanDefinitionEntry : xmlBeanDefinitionReader.getRegistry().entrySet()) {
            beanFactory.registerBeanDefinition(beanDefinitionEntry.getKey(), beanDefinitionEntry.getValue());
        }

        // 3.初始化bean
        beanFactory.preInstantiateSingletons();

        // 4.获取bean
        HelloWorldService helloWorldService = (HelloWorldService) beanFactory.getBean("helloWorldService");
        helloWorldService.helloWorld();

        
    }

}
```



## step6

之前的 step-1 到 step-5 已经基本实现了 spring IOC 的功能
但是使用起来比较麻烦
回想一下之前使用 spring 的时候
一般方法会这样写

```java
ApplicationContext ac = new ClassPathXmlApplicationContext("applicationContext.xml")
HelloWorldService helloWorldService = ac.getBean(HelloWorldService.class);
```

更加简洁优雅，这一步按照此方式进行优化
定义一个 ApplicationContext 接口继承 BeanFactory 接口

```java
public interface ApplicationContext extends BeanFactory {
}
```

定义一个抽象类 AbstractApplicationContext 实现 ApplicationContext 接口
refresh 方法在子类进行实现，用来初始化 bean

```java
public abstract class AbstractApplicationContext implements ApplicationContext {
    protected AbstractBeanFactory beanFactory;

    public AbstractApplicationContext(AbstractBeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    public void refresh() throws Exception{
    }

    @Override
    public Object getBean(String name) throws Exception {
        return beanFactory.getBean(name);
    }
}
```

ClassPathXmlApplicationContext 继承抽象类 AbstractApplicationContext
实现 refresh 方法
与 step-5 比较，核心代码没有修改
只是将初始化 AutowireCapableBeanFactory 容器
XmlBeanDefinitionReader 解析 xml 文件
初始化 bean 的操作 都封装到了这里面

```java
public class ClassPathXmlApplicationContext extends AbstractApplicationContext {
	
	private String configLocation;

	public ClassPathXmlApplicationContext(String configLocation) throws Exception {
		this(configLocation, new AutowireCapableBeanFactory());
	}

	public ClassPathXmlApplicationContext(String configLocation, AbstractBeanFactory beanFactory) throws Exception {
		super(beanFactory);
		this.configLocation = configLocation;
		refresh();
	}

	@Override
	public void refresh() throws Exception {
		XmlBeanDefinitionReader xmlBeanDefinitionReader = new XmlBeanDefinitionReader(new ResourceLoader());
		xmlBeanDefinitionReader.loadBeanDefinitions(configLocation);
		for (Map.Entry<String, BeanDefinition> beanDefinitionEntry : xmlBeanDefinitionReader.getRegistry().entrySet()) {
			beanFactory.registerBeanDefinition(beanDefinitionEntry.getKey(), beanDefinitionEntry.getValue());
		}

		/**
		 * Add by me
		 * 源代码中没有这一步
		 * 
		 */
		beanFactory.preInstantiateSingletons();
	}

}
```

example main

```java
public class ExampleMain {
    public static void main(String[] args) throws Exception {
        /**
         * 熟悉的 ApplicationContext
         * 
         */
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("tinyioc.xml");

        HelloWorldService helloWorldService = (HelloWorldService) applicationContext.getBean("helloWorldService");
        helloWorldService.helloWorld();

    }
}
```

封装好之后，初始化 bean 只需一行代码就可以完成