---
title: Generating time within Verilator
date: 2024-01-23
author: bschouteten
---

# Generating time within Verilator

Welcome back for the third blog in this series. In the last blog we added logging, system options and created a universal testbench. For this blog we will extend our universal testbench and start generating a useful clock signal.

Until now we just toggled the clock pin of our design, without thinking further then what is behind this. So let's think further, within this design we have a single clock based design. However looking ahead into our SoC simulation, which is a design that uses more clocks. Since Verilator is a cycle based simulator it's possible to sync the cycles with a clock signal. Or to say it in different words, create a time instance for our simulation. 

# Timing within Verilator

All this time management sounds very strange within a cycle based simulator. Verilator itself doesn't have any knowledge about time, it just checks cycle by cycle endlessly. However we can create a timing with those cycles, since we can count the number of times the clocked has ticked. If you check the first waveform we generated, it's possible to see that this waveform did have some timing, here everything was written in picoseconds. Every call we made to ``` _trace->dump() ``` we passed the value of the for loop into the dump. The number of cycles that our simulator ran is translated into a time in the waveform, meaning we can also do this within our code. System Verilog normally runs with a time unit and time precision , for example time unit of 1ns and time precision  of 100ps. For Verilator this is by default set to 1ps/1ps to be inline with SystemC. However it's possible to adjust this timescale by passing the ``` --timescale <time unit>/<time precision > ``` to Verilator or by having it within the top level design file of the system. For our simulation this is very helpfull, since we can extract the time unit and the time precision  and use this as a reference in our design. 

The first thought which pops up in this case, if we now tick our clock and then increase the time according to the set time precision . This would be an easy way to keep track of our simulation time and derive the clocks for this. So if we tick 5 times, we have advanced time 5 * time precision . Let's say time precision  has the default setting of 1ps, so 5*1 = 5ps. In this way we can continue counting and counting and we just evaluate the design every time. This is fully correct, this would work perfectly and be easy to understand and easy to read. However let's think about the performance of this all, take for example a 100MHz clock, which has a full period of 10ns, switching states every 5ns. We would need to evaluate 5000ps to just toggle the state of the clock once. This solution does so many evaluations which are unnecessary, remember we have synchronized designs which only do something on a clock edge. Of course we could say at this moment, change the time precision  and set it to 1ns, then you only need 5 ticks to create a 100MHz clock. This is correct and is also more then viable, however when we have a multiple clock design this doesn't work. For example, we have a design with a clock of 100MHz, 125MHz and 133MHz. The 100MHz has a period of 10ns, 125MHz is 8ns and 133MHz is 7.5ns, now it becomes very problematic. Our 1ns tick can not get close to 7.5ns and we need to adjust the precision to get an accurate clock, which brings us back to our original performance problem.

Hmmm... How can we solve this. We know that we only need to evaluate the design at the moment any of the clock changes. So everything what happens between the clock edges is not interesting, meaning we can skip it fully. So if we now instead of counting every cycle a certain amount of time, we just check which clock is the next to toggle and then toggle this clock, evaluate our design and then repeat this process over and over. This would solve our initial problem of having many ticks/evaluations, but how can we at this point keep track of our time? Well as long as every clock knows when to toggle, we can subtract the difference per each clock. So let's take an example, we have a clock of 100MHz and 125MHz. The 100MHz has a period of 10ns, so the first clock edge will come at 5ns. 125MHz clock has a period of 8ns, this first edge will come at 4ns. So from the beginning we check, which clock is first, well this is the 125MHz clock, since it changes at 4ns (4 is lower then 5). But since we now have advanced 4ns in time, we must subtract this 4ns from the 100MHz, so the next edge for the 100MHz clock will come in 5-4 = 1ns. When we now again check what is the next clock that should toggle, the 125MHz clock will say over 4ns, where the 100MHz clock is in 1ns. So we toggle the 100MHz clock at this moment and set it to toggle again in 5ns, the 4ns for the 125MHz clock is then subtracted with 1ns, making it 3ns for the next clock edge. In this way we keep proceeding our internal time and the clocks will then still toggle on the right intervals. Internally we keep track of the elapsed time and can output information or pass this time to our waveform when enabled, syncing everything nicely. Now let's proceed and set this over into a code design and try it out.

# Expanding our design 

To control some sort of timing, it's helpful to have one instance which will control the timing. From this main instance we can then control one or multiple clocks, let's call this our clock manager. This clock manager will then control the "time" within our simulation context, but also control all the clocks in the design. Every clock is basically the same, it has a low and high period which will be used indefinitely, our testbench then needs to drive the corresponding pin with the right period. With this knowledge we can extend our current general testbench to simulate clocks, see the new UML diagram.

## Simulation time

As visible within our UML diagram, the clockmanager keeps track of the time with the simtime_t object. The cSimtime_t class is designed to be a single class which has all the knowledge on how to convert timings. So anyone who uses this class can call any of the helper functions to get the expected value without having to do any conversions. 

It's created as a template class and can be used with multiple different base types, making it universal in usage. The realtime is kept as a private variable, where the time can be gotten with the functions provided within the class. Times can be added and subtracted directly on the variable itself due to the usage of overloaded operators, see [C++ operators](https://www.geeksforgeeks.org/operator-overloading-cpp/) for more information about this.

``` C++
template <typename T> class cSimtime_t
    {
        private:
        T _realtime;  //realtime in seconds


        public:

        constexpr static T max() { return std::numeric_limits<T>::max(); };
 
        constexpr static T seconds_per_minute = 60.0;
        constexpr static T minutes_per_hour   = 60.0;
        constexpr static T seconds_per_hour   = seconds_per_minute * minutes_per_hour;
        constexpr static T hours_per_day      = 24.0;
        constexpr static T minutes_per_day    = hours_per_day * minutes_per_hour;
        constexpr static T seconds_per_day    = minutes_per_day * seconds_per_minute;
        constexpr static T days_per_year      = 365.0;
        constexpr static T seconds_per_year   = days_per_year * seconds_per_day;
        constexpr static T seconds_per_Hz     = 1.0;


        //constructors
        cSimtime_t () : _realtime(0.0){};
        cSimtime_t (T val) : _realtime(val){};

        //convert to T
        //implicitly enables comparison functions
        operator T() const { return _realtime; }

        //convert to frequency
        T frequency(void) const { return (seconds_per_Hz / _realtime); }


        //output formats
        T year   (void) const { return (_realtime / seconds_per_year  ); }
        T day    (void) const { return (_realtime / seconds_per_day   ); }
        T hour   (void) const { return (_realtime / seconds_per_hour  ); }
        T minute (void) const { return (_realtime / seconds_per_minute); }
        T s      (void) const { return  _realtime;                       }
        T ms     (void) const { return (_realtime * 1.0E3             ); }
        T us     (void) const { return (_realtime * 1.0E6             ); }
        T ns     (void) const { return (_realtime * 1.0E9             ); }
        T ps     (void) const { return (_realtime * 1.0E12            ); }
        T fs     (void) const { return (_realtime * 1.0E15            ); }
        T as     (void) const { return (_realtime * 1.0E18            ); }

        T PHz    (void) const { return ( frequency() / 1.0E15         ); }
        T THz    (void) const { return ( frequency() / 1.0E12         ); }
        T GHz    (void) const { return ( frequency() / 1.0E9          ); }
        T MHz    (void) const { return ( frequency() / 1.0E6          ); }
        T kHz    (void) const { return ( frequency() / 1.0E3          ); }
        T Hz     (void) const { return   frequency();                    }
        T mHz    (void) const { return ( frequency() * 1.0E3          ); }
        T uHz    (void) const { return ( frequency() * 1.0E6          ); }
        T nHz    (void) const { return ( frequency() * 1.0E9          ); }
        T pHz    (void) const { return ( frequency() * 1.0E12         ); }
        T fHz    (void) const { return ( frequency() * 1.0E15         ); }


        //Overload operators
        cSimtime_t& operator+=(const cSimtime_t& rhs)
        {
            _realtime += rhs._realtime;
            return *this;
        }

        cSimtime_t& operator-=(const cSimtime_t& rhs)
        {
            _realtime -= rhs._realtime;
            return *this;
        }

        cSimtime_t& operator*=(const cSimtime_t& rhs)
        {
            _realtime *= rhs._realtime;
            return *this;
        }

        cSimtime_t& operator/=(const cSimtime_t& rhs)
        {
            _realtime /= rhs._realtime;
            return *this;
        }

        //overload <<
        friend std::ostream& operator<<(std::ostream& out, const simtime_t& t);
    };
```

Following this also literals are created for units, the main reason for this is the readability of our code. This creates the possibility to pass parameters in a easy readable way, let's take a example to explain this. We want to create a clock of 100MHz, which is a period of 10ns. We now have to pass this into our add clock function, which takes the period of a clock in seconds. In our conventional way we do it like to following: ``` addClock(0,000000001); // Create clock of 10ns```, as visible this is not really readable and we even have to check every time if we added the right number of zero's. With using literals we would add it as ``` addClock(10_ns);```, which makes it easy to read and less error prone. For more information about [c++ literals](https://www.geeksforgeeks.org/user-defined-literals-cpp/).

``` c++
namespace units 
{
    /* cSimtime_t
        */
    inline simtime_t operator""_yr     (long double        val) { return simtime_t(val * simtime_t::seconds_per_year  ); }
    inline simtime_t operator""_year   (long double        val) { return simtime_t(val * simtime_t::seconds_per_year  ); }
    inline simtime_t operator""_d      (long double        val) { return simtime_t(val * simtime_t::seconds_per_day   ); }
    inline simtime_t operator""_day    (long double        val) { return simtime_t(val * simtime_t::seconds_per_day   ); }
    inline simtime_t operator""_h      (long double        val) { return simtime_t(val * simtime_t::seconds_per_hour  ); }
    inline simtime_t operator""_hr     (long double        val) { return simtime_t(val * simtime_t::seconds_per_hour  ); }
    inline simtime_t operator""_hour   (long double        val) { return simtime_t(val * simtime_t::seconds_per_hour  ); }
    inline simtime_t operator""_min    (long double        val) { return simtime_t(val * simtime_t::seconds_per_minute); }
    inline simtime_t operator""_minutes(long double        val) { return simtime_t(val * simtime_t::seconds_per_minute); }
    inline simtime_t operator""_s      (long double        val) { return simtime_t(val * 1.0                          ); }
    inline simtime_t operator""_ms     (long double        val) { return simtime_t(val / 1.0E3                        ); }
    inline simtime_t operator""_us     (long double        val) { return simtime_t(val / 1.0E6                        ); }
    inline simtime_t operator""_ns     (long double        val) { return simtime_t(val / 1.0E9                        ); }
    inline simtime_t operator""_ps     (long double        val) { return simtime_t(val / 1.0E12                       ); }
    inline simtime_t operator""_fs     (long double        val) { return simtime_t(val / 1.0E15                       ); }
    inline simtime_t operator""_as     (long double        val) { return simtime_t(val / 1.0E18                       ); }

    inline simtime_t operator""_yr     (unsigned long long val) { return simtime_t( (long double)val * simtime_t::seconds_per_year  ); }
    inline simtime_t operator""_year   (unsigned long long val) { return simtime_t( (long double)val * simtime_t::seconds_per_year  ); }
    inline simtime_t operator""_d      (unsigned long long val) { return simtime_t( (long double)val * simtime_t::seconds_per_day   ); }
    inline simtime_t operator""_day    (unsigned long long val) { return simtime_t( (long double)val * simtime_t::seconds_per_day   ); }
    inline simtime_t operator""_h      (unsigned long long val) { return simtime_t( (long double)val * simtime_t::seconds_per_hour  ); }
    inline simtime_t operator""_hr     (unsigned long long val) { return simtime_t( (long double)val * simtime_t::seconds_per_hour  ); }
    inline simtime_t operator""_hour   (unsigned long long val) { return simtime_t( (long double)val * simtime_t::seconds_per_hour  ); }
    inline simtime_t operator""_min    (unsigned long long val) { return simtime_t( (long double)val * simtime_t::seconds_per_minute); }
    inline simtime_t operator""_minutes(unsigned long long val) { return simtime_t( (long double)val * simtime_t::seconds_per_minute); }
    inline simtime_t operator""_s      (unsigned long long val) { return simtime_t( (long double)val * 1.0                          ); }
    inline simtime_t operator""_ms     (unsigned long long val) { return simtime_t( (long double)val / 1.0E3                        ); }
    inline simtime_t operator""_us     (unsigned long long val) { return simtime_t( (long double)val / 1.0E6                        ); }
    inline simtime_t operator""_ns     (unsigned long long val) { return simtime_t( (long double)val / 1.0E9                        ); }
    inline simtime_t operator""_ps     (unsigned long long val) { return simtime_t( (long double)val / 1.0E12                       ); }
    inline simtime_t operator""_fs     (unsigned long long val) { return simtime_t( (long double)val / 1.0E15                       ); }
    inline simtime_t operator""_as     (unsigned long long val) { return simtime_t( (long double)val / 1.0E18                       ); }


    inline simtime_t operator""_PHz    (long double        val) { return simtime_t(simtime_t::seconds_per_Hz / (val * 1.0E15)); }
    inline simtime_t operator""_THz    (long double        val) { return simtime_t(simtime_t::seconds_per_Hz / (val * 1.0E12)); }
    inline simtime_t operator""_GHz    (long double        val) { return simtime_t(simtime_t::seconds_per_Hz / (val * 1.0E9 )); }
    inline simtime_t operator""_MHz    (long double        val) { return simtime_t(simtime_t::seconds_per_Hz / (val * 1.0E6 )); }
    inline simtime_t operator""_KHz    (long double        val) { return simtime_t(simtime_t::seconds_per_Hz / (val * 1.0E3 )); }
    inline simtime_t operator""_Hz     (long double        val) { return simtime_t(simtime_t::seconds_per_Hz / (val * 1.0   )); }
    inline simtime_t operator""_mHz    (long double        val) { return simtime_t(simtime_t::seconds_per_Hz / (val / 1.0E3 )); }
    inline simtime_t operator""_uHz    (long double        val) { return simtime_t(simtime_t::seconds_per_Hz / (val / 1.0E6 )); }
    inline simtime_t operator""_nHz    (long double        val) { return simtime_t(simtime_t::seconds_per_Hz / (val / 1.0E9 )); }
    inline simtime_t operator""_pHz    (long double        val) { return simtime_t(simtime_t::seconds_per_Hz / (val / 1.0E12)); }
    inline simtime_t operator""_fHz    (long double        val) { return simtime_t(simtime_t::seconds_per_Hz / (val / 1.0E15)); }

    inline simtime_t operator""_PHz    (unsigned long long val) { return simtime_t(simtime_t::seconds_per_Hz / (val * 1.0E15)); }
    inline simtime_t operator""_THz    (unsigned long long val) { return simtime_t(simtime_t::seconds_per_Hz / (val * 1.0E12)); }
    inline simtime_t operator""_GHz    (unsigned long long val) { return simtime_t(simtime_t::seconds_per_Hz / (val * 1.0E9 )); }
    inline simtime_t operator""_MHz    (unsigned long long val) { return simtime_t(simtime_t::seconds_per_Hz / (val * 1.0E6 )); }
    inline simtime_t operator""_KHz    (unsigned long long val) { return simtime_t(simtime_t::seconds_per_Hz / (val * 1.0E3 )); }
    inline simtime_t operator""_Hz     (unsigned long long val) { return simtime_t(simtime_t::seconds_per_Hz / (val * 1.0   )); }
    inline simtime_t operator""_mHz    (unsigned long long val) { return simtime_t(simtime_t::seconds_per_Hz / (val / 1.0E3 )); }
    inline simtime_t operator""_uHz    (unsigned long long val) { return simtime_t(simtime_t::seconds_per_Hz / (val / 1.0E6 )); }
    inline simtime_t operator""_nHz    (unsigned long long val) { return simtime_t(simtime_t::seconds_per_Hz / (val / 1.0E9 )); }
    inline simtime_t operator""_pHz    (unsigned long long val) { return simtime_t(simtime_t::seconds_per_Hz / (val / 1.0E12)); }
    inline simtime_t operator""_fHz    (unsigned long long val) { return simtime_t(simtime_t::seconds_per_Hz / (val / 1.0E15)); }
}
```

## Single clock instance

Let's talk about our single clock instance, which is a instance to control a single clock input on the testbench. A single design can have multiple of those instances and in this way create multiple clocks in a design. Where every instance will have it's own settings and work according to this. One of the most important things which the clocks needs are it's high and low period, this of course determines the frequency of the clock pin. The low period will determine how long the signal will be '0' and the high period will determine the time the signal is '1'. Next to that we need a reference to the pin in the design, so we can toggle it as needed. Also internally stored is the ``` _timeToNextEvent ```, which will count the time still needed to change the current state of the device. Note that it's not needed to keep track of our state since we can read the pin itself for this. From function perspective most of them speak for them self. Only the ``` updateTime() ``` is interesting, this will update our ```_timeToNextEvent``` and when it reaches zero we have to toggle the state and set the new period inside of our ```_timeToNextEvent```. 

``` c++
 class cClock : common::cUniqueId
{
    private:
    uint8_t&    _clk;             //!< Points to testbench clock variable
    simtime_t  _lowPeriod;        //!< Clock Low Period in seconds
    simtime_t  _highPeriod;       //!< Clock High Period in seconds
    simtime_t  _timeToNextEvent;  //!< Time until next event

    void toggle(void);

    public:

    cClock(uint8_t& clk, simtime_t LowPeriod, simtime_t HighPeriod);
    virtual ~cClock(void);

    virtual simtime_t setLowPeriod(simtime_t Period);
    virtual simtime_t getLowPeriod(void) const;

    virtual simtime_t setHighPeriod(simtime_t Period);
    virtual simtime_t getHighPeriod(void) const;

    virtual simtime_t getPeriod(void) const;
    virtual long double getFrequency(void) const;

    virtual simtime_t getTimeToNextEvent(void) const;
    virtual simtime_t updateTime(simtime_t TimePassed);
    
};

void toggle(void)
{
    //toggle clock, changing the state in the simulation
    _clk = !_clk;

    //Update TimeKeep
    _timeToNextEvent = _clk ? _highPeriod : _lowPeriod;
}

virtual simtime_t updateTime(simtime_t TimePassed)
{
    #ifdef DBG_CLOCK_H
    DEBUG << "CLOCK_H(" << id() << ") updateTime(" << TimePassed << ")" << std::endl;
    #endif

    //Subtract time-passed from the time until the next event
    _timeToNextEvent -= TimePassed;

    //_TimeToNextEvent should never be negative
    assert(_timeToNextEvent >= 0);

    //if TimeToNextEvent==0, then toggle the clock
    if (_timeToNextEvent==0) 
    {
        toggle();
    }

    return _timeToNextEvent;
}

```

## Clock manager

As last is the clock manager, which is the main part of controlling the time. It holds a vector to all clocks, so it can also control the clocks as needed. Next to that it has the ```_time``` variable, which is the main source of time and will be used to generate the waveform files. Next to this there are some functions to add/create new clocks, which speak for themself. It's important that we make sure that there is atleast a single clock, before ticking our system one tick further. Everything depends on at least a single clock else there is no way to have any timekeeping. The ```getTime()``` function will return our current simulation time variable, ```_time```. The most important function within this class will be the tick function, it will loop through all the clocks to find the next clock which has to change state. When it has found this, it will loop through all the clocks again to update their corresponding time and then proceed to update our own time.

``` C++
class cClockManager
{
    private:
        std::vector<cClock*> *_clocks;   //Collection holding all clocks
        simtime_t _time;

    public:

        cClockManager(void);
        virtual ~cClockManager(void);

        virtual void add(cClock &Clock) const;
        virtual void add(cClock* Clock) const;
        virtual cClock* const add(uint8_t& Clock, simtime_t LowPeriod, simtime_t HighPeriod) const;
        virtual cClock* const add(uint8_t& Clock, simtime_t Period) const;

        virtual bool empty(void) const;
        virtual simtime_t getTime(void) const;

        simtime_t tick(void);        
};

simtime_t tick(void)
{
    #ifdef DBG_CLOCKMANAGER_H
    DEBUG << "CLOCKMANAGER_H - tick()" << std::endl;
    #endif

    //Current minimum time to next event = maximum float value
    simtime_t minTimeToNextEvent = simtime_t::max();

    //Find next event
    for (const auto clk : *_clocks)
    {
        simtime_t timeToNextEvent = clk->getTimeToNextEvent();

        if (timeToNextEvent < minTimeToNextEvent)
        {
            minTimeToNextEvent = timeToNextEvent;
        }             

        #ifdef DBG_CLOCKMANAGER_H
        DEBUG << "CLOCKMANAGER_H - tick: " << clk->getTimeToNextEvent() << "," << minTimeToNextEvent << "\n";
        #endif

    }

    //Update all clocks
    //Clocks automatically toggle when their time reaches 0
    for (const auto clk : *_clocks)
    {
        clk->updateTime(minTimeToNextEvent);
    }

    //Store new time
    _time += minTimeToNextEvent;

    return _time;
}

```

# Adjusting our testbench and generating some waveforms

The UART16550 uart has a single clock which is connected through the PCLK clock pin. As we have previously seen, where we also used this pin to toggle. Let's now adjust the generic testbench and the APB16550 uart in such way that we can generate a clock on the PCLK pin.

## General testbench

First we start by extending the general testbench to have the clock manager. This is a HAS-A relationship, however we want to control it's lifetime. Therefore we will create and destroy the clock manager with the constructor and destructor of this class. The clock manager will be stored as a pointer in the private variables, ```cClockManager* _clkMgr;```. In our constructer we then create the clock manager with ```_clkMgr = new cClockManager(); ``` and the destructor deletes the object ```delete _clkMgr;```.

Now let's add the clocks through our general testbench. Two functions could be helpful, first one where we can define the period and a second one where we can set a low and high period. Both functions are very simple, they just pass the given information to the clock manager, who will then continue to create and store the clock. Don't forget, we also need the clock pin itself.

``` C++
virtual cClock* addClock(uint8_t& Clock, simtime_t LowPeriod, simtime_t HighPeriod) const
{
    return _clkMgr->add(Clock, LowPeriod, HighPeriod);
}

virtual cClock* addClock(uint8_t& Clock, simtime_t Period) const
{
    return _clkMgr->add(Clock, Period);
}
```

Secondly for debug purposes it's helpful to get the time a full simulation has run, or to get the timestamp at a certain point for information. Therefore we also add a function to get the full time. The clock manager also stores this, so we can just retrieve it directly from the clock manager.

``` C++
virtual simtime_t getTime(void) const
{
    return _clkMgr->getTime();
}
```
Following this, we also need to update our tick function. Since the timing is now controlled by the clock manager we should adjust accordingly. Where we previously just evaluated the design and if configured used the trace, we now have to also tick the clock manager. So let's proceed to add this into our code, which is not to exiting. 

```c++
void tick(int tickCount) const
{
#ifdef DBG_TESTBENCH_H
    DEBUG << "TESTBENCH_H - tick()" << std::endl;
#endif

    //eval logic
    _core->eval();

    //dump trace
    if (_traceActive && _trace)
    {
        simtime_t time = _clkMgr->getTime();
        _trace->dump( (vluint64_t)(time.ps()) );
    }

    _clkMgr->tick();
}
```

As we can see for our ```trace->dump()``` function we pass the time from our clock manager in pico seconds. Where this is perfectly fine as long as our simulation is running in pico seconds, it will not work when our simulation runs in nano seconds. In the latter case 1ns in simulation time shows up as 1ps in the waveform. To solve this problem we will need to match the simulation time with the waveform. Happily we can get the settings from our Verilated context by the function ``` timeprecision () ```. It returns the prevision back in a negative factor of 10, so it returns 12 for pico seconds and 9 for nano seconds. With this knowledge we can calculate the time precision  in the following way ```_timeprecision  = pow(10, 0 - context->time precision ());```. The calculated precision can then be used to map the simulation time with the waveform time, which is done by getting the simulation time in seconds and this divided by the time precision . Within our code we add the ```_timeprecision ``` as a private variable and perform the calculation in the constructor, since it won't change anymore after that. Our tick function is now updated with the following code, which will calculate the right time.

``` c++
simtime_t time = _clkMgr->getTime();
_trace->dump( (vluint64_t)(time.s() / _time precision ) );
```

## Expanding the UART16550 

Now the only thing left is to change the UART16550 testbench. Now we need to create and control a clock, let's for now create a clock of 100MHz. So first we add the clock into our class definition as ```cClock* pclk;```, in our system Verilog design the clock is called pclk so let's reuse this name to make it nice and consistent. Following this, in the constructor we will add the clock with the functions created in the general testbench, ```pclk = addClock(_core->PCLK, 10.0_ns);```. As visible we use the same pin as we previously used to toggle in the run routine, ```_core->PCLK ^= 1;```. As second parameter we give the period of the signal, 100MHz is 10 ns.
``` c++
class cAPBUart16550TestBench : public cTestBench<Vapb_uart16550>
{
    private:
        cClock* pclk;

    public:

        cAPBUart16550TestBench(VerilatedContext* context, bool traceActive);
        ~cAPBUart16550TestBench();

        int run(int numCycles);
};

cAPBUart16550TestBench::cAPBUart16550TestBench(VerilatedContext* context, bool traceActive) : 
    cTestBench<Vapb_uart16550>(context, traceActive)
{
    //define new clock
    pclk = addClock(_core->PCLK, 10.0_ns);

} 
```

Now we added a clock in our design, so let's see how we have to update our run function. In our previous function we first changed the state of the clock and following that we ticked our general testbench. Since our pin is now controlled by the clock we can now remove the part which changed it's state. That's all that we have to change for now, if we think closely about this it is a bit strange. We still tick our testbench for 20 ticks, but it now has some kind of timing. That's correct this is still a bit strange but we will get onto that in our next blog, where we start to generate some more signals. Let's take a look now into our adjusted function.

``` c++
int cAPBUart16550TestBench::run(int numCycles)
{
    // loop 20 times
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

# Running our new design

Alright everything is now updated, let's compile and run a first time. Don't forget to enable the tracing so we can also take a nice look into our new clock. As we can see in the image below, our clock nicely ticks on 10ns or 100MHz.

![100MHz clock](/images/blog3/100MHzClock.png)

Let's test our code a bit more and introduce some more clocks. For test purposes let's misuse some of the signals and create a few more clocks in our design, for this we introduce 3 different clocks, 120MHz, 100MHz and 50MHz. The following code is added in the constructor of the APBuart16550 testbench class, replacing where we previously added our clock. Now let's run our design and check our 3 clocks.

``` c++
pclk = addClock(_core->PCLK, 8.0_ns);       // 120MHz clock
pclk = addClock(_core->PSEL, 15.0_ns);      // 50MHz clock
pclk = addClock(_core->PENABLE, 10.0_ns);   // 100MHz clock
```

![Wrong clocks](/images/blog3/3ClocksWrong.png)

Hmmm, they all three start nicely as expected. However at 15ns we can see that our 50MHz clock does not go low, let's try to find out why. First enable some of our logging in the clock manager and the individual clocks. Which will output the following trace: 

[DEBUG] CLOCK_H constructor(0) lvl=0 LowPeriod=4ns
[DEBUG] CLOCK_H constructor(1) lvl=0 LowPeriod=7.5ns
[DEBUG] CLOCK_H constructor(2) lvl=0 LowPeriod=5ns
[DEBUG] CLOCKMANAGER_H - tick() 
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:4ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:7.5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:5ns
[DEBUG] CLOCK_H(0) updateTime(4ns)
[DEBUG] CLOCK_H(1) updateTime(4ns)
[DEBUG] CLOCK_H(2) updateTime(4ns)
[DEBUG] Loopcounter: 0
[DEBUG] CLOCKMANAGER_H - tick() 
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:4ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:3.5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:1000ps
[DEBUG] CLOCK_H(0) updateTime(1000ps)
[DEBUG] CLOCK_H(1) updateTime(1000ps)
[DEBUG] CLOCK_H(2) updateTime(1000ps)
[DEBUG] Loopcounter: 1
[DEBUG] CLOCKMANAGER_H - tick() 
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:3ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:2.5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:5ns
[DEBUG] CLOCK_H(0) updateTime(2.5ns)
[DEBUG] CLOCK_H(1) updateTime(2.5ns)
[DEBUG] CLOCK_H(2) updateTime(2.5ns)
[DEBUG] Loopcounter: 2
[DEBUG] CLOCKMANAGER_H - tick() 
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:500ps
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:7.5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:2.5ns
[DEBUG] CLOCK_H(0) updateTime(500ps)
[DEBUG] CLOCK_H(1) updateTime(500ps)
[DEBUG] CLOCK_H(2) updateTime(500ps)
[DEBUG] Loopcounter: 3
[DEBUG] CLOCKMANAGER_H - tick() 
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:4ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:7ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:2ns
[DEBUG] CLOCK_H(0) updateTime(2ns)
[DEBUG] CLOCK_H(1) updateTime(2ns)
[DEBUG] CLOCK_H(2) updateTime(2ns)
[DEBUG] Loopcounter: 4
[DEBUG] CLOCKMANAGER_H - tick() 
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:2ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:5ns
[DEBUG] CLOCK_H(0) updateTime(2ns)
[DEBUG] CLOCK_H(1) updateTime(2ns)
[DEBUG] CLOCK_H(2) updateTime(2ns)
[DEBUG] Loopcounter: 5
[DEBUG] CLOCKMANAGER_H - tick() 
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:4ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:3ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:3ns
[DEBUG] CLOCK_H(0) updateTime(3ns)
[DEBUG] CLOCK_H(1) updateTime(3ns)
[DEBUG] CLOCK_H(2) updateTime(3ns)
[DEBUG] Loopcounter: 6
[DEBUG] CLOCKMANAGER_H - tick() 
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:1000ps
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:4.03897e-10as
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:5ns
[DEBUG] CLOCK_H(0) updateTime(4.03897e-10as)
[DEBUG] CLOCK_H(1) updateTime(4.03897e-10as)
[DEBUG] CLOCK_H(2) updateTime(4.03897e-10as)
[DEBUG] Loopcounter: 7
%Warning: previous dump at t=15000, requesting t=15000, dump call ignored
[DEBUG] CLOCKMANAGER_H - tick() 
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:1000ps
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:7.5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:5ns
[DEBUG] CLOCK_H(0) updateTime(1000ps)
[DEBUG] CLOCK_H(1) updateTime(1000ps)
[DEBUG] CLOCK_H(2) updateTime(1000ps)
[DEBUG] Loopcounter: 8
[DEBUG] CLOCKMANAGER_H - tick() 
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:4ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:6.5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:4ns
[DEBUG] CLOCK_H(0) updateTime(4ns)
[DEBUG] CLOCK_H(1) updateTime(4ns)
[DEBUG] CLOCK_H(2) updateTime(4ns)
[DEBUG] Loopcounter: 9
[DEBUG] CLOCKMANAGER_H - tick() 
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:4ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:2.5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:4.03897e-10as
[DEBUG] CLOCK_H(0) updateTime(4.03897e-10as)
[DEBUG] CLOCK_H(1) updateTime(4.03897e-10as)
[DEBUG] CLOCK_H(2) updateTime(4.03897e-10as)

Until loop counter 6 everything seems to go as expected, all clocks count down nicely and get the expected values. However from loop counter 6 we see that the time to next event for 50MHz is 4.03897e-10as, which is 0,0000000000000000000000000000403987 seconds. This is way smaller than our used accuracy of 1ps, it seems that we have rounding error in our calculation. Theoretically it should work but we are working with a PC simulation so it seems we have a rounding error in our system. 

It at least explains why we see that strange timing value, but why do we see it so strange in our waveform? Well we pass the passed time into our ```_trace->dump()``` function. This function takes the time as vluint64_t, or as integer, in the time precision  of our simulation. So when we pass such a small amount of simulation time, it will be rounded twice to the same value. We can also see this in our trace, as verilator warns us about it: ```%Warning: previous dump at t=15000, requesting t=15000, dump call ignored ```. This happens exactly at 15000ps, exactly the right moment that our 100MHz and 50MHz clock toggle. So we now learned that we can't write two times with the same timestamp into our trace file, which also seems logical. So the toggle which should happen at 15000ps is not registered at this moment but written at the next point, which in our setting is at 16000ps. 

We now learned that our pure theoretical approach doesn't work as expected in reality. So how can we make sure that everything toggles at the right moment, well if we look more closely in our design, we know that we switch a clock signal when we update the time or our clock, ```cClock::updateTime()```. Within this function we check that our clock is exactly 0, however it can happen due to rounding errors that it is a bit higher then 0. If we now check if our ```_timeToNextEvent``` is between 0 and our simulation precision, the clock will then always be updated at the right moment and within the right precision. This means that in our design the ```cClock``` class needs to know about the time precision, so when creating a clock we have to pass this as a parameter.

First we adjust our ```cClock``` class. As a private variable we add ```simtime_t  _precision; ```, following this we extend the constructor to also need a simtime_t precision, so we now the precision of this specific clock.

``` c++
cClock(uint8_t& clk, simtime_t precision, simtime_t LowPeriod, simtime_t HighPeriod) :
        _clk(clk),
        _precision(precision),
        _lowPeriod(LowPeriod),
        _highPeriod(HighPeriod)
    {
        //set clock low
        _clk = 0;

        //clock is low, so TimeToNextEvent is LowPeriod
        _timeToNextEvent = LowPeriod;

        #ifdef DBG_CLOCK_H
        DEBUG << "CLOCK_H constructor(" << id() << ") lvl=" << (unsigned)_clk << " LowPeriod=" << _timeToNextEvent << "\n";
        #endif
    }
```

Following this we now also need to change the ```updateTime()``` function. Where we compare if the ```_timeToNextEvent``` is in between 0 and our precision, if so then we toggle our clock.

``` c++
virtual simtime_t updateTime(simtime_t TimePassed)
{
    #ifdef DBG_CLOCK_H
    DEBUG << "CLOCK_H(" << id() << ") updateTime(" << TimePassed << ")\n";
    #endif
    //Subtract time-passed from the time until the next event
    _timeToNextEvent -= TimePassed;

    //_TimeToNextEvent should never be negative
    assert(_timeToNextEvent >= 0);

    //It's possible that there are some small rouding errors in the 
    //calculation so therefore use a window of the given precision and 0
    if (_timeToNextEvent >= 0 && _timeToNextEvent < _precision) 
    {
        toggle();
    }

    return _timeToNextEvent;
}
```

Our clock is now up to date, however when we create our clock we have to pass the precision now. Meaning that our clock manager also needs to be extended, here we do the same, add a private variable ```simtime_t _precision; ``` and following this extend the constructor to take it as an argument and place it in the variable. When we now add a clock we only have to add this variable into the constructor for the clock and the right value is passed. Anyone who calls the add clock doesn't need the knowledge about the simulation precision, this is now nicely encapsulated.

``` c++
virtual cClock* const add(uint8_t& Clock, simtime_t LowPeriod, simtime_t HighPeriod) const
{
    //Create new VClock
    cClock* clock = new cClock(Clock, _precision, LowPeriod, HighPeriod);

    //Add clock
    add(clock);

    //Return new object
    return clock;
}
```

Only thing left is to pass the right precision into our clock manager. This is the easiest, since we already calculated the precision to calculate the right time for our tracing. Within our constructor of the general testbench we create the clock manager and we have the time precision, only thing left is to pass this value into our clock manager. Solving this at this level means that this is implemented in the general layer and is always function for all different kinds of testbenches we create.

``` c++
cTestBench(VerilatedContext* context, bool traceActive) :
    _context(context),
    _finished(false),
    _traceActive(traceActive)
{
    if(traceActive)
    {
        Verilated::traceEverOn(true);
    }

    _time precision  = pow(10, 0 - context->time precision ());

    _core = new VM; // Create a new verilator model
    _clkMgr = new cClockManager(_time precision ); //Create new Clock Manager
    _trace = nullptr;
}
```

Let's run our design with trace and debug output and see what happens.

[DEBUG] CLOCK_H constructor(0) lvl=0 LowPeriod=4ns
[DEBUG] CLOCK_H constructor(1) lvl=0 LowPeriod=7.5ns
[DEBUG] CLOCK_H constructor(2) lvl=0 LowPeriod=5ns
[DEBUG] CLOCKMANAGER_H - tick()
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:4ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:7.5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:5ns
[DEBUG] CLOCK_H(0) updateTime(4ns)
[DEBUG] CLOCK_H(1) updateTime(4ns)
[DEBUG] CLOCK_H(2) updateTime(4ns)
[DEBUG] Loopcounter: 0
[DEBUG] CLOCKMANAGER_H - tick()
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:4ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:3.5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:1000ps
[DEBUG] CLOCK_H(0) updateTime(1000ps)
[DEBUG] CLOCK_H(1) updateTime(1000ps)
[DEBUG] CLOCK_H(2) updateTime(1000ps)
[DEBUG] Loopcounter: 1
[DEBUG] CLOCKMANAGER_H - tick()
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:3ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:2.5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:5ns
[DEBUG] CLOCK_H(0) updateTime(2.5ns)
[DEBUG] CLOCK_H(1) updateTime(2.5ns)
[DEBUG] CLOCK_H(2) updateTime(2.5ns)
[DEBUG] Loopcounter: 2
[DEBUG] CLOCKMANAGER_H - tick()
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:500ps
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:7.5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:2.5ns
[DEBUG] CLOCK_H(0) updateTime(500ps)
[DEBUG] CLOCK_H(1) updateTime(500ps)
[DEBUG] CLOCK_H(2) updateTime(500ps)
[DEBUG] Loopcounter: 3
[DEBUG] CLOCKMANAGER_H - tick()
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:4ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:7ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:2ns
[DEBUG] CLOCK_H(0) updateTime(2ns)
[DEBUG] CLOCK_H(1) updateTime(2ns)
[DEBUG] CLOCK_H(2) updateTime(2ns)
[DEBUG] Loopcounter: 4
[DEBUG] CLOCKMANAGER_H - tick()
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:2ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:5ns
[DEBUG] CLOCK_H(0) updateTime(2ns)
[DEBUG] CLOCK_H(1) updateTime(2ns)
[DEBUG] CLOCK_H(2) updateTime(2ns)
[DEBUG] Loopcounter: 5
[DEBUG] CLOCKMANAGER_H - tick()
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:4ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:3ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:3ns
[DEBUG] CLOCK_H(0) updateTime(3ns)
[DEBUG] CLOCK_H(1) updateTime(3ns)
[DEBUG] CLOCK_H(2) updateTime(3ns)
[DEBUG] Loopcounter: 6
[DEBUG] CLOCKMANAGER_H - tick()
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:1000ps
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:7.5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:5ns
[DEBUG] CLOCK_H(0) updateTime(1000ps)
[DEBUG] CLOCK_H(1) updateTime(1000ps)
[DEBUG] CLOCK_H(2) updateTime(1000ps)
[DEBUG] Loopcounter: 7
[DEBUG] CLOCKMANAGER_H - tick()
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:4ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:6.5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:4ns
[DEBUG] CLOCK_H(0) updateTime(4ns)
[DEBUG] CLOCK_H(1) updateTime(4ns)
[DEBUG] CLOCK_H(2) updateTime(4ns)
[DEBUG] Loopcounter: 8
[DEBUG] CLOCKMANAGER_H - tick()
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:4ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:2.5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:5ns
[DEBUG] CLOCK_H(0) updateTime(2.5ns)
[DEBUG] CLOCK_H(1) updateTime(2.5ns)
[DEBUG] CLOCK_H(2) updateTime(2.5ns)
[DEBUG] Loopcounter: 9
[DEBUG] CLOCKMANAGER_H - tick()
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:1.5ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:7.5ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:2.5ns
[DEBUG] CLOCK_H(0) updateTime(1.5ns)
[DEBUG] CLOCK_H(1) updateTime(1.5ns)
[DEBUG] CLOCK_H(2) updateTime(1.5ns)
[DEBUG] Loopcounter: 10
[DEBUG] CLOCKMANAGER_H - tick()
[DEBUG] CLOCK_H(0) - getTimeToNextEvent:4ns
[DEBUG] CLOCK_H(1) - getTimeToNextEvent:6ns
[DEBUG] CLOCK_H(2) - getTimeToNextEvent:1000ps
[DEBUG] CLOCK_H(0) updateTime(1000ps)
[DEBUG] CLOCK_H(1) updateTime(1000ps)
[DEBUG] CLOCK_H(2) updateTime(1000ps)

Interesting part in our trace was at loopcounter 6, where we now nicely see that the very low timing is resolved. Meaning that our clocks now toggle at the expected moments, let's verify this with our tracing. We don't have a warning from verilator, so we expect it to function now.

![Right clocks](/images/blog3/3ClocksRight.png)

As we can see in our waveform the clocks are now changing state at the right moment as we expect them to change. We now have introduced timing within our cycle based simulator and according this timing/clocking we can now start generating other signals. This will then also be the next part of this series, generating AHB bus signals.