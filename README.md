
升讯威在线客服与营销系统是基于 .net core / WPF 开发的一款在线客服软件，宗旨是： **开放、开源、共享。努力打造 .net 社区的一款优秀开源产品。**


前段时间我发表了一系列文章，开始介绍基于 .net core 的升讯威在线客服系统开发过程。
随时 Linux \+ MySQL 的广泛应用， **必须让系统同时支持 SQL Server 和 MySQL。**


## 并行查询技术


并行 LINQ (PLINQ) 是语言集成查询 (LINQ) 模式的并行实现。 PLINQ 将整套 LINQ 标准查询运算符实现为 System.Linq 命名空间的扩展方法，并提供适用于并行操作的其他运算符。 PLINQ 将 LINQ 语法的简洁和可靠性与并行编程的强大功能结合在一起。


### 什么是并行查询


一个 PLINQ 查询的许多方面都类似于非并行的 LINQ to Objects 查询。 与顺序 LINQ 查询一样，PLINQ 查询对任何内存中 IEnumerable 或 IEnumerable 数据源执行操作，并且推迟了执行，即在枚举查询前不会开始执行。 主要区别在于，PLINQ 会尝试充分利用系统上的所有处理器。 方法是将数据源分区成片段，然后在多个处理器上针对单独工作线程上的每个片段执行并行查询。 在许多情况下，并行执行意味着查询运行速度显著提高。


通过并行执行，通常只需向数据源添加 AsParallel 查询操作，PLINQ 即可显著提升性能（与某些类型查询的旧代码相比）。 但是，并行可能会引入其自身的复杂性，因此并非所有的查询操作的运行速度在 PLINQ 中都更快。 事实上，并行实际上会降低某些查询的速度。 因此，应了解排序等问题将如何对并行查询产生影响。


### PLINQ 查询性能的影响因素


下面各部分列出了并行查询性能的一些最重要的影响因素。 这些都是一般性说明，本身并不足以用于在所有情况下预测查询性能。


1. 整体工作的计算成本。
为了实现加速，PLINQ 查询必须有足够多的适合并行操作来抵消开销。 工作可以表示为每个委托的计算成本乘以源集合中的元素数量。 假设操作可以并行执行，它的计算成本越高，加速的机会就越大。 例如，如果函数的执行时间为 1 毫秒，那么超过 1000 个元素的顺序查询需要 1 秒的时间才能执行此操作，而在四核计算机上，并行查询可能只需要 250 毫秒就能完成。 这就产生 750 毫秒的加速。 如果函数执行每个元素需要 1 秒，就会产生 750 秒的加速。 如果委托成本很高，PLINQ 可能会让速度显著提升，前提是源集合中只有几项。 相反，包含最简单的委托的小型源集合通常不适合执行 PLINQ。
在下面的示例中，queryA 可能很适合执行 PLINQ，前提是它的 Select 函数涉及很多工作。 queryB 可能不适合执行 PLINQ，因为 Select 语句中没有足够多的工作，并行开销会抵消大部分或全部加速。



```
var queryA = from num in numberList.AsParallel()  
             select ExpensiveFunction(num); //good for PLINQ  

var queryB = from num in numberList.AsParallel()  
             where num % 2 > 0  
             select num; //not as good for PLINQ  

```

2. 系统上的逻辑内核数量（并行度）。
这一点是上一部分的必然结果，在具有更多内核的计算机上，适合并行查询运行得更快，这是因为可以在更多并发线程之间划分工作。 加速总量取决于查询整体工作的并行度百分比。 不过，不要认为所有查询在八核计算机上的运行速度都是在四核计算机上的两倍。 优化查询以实现最佳性能时，请务必在具有不同数量内核的计算机上度量实际结果。 这一点与第 1 点相关：需要更大的数据集，才能利用更多的计算资源。
3. 操作的数量和种类。
如果有必要维护源序列中的元素顺序，PLINQ 提供 AsOrdered 运算符。 虽然排序有相关成本，但此成本通常还算低。 GroupBy 和 Join 操作同样也会产生开销。 如果允许按任意顺序处理源集合中的元素，并在准备就绪后立即将它们传递给下一个运算符，PLINQ 的性能最佳。
4. 查询执行形式。
若要通过调用 ToArray 或 ToList 存储查询结果，所有并行线程的结果都必须合并到一个数据结构中。 这就涉及不可避免的计算成本。 同样，如果使用 foreach（Visual Basic 中的 For Each）循环来循环访问结果，工作线程的结果必须串行化到枚举器线程。 不过，如果只想根据每个线程的结果执行某操作，可以使用 ForAll 方法对多个线程执行此操作。
5. 合并选项类型。
PLINQ 可以配置为缓冲输出并在生成整个结果集后分块区生成或一次性全部生成，也可以配置为在各个结果生成时流式传输它们。 前一个导致总体执行时间减少，后一个导致所生成元素之间的延迟减少。 尽管合并选项不一定会对总体查询性能造成重大影响，但它们可能会影响感知性能，因为它们控制用户在看到结果前必须等待的时间。


### 选择使用模型



```
var source = Enumerable.Range(1, 10000);

// Opt in to PLINQ with AsParallel.
var evenNums = from num in source.AsParallel()
               where num % 2 == 0
               select num;
Console.WriteLine("{0} even numbers out of {1} total",
                  evenNums.Count(), source.Count());
// The example displays the following output:
//       5000 even numbers out of 10000 total

```

AsParallel 扩展方法将后续查询运算符（在此示例中为 where 和 select）绑定到 System.Linq.ParallelEnumerable 实现。


### 执行模式


默认情况下，PLINQ 是保守的。 在运行时，PLINQ 基础结构将分析查询的总体结构。 如果通过并行可能会提高查询速度，PLINQ 则将源序列分区为可以同时运行的任务。 如果并行化查询不安全，PLINQ 则只会按顺序运行查询。 如果 PLINQ 可以在可能会较昂贵的并行算法或成本较低的顺序算法之间进行选择，它会默认选择顺序算法。 可以使用 WithExecutionMode 方法和 System.Linq.ParallelExecutionMode 枚举指示 PLINQ 选择并行算法。 如果你通过测试和测量知道特定查询以并行方式执行得更快时，此做法非常有用。


### 并行度


默认情况下，PLINQ 使用主机计算机上的所有处理器。 可以使用 WithDegreeOfParallelism 方法指示 PLINQ 使用不超过指定数量的处理器。 当你要确保计算机上运行的其他进程收到一定的 CPU 时间量时，此做法将非常有用。 下面的片段将查询限制为最多使用两个处理器。



```
var query = from item in source.AsParallel().WithDegreeOfParallelism(2)
            where Compute(item) > 42
            select item;

```

在查询要执行大量非受计算限制的工作（如文件 I/O）的情况下，最好指定比计算机上的内核数要大的并行度。


### 已排序和未排序的并行查询


在某些查询中，一个查询运算符必须产生保留源序列排序的结果。 为此，PLINQ 提供了 AsOrdered 运算符。 AsOrdered 不同于 AsSequential。 尽管仍并行处理 AsOrdered 序列，但会缓冲和排序它的结果。 由于顺序暂留通常涉及额外的工作，因此处理 AsOrdered 序列可能比处理默认 AsUnordered 序列更慢。 特定的已排序并行操作是否比操作的顺序版本更快取决于许多因素。


下面的代码示例演示了如何选择使用顺序保留。



```
var evenNums =
    from num in numbers.AsParallel().AsOrdered()
    where num % 2 == 0
    select num;

```

### 并行和顺序查询


某些操作要求按顺序提供源数据。 必要时，ParallelEnumerable 查询运算符自动还原为顺序模式。 对于要求顺序执行的用户定义的查询运算符和用户委托，PLINQ 提供了 AsSequential 方法。 使用 AsSequential 时，查询中的所有后续运算符都会顺序执行，直到再次调用 AsParallel。


### 异常


当一个 PLINQ 查询执行时，可能会同时从不同的线程引发多个异常。 此外，处理异常的代码可能与引发异常的代码处于不同的线程上。 PLINQ 使用 AggregateException 类型封装查询抛出的所有异常，并将这些异常封送回调用线程。 在调用线程上，只需要一个 try\-catch 块。 不过，可以循环访问在 AggregateException 中封装的所有异常，并捕获任何可以安全恢复的异常。 在极少数情况下，可能会抛出一些未在 AggregateException 中包装、ThreadAbortException 也没有进行包装的异常。


如果允许异常向上冒泡回到联接线程，则查询也许可以在引发异常后继续处理一些项。


### 自定义分区程序


在某些情况下，可以通过编写利用源数据的某些特征的自定义分区程序来提高查询性能。 在查询中，自定义分区程序本身是被查询的可枚举对象。


\-\-


# 验证与测试


下文我将介绍如何快速部署升讯威在线客服系统，来测试验证并行查询技术的效果。


* CentOS 安装配置 MySQL 数据库，创建数据库，执行脚本创建表结构。
* 安装 Nginx，反向代理到客服系统服务端，并设置开机自启动
* 安装 .net core ，部署客服系统并开机自启动


我详细列出了需要执行的命令的全过程，跟随本文可以在 30 分钟内完成部署。


### 完整私有化包下载地址



> 💾 [https://kf.shengxunwei.com/freesite.zip](https://github.com)


### 当前版本信息


发布日期： 2024\-9\-18
数据库版本： 20240413b
服务器版本： 1\.15\.5\.0
客服程序版本： 1\.15\.19\.0
更新程序版本： 1\.2\.1\.0
资源站点版本： 1\.7\.20\.0
Web管理后台版本： 2\.3\.1


![](https://docs-api.shengxunwei.com/StaticFiles/Upload/15ea0fe9-0392-4acc-bc5a-12735d16d537.png)


![](https://docs-api.shengxunwei.com/StaticFiles/Upload/1d43bed9-b5a8-4941-a2c6-56ef4e3152cf.png)


# 准备操作系统


* 本文以 CentOS 7\.9 为例进行说明，其它版本的 Linux 安装配置过程大同小异。


## 开放防火墙端口


客服系统默认使用 9527 端口进行通信，如果开启了防火墙，请在防火墙中开放此端口。


也可以更改为其它可用端口号，在后续配置客服系统服务端程序时要做对应的修改。



> 请确保您所使用的主机服务商提供的防火墙服务中，也开放了对应端口。如阿里云服务器需要在安全组规则中配置。


# 安装 MySQL 数据库引擎


1. 下载
`wget http://dev.mysql.com/get/mysql80-community-release-el7-10.noarch.rpm`



> 如果提示 command not found，则先执行 `yum -y install wget` 安装
> 
> 
> 注意：此下载地址适用于 CentOS 7 ，如果您使用的是 CentOS 8，请将 rpm 包名更换为 `mysql80-community-release-el8-8.noarch.rpm`
> 不同版本的 MySQL Community 版本下载请参阅：[https://dev.mysql.com/downloads/](https://github.com)


2. 安装
`rpm -ivh mysql80-community-release-el7-10.noarch.rpm`
`yum install mysql-community-server -y`
3. 检查是否安装成功
`rpm -qa | grep mysql`


输出类似：
mysql\-community\-client\-plugins\-8\.0\.34\-1\.el7\.x86\_64
mysql\-community\-libs\-8\.0\.34\-1\.el7\.x86\_64
mysql\-community\-icu\-data\-files\-8\.0\.34\-1\.el7\.x86\_64
mysql\-community\-server\-8\.0\.34\-1\.el7\.x86\_64
mysql80\-community\-release\-el7\-10\.noarch
mysql\-community\-common\-8\.0\.34\-1\.el7\.x86\_64
mysql\-community\-client\-8\.0\.34\-1\.el7\.x86\_64


4. 启动
`sudo systemctl start mysqld`
5. 设置开机自启动
`sudo systemctl enable mysqld`
6. 查看安装时生成的临时密码
`sudo cat /var/log/mysqld.log |grep password`
7. 使用临时密码连接 MySQL
`mysql -uroot -p`
8. 修改 root 密码
`alter user root@localhost identified with mysql_native_password by '你的密码';`



> 密码必须包含大写字母、小写字母、数字、特殊符号


9. 退出 MySQL 连接
`exit`


# 安装 Nginx


1. 安装 Nginx
`sudo yum install -y nginx`



> 如果提示 No package nginx available，则先执行
> 
> 
> * `yum install epel-release -y`



> 安装成功后：
> 可执行文件为：/usr/sbin/nginx
> 默认的网站目录为： /usr/share/nginx/html
> 默认的配置文件为：/etc/nginx/nginx.conf
> 自定义配置文件目录为: /etc/nginx/conf.d/


2. 启动 Nginx
`systemctl start nginx`
3. 启用开机启动 Nginx
`systemctl enable nginx`


# 安装 .Net Core


1. 配置安装源
`sudo rpm -Uvh https://packages.microsoft.com/config/rhel/7/packages-microsoft-prod.rpm`
2. 安装
`sudo dnf install dotnet-sdk-3.1`



> 如果提示 command not found，则先执行
> 
> 
> * `yum -y install dnf`


3. 确认安装成功


在命令行输入 `dotnet` 看到类似如下提示，表示安装成功。


![](https://docs-api.shengxunwei.com/StaticFiles/Upload/568be32d-daf9-4665-81b8-7b82d857a694.jpg)


# 配置服务器主程序


请确认已经完成了对服务器主程序配置文件的配置。
参阅：[使用自动化工具配置服务器端程序](https://github.com)


## 访客聊天测试


登录客服端以后，用浏览器打开你的资源站点域名下的聊天页面，如：



> kf\-resource.shengxunwei.com/WebChat/WebChat.html?sitecode\=freesite


开始聊天。


![](https://docs-api.shengxunwei.com/StaticFiles/Upload/295287db-7946-4f5c-a624-65a2e08a9782.JPG)


![](https://docs-api.shengxunwei.com/StaticFiles/Upload/564f10a0-9b2e-4048-a975-3cb4c7d6d065.JPG)


## 发布


将配置好的客服端程序 Shell 目录，压缩或打包分发给客服使用即可。


## 集成


* [集成到您的网站](https://github.com):[westworld加速](https://tianchuang88.com)
* [集成到您的手机APP](https://github.com)
* [集成到您的公众号等平台](https://github.com)
* [深度集成：传递您的访客数据到客服系统](https://github.com)


