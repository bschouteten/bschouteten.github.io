---
title: Generating time within Verilator
date: 2024-01-20
author: bschouteten
---

# Generating time within Verilator

Welcome back for the third blog in this series. In the last blog we added logging, system options and created a universal testbench. For this blog we will extend our universal testbench and start generating a usefull clock signal.

Untill now we just toggled the clock pin of our design, without thinking further then what is behind this. So let's think further, within this design we have a single clock based design. However looking ahead into our SoC simulation, which is a design that uses more clocks. Since Verilator is a cycle based simulator it's possible to sync the cycles with a clock signal. Or to say it in different words, create a time instance for our simulation. 

# Timing within Verilator

All this time management sounds very strange within a cycle based simulator. Verilator itself doesn't have any knowledge about time, it just checks cycle by cycle endlessly. However we can create a timing with those cycles, since we can count the number of times the clocked has ticked. If you check the first waveform we generated, it's possible to see that this waveform did have some timing, here everything was written in picoseconds. Every call we made to ``` _trace->dump() ``` we passed the value of the for loop into the dump. The number of cycles that our simulator ran is translated into a time in the waveform, meaning we can also do this within our code. System verilog normally runs with a timeunit and timeprecision, for example timeunit of 1ns and timeprecision of 100ps. For Verilator this is by default set to 1ps/1ps to be inline with SystemC. However it's possible to adjust this timescale by passing the ``` --timescale <timeunit>/<timeprecision> ``` to Verilator or by having it within the top level design file of the system. For our simulation this is very helpfull, since we can extract the timeunit and the timeprecision and use this as a reference in our design. 

The first thought which pops up in this case, if we now tick our clock and then increase the time according to the set timeprecision. This would be an easy way to keep track of our simulation time and derive the clocks for this. So if we tick 5 times, we have advanced time 5 * timeprecision. Let's say timeprecision has the default setting of 1 ps, so 5*1 = 5ps. In this way we can continue counting and counting and we just evaluate the design every time. This is fully correct, this would work perfectly and be easy to understand and easy to read. However let's think about the performance of this all, take for example a 100MHz clock, which has a full period of 10ns, switching states every 5ns. We would need to evaluate 5000ps to just toggle the state of the clock once. This solution does so many evaluations which are unnecessary, remember we have synchronized designs which only do something on a clock edge. Of course we could say at this moment, change the timeprecision and set it to 1ns, then you only need 5 ticks to create a 100MHz clock. This is correct and is also more then viable, however when we have a multiple clock design this doesn't work. For exmaple, we have a design with a clock of 100MHz, 125MHz and 133MHz. The 100MHz has a period of 10ns, 125MHz is 8ns and 133MHz is 7.5ns, now it becomes very problematic. Our 1ns tick can not get close to 7.5ns and we need to adjust the precision to get an accurate clock, which brings us back to our original performance problem.

Hmmm... How can we solve this. We know that we only need to evaluate the design at the moment any of the clock changes. So everything what happens between the clock edges is not interesting, meaning we can skip it fully. So if we now instead of counting every cycle a certain amount of time, we just check which clock is the next to toggle and then toggle this clock, evaluate our design and then repeat this process over and over. This would solve our initial problem of having many ticks/evaluations, but how can we at this point keep track of our time? Well as long as every clock knows when to toggle, we can subtract the difference per each clock. So let's take an example, we have a clock of 100MHz and 125MHz. The 100MHz has a period of 10ns, so the first clock edge will come at 5ns. 125MHz clock has a period of 8ns, this first edge will come at 4ns. So from the beginning we check, which clock is first, well this is the 125MHz clock, since it changes at 4ns (4 is lower then 5). But since we now have advanced 4ns in time, we must subtract this 4ns from the 100MHz, so the next edge for the 100MHz clock will come in 5-4 = 1ns. When we now again check what is the next clock that should toggle, the 125MHz clock will say over 4ns, where the 100MHz clock is in 1ns. So we toggle the 100MHz clock at this moment and set it to toggle again in 5ns, the 4ns for the 125MHz clock is then subtracted with 1ns, making it 3ns for the next clock edge. In this way we keep proceeding our internal time and the clocks will then still toggle on the right intervals. Internally we keep track of the elapsed time and can output information or pass this time to our waveform when enabled, syncing everything nicely. Now let's proceed and set this over into a code design and try it out.

# Expanding our design 

To control some sort of timing, it's helpfull to have one instance which will control the timing. From this main instance we can then control one or multiple clocks, let's call this our clock manager. This clock manager will then control the "time" within our simulation context, but also control all the clocks in the design. Every clock is basically the same, it has a low and high period which will be used indefinitely, our testbench then needs to drive the corresponding pin with the right period. With this knowledge we can extend our current general testbench to simulate clocks, see the new UML diagram.

<Add UML diagram>

## Simulation time

As visible within our UML diagram, the clockmanager keeps track of the time with the simtime_t object. The cSimtime_t class is designed to be a single class which has all the knowledge on how to convert timings. So anyone who uses this class can call any of the helper functions to get the expected value without having to do any conversions. 

It's created as a template class and can be used with multiple different base types, making it universal in usage. The realtime is kept as a private variabele, where the time can be gotten with the functions provided within the class. Times can be added and subtracted directly on the variable itself due to the usage of overloaded operators, see [C++ operators](https://www.geeksforgeeks.org/operator-overloading-cpp/) for more infomation about this.

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

Following this also literals are created for units, the main reason for this is the readability of our code. This creates the possibility to pass parameters in a easy readable way, let's take a example to explain this. We want to create a clock of 100MHz, which is a period of 10ns. We now have to pass this into our add clock function, which takes the period of a clock in seconds. In our conventional way we do it like to following: ``` addClock(0,000000001); // Create clock of 10ns```, as visible this is not really readable and we even have to check everytime if we added the right number of zero's. With using literals we would add it as ``` addClock(10_ns);```, which makes it easy to read and less error prone. For more information about [c++ literals](https://www.geeksforgeeks.org/user-defined-literals-cpp/).

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

Let's talk about our single clock instance, which is a instance to control a single clock input on the testbench. A single design can have multiple of those instances and in this way create multiple clocks in a design. Where every instance will have it's own settings and work according to this. One of the most important things which the clocks needs are it's high and low period, this ofcourse determines the frequency of the clock pin. The low period will determine how long the signal will be '0' and the high period will determine the time the signal is '1'. Next to that we need a reference to the pin in the design, so we can toggle it as needed. Also internally stored is the ``` _timeToNextEvent ```, which will count the time still needed to change the current state of the device. Note that it's not needed to keep track of our state since we can read the pin itself for this. From function perspective most of them speak for them self. Only the ``` updateTime() ``` is interesting, this will update our ```_timeToNextEvent``` and when it reaches zero we have to toggle the state and set the new period inside of our ```_timeToNextEvent```. 

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

As last is the clock manager, which is the main part of controlling the time. It holds a vector to all clocks, so it can also control the clocks as needed. Next to that it has the ```_time``` variabele, which is the main source of time and will be used to generate the waveform files. Next to this there are some functions to add/create new clocks, which speak for themself. It's important that we make sure that there is atleast a single clock, before ticking our system one tick further. Everything depends on at least a single clock else there is no way to have any timekeeping. The getTime function will return our current simulation time variable, ```_time```. The most important function within this class will be the tick function, it will loop through all the clocks to find the next clock which has to change state. When it has found this, it will loop through all the clocks again to update their corresponding time and then proceed to update our own time.

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


