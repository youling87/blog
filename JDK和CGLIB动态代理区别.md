# JDK和CGLIB动态代理区别

一 JDK和CGLIB动态代理原理
1、JDK动态代理
利用拦截器(拦截器必须实现InvocationHanlder)加上反射机制生成一个实现代理接口的匿名类，

在调用具体方法前调用InvokeHandler来处理。

2、CGLIB动态代理
利用ASM开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。

3、何时使用JDK还是CGLIB？
1）如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP。

2）如果目标对象实现了接口，可以强制使用CGLIB实现AOP。

3）如果目标对象没有实现了接口，必须采用CGLIB库，Spring会自动在JDK动态代理和CGLIB之间转换。

4、如何强制使用CGLIB实现AOP？
1）添加CGLIB库(aspectjrt-xxx.jar、aspectjweaver-xxx.jar、cglib-nodep-xxx.jar)

2）在Spring配置文件中加入<aop:aspectj-autoproxy proxy-target-class="true"/>

5、JDK动态代理和CGLIB字节码生成的区别？
1）JDK动态代理只能对实现了接口的类生成代理，而不能针对类。

2）CGLIB是针对类实现代理，主要是对指定的类生成一个子类，覆盖其中的方法，

     并覆盖其中方法实现增强，但是因为采用的是继承，所以该类或方法最好不要声明成final，
    
     对于final类或方法，是无法继承的。

6、CGlib比JDK快？
1）使用CGLib实现动态代理，CGLib底层采用ASM字节码生成框架，使用字节码技术生成代理类，

在jdk6之前比使用Java反射效率要高。唯一需要注意的是，CGLib不能对声明为final的方法进行代理，

因为CGLib原理是动态生成被代理类的子类。

2）在jdk6、jdk7、jdk8逐步对JDK动态代理优化之后，在调用次数较少的情况下，JDK代理效率高于CGLIB代理效率，

只有当进行大量调用的时候，jdk6和jdk7比CGLIB代理效率低一点，但是到jdk8的时候，jdk代理效率高于CGLIB代理，

总之，每一次jdk版本升级，jdk代理效率都得到提升，而CGLIB代理消息确有点跟不上步伐。

7、Spring如何选择用JDK还是CGLIB？
1）当Bean实现接口时，Spring就会用JDK的动态代理。

2）当Bean没有实现接口时，Spring使用CGlib是实现。

3）可以强制使用CGlib（在spring配置中加入<aop:aspectj-autoproxy proxy-target-class="true"/>）。

二 代码实例
接口：

package com.jpeony.spring.proxy.compare;

/**
 * 用户管理接口(真实主题和代理主题的共同接口，这样在任何可以使用真实主题的地方都可以使用代理主题代理。)
 * --被代理接口定义
 */
public interface IUserManager {
    void addUser(String id, String password);
}
实现类：

package com.jpeony.spring.proxy.compare;

/**
 * 用户管理接口实现(被代理的实现类)
 */
public class UserManagerImpl implements IUserManager {

    @Override
    public void addUser(String id, String password) {
        System.out.println("======调用了UserManagerImpl.addUser()方法======");
    }
}
JDK代理实现：

package com.jpeony.spring.proxy.compare;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

/**
 * JDK动态代理类
 */
public class JDKProxy implements InvocationHandler {
    /** 需要代理的目标对象 */
    private Object targetObject;

    /**
     * 将目标对象传入进行代理
     */
    public Object newProxy(Object targetObject) {
        this.targetObject = targetObject;
        //返回代理对象
        return Proxy.newProxyInstance(targetObject.getClass().getClassLoader(),
                targetObject.getClass().getInterfaces(), this);
    }

    /**
     * invoke方法
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 一般我们进行逻辑处理的函数比如这个地方是模拟检查权限
        checkPopedom();
        // 设置方法的返回值
        Object ret = null;
        // 调用invoke方法，ret存储该方法的返回值
        ret  = method.invoke(targetObject, args);
        return ret;
    }

    /**
     * 模拟检查权限的例子
     */
    private void checkPopedom() {
        System.out.println("======检查权限checkPopedom()======");
    }
}
CGLIB代理实现：

package com.jpeony.spring.proxy.compare;

import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

/**
 * CGLibProxy动态代理类
 */
public class CGLibProxy implements MethodInterceptor {
    /** CGLib需要代理的目标对象 */
    private Object targetObject;

    public Object createProxyObject(Object obj) {
        this.targetObject = obj;
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(obj.getClass());
        enhancer.setCallback(this);
        Object proxyObj = enhancer.create();
        // 返回代理对象
        return proxyObj;
    }

    @Override
    public Object intercept(Object proxy, Method method, Object[] args,
                            MethodProxy methodProxy) throws Throwable {
        Object obj = null;
        // 过滤方法
        if ("addUser".equals(method.getName())) {
            // 检查权限
            checkPopedom();
        }
        obj = method.invoke(targetObject, args);
        return obj;
    }

    private void checkPopedom() {
        System.out.println("======检查权限checkPopedom()======");
    }
}
客户端测试类：

package com.jpeony.spring.proxy.compare;

/**
 * 代理模式[[ 客户端--》代理对象--》目标对象 ]]
 */
public class Client {
    public static void main(String[] args) {
        System.out.println("**********************CGLibProxy**********************");
        CGLibProxy cgLibProxy = new CGLibProxy();
        IUserManager userManager = (IUserManager) cgLibProxy.createProxyObject(new UserManagerImpl());
        userManager.addUser("jpeony", "123456");

        System.out.println("**********************JDKProxy**********************");
        JDKProxy jdkPrpxy = new JDKProxy();
        IUserManager userManagerJDK = (IUserManager) jdkPrpxy.newProxy(new UserManagerImpl());
        userManagerJDK.addUser("jpeony", "123456");
    }
}
程序运行结果：



三 JDK和CGLIB动态代理总结
JDK代理是不需要第三方库支持，只需要JDK环境就可以进行代理，使用条件:

1）实现InvocationHandler 

2）使用Proxy.newProxyInstance产生代理对象

3）被代理的对象必须要实现接口

CGLib必须依赖于CGLib的类库，但是它需要类来实现任何接口代理的是指定的类生成一个子类，

覆盖其中的方法，是一种继承但是针对接口编程的环境下推荐使用JDK的代理；