+# Lab1 
+
+## 本实验目的
+实现代码，支持时钟，中断，输出
+
+## 代码功能
+基于os0.c的代码进行修改，接收键盘interrupt，每次接收一次键盘的时候中断count加一，利用out()函数进行output
+
+## 中断功能
+定义中断代码如下，也就是v9文档中的错误码“fault code”
+
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
+
+根据v9的文档的定义，发生了中断之后，xem模拟器将
+- PUSH 当前的pc到当前内核栈
+- PUSH 错误码(fault code)到当前内核栈
+
+PUSH后栈帧结构如下
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
+ 中断处理函数中，在处理中断的函数中用
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
+完成了Register A、B，进入具体处理函数trap()，并在完成处理后恢复寄存器A、B的原值（），RTI完成中断返回。
+
+调用trap()函数之前，栈帧结构如下
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
+函数中的b、a是push压入的寄存器b和a，pc为返回指令地址（processcounter）fc是中断错误码
+
+在trap()函数中
+```c
+trap(int b, int a, int fc, int pc)
+{
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
+trap函数的功能是根据栈帧中的fc获得中断码，如果执行键盘的中断，那么coun就加一，执行时钟中断current也加一。
+
+## Main的执行
+
+xem根据编译之后产生的v9文件格式，jump到main函数开始执行，具体地分析功能如下
+```c
+main()
+{
+//开始的数值
+  current = 0;
+  count = 0;
+
+//时钟中断间隔
+  stmr(2000);
+//处理中断函数的地址
+  ivec(alltraps);
+//中断
+  asm(STI);
+
+//时钟中断数小于2000次且键盘中断小于10次时，每次打键盘就产生0、1的output
+
+  while (current < 2000 && count < 10) {
+    if (count & 1) out(1, '1'); else out(1, '0');
+  }
+
+  halt(0);
+}
+```
