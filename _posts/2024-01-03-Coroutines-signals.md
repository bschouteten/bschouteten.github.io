---
title: Generating signals with coroutines
date: 2024-03-01
author: bschouteten
---

# Signals with coroutines

In our previous blog we started with creating signals and came to the conclusion that this wasn't that easy. So to build up tests in the same way as in SystemVerilog we have checked C++ coroutines. Following this we had interesting look in how C++ coroutines work, which we can now utilize to extend our Verilator design and start creating some initial signal generation and tests.

As first let's see where we place our tests and how they work, with this seeing how we can extend our design and how everything should work together. In our current design we have a `run` function in the `cAPBUart16550TestBench` class. We pass in the number of clock cycles that we want our design to run and then we tick our clock so many times. In this setup we also saw that implementing signals within this `run` function is very difficult and we prefer to have the signal generation inside coroutines. Let's take a how this looks like for a software design.

![Software design](/images/blog5/softwareDesign.png)

Note that the cTestBench and cClockManager have been closed since those were already discussed and didn't change. The `cAPBUart16550TestBench` is updated, our `run` method now doesn't take a number of clock cyles anymore but just runs. We added a coroutine for `test` and a coroutine for `generateReset`. The `test` function is as the name describes a test, where we do something with the design and verify that the outcome is correct and returning the result, where `generateReset` will be used to generate a reset signal. Next to this a cClock is added, which is used to control the clock of the design. 

This sofware design is not so exiciting and doesn't show much of the interaction and design flow. To show this, let's create a sequence diagram and see how we want it to work.

![Sequence diagram](/images/blog5/sequenceDiagram.png)

Our design starts again in the `run` routine, which is the controlling part of our test. It will create the coroutine `test`, which then starts to run until it hits a coroutine expression. For now our test is very simple and will only generate a reset, but in the future this test shall be expanded and perform more actions. The reset is generated within the coroutine `generateReset`, this is a seperate coroutine so that we can re-use it but also so that we could generate more signals simultaneously. Due that we will `co_await` the `generateReset` in the `test` function, the test function shall be suspended until the `generateReset` is finished. Meaning that we start our generate reset, as we have seen in our previous blog, we first wait for a few negative edges on the clock and then proceed to set the reset signal 'low', wait for a few negative clock edges and then set the signal 'high' again. When the signal goes high again the function is completed and we can return. But here lies the second point, we have to wait for the negative clock edge. Since we are in a coroutine, we can `co_await` the negative clock edge of the corresponding clock, by doing this the program will return to the `run` function. In our run function we can now `tick` the clock again, as we have done previously. The new thing now is that we have to extend our clock to wake up a waiting coroutine. This is visible in our sequence diagram, the clock ticks, first positive edge, there is no waiting coroutine so it continous and ticks again. This second tick is the negative edge, where our `generateReset` coroutine is waiting on. This is now resumed and will continue with it's logic, where in our case we wait for two negative edges and then change the pin setting. As we follow the sequence diagram we can see this happen. When the `generateReset` function is completed it will return to the `test` coroutine, which then executes the following part of the `test`, for now this is still empty. When the `test` coroutine finishes the `run` routine will end and cleanup, finishing our test.

# Updating run

Let's start with adjusting our run routine. Previously we passed in the number of cycles that we wanted to simulate, this is not necessary anymore since this is dynamically determined by our coroutines. First we create our coroutine, which in this case is `test`, since we added the `operator bool` to our coroutine we can check very easily if is finished. Due to this fact our `for` loop is removed and we use a `while` loop. Within this loop we use the same `tick` function as we already build, which will advance our clocks. At the point we break out of the loop we have to do one more `tick` so that the last state of the system is also written into our trace file. After this we can printout the result of our test case and return the result.

``` c++

int cAPBUart16550TestBench::run()
{
    sCoRoutineHandler myTest = test();

    while (!myTest)
    {
        tick();
    }

    tick();

    INFO << "Test result:" << myTest.getResult() << "\n";

    return myTest.getResult();
}

```

The following function we introduce is the `test` function. As described previously, currently we will only generate a reset. To lengthen the simulation a bit more we add 5 waits on the positive clock edge before we end. Since we first want to generate a reset we `co_await` the `generateReset` functionality, which we will describe later on. To generate the 5 waits on the positive clock edge we use a simple for loop where we internally `co_await` the positive clock edge. Due to the fact that the coroutine is suspended we will actually wait until 5 positive clock edges have passed before we continue. This makes reading and writing it easier then when we have to think about the number of clock ticks. Following those 5 positive clock edges our test will conclude and we return `true` as the test has succeeded.

``` c++
sCoRoutineHandler<bool> cAPBUart16550TestBench::test()
{
    INFO << "Setup test \n";

    co_await generateReset();

    for(uint8_t i = 0; i < 5; i++)
    {
        co_await cClockAwaitable(pclk, eClockEdge::positive);
        INFO << "Coroutine resumed \n";
    }
    
    INFO << "Coroutine return \n";
    co_return true;
}

```

Generating the reset goes in a similar way as in the `test` routine where we wait for the clock edges. At the beginning we directly set our reset pin 'high', following this we `co_await` the negative edge of our clock. In our sequence diagram we waited for two clock edges, which is a bit different now, where we wait for 5 negative edges. When the 5 negative edges have passed we set the reset signal 'low' and wait for 5 more negative clock edges. When those are passed we set the reset signal back to 'high' and return from our coroutine.

``` c++

sCoRoutineHandler<bool> cAPBUart16550TestBench::generateReset()
{
    INFO << "Generate reset \n";
    _core->PRESETn = 1;

    for(uint8_t i = 0; i < 5; i++)
    {
        co_await cClockAwaitable(pclk, eClockEdge::negative);
    }

    INFO << "Reset set high \n";
    _core->PRESETn = 0;

    for(uint8_t i = 0; i < 5; i++)
    {
        co_await cClockAwaitable(pclk, eClockEdge::negative);        
    }

    _core->PRESETn = 1;
    INFO << "Reset done \n";
    co_return true;
}

```

This function is functions exactly the same as when we would write it in SystemVerilog. 

``` systemverilog
initial
begin
    PRESETn = 'b1;
    repeat (5) @(negedge PCLK);
    PRESETn = 'b0;
    repeat (5) @(negedge PCLK);
    PRESETn = 'b1;
end
```

# starting coroutine from a clock edge

There is one thing which we still need to implement, resuming coroutines from a clock edge. We already have a nice mechanism in place to toggle the clocks at the right moment. To get this up and running we just need to link the coroutine and the clock together. We already created the clock class, now we just need to expand it a bit so that it can handle coroutines. 

Every clock will have a positive and negative edge, according to any of those edges we want to resume corresponding waiting coroutines. So on every clock edge we need to check for a waiting coroutine. So let's start by expanding our ```toggle()``` function, in our initial design we inverted the clock signal itself and then set the next period of time. Let's now update the function and resume any waiting coroutine, as mentioned earlier we can have positive and negative edge coroutines waiting. Since this function is only called when the clock pin changes state we know exactly according to the current state of the clock which edge has been triggered. When the clock pin is 'high' we went from 'low' to 'high' or a positive edge. In our function we add a check and then call the appropriate function to process the event.

``` c++

void toggle(void)
{
    //toggle clock, changing the state in the simulation
    _clk = !_clk;

    //Update TimeKeep
    _timeToNextEvent = _clk ? _highPeriod : _lowPeriod;

    //call routines waiting for posedge/negedge
    if (_clk)
    {
        resumeWaitForPosedge();
    }
    else
    {
        resumeWaitForNegedge();
    }
}

```

Let's take a look into our new functions ```resumeWaitForPosedge``` and ```resumeWaitForNegedge```. We want them to resume any of the waiting coroutines, but we need to know which coroutine is waiting for which clock edge. Well for this we use a queue, two queue to be precise, a ```posedgeQueue``` and a ```negedgeQueue```. Both will store the type ```std::coroutine_handle<>```, which is the handle to a waiting coroutine. The queue is needed since there can be more then one coroutine waiting on the edge and we want to resume all the corresponding coroutines. Since we now have the queue we can just go through the queue and resume all the corresponding coroutine handles in it. There is one small caviat here, since we resume the coroutines again they will run until the next wait statement is hit. If this is a clock edge it will be appended in the queue, but our system is still processing the queue, at this point the coroutine will be resumed again. This behaviour is not what we want, so therefore we make a copy of the queue and process this copy. This then holds all the coroutines which must be resumed at this edge of the clock, where the original queue in our class will hold all the new coroutines which should be resumed at the following clock edge. This mechanism counts for the positive as for the negative clock edge.

``` c++

void resumeWaitForPosedge()
{
    if(!posedgeQueue.empty())
    {
        // Create a new temporary empty queue
        std::queue<std::coroutine_handle<>> queueCopy;

        // Swap empty queue and posedgeQueue
        posedgeQueue.swap(queueCopy);

        do
        {
            //pop the function from the queue
            std::coroutine_handle<> h = queueCopy.front();
            queueCopy.pop();

            //resume the coroutine
            h.resume();
        } while (!queueCopy.empty());
    }
}

void resumeWaitForNegedge()
{
    if(!negedgeQueue.empty())
    {
        // Create a new temporary empty queue
        std::queue<std::coroutine_handle<>> queueCopy;

        // Swap empty queue and posedgeQueue
        negedgeQueue.swap(queueCopy);

        do
        {
            //pop the function from the queue
            std::coroutine_handle<> h = queueCopy.front();
            queueCopy.pop();

            //resume the coroutine
            h.resume();
        } while (!queueCopy.empty());
    }
}

```

Last part for our clock is the most fun, we now have to add a coroutine handle into our queue. As we have seen in the last part of our coroutines, coroutines can be waited on with the ```co_await``` operator and a ```awaitable``` type. So for any of our coroutines to wait a clock edge we will need a special ```awaitable``` for the clock. Within this awaitable we can set the logic to add the coroutine handle in the appropiate queue. Let's start by creating a class for our awaitable, within this awaitable we need to have knowledge about the clock on which we are waiting and on which clock edge. In the constructor of the class we pass in a pointer to our clock and we pass the clockedge to wait for. As we have seen previously to make something a ```awaitable``` type it needs three functions, so we add those as well. The ```await_ready``` will always return false, since we always want to suspend the coroutine and make it wait for the clock edge. ```await_suspend``` is the most important function, it is used to kick off any related function when suspending a coroutin. In our case this is excellent, it takes a handle to the coroutine which we can use this to store in the appropiate queue of the clock. This is also the main reason that we passed in a point to our clock, within the ```await_suspend``` we can now call the ```wait_edge ``` function of the clock and pass the coroutine handle to it. The ```wait_edge``` function is simple, it just pushes the coroutine handle into the corresponding queue and that's it. Last function of our ```awaitable``` is the ```await_resume```, in our case we don't need to return anything so we can easily keep it void and don't return anything.


``` c++

class clockAwaitable
{
    private:
    cClock* _clock;
    eClockEdge _edge;

    public:
    clockAwaitable(cClock* aClock, eClockEdge edge) : _clock(aClock), _edge(edge){};

    bool await_ready()
    {
        return false;
    }

    auto await_suspend(coroutine_handle<> handle)
    {
        _clock->waitEdge(_edge, handle);
        return std::noop_coroutine();          
    }

    void await_resume()
    {

    }
};

void waitEdge(eClockEdge edge, coroutine_handle<> h)
{
    switch (edge)
    {
    case eClockEdge::positive:
        posedgeQueue.push(h);
        break;

    case eClockEdge::negative:
        negedgeQueue.push(h);
        break;
    
    default:
        FATAL << "Unkown clock edge \n";
        break;
    }
}

```

# running it all together

Now everything is coded we can run a simulation and see what comes out of it. We have added some debug statements in which we can follow our test, let's also enable our trace so we can check the waveform.

[INFO] Setup test\
[INFO] Generate reset\
[INFO] Reset set high\
[INFO] Reset done\
[INFO] Coroutine resumed\
[INFO] Coroutine resumed\
[INFO] Coroutine resumed\
[INFO] Coroutine resumed\
[INFO] Coroutine resumed\
[INFO] Coroutine return\
[INFO] Test result: 1

In our log it visible that the code exactly does as we expect. First `test` is setup and immediately a reset is generated. Following this we see 5 times that the `test` coroutine is resumed and the test is ended with 1, which is the same as `true`.

Looking into the waveform, we can see that the reset is generated and following the reset we waited for 5 positive clock edges.

![Reset](/images/blog5/reset.png)

We have now updated our run routine and created our first small test program. This test program uses a reset which is triggered according to the negative clock edge. In our next blog we will take a look into creating bus signals, more specifically APB bus signals. 
