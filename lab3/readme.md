# Lab pgtbl  

```
git fetch
git checkout pgtbl
make clean
```  

## Print a page table (easy)  

```c
/*vm print*/
void vmwalk(pagetable_t pagetable, int depth){
  for(int i = 0; i < 512; i++){
      pte_t pte = pagetable[i];
      if(pte & PTE_V){
        uint64 pa = PTE2PA(pte);
        
        int temp = depth;
        while(temp--)
            printf(".. ");

        printf("..%d: pte %p pa %p\n",i, pte, pa);
        if(depth<2)
            vmwalk((pagetable_t)pa, depth+1);
      }
  }
}

void vmprint(pagetable_t pagetable){
   printf("page table %p\n", pagetable);
   vmwalk(pagetable, 0);
}
```  
Add declaration to defs.h:  

```c
void            vmwalk(pagetable_t pagetable, int depth);
void            vmprint(pagetable_t pagetable);
```

Add this to exec.c before `return argc;`
```
if(p->pid==1) vmprint(p->pagetable);
```  


## A kernel page table per process (hard)

Firstly, we should add a kernel_pagetable to the struct proc:
```c
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  pagetable_t kpagetable;
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```
Now we add these function to vm.c. Do not forget to add these declaration to defs.h
```c
kvmmapkern(pagetable_t pagetable, uint64 va, uint64 pa, uint64 sz, int perm);
pagetable_t     kvmcreate();
void            kvmfree(pagetable_t kpagetale) ;
```

```c
void 
kvmmapkern(pagetable_t pagetable, uint64 va, uint64 pa, uint64 sz, int perm)
{
  if (mappages(pagetable, va, sz, pa, perm) != 0) 
    panic("kvmmap");
}

// proc's version of kvminit
pagetable_t
kvmcreate() 
{
  pagetable_t pagetable;
  int i;

  pagetable = uvmcreate();
  for(i = 1; i < 512; i++) {
    pagetable[i] = kernel_pagetable[i];
  }

  // uart registers
  kvmmapkern(pagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  kvmmapkern(pagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  kvmmapkern(pagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  kvmmapkern(pagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  return pagetable;
}


void 
kvmfree(pagetable_t kpagetale) 
{
  pte_t pte = kpagetale[0];
  pagetable_t level1 = (pagetable_t) PTE2PA(pte);
  for (int i = 0; i < 512; i++) {
    pte_t pte = level1[i];
    if (pte & PTE_V) {
      uint64 level2 = PTE2PA(pte);
      kfree((void *) level2);
      level1[i] = 0;
    }
  }
  kfree((void *) level1);
  kfree((void *) kpagetale);
}
```  

Add this to proc.c/allocproc()

```c
p->kpagetable = kvmcreate();
  if(p->kpagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  } 
```

and this to freeproc()
```
if(p->kpagetable)
    kvmfree(p->kpagetable);
p->kpagetable = 0;
```  

Then we will change the scheduler()
```c
if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;

        w_satp(MAKE_SATP(p->kpagetable));
        sfence_vma();

        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;

        found = 1;
}
```

And if proc is not created, use the kernel's pagetable:
```
if(found == 0) {
      intr_on();
      kvminithart();
      asm volatile("wfi");
}
```

## Simplify

core function:

```c
//vm.c
void 
u2kvmcopy(pagetable_t pagetable, pagetable_t kpagetable, uint64 oldsz, uint64 newsz)
{
  pte_t* pte_from, *pte_to;
  uint64 a, pa;
  uint flags;

  if(newsz < oldsz)
    return;

  oldsz = PGROUNDUP(oldsz);
  for (a = oldsz; a < newsz; a += PGSIZE)
  {
    if((pte_from = walk(pagetable, a, 0)) == 0)
      panic("u2kvmcopy: pte should exist");
    if((pte_to = walk(kpagetable, a, 1)) == 0)
      panic("u2kvmcopy: walk fails");
    pa = PTE2PA(*pte_from);
    flags = (PTE_FLAGS(*pte_from) & (~PTE_U));
    *pte_to = PA2PTE(pa) | flags;
  }
}
```

Remeber to rip off PTE_U before entering kernel.

Add its declaration into defs.h:

```c
//defs.h
void            u2kvmcopy(pagetable_t pagetable, pagetable_t kpagetable, uint64 oldsz, uint64 newsz);
```

And then we insert one single line into fork(), sbrk() and exec():

sbrk():

```c
//proc.c
int
growproc(int n)
{
  ..............................
  if(n > 0){
    if(PGROUNDUP(sz + n) >= PLIC)
      return -1;
    if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
      return -1;
    }
    u2kvmcopy(p->pagetable, p->kpagetable, sz-n, sz);
  } else if(n < 0){
    sz = uvmdealloc(p->pagetable, sz, sz + n);
  }
  ............................
}
```

fork():

```c
int fork(void){
  ........................
  np->cwd = idup(p->cwd);

  u2kvmcopy(np->pagetable, np->kpagetable, 0, np->sz);

  safestrcpy(np->name, p->name, sizeof(p->name));

  pid = np->pid;

  np->state = RUNNABLE;

  release(&np->lock);
  ..........................

}
```

exec():

```c
//exec.c
int
exec(char *path, char **argv){
  ....................
  sp = sz;
  stackbase = sp - PGSIZE;

  u2kvmcopy(pagetable, p->kpagetable, 0, sz);

  // Push argument strings, prepare rest of stack in ustack.
  for(argc = 0; argv[argc]; argc++) {
  ....................
}
```

And userinit():

```c
//proc.c
void
userinit(void)
{
  ................................
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  u2kvmcopy(p->pagetable, p->kpagetable, 0, p->sz);

  // prepare for the very first "return" from kernel to user.
  p->trapframe->epc = 0;      // user program counter
  p->trapframe->sp = PGSIZE;  // user stack pointer

 ....................................
}
```

Finally, rewrite copyin and copyinstr_new.

Actually, they've been already offered in vmcopyin.c. What we should do is create a new head file named vmcopyin.h and add their declarations like this:

```c
//vmcopyin.h
int copyin_new(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len);
int copyinstr_new(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max);
```

Then we rewrite the two functions:

``````````````````````c
`````````````````````
#include "vmcopyin.h"
`````````````````````
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  return copyin_new(pagetable, dst, srcva, len);
}
``````````````````````

```c
int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  return copyinstr_new(pagetable, dst, srcva, max);
}
```

All clear!

And we should create two file named answers-pgtbl.txt and time.txt.

This lab spent me nearly 12 hours.

`make grade`

```shell
== Test pte printout == 
$ make qemu-gdb
pte printout: OK (5.3s) 
== Test answers-pgtbl.txt == answers-pgtbl.txt: OK 
== Test count copyin == 
$ make qemu-gdb
count copyin: OK (1.1s) 
== Test usertests == 
$ make qemu-gdb
(83.0s) 
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyinstr1 == 
  usertests: copyinstr1: OK 
== Test   usertests: copyinstr2 == 
  usertests: copyinstr2: OK 
== Test   usertests: copyinstr3 == 
  usertests: copyinstr3: OK 
== Test   usertests: sbrkmuch == 
  usertests: sbrkmuch: OK 
== Test   usertests: all tests == 
  usertests: all tests: OK 
== Test time == 
time: OK 
Score: 66/66
```