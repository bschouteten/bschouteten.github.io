---
title: Coroutines in verilator
date: 2024-01-24
author: bschouteten
---

# Coroutines in Verilator

In the previous blog we introduced clock signals into our testbench. We looked into creating a single clock but also misused our design to create multiple clocks. With the fact that Verilator is a cycle based simulator we can now add stimulus to our design according to our clock. Due to the fact that our designs are synchronous, it's possible to connect the stimulus to our clock signal.

# Generating our first signals

Let's look at some system Verilog testbench code to see how this drives signals to the device under test (DUT). First up the reset signal, as we can see in the code snippet below, reset is started high and after 5 negative clock edges it will be set low for 5 negative edges, after this it is set high for the rest of the simulation. This is a quite common structure to setup a reset signal even for other structures. 

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

Creating something like this with C++ in Verilator is a bit more difficult, our testbench will be looping to tick forward, as we have seen in our previous designs. It's just a plain for loop and will loop for the given number of iterations. Let's try to write some C++ code which would generate the reset pattern as written above in the system Verilog design. Our starting point will be the ```run(int numCycles)``` in the tb_apb_uart16550 class. Following our system Verilog design we would need to set the reset signal low from the fifth negative clock edge until the tenth negative clock edge. Now let's see how this theoretically transforms to our single 100MHz clock design. We start off with the clock signal low, which is our first tick or i = 0. So with i = 1 the clock will go up, i = 2 we have our first negative edge, this will then go on and on. So every even number the clock will be negative, so after 5 * 2 = 10 ticks we would have 5 negative edges and set the reset signal low, going high again after 10 * 2 = 20 ticks. Following this logic we could extend our run routine as below and generate the same reset signal as our system Verilog code. 

``` C++
int cAPBUart16550TestBench::run(int numCycles)
{
    for (size_t i = 0; i < numCycles; i++)   
    {
        tick();

        if(i >= 10 && i < 20)
        {
            _core->PRESETn = 0;
        }
        else
        {
            _core->PRESETn = 1;
        }

        #ifdef DEBUG_TESTBENCH
        DEBUG << "Loopcounter: " << i << "\n";
        #endif
    }

    return 0;
}
```

This works quite alright, however it's not really nice that we have to check if i is between 10 and 20 every single loop, especially when we have to do thousands of loops. It would add unnecessary delays, let's think of a way to optimize this. Maybe if we add it in a single function, it will only compare as long as we assert a reset. Let's try some code and see how this goes. We would need a reset function, this would then generate a reset through our design. What we want to control are the number of negative edges before we assert the reset and the number of negative edges before the reset is de-asserted again, both are passed as parameters. Our for loop is then adjusted a bit, it will loop for the number of negative edges before reset + the number of negative edges of the reset itself. After this the loop should end and the function should return, don't forget that we have to keep ticking our general testbench to proceed the clock forward and to evaluate our design. This would also change our ```run()``` function, since this function now first needs to call the reset function and then proceed with it's normal for loop.

``` c++
// This is a demo function and is not functional in any way
int cAPBUart16550TestBench::reset(int numNegEdgesBeforeReset, int numNegEdgesReset)
{
    for (size_t i = 0; i <= (numLoopsBeforeReset + numLoopsReset); i++)   
    {
        tick();

        if(i >= numNegEdgesBeforeReset && i < (numNegEdgesBeforeReset + numLoopsReset))
        {
            _core->PRESETn = 0;
        }
        else
        {
            _core->PRESETn = 1;
        }

        #ifdef DEBUG_TESTBENCH
        DEBUG << "Loopcounter: " << i << "\n";
        #endif
    }

    return 0;
}

int cAPBUart16550TestBench::run(int numCycles)
{
    reset(10, 10);

    for (size_t i = 0; i < numCycles; i++)   
    {
        tick();

        #ifdef DEBUG_TESTBENCH
        DEBUG << "Loopcounter: " << i << "\n";
        #endif
    }

    return 0;
}

```

At this moment we now have a possibility to create a nice reset signal without evaluating our if statement every loop, another nice point is that we can dynamically change our reset and change the number of negative edges needed. However we need to generate more signals then just a reset signal, so let's take a look how we would do this in system Verilog again and then proceed to our C++ code. Within our UART we need to control the APB bus signals and the UART RX/TX lines, both will have to be controlled and monitored to see if the UART is behaving as we expect. 

Let's first see how we would control the APB bus signals, within system Verilog we would create separate tasks to have certain functionality. Let's take a simplified design and continue to work with this. APB would need a read and write function, with those two tasks we can talk with the design and perform some testing. As we can see a bit below, we pass the address and the data we want and wait for our task to finish. Our task will set the needed signals needed for APB transaction and then wait for the positive clock edge. It will then perform the needed action and return once our read is finished.

``` systemverilog

  task automatic read (
    input  [PADDR_SIZE -1:0] address,
    output [PDATA_SIZE -1:0] data
  );
    PSEL    = 1'b1;
    PADDR   = address;
    PSTRB   = {PDATA_SIZE/8{1'bx}};
    PWDATA  = {PDATA_SIZE{1'bx}};
    PWRITE  = 1'b0;
    @(posedge PCLK);

    PENABLE = 1'b1;
    @(posedge PCLK);

    while (!PREADY) @(posedge PCLK);

    data = PRDATA;

    PSEL    = 1'b0;
    PADDR   = {PADDR_SIZE{1'bx}};
    PWRITE  = 1'bx;
    PENABLE = 1'b0;
  endtask

```

Seems quite easy and shouldn't be to difficult to build in C++ right. Well let's start to think about this a bit more, we can write a function which take the same parameters. Following the example from our reset, our function needs a for loop which ticks the design. For this it's not needed to pass any clock cycles, since we can check the corresponding pins. Sounds not too difficult, even sounds like a small thread, the action is started and returned with the value when executed. Let's setup a function to do this, we pass the address we want to read into the function and receive the value back. However we have to take into account that we need to setup different signals on different moments, so we need some kind of small state machine to function. On the first step we set the address and select the first read, following this we wait one positive clock edge and enable the transaction. Now we have to wait until the ready signal is there before we can read the data from the bus, when this happens our small thread is ready and we can leave.

``` c++
// This is a demo function and is not functional in any way
uint32_t cAPBUart16550TestBench::APBRead(uint32_t address)
{
    uint8_t step = 0;
    uint32_t data = 0;

    while(!_core->PREADY)
    {
        if(step == 0)
        {
            _core->PSEL = 1;
            _core->PADDR = address;
            _core->PWRITE = 0;
        }
        else if(step == 1)
        {
            _core->PENABLE = 1;
        }

        tick();
        step++;
    }

    data = _core->PRDATA;
    _core->PSEL = 0;

    return 0;
}

```

Alright so with this function we should now be able to read from our design through the APB bus, however it's now very difficult to control the clocking of all the functions. This is becoming a drawback in this way, also the state machine is not very nice. For now it's not to difficult to understand and to use, however if we get more and more difficult designs it's going to be a nightmare. Also sending multiple signals to our design is impossible, we can only read from the APB and no other signals can be read or controlled. To support this we need to extend our APB read, making it more and more complex. Thinking ahead it seems that this approach is not really working properly, so let's think what our solution could be.

# Coroutines

As we identified previously it's difficult to control multiple signals simultaneously. In system Verilog many things can run in parallel, due that everything is clocked and handled according to this clock. In our general testbench we also have a very nice clock, so it would be very cool if we could mimic this parallelization. Well it's mentioned earlier, threads. Using threads would be an option, creating a small thread for certain functionality and kill the thread when it ends. This would suit our APB read task in system Verilog very well, however in large designs we would then need to create and control many threads. That's not really what we want, making our design way to complex and difficult to handle. However the threading idea is not so bad, preferred a small subroutine which could be suspended and resumed on a clock edge or on finishing a action. Lukily for us C++ 20 has introduced something like this, coroutines. A coroutine can be described as "functions whose execution you can pause", this means we can write the functions almost the same as in system Verilog.

Let's take an example before we start to go deeper into subroutines. As we look into the function below, we can setup the signals just as in system Verilog. The execution of this function is stopped and it waits on a event of the clock before it continues execution. This is exactly how it works in system Verilog, so this would suit our testbench perfectly. Let's go and dig deeper into this and implement it properly.

``` c++
// This is a demo function and is not functional in any way
uint32_t read(uint32_t address) 
{
    uint32_t result = 0;

    PSEL    = H;
    PENABLE = L;
    PADDR   = address;
    PWRITE  = L;
    waitPosedge(_pclk);

    PENABLE = H;
    waitPosedge(_pclk);

    while (PREADY == L) waitPosedge(_pclk);

    result = PRDATA;
}
```

## Coroutines how does it work

Coroutines are very special functions and are quite complex to understand, let's try to work through them and see how it works. A C++ coroutine has two very special types, the ```promise_type``` and the ```awaitable```. Both need to work in conjunction with each other to get a functional coroutine. The ``` promise_type``` controls the internal state of the coroutine and defines what happens when the coroutine is suspended and resumed. The ```awaitable``` is used to support user dependend actions into the coroutine. There are two default ```awaitable``` types, which are ```suspend_never``` and ```suspend_always```. Both do exactly what they tell to do, one let's the operation continue and doesn't suspend the coroutine where the ```suspend_always``` will always suspend the coroutine. Coroutines are stackless data elements which are created in specific heap elements and also controlled from this point. It's important to know that there is no direct access to the data of the coroutine itself, this always has to happen through the ```promise_type```. A function will be defined as coroutine when one of the following definitions is used, ```co_await```, ```co_yield``` or ```co_return```. The compiler will recognize them and know that it needs to make a coroutine. At first it will look for the ```promise_type``` as we mentioned above. For now this seems to be very difficult to understand and see the relation between all of it, so let's start building it up and see how it works.

A lot of small code snippets are shown below, those are for illustration purposes. They don't work on their own, usually they have a few things missing. Untill the paragraph of co_await, only the standard ```awaitable``` types of ```suspend_never``` and ```suspend_always``` are used.

## co_return

Let's start of with the most simple example of a coroutine and use the identifier ```co_return```. ```co_return``` is used to give a value back from a coroutine, it is the coroutine variant of ```return```. When ```co_return``` is added in the function, the compiler directly know that it is a coroutine and handle it accordingly. Let's make a simple function and add ```co_return```.

``` c++
void myCoroutine()
{
    co_return;
}
```

As we try to compile this function our compiler will give us an error, "error: unable to find the promise type for this coroutine co_return;". The compiler has seen that we want to make a coroutine for this function, but as mentioned earlier we need the ```promise_type``` to build a coroutine. A ```promise_type``` is a structure which exists out of at least 5 mandatory functions. 

At first the function ```get_return_object```, this will return the allocated heap memory object of the coroutine. This is needed to resume the coroutine but also to have access to the internal data, since the data of the corountine cannot be gotten in any other way.

Second function is the ```initial_suspend```, this function returns a ```awaitable```. The creator can determine which ```awaitable``` is used, in most cases one of the standard awaitables is used. When ```suspend_always``` is called the coroutine will directly be suspended before the function even has started. ```suspend_never``` will let the coroutine run untill the first coroutine expression (```co_await```, ```co_yield``` or ```co_return```). In special cases the user can use it's own ```awaitable```.

```final_suspend``` is the user-defined end part of the coroutine. Here lies the opportunity to execute special logic before returning back to the caller/resumer. This function returns just as ```initial_suspend``` a ```awaitable```, with this it can publish a result, signalling completion or resuming a different coroutine. It also gives the possibility to suspend the coroutine before destroying the frame. Which is important if we want to access the internal state of coroutine after completion.

```void return_void``` or ``` void return_value``` has to be in place to return a valuye from the user. Which of both to use depends on the parameters given to ```co_return``` which we will describe just below.

The last function is ```unhandled_exception```, it is called when an exception occurs within the context of a coroutine. So if the execution of the coroutine fails it will be called and the exception can stored/handled as by used wishes. It must be noted that using the coroutine after this can result in undefined behaviour and should be checked very carefully.

Let's now put this together and build a ```promise_type``` to start with, following this we will extend and play around with the coroutine.

``` c++
struct sCoRoutineHandler
{
    struct promise_type
    {
        sCoRoutineHandler get_return_object() { return{};}
        suspend_never initial_suspend() {return{};}
        suspend_never final_suspend() noexcept { return {};}
        void return_void() {}
        void unhandled_exception() {}
    };
};
```
In our example code we have wrapped the ```promise_type``` in the `sCoRoutineHandler`. With this we should be able to compile our previous function and run our first coroutine. There is just one small change needed to our previous function, instead of returning void we need to return the `sCoRoutineHandler`.

``` c++
sCoRoutineHandler myCoroutine()
{
    co_return;
}

int main(int argc, char** argv) 
{
    sCoRoutineHandler myCoroutineHandle = coroutine();
    return 0;
}

```

We now have run our first coroutine, it compiled and our program works. We have now seen one of the main keywords to create a coroutine, which is ```co_return```, it is used to complete execution of coroutine and return a value. Let's check how this part works and extend it to return a value and make sure that we cleanup our code correctly, while also explaining what the corresponding functions do. As first, we will expand our simple example and add some debug statements so we can follow the execution of the program. 

``` C++
struct sCoRoutineHandler
{
    struct promise_type
    {
        sCoRoutineHandler get_return_object() 
        {
            INFO << "Get coroutine return object\n";
            return sCoRoutineHandler{coroutine_handle<promise_type>::from_promise(*this)};
        }
        suspend_never initial_suspend() 
        {   
            INFO << "Initial suspend\n";
            return{};
        }
        suspend_always final_suspend() noexcept 
        { 
            INFO << "Final suspend\n";
            return {};
        }
        void return_void() {INFO << "Return void\n";}
        void unhandled_exception() {}
    };

    using handle_t = coroutine_handle<promise_type>;

    //Handle to coroutine
    handle_t _handle;

    // Constructor
    sCoRoutineHandler (handle_t handle) : _handle(handle) 
    {
        INFO << "Constructing coroutine\n";
    };

    ~sCoRoutineHandler()
    {
        INFO << "Destructing coroutine\n";
    }
};
```

As we can see in the code example above, the ```promise_type``` is encapsulated within the structure of the ```sCoRoutineHandler```. By doing this we can create functions which can communicate with our coroutine, which is helpfull afterwards but for now mainly helps understanding the sequence of events. Our main routine is still the same, so let's run the example and see our log:

[INFO] Get coroutine return object \
[INFO] Constructing coroutine \
[INFO] Initial suspend \
[INFO] Coroutine function start \
[INFO] Return void \
[INFO] Final suspend \
[INFO] Destructor coroutine

First the ```promise_type``` will be created, as we can see in our log the coroutine function of ```get_return_object``` is called. At this point the heap memory for the coroutine state is already constructed. Our ```get_return_object``` returns the coroutine object itself, which is then passed into the constructor of our ```sCoRoutineHandler``` object. We can now store the handle to the coroutine so we have the possibility to access it if we want to. The constructor is also the second part which is called, followed by ```initial_suspend```. As mentioned before at this moment we determine if the coroutine is suspended directly, continued until a ```co_await```, ```co_yield``` or ```co_return``` is used or that it returns a special ```awaitable``` determined by the user. In our example we use ```suspend_never``` so the coroutine will start directly. This can also be noticed by our next statement of coroutine function start, we are now in the coroutine itself. Currently we don't do anything else then just returning by ```co_return```, so as soon as our coroutine is started it ends. Since our ```co_return``` statement doesn't return a value we have implemented ```void return_void```, which is the following function that is executed. As last the ``` final_suspend``` is called, in this case we return  the ```suspend_always``` `awaitable`. This will suspend the coroutine so that we still have access to it's internal state. As last we have the destructor of the encapsulating object, which is called since our main loop has finished and it destroys the object.

Let's update ```initial_suspend``` and change the return parameter to ```suspend_always``` and re-run our example code. Since the coroutine is directly stopped we would expect that we don't see the coroutine function start and the corresponding function calls to close of the coroutine. Which indeed is true as we can see in the log:
[INFO] Get coroutine return object\
[INFO] Constructing coroutine\
[INFO] Initial suspend\
[INFO] Destructor coroutine

Now let's return a value from our coroutine, we can do this by calling ```co_return expr```. But we also should adjust our ```promise_type``` and change the ```void return_void()``` function to ```void return_value(expr)```. It's a bit confusing but our function returns void, since it can't directly return the value to the caller. So we need to adjust the function accordingly and store the value in our structure. After the coroutine has then finished we can retrieve the value from it. Let's see how this works in the code.

``` C++

struct sCoRoutineHandler
{
    struct promise_type
    {
        int _myReturnValue;

        sCoRoutineHandler get_return_object() 
        {
            INFO << "Get coroutine return object\n";
            return sCoRoutineHandler{coroutine_handle<promise_type>::from_promise(*this)};
        }
        suspend_never initial_suspend() 
        {   
            INFO << "Initial suspend\n";
            return{};
        }
        suspend_always final_suspend() noexcept 
        { 
            INFO << "Final suspend\n";
            return {};
        }

        void return_value(int value) 
        {
            INFO << "Return value\n";
            _myReturnValue = value;
        }

        void unhandled_exception() {}
    };

    using handle_t = coroutine_handle<promise_type>;

    //Handle to coroutine
    handle_t _handle;

    // Constructor
    sCoRoutineHandler (handle_t handle) : _handle(handle) 
    {
        INFO << "Constructing coroutine\n";
    };

    ~sCoRoutineHandler()
    {
        INFO << "Destructing coroutine\n";
    }

    int getReturnValue()
    {
        return _handle.promise()._myReturnValue;
    }
};

sCoRoutineHandler myCoroutine()
{
    INFO << "Coroutine function start\n";
    co_return 5; // Coroutine
}

int main(int argc, char** argv) 
{
    auto myTask = myCoroutine();
    INFO << "Coroutine return value: " << myTask.getReturnValue() << '\n';
    return 0;
}

```

As we run with the following code the following trace will be visible:
[INFO] Get coroutine return object\
[INFO] Constructing coroutine\
[INFO] Initial suspend\
[INFO] Coroutine function start\
[INFO] Return value\
[INFO] Final suspend\
[INFO] Coroutine return value: 5\
[INFO] Destructor coroutine

Here we can see very good why we have encapsulated the `promise_type`. Since we can't directly access the coroutine state, we need to store the handle to it and then call `promise()` to access it. This has been done in the `getReturnValue` function, in this way we access the `promise_type` variabele of _myReturnValue. But for this to work the coroutine still has to exist so therefore our `final_suspend` function returns `suspend_always`. If we would change this to `suspend_never` the coroutine state would already be destroyed and it would be undefined behaviour. Within this example we won't see any difference since the program doesn't do much. With this we have seen everything regarding the `co_return` expression and the basics of the `promise_type`.

## co_yield

Let's explore the second coroutine expression ```co_yield expr```, which is used to suspend execution returning a value. This works in the same way as with our ```co_return expr``` example, the only difference here is that our coroutine can still resume. Meaning that we still need to request the value from our co_routine when it is suspended. As for the co_yield we also need to add another special function into our ```promise_type```, ```suspend_always yield_value(expr)```. When a ```co_yield expr``` is executed we will automatically end up in this function and can at that point process the data. The ```yield_value``` function must return a `awaitable`, where ```suspend_always``` or ```suspend_never``` are the easiest. ```suspend_always``` will stop the coroutine and wait until it is resumed, where ```suspend_never``` will let the coroutine continue until the next coroutine expression. 

Next to this we also need a way to resume a suspended coroutine, in our previous examples we only ended the coroutine. To resume a coroutine we need to call `resume()` on the handle of the coroutine. To make this happen we added a function `resume()` to our sCoRoutineHandler. When the coroutine is now suspended we can resume it by calling this function. Let's try this out with a simple example.

``` C++

struct sCoRoutineHandler
{
    struct promise_type
    {
        int _myValue;

        sCoRoutineHandler get_return_object() 
        {
            INFO << "Get coroutine return object\n";
            return sCoRoutineHandler{coroutine_handle<promise_type>::from_promise(*this)};
        }
        suspend_never initial_suspend() 
        {   
            INFO << "Initial suspend\n";
            return{};
        }
        suspend_always final_suspend() noexcept 
        { 
            INFO << "Final suspend\n";
            return {};
        }

        suspend_always yield_value(int value)
        {
            INFO << "Yield value " << value << '\n';
            _myValue = value;
            return {};
        }

        void return_value(int value) 
        {
            INFO << "Return value\n";
            _myValue = value;
        }

        void unhandled_exception() {}
    };

    using handle_t = coroutine_handle<promise_type>;

    //Handle to coroutine
    handle_t _handle;

    // Constructor
    sCoRoutineHandler (handle_t handle) : _handle(handle) 
    {
        INFO << "Constructing coroutine\n";
    };

    ~sCoRoutineHandler()
    {
        INFO << "Destructing coroutine\n";
    }

    int getReturnValue()
    {
        return _handle.promise()._myReturnValue;
    }

    void resume()
    {
        _handle.resume();
    }
};

// Our coroutine is adjusted to the following:
sCoRoutineHandler coroutine()
{
    INFO << "Coroutine function start\n";
    co_yield 1;

    INFO << "Coroutine function resumed\n";
    co_yield 2;
    
    co_return 5; // Coroutine
}

// Where the main routine is adjusted to the following
int main(int argc, char** argv) 
{
    auto myTask = myCoroutine();
    
    INFO << "Coroutine suspended with value " << myTask.getValue() << '\n';
    myTask.resume();

    INFO << "Coroutine suspended with value " << myTask.getValue() << '\n';
    myTask.resume(); 
    
    INFO << "Coroutine return value: " << myTask.getValue() << '\n';
}

```

If we look into our coroutine we can see that it starts, then we suspend the operation by calling `co_yield` and passing in the value of 1. So our coroutine is now suspended and it's internal value is set to 1. Due that our coroutine is suspended the main function is running again and it will get the value from the coroutine, print this out and resume the coroutine. The same thing happens again, now only with the value of 2. At the end it returns with `co_return` as we have seen previously. 

Trace of the simple co_yield example:
[INFO] Get coroutine return object\
[INFO] Constructing coroutine\
[INFO] Initial suspend\
[INFO] Coroutine function start\
[INFO] Yield value 1\
[INFO] Coroutine suspended with value 1\
[INFO] Coroutine function resumed\
[INFO] Yield value 2\
[INFO] Coroutine suspended with value 2\
[INFO] Return value\
[INFO] Final suspend\
[INFO] Coroutine return value: 5\
[INFO] Destructor coroutine

In the trace we can see that the program does exactly what we want it do. It starts the coroutine and the coroutine runs until the first `co_yield` statement. We retrieve this value, print it and then let the coroutine resume. So from the coroutine perspective, `co_yield` is called, which transfers to the function `yield_value`. Here we store the passed value into our internal state. The main routine takes over and requests the value of our coroutine, which we can get through the `promise()`. Then the coroutine is resumed and the same things happens again, only now with the value of 2. Ending the coroutine with `co_return`.

## co_await

```co_await {awaitable}``` is used to suspend execution untill the coroutine is resumed. As parameter it takes a `awaitable` type, we have already seen two of those, ```suspend_never``` and ```suspend_always```. Let's start with those two first and then dive deeper into the `awaitable` type and see what it means. 

First we will adjust our previous example which we used for the `co_yield` example. In our coroutine the first `co_yield` is adjusted to `co_await suspend_always{};`, by doing this we will suspend the coroutine. This happens due that we pass in `suspend_always`, meaning that the coroutine must suspend. It does not pass any value into the coroutine, so `yield_value` is not called and our internal value is not updated. Let's run this and see what our log produces.

``` c++
sCoRoutineHandler coroutine()
{
    INFO << "Coroutine function start\n";
    co_await suspend_always{};

    INFO << "Coroutine function resumed\n";
    co_yield 2;
    
    co_return 5; // Coroutine
}
```

[INFO] Get coroutine return object\
[INFO] Constructing coroutine\
[INFO] Initial suspend\
[INFO] Coroutine function start\
[INFO] Coroutine suspended with value 0\
[INFO] Coroutine function resumed\
[INFO] Yield value 2\
[INFO] Coroutine suspended with value 2\
[INFO] Return value\
[INFO] Final suspend\
[INFO] Coroutine return value: 5\
[INFO] Destructor coroutine

As we can see in our log, the yield_value function is not called and the value is not set, therefore we now see a value of 0. So the coroutine is just suspended without giving any value. All other parts are still exactly the same as with the `co_yield` example. If we now do the same, but instead of ```co_await suspend_always{};``` we use ```co_await suspend_never{}``` we will see that the coroutine is not stopped by the ```co_await``` since it should never suspend. This means that the coroutine is suspended at the point of `co_yield 2` and we therefore only need one resume call before the coroutine ends.

What happens if we still run it with the two resume calls, a segmentation fault will come up. This comes due that the coroutine is already finished and we still try to resume it, which leads into undefined behaviour and in this case perticular into a segmentation fault.

[INFO] Get coroutine return object\
[INFO] Constructing coroutine\
[INFO] Initial suspend\
[INFO] Coroutine function start\
[INFO] Coroutine function resumed\
[INFO] Yield value 2\
[INFO] Coroutine suspended with value 2\
[INFO] Return value\
[INFO] Final suspend\
[INFO] Coroutine suspended with value 5\
make[1]: *** [Makefile:78: sim] Segmentation fault\
make: *** [Makefile:108: verilator] Error 2

We have now seen a basic part of the ```co_await``` with the already known `awaitable` types. As programmers we can also create our own `awaitable` and have the coroutine suspend on this. Just as the `promise_type` the `awaitable` exists out of a few functions which must be implemented, ```await_ready(), await_suspend(coroutine_handle) and await_resume()```.

``` C++
struct awaitable
{
    bool await_ready(){}

    // one of:
    void await_suspend(coroutine_handle<>){}
    bool await_suspend(coroutine_handle<>){}
    coroutine_handle<> await_suspend(coroutine_handle<>){}

    void await_resume(){}
}
```

`await_ready` is used to check if the coroutine should be suspended or that it can continue, in the context of this function the coroutine is not yet suspended. Best way to explain this is to use a example, let's say we need to read data from memory. Our coroutine initiates a read to the memory and suspends itself until the data has been fetched. But in the case that the data is cached, there is no need to be suspended. The coroutine however doesn't have any knowledge about this, it only handles the coroutine state. Well here we would need a special `awaitable` which has the knowledge about the reading the data from memory. The `await_ready` function can then check if the data which is needed is cached, if this is the case it can return `true` and the coroutine will not be suspended. When the data has to be fetched from memory it will return `false` and the coroutine is suspended.

Next to `await_ready` we have `await_suspend`. `await_suspend` will be called when the coroutine is set into the suspended state. It will receive the handle to the coroutine as parameter, with this we can store the handle and resume it later on. How to use `await_suspend` depends heavily on the implementation.

When the coroutine eventually gets resumed, the `await_resume` function will be called just before the resume point. This function will then return the result of the `co_await` expression.

The `awaitable` type is more or less a bridge between system logic and coroutine handle. The user can use system events to control the suspending and resuming of the coroutine, all by placing certain logic elements in the `awaitable` type. Let's look at a example and see what we mention above.

Let's add it into our example with some log statements and run the code to see how the execution works. 

``` C++
struct awaitable
{
    bool await_ready()
    {
        INFO << "Await ready\n";
        return false;
    }

    // one of:
    auto await_suspend(coroutine_handle<>)
    {
        INFO << "Await suspend\n";
    }

    auto await_resume()
    {
        INFO << "Await resume\n";
    }
}

sCoRoutineHandler coroutine()
{
    INFO << "Coroutine function start\n";
    co_await awaitable{};
    INFO << "Coroutine function resumed\n";    
    co_return 5; // Coroutine
}

int main(int argc, char** argv) 
{
    auto myCoroutineHandle = coroutine();
    
    INFO << "Resume coroutine \n";
    myCoroutineHandle.resume();    
    INFO << "Coroutine ended \n";
    return 0;    
}

```

[INFO] Get coroutine return object\
[INFO] Constructing coroutine\
[INFO] Initial suspend\
[INFO] Coroutine function start\
[INFO] Await ready\
[INFO] Await suspend\
[INFO] Resume coroutine\
[INFO] Await resume\
[INFO] Coroutine function resumed\
[INFO] Return value\
[INFO] Final suspend\
[INFO] Coroutine ended\
[INFO] Destructor coroutine

As we can see in this log, the coroutine is stopped with the await ready and await suspend function. When at that point the coroutine is resumed again, it goes through the await resume function and then resumes the coroutine itself. Which is exactly as we would expect from the code that we made. 

The nice thing about the `awaitable` type is that every class/structure can become one, we just need to add in the three functions. Those functions do have access to the internal state of the class/structure and can execute specific functionality. In the following part we will explore this a bit.

## Waiting on a different coroutine

Now we have seen how the coroutine mechanics work, let's bring it together and create a coroutine which waits on completion by a different coroutine. As we now know, we need a `awaitable` for the `co_await` operator. So by extending our sCoRoutineHandler with those three functions we make it an `awaitable`, let's try this and see what happens. 

``` c++
// Note that part of the code is removed
struct sCoRoutineHandler
{
    struct promise_type
    {
        int _myValue;
        coroutine_handle<> waitingCoroutine;

        suspend_always final_suspend() noexcept 
        { 
            INFO << "Final suspend\n";
            return {};
        }
    };

    void resume()
    {
        _handle.resume();
    }

    bool await_ready()
    {
        return false;
    }

    void await_suspend(std::coroutine_handle<> h)
    {
        _h.promise().waitingCoroutine = h;
    }

    auto await_resume()
    {
        return std::move(_h.promise()._myValue);
    }
};
```

In this case we want to always suspend when entering this coroutine, so therefore our `await_ready` function returns false. `await_suspend` passes the handle of the calling coroutine, which is the coroutine we want to resume when our current coroutine is finished. Since we want to resume the coroutine we also need to store it's handle, which we do inside the `await_suspend` function. The handle is stored within the `promise_type` so that the coroutine has access to it and can resume it. The `await_resume` will return the result of the operation, which in our case is stored inside `_myValue`. By returning this we return the result of the coroutine.

Now it's possible to suspend our coroutine waiting on a different coroutine to finish, but we still need to resume our coroutine when the coroutine is finished. As we have seen we also added `waitingCoroutine` in our `promise_type`, so we just have to start the waiting coroutine again. This can be done by changing the `final_suspend` function.

``` c++
auto final_suspend() noexcept 
{
    struct awaiter
    {
        coroutine_handle<> waitingCoroutine;
        bool await_ready() const noexcept { return false;}
        void await_resume() const noexcept {}

        coroutine_handle<> await_suspend(coroutine_handle<promise_type> h) const noexcept
        {
            return waitingCoroutine ? waitingCoroutine : std::noop_coroutine();
        }
    };

    return awaiter{waitingCoroutine}; 
}
```

Previously we have seen that `final_suspend` returns a `awaitable`, this `awaitable` determines what happens. `suspend_never` cleared the coroutine handle, where `suspend_always` suspended the coroutine and we had to manually clear it. In this case we return a special `awaitable` depending if there is any waiting coroutine. When we return a suspended coroutine it will automatically be resumed, which is exactly what we use in this case. In case there is no waiting coroutine we return noop_coroutine(), which translates to a special coroutine handle that has no-op `resume` and `destroy` method. If a coroutine suspends and returns the `noop_coroutine()` handle from `await_suspend`, this indicates that there is no more coroutine to resume and that the function should return to it's caller.

The last part now is to add this into our example code.

``` c++
sCoRoutineHandler secondCoroutine()
{
    INFO << "Second coroutine function start\n";
    co_yield 2;
    INFO << "Second coroutine function resume\n";
    co_return 10;
}

sCoRoutineHandler secondRoutineHandler = secondCoroutine();

sCoRoutineHandler coroutine()
{
    INFO << "Coroutine function start\n";
    int result = co_await secondRoutineHandler;
    INFO << "Coroutine function returned: " << result << "\n";    
    co_return 5; // Coroutine
}

int main(int argc, char** argv) 
{
    sCoRoutineHandler myCoroutineHandle = coroutine();

    secondRoutineHandler.resume();    

    INFO << "Coroutine return value: " << myCoroutineHandle.getValue() << '\n';
}

```

In this example we let our `coroutine` function waits on our `secondCoroutine`. Since our `secondCoroutine` needs to be resumed we have this as a global handle, this would normally be done by some specific logic, however this shows how it works. We start the first coroutine which starts the second coroutine, the second coroutine is suspended by the co_yield and we return back to `main`. In `main` we resume the second handle, which at that point returns 10 and resumes the first coroutine. This then prints the return value and returns itself. With this we end our example, print out the return value and the program exits. As we can see in the log:

[INFO] Coroutine function start\
[INFO] Second coroutine function start\
[INFO] Second coroutine function resume\
[INFO] Coroutine function returned: 10\
[INFO] Coroutine return value: 5

We now went through the basics of a C++ coroutine. There are a lot of possibilities with them and there is more to explore, however with those basics we can go forward and start implementing it into out Verilator related code, which we will start with in our next blog. For now this is the end of this blog, if you want to read more on coroutines check the following links which are very helpful.

https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await \
https://www.chiark.greenend.org.uk/~sgtatham/quasiblog/coroutines-c++20/#awaiters \
https://itnext.io/c-20-coroutines-complete-guide-7c3fc08db89d \
https://maharajan-ses.medium.com/c-coroutine-deep-dive-dd3fc0d57708 \
https://ggulgulia.medium.com/painless-c-coroutines-part-4-69117214bfdc \
https://en.cppreference.com/w/cpp/language/coroutines \
http://www.icce.rug.nl/documents/cplusplus/cplusplus24.html

Full code example:

``` c++

/**
 * @brief Template structure for a coroutine promise type
 * 
 * @tparam T 
 */
template <typename T>
struct sCoRoutineHandler
{
    struct promise_type;
    using handle_t = coroutine_handle<promise_type>;

    //MUST include nested object type promise_type
    struct promise_type
    {
        T _myValue;                     //!< This variable is accessible from the coroutine function
        std::exception_ptr _exception;  //!< Store exception pointer
        coroutine_handle<> waitingCoroutine;
        //bool _finished = false;         //!< Boolean to keep track if the coroutine has finished

        /**
         * @brief Get the return object
         * @details This function returns the created heap memory object 
         * of the coroutine
         * 
         * It must be included in the promise_type
         * 
         * @return sCoRoutineHandler<T>     Returns the heap memory object which is created
         */
        sCoRoutineHandler<T> get_return_object()
        {
            return {coroutine_handle<promise_type>::from_promise(*this)};
        }

        /**
         * @brief Initial suspend
         * @details This function is called at the beginning of a coroutine and determines
         * if the coroutine should suspend at the beginning and wait for a resume or that it
         * continues untill any of the co_expr is used.
         * 
         * When returning suspend_never the coroutine will run until a co_expr
         * When returning suspend_always the coroutine will directly be suspended
         * 
         * For testbenches we want the coroutine function to run until a co_expr and therefore 
         * we return suspend_never
         * 
         * @return suspend_never 
         */
        suspend_never initial_suspend() { return {}; }

        /**
         * @brief final suspend
         * @details This function is called at the end of a coroutine, it is used
         * to cleanup the coroutine. It always returns a awaitable type, where it has 
         * predefined behaviour for the standard awaitables.
         * 
         * When final_suspend returns suspend_never then the object is automatically destroyed
         * When final_suspend returns suspend_always then the object's state remains accessible
         * 
         * It's also possible to return a own awaitable, which is the case in this function,
         * it checks if there is a waiting coroutine. If this is the case we return the 
         * waiting coroutine, which is at this point resumed again. If there is no waiting
         * coroutine we return noop_coroutine, this is a special coroutine handle that has
         * no resume or destroy method, indicating to the coroutine that there is nothing 
         * to resume and it should return to it's caller.
         * 
         * Do note that .done() cannot be called if the object is already destroyed, therefore 
         * we can keep track of the final suspend state with the boolean _finished.
         * 
         * For testbenches we will keep the state, so that we can get the result of a coroutine 
         * at the end.
         * 
         * @return auto
         */
        auto final_suspend() noexcept 
        {
            struct awaiter
            {
                coroutine_handle<> waitingCoroutine;
                bool await_ready() const noexcept { return false;}
                void await_resume() const noexcept {}

                coroutine_handle<> await_suspend(coroutine_handle<promise_type> h) const noexcept
                {
                    return waitingCoroutine ? waitingCoroutine : std::noop_coroutine();
                }
            };
            
            return awaiter{waitingCoroutine}; 
        }

        /**
         * @brief unhandled exception
         * @details This function handles a exception
         * which occurs in the time of a coroutine. When this happens it stores
         * the exception into the structure which can then be gotten to check 
         * from outside of the promise type
         */
        void unhandled_exception()
        {
            _exception = std::current_exception();
        }

        /**
         * @brief yield_value
         * @details This function receives the value passed to co_yield.
         * It will receive the value and store it in our object itself so it
         * can be returned from the promise_type back to the user
         * 
         * This function must return either suspend_always or suspend_never,
         * where both mean the same as for initial_suspend.
         * 
         * 
         * @param val               Value passed by co_yield <value>
         * @return suspend_always   Always suspend the coroutine
         */
        
        suspend_always yield_value(T value)
        {
            _myValue = value;
            return {};
        }

        /**
         * @brief return value
         * @details This function is called just before final suspend and is used 
         * to return a value from the user. A promise type must hold either
         * void return_value() or void return_void() depending on how the coroutine 
         * is used. 
         * 
         * return_value() is used with "co_return e"
         * return_void() is used with "co_return"
         * 
         * For our testbench we will use return_value so we can get test results
         * from our test. (This can always be ignored)
         * 
         * CAUTION: falling of the end without a return_void() or return_value() 
         * causes undefined behaviour!!!!
         * 
         * @param T The value which is returned of the coroutine
         */
        void return_value(T value) 
        {
            _myValue = value;
        }
    };

    handle_t _h; //!< Handle to coroutine

    /**
     * @brief Construct a new coroutine handle
     * @details Constructor of the coroutine handler
     * 
     * @param h 
     */
    sCoRoutineHandler(handle_t h) : _h(h) {};

    /**
     * @brief Destroy the coroutine object
     * @details This function destroys the object
     * 
     * Destroy the object if there is a object
     * 
     */
    ~sCoRoutineHandler()
    {
        if(_h)
        {
            _h.destroy();
        }
    }

    /**
     * @brief Check if the coroutine is done
     * @details
     * 
     * This functions checks if the coroutine has finished, due that 
     * final_suspend returns suspend_always we cannot call .done().
     * Therefore the _finished is introduced in the promise type and 
     * we can check accordingly.
     * 
     * @return true 
     * @return false 
     */
    explicit operator bool() 
    {
        return _h.done(); // Use this when final_suspend returns suspend_never
        //return _h.promise()._finished; // Use this when final_suspend returns suspend_always
    }

    /**
     * @brief Resume the coroutine at the last suspended point
     * 
     */
    void resume()
    {
        _h.resume();
    }

    /**
     * @brief Get the Result 
     * @details This function returns the value gotten from the 
     * promise type. Or easier said, it returns the result of
     * the coroutine. This can either be the result from 
     * co_yield as well as co_return.
     * 
     * @attention When final_suspend returns suspend_always we cannot get 
     * the value of the co_return expr. This value will then be changed, due
     * that the promise is already deleted.
     * 
     * @return T    Returns coroutine value or false when there is no value
     */
    T getValue()
    {
        if (_h) 
        {
            return std::move(_h.promise()._myValue);
        }
        else
        {
            return false;
        }
    }

    /**
     * @brief The await ready function for the awaitable type
     * 
     * @return false    Always return false to suspend the coroutine
     */
    bool await_ready()
    {
        return false;
    }

    /**
     * @brief The await suspend function for the awaitable type
     * @details This function suspends the coroutine and implements
     * the needed logic. It stores the passed coroutine handle into the
     * waitingCoroutine handle of our current promise type.
     * 
     * This can then be used to resume the coroutine that called co_await
     * 
     * @param h     The coroutine handle which calls the co_await
     */
    void await_suspend(std::coroutine_handle<> h)
    {
        _h.promise().waitingCoroutine = h;
    }

    /**
     * @brief The await resume function for the awaitable type
     * @details This function is called when the coroutine is resumed again
     * It will return the result of the coroutine, which is placed in 
     * the promise type.
     * 
     * @return auto     Returns the result of the coroutine
     */
    auto await_resume()
    {
        return std::move(_h.promise()._myValue);
    }
};


sCoRoutineHandler<int> secondCoroutine()
{
    INFO << "Second coroutine function start\n";
    co_yield 2;
    INFO << "Second coroutine function resume\n";
    co_return 10;
}

sCoRoutineHandler secondRoutineHandler = secondCoroutine();

sCoRoutineHandler<int> coroutine()
{
    INFO << "Coroutine function start\n";
    int result = co_await secondRoutineHandler;
    INFO << "Coroutine function returned: " << result << "\n";    
    co_return 5; // Coroutine
}

int main(int argc, char** argv) 
{
    if(setupProgramOptions(argc, argv))
    {
        return 0;
    }

    setupLogger();

    sCoRoutineHandler myCoroutineHandle = coroutine();

    secondRoutineHandler.resume();    

    INFO << "Coroutine return value: " << myCoroutineHandle.getValue() << '\n';
}

```