---
title: Generating signals
date: 2024-01-24
author: bschouteten
---

# Generating signals within Verilator

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

Creating something like this with C++ in Verilator is a bit more difficult, our testbench will be looping to tick forward, as we have seen in our previous designs. It's just a plain for loop and will loop for the given number of iterations. Let's try to write some C++ code which would generate the reset pattern as written above in the system Verilog design. Our starting point will be the ```run(int numCycles)``` in the tb_apb_uart16550 class. Following our system Verilog design we would need to set the reset signal low from the fifth negedge clock edge until the tenth negative clock edge. Now let's see how this theoretically transforms to our single 100MHz clock design. We start off with the clock signal low, which is our first tick or i = 0. So with i = 1 the clock will go up, i = 2 we have our first negative edge, this will then go on and on. So every even number the clock will be negative, so after 5 * 2 = 10 ticks we would have 5 negative edges and set the reset signal low, going high again after 10 * 2 = 20 ticks. Following this logic we could extend our run routine as below and generate the same reset signal as our system Verilog code. 

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

Alright so with this function we should now be able to read from our design through the APB bus, however it's now very difficult to control the clocking of all the functions. This is becoming a drawback in this way, also the state machine is not very nice. For now it's not to difficult to understand and to use, however if we get more and more difficult designs  it's going to be a nightmare. Also sending multiple signals to our design is impossible, we can only read from the APB and no other signals can be read or controlled. To support this we need to extend our APB read, making it more and more complex. Thinking ahead it seems that this approach is not really working properly, so let's think what our solution could be.

# Coroutines

As we identified previously it's difficult to control multiple signals simultaneously. In system Verilog many things can run in parallel, due that everything is clocked and handles according to this clock. In our general testbench we also have a very nice clock, so it would be very cool if we could mimic this parallelization in our testbench. Well it's mentioned earlier, threads. Using threads would be an option, creating a small thread for certain functionality and kill the thread when it ends. This would suit our APB read task in system Verilog very well, however in large designs we would then need to create and control many threads. That's not really what we want, making our design way to complex and difficult to handle. However the threading idea is not so bad, preferred a small subroutine which could be paused and continued on a clock edge or on finishing a action. C++ 20 has introduced something like this, coroutines. A coroutine can be described as "functions whose execution you can pause", this means we can write the functions almost the same as in system Verilog.

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
