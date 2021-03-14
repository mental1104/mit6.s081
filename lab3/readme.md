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