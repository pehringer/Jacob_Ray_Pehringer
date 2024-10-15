# Go Go ~~Power Rangers~~ Gophers!
# Plan9 Is Werid
# Plan9 Crash Course
I always find learning by example to be the most informative.
So lets Go (haha) over a simple example!
```
CoolBeans
 ┣━ AddInts_amd64.s
 ┗━ main.go
```

### AddInts_amd64.s
```
1  // +build amd64
2
3  TEXT ·AddInts(SB), 4, $0
4      MOVL    left+0(FP), AX
5      MOVL    right+8(FP), BX
6      ADDL    BX, AX
7      MOVL    AX, int+16(FP)
8      RET
```
### main.go
```
1  package main
2
3  import "fmt"
4
5  func AddInts(left, right) int
6
7  func main() {
8      fmt.Println("1 + 2 = ", AddInts(1, 2))
9  }
```
---
The file contains x86 specific instructions, so we need to include a Go build tag to make sure Go does not try to compile this file for non x86 machines.
```
1  // +build amd64
```
---
You can think of this line as the functions declaration. **TEXT** declares that this is a function or text section. **·AddInts(SB)** specifies our functions name. **4** represents "NOSPLIT" which we need for some reason. And **$0** is the size of the function's stack frame (used for local variables). It's zero in this case because we can easily fit everything into the registers.
```
3  TEXT ·AddInts(SB), 4, $0
```
---
Go's calling convention is to put the function arguments onto the stack. So we **MOV**e both **L**ong 32-bit values into the **AX** and **BX** registers by dereferencing the frame pointer (**FP**) with the appropriate offsets. The first argument is stored at offset **0**. The second argument is stored at offset **8** (int’s only need 4 bytes but I think Go offsets all arguments by 8 to maintain memory alignment).
```
4      MOVL    left+0(FP), AX
5      MOVL    right+8(FP), BX
```
---
**Add** the **L**ong 32-bit value in **AX** (left) with the **L**ong 32-bit value in **BX**. And store the resulting **L**ong 32-bit value in **AX**.
```
6      ADDL    BX, AX
```
---
Go's calling convention (as far as I can tell) is to put the function return values after its arguments on the stack. So we **MOV**e the **L**ong 32-bit values in the **AX** register onto the stack by dereferencing the frame pointer (**FP**) with the appropriate offset. Which is 16 in this case.
```
7      MOVL    AX, int+16(FP)
8      RET
```
---
This is the forward functions declaration for our Plan9 function. Since they both share the same name (**AddInts**) Go will link them together during compilation.
```
5  func AddInts(left, right) int
```
---
We can now use our Plan9 function just like any other function!
```
8      fmt.Println("1 + 2 = ", AddInts(1, 2))
```
---