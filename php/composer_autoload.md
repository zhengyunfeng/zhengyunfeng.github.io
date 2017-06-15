# composer的自动加载机制

直接上代码执行流程：

1. 代码清单 public/index.php

```
require __DIR__.'/../bootstrap/autoload.php';
$app = require_once __DIR__.'/../bootstrap/start.php';
$app->run();
```

第一行引入了 bootstrap/autoload.php。

2. 代码清单 bootstrap/autoload.php

```
define('LARAVEL_START', microtime(true));
require __DIR__.'/../vendor/autoload.php';
if (file_exists($compiled = __DIR__.'/compiled.php'))
{
    require $compiled;
}
```

第一行定义了程序开始执行的时间点。紧接着第二行，引入了 vendor/autoload.php。

3. 代码清单 vendor/autoload.php

```
// autoload.php @generated by Composer
 
require_once __DIR__ . '/composer' . '/autoload_real.php';
 
return ComposerAutoloaderInitd5eba667adbfad9bcb7ebad25841cbfe::getLoader();
```

到这里，马上就进入自动加载的大门了。

4. 代码清单vendor/composer/autoload_real.php

```
class ComposerAutoloaderInitd5eba667adbfad9bcb7ebad25841cbfe
{
    private static $loader;

    public static function loadClassLoader($class)
    {
        if ('Composer\Autoload\ClassLoader' === $class) {
            require __DIR__ . '/ClassLoader.php';
        }
    }

    public static function getLoader()
    {
        if (null !== self::$loader) {
            return self::$loader;
        }

        spl_autoload_register(array('ComposerAutoloaderInitd5eba667adbfad9bcb7ebad25841cbfe', 'loadClassLoader'), true, true);
        self::$loader = $loader = new \Composer\Autoload\ClassLoader();
        spl_autoload_unregister(array('ComposerAutoloaderInitd5eba667adbfad9bcb7ebad25841cbfe', 'loadClassLoader'));

        $map = require __DIR__ . '/autoload_namespaces.php';
        foreach ($map as $namespace => $path) {
            $loader->set($namespace, $path);
        }

        $map = require __DIR__ . '/autoload_psr4.php';
        foreach ($map as $namespace => $path) {
            $loader->setPsr4($namespace, $path);
        }

        $classMap = require __DIR__ . '/autoload_classmap.php';
        if ($classMap) {
            $loader->addClassMap($classMap);
        }

        $loader->register(true);

        $includeFiles = require __DIR__ . '/autoload_files.php';
        foreach ($includeFiles as $file) {
            composerRequired5eba667adbfad9bcb7ebad25841cbfe($file);
        }

        return $loader;
    }
}

function composerRequired5eba667adbfad9bcb7ebad25841cbfe($file)
{
    require $file;
}
```

加载过程可用下图来描述

![image](http://opisr9ja7.bkt.clouddn.com/composer_autoload1.png)

Composer按照四种规范来加载文件：

- psr-4
- psr-0(这种规范某些部分不是很优雅)
- classmap(命名空间和文件路径的映射)
- files

composerRequired5eba667adbfad9bcb7ebad25841cbfe这个类是composer为了防止类冲突搞了一个命名ComposerAutoloaderInit+hash，不管咋样，require_once这个类后需要返回的是一个加载器$loader，而这个加载器经过四种规范遍历后，由null被填充为含有各种变量值的ClassLoader对象。

如果仔细观察autoload_classmap.php、autoload_namespaces.php、autoload_psr4.php和autoload_files文件后，这些都按照对应的规范返回要么命名空间与路径的映射，要么完整路径与某个哈希的映射。

从上图中能看出这个composer初始化路径的流程，重点是ClassLoader这个类的loadClass($class)这个方法，是通过spl_autoload_register这个PHP自动加载函数来注册到autoload函数栈中，最后返回一个$loader加载器。

```
//ClassLoader类
public function register($prepend = false)
{
    spl_autoload_register(array($this, 'loadClass'), true, $prepend);
}

public function loadClass($class)
{
    if ($file = $this->findFile($class)) {
        includeFile($file);

        return true;
    }
}
```
需要实例化一个类时，就会根据loadClass($class)来寻找对应的文件，上面看到loadClass方法中有个findFile($class)方法。

```
public function findFile($class)
{
    // work around for PHP 5.3.0 - 5.3.2 https://bugs.php.net/50731
    if ('\\' == $class[0]) {
        $class = substr($class, 1);
    }

    // class map lookup
    if (isset($this->classMap[$class])) {
        return $this->classMap[$class];
    }
    if ($this->classMapAuthoritative) {
        return false;
    }

    $file = $this->findFileWithExtension($class, '.php');

    // Search for Hack files if we are running on HHVM
    if ($file === null && defined('HHVM_VERSION')) {
        $file = $this->findFileWithExtension($class, '.hh');
    }

    if ($file === null) {
        // Remember that this class does not exist.
        return $this->classMap[$class] = false;
    }

    return $file;
}
```

这个文件里先做classmap查找，然后进入findFileWithExtension($class,'.php')中做psr-4/psr-0查找，其实就是搜寻psr-4和psr-0的私有变量值，查找文件绝对路径，返回一个$file，再include下就等于这个类可以被实例化了。

over！

总结一下：
1. 初始化时，将命名空间和文件路径映射到ClassLoader类的各个变量里。
2. 把ClassLoader类的loadClass方法注册到autoload栈中。
3. 需要new一个类时，调用loadClass方法去ClassLoader类的各个变量里查找类对应的文件，然后include，就完成了自动加载。


###### [back to PHP](https://zhengyunfeng.github.io/php/index)