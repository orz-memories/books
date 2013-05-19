# 2.0 PHP 中的新特征

截至目前(2013.5), PHP 的最新稳定版本是 PHP5.4, 但有差不多一半的用户仍在使用已经不在维护[1]的 PHP5.2, 其余的一半用户在使用 PHP5.3[2].  
因为 PHP 那“集百家之长”的蛋疼语法，加上社区氛围不好，很多人对新版本，新特征并无兴趣。  
本章将会介绍自 PHP5.2 起，直至 PHP5.5 中增加的新特征，本书后文若不加说明，默认基于目前的最新稳定版本 PHP5.4.

[1]: 已于2011年1月停止支持： http://www.php.net/eol.php
[2]: http://w3techs.com/technologies/details/pl-php/5/all

## PHP5.2
(2006-2011)

### JSON 支持
包括 json_encode(), json_decode() 等函数，JSON 算是在 Web 领域非常常用的数据交换格式，可以被 JS 直接支持，JSON 实际上是 JS 语法的一部分。  
JSON 系列函数，可以将 PHP 中的数组结构与 JSON 字符串进行转换：

    $array = ["key" => "value", "array" => [1, 2, 3, 4]];
    $json = json_encode($array);
    echo "{$json}\n";

    $object = json_decode($json);
    print_r($object);

输出：

    {"key":"value","array":[1,2,3,4]}
    stdClass Object
    (
        [key] => value
        [array] => Array
            (
                [0] => 1
                [1] => 2
                [2] => 3
                [3] => 4
            )
    )

值得注意的是 json_decode() 默认会返回一个对象而非数组，如果需要返回数组需要将第二个参数设置为 true.

## PHP5.3
(2009-2012)

PHP5.3 算是一个非常大的更新，新增了大量新特征，同时也做了一些不向下兼容的修改。

### 匿名函数

### 命名空间
PHP的命名空间有着前无古人后无来者的无比蛋疼的语法：

    <?php
    // 命名空间的分隔符是反斜杠，该声明语句必须在文件第一行。
    // 命名空间中可以包含任意代码，但只有 **类, 函数, 常量** 受命名空间影响。
    namespace XXOO\Test;

    // 该类的完整限定名是 \XXOO\Test\A , 其中第一个反斜杠表示全局命名空间。
    class A{}

    // 你还可以在已经文件中定义第二个命名空间，接下来的代码将都位于 \Other\Test2 .
    namespace Other\Test2;

    // 实例化来自其他命名空间的对象：
    $a = new \XXOO\Test\A;
    class B{}

    // 你还可以用花括号定义第三个命名空间
    namespace Other {
        // 实例化来自子命名空间的对象：
        $b = new Test2\B;

        // 导入来自其他命名空间的名称，并重命名，
        // 注意只能导入类，不能用于函数和常量。
        use \XXOO\Test\A as ClassA
    }

更多有关命名空间的语法介绍请参见官网[3].

命名空间时常和 autoload 一同使用，用于自动加载类实现文件：

    spl_autoload_register(
        function ($class) {
            spl_autoload(str_replace("\\", "/", $class));
        }
    );

当你实例化一个类 \XXOO\Test\A 的试试，这个类的完整限定名会被传递给 autoload 函数，autoload 函数将类名中的命名空间分隔符(反斜杠)替换为斜杠，并包含对应文件。  
这样可以实现类定义文件分级储存，按需自动加载。

[3]: http://www.php.net/manual/zh/language.namespaces.php

### 魔术方法：__invoke(), __callStatic()

### 弃用的功能
使用这些功能会被发出警告。
#### register_globals
#### safe_mode
#### magic_quotes_gpc

### Phar 

### Heredoc 和 nowdoc

### 用 const 声明常量

### 三元运算符简写形式

### 后期静态绑定
