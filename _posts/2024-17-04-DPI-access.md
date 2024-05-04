---
title: DPI access
date: 2024-04-17
author: bschouteten
---

# Scratchpad test
In our previous blog we created our first test for the APB uart16550, this was a very basic test. A value was written and then read back from the scratchpad register, as mentioned this register doesn't do anything functional within the UART itself, but gives the programmer an option to store a value. In our test we did a APB write to this register and then read the value from the register back, which seems to be a valid test. However how do we know that the corresponding value is really written into the right register, it is possible that our mapping from the APB register is wrong but it still reads the right value back. But currently there is no possibility to directly read the register value in our DUT, luckily for us SystemVerilog has something for this, Direct Programming Interface (DPI).

# DPI

Direct Programming Interface (DPI) is a feature that allows users to interface between SystemVerilog and a foreign programming language, mainly C, C++ or SystemC. It consists out of two layers, a SystemVerilog layer and a foreign language layer, both are isolated from each other. It allows designers to easily call C like functions from SystemVerilog and to export SystemVerilog functions, so that they can be called from outside. Verilator supports DPI import and export statements, the "DPI-C is supported.

Import statements are used for functions which are implemented in a foreign language and can be used in SystemVerilog by importing. In the scope of SystemVerilog this is called a imported function. Export statements are the other way around, they are used to call SystemVerilog functions from the foreign language. 

Below is a import function in SystemVerilog, here a function called `add` is instantiated and must be defined in a foreign language to be executed. Within SystemVerilog this can then be called like any other function. 
``` SystemVerilog
import "DPI-C" function int add (input int a, input int b);

initial begin
   $display("%x + %x = %x", 1, 2, add(1,2));
endtask
``` 

When using Verilator a extern function declaration will be created and added into the Vour__Dpi.h file (in our case `#include "Vapb_uart16550__Dpi.h"`), the user then has to define the function itself.

``` C++
// Declaration in Vour__Dpi.h
extern int add(int a, int b);

// Function definition
#include "Vour__Dpi.h"
int add(int a, int b) { return a+b; }
```

Export functions work in the same way, in SystemVerilog the function itself is defined with the export specifier. The same function is also declared in the header file as extern, but in this case there is no need to declare the function, it can just be called from the foreign language.

``` SystemVerilog
export "DPI-C" task publicSetBool;

task publicSetBool;
   input bit in_bool;
   var_bool = in_bool;
endtask
```

``` C++
// Declaration in Vour__Dpi.h
extern void publicSetBool(svBit in_bool);

// Function call
#include "Vour__Dpi.h"
publicSetBool(value);
```

There is one special thing if the DPI task or function accesses a net within the SystemVerilog design, at this moment it requires a scope to be set. see (verilator connecting)[https://verilator.org/guide/latest/connecting.html] for more reference.


# Adding DPI to our UART16550 design

As mentioned before, we need to have access into our internal state of the design, with this we can better verify our design. In the first test we wrote a value into our scratchpad register and then read the value back. Comparing the written and the read back value gives us the result at that moment. However what if our design didn't store it properly but still returns it as expected, at this moment our test was succesfull but the design is failing. Since the values written to any of the UART16550 register are stored within the design we can use DPI functions to peek and poke those registers. With the peek function we want to peek into our design and see what the actual value is of a certain register, so we pass in the register to read and get the register value back. Our poke function is the other way around, with this we want to write a specific value into a register, for this we pass in which register and the value to set. 

So let's check how this looks like in our SystemVerilog design.

``` SystemVerilog
    /**
    * @brief DPI function to peek CSRs
    */
    export "DPI-C" function uart16550_peek;
    function byte uart16550_peek(input byte r);
    begin
        byte result;

        case(r)
            {4'h0,1'b0, RBR_ADR}: result = rx_q_i.d; //read only register
            {4'h0,1'b0, IER_ADR}: result = csr.ier;
            {4'h0,1'b0, IIR_ADR}: result = csr.iir;  //read only register
            {4'h1,1'b0, FCR_ADR}: result = csr.fcr;  //write only register
            {4'h0,1'b0, LCR_ADR}: result = csr.lcr;
            {4'h0,1'b0, MCR_ADR}: result = csr.mcr;
            {4'h0,1'b0, LSR_ADR}: result = csr.lsr;
            {4'h0,1'b0, MSR_ADR}: result = csr.msr;
            {4'h0,1'b0, SCR_ADR}: result = csr.scr;

            {4'h2,1'b0, DLL_ADR}: result = dl.dll;
            {4'h2,1'b0, DLM_ADR}: result = dl.dlm;
            default             : result = 0;
        endcase

        return result;
    end
    endfunction


    /**
    * @brief DPI task to poke CSRs
    */
    export "DPI-C" task uart16550_poke;
    task uart16550_poke(input byte r, input byte d);
    begin
        case(r)
            {4'h0,1'b0, IER_ADR}: begin force csr.ier = d; release csr.ier; end
            {4'h0,1'b0, IIR_ADR}: begin force csr.iir = d; release csr.iir; end
            {4'h1,1'b0, FCR_ADR}:       force csr.fcr = d;
            {4'h0,1'b0, LCR_ADR}: begin force csr.lcr = d; release csr.lcr; end
            {4'h0,1'b0, MCR_ADR}: begin force csr.mcr = d; release csr.mcr; end
            {4'h0,1'b0, LSR_ADR}:       force csr.lsr = d;
            {4'h0,1'b0, MSR_ADR}:       force csr.msr = d;
            {4'h0,1'b0, SCR_ADR}: begin force csr.scr = d; release csr.scr; end

            {4'h2,1'b0, DLL_ADR}: begin force dl.dll  = d; release dl.dll;  end
            {4'h2,1'b0, DLM_ADR}: begin force dl.dlm  = d; release dl.dlm;  end
            default             : ;                                              //some registers are simply not pokeable
        endcase
    end
    endtask
```

In the current design there is no need to have any import functions, so the last thing needed in our design is the scope. For this we use the same code as in the example, the function is introduced in main.cpp since it is a global function. We first get the scope of the simulation and following this directly set it, following this part we can then use our DPI created functions. The RTL design will call this getScope function, so there is no need for us to do more with it.

``` c++
void getScope()
{
    svScope scope = svGetScope();
    const char* scopeName = svGetNameFromScope(scope);

    INFO << "ScopeName:" << scopeName << std::endl;
}
```

## Using our create peek and poke function

As starting point we take our previous scratchpad test and expand it. First we extend our write test, after we write the value with our APB transaction we check if the register is written properly by peeking inside, as we can see below. Note that the corresponding peek and poke functions are wrapped within a specific function for easier reading of the code. 

See the end of the blog for the full test function

``` c++
// Write the random value into the scratchpad register
// SCR is the scratchpad address
co_await apbMaster->write(SCR, &writeValue);

peekval = peek(PEEK_SCR);

if (peekval != writeValue)
{
    APPEND << "Failed: Written:" << std::hex << unsigned(writeValue) << " peeked:" << std::hex << unsigned(peekval) << "\n";
    result = false;
}

```

Following the write check we also need to extend our read part. We can re-use what we had before, just read back the value of the register with a APB read. Now to validate that our APB read is functioning as expected we poke the value in the scratchpad register and read it back. To do this we invert the random generated value, poke it in the register and then read it through our APB interface.

``` C++
writeValue = ~writeValue & 0xff;
poke(PEEK_SCR, writeValue);
co_await apbMaster->read(SCR, &readValue);

if (readValue != writeValue)
{
    APPEND << "poked:" << std::hex << unsigned(writeValue) << " received:" << std::hex << unsigned(readValue) << "\n";
    result = false;
}
```

Our scratchpad test is now adjusted to use DPI and APB transactions. With this we not only verified that the scratchpad is functional but also that our APB is working as expected. This ends our current blog, in the next blog we will look how to further test our UART16550 design. Where the first thing to add is the reading of the RX and TX line of the UART.


``` C++
sCoRoutineHandler<bool> cAPBUart16550TestBench::scratchpadTest (size_t runs)
{
    uint8_t writeValue, readValue, peekval;
    bool result = true;
    INFO << "Start scratchpad test\n";
    co_await generateReset();

    waitPosEdge(pclk);

    for (size_t i = 0; (i < runs) && (result); i++)
    {
        INFO << "Run: " << i << " ";

        writeValue = std::rand();   // Get a random value

        // Write the random value into the scratchpad register
        // SCR is the scratchpad address
        co_await apbMaster->write(SCR, &writeValue);

        peekval = peek(PEEK_SCR);

        if (peekval != writeValue)
        {
            APPEND << "Failed: Written:" << std::hex << unsigned(writeValue) << " peeked:" << std::hex << unsigned(peekval) << "\n";
            result = false;
        }

        // Directly read the value back from the scratchpad register
        co_await apbMaster->read(SCR, &readValue);

        if(writeValue != readValue)
        {
            //values are not the same, test has failed
            result = false;
            APPEND << "Failed: Expected: " << std::hex << unsigned(writeValue) << 
                                    " got " << std::hex << unsigned(readValue) << "\n";
        }

        writeValue = ~writeValue & 0xff;
        poke(PEEK_SCR, writeValue);
        co_await apbMaster->read(SCR, &readValue);
        release(SCR);

        if (readValue != writeValue)
        {
            APPEND << "poked:" << std::hex << unsigned(writeValue) << " received:" << std::hex << unsigned(readValue) << "\n";
            result = false;
        }

        if(result = true)
        {
            APPEND << "ok \n";
        }
    }
    
    INFO << "Scratchpad test ended\n";

    co_return result;
}
```