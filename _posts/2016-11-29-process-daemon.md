---
layout: post
title:  "理解守护进程(Daemon)"
permalink: /article/process-daemon
categories: programs
author: wanghong
---
# 一，起因

最近做了一个启动一个进程，在后台持续处理一些任务，并使用god启动和监控这个进程的功能（有点类似sidekiq）。此进程也可以使用一个命令单独启动，所以需要支持将进程设置成为“守护进程(Daemon)”，在使用了ruby
Process类将进程设置为daemon，并使用god启动时，发生了一个奇怪的事情——god会不停的启动这个进程。百思不得解，参考了sidekiq的代码，以及查找了
Daemon 相关的知识，终于找到原因。

# 二，复现

1.简化版本ruby代码，simple.rb

{% highlight ruby %}
Process.daemon(true, true)

loop do
  puts 'Hello'
  sleep 1
end
{% endhighlight %}

2.god启动脚本，simple.god

{% highlight ruby %}
God.watch do |w|
  w.name = "simple"
  w.start = "ruby /home/user/simple.rb"
  w.keepalive
end
{% endhighlight %}

3.启动命令如下

{% highlight ruby %}
god -c simple.god -D
{% endhighlight %}

通过ps命令查看进程，发现很快就生成一堆ruby simple.rb进程。

{% highlight bash %}
peachcat@peachcat:~ $ ps -ef|grep ruby
peachcat  2652  2314  2 21:36 pts/1    00:00:00 ruby /home/peachcat/.rvm/gems/ruby-2.3.0/bin/god -c simple.god -D
peachcat  2672     1  0 21:36 ?        00:00:00 ruby /home/peachcat/ruby/process/simple.rb
peachcat  2944     1  0 21:36 ?        00:00:00 ruby /home/peachcat/ruby/process/simple.rb
peachcat  2962     1  0 21:36 ?        00:00:00 ruby /home/peachcat/ruby/process/simple.rb
{% endhighlight %}

由于原来的进程需要加载rails环境，项目比较大，加载rails时，需要较长时间，我们在设置进程为daemon时为启动rails环境之后，所以还观察到另外一个现象。

此处模拟一下这种情况，将Process.daemon之前增加一个sleep，修改之后代码如下：

{% highlight ruby %}
sleep 6
Process.daemon(true, true)

loop do
  puts 'Hello'
  sleep 1
end
{% endhighlight %}

在sleep执行完成之前，使用ps查看ruby进程，如下图:

{% highlight bash %}
peachcat@peachcat:~ $ ps -ef|grep ruby
peachcat  3021  2314 18 21:39 pts/1    00:00:00 ruby /home/peachcat/.rvm/gems/ruby-2.3.0/bin/god -c simple.god -D
peachcat  3034     1  7 21:39 ?        00:00:00 ruby /home/peachcat/ruby/process/simple.rb
{% endhighlight %}

sleep执行完之后，执行了Process.daemon之后，使用ps查看进程如下图:

{% highlight bash %}
peachcat@peachcat:~ $ ps -ef|grep ruby
peachcat  3021  2314  3 21:39 pts/1    00:00:00 ruby /home/peachcat/.rvm/gems/ruby-2.3.0/bin/god -c simple.god -D
peachcat  3061     1  0 21:39 ?        00:00:00 ruby /home/peachcat/ruby/process/simple.rb
{% endhighlight %}

**可以看到执行了Process.daemon之后，进程的id发生了变化，也正是这个变化导致god认为原来进程已挂了，就会重新启动一个进程。**

# 三，守护进程(daemon)
在继续之前，我们先看看什么是守护进程。 在linux或者unix操作系统中，守护进程（Daemon）是一种运行在后台的特殊进程，它独立于控制终端并且周期性的执行某种任务或等待处理某些发生的事件。 在linux中，每个系统与用户进行交流的界面称为终端，每一个从此终端开始运行的进程都会依附于这个终端，这个终端被称为这些进程的控制终端，当控制终端被关闭的时候，相应的进程都会自动关闭。但是守护进程却能突破这种限制，它脱离于终端并且在后台运行，并且它脱离终端的目的是为了避免进程在运行的过程中的信息在任何终端中显示并且进程也不会被任何终端所产生的终端信息所打断。它从被执行的时候开始运转，直到整个系统关闭才退出。

# 四，Ruby Daemon

参考一段来自于[rack]将进程设置成为daemon的方法，可以大概理解设置成为守护进程的过程。
{% highlight ruby %}
def daemonize_app
  if RUBY_VERSION < "1.9"
    exit if fork
    Process.setsid
    exit if fork
    Dir.chdir "/"
    STDIN.reopen "/dev/null"
    STDOUT.reopen "/dev/null", "a"
    STDERR.reopen "/dev/null", "a"
  else
    Process.daemon
  end
end
{% endhighlight %}

Ruby 1.9.x之后，可以直接使用Process.daemon设置成为守护进程，但其实现代码同上面的实现基本一致。

1.fork方法，衍生一个进程，此方法会返回两次，子进程中返回0，父进程中返回大于0的数字
{% highlight ruby %}
exit if fork
{% endhighlight %}
表示当fork返回值大于0时，即父进程退出。

2.脱离终端与原进程组和进程会话的控制
{% highlight ruby %}
Process.setsid
{% endhighlight %}

这一句话完成了3件事情：

* 该进程变成一个新会话的会话领导
* 该进程变成一个新进程组的组长
* 该进程没有控制终端

# 五，进程组与会话组

**1.  进程组**

每一个进程都属于某个组，每一个组都有唯一的整数id。进程组是一个相关进程的集合，通常是父进程与其子进程。但是你也可以按照需要将进程分组。

linux提供了两个方法用于获取（查看）进程的进程组Id( pgid)，getpgid(pid)和getpgrp()，其中getpgid返回指定进程的进程组id，getpgrp()返回当前进程的进程组id。

{% highlight bash %}
peachcat@peachcat:~ $ ps
  PID TTY          TIME CMD
14712 pts/1    00:00:00 bash
15369 pts/1    00:00:00 ps
peachcat@peachcat:~ $ irb
2.3.0 :001 > Process.getpgrp
 => 15373
2.3.0 :002 > Process.getpgid(0)
 => 15373
2.3.0 :003 > Process.getpgid(14712)
 => 14712
2.3.0 :004 >
{% endhighlight %}

Shell中可以通过ps直接查看pgid:

{% highlight bash %}
peachcat@peachcat:~ $ ps -o pid,pgid,ppid
  PID  PGID  PPID
 2840  2840  2839
 8947  8947  2840
{% endhighlight %}

用**Process.setpgrp**可以设置为当前进程设置为新的进程组组长。一个进程组中，有一个进程为进程组的组长，通常是父进程做为进程组长。进程组有以下特点：
1. 组长进程，其进程组ID==其进程ID
2. 组长进程创建的子进程自动属于此进程组，子进程再创建进程，也同于此进程组
3. 只要进程组中有一个进程存在，进程组就存在，即使组长进程已经终止，但进程组中最后一个进程终止可转移到另一个进程组，由进程组终止。

**2. 会话组**

在shell支持工作控制(job control)的前提下，多个进程组还可以构成一个会话(session)。会话是由其中的进程建立的，该进程叫做会话的领导进程(session leader)。会话领导进程的PID成为识别会话的SID(session ID)。会话中的每个进程组称为一个工作(job)。会话可以有一个进程组成为会话的前台工作(foreground)，而其他的进程组是后台工作(background)。每个会话可以连接一个控制终端(control terminal)。当控制终端有输入输出时，都传递给该会话的前台进程组。由终端产生的信号，比如CTRL+Z， CTRL+\，会传递到前台进程组。会话的意义在于将多个工作囊括在一个终端，并取其中的一个工作作为前台，来直接接收该终端的输入输出以及终端信号。 其他工作在后台运行。 会话主要是针对一个终端建立的。当我们打开多个终端窗口时，实际上就创建了多个终端会话。每个会话都会有自己的前台工作和后台工作。

一个会话中，可能包含多个进程组，会话，进程组的关系可以表示如下图：

![进程组关系图](/assets/img/upload_1481030481546_85915.png)

通常打开终端进程之后，bash进程就是此会话的领导进程（session leader)。我们可以通过以下命令进行简单的验证：

{% highlight bash %}
peachcat@peachcat:~ $ ps -o pid,pgid,ppid,sid,tty,comm | cat
  PID  PGID  PPID   SID TT       COMMAND
 2707  2707  2706  2707 pts/2    bash
 3191  3191  2707  2707 pts/2    ps
 3192  3191  2707  2707 pts/2    cat
{% endhighlight %}

终端又用一种特殊的方法来处理会话组：发送给会话领导的信号被转发到该会话中的所有进程组内，然后再被转发到这些进程组中的所有进程。

回到上面Rack创建守护进程的代码，第一行代码（exit if fork)之后，父进程退出，子进程继续存在，在父进程退出之后，终端会将控制返回给用户，即此进程不再是前台进程。但是子进程仍然拥有父进程继续而来的进程组ID和会话ID，此进程虽然即不是进程组组长，也不是会话领导，但仍与终端有在牵连，如果终端发送信号到此会话组，子进程依然可以收到。因此子进程还没有完全脱离终端。

ruby提供的Process.setsid，会将调用此方法所在的进程设置为新进程组组长和新会话的会话领导。注意，如果在某个已是进程组组长的进程中调用此方法，会调用失败，即此方法只能从非进程组长的子进程中调用。

关于进程组与会话，与信号的关系，进一步可以参见维基百科： [https://en.wikipedia.org/wiki/Process_group](https://en.wikipedia.org/wiki/Process_group)

# 六， 继续分析Ruby Daemon

1.第3行代码执行之前，第一次fork出的子进程在一个新的进程组与新的会话中，虽然此时已没有终端关联，但技术上来说可以给它分配一个。为此，要彻底脱离终端，还需要再进行一次fork。

{% highlight ruby %}
exit if fork
{% endhighlight %}
 这次同样新进程，父进程退出，此时新的子进程，不再是会话领导，也不是进程组长，与原来终端没有任何关系，并且由于此进程不是进程组长，绝对无法分配控制终端，因为终端只能分配给会话领导。

2.更改工作目录，防止进程运行中，工作目录不会意外消失，或者进程占用了可卸载的文件系统。

{% highlight ruby %}
Dir.chdir "/"
{% endhighlight %}

3.重定向标准输入输出流

{% highlight ruby %}
STDIN.reopen "/dev/null"
STDOUT.reopen "/dev/null", "a"
STDERR.reopen "/dev/null", "a"
{% endhighlight %}

将所有的标准流设置到/dev/null，也就是将其忽略。因为守护进程不再依附于某个终端会话，那么这些标准流也就没什么用了。不能简单地将其关闭，因为一些程序还指望着它们随时可用。重定向到/dev/null 确保了它们对于一些程序依然能用，但实际上毫无效果。

# 七，完成god监控功能

上面分析完了Daemon进程的创建原理，也理解了为什么god，会启动多个监控的进程，下面将会说明如何解决这个问题。实际上god官网上已经给出了解决办法： “ If the process you're watching runs as a daemon (as mine does), you'll need to set the pid_file attribute ”，所以办法就是设置god中的pid_file，修改之后的god文件 ，如下所示：

{% highlight ruby %}
God.watch do |w|
  w.pid_file = "/tmp/simple.pid"
  w.name = "simple"
  w.start = "ruby /home/peachcat/ruby/process/simple.rb"
  w.behavior(:clean_pid_file)

  w.transition(:init, { true => :up, false => :start }) do |on|
    on.condition(:process_running) do |c|
      c.interval = 5.seconds
      c.running = true
    end
  end
end
{% endhighlight %}

为god指定一个pid_file的目录，并且设置behavior，在每次启动/重启之前，清除掉pid文件。那么，被监控的进程需要主动将其pid记入同样的pid文件中， simple.rb文件修改之后如下：

{% highlight ruby %}
Process.daemon(true, true)

pidfile = "/tmp/simple.pid"

File.open(pidfile, 'w') do |f|
  f.puts Process.pid
end

loop do
  puts 'Hello'
  sleep 1
end
{% endhighlight %}

记住，写入pid文件 ，一定要在执行完Process.daemon之后。现在运行的效果如下：

{% highlight bash %}
peachcat@peachcat:~/ruby/process $ god -c simple.god -D
I [2016-12-07 10:27:38]  INFO: Loading simple.god
I [2016-12-07 10:27:38]  INFO: Syslog enabled.
I [2016-12-07 10:27:38]  INFO: Using pid file directory: /home/peachcat/.god/pids
I [2016-12-07 10:27:38]  INFO: Started on drbunix:///tmp/god.17165.sock
I [2016-12-07 10:27:38]  INFO: simple move 'unmonitored' to 'init'
I [2016-12-07 10:27:38]  INFO: simple moved 'unmonitored' to 'init'
/home/peachcat/.rvm/gems/ruby-2.3.0/gems/god-0.13.7/lib/god/system/slash_proc_poller.rb:64:in `readable?': Object#timeout is deprecated, use Timeout.timeout instead.
/home/peachcat/.rvm/gems/ruby-2.3.0/gems/god-0.13.7/lib/god/system/slash_proc_poller.rb:64:in `readable?': Object#timeout is deprecated, use Timeout.timeout instead.
I [2016-12-07 10:27:38]  INFO: simple [trigger] process is not running (ProcessRunning)
I [2016-12-07 10:27:38]  INFO: simple move 'init' to 'start'
I [2016-12-07 10:27:38]  INFO: simple before_start: deleted pid file (CleanPidFile)
I [2016-12-07 10:27:38]  INFO: simple start: ruby /home/peachcat/ruby/process/simple.rb
I [2016-12-07 10:27:38]  INFO: simple moved 'init' to 'up'
{% endhighlight %}

使用ps查看进程，只会创建唯一的simple.rb进程了，如下：

{% highlight bash %}
peachcat@peachcat:~ $ ps -ef|grep ruby
peachcat  3432  2482  4 10:27 pts/2    00:00:00 ruby /home/peachcat/.rvm/gems/ruby-2.3.0/bin/god -c simple.god -D
peachcat  3452     1  0 10:27 ?        00:00:00 ruby /home/peachcat/ruby/process/simple.rb
{% endhighlight %}


>参考资料：
>1. God官网： [http://godrb.com/](http://godrb.com/)
>2. [http://www.cnblogs.com/forstudy/archive/2012/04/03/2427683.html](http://www.cnblogs.com/forstudy/archive/2012/04/03/2427683.html)
>3. [http://www.cnblogs.com/vamei/archive/2012/10/07/2713023.html](http://www.cnblogs.com/vamei/archive/2012/10/07/2713023.html)
>4. 《Working with Unix Processes - Jesse Storimer》

[rack]: http://github.com/rack/rack

