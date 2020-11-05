## Challenge 0: 'Hello There'

Hint: Processes run system calls

### Solution

Use `strace` to see the syscalls that are executed

```
openat(AT_FDCWD, ".hello_there", O_RDONLY) = -1 ENOENT (No such file or directory)
```

`openat` syscall is trying to open the non-existent file `.hello_there`.

So we create it so that the code can proceed.

`touch .hello_there`

## Challenge 1: 'Gatekeeper'

Hint: Stand guard, let noone pass

### Solution

The file `.hello_there` is allowed to be written so "the door is unlocked".
With an `strace` we can see the check that happens within the code:

```
openat(AT_FDCWD, ".hello_there", O_WRONLY) = 4
```

The `openat` syscall, tries to open .hello_there with `O_WRONLY` flags, and succeeds (returns fd = 4). For the challenge to be solved we need this syscall to fail.

So we can modify the permissions of the file to only allow reading and not writing on it.

`chmod -w .hello_there`

## Challenge 2: 'A time to kill'

Hint: Stuck in the dark, help me move on.

### Solution

By using `strace` we see that the last syscall run by the program, is `pause()`, along with `alarm(10)`. The `alarm(10)` will send a SIGALRM after 10 seconds to the app, waking it up by the `pause` that has put it into a `STOP`.

We need to wake it up before the alarm reaches our process.

One solution is: `CTRL-Z` (to suspend the program) and then `fg` to bring it back.
Another would be to just send a SIGCONT to the program by using the kill command.

## Challenge 3: 'what is the answer to life the universe and everything?'

Hint: ltrace

### Solution

By using `ltrace` we observe the usage of the `getenv("ANSWER")` function call. This call will retrieve the content of the ANSWER environment variable.

So we just need to fill this up with the answer to life, the universe and everything.

`ANSWER=42 ./riddle`

## Challenge 3: 'First-in, First-out'

Hint: Mirror, mirror on the wall, who in this land is fairest of all?

### Solution

First-in, First-out reminds us of pipes (FIFO pipe).

But let's investigate it:

1. By using `strace`, we find out that the program tries to open a file with the name "magic_mirror".
   ```
   openat(AT_FDCWD, "magic_mirror", O_RDWR) = -1 ENOENT (No such file or directory)
   ```
2. Let's provide the program with the file and see the result:

   ```
   touch magic_mirror
   strace ./riddle
   ```

   The program tries to write to the file a character and then proceeds to read a character from the same file. Only to find nothing.

   "I cannot see my reflection"

3. The hint for "reflection" as well as the `read()` from the same file descriptor as the `write()`, confirms our suspicions for a FIFO pipe.

So... we use the `mknod` command to create one with the name "magic_mirror"

```
mknod magic_mirror p
```

## Challenge 5: 'my favourite fd is 99'

Hint: when I bang my head against the wall it goes: dup! dup! dup!.

### Solution

We hear file descriptors... so that involves syscalls, so... let's run an `strace`!

We immediatelly stumble upon this line:

```
fcntl(99, F_GETFD)                      = -1 EBADF (Bad file descriptor)
```

This call tries to return the file descriptor's flags. But the file descriptor doesn't exist! So we need to create it somehow...

By searching (and using the hint), we find that the function call we need is `dup2`, to duplicate an existing file descriptor to a new one that has the number we want (99).

But, file descriptors are maintained into the process' context. So we need to create the new file descriptor into the same process (or a child of it). For this we'll create a small C program that creates the new fd, uses the `dup2` call to duplicate it and assign it to the 99 fd, and uses `execve` to start `./riddle` (to substitute the program's code with `./riddle` code).

```C
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

int main (int argc, const char* argv) {
  int fd = open(".hello_there", O_RDONLY);
  int res = dup2(fd, 99);
  char *newargv[] = { NULL };
  char *newenviron[] = { NULL };

  printf("%d\n", res);
  printf("Success\n");
  execve("./riddle", newargv, newenviron);
}
```

We compile and then run this program, and voila! It works ;)

## Challenge 6: 'ping pong'

Hint: 'help us play!'

### Solution

Well... Let's use our good ol' `strace` to inspect what is happening...

```
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7fc8402ada10) = 33732
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD[33732] PING!
, child_tidptr=0x7fc8402ada10) = 33733
wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 1}], 0, NULL) = 33732
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=33732, si_uid=1000, si_status=1, si_utime=0, si_stime=0} ---
wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 1}], 0, NULL) = 33733
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=33733, si_uid=1000, si_status=1, si_utime=0, si_stime=0} ---
```

Clone, child, SIGCHLD, wait... is our riddle spawning children?

Let's run an `ltrace` to investigate it further...

```
fork() = 33782
fork() = 33783
```

There is no doubt anymore. But our trusty tools, strace and ltrace don't follow the children's executions. So we'll need to use `-f` flag to see what happens inside the children's code.

```
strace -f ./riddle
```

The output becomes mingled but after some reading, we see this:

```
[pid 33905] read(33,  <unfinished ...>
...
[pid 33905] <... read resumed>0x7ffefc3575ec, 4) = -1 EBADF (Bad file descriptor)
```

So the fork with pid 33905, tries to read from the non-existant file descriptor 33.

After some lines, we read this:

```
[pid 33904] write(34, "\0\0\0\0", 4 <unfinished ...>
...
[pid 33904] <... write resumed>)        = -1 EBADF (Bad file descriptor)
```

So the fork with pid 33904, tries to write from the non-existant file descriptor 34. Let's give them their file descriptor, exactly like how we did it with the previous challenge, just to see what will happen if the write and read syscalls can proceed.

```C
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

int main (int argc, const char* argv) {
  int fd1 = open(".hello_there", O_RDONLY);
  int fd2 = open(".hi_there", O_WRONLY);
  dup2(fd1, 33);
  dup2(fd2, 34);
  char *newargv[] = { NULL };
  char *newenviron[] = { NULL };

  printf("Success\n");
  execve("./riddle", newargv, newenviron);
}
```

We compile the program and run it with `strace -f ./a.out`

We observe this:

```
[pid 34827] read(53, 0x7fffbd1092ac, 4) = -1 EBADF (Bad file descriptor)
```

Let's add another file descriptor to the mix...

```C
...

int fd3 = open(".placeholder", O_RDONLY);
dup2(fd3, 53);

...
```

DEAD END. All the syscalls proceed but the challenge still Fails. Let's think...

PING - PONG, `[35029] PING!`...

Is that intra-process communication? Maybe! Let's try pipes!

we modify our program above, by substituting all open syscalls with pipe().

```C
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

int main (int argc, const char* argv) {
  int fd[2];
  pipe(fd);
  dup2(fd[0], 33);
  dup2(fd[1], 34);
  pipe(fd);
  dup2(fd[0], 53);
  dup2(fd[1], 54);

  char *newargv[] = { NULL };
  char *newenviron[] = { NULL };

  printf("Success\n");
  execve("./riddle", newargv, newenviron);
}
```

We compile and run... And... It works! :D

## Challenge 7: 'What's in a name?'

Hint: 'A rose, by any other name...'

### Solution

Let's google the hint...

> "A rose by any other name would smell as sweet" is a popular reference to William Shakespeare's play Romeo and Juliet, in which Juliet seems to argue that it does not matter that Romeo is from her family's rival house of Montague, that is, that he is named "Montague". The reference is often used to imply that the **names of things do not affect what they really are**.

Hmm... Let's run our trusty ol' `strace`...

We observe those lines...

```
lstat(".hello_there", {st_mode=S_IFREG|0444, st_size=0, ...}) = 0
lstat(".hey_there", 0x7ffdbda4aa90)     = -1 ENOENT (No such file or directory)
```

Let's give it the file it wants: `touch .hey_there` and run `strace` again. Seems to kinda work, but we get this error-hint: "Oops. 9613388 != 9613400."

Let's see what those `lstat` calls return.

```C
struct stat {
    dev_t     st_dev;     /* ID of device containing file */
    ino_t     st_ino;     /* inode number */
    mode_t    st_mode;    /* protection */
    nlink_t   st_nlink;   /* number of hard links */
    uid_t     st_uid;     /* user ID of owner */
    gid_t     st_gid;     /* group ID of owner */
    dev_t     st_rdev;    /* device ID (if special file) */
    off_t     st_size;    /* total size, in bytes */
    blksize_t st_blksize; /* blocksize for file system I/O */
    blkcnt_t  st_blocks;  /* number of 512B blocks allocated */
    time_t    st_atime;   /* time of last access */
    time_t    st_mtime;   /* time of last modification */
    time_t    st_ctime;   /* time of last status change */
};
```

Could those numbers (9613388 & 9613400) be inode numbers? This together with the Shakespear's quote, makes us think: Can it ask for a file that has different name (and location) with another, but refer to the same inode number? This is the description of a **hard link**!

So we need the 2nd file to be a hard link of the first. Let's do it!

```
ln .hello_there .hey_there
```

DONE! :D

## Challenge 8: 'Big Data'

Hint: Checking footers

### Solution

Well... you know the drill... `strace`!!!

```
openat(AT_FDCWD, "bf00", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf00", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf00", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf00", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf00", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf00", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf00", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf00", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf00", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf00", O_RDONLY)      = -1 ENOENT (No such file or directory)
```

At a first look, it seems obvious what is going wrong. Let's give it the file it wants...

`touch bf00`

Let's run `strace` again:

```
openat(AT_FDCWD, "bf00", O_RDONLY)      = 4
lseek(4, 1073741824, SEEK_SET)          = 1073741824
read(4, "", 16)                         = 0
```

Here we see that the program proceeds to open the file "bf00", skip the first 1073741824 bytes and then read the next 16 bytes.

But, the read doesn't produce any output because the file is really smaller than 1073741824 + 16 bytes.

So, we need to solve that. Let's create a file that is 1073741824 + 16 bytes long.

To achieve this, we'll create a small program in C:

```C
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

int main (int argc, const char* argv) {
   int fd = openat(AT_FDCWD, "bf00", O_RDWR | O_CREAT, S_IRUSR | S_IWUSR);
   lseek(fd, 1073741824, SEEK_SET);
   write(fd, "XXXXXXXXXXXXXXXXX", 16);
   close(fd);
}
```

We compile this and run it, to create the "bf00" file, and then run `strace ./riddle` again.

```
openat(AT_FDCWD, "bf01", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf01", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf01", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf01", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf01", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf01", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf01", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf01", O_RDONLY)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "bf01", O_RDONLY)      = -1 ENOENT (No such file or directory)
```

Oops, it wants another file... (After some trials, we find out that it wants 10 files :P), So we modify the program accordingly...

```C
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

void createFile(char* filename) {
   int fd = openat(AT_FDCWD, filename, O_RDWR | O_CREAT, S_IRUSR | S_IWUSR);
   lseek(fd, 1073741824, SEEK_SET);
   write(fd, "XXXXXXXXXXXXXXXXX", 16);
   close(fd);
}

int main (int argc, const char* argv) {
    createFile("bf00");
    createFile("bf01");
    createFile("bf02");
    createFile("bf03");
    createFile("bf04");
    createFile("bf05");
    createFile("bf06");
    createFile("bf07");
    createFile("bf08");
    createFile("bf09");
}
```

## Challenge 9: 'Connect'

Hint: Let me whisper in your ear

### Solution

Again... `strace`

```
connect(4, {sa_family=AF_INET, sin_port=htons(49842), sin_addr=inet_addr("127.0.0.1")}, 16) = -1 ECONNREFUSED (Connection refused)
```

So the program tries to open a socket that connects to 127.0.0.1:49842 (localhost:49842)
Because no process is listening on that port, the connection is refused.

So we need to add a small TCP listener on that port. An easy way to achieve this is with `nc`.

On a separate terminal we start this: `nc -l -p 49842`
And then we run `./riddle`.

And... there is contact! ;)

DONE! :D

## Challenge 10: 'ESP'

Hint: Can you read my mind?

### Solution

Let's `strace` the shit out of this...

```
openat(AT_FDCWD, "secret_number", O_RDWR|O_CREAT|O_TRUNC, 0600) = 4
unlink("secret_number")                 = 0
write(4, "The number I am thinking of righ"..., 4096) = 4096
close(4)
```

Let's examine this closely.

1. The program opens/creates the file named "secret_number"
2. It then "unlinks" - deletes the file
3. THEN (!) writes on it "The number I am thinking of right..."
4. And it closes the file.

How can the program unlink/delete the file and then write on it? Well...let's pay a visit on unlink's man page.

From the man page, we see two interesting points:

1. > unlink() deletes a name from the file system.

   So, unlink, doesn't deletes a file, but a **name** (Remember Challenge 7?)

2. > If that name was the last link to a file and no processes have the file open the file is deleted and the space it was using is made available for reuse.

   The file associated with this name will only be deleted if that name was the last link to a file and no processes have the file open.

So, flash backs are coming from Challenge 7! If we create a hard link to this file, then unlink will only delete the initial "name" for the file, but not the hard link of it. Let's try this assumption...

```
touch secret_number
ln secret_number alias_secret_number
```

If we run the program and while we wait run `cat alias_secret_number`, we actually see the correct contents!

DONE! :D

## Challenge 11: 'ESP-2'

Hint: Can you read my mind?

### Solution

`strace` again...

```
openat(AT_FDCWD, "secret_number", O_RDWR|O_CREAT|O_TRUNC, 0600) = 4
unlink("secret_number")                 = 0
fstat(4, {st_mode=S_IFREG|0600, st_size=0, ...}) = 0
write(4, "The number I am thinking of righ"..., 4096) = 4096
...

close(4)
```

Well, isn't this kinda the same? Let's try the same strategy as the previous challenge...

Oops:

> You're employing treacherous tricks.
>
> FAIL

We are busted.

Actually, there is an additional syscall to fstat after the unlink call, comparing to the strace's output of the previous challenge.

Let's see what fstat returns:

```
struct stat {
    dev_t     st_dev;     /* ID of device containing file */
    ino_t     st_ino;     /* inode number */
    mode_t    st_mode;    /* protection */
    nlink_t   st_nlink;   /* number of hard links */
    uid_t     st_uid;     /* user ID of owner */
    gid_t     st_gid;     /* group ID of owner */
    dev_t     st_rdev;    /* device ID (if special file) */
    off_t     st_size;    /* total size, in bytes */
    blksize_t st_blksize; /* blocksize for file system I/O */
    blkcnt_t  st_blocks;  /* number of 512B blocks allocated */
    time_t    st_atime;   /* time of last access */
    time_t    st_mtime;   /* time of last modification */
    time_t    st_ctime;   /* time of last status change */
};
```

**number of hard links** (nlink_t), that's how we were busted... so we need to employ a different strategy. Let's go back to unlink's man page:

There is another interesting point on the documentation:

> If the name was the last link to a file but any processes still have
the file open, **the file will remain in existence until the last file
descriptor referring to it is closed**.

So, this gives us a small loophole to maintain the file even after the unlink() and the close() calls. If we could open the same file from another process, then the file would not be deleted until the second process closes the fd too.

So we can create a program that continuously reads the file and outputs the contents written to it on stdout.

```C
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

int main (int argc, const char* argv) {
   int fd = openat(AT_FDCWD, "secret_number", O_RDWR);

   char buf[1000];

   int nbytes;
   while (1) {
     nbytes = read(fd, buf, 16);
     if (nbytes > 0) {
        printf("%s\n", buf);
     }
   }
}
```

We just have to pre-create the file with `touch secret_number`, run the above program, execute ./riddle and watch.

DONE!

## Challenge 12: 'A delicate change'

Hint: 'Do only what is required, nothing more, nothing less'.

> I want to find the char 'Y' at 0x7f7c2413906f

### Solution

I might be repetitive... but... `strace` again.

```
openat(AT_FDCWD, "/tmp/riddle-w2hAKp", O_RDWR|O_CREAT|O_EXCL, 0600) = 4
ftruncate(4, 4096)                      = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_SHARED, 4, 0) = 0x7f7c24139000
```

So, the program open/creates a new file at "/tmp/riddle-{random}", increases its size/length by 4096 bytes with ftruncate, and then memory maps it (with mmap). 

The mmap returns the address in the memory where the beginning of the file is mapped. On this case, this is 0x7f7c24139000. The directions say that the program want's to find the char 'Y' at 0x7f7c2413906f. 

0x7f7c2413906f - 0x7f7c24139000 = 0x6f = 111

So the program wants to see the char 'Y' at the 111th byte of the file.

One solution we can try is to write 111 characters to the file, making sure that the 111th character would be the one it wants.

This can be easilly done with a text editor (but you must be fast, before the countdown ends!)

But... The program is smarter than that! 

> You need to change only 0x7f19edf8906f!

On our case we had changed all other addresses before the 111th too!

So we need to change our approach. To be more **delicate**, we'll need to use mmap. Let's write a program that does exactly that.

```C
#include <sys/types.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <unistd.h>
#include <stdio.h>
#include <sys/mman.h>

int main(int argc, char** argv) {
    char* buff[1];
    buff[0] = argv[2];
    int fd = open(argv[1], O_RDWR);
    char* data = (char*) mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
    data[111] = *argv[2];
    printf("Success\n");
}
```

The above program, takes as input from the argv the filename and the character we want to write, and performs the writing using memory map.

eg: `./a.out /tmp/riddle-MFiZZC Y`

We compile and run it before the countdown ends.

AND DONE!

## Challenge  13: 'Bus error'

Hint: Memquake! Don't lose the pages beneath your something something


### Solution

\- What is the answer to life the universe and everything?

\- `strace` 

We immediatelly see the first issue:

```
openat(AT_FDCWD, ".hello_there", O_RDWR|O_CREAT, 0600) = -1 EACCES (Permission denied)
```

Well, this problem was leftover from the 2nd challenge (remember where we removed the write permission?), we can simply remove the file and the newly created one will have the correct permissions.

So let's strace again...

```
openat(AT_FDCWD, ".hello_there", O_RDWR|O_CREAT, 0600) = 4
ftruncate(4, 32768)                     = 0
mmap(NULL, 32768, PROT_READ|PROT_WRITE, MAP_SHARED, 4, 0) = 0x7fdf9f6d5000
ftruncate(4, 16384)                     = 0
read(0,
```

There it waits for stdin, and when we hit any character...

```
--- SIGBUS {si_signo=SIGBUS, si_code=BUS_ADRERR, si_addr=0x7f28a858d000} ---
+++ killed by SIGBUS (core dumped) +++
```

Oops... Why did this happened? 

Firstly, what is the SIGBUS error?

>SIGBUS (bus error) is a signal that happens **when you try to access memory that has not been physically mapped**. This is different to a SIGSEGV (segmentation fault) in that a segfault happens when an address is invalid, while a bus error means **the address is valid but we failed to read/write**.

Let's see what the program did...

1. It opens the file ".hello_there".
2. It increases its size to 32768 bytes.
3. It memory maps it.
4. It decreases its size to 16384 bytes.

Now, it becomes apparent. After we decreases the size of the file, a portion of the memory mapped to it, isn't mapped to something anymore. So, when we try to access that portion of the memory, a SIGBUS is raised.

Maybe, if we could raise the file's size again to its original size, the memory will become accessible again. We'll use this small program to achieve that:

```
#include <sys/types.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <unistd.h>
#include <stdio.h>
#include <sys/mman.h>

int main(int argc, char** argv) {
    int fd = open(argv[1], O_RDWR);
    ftruncate(fd, 32768);
    printf("Success\n");
}
```

This program, changes the size of the file inputted to it through argv to 32768 bytes.

Let's compile it and run it

`./a.out .hello_there`

DONE!

## Challenge 14: 'Are you the One?'

Hint: Are you 32767? If not, reincarnate!

### Solution

Well this one is pretty obvious, if the program has a pid of 32767, then we can pass. What is not trivial is how to change the pid of a program. 

There is only one way, through fork(). But fork() only gives the next pid.

A trivial solution would be this, but with this solution you should pray that the last pid is smaller than 32767. Maybe you are lucky. Maybe you can be lucky with a restart. 

```C
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

int main (int argc, const char* argv) {
    char *newargv[] = { NULL };
    char *newenviron[] = { NULL };

    while (getpid() < 32767) {
        if (fork() == 0) {
            if (getpid() == 32767) {
                execve("./riddle", newargv, newenviron);
            }
        }
    }

    return 0;
}
```

If you are not so lucky, you can try this:

The last pid is stored on `/proc/sys/kernel/ns_last_pid`. If you edit this file, you can manipulate the pid the next process started will have. If the 

To be sure that your solution will work, you need to lock the file so that no other process can write on it, or change the pid before you run fork();

```C
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/file.h>

int main(int argc, char *argv[])
{
    char *newargv[] = { NULL };
    char *newenviron[] = { NULL };
    int fd, pid;
    char buf[32];

    if (argc != 2)
     return 1;

    printf("Opening ns_last_pid...\n");
    fd = open("/proc/sys/kernel/ns_last_pid", O_RDWR | O_CREAT, 0644);
    if (fd < 0) {
        perror("Can't open ns_last_pid");
        return 1;
    }
    printf("Done\n");

    printf("Locking ns_last_pid...\n");
    if (flock(fd, LOCK_EX)) {
        close(fd);
        printf("Can't lock ns_last_pid\n");
        return 1;
    }
    printf("Done\n");

    pid = atoi(argv[1]);
    snprintf(buf, sizeof(buf), "%d", pid - 1);

    printf("Writing pid-1 to ns_last_pid...\n");
    if (write(fd, buf, strlen(buf)) != strlen(buf)) {
        printf("Can't write to buf\n");
        return 1;
    }
    printf("Done\n");

    if (fork() == 0) {
        execve("./riddle", newargv, newenviron);
        return 0;
    }

    printf("Done\n");

    printf("Cleaning up...");
    if (flock(fd, LOCK_UN)) {
        printf("Can't unlock");
    }

    close(fd);

    printf("Done\n");

    return 0;
}
```

Compile, and run it with root privileges (sudo). It should work.

Done :D 

TIER ONE COMPLETE!