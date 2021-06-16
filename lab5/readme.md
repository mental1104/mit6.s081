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