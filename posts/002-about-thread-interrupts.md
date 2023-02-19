# About thread interrupts

## When does InterruptedException happen?

- A thread is blocking on IO (waiting for something outside of its control to happen)
  - This could be: waiting for a network response to come back, waiting for another thread to release a lock, waiting
    for file I/O, etc
- The thread is "told" by another thread to "at your earliest convenience, please stop whatever you are doing, clean
  up, and safely die"

## The thread interrupt mechanism

- A Thread `t1` can be interrupted by calling `t1.interrupt()`
- This basically just sets a flag on the Thread's state, then "informs VM of interrupt"
- If the Thread is blocked on IO (implying `interrupt()` was called by a different thread), the VM will wake up `t1`
- All lowest-level blocking methods in the JDK are written to check the thread interrupt status upon waking up
- If the interrupt status is set, they will clear it (more on this later), then throw `InterruptedException`
  - This "read and clear" is actually one method: [Thread.interrupted()](https://docs.oracle.com/en/java/javase/19/docs/api/java.base/java/lang/Thread.html#interrupted())

TODO: Probably could benefit from some examples of stack traces showing the typical "lowest-level" methods that threads
block on

## Can a thread be interrupted without it throwing InterruptedException?

- Yes! If it is not blocked when another thread interrupts it, then it will not immediately know that it has been
  interrupted
- But, remember that a flag was set - if the interrupted thread later calls a blocking method, it will immediately throw
  `InterruptedException`
- If a thread might not call a blocking method for a long time, ie the thread is doing "CPU-bound" work, it may make
  sense for that CPU-bound work to include some occasional checks of its thread's interrupted status, so that its thread
  can remain responsive to interrupts

## What to do when we need to call a method that may throw InterruptedException?
- First rule: If an interrupt happens,
  - **Do NOT, under any circumstances, allow the caller to exit (including exceptionally) without "remembering" that the
    interrupt happened!**
    - One easy way to mess up is by using `catch (Exception e)` and never checking whether the exception was
      `InterruptedException`
  - An interrupt is supposed to put us in "clean up and exit" mode, not "recover" mode - can't clean up and exit if we
    forget that's what we're doing
  - "Remembering" means either (re-)throwing an `InterruptedException`, or (re-)setting the thread interrupt status
- Now ask: Does the caller have any cleanup to do before it exits?
  - Cleanup may include: releasing locks, ensuring any shared data is in a reasonable state, interrupting any threads
    that *it* spawned…
    - This "interrupting any threads that *it* spawned" (along with, ideally, waiting for them to complete) forms the
      basis of "structured concurrency"
    - Not to go too deep, but "structured concurrency" is about making concurrent behavior easier to reason about, by
      tying it to lexical scope
      - "when I exit this block, all concurrent tasks I spawned WILL be done" - a powerful guarantee
  - If there is cleanup to do, we should catch the `InterruptedException`, and perform the cleanup before exiting
- How should we exit?
  - If it is sensible for the caller to return normally at this point:
    - Then do that (after any necessary cleanup), BUT at least call `Thread.currentThread().interrupt()` first
    - Why is this call needed to remember that the interrupt happened?
    - When the lowest-level method throws `InterruptedException`, by convention it also clears the thread interrupt
      status
      - This was probably designed so that, upon catching the `InterruptedException`, any necessary blocking calls that
        cleanup code needs to make can be made without them just throwing `InterruptedException` again
      - eg if we need to make a blocking call to acquire a lock, so that we can clean up some shared data
      - Remember: After we catch an `InterruptedException` (or read an interrupted status), the thread should be on a
        "cleanup" path, NOT a "recover" path
    - In our caller, re-setting the interrupt status before returning ensures that the thread remembers it was
      interrupted after it returns
      - So, if it later calls another blocking method, another `InterruptedException` will be thrown
      - Or, if it happens to have CPU-bound work to do after this, it should be checking the thread interrupt status
        periodically, so it should see the interrupt and respond accordingly
      - In other words, after that first interrupt, the thread STAYS on a "quickly and safely die" path, until the
        thread actually dies / exits its task
      - Therefore, a thread should never need to be externally interrupted more than once (though it may re-interrupt
        itself several times while cleaning up)
  - If it is NOT sensible for the caller to return normally at this point:
    - Then don't - just propagate the `InterruptedException` (will need to declare `throws InterruptedException` on our
      caller)
    - If there is cleanup to do, then catch the `InterruptedException`, do cleanup, then re-throw
      - Reminder: do not allow the caller to exit without remembering that the interrupt happened - this includes via
        exceptions in cleanup code
    - If not, no need to catch
    - What if we can't declare `throws InterruptedException` on the caller (ie we need to conform to some contract)?
      - This is the trickiest case
      - Ask again if there's any sensible way for the caller to return normally (with the interrupt status set) at
        this point
      - If not, the safest thing to do is probably to throw some other unchecked exception, but set the thread
        interrupt status first
