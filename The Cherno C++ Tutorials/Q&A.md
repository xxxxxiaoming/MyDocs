# TheCherno C++ Lessons

## How The C++ Compiler Works

### C++ Compiler's responsibility

1. pre-processing .cpp file such as #include(copy & paste), #define, #if #ifdef

2. generate .obj file for every compiling unit.

3. optimize your code. 

### An example of what ```#include``` doing (trust me, it's just coping and pasting)

Suppose now we have these two source files.

```C++ {.line-numbers}
// Math.h
}

// Yes, this file just have an end brace, nothin else.

// Main.cpp

int main()
{
    int reuslt = 1;
#include "Math.h"

```

Press ctrl + F7 and compile the Main.cpp and ***you will find that the compiling is successful!!!***

Select Project -> Properties -> C/C++ -> Preprocess -> Preprocess to a file -> Yes

Press ctrl + F7 to re-compile the Main.cpp

Now you can find Main.i file in the following path:

```
D:\GitLab\Repo\C++\HelloWorld\HelloWorld\x64\Debug
```

Open this Main.i, you can see that:

```C++ {.line-numbers}
#line 1 "D:\\GitLab\\Repo\\C++\\HelloWorld\\HelloWorld\\Mian.cpp"

int main()
{
	return 0;
#line 1 "D:\\GitLab\\Repo\\C++\\HelloWorld\\HelloWorld\\Math.h"
}
#line 6 "D:\\GitLab\\Repo\\C++\\HelloWorld\\HelloWorld\\Mian.cpp"
```

Yes, the compiler copied the whole Math.h file and pasted where the ```#include``` is. Simply lovely.

### What is compiling unit, is it true that a .cpp file is a compiling unit?

Generally, a compiling unit will produce a .obj file. 

But it doesn't mean that a .cpp file is a compiling unit. 

A .cpp file is just something that we can feed code to the compiler. 

Actually a .cpp file can be serveral compile units. 

Yes, you can ```#include``` serveral .cpp files a .cpp file. See https://chatgpt.com/share/68286e49-7274-800f-8b34-3b705777d0b8

### An example that compiler can optimize your code

Suppose that we have this source file:

```C++ {.line-numbers}
// Math.cpp
int Add()
{
    return 5 + 2;
}
```

1. Select ***Project -> HelloWorld properties -> C/C++ -> Optimization -> Maximum Optimization(Favor Speed)***

2. Remember to close basic runtim check by selecting ***Project -> HelloWorld properties -> C/C++ -> Code Generation -> Basic Runtime Check -> Default***. Otherwise, you may get a "Build Failed".

3. Select ***Project -> HelloWorld properties -> C/C++ -> Output Files -> Assembler output -> Assembly Only Listing.***

Now, you can see a Math.asm file in the path :

```
D:\GitLab\Repo\C++\HelloWorld\HelloWorld\x64\Debug
```

And from this assembly file, you can see the assembly code of function Add :

```C++ {.line-numbers}
?Add@@YAHXZ PROC					; Add, COMDAT
; File D:\GitLab\Repo\C++\HelloWorld\HelloWorld\Math.cpp
; Line 2
$LN4:
	sub	rsp, 40					; 00000028H
	lea	rcx, OFFSET FLAT:__67D1A559_Math@cpp
	call	__CheckForDebuggerJustMyCode
	mov	eax, 7
; Line 4
	add	rsp, 40					; 00000028H
	ret	0
?Add@@YAHXZ ENDP					; Add
_TEXT	ENDS
END
```
You can see that the compiler computed the result of 5 + 2 while compiling instead of storing the constant values 5 and 2 in the register :

```C++ {.line-numbers}
mov eax, 7
```

This kind of optimization is so-called "constant folding".

## Variables in C++

### How much space does a boolean variable take in memory?

Actually, although 1 bit of memory space is enough for a boolean to present, it takes 1 byte of space in memory.

Because **computer addressing memory by byte, not by bit**, that's it.

## C++ Header Files

### What is the difference between head files with .h extension and head files without?

Head files with .h head files are c head files, otherwise, they are c++ head files. Commonly seen in the c standard libs header files and c++ standard libs header files.

## How to debug C++ in Visual Studio 2022

### Viewing memory

```C++ {.line-numbers}
int main()
{
	int variable = 17;
	variable++;
	
	return 0;
}
```

Add a breakpoint in the following line (**<font color="gold">remeber to turn off the optimization!!!</font>**):

```C++ {.line-numbers}
variable++;
```

Start debuging, stop at the breakpoint.

Choose Debug -> Windows -> Memory -> Memory 1/2/3

Type "&variable" inside "Adress" -> Enter

Now you should see the memory content of the variable "variable" below.

![viewing memory of variable](image.png)

### Viewing Disassembly Code

**<font color = "gold">Under debug, when your program runs into a breakpoint</font>**，you can check the assemably code by two ways.

One way by right clicking the mouse and selecting the "Go to Disassembly" option of the menu.

Another way is the shortcut-keys way. Press Ctrl+K, and then press G. Remeber releasing after pressing Ctrl+K.

You can see the following window if there's no accidents.

![disassembly](image-1.png)

## Setup Your C++ Projects in Visual Studio 2022

### Filters VS Folders

There are filters in Visual Studio. Filters are something that Visual Studio helps you to classify your files. They are not existed in your disk.

Folders are real folders that created actual folders in your disk.

Your can treat filters as virtual folders.

The following button supply you a way to switch between filters'view and folders'view.

![switch between two views](image-2.png)

### Setup Directories For Different Kinds Of Output Files

Two output directoires are suggested to config in order to manager your project.

The first one is the "Output Directory", and the secord one is the "Intermediate Directory".

Choose Project -> YourProject Properties -> General

Setup the above configurations as the following:

![Configurations](image-3.png)

You can check the meanings of all the macros that vs 2022 provides for you here.

![Marcros](image-4.png)