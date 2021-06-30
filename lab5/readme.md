# Lab5 - traps

```shell
  $ git fetch
  $ git checkout lazy
  $ make clean
```

## Eliminate allocation from sbrk() (easy)

```c
//kernel/sysproc.c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  myproc()->sz = addr + n;
  /*
  if(growproc(n) < 0)
    return -1;
  */
  return addr;
}
```

```shell
xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ echo hi
usertrap(): unexpected scause 0x000000000000000f pid=3
            sepc=0x00000000000012a6 stval=0x0000000000004008
panic: uvmunmap: not mapped
```
## Lazy allocation (moderate)

We should ignore those unmapped pages:
```c
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  ..............
  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      continue;
      //panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      continue;
      //panic("uvmunmap: not mapped");
  ...............
    *pte = 0;
  }
}
```
From above we can know that:  
`unexpected scause 0x000000000000000f`  
This corresponds to `r_scause() == 15`  
However, the lecture also tell us that 
> You can check whether a fault is a page fault by seeing if r_scause() is 13 or 15 in usertrap().  

So the type of trap is either 15 or 13.  

```c
void
usertrap(void)
{
  .....................
  
  if(r_scause() == 8){
    // system call
    .................
  } else if(r_scause() == 15 || r_scause() == 13){
    uint64 va = r_stval();
    uint64 ka = (uint64) kalloc();
    if(ka == 0) p->killed = -1;
    else{
      memset((void*)ka, 0, PGSIZE);
      if(mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, ka, PTE_W|PTE_X|PTE_R|PTE_U) != 0){
        kfree((void*)ka);
        p->killed = -1;
      }
    }
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    ...............
  }
    ...............
}
```
Now all goes normally: 

```shell
xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$ echo hi
hi
```

## Lazytests and Usertests (moderate)

1. Handle negative sbrk() arguments.
2. Kill a process if it page-faults on a virtual memory address higher than any allocated with sbrk().
3. Handle the parent-to-child memory copy in fork() correctly.
4. Handle the case in which a process passes a valid address from sbrk() to a system call such as read or write, but the memory for that address has not yet been allocated.
5. Handle out-of-memory correctly: if kalloc() fails in the page fault handler, kill the current process.
6. Handle faults on the invalid page below the user stack.

To avoid deallocate more than the size it already have, we should check its value:  

```c
//kernel/sysproc.c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  struct proc* p = myproc();
  addr = p->sz;
  if(n < 0){
    //1. Handle negative sbrk() arguments.
    if(p->sz + n < 0) return -1;//n cannot beyond sz
    else 
      uvmdealloc(p->pagetable, p->sz, p->sz+n);
  }
  p->sz = addr + n;
  /*
  if(growproc(n) < 0)
    return -1;
  */
  return addr;
}
```
Here we should delve into uvmdealloc:
```c
uint64
uvmdealloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz)
{
  if(newsz >= oldsz)
    return oldsz;

  if(PGROUNDUP(newsz) < PGROUNDUP(oldsz)){
    int npages = (PGROUNDUP(oldsz) - PGROUNDUP(newsz)) / PGSIZE;
    uvmunmap(pagetable, PGROUNDUP(newsz), npages, 1);
  }

  return newsz;
}


```
Imagine we've lazy allocated some memory and have not actually used it(so it's not allocated).  
Now we're asked to `sbrk()` some memory which means we should deallocate some memory.  
However, what will happen if we deallocate some memory that have not even been allocate? It's undefined, so we should ignore those panics in `uvmdealloc()`, more concisely, `uvmunmap()`.

```c
//kernel/vm.c
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      continue;
      //panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      continue;
      //panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```
Next we should make sure that va is smaller than process's size(since that's the range of lazy allocation)   
and it's beyond it's stack's top(remeber that stack is growing downward).

```c
//kernel/trap.c
} else if(r_scause() == 15 || r_scause() == 13){
      uint64 va = r_stval();
      if(va < p->sz && va > PGROUNDDOWN(p->trapframe->sp)){
        //2, 6
        uint64 ka = (uint64) kalloc();
        if(ka == 0) p->killed = -1;
        //5. Handle out-of-memory correctly: if kalloc() fails in the page fault handler, kill the current process.
        else{
          memset((void*)ka, 0, PGSIZE);
          if(mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, ka, PTE_W|PTE_X|PTE_R|PTE_U) != 0){
            p->killed = -1;
          }
          //4. Handle the case in which a process passes a valid address from sbrk() to a system call such as read or write, but the memory for that address has not yet been allocated.
        }
      }
      //2. Kill a process if it page-faults on a virtual memory address higher than any allocated with sbrk().  
      //6. Handle faults on the invalid page below the user stack.
      else p->killed = -1;
  } else if((which_dev = devintr()) != 0){
    // ok
```

Next we consider `fork()`.

Having examined `fork()`, we discovered that uvmcopy will walk in some address that has not yet been allocated(which will be later lazy allocated). So we should ignore those panics when we walk through them.
```c
//kernel/proc.c
if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
}
//just for presentation

//3. Handle the parent-to-child memory copy in fork() correctly.

//kernel/vm.c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  ................
  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      continue;
      //panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      continue;
      //panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    ..............
  }
  ..............
}
```

Finally, we have a allocation-related code in `exec.c`, it's walkaddr, which translate virtual memory into physical memory.  

```c
static int
loadseg(pagetable_t pagetable, uint64 va, struct inode *ip, uint offset, uint sz)
{
  ...........
  for(i = 0; i < sz; i += PGSIZE){
    pa = walkaddr(pagetable, va + i);
    ...............
  }
```

It's just the same as what we've done in `trap.c`.  
So we simulate those code with some extra details.  

```c
//kernel/vm.c
uint64
walkaddr(pagetable_t pagetable, uint64 va)
{
  .........
  struct proc* p = myproc();
  ........
  
  if(pte == 0 || (*pte & PTE_V) == 0){
    if (va >= p->sz || va < PGROUNDUP(p->trapframe->sp))
      return 0;
    uint64 ka = (uint64)kalloc();
    if (ka == 0)
      return 0;
    if (mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, ka, PTE_W|PTE_X|PTE_R|PTE_U) != 0)
    {
      kfree((void*)ka);
      return 0;
    }
    return ka;
  }


  if((*pte & PTE_U) == 0)
    return 0;
  pa = PTE2PA(*pte);
  return pa;
}
```

Remeber to put those definition macros at the top, and `spinlock` must be the former like this:

```c
#include "spinlock.h"
#include "proc.h"
```

```shell
== Test running lazytests == 
$ make qemu-gdb
(3.6s) 
== Test   lazy: map == 
  lazy: map: OK 
== Test   lazy: unmap == 
  lazy: unmap: OK 
== Test usertests == 
$ make qemu-gdb
(80.6s) 
== Test   usertests: pgbug == 
  usertests: pgbug: OK 
== Test   usertests: sbrkbugs == 
  usertests: sbrkbugs: OK 
== Test   usertests: argptest == 
  usertests: argptest: OK 
== Test   usertests: sbrkmuch == 
  usertests: sbrkmuch: OK 
== Test   usertests: sbrkfail == 
  usertests: sbrkfail: OK 
== Test   usertests: sbrkarg == 
  usertests: sbrkarg: OK 
== Test   usertests: stacktest == 
  usertests: stacktest: OK 
== Test   usertests: execout == 
  usertests: execout: OK 
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyout == 
  usertests: copyout: OK 
== Test   usertests: copyinstr1 == 
  usertests: copyinstr1: OK 
== Test   usertests: copyinstr2 == 
  usertests: copyinstr2: OK 
== Test   usertests: copyinstr3 == 
  usertests: copyinstr3: OK 
== Test   usertests: rwsbrk == 
  usertests: rwsbrk: OK 
== Test   usertests: truncate1 == 
  usertests: truncate1: OK 
== Test   usertests: truncate2 == 
  usertests: truncate2: OK 
== Test   usertests: truncate3 == 
  usertests: truncate3: OK 
== Test   usertests: reparent2 == 
  usertests: reparent2: OK 
== Test   usertests: badarg == 
  usertests: badarg: OK 
== Test   usertests: reparent == 
  usertests: reparent: OK 
== Test   usertests: twochildren == 
  usertests: twochildren: OK 
== Test   usertests: forkfork == 
  usertests: forkfork: OK 
== Test   usertests: forkforkfork == 
  usertests: forkforkfork: OK 
== Test   usertests: createdelete == 
  usertests: createdelete: OK 
== Test   usertests: linkunlink == 
  usertests: linkunlink: OK 
== Test   usertests: linktest == 
  usertests: linktest: OK 
== Test   usertests: unlinkread == 
  usertests: unlinkread: OK 
== Test   usertests: concreate == 
  usertests: concreate: OK 
== Test   usertests: subdir == 
  usertests: subdir: OK 
== Test   usertests: fourfiles == 
  usertests: fourfiles: OK 
== Test   usertests: sharedfd == 
  usertests: sharedfd: OK 
== Test   usertests: exectest == 
  usertests: exectest: OK 
== Test   usertests: bigargtest == 
  usertests: bigargtest: OK 
== Test   usertests: bigwrite == 
  usertests: bigwrite: OK 
== Test   usertests: bsstest == 
  usertests: bsstest: OK 
== Test   usertests: sbrkbasic == 
  usertests: sbrkbasic: OK 
== Test   usertests: kernmem == 
  usertests: kernmem: OK 
== Test   usertests: validatetest == 
  usertests: validatetest: OK 
== Test   usertests: opentest == 
  usertests: opentest: OK 
== Test   usertests: writetest == 
  usertests: writetest: OK 
== Test   usertests: writebig == 
  usertests: writebig: OK 
== Test   usertests: createtest == 
  usertests: createtest: OK 
== Test   usertests: openiput == 
  usertests: openiput: OK 
== Test   usertests: exitiput == 
  usertests: exitiput: OK 
== Test   usertests: iput == 
  usertests: iput: OK 
== Test   usertests: mem == 
  usertests: mem: OK 
== Test   usertests: pipe1 == 
  usertests: pipe1: OK 
== Test   usertests: preempt == 
  usertests: preempt: OK 
== Test   usertests: exitwait == 
  usertests: exitwait: OK 
== Test   usertests: rmdot == 
  usertests: rmdot: OK 
== Test   usertests: fourteen == 
  usertests: fourteen: OK 
== Test   usertests: bigfile == 
  usertests: bigfile: OK 
== Test   usertests: dirfile == 
  usertests: dirfile: OK 
== Test   usertests: iref == 
  usertests: iref: OK 
== Test   usertests: forktest == 
  usertests: forktest: OK 
== Test time == 
time: OK 
Score: 119/119
```