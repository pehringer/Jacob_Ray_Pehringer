# Always Have A Plan B... Or Plan9?
**Simd Package:** [github.com/pehringer/simd](https://github.com/pehringer/simd)

I want to take advantage of Go's concurrency and parallelism for some of my upcoming projects. It allows for some serious number crunching capabilities. But what if I want EVEN MORE POWER?!? Enter SIMD, **S**ame **I**nstruction **M**uliple **D**ata. Simd instructions allow for parallel number crunching capabilities right down at the hardware level. Many programming languages either have compiler optimizations that use simd or libraries that offer simd support. However (as far as I can tell) Go's compiler does not utilizes simd. And I cound not find a general propose simd package that I liked. ***I just want a package that offers a thin abstraction layer over arithmetic and bitwise simd operations***. So like any good programmer I decided to slightly reinvent the wheel and write my very own simd package. How hard could it be?

After doing some preliminary research I discovered that Go uses its own internal assembly language called Plan9. I consider it more of a assembly format rather than its own language. Plan9 uses the target platforms instructions and registers with slight modifications to their names and usage. This means that x86 Plan9 is different then say arm Plan9. Overall pretty weird stuff. I am not sure why the Go team went down this route. Maybe it simplifies the compiler by having this bespoke assembly format.
# Plan9 Crash Course
I always find learning by example to be the most informative.
So lets Go (haha) over a simple example.
```
example
 ┣━ AddInts_amd64.s
 ┗━ main.go
```
**example/AddInts_amd64.s**
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
**LINE 1**: The file contains ```amd64``` specific instructions, so we need to include a Go build tag to make sure Go does not try to compile this file for non x86 machines.

**LINE 3**: You can think of this line as the functions declaration. ```TEXT``` declares that this is a function or text section. ```·AddInts(SB)``` specifies our functions name. ```4``` represents "NOSPLIT" which we need for some reason. And ```$0``` is the size of the function's stack frame (used for local variables). It's zero in this case because we can easily fit everything into the registers.

**LINE 4 & 5**: Go's calling convention is to put the function arguments onto the stack. So we ```MOV```e both ```L```ong 32-bit values into the ```AX``` and ```BX``` registers by dereferencing the frame pointer (```FP```) with the appropriate offsets. The first argument is stored at offset ```0```. The second argument is stored at offset ```8``` (int’s only need 4 bytes but I think Go offsets all arguments by 8 to maintain memory alignment).

**LINE 6**: ```Add``` the ```L```ong 32-bit value in ```AX``` (left) with the ```L```ong 32-bit value in ```BX```. And store the resulting ```L```ong 32-bit value in ```AX```.

**LINE 7 & 8**: Go's calling convention (as far as I can tell) is to put the function return values after its arguments on the stack. So we ```MOV```e the ```L```ong 32-bit values in the ```AX``` register onto the stack by dereferencing the frame pointer (```FP```) with the appropriate offset. Which is 16 in this case.
  
**example/main.go**
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

**LINE 5**: This is the forward functions declaration for our Plan9 function. Since they both share the same name (```AddInts```) Go will link them together during compilation.

**LINE 8**: We can now use our Plan9 function just like any other function.

# My Design Plan . . . . 9
Now that we are Go assembly experts I can explane the overall design of my package. ***My main goal for the package was to offer a thin abstraction layer over arithmetic and bitwise simd operations***. Basically I wanted a set of functions that would allow me to perform simd operations on slices.

Let's go over a mini version of my projects.
```
example
 ┣━ internal
 ┃   ┗━ addition
 ┃       ┣━ AddInts_amd64.s
 ┃       ┗━ addition_amd64.go
 ┣━ init_amd64.go
 ┗━ example.go
```
I first created a set of private function pointers with corresponding public functions that wrapped around them. By default the private pointers pointed to fallback software implemations.
  
**example/example.go**: 
```
 1  package example
 2
 3  func fallbackAddInts(left, right) int {
 4     return left + right
 5  }
 6
 7  var func addInts(left, right) int = fallbackAddInts
 8
 9  func AddInts(left, right) int {
10    return addInts(left, right)  
11  }
```
I then create internal packages that contained the arciticuture spefic Plan9 functions:
