# 应用编排和管理：Job和DaemonSet



## （1）Job

## 一、需求来源

### Job背景问题

K8S里最小的调度单元是Pod，如果我们直接通过Pod来运行任务进程，将会产生几个问题：

· 如何保证Pod内进程正确的结束？

· 如何保证进程运行失败后重试？

· 如何管理多个任务，且任务之间有依赖关系？

· 如何并行地运行任务，并管理任务的队列大小？

### Job：管理任务的控制器

Job的功能：

· Kubernetes的Job是一个管理任务的控制器，它可以创建一个或多个pod来指定pod的数量，并可以监控它是否成功地运行或终止；

· 我们可以根据pod的状态来给job设置重置的方式及重试的次数；

· 我们还可以根据依赖关系保证上一个任务运行完成之后再运行下一个任务；

· 可以控制任务的并行度，根据并行度来确保pod运行过程中的并行次数和总体完成大小。

![image-20200511232723972](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200511232723972.png)



## 二、用例解读

### Job语法

![image-20200511232933025](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200511232933025.png)

上图是一个job的yaml文件，可以看到kind类型是job，metadata有个name字段为pi，name就是指定这个job的名称，这个job的作用是计算（输出）圆周率。spec.template里面包含有pod的spec。

这个yaml比之前的yaml多了两点：

· 第一个是重启策略restartPolicy，在Job里面可以设置为Never、OnFailure、Always三种重启策略。分别表示永远不要重新运行、失败的时候再重新运行、不论什么情况下都重新运行；

· 第二个是重试次数backoffLimit，因为job不能无限的去重试嘛，所以我们需要一个参数来控制重试的次数。这个backoffLimit就是保证一个Job到底能重试多少次。

在job里面，我们重点关注的就是重启策略restartPolicy和重试次数限制backoffLimit。

### Job状态

![image-20200511233616503](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200511233616503.png)

Job创建完成后，可以通过```kubectl get jobs```命令来查看当前job的运行状态。得到的结果里有job的名称（NAME），当前完成了多少个(COMPLETIONS), job的持续时间(DURATION)。

COMPLETIONS主要是看任务里面这个pod一共有几个，以及它其中多少个状态是完成。

DURATION主要是我们看job里面的实际业务到底运行了多长时间，这个参数在性能调优的时候很有用。

最后这个AGE的含义是指这个POD从当前时间算起，减去它当时创建的时间。这个时长主要是用来告诉你pod的历史、pod距今创建了多长时间。

### 查看Pod

![image-20200511234046638](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200511234046638.png)

其实Job最后的执行单元还是Pod，所以我们查看一下Pod。

```kubectl get pod```

再查看一下这个pod的yaml。

```kubectl get pods ${pod-name} -o yaml```

可以发现刚才创建的job创建出了一个叫pi的pod，名称后面跟着一个随机字符串，这个任务是计算圆周率。

从这个pod的yaml可以发现它比普通的pod多了一个OwnerReference，这个东西声明了此pod是归哪一个上一层controller来管理的。可以看到这里的OwnerReference是归batch/v1，也就是刚才那个job管理的。还声明了它的controller是job（controller：true 还有 name和uid也能发现这个pod的控制器）。

### 并行运行job

我们有时候有需求：希望job运行的时候最大化的并行，并行出n个pod去快速地执行。然而有时候我们节点数有限制，所以也不能让同时并行的pod数目过多，所以需要这么一个管道的概念，指定我们希望的最大并行数。job控制器可以帮我们做到。

![image-20200511234936350](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200511234936350.png)

重点关注上面的两个参数：completions和parallelism。

· completions参数用来指定本pod队列执行次数。可以把它认为是这个pod指定的可以运行的总次数。比如这里设置为8，也就是这个任务一共会被执行8次。

· parallelism代表并行执行的个数。也就是一个管道或者缓冲器中缓冲队列的大小。这里设置为2，就是说这个job一定要执行8次，每次并行2个pod，所以会执行4个批次。

### 查看并行job运行

![image-20200511235350165](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200511235350165.png)

上图是这个job整体运行完毕后看到的效果，首先看到job的名字是param-1, duration是2分23秒，这是job创建的时间。

下面是真正的pods，一共创建了8个pod，每个pod后面都带有不同的随机字符串，每个pod的状态都是完成的(Completed)。从这些pod的age可以发现，有40s的、73s的、110s的、2m26s的。也就是说：每一组有两个pod的时间是相同的，时间段是40s的是最后一批创建的pod，2m26s的是第一批创建的pod。总有两个pod被同时创建出来，并行完毕、消失、然后再创建、再运行、再完毕。

这里是两个的pod一个批次的原因就是刚才设置了parallelism参数为2，所以是两个pod并行执行，这里就可以了解到这个缓冲器或者说管道队列大小的作用。

### Cronjob语法

有一种job叫做Cronjob，也可以叫定时运行job。Cronjob和job大体上是相似的，唯一的不同是它可以设定一个时间。比如说几点几分执行这个job（很适合晚上做一些清理任务）。还可以设置几分钟执行一次、几小时执行一次等，这就是定时任务。

![image-20200512000644227](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512000644227.png)

定时任务和job相比会有几个不同的字段：

· schedule：这个字段主要是设置时间格式，它的时间格式和Linux的crontime是一样的，所以可以直接根据Linux的crontime书写格式来书写。举例：*/1表示每分钟去执行一下这个job。

可以看到spec里有这么一行

```-date; echo Hello from the Kubernetes cluster```

也就是说这个job做的事情就是打印出大约时间，然后打印出"Hello from the Kubernetes cluster"这一句话；

· startingDeadlineSeconds: 每次运行job的时候，它最长可以等待的时间。有时这个job可能长时间不会启动。所以这时如果超过较长时间的话，CronJob就会停止这个job；

· concurrencyPolicy: 是否允许并行运行。所谓并行运行就是，比如说我每分钟执行一次，但是这个job可能运行的时间特别长（比如两分钟才能运行成功），也就是第二个job到了要运行的时间，上一个job还没有完成。如果concurrencyPolicy设置为true的话，那么不管前面的job是否运行完成，每分钟都会去执行；如果是false，那么就会等上一个job运行完成之后才会运行下一个job运行；

· JobsHistoryLimit：每一个CronJob运行完之后，它都会遗留上一个Job的运行历史、查看时间。当然这不能是无限的，所以需要设置一下历史存留数，一般可以设置默认10个或者100个都可以，这主要取决于集群，根据集群数来确定这个大小。



## 三、操作演示

### Job的编排文件

这个是job的yaml文件：

![image-20200512002530868](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512002530868.png)

### Job的创建及运行验证

执行一下这个job：```kubectl create -f job.yaml```

然后可以查看一下jobs和pods:

```kubectl get jobs```

```kubectl get pods```

![image-20200512002635976](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512002635976.png)

可以发现有一个叫pi的job产生了，然后还产生了一个叫pi-mz58c的pod，- 后面的是随机数。

logs一下这个pod，可以看见这里打印出了圆周率。这里由于我重启了一下job（因为上一个job达到了backoffLimit），所以后面的随机数和刚才不一样了。

```kubectl logs pi-tgfrq```

![image-20200512003201136](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512003201136.png)

### 并行job的编排文件

job1.yaml文件如下：

![image-20200512003858168](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512003858168.png)

### 并行Job的创建及运行验证

job跑起来，然后我们查看一下。

![image-20200512004036012](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512004036012.png)

可以发现刚开始创建了两个paral-1的pod，因为刚开始，还处于ContainerCreating阶段，并不是running阶段。

![image-20200512004122581](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512004122581.png)

过个十几秒再看，已经是running阶段了。

过一段时间再看，pods更多了。而且可以明显看出来它们是分批执行的：

![image-20200512004232335](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512004232335.png)

8个pods，两个pods一批并行运行。这正是我们想要的。

### Cronjob的编排文件

cron.yaml :

![image-20200512004725557](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512004725557.png)

### Cronjob的创建及运行验证

执行一下看看：

![image-20200512004835949](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512004835949.png)

我们看看这个定时任务是不是每分钟产生一个：

```kubectl get jobs```

![image-20200512005040720](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512005040720.png)

可以发现确实每分钟都会产生一个新的hello job。

如果不去干扰它的话，它以后大概会每一分钟都会创建出一个新的job，除非我们什么时候指定它不可以再运行的时候它才会停止创建。我把restartPolicy改为never之后再删掉hello这个cronjob，就会发现不再创建新的job了：

![image-20200512005429523](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512005429523.png)

CronJob其实主要是用来做一些清理任务或者说执行一些定时任务。



## 四、架构设计

### Job管理模式

![image-20200512005532070](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512005532070.png)

Job的架构设计是这样的。Job Controller主要去创建job相应的pod，然后job controller会去跟踪job的状态，及时地根据我们提交的一些配置重试或者继续创建。Job Controller还会自动添加label来跟踪对应的pod，并根据配置并行或者串行创建pod。

### Job控制器

![image-20200512010128487](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512010128487.png)

上图是一个job控制器的主要流程。job的controller会watch API Server，我们每次提交一个job的yaml都会经过API Server传到etcd里面去，然后Job Controller会注册几个Handler，每当有添加、更新、删除等操作的时候，它会通过一个内存级的消息队列，发到controller里面。

通过job controller检查当前是否有运行的pod，如果没有的话，通过scale up把这个pod创建出来；如果有的话，或者如果大于这个数（我猜是指期望的数？），就对它进行scale down，如果这时pod发生了变化，需要及时update它的状态。

同时要去检查它是否是并行的job或者是串行的job，根据设置的配置并行度、串行度，及时地把pod的数量给创建出来。最后，它会把job的整个状态更新到API Server里面去，这样我们就能看到最终呈现出来的效果了。





## （2）DaemonSet

## 一、需求来源

### DeamonSet背景问题

DaemonSet是第二个控制器。如果没有它，会遇到几个问题：

假设我们希望集群内的每个节点都运行同样一个pod：

· 如果新节点加入集群的时候，想要立刻感知到它，然后去部署一个pod，帮助我们初始化一些东西，这个需求怎么做？

· 如果有节点退出的时候，希望对应的pod会被删除掉，应该怎么操作？

· 如果pod状态异常的时候，我们需要及时地监控这个节点异常，然后做一些监控或者汇报的一些操作，那么这个需求运用什么控制器来做？

![image-20200512011455009](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512011455009.png)

### DaemonSet：守护进程控制器

![image-20200512011705289](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512011705289.png)

DaemonSet也是Kubernetes提供的一个default controller。它实际上是一个守护进程的控制器。可以做以下几个事情：

· 保证集群内的每一个节点都运行一组相同的pod；

· 同时还能根据节点的状态保证新加入到节点自动创建对应的pod；

· 在移除节点的时候，能删除对应的pod；

· 而且它会跟踪每个pod的状态，当这个pod出现异常、crash掉了，会及时地去恢复这个状态。



## 二、用例解读

### DaemonSet语法

![image-20200512012118025](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512012118025.png)

上图给了个DeamonSet的yaml文件。

kind: DaemonSet 表示这是个DaemonSet。

它还有matchLables，通过matchLabel去管理对应所属的pod，这个pod.label也要和DaemonSet.controller.label想匹配，它才能去根据label.selector去找到对应的管理Pod。下面spec.container里面的东西都是一致的。

DaemonSet最适用的场景如下：

· 集群存储进程：GlusterFS或者ceph之类的，需要每台节点上都运行一个类似于Agent的东西，DaemonSet就可以做这么一个agent的角色；

· 日志收集进程：比如fluentd或者logstash，这些都是同样的需求，需要每台节点都运行一个agent，这样的话，我们可以很容易搜集到它的状态，把每个节点里面的信息及时地汇报到上面；

· 监控收集器：有时需要每个节点去运行一些监控的事情，也需要每个节点去运行同样的事情。比如说Promethues这些东西，也需要DaemonSet的支持。

### 查看DeamonSet状态

![image-20200512012909018](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512012909018.png)

可以用```kubectl get DaemonSet```或者```kubectl get ds```命令查看DaemonSet。可以看到DaemonSet返回值和deployment特别像，DESIRED、CURRENT之类的都是一个意思。当然说的都是pod啦。

NODE SELECTOR在DaemonSet里面非常有用。有时候我们可能希望只有部分节点去运行这个pod而不是所有节点都运行这个pod，所以有些节点打了label的话，DaemonSet就可以只运行在这些节点上。比如我希望只在master节点运行某些pod，或者只希望worker节点运行某些pod，就可以用到NODE SELECTOR。

### 更新DeamonSet

![image-20200512013351780](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512013351780.png)

DeamonSet和deployment特别像，也有两种更新策略：一个是RollingUpdate, 另一个是OnDelete。

· RollingUpdate就是一个一个的更新。先更新第一个pod,然后移除旧的pod，通过健康检查之后再去建第二个pod，这样对于业务来说可以比较平滑地升级，不会中断；

· OnDelte就是模板更新之后，pod不会有任何变化，需要我们手动控制去删除某一个节点对应的pod之后它才会重建，不删除的话就不会重建。这对于一些需要我们手动控制的特殊需求也有特别好的作用。



## 三、操作演示

### DaemonSet的编排

daemonset.yaml文件：

![image-20200512014809139](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512014809139.png)

### DaemonSet的创建与运行验证

运行一下DaemonSet：```kubectl create -f daemonset.yaml --validate=false```

然后查看一下pod：```kubectl get pods```，发现产生了3个fluentd-elasticsearch pod，为什么产生3个这个pod？因为我这个集群里有三个节点，daemonset在每个节点都运行一个fluentd-elasticsearch，所以产生了三个pod。

![image-20200512015435978](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512015435978.png)

### DaemonSet的更新

更新一下deamonset的镜像版本为fluentd:v1.4:```kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=fluent/fluentd:v1.4```

然后再查看一下更新的状态: ```kubectl rollout status ds/fluentd-elasticsearch```

![image-20200512020120230](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512020120230.png)

可以看到DaemonSet默认是RollingUpdate的。滚动更新每个pod。RollingUpdate可以做到全自动化的更新，无人值守，更新的过程比较平滑，这样有利于我们在现场发布或者做一些其他操作。

上图结尾可以看到，整个的DaemonSet已经RollingUpdare完毕。



## 四、架构设计

### DaemonSet管理模式

DaemonSet是一个controller，它最后真正的业务单元也是pod，DaemonSet其实和Job controller特别相似，也是通过controller去watch API Server的状态，然后及时地添加pod。唯一不同的是，它会监控节点的状态，节点新加入的时候会在节点上创建对应的pod，然后同时根据我们设置的一些约束affinity或者label去选择对应的节点。

![image-20200512020604341](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512020604341.png)

### DaemonSet控制器

DaemonSet其实和Job controller做的差不多：两者都需要watch API Server的状态。

DaemonSet和Job controller唯一的不同点在于：DaemonSet Controller需要去watch节点的状态，但其实节点的状态还是通过API Server传递到etcd上。

当有节点状态发生变化时，它会通过一个内存消息队列发进来，然后DaemonSet controller会去watch这个状态，看一下各个节点上是否都有对应的pod，如果没有的话就去创建。当然它也会去做对比，如果有的话，对比一下版本，然后考虑是否做一个RollingUpdate。如果没有的话就会去重新创建。OnDelete删除pod的时候也会去check它做一遍检查，是否去更新或者去创建对应的pod。

如果全部更新完了之后，它会把整个DaemonSet的状态更新到API Server上，完成最后全部的更新。

![image-20200512021157445](C:\Users\Lin\AppData\Roaming\Typora\typora-user-images\image-20200512021157445.png)