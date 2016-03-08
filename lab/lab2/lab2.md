## Lab2

### 实验目的
实现代码，支持建立页表

### 分析功能
os2.c代码中实现了中断处理和建立页表，lab2.c增加了写入、读取页表映射范围内数据的测试项。

## 中断功能
定义中断码，也就是v9文档中所述的错误码fault code, 和lab1的一样。

```c
enum { // processor fault codes
  FMEM,   // bad physical address
  FTIMER, // timer interrupt
  FKEYBD, // keyboard interrupt
  FPRIV,  // privileged instruction
  FINST,  // illegal instruction
  FSYS,   // software trap
  FARITH, // arithmetic trap
  FIPAGE, // page fault on opcode fetch
  FWPAGE, // page fault on write
  FRPAGE, // page fault on read
  USER=16 // user mode exception
};
```

本实验中主要关注页访问异常FRPAGE和FWPAGE
```
trap(int c, int b, int a, int fc, int pc)
{
  printf("TRAP: ");
  switch (fc) {
  case FINST:  printf("FINST"); break;
  case FRPAGE: printf("FRPAGE [0x%08x]",lvadr()); break;
  case FWPAGE: printf("FWPAGE [0x%08x]",lvadr()); break;
  case FIPAGE: printf("FIPAGE [0x%08x]",lvadr()); break;
  case FSYS:   printf("FSYS"); break;
  case FARITH: printf("FARITH"); break;
  case FMEM:   printf("FMEM [0x%08x]",lvadr()); break;
  case FTIMER: printf("FTIMER"); current = 1; stmr(0); break;
  case FKEYBD: printf("FKEYBD [%c]", in(0)); break;
  default:     printf("other [%d]",fc); break;
  }
}

alltraps()
{
  asm(PSHA);
  asm(PSHB);
  asm(PSHC);
  trap();
  asm(POPC);
  asm(POPB);
  asm(POPA);
  asm(RTI);
}
```

### 分页机制
分页机制的目的是建立虚拟地址和物理地址之间的关系，企管系以页目录和页表定义。

分配用于存放页目录和页表项的内存
```c
char pg_mem[6 * 4096]; // page dir + 4 entries + alignment
int *pg_dir, *pg0, *pg1, *pg2, *pg3;
```

页目录和页表是用setup_paging()函数建立的
```c
setup_paging()
{
  int i;
  //对齐页目录地址
  pg_dir = (int *)((((int)&pg_mem) + 4095) & -4096);
  //页表0的地址等于页目录加1024
  pg0 = pg_dir + 1024;
  pg1 = pg0 + 1024;
  pg2 = pg1 + 1024;
  pg3 = pg2 + 1024;
 
 //页目录中页表的属性为present、可写和用户态
  pg_dir[0] = (int)pg0 | PTE_P | PTE_W | PTE_U;  // identity map 16M
  pg_dir[1] = (int)pg1 | PTE_P | PTE_W | PTE_U;
  pg_dir[2] = (int)pg2 | PTE_P | PTE_W | PTE_U;
  pg_dir[3] = (int)pg3 | PTE_P | PTE_W | PTE_U;
  //剩余页表项清0
  for (i=4;i<1024;i++) pg_dir[i] = 0;
  
  //建立页表项，4Kb为一页，依次递增，并页属性为present、可写和用户态，共映射了16M(4096*4096)的地址空间
  for (i=0;i<4096;i++) pg0[i] = (i<<12) | PTE_P | PTE_W | PTE_U;  // trick to write all 4 contiguous pages
  //设置页目录地址
  pdir(pg_dir);
  //分页使能
  spage(1);
}
```

## 分页功能测试
main()函数中测试了不同的中断，如illegal指令, 下面主要分析分页功能的测试。
###设置堆栈、启动分页功能
```c
// 设置SP为4M
  asm(LI, 4*1024*1024); // a = 4M
  asm(SSP); // sp = a
// 启动分页
  setup_paging();
  printf("identity map...ok\n");
```

### 内存访问
基于现有的os2.c代码，开启分页测试可以进行数据访问，然后删除页表项，再访问异常。
```c
//启动分页后，进行正常测试
  printf("test page read...");
  t = *(int *)(50<<12);
  printf("...ok\n");
  printf("test page write...");
  *(int *)(50<<12) = 5;
  printf("...ok\n");
//设置页表项为0，访问时将导致异常，进入处理中断函数，用printf（）函数将异常输出。 
  printf("test page fault read...");
  pg0[50] = 0;
  pdir(pg_dir);
  t = *(int *)(50<<12);
  printf("...ok\n");
  
  printf("test page fault write...");
  *(int *)(50<<12) = 5;
  printf("...ok\n");
```
