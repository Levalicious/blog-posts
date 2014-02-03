# Introduction
In [Enigma Studio](http://www.braincontrol.org/enigma.php) we have a number of *operators*. Each operator performs a certain operation on a data structure (e.g. mesh, bitmap, ...) based on a set of *parameters*. The parameters can be of different type; e.g.  1-, 2-, 3- and 4-dimensional `int` or `float` vector, flag, enumeration or string. When an operator is executed to perform its operation the operator's *execute handler* gets called. In the implementation of the execute handler there has to be a way for the programmer to access the operator's set of parameters. Coming up with a programmer-friendly way to access the parameters is not that straightforward, because all obvious approaches require writing code where the individual parameters have to be accessed explicitly by calling a getter function or by accessing an array. Look at the following code for an illustration. Note, that all code presented in the following is purely for educational purpose. It is by no mean intended to be a feature complete operator stacking system.

``` cpp
enum OpParamType
{
    OPT_INT,
    OPT_FLT,
    OPT_STR
};

struct OpParam
{
    OpParamType type;
    int         chans;
 
    union
    {
        float   flts[4];
        int     ints[4];
    };

    std::string str; // can't be in union
};

struct Op
{
    void execute()
    {
        execFunc(this);
    }

    void (* execFunc)(Op *op); // pointer to execute handler
    OpParam params[16];
    size_t  numParams;
};

// execute handler
// param[0]: width
// param[1]: height
// param[2]: UV coordinates
void OpExec(Op *op)
{
    int size = op->params[0].ints[0]*op->params[1].ints[0];
    Vector2 uv(op->params[2].flts[0], op->params[2].flts[1]);
}
```

This approach has some issues. First of all, it is very error-prone because a parameter is solely identified by its index (we could use strings, but it doesn't make it much better). Furthermore, whenever the order of parameters or the type of a parameter changes, each access to the `params` array of the respective operator has to be changed as well. This sucks and this is something no programmer wants to do. Finally, the syntax used above is not very readable; especially when user-defined data types like `Vector2` are used. Let's do something about it!

# Concept
In the following I'll present the approach I've come up with. It's based on directly passing the parameter data to the execute handler. This way the only thing that has to be done is to declare/define the execute handler accordingly to its parameter definition. Meaning, that the order of arguments and the type of arguments has to match with the definition of the operator's set of parameters. The execute handler from above e.g. would simply look like:

``` cpp
void OpExec(Op *op, int width, int height, const Vector2 &uv)
{
    int size = width*height;
}
```

This is exactly what you are used to when programming. The code is not cluttered with accesses to the `params` array anymore, nor is it as error-prone. Parameters are identified by their names and there is only one source of error (the function's declaration/definition) when it comes to the parameters' types and order.

# Implementation
In order to implement such a system we have to go low-level. This means crafting handwritten assembly code to implement the underlying calling convention's argument passing and function calling ourselves. Consequently, you need profound knowledge of C++, x86/x64 assembly and calling conventions to understand what follows. Furthermore, the following code is mostly neither platform nor assembler independent, because of differences in how the operating systems implement certain calling conventions and the assembly syntax. I used Windows 8, Visual C++ 2012 and MASM.

## x86 Implementation
On x86 I use Visual C++'s standard calling convention `CDECL`. `CDECL` as well as `STDCALL` (which differ only in who, the caller or the callee, cleans the stack) are very easy to implement by hand, because all arguments are passed on the stack. To do so, all parameters that should be passed to an execute handler are copied into an array first. This array is passed to the `callFunc()` function, which creates a stack frame, copies the data to the stack and calls the execute handler. The following code is used to copy each parameter to the array. To account for call-by-value and call-by-reference arguments the data's address or the data it-self is copied, depending on the parameter type.

``` cpp
// implemented in an .asm file
extern "C" void callFunc(const void *funcPtr, const size_t *stack, size_t numArgs);

class Op
{
public:
    Op(const void *execFunc, const OpParam *params, size_t numParams) : 
       m_execFunc(execFunc),
       m_numParams(numParams)
    {
        // copy over parameters
        std::copy(params, params+numParams, m_params);
    }

    void execute()
    {
        size_t args[16+1] = {(size_t)this}; // pass pointer to operator

        for (uint32_t i=0; i<m_numParams; i++)
        {
            const OpParam &p = m_params[i];
            switch (p.type)
            {
            case OPT_STR:
                args[i+1] = (size_t)&p.str;
                break;
            default: // float and integer types (it's a union!)
                args[i+1] = (p.chans > 1 ? (size_t)p.ints : (size_t)p.ints[0]);
                break;
            }
        }

        callFunc(m_execFunc, args, m_numParams+1);
    }
 
private:
    const void * m_execFunc;
    size_t       m_numParams;
    OpParam      m_params[16];
};
```

The implementation of `callFunc()` is given below. I used the MASM assembler because its shipped with Visual C++. I don't use inline assembly, because Visual C++ doesn't support x64 inline assembly and I wanted to have both `callFunc()` implementations in one file.

```
.686
.model flat, c             ; calling convention = cdecl
.code
callFunc proc funcPtr:ptr, stack:ptr, numArgs:dword
    push    esi            ; save callee saved regs
    push    edi
    mov     eax, [funcPtr] ; store arguments on stack before new
    mov     ecx, [numArgs] ; stack frame is installed (they
    mov     esi, [stack]   ; won't be accessible anymore)
    push    ebp
    mov     ebp, esp
    sub     esp, ecx       ; reserve stack space for parameters
    sub     esp, ecx       ; (4 * number of arguments)
    sub     esp, ecx
    sub     esp, ecx
    mov     edi, esp       ; copy parameters on stack
    rep     movsd
    call    eax            ; call execute handler
    mov     esp, ebp       ; remove stack frame
    pop     ebp
    pop     edi            ; restore callee saved regs
    pop     esi
    ret
callFunc endp
end
```

## x64 Implementation
The x64 code is slightly more complex. The reason for this is that on x64 there is only one calling convention called `FASTCALL`. In this calling convention not all arguments just go onto the stack, but some of them have to be passed in registers. Furthermore, there are some differences in how call-by-value for `structs` works. Structures which's size does not exceed 64 bytes are passed in registers or on the stack. For bigger structures a pointer is passed. If the structure is passed by value, its data is first copied to the *home space* and the passed pointer points there. As I only needed call-by-reference for user-defined data types bigger than 64 bytes, I didn't have to care about this. Furthermore, the stack pointer `RSP` has to be 16 byte aligned when calling the execute handler. You might be wondering why 16 and not 8 byte. The reason for this is that it simplifies the alignment of SSE data types for the compiler (`sizeof(__m128) = 16` bytes). Describing the calling convention in more detail is beyond the scope of this post, but there are plenty of good articles online.
As I mentioned before already, Visual C++ does not support x64 inline assembly. Therefore, I had to go for an extra `.asm` file which gets assembled by MASM64. The steps undertaken by the `callFunc()` implementation are:

1. Allocate stack space for the parameters that won't be passed in registers.
1. Align the stack pointer `RSP` on 16 byte boundary.
1. Copy the 5<sup>th</sup>, 6<sup>th</sup>, ... arguments to the stack.
1. Copy the first four arguments to the registers `RAX`, `RDX`, `R8` and `R9`.
1. Call the execute handler.

In the following the source code of the x64 `callFunc()` function is depicted. You should note that MASM64 does not support using the arguments declared in the `proc` statement. This is why I directly access the arguments via the registers. For the ease of much simpler manual register allocation I copy them in the beginning to memory.

```
.data
numArgs dq 0
stack   dq 0
funcPtr dq 0

.code
callFunc proc funcPtrP:ptr, stackP:ptr, numArgsP:dword
    push    rdi
    push    rsi

    mov     [funcPtr], rcx  ; simplifies register allocation
    mov     [stack], rdx    ; don't use passed variable names!
    mov     [numArgs], r8   ; MASM doesn't support this for x64

; ----- allocate stack space and copy parameters -----

    mov     rcx, [numArgs]  ; number of passed arguments
    sub     rcx, 4          ; 4 of them will be passed in regs
    cmp     rcx, 0          ; some left for the stack?
    jng     noParamsOnStack
    lea     r10, [rcx*8]    ; calculate required stack space
    sub     rsp, r10        ; reserve stack space

    mov     r11, rsp        ; align stack pointer to 16 bytes
    and     r11, 15         ; mod 16
    jz      dontAlign       ; is already 16 bytes aligned?
    sub     rsp, r11        ; perform RSP alignment
dontAlign:

    mov     rdi, rsp        ; copy parameters to stack
    mov     rsi, [stack]
    add     rsi, 4*8        ; first 4 parameters are in regs
    rep     movsq

noParamsOnStack:

; ----- store first 4 arguments in RCX, RDX, R8, R9 -----

    mov     rsi, [stack]
    mov     r10, [numArgs]  ; switch (numArgs)
    cmp     r10, 4
    jge     fourParams
    cmp     r10, 3
    je      threeParams
    cmp     r10, 2
    je      twoParams
    cmp     r10, 1
    je      oneParam
    jmp     noParams

fourParams:                 ; fall through used
    mov     r9, [rsi+24]
    movss   xmm3, dword ptr [rsi+24]
threeParams:
    mov     r8, [rsi+16]
    movss   xmm2, dword ptr [rsi+16]
twoParams:
    mov     rdx, [rsi+8]
    movss   xmm1, dword ptr [rsi+8]
oneParam:
    mov     rcx, [rsi]
    movss   xmm0, dword ptr [rsi]
noParams:
            
; ----- call execute handler for operator -----

    sub     rsp, 20h        ; reserve 32 byte home space
    call    [funcPtr]       ; call function pointer

; ----- restore non-volatile registers -----

    mov     rsp, rbp
    pop     rsi
    pop     rdi
    ret
callFunc endp
end
```

# Conclusion
The presented solution for implementing execute handlers works very well for me. Nevertheless, it is important to get the handler definitions/declarations right in order to avoid annoying crashes. There are no validity checks at compile-time. Apart from argument order, parameters cannot be passed call-by-value (at least in x86, in x64 call-by-value silently turns into call-by-reference because the parameter data is not copied to home space; which is not much better). You can download the full source code from my [github](https://github.com/geidav/op-sys) repository.