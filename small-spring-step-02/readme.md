本节主要是根据单一职责对功能进行拆分，作为一个IOC容器，对外的核心功能是BeanFactory，即获取bean的接口
## 需求分析
首先我们考虑获取bean的时候需要做些什么。
1、 考虑单例，对于单例我们可以直接尝试从单例缓存中获取
2、 考虑非单例，或单例尚未创建时获取不到我们需要根据bean的定义进行bean的创建

根据以上步骤，我们得到抽象BeanFactory的基本结构如下：

```java
package cn.bugstack.springframework.beans.factory.support;

import cn.bugstack.springframework.beans.BeansException;
import cn.bugstack.springframework.beans.factory.BeanFactory;
import cn.bugstack.springframework.beans.factory.config.BeanDefinition;


public abstract class AbstractBeanFactory implements BeanFactory {
    
    public Object getBean(String name) throws BeansException {
        Object bean = getSingleton(name);
        if (bean != null) {
            return bean;
        }

        BeanDefinition beanDefinition = getBeanDefinition(name);
        return createBean(name, beanDefinition);
    }

    protected abstract Object getSingleton(String beanName);

    protected abstract BeanDefinition getBeanDefinition(String beanName) throws BeansException;

    protected abstract Object createBean(String beanName, BeanDefinition beanDefinition) throws BeansException;

}
```

接下来进行单一职责的拆分

## 单例功能

单例这一块与beanFactory的关联性稍弱，将其拆分出去，拆分后由于依赖抽象所以先定接口

```java
package cn.bugstack.springframework.beans.factory.config;


public interface SingletonBeanRegistry {

    Object getSingleton(String beanName);

}
```
然后进行实现

```java
package cn.bugstack.springframework.beans.factory.support;

import cn.bugstack.springframework.beans.factory.config.SingletonBeanRegistry;

import java.util.HashMap;
import java.util.Map;


public class DefaultSingletonBeanRegistry implements SingletonBeanRegistry {

    private final Map<String, Object> singletonObjects = new HashMap<>();

    @Override
    public Object getSingleton(String beanName) {
        return singletonObjects.get(beanName);
    }

    protected void addSingleton(String beanName, Object singletonObject) {
        singletonObjects.put(beanName, singletonObject);
    }

}
```

这个实现其实是比较简单的，就是利用map做一个bean的缓存。
这里需要注意的是addSingleton方法，我们考虑单例注册表中的对象肯定不是凭空变出来的，而且需要注意封装性，singletonObjects不能对外暴露
肯定需要提供一个添加方法。然后注意这个方法的访问修饰符为protected，最小可见性，添加方法主要是提供给子类进行添加的


## bean的实例化
接下来是bean的实例化createBean方法
考虑到bean的实例化本身可能比较复杂，未来可能有扩展需求，为了主线清晰，所以将其放到子类进行实现

```java
package cn.bugstack.springframework.beans.factory.support;

import cn.bugstack.springframework.beans.BeansException;
import cn.bugstack.springframework.beans.factory.config.BeanDefinition;

public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory {

    @Override
    protected Object createBean(String beanName, BeanDefinition beanDefinition) throws BeansException {
        Object bean;
        try {
            bean = beanDefinition.getBeanClass().newInstance();
        } catch (InstantiationException | IllegalAccessException e) {
            throw new BeansException("Instantiation of bean failed", e);
        }

        addSingleton(beanName, bean);
        return bean;
    }

}
```

## bean定义管理
最后AbstractBeanFactory中的getBeanDefinition与bean定义的管理关联性较大，应当让他们处于同一个类中

```java
package cn.bugstack.springframework.beans.factory.support;

import cn.bugstack.springframework.beans.BeansException;
import cn.bugstack.springframework.beans.factory.config.BeanDefinition;

import java.util.HashMap;
import java.util.Map;


public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory implements BeanDefinitionRegistry {

    private final Map<String, BeanDefinition> beanDefinitionMap = new HashMap<>();

    @Override
    public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) {
        beanDefinitionMap.put(beanName, beanDefinition);
    }

    @Override
    public BeanDefinition getBeanDefinition(String beanName) throws BeansException {
        BeanDefinition beanDefinition = beanDefinitionMap.get(beanName);
        if (beanDefinition == null) throw new BeansException("No bean named '" + beanName + "' is defined");
        return beanDefinition;
    }

}
```
添加了一个注册备案的方法registerBeanDefinition，同时为了后续依赖抽象将其抽取成一个接口BeanDefinitionRegistry

本节过后IOC基本雏形已经有了。
