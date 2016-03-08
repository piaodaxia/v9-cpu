# Lab1 实验报告

## 实验目的
实现代码，支持时钟，中断，输出

## 代码功能
基于课程提供的os0.c的源码进行修改，接收键盘中断，每接收一次键盘中断使count加一，并利用out()函数进行输出。

## 中断功能
首先定义了中断码，也就是v9文档中所述的错误码fault code。

+```c
+enum { // processor fault codes
+  FMEM,   // bad physical address
+  FTIMER, // timer interrupt
+  FKEYBD, // keyboard interrupt
+  FPRIV,  // privileged instruction
+  FINST,  // illegal instruction
+  FSYS,   // software trap
+  FARITH, // arithmetic trap
+  FIPAGE, // page fault on opcode fetch
+  FWPAGE, // page fault on write
+  FRPAGE, // page fault on read
+  USER=16 // user mode exception
+};
+```
+本实验主要增加了对键盘中断的处理。
+
+根据v9的文档定义，当中断发生后，xem模拟器将
+- PUSH当前的pc到当前内核栈
+- PUSH 错误码(fault code)到当前内核栈
+
+即栈帧结构为
+```
+	   =======
+      PC   
+    =======
+      FC 
+    =======
+```
+
+随后跳转到代码中用
+```c
+ ivec(alltraps);
+```
+所定义的中断处理函数中，而在中断处理函数中
+```c
+alltraps()
+{
+  asm(PSHA);
+  asm(PSHB);
+
+  /*current++;*/
+  trap();
+
+  asm(POPB);
+  asm(POPA);
+  asm(RTI);
+}
+```
+完成了对寄存器A、B的保存，然后进入具体的处理函数trap()，并在完成处理后恢复寄存器A、B的原值（），RTI完成中断返回。
+
+在调用trap()函数前，其栈帧结构为
+
+```
+	   ========
+       PC   
+    ========
+       FC  
+    ========
+      regA 
+    ========
+      regB 
+    ========
+```
+函数中的b、a、fc和pc即对应于中断处理函数所压入的栈帧数据，pc为返回指令地址，fc为中断错误码，b和a对应于push压入的寄存器b和a
+
+在trap()函数中，
+```c
+trap(int b, int a, int fc, int pc)
+{
+
+    switch(fc) {
+        case FTIMER:
+            current++;
+            break;
+        case FKEYBD:
+            count++;
+            break;
+        default:
+            ;
+            /*out(1, "d");*/
+    }
+}
+```
+
+
+
+trap函数的功能是根据栈帧中的fc获得中断码，若为键盘中断则count加一，时钟中断则current加一。
+
+## Main执行流程
+
+xem根据编译后形成的v9可执行文件格式，跳入main函数开始执行，其具体功能分析如下
+```c
+main()
+{
+//设置初值
+  current = 0;
+  count = 0;
+
+//设置时钟中断间隔
+  stmr(10000);
+//设置中断处理函数地址
+  ivec(alltraps);
+//中断使能
+  asm(STI);
+
+//时钟中断次数小于10000次且键盘中断小于10次时，每次敲击键盘将产生0、1的跳变输出
+
+  while (current < 10000 && count < 10) {
+    if (count & 1) out(1, '1'); else out(1, '0');
+  }
+
+  halt(0);
+}
+```
+
+## 其余辅助函数
+外设输出函数
+```
+out(port, val)  { asm(LL,8); asm(LBL,16); asm(BOUT); }
+```
+将栈帧中距栈指针sp偏移为8的地址内数据加载的寄存器A，将偏移16的地址内数据加载到寄存器B，然后调用外设输出指令。
+
+以下函数类似，利用了栈帧进行参数传递，第一个参数的地址距sp的偏移为8，加载到寄存器A后，调用对应CPU指令。
+```
+ivec(void *isr) { asm(LL,8); asm(IVEC); }
+stmr(int val)   { asm(LL,8); asm(TIME); }
+halt(val)       { asm(LL,8); asm(HALT); }
+```
+
