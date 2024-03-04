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

As we identified previously it's difficult to control multiple signals simultaneously. In system Verilog many things can run in parallel, due that everything is clocked and handled according to this clock. In our general testbench we also have a very nice clock, so it would be very cool if we could mimic this parallelization in our testbench. Well it's mentioned earlier, threads. Using threads would be an option, creating a small thread for certain functionality and kill the thread when it ends. This would suit our APB read task in system Verilog very well, however in large designs we would then need to create and control many threads. That's not really what we want, making our design way to complex and difficult to handle. However the threading idea is not so bad, preferred a small subroutine which could be paused and continued on a clock edge or on finishing a action. C++ 20 has introduced something like this, coroutines. A coroutine can be described as "functions whose execution you can pause", this means we can write the functions almost the same as in system Verilog.

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

Coroutines are functions which can be suspended and will resume their execution at the exact same point as where it was suspended. All the needed data is not stored on the stack but in a specific heap memory element. When the execution is stopped it will return to the caller, it can from that point be started from multiple different locations or events. With this it is possible to mimic multithreading in a single threaded application. Using coroutines to drive the Verilator context is very helpful since Verilator is cycle based, so the different drivers can in that way work on the same cycle as that the Verilator driver works. Coroutines are newly introduced in C++20.

A lot of small code snippets are shown below, those are for illustration purposes. They don't work on their own, usually they have a few things missing. Let's first start with how do we make a function a coroutine, this happens when one of the following definitions is used, ```co_await```, ```co_yield``` or ```co_return```. However this makes the function a coroutine, but for being able to have a coroutine we need a struct called ```promise_type```, this struct must hold a number of functions for the compiler to generate a coroutine for this. Let's look below and see the most simple example of a coroutine.

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

sCoRoutineHandler myCoroutine()
{
    co_return; // Making the function a coroutine
}

int main(int argc, char** argv) 
{
    sCoRoutineHandler myTask = myCoroutine();
    return 0;
}

```

We have now seen one of the main keywords to create a coroutine, which is the ```co_return```, it is used to complete execution of coroutine and return a value. Let's check how this part works and extend it to return a value and make sure that we cleanup our code correctly, while also explaining what the corresponding functions do. As first, we will expand our simple example and add some debug statements so we can follow the execution of the program. 

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

[INFO] Get coroutine return object
[INFO] Constructing coroutine
[INFO] Initial suspend
[INFO] Coroutine function start
[INFO] Return void
[INFO] Final suspend
[INFO] Destructor coroutine

At first the function ```get_return_object``` is called, this will return the newly allocated heap memory object for the coroutine. Following this the constructor is called, with as parameter a handle to the coroutine itself, which we store in our structure, giving us access to the coroutine. The object is now created and ```initial_suspend``` is called, in our current design it returns ```suspend_never``` which means that our cororoutine function is executed until a ```co_await```, ```co_yield``` or ```co_return``` is used. The other case would be to return ```suspend_always```, suspending the coroutine function immediately. As we noted just before, after the initial suspend with ```suspend_never``` the coroutine function itself is called, visible by the coroutine function start statement. Since our function immediately returns with the ```co_return``` statement, it directly goes to the ```void return_void``` function, which is mandatory to have, without it's possible to have undefined behaviour. The last call of the coroutine goes to ```final_suspend```, this is the function for cleaning up the coroutine. In this example it has ```suspend_never``` as return parameter, the object is then automatically destroyed, where the state is not accessible anymore. Calling the coroutine after this point will lead into a segmentation fault. When the object's state has to be accessible after the coroutine finishes ```suspend_always``` must be used a return parameter. 

Let's update ```initial_suspend``` and change the return parameter to ```suspend_always``` and re-run our example code. Since the coroutine is directly stopped we would expect that we don't see the coroutine function start and the corresponding function calls to close of the coroutine. Which indeed is true as we can see in the log:
[INFO] Get coroutine return object
[INFO] Constructing coroutine
[INFO] Initial suspend
[INFO] Destructor coroutine

Almost all function have now been explained and checked in which order they are called and executed, except the ```unhandled_exception```. Which is only called when an exception occurs within the context of a coroutine. So if the execution of the coroutine fails it will be called and the exception can stored/handled as by used wishes. It must be noted that using the coroutine after this can result in undefined behaviour and should be checked very carefully.

Now let's return a value from our coroutine, we can do this by calling ```co_return expr```. But we also should adjust our ```promise_type``` and change the ```void return_void()``` function to ```void return_value(expr)```. It's a bit confusing but our function returns void, since it can't directly return the value to the caller. So we need to adjust the function accordingly and store the value in our structure. After the coroutine has then finished we can retrieve the value from it. Let's see how this works in the code.

``` C++

// In our promise_type
void return_value(int value) 
{
    INFO << "Return value\n";
    _myReturnValue = value;
}

// In the sCoroutineHandler
int getReturnValue()
{
    return _handle.promise()._myReturnValue;
}

// In main
sCoRoutineHandler myCoroutine()
{
    INFO << "Coroutine function start\n";
    co_return 5; // Coroutine
}

auto myTask = myCoroutine();
INFO << "Coroutine return value: " << myTask.getReturnValue() << '\n';

```

As we run with the following code the following trace will be visible:
[INFO] Get coroutine return object
[INFO] Constructing coroutine
[INFO] Initial suspend
[INFO] Coroutine function start
[INFO] Return value
[INFO] Final suspend
[INFO] Coroutine return value: 5
[INFO] Destructor coroutine

Visible here is that the return void is now replaced with return value and that we get the value from our coroutine and print it out. Do note that this must happen before the handler of the coroutine is destroyed.

## co_yield

Let's explore the second coroutine expression ```co_yield expr```, which is used to suspend execution returning a value. This works in the same way as with our ```co_return expr``` example, the only difference here is that our coroutine can still resume. Meaning that we still need to request the value from our co_routine when it is suspended. As for the co_yield we also need to add another special function into our ```promise_type```, ```suspend_always yield_value(expr)```. When a ```co_yield expr``` is executed we will automatically end up in this function and can at that point process the data. The ```yield_value``` function must return ```suspend_always``` or ```suspend_never```, where it means the same as previously described. ```suspend_always``` will stop the coroutine and wait until it is resumed, where ```suspend_never``` will let the coroutine continue until the next coroutine expression. Let's try ```co_yield``` out with a simple example.

``` C++
// The following is added in the promise_type
suspend_always yield_value(int value)
{
    INFO << "Yield value " << value << '\n';
    _myValue = value;
    return {};
}

// Our coroutine is adjusted to the following:
sCoRoutineHandler myCoroutine()
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

Running this function we would expect that our coroutine is started and then yields with the value of 1, where our main routine takes over and prints the value of the first yield. Following this the coroutine is resumed again and it will suspend with the value of 2. At last main takes over again and prints out the value, resuming the coroutine for the last time and printing the return value. This is also our first example which shows the power and especially the readines of a coroutine.

Trace of the simple co_yield example:
[INFO] Get coroutine return object
[INFO] Constructing coroutine
[INFO] Initial suspend
[INFO] Coroutine function start
[INFO] Yield value 1
[INFO] Coroutine suspended with value 1
[INFO] Coroutine function resumed
[INFO] Yield value 2
[INFO] Coroutine suspended with value 2
[INFO] Return value
[INFO] Final suspend
[INFO] Coroutine return value: 5
[INFO] Destructor coroutine

## co_await

```co_await``` is used to suspend execution untill the coroutine is resumed. As parameter it takes a awaitable type, where we already have seen two of those, ```suspend_never``` and ```suspend_always```. Let's try out with those two first and then dive deeper into the awaitables and see what it means. In our function we change ```co_yield 1;``` to ```co_await suspend_always{};```, which will give us the following log:

[INFO] Get coroutine return object
[INFO] Constructing coroutine
[INFO] Initial suspend
[INFO] Coroutine function start
[INFO] Coroutine suspended with value 0
[INFO] Coroutine function resumed
[INFO] Yield value 2
[INFO] Coroutine suspended with value 2
[INFO] Return value
[INFO] Final suspend
[INFO] Coroutine return value: 5
[INFO] Destructor coroutine

As we can see in our log, the yield_value function is not called and the value is not set, therefore we now see a value of 0. So the coroutine is just paused without giving any value back. If we now do the same, but instead of ```co_await suspend_always{};``` we use ```co_await suspend_never{}``` we will see that the coroutine is not stopped by the ```co_await``` since it should never suspend. 

When we now run this, a segmentation fault will come up. This comes due that we delete the coroutine before it has ended, however why would we want to call ```co_await``` when we don't want to suspend, currently there is no reason for this.
[INFO] Get coroutine return object
[INFO] Constructing coroutine
[INFO] Initial suspend
[INFO] Coroutine function start
[INFO] Coroutine function resumed
[INFO] Yield value 2
[INFO] Coroutine suspended with value 2
[INFO] Return value
[INFO] Final suspend
[INFO] Coroutine suspended with value 5
make[1]: *** [Makefile:78: sim] Segmentation fault
make: *** [Makefile:108: verilator] Error 2

We have now seen a basic part of the ```co_await``` with the already known awaitable types. However there is more behind the awaitable, let's explore this and see how it works. 
The awaitable type is about the same as our ```promise_type``` which we have seen previously for our coroutine. Just as the ```promise_type```, the awaitable type is also a special interface just for coroutines. It exists out of three functions, ```await_ready(), await_suspend(coroutine_handle) and await_resume()```.

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

Let's see what those functions exactly mean and what they do. When ```co_await``` is called it will first call the ```await_ready()``` function of the awaitable type. This is the place where the coroutine can determine if it suspends or not. This is done by the return type, when ```true``` is returned it means that the coroutine can continue without suspending. When ```false``` is returned the coroutine should be suspended. A example for this is cached data, inside the ```await_ready()``` it could be checked if certain data is already cached, when this is the case there is no need to suspend the function, it can be continued directly.

When ```await_ready()``` determineds that the coroutine must suspend, the compiler will kick in and suspend the coroutine. Following this it calls the ```await_suspend``` with the coroutine handle. Within this function the coroutine is suspended and we can now kick off any related operation. The return value determines the behaviour of the function, when a coroutine handle is returned it will automatically resume that coroutine, this can be the own coroutine handle but also a handle to a different coroutine. When nothing is returned the coroutine will be kept in the suspended state. It's also possible to do the same as for the ```await_ready()``` function, return a boolean, where ```false``` will resume the coroutine and ```true``` will keep the coroutine suspended. Last function is the ```await_resume()```, which should return the result of the awaitable, what the exact return type is depends on the implementation. Let's take a look into a simple example and follow the trace of execution.

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

sCoRoutineHandler myCoroutine()
{
    INFO << "Coroutine function start\n";
    co_await awaitable{};
    INFO << "Coroutine function resumed\n";    
    co_return 5; // Coroutine
}

int main(int argc, char** argv) 
{
    auto myTask = myCoroutine();
    
    INFO << "Resume coroutine \n";
    myTask.resume();    
    INFO << "Coroutine ended \n";
    return 0;    
}

```

As we run this code we will see the following output in our terminal (keep in mind that we still have some output in our coroutine itself):

[INFO] Get coroutine return object
[INFO] Constructing coroutine
[INFO] Initial suspend
[INFO] Coroutine function start
[INFO] Await ready
[INFO] Await suspend
[INFO] Resume coroutine
[INFO] Await resume
[INFO] Coroutine function resumed
[INFO] Return value
[INFO] Final suspend
[INFO] Coroutine ended
[INFO] Destructor coroutine

As we can see in this log, the coroutine is stopped with the await ready and await suspend function. When at that point the coroutine is resumed again, it goes through the await resume function and then resumes the coroutine itself. Which is exactly as we would expect from the code that we made. The awaitable type is very versatile, but has to be tailored to suit the implementation.

We now went through the basics of a C++ coroutine. There are a lot of possibilities with them and there is more to explore, however with those basics we can go forward and start implementing it into out Verilator related code, which we will start with in our next blog. For now this is the end of this blog, if you want to read more on coroutines check the following links which are very helpful.

https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await
https://www.chiark.greenend.org.uk/~sgtatham/quasiblog/coroutines-c++20/#awaiters
https://itnext.io/c-20-coroutines-complete-guide-7c3fc08db89d
https://maharajan-ses.medium.com/c-coroutine-deep-dive-dd3fc0d57708
https://ggulgulia.medium.com/painless-c-coroutines-part-4-69117214bfdc
https://en.cppreference.com/w/cpp/language/coroutines
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
        T _myValue;                       //!< This variable is accessible from the coroutine function
        std::exception_ptr _exception;  //!< Store exception pointer

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
         * to cleanup the coroutine. Depending on it's return type it will either 
         * automatically destroy the coroutine or the user must handle the destroy by itself.
         * 
         * When final_suspend returns suspend_never then the object is automatically destroyed
         * When final_suspend returns suspend_always then the object's state remains accessible
         * 
         * Do note that .done() cannot be called if the object is already destroyed
         * 
         * For testbenches destroying the object on completion is the correct approach
         * 
         * @return suspend_never
         */
        suspend_never final_suspend() noexcept { return {}; }

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
     * Since our final_suspend returns suspend_never the object is automatically destroyed, 
     * so we don't have to do it
     * 
     */
    ~sCoRoutineHandler()
    {

    }

    /**
     * @brief Check if the coroutine is done
     * 
     * @return true 
     * @return false 
     */
    explicit operator bool() 
    {
        return _h.done();
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
     * @return T    Returns coroutine value or false when there is no value
     */
    T getResult()
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
};

struct awaitable
{
    bool await_ready()
    {
        std::cout << "Await ready\n";
        return false;
    }

    // one of:
    auto await_suspend(coroutine_handle<> handle)
    {
        // Perform any waiting action
        std::cout << "Await suspend\n";
    }
    // bool await_suspend(coroutine_handle<>){}
    // coroutine_handle<> await_suspend(coroutine_handle<>){}

    auto await_resume()
    {
        std::cout << "Await resume\n";
    }
};

sCoRoutineHandler<int> myCoroutine()
{
    std::cout << "Coroutine function start\n";
    co_await awaitable{};
    //int waitedValue = 
    co_yield 3;
    //INFO << "Waited value: " << waitedValue << "\n";
    std::cout << "Coroutine function resumed\n";    
    co_return 5; // Coroutine
}

int main(int argc, char** argv) 
{
    if(setupProgramOptions(argc, argv))
    {
        return 0;
    }

    setupLogger();

    auto myTask = myCoroutine();

    std::cout << "Resume coroutine \n";
    myTask.resume();

    std::cout << "Coroutine suspended with value " << myTask.getValue() << '\n';
    myTask.resume(); 
    
    std::cout << "Coroutine ended with value" <<  myTask.getResult() << "\n";
}

```