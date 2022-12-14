# Assignment 8 Part 5


##1. 

Our first question focuses on `main-two-cvs-while.c` (the working solution). First, study the code. Do you think you have an understanding of what should happen when you run the program?
    - Yes. This is a further fleshed out, working example of the figures from the chapter. 

##2. 

Now run with one producer and one consumer, and have the producer produce a few values. Start with a buffer of size 1, and then increase it. How does the behavior of the code change when the buffer is larger? (or does it?) What would you predict num full to be with different buffer sizes (e.g., `-m 10`) and different numbers of produced items (e.g., `-l 100`), when you change the consumer sleep string from default (no sleep) to `-C 0,0,0,0,0,0,1`?
    - The behaviour changes in that the producer can add multiple values to the buffer (up to the length of the buffer -1, depending on context switches) before it waits, and vice versa for the consumer. 
    - My prediction for `num_full` is that it would fluctuate between `max` and `0` for smaller buffer sizes. For larger sizes it might not hit the extrema much, or at all. And that proves to be the case with `-p 1 -c 1 -m 10 -l 100 -v`
    - Assuming a buffer size > 1 and `-C 0,0,0,0,0,0,1`, the producer will fill the buffer, then the consumer will run once, and sleep for 1 second. Since there is only one empty index, the producer will run once, and until the loops are over for each they will take turns to add/remove one at a time.

##3. 

If possible, run the code on different systems (e.g., a Mac and Linux). Do you see different behavior across these systems?
    - Yes. On linux, running in a VirtualBox with 2 processors, the result is almost as if there is just one processor. And my answers above hold. However on my Mac, with 4 processors, the producer and consumer threads interleave far, far more, and keep pace with each other for arguments like `-p 1 -c 1 -m 10 -l 100 -v`. Here's a side by side visual of the output, with Linux on the left, and Mac on the right: 

![Linux vs Mac](https://github.com/martalist/ostep/raw/master/chapter_30/hw30/images/lvsm.png)

    - Curiously, though, the Mac reports a slower total time with the `-t` flag. For this set of flags `-p 1 -c 1 -m 10 -l 10000 -v -t` the Mac runs in ~0.5s, while linux is done in ~0.15s. Quite a difference! ... perhaps a result of the constant lock contention? 

    - For a buffer size > 1 and  `-C 0,0,0,0,0,0,1`, the behaviour is very much the same. 

##4. 

Let???s look at some timings of different runs. How long do you think the following execution, with one producer, three consumers, a single-entry shared buffer, and each consumer pausing at point `c3` for a second, will take?
     `prompt> ./main-two-cvs-while -p 1 -c 3 -m 1 -C 0,0,0,1,0,0,0:0,0,0,1,0,0,0:0,0,0,1,0,0,0 -l 10 -v -t`
    - It's not immediately clear if the consumer thread that sleeps at `c3` holds the lock. If so, and they sleep whilst holding the lock, then the result could be 10 loops x 3 consumers for each wait. If the lock isn't held, then the result ought to be closer to 10 seconds, since each consumer can sleep concurrently. 
    - *Answer*: It turns out that the answer is the latter - the execution time is ~12s.

##5. 

Now change the size of the shared buffer to 3 (`-m 3`).Will this make any difference in the total time?
    - No, it shouldn't. At least not by much. The producer ought be done sooner, but the consumers will each need to wait the full second on each loop. 
    - *Answer*: My guess was about right. The execution time is about the same, erring on the side of a fraction less. 

##6. 

Now change the location of the sleep to `c6` (this models a consumer taking something off the queue and then doing something with it for a while), again using a single-entry buffer. What time do you predict in this case?
     `prompt> ./main-two-cvs-while -p 1 -c 3 -m 1 -C 0,0,0,0,0,0,1:0,0,0,0,0,0,1:0,0,0,0,0,0,1 -l 10 -v -t`
    - So, after completing the prior questions, the idea here is that a consumer thread will suspend execution for 1 second after releasing the lock for the critical section, thus mimicking doing some useful work the value taken off the buffer. 
    - There should therefore be some speed up, since contention between the 3 consumers for the lock will be reduced. The execution should follow some order, like: 
        - p adds 1 item
        - c0 removes, and is busy 1s
        - p adds 1 item
        - c1 removes, and is busy 1s
        - p adds 1 item
        - c2 removes, and is busy 1s
        - p adds 1 item
        - wait for c0
        - c0 removes, and is busy 1s
        - ... 
    - So the answer should be around  `number_of_loops / consumers * 1 second`; ~3.5s
    - *Answer*: I was close. It's ~5 sec, on both Mac and Ubuntu. Since `10 % 3 != 0`, the remaining consumers will complete the consumer function to reach an `EOS`, adding 2 extra seconds.

##7.

Finally, change the buffer size to 3 again (`-m 3`). What time do you predict now?
    - The producer will race ahead of the consumers, but the bottleneck here is still the consumers. So expect the execution time to be much the same as above. 
    - *Answer*: Looks like I got this one right. 

##8.

Now let???s look at `main-one-cv-while.c`. Can you configure a sleep string, assuming a single producer, one consumer, and a buffer of size 1, to cause a problem with this code?
    - I don't think that we can run a sleep string that causes catastrophe in this case. There is only one cv, and that is potential for dissaster if a consumer wakes an consumer (or producer wakes a producer), but since we only have two threads we can't have any incorrect awakenings. i.e. in this case a producer can only wake a consumer, and vice versa. 

##9. 

Now change the number of consumers to two. Can you construct sleep strings for the producer and the consumers so as to cause a problem in the code?
    - Yes: `./main-one-cv-while -p 1 -c 2 -m 1 -l 1 -P 0,0,0,0,0,0,1`
    - The key here is to have one of the two consumers wake the other when the buffer is empty, so that the woken thread will call `Cond_wait()` and sleep forever. The easiest way to do that is to make sure the producer is not waiting for the cv when one consumer calls the other.

##10. 

Now examine `main-two-cvs-if.c`. Can you cause a problem to happen in this code? Again consider the case where there is only one consumer, and then the case where there is more than one.
    - With one consumer: As above, since we have just one producer and one consumer this code works.
    - With two consumers: `./main-two-cvs-if -p 1 -c 2 -m 1 -l 2 -P 1 -C 1:0,0,0,1 -v | less` produces an empty buffer error by allowing c1 to execute 1st, which waits for a signal. The producer ought to run next, then c0 runs to consume the only value in the buffer, followed by c1. Since c1 is only in an `if` block, not a `while` block, c1 attemps to consume from an empty buffer: 

![c1 tries to get from an empty buffer](https://github.com/martalist/ostep/raw/master/chapter_30/hw30/images/empty_buffer.png)

##11. 

Finally, examine `main-two-cvs-while-extra-unlock.c`. What problem arises when you release the lock before doing a put or a get? Can you reliably cause such a problem to happen, given the sleep strings? What bad thing can happen?
    - We remove the mutual exclusion for the critical sections within `get` and `put`. Namely, there's a race condition for shared memory (`buffer`, `fill_ptr`, `use_ptr`, and `num_full`), when two consumers or two producers are executing.
    - I haven't been able to get this to occur reliably. Since there's no way to pause execution within the `get()` or `put()` I'm not sure how to manipulate the sleep strings to get the conditions just right.
