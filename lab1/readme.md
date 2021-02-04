## Environment  

`git clone git://g.csail.mit.edu/xv6-labs-2020`  
`cd xv6-labs-2020`  
`git checkout util`  

To install toolchain:  
`sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu`  


## Sleep

Create a file named "sleep.c" in user.  
Just like: xv6-labs-2020/user/sleep.c  

Code:
```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int 
main(int argc, char *argv[])
{
    int duration;
    if(argc<=1){
        fprintf(2,"Require more parameters!");
        exit(1);
    }
    duration = atoi(argv[1]);
    sleep(duration);
    exit(0);
}
```  

Do not forget to add this to your Makefile:
`$U/_sleep\`  
at line 152.  

### Test 
`make qemu`  
`sleep 10`  
It will stop for a little while.  

If it's all set, then run `make grade` to see if it's correct.  

If you get error like this:  

`/usr/bin/env: ‘python’: No such file or directory`  
`make: *** [Makefile:233: grade] Error 127`  

Your machine may not include python2, try:  
`sudo apt-get install python`  

Then `make grade`, it will print your results as follows:  
```
== Test sleep, no arguments == 
$ make qemu-gdb
sleep, no arguments: OK (4.7s) 
== Test sleep, returns == 
$ make qemu-gdb
sleep, returns: OK (0.8s) 
== Test sleep, makes syscall == 
$ make qemu-gdb
sleep, makes syscall: OK (0.9s) 
== Test pingpong == 
$ make qemu-gdb
pingpong: FAIL (1.0s) 
    
Score: 20/100
```
