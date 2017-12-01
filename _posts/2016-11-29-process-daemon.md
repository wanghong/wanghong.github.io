---
layout: post
title:  "理解守护进程(Daemon)"
categories: programs
---
# 一，起因

>最近做了一个启动一个进程，在后台持续处理一些任务，并使用god启动和监控这个进程的功能（有点类似sidekiq）。此进程也可以使用一个命令单独启动，所以需要支持将进程设置成为“守护进程(Daemon)”，在使用了ruby
Process类将进程设置为daemon，并使用god启动时，发生了一个奇怪的事情——god会不停的启动这个进程。百思不得解，参考了sidekiq的代码，以及查找了
Daemon 相关的知识，终于找到原因。

# 二，复现
1. 简化版本ruby代码，simple.rb

{% highlight ruby %}
Process.daemon(true, true)

loop do
  puts 'Hello'
  sleep 1
end
{% endhighlight %}

2. god启动脚本，simple.god

{% highlight ruby %}
God.watch do |w|
  w.name = "simple"
  w.start = "ruby /home/user/simple.rb"
  w.keepalive
end
{% endhighlight %}


