---
title: Generating signals with coroutines
date: 2024-01-24
author: bschouteten
---

# Signals with coroutines
In the previous blog we started with generating signals, where we came to the conclusion that we needed coroutines for this. Following that we explored C++ 20 coroutines, investigating how they work and what they do. Now we have a stable starting point and can use this to generate signals into our simulation. As done in the previous blog, we will start with the reset signal, however now fully controlled by a coroutine and following this we will start generating some APB bus signals. 

# starting coroutine from a clock edge

Our first part will be resuming coroutines from a clock edge, we already have a nice mechanism in place to toggle the clocks at the right moment. To get this up and running we just need to link the coroutine and the clock together. We already created the clock class, now we just need to expand it a bit so that it can handle coroutines. 

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

So now we have added everything in the clock itself to suspend and resume a coroutine, we now just have to create a coroutine. In all our previous examples we let the design run for a number of ticks in our ```run``` function. We will adjust this function to just tick the clock forward and check if the coroutine has finished, everything else is controlled from the coroutine. Let's look at how this is constructed and see what happens.

``` c++

int cAPBUart16550TestBench::run()
{
    // loop 20 times
    int i = 0;

    sCoRoutineHandler myTest = test();

    do
    {
        i++;
        tick();
    } while (!myTest && (i < 20));

    return 0;
}

sCoRoutineHandler<bool> cAPBUart16550TestBench::test()
{
    INFO << "Setup test \n";

    for(uint8_t i = 0; i < 5; i++)
    {
        co_await cClockAwaitable(pclk, eClockEdge::positive);
    }
    
    INFO << "Coroutine return \n";
    co_return true;
}

```

TODO: Describe above functionality, 
TODO: Rework above functionality since it needs to show one more clock pulse
TODO: Check coroutine end returning, could be improved