---
layout: post
title: "Ioc容器与laravel服务容器初探"
date: 2019-08-11 21:51:00
description: Ioc容器与laravel服务容器初探
comments: true
tags: 
 - PHP
 - Laravel
---

# 一、Ioc容器
某天，小J心血来潮，决定建造一艘星舰，这艘星舰要搭载“与众不同最时尚，开火肯定棒”的电磁炮。于是他写了一个星舰类：
```php
class ElectromagneticGun
{
    public function fire() {
        echo 'da da da!';
    }
}
class StarShip
{
    protected $gun;
    public function __construct()
    {
        $gun = new ElectromagneticGun();
    }
}
$powerfulStarShip = new StarShip();
```
StarShip在构造时必须要new一个ElectromagneticGun，依靠其实现fire的功能。即StarShip“依赖”于ElectromagneticGun。
这艘星舰火力强大，所向披靡。直到有一天，他碰到了速度奇快的“无敌舰队”。他决定将电磁炮更换“炮弹”速度为光速的激光炮。
```php
class LaserGun
{
    public function fire() {
        echo 'biu biu biu!';
    }
}
```
当然，最直接的办法就是在starShip的构造函数中将electromagneticGun()更改为laserGun()。但小J并不想将他的宝贝星舰大卸八块，然后塞进一个新的东西。于是他对星舰与战炮进行了控制反转（IOC）的改造，采用了依赖注入（DI）的方式。现在，他的星舰与战炮看起来像这个样子了：
```php
interface Gun
{
    public function fire();
}

class ElectromagneticGun implements gun
{
    public function fire() {
        echo 'da da da!';
    }
}

class LaserGun implements Gun
{
    public function fire() {
        echo 'biu biu biu!';
    }
}

class starShip
{
    protected $gun;
    public function __construct(Gun $powerfulGun)
    {
        $gun = $powerfulGun;
    }
}
$newGun = new LaserGun();
$powerfulStarShip = new starShip($newGun);
```
在这次改造中，原本由对象自己new的依赖，变成了在对象外部创建。获得依赖的方式转移（反转）了，即“控制反转”。
装备了激光炮的小J，在与无敌舰队的战斗中大获全胜，缴获了大量的资源。
小J想要建造更多的星舰，每艘星舰搭载一门电磁炮或一门激光炮，但作为一个程序员，他无法忍受每次想要新造一艘星舰，都要手动new一个Gun的对象，然后注入，甚至在有些时候，他会忘记为星舰new一个Gun然后Gun，导致这个星舰无法建造。他想要更加优雅的方式来建造星舰，管理武器。于是，他创建了一个简单的Ioc容器。
```php
class Container
{
    protected $binds = array();


    /**在容器内注册类
     * @param $abstract 类的名称
     * @param $concrete 闭包，在闭包内new一个此类的对象
     */
    public function bind($abstract, $concrete)
    {
        $this->binds[$abstract] = $concrete;
    }

    /**解析一个对象实例
     * @param $abstract 类的名称
     * @param array $parameters
     * @return mixed 新解析出来的对象实例
     */
    public function make($abstract, $parameters = [])
    {
        //请注意这里，将容器本身作为第一个参数传给了闭包
        array_unshift($parameters, $this);

        return call_user_func_array($this->binds[$abstract], $parameters);
    }
}

$container = new Container();
$container->bind('ElectromagneticGun',function (Container $container) {
    return new ElectromagneticGun();
});
$container->bind('LaserGun',function (Container $container) {
    return new LaserGun();
});

$container->bind('StarShip',function (Container $container, $gunType) {
    return new StarShip($container->make($gunType));
});

$starShip1 = $container->make('StarShip',['ElectromagneticGun']);
$starShip2 = $container->make('StarShip',['LaserGun']);
```
在这里我们在容器内存储可生成具体实例的闭包而不是直接存储实例，其原因是显而易见的：实现某种程度的“延迟绑定”，从而减少资源消耗。这种处理技巧在后面我们看Laravel的源码是也会用到。
请注意，我们在注册容器的时候，为了能够使用容器解决StarShip的“次要依赖”（类绑定在了容器中，但生成它的对象还需要其他依赖），将容器本身作为了闭包的第一个参数。
现在，通过Ioc容器这样一个第三者，在创建StarShip对象的时候，我们完全不需要去考虑如何满足它的依赖：只要我们提前绑定好，容器就会为我们解析出我们想要的依赖。
引用几张经典的图片，我们的系统由解耦之前：
<!-- ![](https://i.imgur.com/Ux2Ey8Y.png){:.center-image } -->
![](/resource/images/2019-08-11-ioc-in-laravel/解耦前.png){:.center-image }
变成了解耦之后的：
<!-- ![](https://i.imgur.com/DJhlvza.png){:.center-image } -->
![](/resource/images/2019-08-11-ioc-in-laravel/解耦后-1.png){:.center-image }
<!-- ![](https://i.imgur.com/VDHwgid.png){:.center-image } -->
![](/resource/images/2019-08-11-ioc-in-laravel/解耦后-2.png){:.center-image }
这就是Ioc容器的好处。
另外，成熟的Ioc容器一般都会利用其实现语言的“反射”技术来取代上面需要我们“手写”的闭包，以及自动解决容器内的“次要依赖”。当然，对某些不支持反射的语言，就要另当别论了（没错，我说的就是C++）。。。
# 二、Laravel中的服务容器
Laravel框架是一个容器框架，框架应用程序的实例就是一个超大的Ioc容器，Laravel将其称为“服务容器”。
在每次请求的声明周期中，Laravel 自身的第一个动作就是创建一个服务容器的实例，它在整个请求声明周期中是唯一的。熟悉的，我们可以通过app()这个help函数以及APP这个facade来获取或访问到这个服务容器的实例。
服务容器的实现在 Illuminate\Container\Container.php 中，我们观察一下它的核心代码（精简了大部分代码）：
```php
class Container
{

    //绑定数组
    protected $bindings = [];

    //已有部分实例存储在容器内
    protected $instances = [];

    //绑定一个名称或回调闭包到容器
    public function bind($abstract, $concrete = null, $shared = false)
    {

        if (is_null($concrete)) {
            $concrete = $abstract;
        }

        if (! $concrete instanceof Closure) {
            $concrete = $this->getClosure($abstract, $concrete);
        }

        $this->bindings[$abstract] = compact('concrete', 'shared');

    }


    protected function getClosure($abstract, $concrete)
    {
        return function ($c, $parameters = []) use ($abstract, $concrete) {
            $method = ($abstract == $concrete) ? 'build' : 'make';

            return $c->$method($concrete, $parameters);
        };
    }


    public function make($abstract, array $parameters = [])
    {

        // 若请求的实例已在容器内存在，则直接返回

        if (isset($this->instances[$abstract])) {
            return $this->instances[$abstract];
        }

        // 获取绑定的闭包，若未找到则返回的$abstract的上下文绑定或更改了形式（加斜杠前缀？）的$abstract
        $concrete = $this->getConcrete($abstract);

        if ($this->isBuildable($concrete, $abstract)) {
            $object = $this->build($concrete, $parameters);
        } else {
            $object = $this->make($concrete, $parameters);
        }

        return $object;
    }


    public function build($concrete, array $parameters = [])
    {
        if ($concrete instanceof Closure) {
            return $concrete($this, $parameters);
        }

        $reflector = new ReflectionClass($concrete);
        
        //获取构造函数
        $constructor = $reflector->getConstructor();

        //没有构造函数，就意味着没有依赖，可以立即构造
        if (is_null($constructor)) {
            array_pop($this->buildStack);

            return new $concrete;
        }

        //获取构造函数的所有参数（依赖）
        $dependencies = $constructor->getParameters();

        //解决依赖
        $instances = $this->getDependencies(
            $dependencies, $parameters
        );


        return $reflector->newInstanceArgs($instances);
    }

    //通过反射机制解决所有的依赖
    protected function getDependencies(array $parameters, array $primitives = [])
    {
        $dependencies = [];

        foreach ($parameters as $parameter) {
            $dependency = $parameter->getClass();

            if (array_key_exists($parameter->name, $primitives)) {
                $dependencies[] = $primitives[$parameter->name];
            } elseif (is_null($dependency)) {
                $dependencies[] = $this->resolveNonClass($parameter);   //解决一个不是类的依赖：若允许默认值则返回默认值，否则抛出异常
            } else {
                $dependencies[] = $this->resolveClass($parameter);      //解决一个类的依赖：从Container内解析此类。
            }
        }

        return (array) $dependencies;
    }

}
```
bind() 方法中会处理两种情况：
1. concrete 为闭包：直接添加至bindings数组
2. concrete 为类名：生成一个闭包将其包裹起来（ getClosure方法）

make()方法会查找请求的实例，若已在容器内构造好了，则直接返回构造好的实例。
否则利用isBuildable方法判断是否可构造（判断标准为是否已注册或为明确的类名），若可构造则利用build方法进行构造，否则更改abstract继续查找。

build() 方法会检测concrete是否为闭包。若为闭包，直接构造即可。若为类名，则利用反射机制进行构造，还要解决其所有依赖。

以上的服务容器源码是大幅精简之后的，完整的源码还有许多诸如别名、绑定实体、获取容器属性等等的处理与方法。但其实我们可以发现其和我们在上面实现的“幼儿园”版Ioc容器的思想是一样的：利用一个array存储可生成对象的闭包，从而能够作为一个第三者，为各组件提供自动化的依赖注入，实现了组件之间的解耦。

# 三、服务提供者
只要能够拿到服务容器的实例，我们就可以进行类（服务）的绑定。但若我们不进行一个统一的绑定/初始化操作，可能会出现需要某一对象时，忘记绑定其依赖的情况。那岂不是失去了服务容器的原本意义？所以Laravel提供了服务提供者，能够使我们在业务代码开始之前，应用程序初始化的时候进行绑定。
laravel加载自定义服务提供者的时候，是从config/app.php这个配置文件里面的providers中找到所有的服务提供者的。
所以你如果自己写了一个服务提供者，那么只要配置到这里面，laravel就会自动帮你注册它了。

其主要提供了两个方法：
register与boot
Laravel通过配置文件找到服务提供者的类后，构造一个对象，然后会调用这个对象的register方法。所以我们可以在服务提供者的register方法中将类（服务）绑定至服务容器中。注意一定不要在register方法中使用一些未注册的类（服务）。
在所有的服务提供者都注册完成之后，会执行boot方法，所以这时候就可以尽情地使用服务容器来解析出实例来了。

参考资料：
http://www.cnblogs.com/xingyukun/archive/2007/10/20/931331.html
https://www.martinfowler.com/articles/injection.html
http://www.cnblogs.com/DebugLZQ/archive/2013/06/05/3107957.html
https://laravel-china.org/articles/789/laravel-learning-notes-the-magic-of-the-service-container
https://www.cnblogs.com/lyzg/p/6181055.html
http://www.ituring.com.cn/article/505006