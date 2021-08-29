# Spring MVC和Spring Boot有什么区别？ 这样答，面试官直呼666

> 作为初级程序员，这样的问题在面试中，也被问到过，随着越来越了解，发现以前自己答的真水。

## 一般的回答

​		先来说说我以前的粗浅的回答：

- 两者没有什么大关系，除了都是Spring家族里的。
- Spring mvc 是web层的框架，通过Controller提供Http接口服务。
- Spring Boot 是一种快速搭建的脚手架，通过依赖各种Starter，省略了Spring特别多而繁琐的xml配置。

## 分析问题

1. 问spring mvc与 spring boot的区别用意何在？

   因为spring boot虽然兴起时间不短了，但是在2019年时，许多公司还并未去使用。面试官就想通过这个问题知道面试者对技术趋势的了解，以及是否有使用过。

2. 该如何答的和一般面试者不同，让面试官眼前一亮？

   面试其实是个交流过程，向面试官展示你能力水平，答的太浅，食之无味，自然是要多说，自己掌握面试的节奏。那么针对这个问题，可以引申出去说spring mvc产生的来由，有啥其他可代替的技术选型，有啥差别等等。在Spring Boot角度，可以说说常用到的注解，直接让面试官知道你对其有真正的使用。

## 亮眼的回答

- 总：两者作为Spring生态中的组件，产生时间不同，spring mvc很早就诞生，例如之前最主流的企业开发框架ssm，就用到了Spring mvc。Spring Boot作为后起之秀，通过“约定大于配置”来减少许多配置，大大的提高了生产力。

1. spring mvc

- 历史：spring mvc诞生在servlet之后，将其封装，简化其开发难度，让开发人员无需处理整个HttpRequest，也无需处理IO流，只需关心业务处理。同时它也进行了切面封装，可以定义全局异常处理器。基于Servlet开发时，IO返回的即是页面显示的，而spring mvc却可以返回一个渲染后页面。
- 发展：spring mvc发展到现在，已经有@GetMapping, @PostMapping, @RestController等注解，进一步简化了开发。同时REST与前后端分离兴起之后，后端返回前端只需要返回数据即可。
- 选型：像spring mvc这样的框架，我比较了解的还有一个Jersey，这个框架在外国用的比较多，比如Spring Cloud     Eureka就是依赖于Jersey进行Http请求的。

2. spring boot

- ~~历史~~：spring boot的产生主要是提供工程开发便捷。之前开发Spring工程，除了引入依赖，还需要配置许多上下文容器中的配置，例如我们数据库配置，bean的配置，mvc mapping的声明，都是十分麻烦的。在spring boot上这搭建工程仅需几分钟即可，就像它官网说的那样开箱即用，“just run”。
- 使用：在spring boot使用上，构建工程时，使用最多的就是引入对应组件的starter，版本交由spring boot管理，省去了解决依赖冲突的工作量。在开发过程中，结合spring以及spring boot引入的一些注解，例如@Configuration， @Bean， @ConditionalOnClass， @ConditionalOnMissingClass等注解，让我们可以更优雅的注入Bean，以及替换掉默认引入的Bean。

- 总：总之，现在这两个组件我们工程都在使用，搭配起来，开发十分方便。



## 答案分析及知识点

​		一个简单的问题是不是回答了特别多，想必面试官用5分钟听完也是耳目一新。那么我们来分析一下，这个答案为啥能够做到呢：

1. “约定大于配置” 术语的引入，提高层次

2. 提到servlet,IO流，展示你对基础技术的深入了解程度，对技术历史的了解

3. @RestController等注解使用，让面试官知道你不仅会ssm中老式的开发，也会较新的开发模式。

4. 提到Jersey, Eureka，让面试官知道你对技术宽度上的了解，像Jersey这样的选型，我周边许多同事还是根本不知道的，如果面试官没听过Jersey，那么这个问题你已经赢了。

5. 提到spring boot官网对其描述开箱即用，“just run”，让他知道你对一门技术的学习是会通过官网进行的，有不错的英语阅读的能力。

6. @ConditionalOnMissingClass等注解是Spring boot中引入的，让面试官知道你是有实际开发经验的，这一点很重要。

​		一个简单的问题答到这样的程度，对一个有开发经验的初级程序员来说，应该是一个十分亮眼的回答了，也欢迎大佬们进行补充，指出错误！！



## 接下来的问题，掌握局面

​		一个简单问题的回答中，已经引出了许多知识点，面试官很有可能就你的回答继续深入的往下问，那么下面这些问题、知识点，也许你也该准备起来了，掌握局面：

1. servlet容器你们都用什么，tommcat还是jetty? 他们的IO模型有什么区别？

2. 什么是REST风格，你在项目中是如何实践的？

3. Jersey与Spring mvc有什么区别，你们为什么选择spring mvc？

4. spring cloud和spring boot有什么区别？

5. spring cloud有哪些组件？你们项目里用了哪些？

6. spring boot 的starter是怎么实现的，我们怎么自定义一个starter?

   



