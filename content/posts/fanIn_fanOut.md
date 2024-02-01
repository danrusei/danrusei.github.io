---
title: "Fan-Out, Fan-In pipeline in Go and Rust"
date: 2020-05-15
draft: false
categories: ["Rust"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

In this article I'm exploring the process of implementing a Fan-Out/ Fan-In pipeline in Go and Rust. Fan-Out is a term to describe the process of starting multiple workers to handle input from the pipeline, and Fan-In is a term to describe the process of combining multiple results into one channel.  
I'm assuming that you have some familiarity with both Go and Rust languages. I'm new to Rust also, therefore it's fair to say that this is about concurrency in Go & Rust through the eyes of a GO developer. What I am aiming for is to describe the patterns I learned and tried out by implementing this in both languages.

**Important** - This is just a practical guide, I'm not going into the intricacies of the underlying technologies. This is not a deep dive into the language features.  
There are similarities and differences between these two languages but both were created from ground-up to enable developers to build modern and parallel computing applications. Their approach to memory safety and concurrency is different, therefore use the one that fits the need of your application. Many companies are using both to cover different use cases, therefore I believe both tools are valuable assets in the developer toolkit.

## The Intent

The below code is available on [Github](https://github.com/danrusei/dev-state_blog_code/tree/master/fanOut_fanIn) and you can use it to your needs. I'm not covering all the parts of the code therefore checkout the complete code on github. This is a simplified model but can be adapted to any kind of workloads. 
The flow is as follows:

* a number of jobs are defined (in our case the `generator` function create the job requests and send those over channel)
* a number of workers are processing the jobs concurrently (in our case the `fanOut/fanOutFunc` function takes care of it and send the results over channels)
* the results are aggregated and maybe augmented (in our case the `fanIn` function receive the results and present those to the caller )

Besides the `main.go or main.rs` file, where the pipeline is defined, there is a second file named `parse.go or parse.rs`, which is not covered in this article. The `parse` file contains the code that is invoked by the `fanOut` function. Basically it sends a GET request to a site. It Unmarshal/Deserialize the json content into a Struct type. For the sake of this exercise it parses the content and sends back to the caller the details about the largest comment of the requested post `post ID`.

Initially I tried to use the [JSONPlaceholder](https://jsonplaceholder.typicode.com/comments) site for the tests, but it was very slow and decided to create a very simple http server to serve the requests. In case you want to test it out, the http server code is available on [Github](https://github.com/danrusei/dev-state_blog_code/tree/master/fanOut_fanIn/server) as well, and it has everything it needs to be deployed on [GCP App Engine](https://cloud.google.com/appengine/docs/standard/go/building-app). 

## Go pattern - goroutines

Go is a general-purpose programming language with builtin concurrency capabilities. Go's standard library, often is described as coming with "batteries included", that provides building blocks and APIs to create I/O, networking and distributed applications. Go's concurrency model makes it simple to develop data pipelines. It is important to make the distinction between [concurrency and parallelism](https://blog.golang.org/waza-talk).

The `generator` function takes an integer and generates a random number within 0 to 100 range that is sent over the channel, the returned channel is read-only. The channel is closed at the end of the `for` iteration but the function can be canceled anytime if it receives a value over `done` channel. The value type of done is the empty struct because the value doesn't matter: it is the receive event that indicates the send on `streamID` should be abandoned. A goroutine is started to ensure that the process is running concurrently. 

{{< code language="go" isCollapsed="false" >}}
func generator(done <-chan struct{}, iter int) <-chan int {
	streamID := make(chan int)
	r := rand.New(rand.NewSource(time.Now().UnixNano()))
	genID := 0

	go func() {
		defer close(streamID)
		for i := 0; i < iter; i++ {
			genID = r.Intn(100)
			select {
			case <-done:
				return
			case streamID <- genID:
			}
		}
	}()
	return streamID
}
{{< /code >}}

The role of the `fanOutFunc` function is quite simple. The way the function is called makes the difference, and that will be described below, in the main block. It takes an int read-only channel and returns a string read-only channel, that is the result of the `parse()` function, which is defined in the [parse.go file](https://github.com/danrusei/dev-state_blog_code/blob/master/fanOut_fanIn/pattern_Go/parse.go). 
The **`for n := range in`** syntax is one of my favorites, as it shows how simple it is to iterate over the values received from the `in` channel. The iteration terminates and the function returns once the `in` channel is closed or a value is received on the `done`.
The heavy lifting is performed here, as it invokes the `parse()` function. Imagine that the worker can be used to perform a large variety of tasks, from heavy processing to I/O or Network tasks.

{{< code language="go" isCollapsed="false" >}}
func fanOutFunc(done chan struct{}, in <-chan int) <-chan string {
	resultValue := make(chan string)
	go func() {
		defer close(resultValue)
		for n := range in {
			select {
			case <-done:
				log.Println("funOutFunc has been canceled")
				return
			case resultValue <- parse(done, n) + " _ Processed":
			}
		}
	}()
	return resultValue
}
{{< /code >}}

The role of the `fanIn` function is to aggregate all the results sent by the Workers (funOutFunc), augment the result if needed and send it over the `resultValue` channel, that is returned by the function. As we are running many Workers concurrently, each one will send the result over its own channel. This function converts a list of channels to a single channel by starting a goroutine for each inbound channel that copies the values to the single `resultValue` channel. 
Once all the `multiplex` goroutines have been started, `fanIn` creates one more goroutine to close the `resultValue` channel after all sends on that channel are done. Sends on a closed channel panic, so it's important to ensure all sends are done before calling close, therefore `wg.Wait()` is used to wait for all goroutines to conclude. Each goroutine notify the waitgroup when it's done by executing `wg.Done()`.

{{< code language="go" isCollapsed="false" >}}
func fanIn(done <-chan struct{}, cs ...<-chan string) <-chan string {
	var wg sync.WaitGroup
	resultValue := make(chan string)

	// Start an output goroutine for each input channel in cs.  multiplex
	// copies values from c to out until c is closed, then calls wg.Done.
	multiplex := func(c <-chan string) {
		defer wg.Done()
		for text := range c {
			select {
			case <-done:
				log.Println("funIn has been canceled")
				return
			case resultValue <- text:
			}
		}
	}
	wg.Add(len(cs))
	for _, c := range cs {
		go multiplex(c)
	}

	//a goroutine to close out the "resultValue" channel once all goroutines are done
	go func() {
		wg.Wait()
		close(resultValue)
	}()

	return resultValue
}
{{< /code >}}

Now, as the generator, fanOutFunc and fanIn functions were defined above, we have to put those at work.  A slice of channels of the length of `nWorkers` is created. The`fanOut` defines the number of concurrent "fanoutFunc" functions. The pipeline is runing untill all the fanOut channels are drained and the result is printed out to the stdout.

{{< code language="go" isCollapsed="false" >}}
func main() {
	nWorkers := 4
	nJobs := 8

	done := make(chan struct{})
	defer close(done)

	fanOut := make([]<-chan string, nWorkers)
	// fanOut defines the number of concurrent "fanoutFunc" functions (goroutines)
	for i := 0; i < nWorkers; i++ {
		fanOut[i] = fanOutFunc(done, generator(done, nJobs/nWorkers))
	}

	//this pipeline yields the result of each channel of the fanOut slice
	for result := range fanIn(done, fanOut...) {
		fmt.Println(result)
	}

}
{{< /code >}}

### Run the code

{{< code language="bash" isCollapsed="false" >}}
$ /usr/bin/time -f "%MK %E" ./pattern_Go

70 Arjun@natalie.ca 167 _ Processed
58 Juana_Stamm@helmer.com 207 _ Processed
17 Caleb_Herzog@rosamond.net 192 _ Processed
52 Glenna@caesar.org 214 _ Processed
98 Hilma.Kutch@ottilie.info 167 _ Processed
53 Lenora@derrick.biz 200 _ Processed
39 Eugene@mohammed.net 169 _ Processed
96 Maryam.Mann@thelma.info 196 _ Processed

13976K 0:00.32
{{< /code >}}

## Rust pattern - threads 

Rust is a multi-paradigm programming language that runs blazingly fast, prevents segfaults, guarantees thread safety and it can power performance-critical services. Providing zero cost abstractions is a goal of Rust, therefore it has no garbage collector. The memory-safety and thread-safety are guaranteed using Rust's ownership and borrowing model. 

Unlike Go, Rust has a minimal standard library, it offers a set of battle-tested shared abstractions for the broader [Rust ecosystem](https://crates.io/). Rust's runtime system and green-threading model has been entirely removed some time ago, which is inline with zero cost abstractions. 
Rust does not have the notion of builtin channels like Go but it does offer [mpsc](https://doc.rust-lang.org/std/sync/mpsc/) (multiple producer, single consumer) in the standard library, that can be shared across threads. The channels behave differently than the Go channels, [here is](https://gsquire.github.io/static/post/a-rusty-go-at-channels/) a good blog post that describes the channel's differences.
We are using threads to create the distributed model. [Threadpool](https://crates.io/crates/threadpool) is a convenient way for running a number of jobs on a fixed set of worker threads.

{{< code language="rust" isCollapsed="false" >}}
use std::io;
use std::error::Error;
use std::thread;
use std::sync::mpsc::{channel, Receiver};
use threadpool::ThreadPool;
use rand::Rng;
{{< /code >}}

The generator function takes an `u32` and returns the channel receiver. The rand crate is using `thread_rng()` to create a `ThreadRng` struct. It does not implement the Send or Sync traits so it can't be sent over the channel.
As a workaround we construct a vector `Vec<u32>`, which is a contiguous growable array type, from the iterator with the random values.

{{< code language="rust" isCollapsed="false" >}}
fn generator(n_jobs: u32) -> io::Result<Receiver<u32>> {
    let (tx, rx) = channel();
    let mut rng = rand::thread_rng();
    let nums: Vec<u32> = (0..n_jobs).map(|_| rng.gen_range(1,100)).collect();
    thread::spawn(move || {
        for num in nums{
            tx.send(num).expect("Could not send the generated number over gen_sender channel")
        }
    });
    Ok(rx)
}
{{< /code >}}

The `threadpool` is constructed in the main and added as an argument along with the `n_jobs` and the generator `Receiver` to the fan_out function. In the body of the function we iterate over the number of jobs and instruct threadpool to execute the work.  Notice that the `tx` of the channel is cloned, it's allowed as it is a mpsc channel, and together with the number received from the generator are moved within the pool closure.
The [parse function](https://github.com/danrusei/dev-state_blog_code/blob/master/fanOut_fanIn/pattern_Rust/src/parse.rs) is similar to the one mentioned above for GO. It sends a GET request to the same server, deserializes the json into a struct and returns a string with details about the longest comment for the requested post. 

{{< code language="rust" isCollapsed="false" >}}
fn fan_out(rx_gen: Receiver<u32>, pool: ThreadPool, n_jobs: u32) -> Result<Receiver<String>, Box<(dyn Error)>>{
    let (tx, rx) = channel();
    for _ in 0..n_jobs {
        let tx = tx.clone();
        let n = rx_gen.recv().unwrap();
        pool.execute(move || {
            let parse_result = parse(n).unwrap();
            tx.send(parse_result)
                .expect("channel will be there waiting for the pool");
        });
    }
    Ok(rx)
}
{{< /code >}}

The fan_in function takes the `fan_out Receiver` and returns the `fan_in Reicever`. If I said on Go implementation that ranging over the channel is one of my favorite syntax, the fact that the channel Receiver in Rust implements the `IntoIterator ` trait is one of my favorites as well. [Iterators](https://doc.rust-lang.org/std/iter/trait.Iterator.html) in Rust are very rich and provide plenty of methods. Iterators are frequently used whenever we are dealing with collection types.

{{< code language="rust" isCollapsed="false" >}}
fn fan_in(rx_fan_out: Receiver<String>) -> Result<Receiver<String>, Box<(dyn Error)>>{
    let (tx, rx) = channel();
    thread::spawn(move || {
        for value in rx_fan_out.iter().map(|value| format!("{} _ Processed", value) ) {
            tx.send(value).expect("could not send the value");
        }
    });

    Ok(rx)
}
{{< /code >}}

In the main body the threadpool is defined and the pipeline is executed. The results of the pipeline are printed out at the console.

{{< code language="rust" isCollapsed="false" >}}
fn main() {
    let n_workers = 4;
    let n_jobs = 8;
    let pool = ThreadPool::new(n_workers);

    let rx_gen = generator(n_jobs).unwrap();
    let rx_fan_out = fan_out(rx_gen, pool, n_jobs).unwrap();
    for item in fan_in(rx_fan_out).unwrap() {
        println!("{}",item)

    }
}
{{< /code >}}


### Run the code

{{< code language="bash" isCollapsed="false" >}}
$ /usr/bin/time -f "%MK %E" target/release/pattern_rust
 
12 Americo@estrella.net 179 _ Processed
10 Pearlie.Kling@sandy.com 179 _ Processed
85 Holden@kenny.io 162 _ Processed
73 Kenyon@retha.me 205 _ Processed
29 Estel@newton.ca 166 _ Processed
58 Juana_Stamm@helmer.com 207 _ Processed
73 Kenyon@retha.me 205 _ Processed
50 Samara@shaun.org 189 _ Processed

5208K 0:00.38
{{< /code >}}

## Rust pattern - async

### Async / Await

Asynchronous Programming in Rust is the new hype around so why not to try to recreate our pipeline in the asynchronous world.

Asynchronous code allows us to run multiple tasks concurrently on the same OS thread. Switching between the system threads and sharing data between threads involve a lot of overhead, the async promise is to eliminate these costs.
**async/await** is Rust's built-in tool for writing asynchronous functions that look like synchronous code. `async` transforms a block of code into a state machine that implements a trait called `Future`. `await` asynchronously waits for the future to complete, allowing other tasks to run if the future is currently unable to make progress. So, unlike a regular function, calling an `async fn` doesn't have any immediate effect. Instead, it returns a `Future` and to execute the future, use the .await operator.

Threads are natively supported by the operating system, asynchronous functions require special support from the language or libraries. There are a number of Async I/O runtimes in the Rust ecosystem, the two major ones are [async-std](https://async.rs/) and [tokio](https://tokio.rs/). I encourage you to try out both of them and choose one which fits your needs, however in this article I'm using async-std runtime. What I really like about async-std is the idea that it emulates standard library, basically you are using similar APIs with the ones you are used to.   
Executing asynchronous Rust program consists of a collection of native OS threads, on top of which multiple stackless coroutines are multiplexed.`Tasks` in async_std are one of the core abstractions,  that works with an API similar to std::thread. There are other useful modules implemented in async_std, like [Stream](https://docs.rs/async-std/1.5.0/async_std/stream/index.html), [Sync](https://docs.rs/async-std/1.5.0/async_std/sync/index.html) and others.
One interesting piece of information I found [on the official blog post](https://async.rs/blog/stop-worrying-about-blocking-the-new-async-std-runtime/) :
> The new task scheduling algorithm and the blocking strategy are adaptations of ideas used by the Go runtime to Rust.

### Pipeline in the async world

The aync-std runtime is in rapid development and it will take a little bit of time until most of the std libraries will be implemented in the async world.  if you are navigating through its modules, you'll see that some of them are marked as unstable, but alot were already implemented. Unfortunately for our case `async_std::sync::channel` is unstable yet, so we'll have to use the `futures::channel::mpsc` instead . `StreamExt` and `SinkExt` are extension traits to `Stream` and respective `Sink`, that provides a variety of convenient combinator functions, like `send` for the channels.

{{< code language="rust" isCollapsed="false" >}}
use async_std::task;
use futures::channel::mpsc::{channel, Receiver};
use futures::stream::StreamExt;
use futures::sink::SinkExt;
{{< /code >}}

For convenience I'm creating an Error type. `std::error::Error` is a trait representing the basic expectations for error values, `dyn` is needed as the compiler does not know the concrete type that is being passed. A way to write simple code while preserving the original errors is to Box them. The drawback is that the underlying error type is only known at runtime and not statically determined. The stdlib helps in boxing our errors by having Box implement conversion from any type that implements the Error trait into the trait object `Box<Error>`, via From.

{{< code language="rust" isCollapsed="false" >}}
pub type Error = Box<(dyn std::error::Error + Send + Sync + static)>;
{{< /code >}}

If you compare with the previous implementation of the `generator` function, using the threads, it looks similar. The 'thread' was swapped with `tak`, and `async` was added to the function definition. Otherwise the syntax is the same, that is the beauty of using the async-std.
The task is calling `spawn`, which produces a `JoinHandle`, that implements a `Future` and can be awaited. The spawned task is detached from the current task. This means that it can outlive its parent (the task that spawned it), unless this parent is the root task.

{{< code language="rust" isCollapsed="false" >}}
async fn generator(n_jobs: u32) -> Result<Receiver<u32>, Error> {
    let (mut tx, rx) = channel(0);
    let mut rng = rand::thread_rng();
    let nums: Vec<u32> = (0..n_jobs).map(|_| rng.gen_range(1, 100)).collect();
    task::spawn(async move {
        for num in nums {
            tx.send(num).await
                .expect("Could not send the generated number over the channel")
        }
    });
    Ok(rx)
}
{{< /code >}}

The async function sets up a deferred computation. When the function is called, it will produce a `Future<Output = Result<Receiver<String>, Error>>` instead of immediately returning a `Result<Receiver<String>, Error>` . 
Within the loop we match against the values received from the channel, if there is any new, a task is spawn.  
I have adapted the [parse()](https://github.com/danrusei/dev-state_blog_code/blob/master/fanOut_fanIn/pattern_Rust_Async/src/parse.rs) function to be asynchronously as well. I'm using [surf](https://crates.io/crates/surf) crate, which is based on async_std. The result of the computation is sent over the channel to be further processed by the `fan_in` function.
The `JoinHandle` of each spawn task was added to the Vec and awaited on each to be completed before the main function concludes.

{{< code language="rust" isCollapsed="false" >}}
async fn fan_out(mut rx_gen: Receiver<u32>, n_jobs: u32) -> Result<Receiver<String>, Error> {
    let (tx, rx) = channel(n_jobs as usize);

    let mut handles = Vec::new();
    loop {
        match rx_gen.next().await {
            Some(num) => {
                let mut tx_num = tx.clone();
                let handle = task::spawn(async move {
                    let rep = parse(num).await.unwrap();
                    tx_num.send(rep).await
                    .expect("Could not send the parsed string over the channel");
                        
                });
                handles.push(handle);
            }
            _ => break,
        }
    }

    for handle in handles {
        handle.await;
    }

    Ok(rx)
}
{{< /code >}}

`fan_in` function is straight forward, it spawn a task and awaits for the Receiver to get all the data. The received string is further composed by adding "_ Processed".

{{< code language="rust" isCollapsed="false" >}}
async fn fan_in(mut rx_fan_out: Receiver<String>) -> Result<Receiver<String>, Error> {
    let (mut tx, rx) = channel(0);
    task::spawn(async move {
        loop {
            match rx_fan_out.next().await {
                Some(value) => {
                    let processed_value = format!("{} _ Processed", value);
                    tx.send(processed_value).await
                        .expect("Could not send the processed string over the channel");
                }
                _ => break,
            }  
        }
    });

    Ok(rx)
}
{{< /code >}}

`async_std::main` attribute macro enables main function to become async. The pipeline is cleanly composed in the end. 

{{< code language="rust" isCollapsed="false" >}}
#[async_std::main]
async fn main() -> Result<(), Error> {
    let n_jobs = 8;
    let mut rx_fan_in = fan_in(fan_out(generator(n_jobs).await?, n_jobs).await?).await?;
    
    loop {
        match rx_fan_in.next().await {
            Some(value) => {
                println!("{}", value);
            }
            None => break,
        }
    }

    Ok(())
}
{{< /code >}}

### Run the code

{{< code language="bash" isCollapsed="false" >}}
$ /usr/bin/time -f "%MK %E" target/release/pattern_rust_async

85 Holden@kenny.io 162 _ Processed
81 Keshaun@brown.biz 214 _ Processed
24 Joshua.Spinka@toby.io 221 _ Processed
21 Adolph.Ondricka@mozell.co.uk 218 _ Processed
43 Faustino.Keeling@morris.co.uk 168 _ Processed
86 Drew_Lemke@alexis.net 196 _ Processed
6 Lurline@marvin.biz 181 _ Processed
20 Claudia@emilia.ca 223 _ Processed

15980K 0:00.27
{{< /code >}}

## Conclusion

The data from running the programs is not really relevant, as the volume of the requests were too low. I could have increased the number of tasks/workers/jobs but I didn't, as my intention was not to compare the performance but to show the ways to create pipelines using the 2 languages. The scripts are available on [Github](https://github.com/danrusei/dev-state_blog_code/tree/master/fanOut_fanIn)

Go has emerged as a language of the cloud infrastructure, as Rob Pike was saying in [the recent interview](https://evrone.com/rob-pike-interview). It's easy to observe that most of the [CNCF projects](https://www.cncf.io/projects/) are written in GO. I like Go's first-class support for concurrent computation, the [compatibility promise](https://golang.org/doc/go1compat), its syntax simplicity and its standard library.

Rust has shown a large adoption lately, by no surprise, for 4 years in a row Rust seems to be the most loved programming language based on the [Stackoverflow survey](https://insights.stackoverflow.com/survey/2019#most-loved-dreaded-and-wanted). I like Rust's ownership and borrowing model that guarantee memory-safety and thread-safety, the functional features especially the Iterators and the Generics.

The async runtimes are in rapid development and it will take a little bit of time until most of the std libraries will be ported in async world. In parallel there are many crates that are in the process of adopting the async. I do think that this will drive greater Rust adoption.

## References
* [Go Concurrency Patterns: Pipelines and cancellation](https://blog.golang.org/pipelines)
* [Concurrency Patterns in Go](https://learning.oreilly.com/library/view/concurrency-in-go/9781491941294/ch04.html#concurrency_patterns)
* [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/)
* [Async programming in Rust with async-std](https://book.async.rs/introduction.html)
* [Stream Concurrency](https://blog.yoshuawuyts.com/streams-concurrency/)
