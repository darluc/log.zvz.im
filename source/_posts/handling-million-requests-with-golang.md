title: 百万请求一分钟，Golang 轻松来搞定
date: 2018-02-28 14:57:54
tags:
- golang
---

![](https://ws1.sinaimg.cn/large/7327fe71gy1fouyezw261j218g0rsqt8.jpg)

我在反广告、杀病毒、检木马等行业的不同软件公司里已经工作 15 年以上了，非常了解这类系统软件因每天处理海量数据而导致的复杂性。

目前我作为 [smsjunk.com](https://smsjunk.com/) 的 CEO 和 [KnowBe4](http://knowbe4.com/) 的主架构师，在这两个网络安全领域的公司里工作。
<!-- more -->
有趣的是，在过去的 10 年里，作为软件工程师，我接触到的 web 后端代码大多是用 Ruby on Rails 开发的。请不要误会，我很喜欢 Ruby on Railds 框架，而且我认为它是一套令人称赞的框架，不过时间一长，你就会习惯于使用 ruby 语言的方式思考和设计系统，会忘记利用多线程，并行化，快速执行和小的内存消耗，软件架构本可以如此高效且简单。很多年来，我也是一个 C/C++，Delphi 以及 C# 的使用者，而且我开始认识到使用正确的工具能让事情变得更简单。

> 我对互联网上没完没了的语言框架之间的论战并不感冒。因为我相信解决方案的效能及代码可维护性主要倚仗于你的架构能做到多简单。

## 实际问题

在实现某个遥测分析系统时，我们遇到一个实际问题，要处理来自数百万终端的 POST 请求。其中的 web 请求处理过程会接收到一个 JSON 文档，它包含一个由许多荷载数据组成的集合，我们要把它写到 Amazon S3 存储中，之后我们的 map-reduce 系统就可以对这些数据进行处理。

一般我们会利用如下的组件去创建一个有后台工作层的架构，如：

* Sidekiq
* Resque
* DelayedJob
* Elasticbeanstalk Worker Tier
* RabbitMQ
* 等等

并且建立两个不同的服务集群，一个用作 web 前端接收数据，另一个执行具体的工作，这样我们就能动态调整后台处理工作的能力了。

不过从项目伊始，我们的团队就认为应该用 Go 语言来实现这项工作，因为在讨论过程中我们发现这可能是一个流量巨大的系统。我已经使用 Go 语言快两年了，而且我们已经在工作中用它开发了一些系统，只是还没遇到过负载如此大的系统。

我们从定义一些 web 的 POST 请求载荷数据结构开始，还有一个用于上传到 S3 存储的方法。

```go
type PayloadCollection struct {
	WindowsVersion  string    `json:"version"`
	Token           string    `json:"token"`
	Payloads        []Payload `json:"data"`
}

type Payload struct {
    // [redacted]
}

func (p *Payload) UploadToS3() error {
	// the storageFolder method ensures that there are no name collision in
	// case we get same timestamp in the key name
	storage_path := fmt.Sprintf("%v/%v", p.storageFolder, time.Now().UnixNano())

	bucket := S3Bucket

	b := new(bytes.Buffer)
	encodeErr := json.NewEncoder(b).Encode(payload)
	if encodeErr != nil {
		return encodeErr
	}

	// Everything we post to the S3 bucket should be marked 'private'
	var acl = s3.Private
	var contentType = "application/octet-stream"

	return bucket.PutReader(storage_path, b, int64(b.Len()), contentType, acl, s3.Options{})
}
```

## Go routines 的傻瓜式用法

起初我们实现了一个非常简单的 POST 处理接口，尝试用一个简单的 goroutine 并行工作处理过程：

```go
func payloadHandler(w http.ResponseWriter, r *http.Request) {

	if r.Method != "POST" {
		w.WriteHeader(http.StatusMethodNotAllowed)
		return
	}

	// Read the body into a string for json decoding
	var content = &PayloadCollection{}
	err := json.NewDecoder(io.LimitReader(r.Body, MaxLength)).Decode(&content)
	if err != nil {
		w.Header().Set("Content-Type", "application/json; charset=UTF-8")
		w.WriteHeader(http.StatusBadRequest)
		return
	}
	
	// Go through each payload and queue items individually to be posted to S3
	for _, payload := range content.Payloads {
		go payload.UploadToS3()   // <----- DON'T DO THIS
	}

	w.WriteHeader(http.StatusOK)
}
```

在普通负载的情况下，这段代码对于大多数人已经够用了，不过很快就被证明了不适合大流量的情形。当我们把第一个版本的代码部署到生产环境后，才发现实际情况远远超出我们的预期，系统流量比之前预计的大许多，我们低估了数据负载量。

上面的处理方式从几个方面来看都有问题。我们无法办法控制创建的 go routines 的数量。而且我们每分钟收到一百万次的 POST 请求，代码必然很快就崩溃。

## 再次尝试

我们需要寻找别的出路。从一开始，我们就在讨论怎样保证请求处理时间较短，然后在后台进行工作处理。当然，在 Ruby on Rails 里必须这样做，否则你会阻塞掉所有的 web 处理进程，无论你是否使用了 puma，unicorn，passenger（我们这里就不讨论 JRuby 了）。然后我们可能会使用常见的解决方案，比如 Resque，Sidkiq，SQS，等等。有许多方法可以完成这个任务。

所以第二次迭代采用了缓冲通道（ buffered channel ），我们可以将一些工作先放入队列，再将它们上传至 S3，由于我们能够控制队列的大小，而且有充足的内存可用，所以我们以为将任务缓冲到 channel 队列中就可以了。

```go
var Queue chan Payload

func init() {
    Queue = make(chan Payload, MAX_QUEUE)
}

func payloadHandler(w http.ResponseWriter, r *http.Request) {
    ...
    // Go through each payload and queue items individually to be posted to S3
    for _, payload := range content.Payloads {
        Queue <- payload
    }
    ...
}
```

然后将任务从队列中取出再进行处理，我们使用了类似下面的代码：

```go
func StartProcessor() {
    for {
        select {
        case job := <-Queue:
            job.payload.UploadToS3()  // <-- 仍然不好使！
        }
    }
}
```

老实说，我都不知道当时我们在想些什么。这一定是喝红牛熬夜导致的结果。这个方案没给我们带来任何好处，我们只是将一个有问题的并发过程替换为了一个缓冲队列，它只是将问题推后了而已。我们的同步处理过程每次只将一份载荷数据上传到 S3，由于接受到请求的速率远大于单例程上传到 S3 的能力，我们的缓冲队列很快就满了，导致请求处理过程阻塞，无法将更多的数据送入队列。

我们傻乎乎地忽略了问题，最终开始了系统的死亡倒计时。在部署了这个问题版本之后几分钟里，系统的延迟以固定的速率不断增加。

![](https://ws1.sinaimg.cn/large/7327fe71gy1fovfye6vjyj20mz0f8goj.jpg)

## 更好的解决方案

我们决定使用 Go 通道的一种常用模式构建一个两层的通道系统，一个通道用作任务队列，另一个来控制处理任务时的并发量。

这个办法是想以一种可持续的速率、并发地上传数据至 S3 存储，这样既不会把机器跑挂掉也不会产生 S3 的连接错误。因此我们选择使用了一种 Job/Worker 模式。如果你熟悉 Java，C# 等语言，可以认为这是使用通道以 Go 语言的方式实现了一个工作线程池。

```go
var (
	MaxWorker = os.Getenv("MAX_WORKERS")
	MaxQueue  = os.Getenv("MAX_QUEUE")
)

// Job represents the job to be run
type Job struct {
	Payload Payload
}

// A buffered channel that we can send work requests on.
var JobQueue chan Job

// Worker represents the worker that executes the job
type Worker struct {
	WorkerPool  chan chan Job
	JobChannel  chan Job
	quit    	chan bool
}

func NewWorker(workerPool chan chan Job) Worker {
	return Worker{
		WorkerPool: workerPool,
		JobChannel: make(chan Job),
		quit:       make(chan bool)}
}

// Start method starts the run loop for the worker, listening for a quit channel in
// case we need to stop it
func (w Worker) Start() {
	go func() {
		for {
			// register the current worker into the worker queue.
			w.WorkerPool <- w.JobChannel

			select {
			case job := <-w.JobChannel:
				// we have received a work request.
				if err := job.Payload.UploadToS3(); err != nil {
					log.Errorf("Error uploading to S3: %s", err.Error())
				}

			case <-w.quit:
				// we have received a signal to stop
				return
			}
		}
	}()
}

// Stop signals the worker to stop listening for work requests.
func (w Worker) Stop() {
	go func() {
		w.quit <- true
	}()
}
```

我们修改了 web 请求处理过程，使用数据载荷创建了一个 `Job` 实例，然后将其送入 `JobQueue` 通道中供工作例程使用。

```go
func payloadHandler(w http.ResponseWriter, r *http.Request) {

    if r.Method != "POST" {
		w.WriteHeader(http.StatusMethodNotAllowed)
		return
	}

    // Read the body into a string for json decoding
	var content = &PayloadCollection{}
	err := json.NewDecoder(io.LimitReader(r.Body, MaxLength)).Decode(&content)
    if err != nil {
		w.Header().Set("Content-Type", "application/json; charset=UTF-8")
		w.WriteHeader(http.StatusBadRequest)
		return
	}

    // Go through each payload and queue items individually to be posted to S3
    for _, payload := range content.Payloads {

        // let's create a job with the payload
        work := Job{Payload: payload}

        // Push the work onto the queue.
        JobQueue <- work
    }

    w.WriteHeader(http.StatusOK)
}
```

在 web 服务初始化的过程中，我们创建了一个 `Dispatcher` 实例，调用 `Run()` 方法创建了工作例程池，并且通过监听 `JobQueue` 获取工作任务。

```go
dispatcher := NewDispatcher(MaxWorker) 
dispatcher.Run()
```

下面的代码是任务分派器的具体实现：

```go
type Dispatcher struct {
	// A pool of workers channels that are registered with the dispatcher
	WorkerPool chan chan Job
}

func NewDispatcher(maxWorkers int) *Dispatcher {
	pool := make(chan chan Job, maxWorkers)
	return &Dispatcher{WorkerPool: pool}
}

func (d *Dispatcher) Run() {
    // starting n number of workers
	for i := 0; i < d.maxWorkers; i++ {
		worker := NewWorker(d.pool)
		worker.Start()
	}

	go d.dispatch()
}

func (d *Dispatcher) dispatch() {
	for {
		select {
		case job := <-JobQueue:
			// a job request has been received
			go func(job Job) {
				// try to obtain a worker job channel that is available.
				// this will block until a worker is idle
				jobChannel := <-d.WorkerPool

				// dispatch the job to the worker job channel
				jobChannel <- job
			}(job)
		}
	}
}
```

注意我们提供了一个最大数量的参数，用于控制工作池中初始的例程数量。因为这个项目使用了 Amazon Elasticbeanstalk 以及 docker 中的 Go 环境，所以我们努力遵循 [12-factor](http://12factor.net/) 的方法，从环境变量中读取配置值，便于在生产环境中进行系统配置。通过这种方式，我们可以控制工作例程的数量和工作队列的长度，无需对集群进行重新部署，我们就能快速调整参数值。

```go
var ( 
  MaxWorker = os.Getenv("MAX_WORKERS") 
  MaxQueue  = os.Getenv("MAX_QUEUE") 
)
```

在部署这份代码后，我们发现系统的延迟立刻大幅下降，而我们处理请求的能力得到了巨大的提升。

![](https://ws1.sinaimg.cn/large/7327fe71gy1fovyepplmqj20mv0eo0v1.jpg)

在我们的 Elastic Load Balancers 全部预热完成几分钟后，可以看到我们的 ElasticBeanstalk 应用每分钟可以处理近一百万的请求，常常会在流量早高峰的时候突破每分钟一百万。

我们刚把新代码部署上去，服务器数量就从 100 台服务器大幅下降到大约 20 台服务器。

![](https://ws1.sinaimg.cn/large/7327fe71ly1fovypyezcmj20my0d775l.jpg)

在我们调整集群配置和自动缩放配置后，我们能将服务器的使用数量降低到四个 EC2 c4.Large 实例，再将 Elastic Auto-Scaling 设置为 CPU 使用率持续五分钟超 90% 的时候，增加一个实例。

![](https://ws1.sinaimg.cn/large/7327fe71gy1fovz7tuy0hj213c0k6jv1.jpg)

## 结论

在我的认知中，「简单化」才是常胜秘诀。我们本可能设计一个更复杂的系统，拥有许多队列和后台工作例程，部署也更复杂。但是我们最终利用了 Elasticbeanstalk 的自动缩放能力和 Go 语言为我们带来的高效简单的并发解决方案。

并不是每天都能发生这样的事情：一个只有四台机器集群处理着每分钟一百万的 POST 请求，把数据写入 Amazon S3 存储中，而且这些机器可能比我现在的 MacBook Pro 性能还差。

每件工作总会有更合适的工具。当你的 Ruby on Rails 系统需要强大的请求处理能力时，不妨尝试一下 ruby 生态圈外那些更加简单有效的解决方案。



翻译自：[Handling 1 Million Requests per Minute with Golang](https://medium.com/smsjunk/handling-1-million-requests-per-minute-with-golang-f70ac505fcaa)