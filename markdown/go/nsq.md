# NSQ简明教程

周末试了一下NSQ，发现挺好用的，简单方便。NSQ是一个消息队列，比如异步任务时，我们需要一个broker，或者是将日志统一 处理时，我们需要一个日志流中间件。NSQ就可以用来干这个。

## 概念

NSQ由三个基本组件组成：

- nsqadmin：管理工具
- nsqd：这是真正的队列所在的进程，如果想要使用nsqlookupd的话，需要在启动的时候传入参数
- nsqlookupd：通过nsqlookupd可以注册和访问多个nsqd

了解了上述概念之后，再来继续普及一下，nsqd可以在多个机器上部署，consumer通过nsqlookupd来连接到多个nsqd进行消费， nsqadmin连上nsqlookupd进行管理。

## 安装

因为Debian的仓库里没有nsq，所有安装方式是从Github下载Release，然后自己配置启动。我用supervisor来管理他们：

```conf
[group:nsq]
programs=nsqd,nsqlookupd,nsqadmin

[program:nsqd]
command=/home/devhg/nsq/nsq-1.2.1.linux-amd64.go1.16.6/bin/nsqd --lookupd-tcp-address=127.0.0.1:4160 -broadcast-address=172.16.8.2
directory=/home/devhg/nsq/nsq-1.2.1.linux-amd64.go1.16.6/bin/
autostart=true
autorestart=true
user=root
environment=

[program:nsqlookupd]
command=/home/devhg/nsq/nsq-1.2.1.linux-amd64.go1.16.6/bin/nsqlookupd -broadcast-address=172.16.8.2
directory=/home/devhg/nsq/nsq-1.2.1.linux-amd64.go1.16.6/bin/
autostart=true
autorestart=true
user=root
environment=

[program:nsqadmin]
command=/home/devhg/nsq/nsq-1.2.1.linux-amd64.go1.16.6/bin/nsqadmin --lookupd-http-address=127.0.0.1:4161
directory=/home/devhg/nsq/nsq-1.2.1.linux-amd64.go1.16.6/bin/
autostart=true
autorestart=true
user=root
environment=
```

这样就可以启动他们：

```bash
$ sudo supervisorctl start nsq:*
```

解压之后还会发现目录下还有以下工具：

```bash
$ ls nsq_*
nsq_stat  nsq_tail  nsq_to_file  nsq_to_http  nsq_to_nsq
```

我们可以通过这些工具来进行调试、操作等。例如如果要备份，就可以用 `nsq_to_file` 把所有的消息备份下来。如果想要 tail然后进行grep，就可以用 `nsq_tail` 等。

## Go生产者和消费者

这是消费者：

```go
package main

import (
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/nsqio/go-nsq"
)

func processMessage(m []byte) error {
	fmt.Printf("%s\n", m)
	return nil
}

type myMessageHandler struct{}

// HandleMessage implements the Handler interface.
func (h *myMessageHandler) HandleMessage(m *nsq.Message) error {
	if len(m.Body) == 0 {
		// Returning nil will automatically send a FIN command to NSQ to mark the message as processed.
		// In this case, a message with an empty body is simply ignored/discarded.
		return nil
	}

	// do whatever actual message processing is desired
	err := processMessage(m.Body)

	// Returning a non-nil error will automatically send a REQ command to NSQ to re-queue the message.
	return err
}

func main() {
	// Instantiate a consumer that will subscribe to the provided channel.
	config := nsq.NewConfig()
	consumer, err := nsq.NewConsumer("topic", "channel", config)
	if err != nil {
		log.Fatal(err)
	}

	// Set the Handler for messages received by this Consumer. Can be called multiple times.
	// See also AddConcurrentHandlers.
	consumer.AddHandler(&myMessageHandler{})

	// Use nsqlookupd to discover nsqd instances.
	// See also ConnectToNSQD, ConnectToNSQDs, ConnectToNSQLookupds.
	err = consumer.ConnectToNSQLookupd("192.168.250.2:4161")
	if err != nil {
		log.Fatal(err)
	}

	// wait for signal to exit
	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
	<-sigChan

	// Gracefully stop the consumer.
	consumer.Stop()
}
```

这是生产者：

```go
package main

import (
	"fmt"
	"log"
	"time"

	"github.com/nsqio/go-nsq"
)

func main() {
	// Instantiate a producer.
	config := nsq.NewConfig()
	producer, err := nsq.NewProducer("192.168.250.2:4150", config)
	if err != nil {
		log.Fatal(err)
	}

	topicName := "topic"

	// Synchronously publish a single message to the specified topic.
	// Messages can also be sent asynchronously and/or in batches.
	for i := 0; i < 99999; i++ {
		messageBody := []byte(fmt.Sprintf("hello %d", i))
		err = producer.Publish(topicName, messageBody)
		if err != nil {
			log.Fatal(err)
		}

		time.Sleep(time.Second * time.Duration(1))
	}

	// Gracefully stop the producer when appropriate (e.g. before shutting down the service)
	producer.Stop()
}
```

这是通过代码来使用NSQ的示例。非常简单，至于持久化，在重启nsqd的时候，nsqd会自动将内存中的信息持久化到磁盘里，所以 通常情况下来说，nsq是不会丢失消息的，但是在一些特殊情况下，例如机器突然crash，这种情况下是会丢失的。所以如果特别 在乎消息是否丢失，那么nsq可能不是特别适合。但是其它情况下，nsq是非常好用的。

## topic和channel

topic与其它MQ中的概念差不多，相当于一个队列，也叫queue，这是对消息的分类。比如某一类消息，就放在同一个queue里，在 nsq中也就是一个topic下。我们把不同的消息发往不通的topic。

在topic之下，还有一个概念，就是channel。发往topic的一个消息，会被复制到topic下的所有channel，而consumer是订阅在一个 channel下的，并且消息在同一个channel里，只会被一个consumer消费。这段话可能有点绕，我们看看官方的图：

![NSQ](https://f.cloud.github.com/assets/187441/1700696/f1434dc8-6029-11e3-8a66-18ca4ea10aca.gif)

可以看到，消息A和消息B，都被复制到了三个channel（分别是metrics，spam_analysis，arvhive），但是metrics这个channel下 的三个consumer，对于每一个消息，都只会有一个consumer被选中来消费消息。

## 总结

这篇文章简单的介绍了以下nsq的相关知识，了解了nsq中的相关组件，概念，以及消息是如何分发的。