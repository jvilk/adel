# adel
A new way to program microcontrollers

Adel is a library that makes it easier to program microcontrollers, such as the Arduino. The main idea is to provide a simple kind of concurrency, similar to coroutines. Using Adel, any function can be made to behave in an asynchronous way, allowing it to run concurrently with other Adel functions without interfering with them. The library is implemented entirely as a set of C/C++ macros, so it requires no new compiler tools or flags. Just download the Adel directory and install it in your Arduino IDE libraries folder.

## Overview and examples

Adel came out of my frustration with microcontroller programming. In particular, that seemingly simple behavior can be very hard to implement. As an example, consider a function that blinks an LED attached to some pin every N milliseconds:

    void blink(int some_pin, int N) {
       digitalWrite(some_pin, HIGH);
       delay(N);
       digitalWrite(some_pin, LOW);
       delay(N);
    }

OK, that's easy enough. I can call it with two different values, say 500ms and 300ms:

    for (int i = 0; i < 100; i++) blink(3, 500);
    for (int i = 0; i < 100; i++) blink(4, 300);

But what if I want to blink them **at the same time**? Now, suddenly, I have lost composability. This code doesn't work:

    for (int i = 0; i < 100; i++) {
      blink(3, 500);
      blink(4, 300);
    }

To get **concurrent** behavior I have to write completely different code, which makes the scheduling explicit:

    int last300, last500;
    loop() {
      uint32_t t = millis();
      if (t - last300 > 300) {
        if (pin3_on) digitalWrite(3, LOW);
        else digitalWrite(3, HIGH);
        last300 = t;
      }
      if (t - last500 > 500) {
        ... etc ...
     }

Aside from the obvious complexity of this code, there are a couple of specific problems. First, all behaviors that might occur concurrently must be part of the same loop with the same timing control. The modularity is completely gone. Second, we need to introduce a global variable for each behavior that remembers the last time it executed. The code would become significantly more complex if we wanted to blink the lights for a specific amount of time, or if we had other modes where the lights are not blinking. Similar problems arise with input as well. Imagine if we want to blink a light until a button is pressed (inluding debouncing the button signal). 

The central problem is the `delay()` function, which makes timing easy for individual behaviors, but blocks the whole processor. The key feature of Adel, therefore, is an asynchronous delay function called `adelay` (hence the name Adel). The `adelay` function works just like `delay`, but allows other code to run concurrently. 

Concurrency in Adel is specified at the function granularity, using a fork-join style of parallelism. Functions are designated as "Adel functions" by defining them in a stylized way. The body of the function can use any of the Adel library routines shown below:

* `adelay( T )` : asynchronously delay this function for T milliseconds.
* `andthen( f )` : run Adel function f to completion before continuing.
* `aforatmost( T, f )` : run Adel function f until it completes, or T milliseconds (whichever comes first)
* `aboth( f , g )` : run Adel functions f and g concurrently until they **both** finish.
* `adountil( f , g )` : run Adel function f until g finishes
* `auntileither( f , g ) { ... } else { ... }` : run Adel functions f and g concurrently until **one** of them finishes. Executes the true branch if f finishes first or the false branch if g finishes first.
* `afinish` : finish executing the current function
* `ayield` : pause this function and return to the caller (see aforevery)
* `aforevery( f ) { ... }` : run f continuously, and execute the code in the block each time it yields.

Using these routines we can rewrite the blink routine:

    adel blink(int some_pin, int N) 
    {
      abegin;
      asteps:
      while (1) {
        digitalWrite(some_pin, HIGH);
        adelay(N);
        digitalWrite(some_pin, LOW);
        adelay(N);
      }
      aend;
    }

Every Adel function contains a minimum of three things: return type `adel`, and macros `abegin`, `asteps:`, and `aend` at the begining and end of the function. But otherwise, the code is almost identical. The key feature is that we can run blink concurrently, like this:

    aboth( blink(3, 500), blink(4, 500) );

This code does exactly what we want: it blinks the two lights at different intervals at the same time. We can aluse the `auntileither` macro to call two routines and check which one finished first. This macro is useful for things like timeouts:

    adel timeout(int ms) {
      abegin;
      asteps:
      adelay(ms);
      aend;
    }
    
    adel button_or_timeout() {
      abegin;
      asteps:
      auntileither( button(9), timeout(2000) ) {
        // -- Button was pushed (button() finished fist)
      } else {
        // -- Timeout finished first
      }
      aend;
    }

Notice that the `auntileither` macro is set up to look like a control structure, which allows it to have arbitrary code for handling to two cases (which routine finished first).

Here is the `button()` function, which returns if the user pushes a button:

    adel button(int pin)
    {
      abegin;
      asteps:
      // -- Wait for a high signal
      while (digitalRead(pin) != HIGH) {
        adelay(20);
      }
      
      // -- Wait 50ms and then check again
      adelay(50);
      if (digitalRead(pin) == HIGH) {
        // -- Still pressed, wait until it is released
        while (digitalRead(pin) != LOW) {
          adelay(20);
        }
      }
      aend;
    }

## Local variables

One of the challenges in Adel is supporting local variables. From the standpoint of the C runtime, control enters and exits each function many times before it reaches the Adel finish, which would cause the local variables to disappear and lose their values. The latest version of Adel allows the user to declare local variables between `abegin` and `asteps:`. Syntactically, these variables look like local variables, but they are secretly held in storage on the heap. These variables must be accessed througn the `my` macro, which hides some pointer junk.

    adel counter()
    {
      abegin;
        int i;
      asteps:
      aend;
    }



## WARNINGS

(1) Be very careful using `switch` and `break` inside Adel functions. The co-routine implementation encloses all function bodies in a giant switch statement to allow them to be reentrant. Adding other switch and break statements can have unpredictable results.

(2) The internal stack that keeps track of concurrent functions has a limited depth -- currently hard-wired to 6. Going beyond six levels deep will cause an array overflow.
