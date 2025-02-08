# V8-Sandbox-Bypass-via-wrapper-and-call-target-mismatch-in-wasm

Issue: [336009921](https://issues.chromium.org/issues/336009921)

This issue spans a long time period, so the latest poc and exploit differ significantly from the initial POC submitted when the issue was reported.

## RCA

When calling a WebAssembly function from JavaScript, there are 3 stages:

```
JS -> JS-to-Wasm Wrapper -> Wasm Internal Function
```

The `JS-to-Wasm` Wrapper exists in function object as a code wrapper. The WebAssembly internal function can only be fetched from `SharedFunctionInfo->function_data`. While we cannot directly modify the code object, there is no validation to ensure the `JS function`, `wrapper`, and `internal function` match. Exchanging `code wrapper`s or `function_data`s between functions does not trigger any violations.

The wrapper serves a specific purpose — functions only share the same wrapper when their signatures match. If the signatures of the wrapper and internal function mismatch, it creates opportunities for stack-based read and write operations, potentially enabling arbitrary code execution.

Using out-of-bounds reads, we can pass a text section address to the internal function to calculate the code base and subsequent gadget addresses. With out-of-bounds writes, we can write ROP chains to the stack for arbitrary code execution.

Memory corruption can occur before the return. We must satisfy certain constraints in the wrapper code after returning from the internal function. Below is the code from a fresh release build for commit `0379746345d071adf30b4e084183734999b7aede` using the report's build flags. (Note that different build flags affect code generation—for example, debug builds include additional checks here.)

```
   0x5575b22c0084:      movabs rdi,0x7f3effffffffffff
   0x5575b22c008e:      and    rdi,QWORD PTR [rcx+rbx*1]
   0x5575b22c0092:      mov    rbx,QWORD PTR [rdi+0x13]
   0x5575b22c0096:      mov    esi,DWORD PTR [rdi+0x7]
   0x5575b22c0099:      or     rsi,QWORD PTR [r13+0x1e0]
   0x5575b22c00a0:      mov    QWORD PTR [rbp-0x20],rdx
   0x5575b22c00a4:      call   rbx
   0x5575b22c00a6:      mov    rbx,QWORD PTR [rbp-0x20]
   0x5575b22c00aa:      mov    DWORD PTR [rbx],0x0
   0x5575b22c00b0:      movabs rax,0x1db000000069
   0x5575b22c00ba:      mov    rcx,QWORD PTR [rbp-0x18]
   0x5575b22c00be:      mov    rsp,rbp
   0x5575b22c00c1:      pop    rbp
   0x5575b22c00c2:      cmp    rcx,0x1
   0x5575b22c00c6:      jg     0x5575b22c00cb
   0x5575b22c00c8:      ret    0x8
   0x5575b22c00cb:      pop    r10
   0x5575b22c00cd:      lea    rsp,[rsp+rcx*8]
   0x5575b22c00d1:      push   r10
   0x5575b22c00d3:      ret
```

First constraint: The address at `rbp-0x20` must be writable to prevent invalid memory access from the instruction sequence `mov rbx,QWORD PTR [rbp-0x20]; mov DWORD PTR [rbx],0x0;`

Second constraint: The value at `rbp-0x18` must be 0 or small to prevent `lea rsp,[rsp+rcx*8]` from exceeding our padding data.

## POC

commit `0379746345d071adf30b4e084183734999b7aede`

see [poc.js](poc.js)

```
➜  x64.release git:(main) ./d8 --sandbox-testing poc.js
Sandbox testing mode is enabled. Write to the page starting at 0x3882456cb000 (available from JavaScript as `Sandbox.targetPage`) to demonstrate a sandbox bypass.

## V8 sandbox violation detected!

Received signal 11 SEGV_ACCERR 3882456cb000

==== C stack trace ===============================

 [0x5644286660b7]
 [0x7f914e0bf420]
 [0x3882456cb000]
[end of stack trace]
[1]    2512366 segmentation fault (core dumped)  ./d8 --sandbox-testing poc.js
```

The full exploit code is available in my SekaiCTF 2024 challenge:

[dist](https://github.com/project-sekai-ctf/sekaictf-2024/tree/main/pwn/context-reducer)

[exp.js](https://github.com/rycbar77/writeups/blob/master/2024/sekaictf/ContextReducer/exp.js)
