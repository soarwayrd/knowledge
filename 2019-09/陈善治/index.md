1. .Net Core 2.2踩坑 -- 修改了系统的默认函数名，导致code first失败

    以下两段代码均为可正常运行的代码。
    ```csharp
     // 框架代码
     namespace Soarway.Hummer.Core.API
     {
         public class Program
         {
             public static void Main(string[] args)
             {
                 CreateWebHostBuilder(args).Build().Run();
             }
 
             public static IWebHostBuilder CreateWebHostBuilder (string[] args)
             {
             
             }
         }
     }
    ```

    ```csharp
     // 系统代码
     namespace Soarway.Charge.API
     {
         public class Program
         {
             public static void Main(string[] args)
             {
                 CreateWebHostBuilder(args).Build().Run();
             }
     
             public static IWebHostBuilder CreateWebHostBuilder(string[] args)
             {
                 return Soarway.Hummer.Core.API.Program.CreateWebHostBuilder(args).UseStartup<Startup>();
             }
         }
     }

    ```
    以下阐述坑在哪里。
    为了凸显出框架与系统的差别，我们将框架的**CreateWebHostBuilder**更名为了**CreateBaseWebHostBuilder**,业务系统的调用代码同步变为
    **return Soarway.Hummer.Core.API.Program.CreateBaseWebHostBuilder(args).UseStartup<Startup>();**。这一看没有任何问题，编译运行，也确实没有任何问题。但三天后，需要添加向某一张表以Code First的方式添加一个字段时，大问题来了。命令行报错，提示读不到配置文件，通过打日志得知，脚本迁移命令根本没有执行到指定函数里，故更不可能读到配置信息。反复的思考这几天的改动，最后将目光定位到了系统默认提供的函数名上，将**Base**等字眼去除后，Code First命令终于得以正确运行。

2. 订阅发布模式
   在框架的实际使用过程中，遇到了框架模块需要与业务模块在程序内部进行通讯的需求。以**添加部门**这个需求为例：
   框架接口定义了添加部门的api及参数，对于前端框架而言，添加部门的按钮应该始终指向该api及传递规定的参数。但是在开发过程中，使用框架的项目组传递出了“希望在部门添加成功后，将部门的相关的信息插入到系统的业务表里”，当时项目组提出了一个方案，希望框架能够扩展**添加部门**api参数，以便于他们传递一个存储过程名称，在框架添加部门完毕后，再有框架发起调用业务系统的存储过程。当时我拒绝了这种需求，我们从几个方面剖析，为什么拒绝了这个非常简单的需求。
   1、做框架我们要确定好边界在那，不能让业务入侵框架，框架不去管业务，只有定好了边界，框架才不会走样，才有了复用的可能。
   2、“部门添加成功后，在执行一个存储过程”，部门添加成功，数据提交入库了了，再执行一个存储过程，如果存储过程执行失败，已添加的部门是没法回滚的。因为两个动作不在一个会话内完成。那这时候是手动删除已添加的部门，还是代码的异常处理里面加一个补删的操作？这都不是好办法。
   3、如果过程在框架里执行，框架就要负责过程的异常处理，过程的业务变动带来的框架变动，边界不清晰，框架开发人员跟业务开发人员就要频繁的接触，探讨处理的细节。

   以上的问题，总结一下，就是以下几点
   1、边界明晰，明确框架该做什么，不该做什么。不做不是因为难易，而是因为原则。
   2、事务控制，确保数据一致性，要么全对，要么回滚。
   3、职责明确，框架开发人员应确保框架的纯净，尽可能少出现业务层面的代码，对于业务开发人员提出的不合理的需求，得明确拒绝，把框架搞脏了，是对其他框架使用者的不负责。

   解决以上问题的方法，订阅发布模式。
   订阅发布模式，其实在我们的开发过程中随处可见，rxjs里面的subscribe获取接口请求数据；windows gui 应用；JavaScript的onClick事件，app的换肤功能等，都是订阅发布模式的具体体现。这个模式的最大好处就是完全的松耦合，以发布者为中心，发布者不需要知道订阅者的存在，订阅者可以有一个或多个，也可以一个都没有。在订阅发布模式下，以上问题都迎刃而解。发布者将Orm上下文通过参数传递给订阅者，订阅者在做完自己的后续业务后，连同发布者的业务一并通过事务提交，保证了数据的一致性。
   解决了事务控制的问题，同时也解决了职责明确的问题。框架（发布者）负责的是部门添加，通过事件广播，将部门及Orm上下文传给了业务方（订阅者），最后的事务由业务方进行提交，将两个割裂的代码，通过事件的方式又有机的结合在了一起。

3. 包装模式（装饰器模式）
   开发过程中，时长因为需求的变动，我们需要改动代码，添加功能。但改动代码带来的副作用有可能是影响到现已运行稳定的功能。我们希望最好的结果是，就的功能继续保持稳定，需要改动的代码保持可控。包装模式，就是这种场景下最好的选择。其实包装模式很简单，甚至我觉得都不能称之为一种模式，算是一种技巧而已。

   其就好比俄罗斯套娃，一层套一层，每个看起来差不多，却有大小不同，一个套一个又有关联。像极了包装模式。v1.0的方法，运行了很长时间，保持稳定了。这时候开发者提出，系统方法能扩展一个参数。这时候你不应该去改动稳定的v1.0方法，因为直接改动其代价是巨大的。最好的做法是写一个v1.0方法的重载，在重载的基础调用v1.0，并做扩展。该模式不但可用于扩展方法，还可用于处理一些改不动的“祖传代码”，祖传代码，由于年代久远，经历人手众多，处于改不动的状态，倒是他能够跑的动，这时候后你如果要对祖传代码加功能，还是选择在他外面包一层，实现自己的功能比较好。

   以下举个最简单的示例。

   ```csharp
   public void Hello(string name)
   {
       // 你好，张三
       Console.WriteLine("你好," + name);
   }

   public void Hello(string name, string datetime)
   {
       Hello(name);
       Console.WriteLine("上次登陆时间: "+ datetime);
   }
   ```

   而不应该直接在第一个hello函数上去扩展一个参数。其缺点有
   1. 代码的变动不可避免的带来bug的风险，也许hello是一个运行了很久很稳定的代码，你一改，所有系统都跟着你的改动变得战战兢兢，由于是否要冒风险进行升级。
   2. 所有调用Hello的地方，编译都无法将无法通过，自己把向下兼容的通道给堵死了，对下游及其不友好。