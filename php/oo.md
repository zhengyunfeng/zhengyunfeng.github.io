# PHP面向对象的若干特性

## static

### 声明静态属性和方法
static关键字的作用，通常是用来声明静态属性和方法，其特点为无需实例化即可被访问到，static属性在内存中只有一份，为所有的实例共用。

详细的用法这里不赘述，需要注意的是：

1. 静态方法不能使用非静态属性

```
class PI
{
	public $pi = 3.14;
	public static function get_pi() {
		return self::$pi;   //静态方法不能使用非静态属性
	}
}

echo PI::get_pi();
```

2. 静态方法调用非静态方法，只能用self::而不能使用$this->，此时系统会自动将这个方法转换为静态方法。

```
class PI
{
	public static $pi = 3.14;
	public function get_pi() {
		return self::$pi;
	}

	public static function get_pi_square() {
		return self::get_pi() * self::get_pi();  //这种用法不会报错
	}
}

echo PI::get_pi_square();
```

一般来说，若属性或方法与类和实例无关，则可以声明为static。PHP因为历史遗留问题，命名空间对函数支持比较差。因此，在PHP中将与实例无关的函数定义为一个类的静态方法，是一种比较常见的设计，在 Laravel 中大量被用到。

### 延迟静态绑定

这是static的另一个用法，表示当前正在运行的类。什么意思呢，这里涉及到php5.3开始支持的一个特性：延迟静态绑定(late static binding)。

```
class A 
{ 
  public static function echoClass(){ 
    echo __CLASS__; 
  }
  public static function test(){ 
    self::echoClass();    
  }
}
class B extends A 
{    
  public static function echoClass() 
  { 
     echo __CLASS__; 
  } 
} 
B::test();
```

这里的输出是A，因为self表示的是所在类。若想输出B，只需把self改成static即可。

这就是延迟静态绑定，正式些的定义是：该特性允许在一个静态继承的上下文中对一个被调用类的引用。说白了就是父类可使用子类重载的方法，在上面的例子中，改成static后，父类的test使用了子类的echoClass方法。

## 引用

在PHP中，引用的意思是：不同的名字访问同一个变量内容。与Ｃ语言中的指针是有差别的，Ｃ语言中的指针里面存储的是变量的内容在内存中存放的地址，而PHP中，你可以打印一下&$a试试。

### 变量引用

首先看一下最常用的变量引用，没什么可讲的，示例如下：

```
$a = "aaa";
$b = &$a;  //变量引用
$b = "bbb";
echo $a;  //输出bbb

function change(&$c) { //函数的传址调用
    $c = "ccc";
}
change($b);
echo $a;  //输出ccc
```

### 函数的引用返回

手册里是这么写的：引用返回用在当想用函数找到引用应该被绑定在哪一个变量上面时。不要用返回引用来增加性能，引擎足够聪明来自己进行优化。仅在有合理的技术原因时才返回引用！

```
class foo {
    public $value = 42;
    
    public function &getValue() {
        return $this->value;
    }
}

$obj = new foo;

// $myValue 是 $obj->value 的引用，它们的值都是 42
$myValue = &$obj->getValue();

$obj->value = 2;
echo $myValue;  // 程序输出 2
```

和参数传递不同，这里必须在两个地方都用&符号指出返回的是一个引用，而不是通常的一个拷贝，同样也指出$myValue是作为引用的绑定，而不是通常的赋值。

“引用返回用在当想用函数找到引用应该被绑定在哪一个变量上面时”，意思就是说，函数&getValue()把引用绑定在成员变量$value上。

再看个例子：

```
function &func($a,$b){ 
    // 这里为了更直观看到效果，定义一个静态变量
    static $result = 0;    
    $result += $a + $b;
    return $result;
}

// PHP里这样写函数的引用调用，和调用普通函数没有区别（只是将函数的返回值复制给$c这个变量，$c做任何改变不会影响上面函数中的$result），PHP里的函数引用定义及调用都要在函数名前加上 &
$c = func(10,10); // $result的值变为 20(10+10)

$c = 666; 
$c = func(10,10); // $result的值变为 40(20+10+10)

$d = &func(10,10); // 这样才是PHP中引用函数的调用方式，$result的值变为 60(40+10+10)

$d = 888; // $d实际上是$result的引用
$d = func(10,10); //$result的值变为 908(888+10+10)
```

### 对象引用

在PHP中，对象间的赋值操作实际上是引用操作，不再展开讲，这里讲一下clone。在有些时候，我们希望能完全复制一个对象，可以用关键字clone。

```
class myclass {
    public $data;
}

$obj1 = new myclass();
$obj1->data = "aaa";

$obj2 = $obj1;       // 引用
$obj2->data = "bbb"; // $obj1->data的值也被改为"bbb"

$obj3 = clone $obj1;
$obj3->data = "ccc"; // $obj1->data的值仍然为"bbb"
```

但是，PHP的object clone采用的是浅拷贝(shallow copy)的方法，如果对象里的属性成员本身就是引用类型的，clone以后这些成员并没有被真正复制，仍然是引用的。

此时，就需要用到__clone()方法，该方法在使用clone关键字时被调用，方法里再对内部的引用使用clone操作，确保作为引用进行处理的类属性能够正确的被复制。

### unset

unset 一个引用，只是断开了变量名和变量内容之间的绑定，这并不意味着变量内容被销毁了。

```
$a = 1; 
$b = &$a; 
unset ($a);
echo $b; // $b仍然有值
```

### 补充个知识点

php中对于地址的指向（类似指针）功能不是由用户自己来实现的，是由Zend核心实现的，php中引用采用的是“写时拷贝”的原理，就是除非发生写操作，指向同一个地址的变量或者对象是不会被拷贝的。 

```
$a = "ABC"; 
$b = $a; // $a与$b都是指向同一内存地址　而并不是$a与$b占用不同的内存 
$a = "EFG"; // 由于$a与$b所指向的内存的数据要重新写一次了，此时Zend核心会自动判断，自动为$b生产一个$a的数据拷贝，重新申请一块内存进行存储
```

## __call()和重载

通常所说的重载，是指函数名相同但函数的参数列表不同(包括参数个数和参数类型)的情况，这两点对于PHP而言，都无法实现：你可以对函数多添加参数，只是相当于多传了个临时变量，而且PHP传参数不区分类型。那么，PHP怎么实现类似重载的功能呢？

这个时候就要利用它的__call()魔术方法了，在PHP5.3.0之后，又增加了一个__callStatic()方法。

```
class  MethodTest {
    public function __call($name,$arguments) 
    {
        echo "Calling object method '$name' 的参数有多个，分别是:" . implode ('、', $arguments);
    }
    
    public static function __callStatic($name,$arguments) 
    {
        echo  "Calling static method '$name' 的参数有多个，分别是:" . implode ('、', $arguments);
    }
}

$obj=new MethodTest ;
$obj->runTest('in object context', '另外一个参数');
MethodTest::runTest ('in static context', '另外一个参数');
```

输出为
```
Calling object method 'runTest' 的参数有多个，分别是:in object context、另外一个参数
Calling static method 'runTest' 的参数有多个，分别是:in static context、另外一个参数
```

在类外部调用一个类中的一个不可访问（不可访问不仅仅代表不存在）的方法时，对象方式就触发__call()方法，静态方式就触发__callStatic()方法。

触发__call()或__callStatic()方法时，系统会自动将所调用的那个不可访问的方法的方法名作为第一个参数传入__call()或__callStatic()方法中，而将传入的参数，作为第二个参数(而且是封装成了一个数组)传入__call()或__callStatic()方法中。

可以利用这个特性实现重载，下面是个例子。

```
class Foo1{
    public function __call($name,$arguments){
        if($name == "doStuff"){
            if(is_int($arguments[0])){
                $this->doStuffForInt($arguments[0]);
            }else if(is_string($arguments[0])){
                $this->doStuffForString($arguments[0]);
            }
        }
    }

    private function doStuffForInt($a){
        echo "执行的是doStuffForInt()方法";
    }

    private function doStuffForString($a){
        echo "执行的是doStuffForString()方法";
    }
}

$foo1=new Foo1;

$foo1->doStuff('1');  //执行的是doStuffForString()方法
$foo1->doStuff(1);    //执行的是doStuffForInt()方法
```

## __autoload()和spl_autoload

首先要说的是，__autoload()不是一个类方法，而是一个单独的函数，也就是说，可以在任何类声明之外声明这个函数。如果实现了这个函数，它将在实例化一个没有被声明的类时自动调用。

```
// testA.php文件
class testA{
	public function __construct() {
		echo "testA constructed";
	}
}

// 同级目录下新建testB.php文件
function __autoload($className) {
	$classPath = "./" . $className . ".php";
	if (file_exists($classPath)) {
		require_once($classPath);
	}
}

new testA(); // 输出testA constructed
```

testB文件中，没有require testA类，但是却可以new成功，就是因为autoload方法中require_once了testA文件。

PHP5.1.2后，增加了spl_autoload系列方法，SPL是Standard PHP Library(标准PHP库)的缩写。它是PHP5引入的一个扩展库，其主要功能包括autoload机制的实现及包括各种Iterator接口或类。SPL autoload机制的实现是通过将函数指针autoload_func指向自己实现的具有自动装载功能的函数来实现的。

核心是spl_autoload_register方法，看下这个方法的定义：

bool spl_autoload_register ([ callback $autoload_function ] ) 

参数autoload_function表示要注册的自动装载函数，可以是类名::函数名或者array(类名,函数名)的形式。

把上面例子中的testB文件改一下：

```
function __autoload($className) {
	echo "autoload by function __autoload \r\n";
	$classPath = "./" . $className . ".php";
	if (file_exists($classPath)) {
		require_once($classPath);
	}
}

class testB {
	public static function autoload($className) {
		echo "autoload by class testB function autoload \r\n";
		$classPath = "./" . $className . ".php";
		if (file_exists($classPath)) {
			require_once($classPath);
		}
	}
}

spl_autoload_register(array('testB', 'autoload'));

new testA(); //不再调用__autoload()，输出autoload by class testB function autoload

```

那么，为什么在__autoload后，又要定义一个spl_autoload来取代它呢？

文档里有这么句话：spl_autoload_register 可以很好地处理需要多个加载器的情况，这种情况下 spl_autoload_register 会按顺序依次调用之前注册过的加载器。作为对比， __autoload 因为是一个函数，所以只能被定义一次。

另外，SPL函数很丰富，可以提供更多功能，如spl_autoload_unregister()注销已经注册的函数、spl_autoload_functions()返回所有已经注册的函数等

扩展：[laravel中composer的自动加载机制](http://zhengyunfeng.github.io/php/composer_autoload)


## __toString()

__toString()方法在直接输出对象引用时自动调用的。前面我们讲过对象引用是一个指针，比如说：$p = new Person()中，$p就是一个引用，我们不能使用echo直接输出$p, 这样会输出"Catchable fatal error: Object of class Person could not be converted to string"这样的错误，如果你在类里面定义了__toString()方法，在直接输出对象引用的时候，就不会产生错误，而是自动调用了__toString()方法

```
class testA{
	public $a;
	public $b;

	public function __toString() {
		return(var_export($this, TRUE)); //var_export打印类中所有的属性
	}

}

echo new testA();
```

## 反射

PHP 5 具有完整的反射 API，添加了对类、接口、函数、方法和扩展进行反向工程的能力。 此外，反射 API提供了方法来取出函数、类和方法中的文档注释。详细API文档见http://php.net/manual/zh/intro.reflection.php

这里不多说反射的用法。我们都知道依赖注入里边用到了反射，这里简单写个反射实现依赖注入的例子

```
/**
 *
 * 工具类，使用该类来实现自动依赖注入。
 *
 */
class Ioc {
    // 获得类的对象实例
    public static function getInstance($className) {

        $paramArr = self::getMethodParams($className);

        return (new ReflectionClass($className))->newInstanceArgs($paramArr);
    }

    /**
     * 执行类的方法
     * @param  [type] $className  [类名]
     * @param  [type] $methodName [方法名称]
     * @param  [type] $params     [额外的参数]
     * @return [type]             [description]
     */
    public static function make($className, $methodName, $params = []) {

        // 获取类的实例
        $instance = self::getInstance($className);

        // 获取该方法所需要依赖注入的参数
        $paramArr = self::getMethodParams($className, $methodName);

        return $instance->{$methodName}(...array_merge($paramArr, $params));
    }

    /**
     * 获得类的方法参数，只获得有类型的参数
     * @param  [type] $className   [description]
     * @param  [type] $methodsName [description]
     * @return [type]              [description]
     */
    protected static function getMethodParams($className, $methodsName = '__construct') {

        // 通过反射获得该类
        $class = new ReflectionClass($className);
        $paramArr = []; // 记录参数，和参数类型

        // 判断该类是否有构造函数
        if ($class->hasMethod($methodsName)) {
            // 获得构造函数
            $construct = $class->getMethod($methodsName);

            // 判断构造函数是否有参数
            $params = $construct->getParameters();

            if (count($params) > 0) {

                // 判断参数类型
                foreach ($params as $key => $param) {

                    if ($paramClass = $param->getClass()) {

                        // 获得参数类型名称
                        $paramClassName = $paramClass->getName();

                        // 获得参数类型
                        $args = self::getMethodParams($paramClassName);
                        $paramArr[] = (new ReflectionClass($paramClass->getName()))->newInstanceArgs($args);
                    }
                }
            }
        }

        return $paramArr;
    }
}
```

再定义三个类

```
class A {
    protected $cObj;

    /**
     * 用于测试多级依赖注入 B依赖A，A依赖C
     * @param C $c [description]
     */
    public function __construct(C $c) {
        $this->cObj = $c;
    }

    public function aa() {
        echo 'this is A->test';
    }

    public function aac() {
        $this->cObj->cc();
    }
}

class B {
    protected $aObj;

    /**
     * 测试构造函数依赖注入
     * @param A $a [使用引来注入A]
     */
    public function __construct(A $a) {
        $this->aObj = $a;
    }

    /**
     * [测试方法调用依赖注入]
     * @param  C      $c [依赖注入C]
     * @param  string $b [这个是自己手动填写的参数]
     * @return [type]    [description]
     */
    public function bb(C $c, $b) {
        $c->cc();
        echo "\r\n";
        echo 'params:' . $b;
    }

    /**
     * 验证依赖注入是否成功
     * @return [type] [description]
     */
    public function bbb() {
        $this->aObj->aac();
    }
}

class C {
    public function cc() {
        echo 'this is C->cc';
    }
}
```

实际调用的代码是：

```
// 使用Ioc来创建B类的实例，B的构造函数依赖A类，A的构造函数依赖C类。
$bObj = Ioc::getInstance('B');

// 打印$bObj
var_dump($bObj);

Ioc::make('B', 'bb', ['this is param b']);
```

输出为：

```
// var_dump($bObj)的输出
object(B)#3 (1) {
  ["aObj":protected]=>
  object(A)#7 (1) {
    ["cObj":protected]=>
    object(C)#10 (0) {
    }
  }
}

// Ioc::make('B', 'bb', ['this is param b'])的输出
this is C->cc
params:this is param b
```

可以看到实例化B对象时，自动将其依赖的A和C都实例化了。这段代码的核心在getMethodParams方法递归地获取并实例化__construct方法的参数。实际的依赖注入中对反射的使用比这个例子复杂得多，这里只是说明一下原理。

补充（控制反转和依赖注入）：
- 当调用者需要被调用者的协助时，在传统的程序设计过程中，通常由调用者来创建被调用者的实例，但在这里，创建被调用者的工作不再由调用者来完成，而是将被调用者的创建移到调用者的外部，从而反转被调用者的创建，消除了调用者对被调用者创建的控制，因此称为控制反转。

- 要实现控制反转，通常的解决方案是将创建被调用者实例的工作交由 IoC 容器来完成，然后在调用者中注入被调用者（通过构造器/方法注入实现），这样我们就实现了调用者与被调用者的解耦，该过程被称为依赖注入。

###### [back to PHP](https://zhengyunfeng.github.io/php/index)
