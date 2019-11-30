# DLL Hijack on music software
A brief note on using MITM attack to extract vocal files from a music software's RAM


## Overview: 
Softwares nowadays usually use a lot of dll to reduce their file size and to reduce their code complexities. However, this also poses a security risk as valuable information is exposed during the communication with the dll. Using api interception and dll redirection, one can easily extract and manipulate the data as a “man in the middle”.

Note: This is just a brief guide to provide some insights for beginners to dig into the reverse engineering field. This is NOT a complete guide for reasons to protect the hacked software.

## Background:
I accidentally came across the area of reverse engineering when I needed to study a large amount vocal singing dataset with lyrics. Sadly, there was little available out there so I decided to hack my way out.

## Prerequisite:
1. I tried a few disassembler debuggers and found that [x64dbg](https://x64dbg.com/#start) to be the best free alternative out there.

   Download it and open "Options"->"Preferences". Disable the TLS callback and also ignore all exceptions in the range of 00000000-      FFFFFFFF. Otherwise, you are likely to get a lot of pauses during debugging.

2. Download [Visual Studio IDE](https://visualstudio.microsoft.com) and open a dll project. Set the debug mode to x86 as the dll we hack is 32bit.

   We are going to compile some .asm files for better control of the interception process. Right-click on the project in solution explorer to open "Build Dependencies"->"Build Custumizations...", tick the masm option to enable asm compilation.

3. You also need a documentation of the target dll. you may also need the header files of the dll if the function to intercept has self-defined return type or argument types.

## Create the fake dll:
Our plan is to replace the target dll with a fake dll, so the software will call the fake dll instead. We are going to manipulate the data in the call, and redirect it to the target dll afterwards to let it do the actual stuff.

1. First, use `dumpbin \exports [targetdll]` to get the dll exported functions. We choose those we do not want to intercept and set them to forward directly to the target dll when called from the fake dll. For each of the functions,  write 
```
#pragma comment(linker, "/export:[function name to forward]=[renamed target dll].[same function to the left],@[ordinal no.]")
```
   to the main file. You may find codes to automate this process in (1).

2. Then, write your intercepted function definitions. Here we assume the intercepted function has a prototype of `int foo(bool x)`.
```
extern "C" {
	void* funcs[1];
}

extern "C" int foo_bridge(BOOL x)

__declspec (dllexport) int foo(BOOL x)
{
	 HINSTANCE handle;
	 FARPROC function;

	 handle = LoadLibraryA("[renamed target dll]");

	 function = GetProcAddress(handle, "foo");
	 
   // Extra code to manipulate the data goes here.
   
	 funcs[0] = function;
	 return foo_bridge(x);
}
```
   If you use c++, use `extern "C"` prefix before declarations to avoid name mangling by c++ compiler, as they need to be referenced in .asm compiled by MASM. The above code first declares the exported function to be `int foo(BOOL x)`, loads the target dll and get the real function's address. The address is saved in `funcs[0]`, which is a global variable that is used by `foo_bridge()` as we will see later to call the real function. Code to manipulate or use param(x) can be inserted anywhere in the function.

3. Write asm for `foo_bridge()`
```
function_index equ	 0			 ;; index of function to call. Remember we set func[0] = function?
.586
.MODEL FLAT, C
.STACK
.DATA
.CODE

;; EXTERNs here
EXTERN funcs: DWORD	;; array of function pointers

_x$ = 8                                            ; bool, size = 1
foo_bridge PROC

	LEA		EAX, funcs
	MOV		EAX, [EAX + function_index * 4]
	jmp		DWORD PTR EAX

foo_bridge ENDP

```
   So `foo_bridge()` will directly jump to the address funcs[0] to execute the real dll function when called!

4. Write a .def file to tell the linker by what name and ordinal number we want the intercepted function to be exported.
```
LIBRARY		[original name of target dll]
EXPORTS
	foo=foo @[ordinal number]
```

5. That's it! Compile it and we should see a dll in the project's Dubug folder. Now rename the target dll and place the fake dll into its original position. Rerun the software and we can intercept it now.

## Complications
1. Sometimes we don't know what functions we want to intercept. So open x64dbg with the software. Next, click "Symbols" and double click the software.exe. Then, click the little "retro mobile phone" icon that is labeled "Find Intermodular Calls" and specific the target dll name as the search pattern. Toggle breakpoints on functions you may want to intercept and redebug the software to see if these functions are your target.

2. Sometimes I found that the software crashed when using the fake dll. After some debugging, I noticed that when the call returned to software.exe after calling fake dll, the passed arguments are not yet cleared on the stack compared to using the real dll! It appears that the intercepted function is using the `__stdcall` calling convention that relies on the callee to clean up the stack. So I went back to VS and add the `__stdcall` or the `CALLBACK` prefix to the function and the problem is solved.

3. It appears that using `__stdcall` will make the exported function to be named `function@some number` instead of `function`. So go back to the corresponding .asm file and change the function name to adjust to the changed version.

4. I originally made a fake dll that only forward function and did nothing else, but that still caused the program to crash with the warning that a function named `_` is not found. So I used `dumpbin` again to examine the fake dll and found that function `_` was not exported. Later I found out that when forwarding functions, if the function is named `_`, the preprocessor statement should be written as
```
#pragma comment(linker, "/export:__=[renamed dll]._,@[ordinal number]")
```
   with two `_` for the first name. Otherwise, the function will simply be skipped.

## References
Huge thanks to the people providing great tutorials and examples on how to hijack dll.

(1) https://github.com/kevinalmansa/DLL_Wrapper
(2) https://www.exploit-db.com/docs/english/13140-api-interception-via-dll-redirection.pdf
