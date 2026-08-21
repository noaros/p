# Blinky Printf

My educational microcontroller journey!

I found this excellent bare-metal tutorial.

https://github.com/cpq/bare-metal-programming-guide

I bought the board for it (STM32 Nucleo-F429ZI) and tried out the steps, ensuring I had a working version to consult.

# Minimal

Next I started all over in an attempt to learn and do the project my way. I love minimal solutions and minimal starting points. Among other things, they are an excellent way to learn because you find the most important elements of the solution while exploring system boundaries. And so I set up out to find the minimal starting point for this project. 

It turns out to not take much code at all! The results are in the 'minimal' folder, and shown below.

![](0_minimal/c.png)
![](0_minimal/ld.png)
![](0_minimal/go.png)

We just need code to set a value in memory, and a linker script to set the startup bytes to run it, and of course a way to compile and deploy the firmware. The 0 in the linker script is where the top of the stack goes, but we won't even need that until dealing with variables. Also we leave space for the rest of the startup vector that we aren't using. There are many ways to set the startup vector. Besides the linker script itself we could have used C (as in the tutorial), or even assembly.

There are however some nuances that were learned from trial and error. The most sneaky one, not discussed at all in the original tutorial, is that the ARM Cortex F4 and many others operates in 'thumb' mode and require the least significant bit of startup vector addresses to be set. This happens in a hidden manner in the C code of original tutorial, via linker adjustment to the reolocation of symbols marked as thumb functions. However, my use of LONG in the linker script does not make this adjustment, and so we must do so explicitly. This was tricky to track down, because I otherwise had code that worked fine when run from STM32CubeProgrammer but would fail on reset or power cycle.

Another nuance is that the actual startup address set by the linker script would seem to be incorrect, as the code is loaded at address 0x8000000, but the linker assumes 0 and thereby sets the address of the startup function to be offset from 0. Yet because of how the chip hardware actually works, there is mirroring between the two address regions and so the wrong address becomes right, even preferred!

Note how much less code is involved here than the 'minimal' part of the original tutorial. This gives us the most primitive yet essential way to get feedback from the device. By reading that value in memory we can conclude whether or not our code (or any spot of code we care about) has run at all. This is the first step of debugging when experimenting with new code that doesn't seem to work. There are much fancier debugging techniques of course, but this is the one we get to first, and requires the least setup and preconditions to work. If we can do anything at all, we can do this. Technically we don't even need the loop at the end merely to get feedback, but without it the processor continues past the end and results in a "core is locked up" error. Speaking of that loop, I was being a bit cute using *goto*, but that is the simplest form of an infinite loop.

Using STM32CubeProgrammer confirms that indeed our code ran and poked its hardcoded value into memory. It is also important to vet this after power reset, to clear the memory and know that the code starts up properly. For this reason I often found it more convenient to instead use a continually incrementing value.

![](0_minimal/mem.png)

Below is a clip of a compile, run, and check loop using STM32CubeProgrammer.

https://github.com/user-attachments/assets/1ac60f18-6b0d-4a01-875a-ddc045292433

# Blinky Light

Ah, time for the 'hello world' of microcontrollers, making a LED blink! I was able to do it with considerably less code than the original tutorial. To be fair, the author was trying to illustrate what he regards as good practices and abstractions, whereas I'm collapsing those abstractions to find the essence.

![](1_blinky/c.png)
![](1_blinky/ld.png)

The code is just setting addresses to values, and a for-loop for delays. Of course the meaning of those addresses, documented in the datasheet, is quite significant. We have to activate the GPIO pin bank we are interested in by enabling its clock using the appropriate RCC (reset and clock control) register. Then we must set the mode to 'output' for the appropriate pin. Then we can flip a bit to turn it on, or a different bit (oddly) to turn it off. As I worked through this exercise I began to ween myself off the original tutorial and derive directly from the datasheet more and more.

Because the for-loops have a variable, which requires a working stack, we must also tweak our setup by replacing the 0 in the linker script for the initial bytes to be a valid stack address.

Behold, the blinky blue light, in all its glory..

https://github.com/user-attachments/assets/6c7ab9b1-b4a0-4f6e-8eb2-5e1eaab81807

Isn't it beautiful? Apologies for the shaky camera work.

# Timer

A potential problem with the previous version is a simple for-loop was used for the delay, which is a bit ad hoc and makes it difficult to control precise timing. The original tutorial solves this by using an interrupt in conjunction with the SysTick timer. I opted for a simpler solution using only the SysTick timer. 

![](2_systick/c.png)

We first configure the timer and enable its clock. Then we simply let the timer count down and check a built in flag that sets when it reaches zero. The flag is useful because it would be difficult to detect the exact zero state, as without special caution, the code that checks might take more than one clock cycle and hence miss it.

# Interrupts

Even though the previous iteration demonstrated we don't need an interrupt handler to have more precise timing, since interrupts are such a core concept in embedded systems I implemented a basic example anyway. Even still, I opted for a simpler implementation than the author. I merely toggle the light inside the handler, checking the initial state to avoid even needing a variable. We need no timer math of any sort.

![](3_interrupt/c.png)

We need to add our handler to the 16th spot in the startup vector, as shown in the datasheet.

![](3_interrupt/vector.png)

![](3_interrupt/ld.png)

The . syntax is starting to get a bit awkward, so with more startup vectors I'd probably seek alternatives.

The author's approach is worth considering and could be handy in other scenarios. It requires a state variable and timer math, but does allow for event handling logic outside the interrupt itself for multiple events on different recurring timings. The prevailing wisdom is not to put long duration event handling inside interrupt handlers. That doesn't apply here, but is probably correct in many cases. Still, my preference would be to understand precisely why it does or does not apply in any specific case, at is very dependent on the specific system in question. For our purposes we are fine, and would be even if the handling took longer.

# UART

Since we didn't actually need the interrupt, I reverted to the prior base before adding UART functionality.

UART3 uses pins 8 and 9 on GPIOD. We need to enable the clock for GPIOD, set the mode for those pins to 'alternate function', and then set the specific alternate function to be UART3. We set the baud rate, enable the UART, and wake up the transmitter. We can then write one character at a time using the data register and waiting on the status register before we proceed with the next one.

![](4_uart/c.png)

Using putty we can see our transmission.

https://github.com/user-attachments/assets/1981626a-3964-45fb-beb6-2d1f43e960b7

# Printf

I've had fun so far getting by with a non-custom trivial linker script, but that ends with bringing in C library support. That imports a lot of complexity beyond my control, global initialized data with precise ordering in particular. I've managed to still keep a little weirdness, but the linker script becomes more typical. We need to do the standard startup things such as load the DATA segment in flash, but make sure it's referenced in sram, and copy over its contents. In addition we clear the BSS section. These steps would have been done for us except we are using the *-nostdlib* option. Various IDE starting points also provide this.

![](5_printf/ld.png)
![](5_printf/c.png)

Similar to the author I had to supply stub methods, and an implementation for *_sbrk* and *_write*. *Printf* calls *_write*, but it was disconcerting that in many error cases I had a silent fail. Additional build parameters were also needed.

![](5_printf/go.png)

After playing with a number of variations, I eventually established a working *printf*.

https://github.com/user-attachments/assets/4f8f5bb9-3610-493a-b2ec-ab9d4de99425

# CMSIS

Adapting the code required some changes. *SystemInit*, *main*, *_estack*, *_init* definitions are expected by the CMSIS files. Simply fixing those symbols gets our program to work, but that isn't taking advantage of the additional definitions provided, defeating the point. In addition, our startup code copying and initializing data is now redundant, and can be removed.

I encountered a pretty bizarre error. Since I didn't use interrupts, I didn't implement *SysTick_Handler*. The result did not give me a compile error, but a working program that hung in an infinite loop! I learned that apparently this is the default handler implementation, lol. Lesson learned. The solution was to immediately disable the interrupt. An alternative would be to use the interrupt (as done previously).

At this point the program worked and what remained was swapping out my handcoded addresses with the CMSIS equivalents. Had I realized I was going to end up with all these predefined constants it would have been tempting to start there. But I'm glad I got some work deriving directly from the data sheet. I actually used more constants than the author, as he kept in his scheme for pin assignments. I miss the pin numbers a bit myself, so maybe I went too far. I also found it far easier to just google/AI ask for the constants than to find them any other way. That worked surprisingly well, and there were constants defined I never expected.

![](6_cmsis/c.png)
![](6_cmsis/c2.png)

The linker script required some changes because the CMSIS startup sets the vectors.

![](6_cmsis/ld.png)

And the CMSIS includes were needed for the *gcc* command. These directories aren't in the project; they are massive.

![](6_cmsis/go.png)

# Debuggers

For early work on this project I used the Segger Ozone debugger. It was fancy though I didn't relish having to upgrade firmware then restore it. At the time I was using the older ST-LINK utility. The newer STM32CubeProgrammer looks like it works with JLINK, and so this could in theory be avoided, but I didn't test it. When I attempted to work again with Ozone on a new laptop, I somehow destroyed the board and had to order another! After that I was afraid to go back, and setup GDB using OpenOCD. I was able to see and step through the infinite loop of the default interrupt handler.

# IDE

I don't relish IDE's, frameworks, and code wizards, as they generate so much crap in a project while hiding what truly matters. I did use VS Code throughout, but in a vanilla file editing way. Still, I had tentative plans to try out the STM IDE and convert the project. I learned the STM VS Code extension is the preferred path over the standalone IDE, so I gave that a shot after using STM32CubeMX for project generation. The debugger was quite nice, but there was just too much code generated for things I did not need. To be sure I need to continue exploring these options in the future to find the best balance for me, but it was all different enough I decided to abort incorporating anything into this project. I might give it a shot on the next one, and there may be ways to reduce some of the junk with tweaks to MX. If I started with that kind of scaffolding, my first instinct would be to understand all I needed, and delete the rest. My issue was more with MX than the VS Code extension itself, as it was more reasonable when creating an empty project. The linker script was still quite ridiculous, and even the CMake a bit much, but there was some helpful stub scaffolding and startup assembly. Seemed a better balance and worth considering as a startup point. Yet it was still not as simple as any of the previous examples. Controlling and providing all the code myself, but using the CSMIS headers, is my favorite option of what I've seen so far.

# Conclusion

I had fun and learned a lot! Most of the work done on this project is not obvious from the final forms but was spent investigating edge cases and alternates, trying to understand why something wasn't working. I lost a board, and had a plethora of USB connection issues. By the end I gained a lot of comfort with the reference manual (which I thought was the datasheet lol) and general patterns of this kind of development.

 There are some naughty bits in this project as I sought the minimal path. You know, for educational purposes! I omitted many *gcc* build parameters and attributes. Those probably deserve a closer look. I messed around with ad hoc linker scripts, but that's just fun. :smile: I used tight spin loops for delays, those need a *nop* or similar. Poking the start of *sram* with arbitrary logging data is dirty, and probably corrupts something in the *C* startup. I could make it safe by reserving some space with the linker script. I used basic types that could get me in trouble, such as *int* instead of *uint32_t*. Still, the blue light blinks. 

I rather like the bare-metal approach for the simplicity and control. Those are pretty much my programming values, so I suppose that's not a surprise. I know the prevailing wisdom is to turn to RTOS once things get complicated, and while I intend to check that out, I'll be surprised if that is my preference. I love simplicity and I love control. We shall see.

This was a good start. Now that I've got my feet wet, it is time for something more challenging!
