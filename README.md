'''html
<p align="center"><font size="+2">Cornell University ECE4760 <br>
  A preemptive  kernel <br>
  for Atmel Mega1284 microcontrollers</font></p>
<p align="left"><b>Introduction</b></p>
<p align="left"> It is useful to be able to write several small tasks running 
  on a microcontroller, as if each one were the only program running. For instance, 
  if you were running several communications protocols, it would be nice if each 
  could run without worrying too much about the operation and timing of the other 
  protocols. It is also handy if you have several state machines running at different 
  rates. Obviously, only one task at a time can actually execute on the MCU. To 
  make each task think it owns the whole machine, the state of the task has to 
  be saved while other tasks execute. The swapping process, as well as other functions 
  (e.g. intertask communication) are handled by an real-time kernel. This 
  page describes a very small, fairly fast kernel for the Mega644. While an real-time kernel makes 
  some aspects of programming easier, there are many times that an MCU cannot 
  run an OS and meet requirements for timing or memory use. For instance, if you 
  are doing one or two very fast operations, like generating video, don't use 
  an operating system. (older STK500/Mega644 <a href="index_stk500.html">documentation</a>)</p>
<p><b>Tiny Real Time (TRT), a real-time kernel</b></p>
<p>The operating system I will describe, called <a href="http://www.control.lth.se/~anton/tinyrealtime/">TinyRealTime</a> (TRT) was written by Dan Henriksson and Anton Cervin (<a href="../RTOS/TinyRealTime.pdf">technical report</a>). (See also <a href="http://lup.lub.lu.se/luur/download?func=downloadFile&recordOId=546028&fileOId=546030">Resource&minus;Constrained Embedded  Control and Computing Systems</a>, Dan Henriksson, 2006) This kernel has several characteristics which make 
  it interesting for ECE4760:</p>
<ul>
  <li>Several tasks can be created, limited by memory space. Practically, probably 
    a maximum of a dozen or so tasks can be run on a Mega1284.</li>
  <li>Each task has a release time and deadline which determines when it executes. The scheduling algorithm is Earliest Deadline First (EDF). </li>
  <li>A task can be put to sleep for a period, thereby freeing the MCU for another 
    task.</li>
  <li>There are semaphores to protect shared resources (e.g. memory, or the UART) 
    which are used by more than one task.</li>
  <li>TRT was written in GCC for the Mega8. Porting to the Mega1284 was 
    easy (rename timer registers, add two bytes to the stack). It could run on any Atmel AVR processor with a 16-bit timer and sufficient 
    RAM.</li>
</ul>
<p align="left">There are several requirements and limitations to apparent task 
  independence on one cpu:</p>
<ul>
  <li> If you can run several tasks (multitask), then it is necessary to communicate 
    between tasks and to synchronize task execution. Communications implies that 
    either shared memory or messages must be passed between tasks. Needless to 
    say, it is easy to corrrupt memory if two or more tasks are &quot;simultaneously&quot; 
    writing to it.</li>
  <li>The ability to preempt a running task implies that the state of the task
     must be saved (including all registers and a stack pointer). You 
    might therefore guess that switching between tasks would take a considerable
     number of cycles. Also, since each task must have its own stack, memory
    is  used up fast.</li>
  <li>The Mega644 hardware does not support any type of memory protection. One 
    task can stomp on another's stack, for instance by defining too many local variables. </li>
  <li><span class="red">NOTE</span>: This code has been verified on AVRstudio 4.15 using the WinAVR-20100110 avr-gcc.exe compiler. It may not compile on other versions. The symptom will be endless resets.</li>
</ul>
<p>The API supplied with TRT includes core real time routines:<br>
</p>
<div align="center"> 
  <table width="100%" border="1">
    <tr> 
      <td width="48%"> 
        <div align="center"><code>void trtInitKernel(uint16_t idletask_stack) </code></div>
      </td>
      <td width="52%"> 
        <p>Sets up the kernel data structures. The parameter is the desired starck
          size of the idle task. For a null idle task, a stack size of 80 should
          be sufficient.</p>
      </td>
    </tr>
    <tr> 
      <td width="48%"> 
        <div align="center"><code>void trtCreateTask(void (*fun)(void*), <br>
          uint16_t stacksize,<br>
        uint32_t release, uint32_t deadline, void *args)</code></div>
      </td>
      <td width="52%">Identifies a function to the kernel as a thread. The parameters
        specify a pointer to the function, the desired stack size, the <em>initial</em> release
        time, the <em>initial</em> deadline time, and an abitrary data input
        sturcture. The release time and deadline must be updated in each task
        whenever <code>trtSleepUntil</code> is called. The task structures are
        statically allocated. Be sure to configure <code>MAXNBRTASKS</code> in
        the kernel file to be big enough. When created, each task initializes
      35 bytes of storage for registers, but <code>stacksize</code> minimum is
      around 40 bytes. If any task stack is too small, the system will crash!</td>
    </tr>
    <tr> 
      <td width="48%" height="28"> 
        <div align="center"><code>void trtTerminate(void)</code></div>
      </td>
      <td width="52%" height="28">Terminates the running task. </td>
    </tr>
    <tr> 
      <td width="48%" height="28"> 
        <div align="center"><code>uint32_t trtCurrentTime(void)</code></div>
      </td>
      <td width="52%" height="28"> 
        <div align="left">Get the current global time in timer ticks. </div>
      </td>
    </tr>
    <tr> 
      <td width="48%" height="52"> 
        <div align="center"><code>void trtSleepUntil(uint32_t release, <br>
        uint32_t deadline)</code></div>
      </td>
      <td width="52%" height="52">Puts a task to sleep by making it ineligible
        to run until the release time. After the release time, it will run when
        it has the nearest deadline. Never use this function in an ISR. The <code>(deadline)
        - (release time) </code>should be greater than than the execution time
        of the thread between <code>trtSleepUntil</code> calls so
        that the kernel can meet all the deadlines. If you give a slow task a
        release time equal to it's deadline, then it has to execute in <em>zero
        time</em> to meet deadline, and nothing else can run until the slow task
        completes. Experiment with <a href="spoiler/TwoTasks_plus_spoiler.c">this
        test code</a> by changing the difference in the <code>spoiler</code> task
      while watching A.0 on the scope. </td>
    </tr>
    <tr> 
      <td width="48%" height="38"> 
        <div align="center"><code>uint32_t trtGetRelease(void)</code></div>
      </td>
      <td width="52%" height="38"> Gets the current release time of the running task. </td>
    </tr>
    <tr> 
      <td width="48%"> 
        <div align="center"><code>uint32_t trtGetDeadline(void)</code></div>
      </td>
      <td width="52%">Gets the current deadline time of the running task. </td>
    </tr>
    <tr> 
      <td width="48%"> 
        <div align="center"><code>void trtCreateSemaphore(uint8_t semnumber, <br>
        uint8_t initval)</code></div>
      </td>
      <td width="52%">Creats a semaphore with identifer semnumber and initial value initval. Be sure to configure <code>MAXNBRSEMAPHORES</code> in the kernel file to be big enough. The identifer number is 1-based, so the first semaphore you define should be numbered <code>1</code>. </td>
    </tr>
    <tr> 
      <td width="48%"> 
        <div align="center"><code>void trtWait(uint8_t semnumber)</code></div>
      </td>
      <td width="52%">Causes the running task to wait for the semaphore if it's value is zero, but coutinues execution (and decrements the semaphore) if the value is greater than zero. Never use this function in an ISR. </td>
    </tr>
    <tr>
      <td><div align="center"><code>void trtSignal(uint8_t semnumber)</code></div></td>
      <td>Adds one to the semaphore. If another thread is waiting on the semaphore, and has a nearer deadline, a context switch will occur. You can use this function in an ISR.</td>
    </tr>
    <tr>
      <td><div align="center"><code>uint8_t trtAccept(uint8_t semnumber)</code></div></td>
      <td>Returns the count of the specified semaphore (before it is decremented). If the semaphore has a nonzero value, the value is decremented. <i>Your task must check the return value. This function does not block. </i>You can use this function in an ISR.</td>
    </tr>
  </table>
</div>
<p><b>Running TRT: </b><strong>Example 1</strong><br>
  Download the following files and put them on one directory. <br>
  <em><strong>Configure a 
    project with <code>test1TRT.c</code> as the only source or header file.
  </strong></em><br>
The code assumes that PORTC is connected to  LEDs on pins 1,2,3,4 and 7. PORTB pin 0 is connected to a switch (and 300 ohm resistor to ground) and the USB-USART jumpers are mounted. The kernel source code is included below by permission of the authors ( Dan Henriksson and Anton Cervin).</p>

  <ul>
    <li><a href="example1/trtkernel_1284.c">trtkernel_1284.c</a></li>
    <li><a href="example1/trtSettings.h">trtSettings.h</a> (configured for 20 MHz crystal, prescalar of 1024, 4 tasks, 7 semaphores) </li>
    <li><a href="example1/test1TRT_1284.c">example</a> test1TRT.c </li>
    <li><a href="example1/trtUart.c">trtUart.c</a>, <a href="example1/trtUart.h">trtUart.h</a> Includes semaphores to signal each received character and end of string.</li>
  </ul>
  <p>The example contains four tasks: 
</p>

  <ul>
    <li>Task1 blinks LED 1 only when button 0 is pressed (to signal a semaphore).</li>
    <li>Task2 blinks LED 2 5 times/sec and signals task1 to execute </li>
    <li> Task3 blinks LED 3 whenever task4 receives user input from the uart, receives a message from task4 and prints a result. If there is no message ready, it just blinks the LED. </li>
    <li>Task4 blinks LED 4 whenever it receives user input from the uart, and prints the system tick-count, number of task executions, and the user input.</li>
    <li>Null task blinks LED 7. You would normally comment out this behavior in a production program. </li>
    <li>All four tasks increment a semaphore-protected shared variable. </li>
  </ul>

  <p>With 4 tasks running, it takes about 700 cycles to perfrom a context switch by signalling a semaphore (<a href="performance/test1TRT.c">Code</a> to test this). The actual time spent in the scheduler ISR (timer1 compare-match) is 390 cycles, plus 70 cycles to store state, plus 70 cycles to restore state equals 530 cycles to service a timer tick. </p>
  <hr>
<p><b>Running TRT: </b> <strong>Example 2 <br>
</strong>Download the following files and put them on one directory. <em><strong><br>
Configure a project with <code>TwoTasksTRT.c</code> as the only source or header file.</strong></em> <br>
The code assumes that PORTC is connected to  eight LEDs (and 300 ohm resistors). PORTB  is connected to eight switches (and 300 ohm resistors to ground) and the USB-USART jumpers are mounted. The kernel source code is included below by permission of the authors ( Dan Henriksson and Anton Cervin).</p>

  <ul>
    <li><a href="example2/trtkernel_1284.c">trtkernel_1284.c</a></li>
    <li><a href="example2/trtSettings.h">trtSettings.h</a> (configured for 20 MHz crystal, prescalar of 1024, 2 tasks, 4 semaphores) </li>
    <li><a href="example2/TwoTasksTRT_1284.c">example</a> TwoTasksTRT.c </li>
    <li><a href="example2/trtUart.c">trtUart.c</a>, <a href="example2/trtUart.h">trtUart.h</a> Includes semaphores to signal each received character and end of string. </li>
  </ul>

<p>The example contains two tasks: </p>
<ul>
  <li>Task 1 reads the buttons (PORTB), latches on the corresponding LED (PORTC), and send a message to the uart console showing which button was pushed. </li>
  <li>Task 2 reads a command from the uart console, then sets, clears, or toggles an LED. The command format is  the character <code>s, c</code> or <code>t</code> followed by a space then a number between 0 and 7. For example <code>s 0</code> sets LED on. </li>
</ul>
<hr>
<p><strong>Configuring TRT</strong>.  
</p>
<p>You need to set the maximum number of tasks in your application and the maximum number of semaphores used. The prescalar can be set to any value allowed for Timer1, but the two prescalar <code>defines</code> need to match. You also need to tell the kernel what the crystal clock frequency is and tell the UART what baud rate you want. In the file <code>trtSettings.h</code> you will want to modify:</p>
<pre>#define MAXNBRTASKS 4<br>#define MAXNBRSEMAPHORES 6</pre>
<pre>#define PRESCALER 1024                     // the actual prescalar value<br>#define PRESCALEBITS 5                     // the bits to be set in TCCR1b 
#define F_CPU 20000000UL                   // crystal frequency in Hz<br>#define TICKSPERSECOND F_CPU/PRESCALER     //The CRYSTAL frequency/prescalar

/* UART baud rate */<br>#define UART_BAUD  9600</pre>
<p>Each task should be written as a void function with one parameter. See the
  example code. </p>
<p>The <code>(deadline) - (release time) </code>should be greater
  than than the execution time of the thread between <code>trtSleepUntil</code> calls
  so that the kernel can meet all the deadlines. If you give a slow task a release
  time equal to it's deadline, then it has to execute in <em>zero time</em> to
  meet deadline, and nothing else can run until the slow task completes. Experiment
  with <a href="spoiler/TwoTasks_plus_spoiler.c">this test code</a> by changing
  the difference in the <code>spoiler</code> task while watching A.0 on the scope. </p>
<p><font color="#FF0000">Do 
      NOT modify any registers affecting timer1</font>! <br>
  If the tick time is critical 
for some aspect of your code (e.g. realtime clock), DO NOT USE the TRT timer 
tick as a reference. It is not accurate enough.<br>
You can use the soft timers defined below. </p>
<hr>
<p><strong>Adding Soft Timers</strong></p>
<p>It is convienent to be able to define an arbitrary number of timers. An additional module allows you to define periodic or one-shot times of 0.1 millisecond to several second durations, and to signal a TRT semaphore when the timer completes. The timer API is: </p>
<table width="95%"  border="1">
  <tr>
    <td width="49%"><div align="center"><code>void trtInitTimer(void)</code></div></td>
    <td width="51%">Initializes timer0 to either 0.1, 1 or 10 mSec tick time as determined by the <code>TIMERTICK</code> macro value. Only 0.1, 1 and 10 MSec are supported. This function also sets up timer structs. The macro value <code>MAXNBRTIMERS</code> must be greater than or equal to the total number of timers you define. This function initializes timer0 and enables the timer0 compare-match ISR.</td>
  </tr>
  <tr>
    <td><div align="center"><code>void trtSetTimer(uint8_t timer_number, <br>
  uint16_t period, <br>
  uint8_t mode,<br>
  uint8_t sem) {</code></div></td>
    <td>Sets a timer denoted by <code>timer_number</code>. The mode can be either <code>PERIODIC</code> or <code>ONESHOT</code>. A semaphore value of zero disables signalling, otherwise a semaphore will be signalled when the timer times-out. This function does not start the timer running. Note that <code>timer_number</code> is one-based. The first timer used should be labeled one. </td>
  </tr>
  <tr>
    <td><div align="center"><code>void trtStartTimer(uint8_t timer_number)</code></div></td>
    <td>Starts a timer. </td>
  </tr>
  <tr>
    <td><div align="center"><code>void trtStopTimer(uint8_t timer_number)</code></div></td>
    <td>Stops a timer. </td>
  </tr>
  <tr>
    <td><div align="center"><code>void trtDisableTimer(uint8_t timer_number)</code></div></td>
    <td>Disables a timer until the next time <code>trtSetTimer</code> is called for that timer. </td>
  </tr>
  <tr>
    <td><div align="center"><code>uint8_t trtStatusTimer(uint8_t timer_number)</code></div></td>
    <td>Returns the status of the timer. Bit zero is run/stop (1/0), bit one is oneshot/periodic (1/0) and bit seven is enable/disable (1/0). </td>
  </tr>
  <tr>
    <td><div align="center"><code>uint16_t trtNumPeriods(uint8_t timer_number)</code></div></td>
    <td>Returns the number of times that the timer has timed-out since it was last set. </td>
  </tr>
</table>
<p>The <a href="Timers/trtTimers.c">timer module</a> includes two macros which need to be configured. <code>MAXNBRTIMERS</code> and <code>TIMERTICK</code> must be set by the user. The code assumes a 16 MHz crystal driving the MCU. Modification to <code>trtInitTimer</code> will be necessary for different clock frequencies. An <a href="Timers/TestTimersTRT.c">example program</a> sets up three timers and uses them to control some tasks. Note that this module uses timer0 for interval timing. All the real work happens in the compare-match ISR. <span class="red">If you use this module, do not change any timer0 registers.</span> The program assumes that  PORTC pins 1,2,3 and 4 are connected to  four LEDs (and 300 ohm resistors).</p>
<hr>
<p><strong>Adding a Mutex</strong> </p>
<p>A mutex is a binary semaphore with states <code>LOCKED</code> and <code>UNLOCKED</code>. A task can gain control of a resource by locking a mutex. The same task must unlock the mutex (unlike a semaphore which can be signaled by any task). A basic mutex facility was built on top of the existing TRT semaphores. Since a mutex is a special semaphore (and uses the semaphore signaling mechanism) , each mutex must be numbered not to conflict with another semaphore. The mutex API is: </p>
<table width="100%"  border="1">
  <tr>
    <td width="47%"><div align="center"><code>void trtInitMutex(void)</code></div></td>
    <td width="53%">Initializes a data structure with size <code>MAXNBRSEMAPHORES</code>. and sets the state of each entry to <code>UNLOCKED</code> and the owner to <code>NONE</code>. </td>
  </tr>
  <tr>
    <td><div align="center"><code>void trtCreateMutex(uint8_t mutex_number)</code></div></td>
    <td>Creats a mutex with identifer <code>mutex_number,</code> initial value <code>UNLOCKED</code> and owner to <code>NONE</code>. Since a mutex is a special case of a semaphore, be sure to configure <code>MAXNBRSEMAPHORES</code> in the kernel file to be big enough to hold all semaphores+mutexes. The semaphore identifer number is 1-based.<code></code></td>
  </tr>
  <tr>
    <td><div align="center"><code>void trtLockMutex(uint8_t mutex_number)</code></div></td>
    <td>Locks the mutex and sets the owner to the current task. </td>
  </tr>
  <tr>
    <td><div align="center"><code>void trtUnlockMutex(uint8_t mutex_number)</code></div></td>
    <td>Unlocks the mutex and sets the owner to <code>NONE</code>. </td>
  </tr>
  <tr>
    <td><div align="center"><code>uint8_t trtQueryMutex(uint8_t mutex_number)</code></div></td>
    <td>Returns the status: <code>LOCKED</code> or <code>UNLOCKED</code>. </td>
  </tr>
  <tr>
    <td><div align="center"><code>uint8_t trtOwnerMutex(uint8_t mutex_number)</code></div></td>
    <td>Returns a binary <code>true</code> if the current task is the owner. </td>
  </tr>
</table>
<p>The <a href="mutex/trtMutex.c">mutex module</a> and <a href="mutex/MutexTestTRT.c">an example</a> show how to use a mutex to control LEDs flashing. LEDs are on PORTC, and switches on PORTB. </p>
<hr>
<p><strong>Adding a trace facility</strong></p>
<p>A realtime trace facility was added to TRT to enable following which tasks are executing and which semaphores are signaled. There are also to user defined <em>events</em> available. The trace facility uses one port of the MCU to dump data to an oscilloscope. A simple 3-bit DAC is used to convert task number and semaphore number to two separate voltages for display. Each task number adds about 125 mV to the output, so that when task 2, for example, is executing the voltage output is 250 mV. The semaphore number is specified when you initialize a semaphore. The task number starts at zero for the null task, then each task has a number defined by the order in which the tasks were created. Either of the events may be used as a scope trigger or to display. The connections are shown below. <br>
<img src="Trace/Trace_port_sch.png" width="674" height="251"><br> 
Adding a trace facility requires a modified kernel and settings file. Download: </p>
<ul>
  <li><a href="Trace/trtkernel_1284_trace.c">trtkernel_1284_trace.c</a></li>
  <li><a href="Trace/trtSettings.h">trtSettings.h</a> (configured for 20 MHz crystal, prescalar of 1024, 4 tasks, 7 semaphores, PORTA used for trace) </li>
  <li><a href="Trace/testTRTtrace_1284.c">example</a> (a  version of Example 1 above with a trace event inserted in task 4) </li>
  <li><a href="example2/trtUart.c">trtUart.c</a>, <a href="example2/trtUart.h">trtUart.h</a> Includes semaphores to signal each received character and end of string. </li>
</ul>
<p>The trace port is chosen in <code>trtSettings.h</code>. Defining a trace port turns on the trace facility. Commenting out the definition disables trace. You must also define the data direction register for the trace port as shown below. </p>
<pre>// trace options
// use one port for realtime trace
// IF TRACE_PORT is undefined, then trace is turned off
#define TRACE_PORT PORTA
#define TRACE_DDR  DDRA
</pre>
<p>You can also choose to have a semaphore number held for only a few microceconds (<code>INSTANT_SEM</code>) or to be held through a context switch. This means that semaphore signals which force a context switch will result in longer lasting waveforms and that you can tell when a context switch occurs as a result of a signal. Comment out one of:</p>
<pre>//#define EXTEND_SEM
  #define INSTANT_SEM </pre>
<p>There are four macros which can be inserted anywhere in your code. If you disable the trace facility you must remove the macros you inserted in your code. </p>
<table width="51%"  border="1">
  <tr>
    <td width="44%"><div align="center"><code>TRACE_EVENT_A_ON</code></div></td>
    <td width="56%">Turns on bit 7 of the trace port </td>
  </tr>
  <tr>
    <td><div align="center"><code>TRACE_EVENT_A_OFF</code></div></td>
    <td>Turns off bit 7 of the trace port </td>
  </tr>
  <tr>
    <td><div align="center"><code>TRACE_EVENT_B_ON</code></div></td>
    <td>Turns on bit 3 of the trace port </td>
  </tr>
  <tr>
    <td><div align="center"><code>TRACE_EVENT_B_OFF</code></div></td>
    <td>Turns off bit 3 of the trace port </td>
  </tr>
</table>
<p>Three images show typical event sequences. The first image shows four tasks on the upper trace and <em>trace event A</em> on the lower trace. As can be seen in the example code, <em>trace event A</em> is turned on just before an <code>fprintf</code> in task 4 and turned off just after. The second image shows  four tasks on the upper trace and seven semaphores on the lower trace. The number 1 semaphore in the middle of the screen is triggered by a character from the UART. The number 2 semaphore just to the right is the signal that the UART string is complete (user pressed <code>enter</code>). The long period spent in task 4 is the same as in the first image. Near the end of the long duration task 4, semaphore 5 is signaled which causes task 3 to execute, then bounce back to task 4 when semaphore 4 is signaled.  The third image is similar, but with <code>EXTEND_SEM</code> turned on to show when context switches occure as a result of semaphore signals. <br>
  <img src="Trace/Trace_task_eventA.jpg"><img src="Trace/Trace_task_semaphore.jpg"><img src="Trace/Trace_task_semaphore_extend.jpg" width="377" height="280"></p>
<hr>
<p><strong>Adding a system dump facility</strong></p>
<p>A dump facility was added to TRT to enable debugging of timing errors and stack overflows. A set of low level routines read out the status of any task, semaphore or mutex. A set of printing routines uses the low level routines to print status information to the serial port. The task number parameter in the table below refers to the lexical order in which tasks are created in the program being tested. The first user task is labeled 1.</p>
<table width="100%" border="1">
  <tr>
    <td width="289"><code>uint8_t trtTaskState(char tsk) </code></td>
    <td width="630">Returns the state of a task. 0=terminated; 1=readyQ; 2=timerQ; 3=wait Sem 1; 4=wait Sem 2; etc</td>
  </tr>
  <tr>
    <td><code>uint16_t trtTaskStack(char tsk) </code></td>
    <td>Returns the stack pointer of a task.</td>
  </tr>
  <tr>
    <td><code>uint16_t trtTaskStackBottom(char tsk) </code></td>
    <td>Returns the stack minimum address of a task.</td>
  </tr>
  <tr>
    <td><code>uint32_t trtTaskRelease(char tsk)</code></td>
    <td>Returns the release time of a task in system ticks.</td>
  </tr>
  <tr>
    <td><code>uint32_t trtTaskDeadline(char tsk) </code></td>
    <td>Returns the deadline time of a taskin system ticks.</td>
  </tr>
  <tr>
    <td><code>uint16_t trtTaskStackFreeMin(char tsk)</code></td>
    <td>Computes and returns the minimum  space left on the task stack since
      the task was defined. If this value reaches zero for any task, the system
      crashes! When created, each task initializes 35 bytes of storage for registers.</td>
  </tr>
  <tr>
    <td><code>uint8_t trtSemValue(uint8_t sem)</code></td>
    <td>Returns the value of a semaphore.</td>
  </tr>
  <tr>
    <td><code>uint8_t trtMutexOwner(uint8_t mut)</code></td>
    <td>Returns the owner of a mutex.</td>
  </tr>
  <tr>
    <td><code>uint8_t trtMutexState(uint8_t mut)</code></td>
    <td>Returns the state (locked/unlocked) of a mutex.</td>
  </tr>
  <tr>
    <td><code>void trtTaskDump(char tsk, char freeze_timers)</code></td>
    <td>If <code>tsk</code> is zero, prints all task information. If <code>tsk</code> is not zero, prints only the info for that task number.The task number,  state,     release time (relative to current system time), deadline time (relative to current system time), current free stack space and minimum free task space are printed. <code>Freeze_timers</code> can take the values <code>FREEZE_TIMER</code> and <code>RUN_TIMER</code>. Freezing the timers stops them for the duration of the print operation. This routine can use as much as 90 bytes of stack space for the calling task!</td>
  </tr>
  <tr>
    <td><code>void trtSemDump(char sem, char freeze_timers)</code></td>
    <td>If <code>sem</code> is zero, prints all semaphore information. If <code>sem</code> is not zero, prints only the info for that semaphore number. The semaphore number and value are printed. <code>Freeze_timers</code> can take the values <code>FREEZE_TIMER</code> and <code>RUN_TIMER</code>. Freezing the timers stops them for the duration of the print operation. This routine can use as much as 90 bytes of stack space for the calling task!</td>
  </tr>
  <tr>
    <td><code>void trtMutexDump(char mut, char freeze_timers)</code></td>
    <td>If <code>mut</code> is zero, prints all mutex information. If <code>mut</code> is not zero, prints only the info for that mutex number. The mutex number. owner and state are printed. <code>Freeze_timers</code> can take the values <code>FREEZE_TIMER</code> and <code>RUN_TIMER</code>. Freezing the timers stops them for the duration of the print operation. This routine can use as much as 90 bytes of stack space for the calling task!</td>
  </tr>
</table>
A example output is shown below. Three tasks, semaphore 5 and mutex 3 are printed.
The negative times indicate that the deadline is past for task 2. Two of the
task stacks have (at some point during program execution) been less than 10 bytes,
which is about as close as you want to be to zero. The output is from the example
code below.
<pre># state     relT   deadT stkfree stkfreeMin
1    2     1133     1133     8     8
2    1      -12      -12   106    19
3    2     2462     2462    87     9
# value
5    1
# owner UNLOCK
3    0     1 </pre>
<p>Adding a dump facility requires a modified kernel and settings file. The following includes both the trace and dump facilities. Download: </p>
<ul>
  <li><a href="Dump/trtkernel_1284_trace_dump.c">trtkernel_1284_trace_dump.c</a></li>
  <li><a href="Trace/trtSettings.h">trtSettings.h</a> (configured for 20 MHz crystal, prescalar of 1024, 4 tasks, 7 semaphores, PORTA used for trace) </li>
  <li><a href="Dump/trtQuery.c">trtQuery.c</a> (The routines explained in the table above)</li>
  <li><a href="Dump/QueryTestTRT_1284.c">example</a> (Three tasks which blinnk leds and respond to a variety of buttons) </li>
  <li><a href="example2/trtUart.c">trtUart.c</a>, <a href="example2/trtUart.h">trtUart.h</a> Includes semaphores to signal each received character and end of string. </li>
</ul>
<hr>
<p>Simple command shell:</p>
<p>A simple command shell was written as a task which can be signaled (entered)
  from any other task and which supplies a command-line interface to task values,
  semaphore values, memory contents, and i/o registers. The <a href="Shell/test1_command_TRT_1284.c">example</a> assumes
  at buttons are connected to PINB, LEDs
  to PORTC, and the D0 and D1 to the uart. The <a href="Dump/trtkernel_1284_trace_dump.c">trtkernel_1284_trace_dump.c </a> kernel
  and <a href="Dump/trtQuery.c">trtQuery.c</a> are required as above. Pushing
  button 2 or 3 will enter the shell with different IDs.</p>
<p>Shell commands:</p>
<table width="90%" border="1">
  <tr>
    <td width="36%"><div align="center"><code>g</code></div></td>
    <td width="64%">Exit command shell and run other tasks</td>
  </tr>
  <tr>
    <td width="36%"><div align="center"><code>signal_shell(char ID) </code></div></td>
    <td width="64%">When used in other tasks, this macro enters the command shell
      and prints the specified ID. Timers are frozen and interrupts disabled.<br>
      Example: <code>signal_shell(4)</code></td>
  </tr>
  <tr>
    <td width="36%"><div align="center"><code>x</code></div></td>
    <td width="64%">Forces a <em>RESET</em> of the MCU and trashes the state of your program.</td>
  </tr>
  <tr>
    <td width="36%"><div align="center"><code>i ioregAddress</code></div></td>
    <td width="64%">Read an i/o register. The register address is entered in
      hexadecimal, with  space delimiters. The address used must be the value
      in parenthesis in the table (see data sheet in the range 0x20 to 0xff
        ). The result displayed in in
      hex. <br>
      Example:<code> i 23</code><br>
      Reads the state of the pushbuttons attached to PINB.</td>
  </tr>
  <tr>
    <td width="36%"><div align="center"><code>I ioregAddress  iodata</code></div></td>
    <td width="64%"><p>Write to an i/o register. The register address and data
        are entered in hexadecimal , with  space delimiters. The address used
        must be the value in parenthesis in the table (see data sheet in the
        range 0x20 to 0xff ). <br>
      Example: <code>I 28 f0</code><br>
      Turns on LEDs 0 to 3 attached to PORTC.</p></td>
  </tr>
  <tr>
    <td width="36%" height="132"><div align="center"><code>t tasknumber </code></div></td>
    <td width="64%" height="132"><p>Gets information about a task. The first
        task has index 1. <code> </code>Information returned for eack task is:</p>
      <ul>
        <li>state (0=terminated, 1=readyQ, 2=timeQ,
        3=waiting for Sem1, 4=waiting for Sem2, etc.)</li>
        <li>release time relative to current time in system ticks</li>
        <li>deadline time relative to current time in system ticks</li>
        <li>Current stack space available in bytes</li>
        <li>Minimum stack space not used
          </li>
<br>
      </ul>
        Example:<code> t 4 </code>returns information on task 4.<br>
    Example:<code> t 0 </code>returns information on all tasks.    </td>
  </tr>
  <tr>
    <td width="36%" height="124"><div align="center"><code>s semaphorenumber </code></div></td>
    <td width="64%"><p>Gets information about a semaphore. The first semaphore
        has index 1. <code> </code>Information returned for each semaphore is
        the value.</p>
Example:<code> s 4 </code>returns information on semaphore 4.<br>
Example:<code> s 0 </code>returns information on all semaphores</td>
  </tr>
  <tr>
    <td height="124"><div align="center"><code>m memAddress</code></div></td>
    <td>Reads SRAM. The memory address is entered in hexadecimal,
      with space delimiters. Addresses <code>0x00-0x1f</code> are the cpu data registers,
      <code>0x20-0xff</code> are i/o registers, and <code>0x100-0x10ff</code> is actual RAM. The result
      displayed in in hex. You can find out where global variables are stored
      by opening the <code>.map</code> file and searching for the word <code>common</code> three times.<br>
Example:<code> m 1f</code><br>
Reads cpu register 31.</td>
  </tr>
  <tr>
    <td height="124"><div align="center"><code>M memAddress data</code></div></td>
    <td>Write to an SRAM address. The memory address and data are entered in
      hexadecimal , with space delimiters. Addresses <code>0x00-0x1f</code> are
      the cpu data registers, <code>0x20-0xff</code> are i/o registers, and <code>0x100-0x10ff</code> is
      actual RAM. You
      can find out where global variables are stored by opening the <code>.map</code> file
      and searching for the word <code>common</code> three times.<br>
Example: <code>I 28 f0</code><br>
Turns on LEDs 0 to 3 attached to PORTC.</td>
  </tr>
</table>
<hr>
<p><font size="-1">Copyright Cornell University </font>
  <!-- #BeginDate format:Am1 -->October 18, 2013<!-- #EndDate -->
</p>
<p>&nbsp; </p>
</body>
</html>
