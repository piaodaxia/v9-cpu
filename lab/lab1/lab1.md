## Lab1 

### 本实验目的
实现代码，支持时钟，中断，输出

### 代码功能
基于os0.c的代码进行修改，接收键盘interrupt，每次接收一次键盘的时候中断count加一，利用out()函数进行output

### 中断功能
定义中断代码如下，也就是v9文档中的错误码“fault code”

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

根据v9的定义，发生中断之后，xem模拟器将
- PUSH 当前的pc到当前内核栈
- PUSH 错误码(fault code)到当前内核栈

栈帧结构如下
```
    =======
      PC   
    =======
      FC 
    =======
```

随后跳转到代码中
```c
 ivec(alltraps);
```
中断处理函数中
```c
alltraps()
{
  asm(PSHA);
  asm(PSHB);

  /*current++;*/
  trap();

  asm(POPB);
  asm(POPA);
  asm(RTI);
}
```
做成了RegisterA、B的保存空间，然后进入具体的处理函数trap()，并在处理之后恢复RegisterA、B的原值（），RTI是中断返回

调用trap()函数之前，栈帧结构如下

```
	  ========
       PC   
    ========
       FC  
    ========
      regA 
    ========
      regB 
    ========
```
函数中的b、a是对应于push压入的寄存器b、a，fc是中断错误码，pc返回地址（process counter），fc为中断错误码，b和a对应于push压入的寄存器b和a

trap()函数中的代码，
```c
trap(int b, int a, int fc, int pc)
{

    switch(fc) {
        case FTIMER:
            current++;
            break;
        case FKEYBD:
            count++;
            break;
        default:
            ;
            /*out(1, "d");*/
    }
}
```



trap函数的功能是根据栈帧中的fc获得中断码，如果执行键盘中断count就加一，执行时钟中断current也加一。

### Main函数执行

xem根据编译之后产生的v9可执行文件格式，jump到main函数开始执行，分析具体功能如下
```c
main()
{
//开始数值
  current = 0;
  count = 0;

//时钟中断间隔
  stmr(2000);
//处理中断函数的地址
  ivec(alltraps);
//中断使能
  asm(STI);

//时钟中断小于2000次键盘中断小于10次时，每次打键盘将产生0、1的跳变output

  while (current < 2000 && count < 10) {
    if (count & 1) out(1, '1'); else out(1, '0');
  }

  halt(0);
}
```



