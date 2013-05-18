# 1.0 安全

我们将安全放到了最前边，对于Web应用来说，安全的重要性不言而喻，如果你编写的程序存在漏洞，攻击者将有可能取得服务器的控制权，进行破坏，甚至利用你的站点进一步散播木马。  
有些人认为测试性的代码，位于内网的应用，可以暂时不必关心安全，这是非常不负责任的。你很难预见这段代码什么时候会被作为你的应用的一部分，保险的做法是，为你写下的每一段代码负责，通过本章提出的几项原则，这并不麻烦。

## SQL注入
如果你的应用需要将一个包含用户输入的字符串作为代码(如SQL)执行，那么攻击者可以通过构造巧妙的输入，来将额外的代码**注入**到要执行的字符串中，实现攻击.

下面举一个 SQL 注入的例子, 如果有如下的用户验证的代码：

    $result = $db->query("SELECT * FROM `users` WHERE (`name` = '{$_POST['user']}') and (`passwd` = '{$_POST['passwd']}')");
    if($result->fetch())
        echo "登录成功";

这段代码从 users 表中查询用户名为 $_POST['user'] 且密码为 $_POST['passwd']的记录，如存在这样的记录，就认为登录成功。  

但攻击者可以通过将 user 和 passwd 分别构造为如下的字符串进行攻击：

    admin
    1' OR '1'='1

这样一来，实际执行的SQL变成了这个样子：

    SELECT * FROM `users` WHERE (`name` = 'admin') and (`passwd` = '1' OR '1'='1')

就可以绕过验证，实现以管理员登录.

产生SQL注入的原因即是单引号在 SQL 中具有特殊含义，单引号标识这一段字符串的开始和结束，如果用户输入中含有单引号，那么该字符串就会提前结束，剩下的部分就会作为SQL命令来解析.  
所以，我们有必要对进入 SQL 语句的所有用户输入进行转义，所谓转义就是让代码中具有特殊含义的字符，失去特殊含义，仅仅表示它本身.  

具体来说，我们在SQL中需要转义的字符有：

* 反斜杠 \  
* NUL \0
* 换行 \n
* 回车 \r"
* 单引号 '
* 双引号 "
* Ctrl+Z \x1a

还有一些和字符集相关的字符(在特定的字符集中，会有一些同义字).  

所幸我们不必自己做这些工作，MySQL提供了一个名为 mysql_real_escape_string() 的函数用于将字符串转义，以安全地嵌入SQL.  
而 PDO[1] 里同样有 PDO::quote() 完成同样的工作。

值得注意的是 mysql_real_escape_string() 和 PDO::quote() 的行为有细微的区别，前者只是转义字符串，后者在转义后还会在字符串前后加上单引号。

经过修改的代码会是这样：

    $user = $db->quote($_POST['user']);
    $passwd = $db->quote($_POST['passwd']);
    $result = $db->query("SELECT * FROM `users` WHERE (`name` = {$user}) and (`passwd` = {$passwd})");
    if($result->fetch())
        echo "登录成功";

如果攻击者继续构造类似之前的注入字符串，那么数据库会去查询用户名为 admin 且密码为 `1' OR '1'='1` 的记录，数据库仅仅将这段注入字符串视为它字面的意思，不会作为命令来解析，这样攻击者就不会得逞了.

所以，记住我们的第一个原则：

**如果你需要将一段字符串作为代码来执行，那么要对所有进入字符串的用户输入进行转义**

用户输入不仅仅在于 $_GET 和 $_POST, 包括HTTP Header, Cookie, 文件都是用户输入。

[1]: PHP Data Object, PHP的新式数据库访问接口, 后文会讲解.  
更多SQL注入的例子参见: http://php.net/manual/zh/security.database.sql-injection.php

## Shell注入
如有如下代码：

    shell_exec("rm /data/files/{$_POST['file']}");

这段代码会根据用户输入，删除 /data/file/ 下的指定文件.

既然你已经知道了我们的第一个原则，显然我们要对输入进行转义。 
于是我们找到了 escapeshellarg(), 它会对进入Shell的参数进行转义，添加空格。  

经过修改的代码：

    $arg = escapeshellarg("/data/files/{$_POST['file']}");
    shell_exec("rm {$arg}");

如果攻击者输入 `xxoo;rm -rf /`, 那么实际执行的命令是：

    rm '/data/files/xxoo;rm -rf /'

可以看到攻击者没有成功。
    
但是攻击者可以继续尝试输入 `../../etc/passwd`, 这下你傻眼了, 因为执行了：

    rm '/data/files/../../etc/passwd'

相当于：

    rm '/etc/passwd'

删掉了系统的用户数据库.

所以，我们除了要对用户输入进行转义之外，还需要从逻辑上对输入进行检验.  
在前面SQL注入的例子中，这种检验是不必要的，因为我们知道数据库里面用户的密码不可能恰巧等于攻击者构造的恶意字符串.  
但在这个Shell注入的例子中，我们有这个必要：

    if(strstr($_POST['file'], "/") !== false)
    {
        echo "文件名非法";
    }
    else
    {
        $arg = escapeshellarg("/data/files/{$_POST['file']}");
        shell_exec("rm {$arg}");
    }

这样我们便安全了，如果文件名包含了斜杠，程序会拒绝执行命令.  
在Linux下，文件名可以使用除斜杠外的所有字符，如果需要迁移到Windows下，我们还需要做更多工作。

所以请记住我们的第二个原则：

**不要相信用户的输入，考虑到每一种可能性**

你最好把编写代码的过程，看作“证明这段代码不会出错”的过程。

## 关注官方通告
有时候，漏洞并不在于我们编写的代码，而在于我们所使用的软件本身，如PHP, 或者Linux, MySQL等等。

虽然这些软件的代码审核和发布过程是非常严谨的，但也难免有漏网之鱼。一些严重的漏洞会在一天之间被传播，利用和修复。  
对此我们没有什么对策，毕竟这些软件不是我们自己写的，我们能做的但只有：

* 使用官方的稳定发行版
* 及时关注官方通告，安装补丁

在PHP领域，有很多问题是官方在反复提醒，但仍广泛存在的，我们在此指出几点：

### 关闭 Register Globals
这是 php.ini 中的一个选项(register_globals), 开启后会将所有表单变量($_GET和$_POST)注册为全局变量.  
看下面的例子：

    <?php
    // 当用户合法的时候，赋值 $authorized = true
    if(isAuth())
        $authorized = true;

    if($authorized)
        include("page.php");

这段代码在通过验证时，将 $authorized 设置为 true. 然后根据 $authorized 的值来决定是否显示页面.

但由于并没有事先把 $authorized 初始化为 false, 当 register_globals 打开时，可能访问 /auth.php?authorized=1 来定义该变量值，绕过身份验证。

该特征属于历史遗留问题，在 PHP4.2 中被默认关闭，在 PHP5.4 中被移除。

### 关闭 Magic Quotes
这个特征同样属于历史遗留问题，已经在 PHP5.4 中移除。

该特征会将所有用户输入进行转义，这看上去不错，我们之前提到过要对用户输入进行转义。  
但是 PHP 并不知道哪些输入会进入 SQL , 哪些输入会进入 Shell, 哪些输入会被显示为 HTML, 所以很多时候这种转义会引起混乱。

## 最小权限原则

## Eval
## 中间人攻击


## XSS
## CSRF
## 隐藏信息
## CC