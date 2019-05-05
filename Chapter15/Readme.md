1. In the program shown in Figure 15.6, remove the `close` right before the
   `waitpid` at the end of the parent code. Explain what happens.

   When the child reaches the end of the document, it blocks on a call to
   `read` from the pipe. Since the write-end of the pipe isn't closed in the
   parent, the child will block on the `read` forever.

2. In the program in Figure 15.6, remove the `waitpid` at the end of the parent
   code. Explain what happens.

   The parent reads the file and writes the content to the pipe.  Assuming that
   the size of the file is less than the size of pipe buffer, the parent will
   write the entire file to the buffer, then terminates immediately.

   The behavior of the child depends on the program.  If, for instance, I do:

   ```
   PAGER=/bin/cat ./a.out /etc/services
   ```

   The the content of `/etc/services` fits in the pipe buffer, and `/bin/cat`
   reads from the pipe and writes everything to standard output (the terminal's
   character device) before the parent's exit.

   If I instead do:

   ```
   PAGER=/bin/more ./a.out /etc/services
   ```

   Then I see only the first page of output.  I think that what's happening
   is the parent wrote the entire content of the file to the pipe, and the child
   read the first page and printed it.  Then when the parent terminated, it
   triggered an `exit_group`, which resuted in the child getting terminated.  I
   base this on `man 2 exit_group`:


   > NOTES  
   > &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Since
   > glibc  2.3,  this is the system call invoked when the _exit(2) wrapper
   > function is called.

   And that the `exit` function called by `main` will eventually call `_exit`.

   Finally, if I do:

   ```
   PAGER=/usr/bin/less ./a.out /etc/services
   ```

   Then I don't see any of the output, but I do sometimes see the screen
   "flash".  My guess is that `less` (1) grabs and saves the content of the
   terminal display, (2) renders content to standard output, and (3) restores
   the originally saved display content on exit.  I think the "flash" that
   I see is what was written to standard output, followed immediately by the
   on-exit restoration.

3. What happens if the argument to `popen` is a nonexistent command? Write a
   small program to test this.

   Here's the program (see also `exercise_3.c`):

   ```c
   #include <stdio.h>
   
   int
   main(void)
   {
   	FILE* const file = popen("does-not-exist", "r");
   
   	if (file == NULL) {
   		perror("popen");
   		return 1;
   	}
   
   	pclose(file);
   
   	return 0;
   }
   ```

   The output of running this program is:

   ```
   $ ./a.out
   sh: does-not-exist: command not found
   ```

   The call to `popen` successfully starts `/bin/sh`, which tries to invoke
   the non-existant program.  That fails, so `/bin/sh` writes the error
   message to standard error and terminates.

   Since `open` successfully started the shell, it did not fail.

4. In the program shown in Figure 15.18, remove the signal handler, execute the
   program, and then terminate the child. After entering a line of input, how
   can you tell that the parent was terminated by `SIGPIPE`?

   See `exercise_4.c`.

   Running the program I see:

   ```
   $ ./a.out
   1 1
   2
   ```

   Next, I terminate the `add2` process and continue to provide input to the
   parent process.  The parent terminates immediately:

   ```
   2 2
   $
   ```

   I examine the exit status of the process:

   ```
   $ echo $?
   141
   ```

   My shell is `bash`, and according to `man bash`:

   > The return value of a simple command is its exit status, or 128+n if the
   > command is terminated by signal n.

   So `141 - 128 = 13`.  What is signal number 13?

   ```
   $ kill -l 13
   PIPE
   ```

   This shows that the parent process was terminated by a `SIGPIPE` signal.

5. In the program in Figure 15.18, use the standard I/O library for reading
   and writing the pipes instead of `read` and `write`.

   See `exercise_5.c`; I used `fdopen` to create `FILE` streams assocaited
   with the read and write ends of the pipes, and used `fprintf` and `fgets`
   to write to and read from those streams.

6. The Rationale for POSIX.1 gives as one of the reasons for adding the
   `waitpid` function that most pre-POSIX.1 systems can't handle the following:

   ```c
   if ((fp = popen("/bin/true", "r")) == NULL)
   	...
   if ((rc = system("sleep 100")) == -1)
   	...
   if (pclose(fp) == -1)
   	...
   ```

   What happens in this code if `waitpid` isn’t available and `wait` is used
   instead?

   Without `waitpid`, the implementation of `system` would call `wait` to
   wait for the command to terminate.  If the `popen` failed, then the call
   to `system` would return almost immediately --- before the command that it
   executed completes -- and would return the exit status of the `popen`
   command.
   
   Similarly, the `popen` would also call `wait`, and would block waiting for
   `system`'s child to terminate (if it hadn't already).  It would return the
   exit status of `system`'s child.

7. Explain how `select` and `poll` handle an input descriptor that is a pipe,
   when the pipe is closed by the writer. To determine the answer, write two
   small test programs: one using `select` and one using `poll`.

   Redo this exercise, looking at an output descriptor that is a pipe, when the
   read end is closed.

   1. `select` on read-end  
      If a process is blocked on a call to `select` with the read-end of a
      pipe in the `readfds` set, then when the last process that had the
      write-end open closes that file descriptor, `select` will return the
      read-end in `readfds`.

      Sample output of `exercise_7a.c`:

      ```
      $ ./a.out
      fdcount: 1
      readfds set:   1
      writefds set:  0
      exceptfds set: 0
      ```

   2. `select` on write-end  
      If a process is blocked on a call to `select` with the write-end of a
      pipe in the `writefds` set, then when the last process that had the
      read-end open closes that file descriptor, `select` will return the
      write-end in `writefds`.

      Sample output of `exercise_7b.c`:

      ```
      $ ./a.out
      fdcount: 1
      readfds set:   0
      writefds set:  1
      exceptfds set: 0
      ```

   3. `poll` on read-end  
      If a process is blocked on a call to `poll` with the read-end of a
      pipe in a `struct pollfd` marked with the `POLLIN` event, then when
      the last process that had the write-end closes the file descriptor,
      `poll` will return with `POLLHUP` set in the `revents` field of the
      corresponding `struct pollfd`.

      Sample output of `exercise_7c.c`:

      ```
      $ ./a.out
      fdcount: 1
      revents: POLLHUP
      ```

   4. `poll` on write-end  
      If a process calls `poll` with the write-end of a pipe in a
      `struct pollfd` marked with the `POLLOUT` event, if write end of the
      pipe isn't closed and the pipe isn't full (i.e., if data can successfully
      be written to the pipe), then `poll` returns immediately with `POLLOUT`.

      If a process calls `poll` _after_ the write-end of the pipe is closed,
      then `poll` will return immediately with `POLLOUT` and `POLLERR`.
      
      Sample output of `exercise_7d.c`:

      ```
      $ ./a.out
      fdcount: 1
      revents: POLLOUT POLLERR
      ```

8. What happens if the _cmdstring_ executed by `popen` with a _type_ of `"r"`
   writes to its standard error?

   See `exercise_8.c`.  The `popen` function sets up a pipe and forks to run
   the given command.  With the `"r"` type, it set the pipe up to redirect the
   standard output of the given command to the pipe -- it does not modify
   standard error.  Therefore, the child inherits the standard error file
   descriptor from the calling process.

9. Since `popen` invokes a shell to execute its _cmdstring_ argument, what
   happens when `cmdstring` terminates? (Hint: Draw all the processes involved.)

10. POSIX.1 specifically states that opening a FIFO for read–write is undefined.
    Although most UNIX systems allow this, show another method for opening a
    FIFO for both reading and writing, without blocking.

11. Unless a file contains sensitive or confidential data, allowing other users
    to read the file causes no harm. (It is usually considered antisocial,
    however, to go snooping around in other people's files.) But what happens
    if a malicious process reads a message from a message queue that is being
    used by a server and several clients? What information does the malicious
    process need to know to read the message queue?

12. Write a program that does the following. Execute a loop five times: create
    a message queue, print the queue identifier, delete the message queue. Then
    execute the next loop five times: create a message queue with a key of
    `IPC_PRIVATE`, and place a message on the queue. After the program
    terminates, look at the message queues using `ipcs(1)`. Explain what is
    happening with the queue identifiers.

13. Describe how to build a linked list of data objects in a shared memory
    segment. What would you store as the list pointers?

14. Draw a timeline of the program in Figure 15.33 showing the value of the
    variable `i` in both the parent and child, the value of the long integer
    in the shared memory region, and the value returned by the `update`
    function.  Assume that the child runs first after the `fork`.

15. Redo the program in Figure 15.33 using the XSI shared memory functions from
    Section 15.9 instead of the shared memory-mapped region.

16. Redo the program in Figure 15.33 using the XSI semaphore functions from
    Section 15.8 to alternate between the parent and the child.

17. Redo the program in Figure 15.33 using advisory record locking to alternate
    between the parent and the child.

18. Redo the program in Figure 15.33 using the POSIX semaphore functions from
    Section 15.10 to alternate between the parent and the child.