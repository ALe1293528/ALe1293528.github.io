## <font style="color:rgba(0, 0, 0, 0.85);">SQL注入原理</font>
<font style="color:rgb(51, 51, 51);">sql注入是指攻击者拼接恶意SQL语句到接受外部参数的动态SQL查询中，程序本身  
</font><font style="color:rgb(51, 51, 51);">未对插入的SQL语句进行过滤，导致SQL语句直接被服务端执行。  
</font><font style="color:rgb(51, 51, 51);">拼接的SQL查询例如，通过在id变量后插入or 1=1这样的条件，来绕过身份验证，获  
</font><font style="color:rgb(51, 51, 51);">得未授权数据的访问权。</font>

```plain
SELECT * FROM user WHERE id = -1 or 1=1
```

<font style="color:rgb(51, 51, 51);">由于or 1=1 满足永真结果，sql语句会执行输出user中的全部内容。</font>

<font style="color:rgb(51, 51, 51);">那么这么危险的漏洞，有没有办法进行阻止呢</font>

**<font style="color:rgb(51, 51, 51);">有的兄弟，有的</font>**

<font style="color:rgb(51, 51, 51);">预编译就能解决大部分的SQL注入问题</font>

## <font style="color:rgb(51, 51, 51);">什么是预编译（Prepared Statement）？</font>
<font style="color:rgb(51, 51, 51);">预编译就是</font>**在执行 SQL 前，把 SQL 语句先告诉数据库服务器，编译好结构，然后再单独传参数进去执行**<font style="color:rgb(51, 51, 51);">！</font>

<font style="color:rgb(51, 51, 51);">它的全名叫：</font>

**Prepared Statement（预处理语句 / 预编译语句）**

## **正常写 SQL 是怎样的？**
我们先看看普通的拼接 SQL 是怎样的：

```plain
username = input("请输入用户名：")
sql = "SELECT * FROM users WHERE username = '" + username + "'"
cursor.execute(sql)
```

这就好像直接把“用户输入”和“SQL语句”拼成一整句话。  
用户只要输入了奇怪的东西，就能控制整个 SQL 的逻辑！Σ(っ °Д °;)っ

## **使用预编译是这样写的：**
```plain
username = input("请输入用户名：")
sql = "SELECT * FROM users WHERE username = ?"
cursor.execute(sql, (username,))
```

**重点就是！  
****SQL 写的时候，用 占位符（?） 或者 命名参数（:name），  
****参数是后面传进去的！不是拼进去的！**

## **预编译的执行流程（详细版！）**
1. **发送 SQL 模板给数据库服务器  
**比如：

```plain
SELECT * FROM users WHERE username = ?
```

这个时候数据库就把这个 SQL 的结构编译好了，生成了“执行计划”

2. **服务器把这个语句存起来  
**存的是“只差参数”的 SQL 模板。
3. **客户端发送参数  
**比如：

```plain
("admin",)
```

4. **数据库执行之前编译好的 SQL  
**把你传进去的参数当成“纯数据”，直接放进语句执行！

## 为什么这样能防止 SQL 注入？
因为参数**永远只是值**，不会被当作 SQL 代码执行！  
哪怕用户输入的是：

```plain
' OR '1'='1
```

数据库也会当成一个完整的字符串 `' OR '1'='1` 来处理，**它不会让它改变 SQL 语句的逻辑结构**

但是预编译真的能完美防御SQL注入吗？笔者在写这篇文章前一直没有思考过这个问题，一是因为知识面浅薄，没有想这么多；二是因为确实没怎么研究过防御漏洞相关的知识，直到翻到了某篇blog[预编译与sql注入 – fushulingのblog](https://fushuling.com/index.php/2023/10/27/%E9%A2%84%E7%BC%96%E8%AF%91%E4%B8%8Esql%E6%B3%A8%E5%85%A5/)[再谈预编译与sql注入 – fushulingのblog](https://fushuling.com/index.php/2025/03/30/%e5%86%8d%e8%b0%88%e9%a2%84%e7%bc%96%e8%af%91%e4%b8%8esql%e6%b3%a8%e5%85%a5/)

假设就用上面的例子，例子中 where语句中的内容是被参数化的。这就是说，预编译仅仅只能防御住可参数化位置的sql注入。那么，对于不可参数化的位置，预编译将没有任何办法。

那么不可参数化的位置都有哪些？

```plain
表名、列名
order by、group by
limit
join
等
```

我们以order by举例，现在有一个sql语句如下（以下为伪代码）

`SELECT * FROM users ORDER BY {user_input};`

    其中user_input是传递过来的参数，例如 id

`SELECT * FROM users ORDER BY id;`

    这个语句是正确的，但是如果user_input输入 id;drop table users --

`SELECT * FROM users ORDER BY id;drop table users --`

这样就被成功注入了，而这种位置是不可被参数化的，所以是无法通过预编译防御的。

[SQL预编译中order by后为什么不能参数化原因 - 诸子流 - 博客园](https://www.cnblogs.com/lsdb/p/12084038.html)

这篇文章中提到

![](https://cdn.nlark.com/yuque/0/2025/png/51527989/1744971033212-2d7d9fb2-c44e-43e7-b671-3c592b56a708.png)

大概就是说order by的后面是字段，字段不能用引号，但是预编译又只有用引号的setString()这一种方法，所以导致一切<font style="color:rgb(75, 75, 75);">是字符串但又不能加引号的位置都不能参数化</font>

<font style="color:rgb(75, 75, 75);">原文以java为例进行说明，但是php中又是怎样呢</font>

### 模拟预编译
网上一般讲的预编译是这么写的：

```plain
<?php
$username = $_POST['username'];
$db = new PDO("mysql:host=localhost;dbname=test", "root", "root123");
$stmt = $db->prepare("SELECT password FROM test where username= :username");
$stmt->bindParam(':username', $username);
$stmt->execute();
$result = $stmt->fetchAll(PDO::FETCH_ASSOC);
var_dump($result);
$db = null;
?>
```

这里如果post传参`username=root`，就可以正常查到值，但是传`'root'`就查不到，通过查看日志可以发现在sql执行的过程中其实根本没有参数绑定、预编译的过程，本质上只是对符号做了过滤

这里参考文献中的作者将其称为虚假的预编译

> <font style="color:rgb(82, 95, 127);">为什么开发者要做一个虚假的预编译呢，那是因为一个参数——PDO::ATTR_EMULATE_PREPARES，这个选项用来配置PDO是否使用模拟预编译，默认是true，因此默认情况下PDO采用的是模拟预编译模式，设置成false以后，才会使用真正的预编译。开启这个选项主要是用来兼容部分不支持预编译的数据库(如sqllite与低版本MySQL)，对于模拟预编译，会由客户端程序内部参数绑定这一过程(而不是数据库)，内部prepare之后再将拼接的sql语句发给数据库执行。</font>
>

### <font style="color:rgb(50, 50, 93);">真正的预编译</font>
<font style="color:rgb(82, 95, 127);">我们在原先的代码上把ATTR_EMULATE_PREPARES设为false取消模拟预编译</font>

```plain
<?php
$username = $_POST['username'];

$db = new PDO("mysql:host=localhost;dbname=test", "root", "root123");
$db -> setAttribute(PDO::ATTR_EMULATE_PREPARES, false);

$stmt = $db->prepare("SELECT password FROM test where username= :username");

$stmt->bindParam(':username', $username);

$stmt->execute();

$result = $stmt->fetchAll(PDO::FETCH_ASSOC);

var_dump($result);

$db = null;

?>
```

我们post一个username=root

这时数据库中执行的顺序变成了：先连接，然后准备语句，用问号?占位，接着用输入替换问号?执行语句，专业点的说法叫做：

1. 建立连接；
2. 构建语法树；
3. 执行

这也是为什么我们之前说的，预编译的作用是让整个语句的功能已经提前定死，消除了sql语句的歧义。当我们输入username= ‘root’同样会没有任何输出

## 模拟预编译的注入点
### 宽字节注入
```plain
2023-10-22T13:12:13.619960Z	    9 Query	SELECT password FROM test where username= '\'root\''
```

从模拟预编译的日志，我们可以发现这里仅仅是用到\的转义，所以我们是否可以进行宽字节注入呢

 答案当然是可以的吗，但是我没复现

### <font style="color:rgb(50, 50, 93);">没有参数绑定</font>
**没有参数绑定的预编译等于没有预编译**，无论是真编译还是模拟预编译，没有参数绑定等于没编译，并且由于pdo默认支持堆叠注入，我们可以通过堆叠注入先插入值然后查询插入的值获取输出结果。

这两个的复现具体可以看下面这个文章：

[https://fushuling.com/index.php/2023/10/27/%E9%A2%84%E7%BC%96%E8%AF%91%E4%B8%8Esql%E6%B3%A8%E5%85%A5/](https://fushuling.com/index.php/2023/10/27/%E9%A2%84%E7%BC%96%E8%AF%91%E4%B8%8Esql%E6%B3%A8%E5%85%A5/)

--------------------------------------------------------------------------------------

对于order by、ground by这种无法进行预编译的场景我们该怎么防御呢，比如Mybaits必须使用${}order by参数，可通过白名单思路对传入的参数进行判断，或者使用间接对象引用，前端传递引用数字等，用于与后端排序参数做数组映射，避免前端直接传入order by参数造成sql注入。

比如我们想执行select xx order by name，那么前端就不要传入name这个值，而是数字比如1，然后在后端将1与真正想查询的参数name进行对应，然后再执行sql语句。比如映射表为1->name，2->age，3->gender，想要查询order by name、age、gender的结果前端只用传入1、2、3即可，通过防止直接执行用户传入的值来从根本上防止sql注入的产生。

ps：order by后面以及group by 后面的注入，有报错回显的直接报错注入就行了，这个简单，没有报错的话我们可以通过构造布尔条件进行注入：随rand()中值真假的不同，排序出来的结果也是不同的，因此可以通过这个特征进行布尔注入，比如输入rand(ascii(mid((select database()),1,1))>96)，如果成立和不成立输出结果显然是不同的，如果我们成功注入，输出应该是root dingzhen admin的顺序

<font style="color:rgb(82, 95, 127);"></font>

