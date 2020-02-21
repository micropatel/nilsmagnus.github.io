+++
date = "2018-02-25T14:28:58+01:00"
draft = false
title = "Concurrency made easy(easier) with coroutines"
tags        = [ "kotlin", "coroutines", "async"]
+++


Coroutines can be looked at as lightweight threads that enables us to write simple 
concurrent code in kotlin. It enables us to execute concurrent code without much 
effort and write async code in a sequential style, 
hiding the noise introduced by explicitly handling async events and callbacks. The result is 
readable, high-performant code.

Instead of the dealing with callbacks and synchronisation, the developer can focus on real, 
value-adding code and let the runtime handle callbacks transparently.

In this post I will give a short background for coroutines and show some samples of 
how coroutines can simplify and improve async code. 


## The basic case for coroutines

Coroutines are cheaper than Threads and are therfore faster to create. Threads are by default 
allocated 1024kb on most jvms, so creating a lot of new threads will cost you
a lot of memory. This might be fine for a low number of threads, but dealing with large number of
threads quickly becomes a problem. 

Starting 100_000 concurrent threads on my laptop quickly produces an OutOfMemoryError:
```java
(0..100_000).forEach {
    Thread {
       sleep(1000)
       print(".")
    }.start()
}
Thread.sleep(10_000)
```


```bash
Exception in thread "main" java.lang.OutOfMemoryError: unable to create new native thread
	at java.lang.Thread.start0(Native Method)
	at java.lang.Thread.start(Thread.java:717)
	at nilspils.ManyThreadsKt.main(ManyThreads.kt:11)
```

Using 100_000 coroutines on the other hand has no difficulties to complete:

```java
(1..100_000).forEach{
    launch {
        delay(1000)
        print(".")
    }
}

// "runBlocking" is a special coroutine that runs on mainthread 
// and waits to finish before continuing main
runBlocking { 
    delay(10_000)   
    println("done")
}
```

Kotlin also has built in support for channels meant for communicating between coroutines. We will get back to that (and CSP) later in the article.

## Launching a coroutine

To launch a coroutine, the `launch` keyword is used. Everything inside a `launch` block is a co-routine that will now be 
executed concurrently by a special `Threadpool` managed by the runtime. Since the coroutine is managed by the runtime, we cannot know for certain
that it will execute before or after the code outside the coroutine, the scheduling of the coroutine is entirely done by the runtime. 

```java
launch{
    println("in coroutine ${Thread.currentThread().getName()}") 
    delay(199)
}
println("outside coroutine ${Thread.currentThread().getName()}") 
```
outputs  
```bash
outside coroutine: main
inside coroutine: ForkJoinPool.commonPool-worker-9
```
Notice that the thread of the coroutine is named `ForkJoinPool.commonPool-worker-9`, a thread managed by the runtime. 

The `delay()` method is a suspendable function, wich means that at this point the runtime knows that it is safe to suspend the execution of the coroutine at this point. 


## Callbacks vs coroutines

Using callbacks to deal with async code can quickly lead to "callback-hell" style code, where you end up with 
nested. 

Consider the following async-code 
```java
fun asyncCallBackCodeProcessing(arg: String) {
    requestValidation(arg) {
        postValidationAction { validationResult ->
            saveResult(validationResult) { savedResult ->
                writeResultToResponse(savedResult)
            }// aka-
        } // call-
    } // back-
} // hell
```

And this does not handle any errors in the process. What starts out as a simple call-back evolves into a complex callback-chain
that quickly becomes code that is both unreadable and hard to reason about and test. 

Now consider the equivalent sequential code:

```java
suspend fun sequentialProcessing(arg: String) {
    val validationResult = requestValidation(arg)
    postValidationAction(validationResult)
    val savedResult = saveResult(validationResult)
    writeResultToResponse(savedResult)
}
```

Can you see all the times we __don't need to care about any asynchronous behavior__ in the latter code-example?

On android we can take use of the built-in AsyncTask for dealing with async operations. It is an interface that allows you 
 to not dealing with callbacks yourself, but you have to split the code into backtround-code in the method 
  `doInBackground` and what happens when the code has completed in the method `onPostExecute`. 
 

In one of my android-apps I have the following code

```java
// with AsyncTask
fun addPackage(phoneId: String, packageNumber: String) {
        object : AsyncTask<Void, Void, Boolean>() {
            override fun doInBackground(vararg params: Void): Boolean? {
                try {
                    return server.addPackage(phoneId, packageNumber)
                } catch (e: IOException) {
                    Log.e(TAG, "Error adding package to server", e)
                }
                return false
            }

            override fun onPostExecute(success: Boolean?) {
                super.onPostExecute(success)
                val intent = Intent(BroadcastActions.ADD_PACKAGE.name)
                val bundle = Bundle()
                bundle.putBoolean("success", success!!)
                bundle.putString("package", packageNumber)
                intent.putExtras(bundle)
                context.sendBroadcast(intent)
            }
        }.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR)
    }    
```

Making use of coroutines I can now have a sequential style code without having to deal with any of these callbacks, and the 
code instantly becomes more understandable:

```java
// with coroutine 
fun addPackage(phoneId: String, packageNumber: String) {
   launch {
        val success = server.addPackage(phoneId, packageNumber)
        val intent = Intent(BroadcastActions.ADD_PACKAGE.name)
        val bundle = Bundle()
        bundle.putBoolean("success", success)
        bundle.putString("package", packageNumber)
        intent.putExtras(bundle)
        androidContext.sendBroadcast(intent)
   }
}
```

Now all my async-code is gone in favour of a simple coroutine, and all I had to do was using the `launch` method.

## CSP and actors with coroutines

If you want to do some paralell work, not just concurrent, you should be aware of CSP or the actor model. CSP(communicating sequential processes)
is not a new way of thinking, but dates back to a theorem from [Tony Hoares](http://www.usingcsp.com/). The theory in practice ends up in a number of 
routines that communicates over `channels` instead of some shared state. The theory can be summarized as this:

_Do not communicate by sharing memory; instead, share memory by communicating._

In kotlin we use the built in `Channel<T>` as the link between any number of coroutines as a form of typed micro-message-bus. 

Here we have 150 coroutines that each post an int to a common channel and a routine that reads from this channel. 

```java 
// simple CSP example

val channel = Channel<Int>()

(1..150).forEach { number ->
    launch {
        channel.send(number)
    }
}

runBlocking {
    for (i in channel) {
        println("got from channel" + i)
    }
}

```

With csp we use _anonymous coroutines and named channels_. A different pattern, the actor model, uses _named coroutines_ with anonymous channels. 

The principle is much the same: don't communicate with a shared state, communicate over a channel.  In kotlin we use `Actor<T>` wich implicitly 
has a channel of type `<T>` it can refer to. 

```java 
// simple actor example

val actor = actor<Int> {
    for (message in channel) {
        println("got " + message)
    }
}

launch {
    repeat(10){count->
        actor.send(count)
    }
    actor.close()
}
```


## Learn more

* [Youtube: introduction to coroutines](https://www.youtube.com/watch?v=_hfBv0a09Jc)
* [Youtube: deep dives into coroutines](https://www.youtube.com/watch?v=YrrUCSi72E8)
* [kotlinlang.org: Coroutine reference](https://kotlinlang.org/docs/reference/coroutines.html)
* [github.com: Coroutines by example](https://github.com/Kotlin/kotlinx.coroutines/blob/master/coroutines-guide.md)
* [Rob Pike featured on the golang-blog: concurrency is not parallelism](https://blog.golang.org/concurrency-is-not-parallelism)
