# V8



* Terms
- ICU: International Components for Unicode data
- CSA: CodeStubAssembler
- Torque: DSL for codegen

* Builtin
Builtins can be seen as chunks of code that are executable by the VM at runtime.

V8’s builtins can be implemented using a number of different methods (each with different trade-offs):
    1. Platform-dependent assembly language
    2. C++
    3. JavaScript
    4. CodeStubAssembler
    5. Torque

CodeStubAssembler provides efficient low-level functionality that is very close to assembly language while remaining platform-independent and preserving readability.

Torque is a V8-specific domain-specific language that is translated to CodeStubAssembler. As such, it extends upon CodeStubAssembler and offers static typing as well as readable and expressive syntax.

#+begin_src org
CPP: Builtin in C++. Entered via BUILTIN_EXIT frame.
     Args: name
TFJ: Builtin in Turbofan, with JS linkage (callable as Javascript function).
     Args: name, arguments count, explicit argument names...
TFS: Builtin in Turbofan, with CodeStub linkage.
     Args: name, explicit argument names...
TFC: Builtin in Turbofan, with CodeStub linkage and custom descriptor.
     Args: name, interface descriptor
TFH: Handlers in Turbofan, with CodeStub linkage.
     Args: name, interface descriptor
BCH: Bytecode Handlers, with bytecode dispatch linkage.
     Args: name, OperandScale, Bytecode
ASM: Builtin in platform-dependent assembly.
     Args: name, interface descriptor
#+end_src



* JIT Code

kStartOfStrongFieldsOffset?

| Field                                | Size            |
|--------------------------------------+-----------------|
| HeapObject header / map              | 8B              |
| DeoptimizationData / InterpreterData | 8B (tagged)     |
| PositionTable (ByteArray)            | 8B (tagged)     |
| InstructionStream (ptr)              | 8B (tagged)     |
| instruction_start                    | 16B (CHERI cap) |
| instruction_sentry                   | 16B (sealed)    |
| flags                                | 4B (uint32)     |
| instruction_size                     | 4B (int)        |
| metadata_size                        | 4B (int)        |
| inlined_bytecode_size                | 4B (int)        |
| osr_sentry                           | 16B (sealed)    |
| osr_offset                           | 4B (int32)      |
| handler_table_offset                 | 4B (int)        |
| unwinding_info_offset                | 4B (int32)      |
| constant_pool_offset                 | 4B (int)        |
| code_comments_offset                 | 4B (int)        |
| builtin_id                           | 2B (int16)      |
| padding                              | ~6B             |

- flags

| Bits    | Field                     | Description                              |
|---------+---------------------------+------------------------------------------|
| [0:4]   | CodeKind                  | TURBOFAN, MAGLEV, BASELINE, BUILTIN, ... |
| [4:5]   | is_turbofanned            | Compiled by TurboFan                     |
| [5:29]  | stack_slots               | Number of stack slots (24 bits)          |
| [29:30] | marked_for_deoptimization | Pending deopt on next call               |
| [30:31] | embedded_objects_cleared  | Weak refs to embedded objects cleared    |
| [31:32] | can_have_weak_objects     | Code may reference weak heap objects     |


** InstructionStream

|               Offset | Field           | Size | Type       | Notes                            |
|----------------------+-----------------+------+------------+----------------------------------|
|                   +0 | map (header)    | 8B   | tagged     | HeapObject map / object type     |
|                   +8 | code            | 8B   | tagged ptr | Back-pointer to Code object      |
|                  +16 | relocation_info | 8B   | tagged ptr | ByteArray with reloc entries     |
|                  +24 | body_size       | 4B   | int32      | instruction_size + metadata_size |
|                  +28 | [padding]       | 4B   | —          | Alignment to pointer size        |
|                  +32 | instruction     | —    | -          | instruction_start() base         |
| +32+instruction_size | metadata        | -    | -          | metadata_start() base            |


*** Metadata

| Section         | Consumer             | Trigger                    | Purpose                                                |
|-----------------+----------------------+----------------------------+--------------------------------------------------------|
| Safepoint table | V8 GC                | GC pause / allocation      | Find all tagged pointers (roots) in live frames        |
| Handler table   | V8 runtime           | JS throw                 | Find the catch/finally handler for the current frame   |
| Trampoline      | Deoptimizer          | Optimization invalidation  | Bail out of optimized code back to interpreter         |
| Constant pool   | CPU / assembler      | ~ldr~ of large constant    | Pool of large immediates that cannot be encoded inline |
| Unwinding info  | OS / libunwind / gdb | C++ exception, backtrace() | DWARF frame description for OS-level stack unwinding   |

** Safepoints
*What it solves*: The garbage collector needs to move objects on the heap. When GC runs, it must find every pointer (tagged reference) in the currently executing code — both
  in registers and on the stack — to update them. But it can only safely do this at specific "safe" moments.

*How it works*: A safepoint is a PC offset where the GC is allowed to interrupt execution. At each such point, the safepoint table records:
  - Which registers hold tagged (GC-managed) pointers
  - Which stack slots hold tagged pointers (as a bitvector)
  - For deoptimizable code: a deopt index and trampoline PC

Safepoints are inserted at every call site (calls can trigger GC), and sometimes at loops.

 | Field                 | Size (bytes)  | Present when   | Description                                        |
 |-----------------------+---------------+----------------+----------------------------------------------------|
 | PC offset             | 1–4 (pc_size) | Always         | Offset from instruction_start of the safepoint     |
 | Deopt index           | 0–4           | has_deopt_data | Index into DeoptimizationData                      |
 | Trampoline PC         | 0–4           | has_deopt_data | PC offset of deopt trampoline stub                 |
 | Tagged register mask  | 0–4           | Always         | Bitmask of registers holding tagged pointers       |
 | Tagged slot bitvector | N bytes       | Always         | One bit per stack slot indicating tagged pointers  |
 | Trampoline sentry     | 16 (CHERI)    | has_deopt_data | Sealed capability to trampoline (CHERI ARM64 only) |

*** Safepoint EntryConfiguration

| Bits    | Field                 | Size   | Description                                |
|---------+-----------------------+--------+--------------------------------------------|
| [0:1]   | has_deopt_data        | 1 bit  | Whether deopt index/trampoline are present |
| [1:4]   | register_indexes_size | 3 bit  | Size in bytes of register bitmask (0–4)    |
| [4:7]   | pc_size               | 3 bit  | Size in bytes of PC offset field (1–4)     |
| [7:10]  | deopt_index_size      | 3 bit  | Size in bytes of deopt index field (0–4)   |
| [10:32] | tagged_slots_bytes    | 22 bit | Number of bytes in tagged slot bitvector   |

*** Trampoline (in safepoint table)

*What it solves*: Deoptimization. When TurboFan code is running and the optimistic assumptions it made turn out to be wrong (e.g., a value is not always a Smi), V8 must "bail
   out" back to the interpreter. This is called deoptimization.

*How it works*: At each deopt point, TurboFan emits a small stub call (the deopt trampoline). The safepoint entry for that PC records:
  - deopt_index: which deopt entry in DeoptimizationData describes the frame state
  - trampoline_pc: the PC of the trampoline stub to jump to when deopt triggers

The trampoline is a small assembly stub that:
  1. Saves the current register state
  2. Calls into the deoptimizer C++ code
  3. The deoptimizer reconstructs the interpreter frame from DeoptimizationData
  4. Execution resumes in the bytecode interpreter

** Handler Table
*What it solves*: JavaScript exceptions. When throw happens inside a try/catch, the runtime needs to know where to jump to (the catch handler), and which try-block's catch
  applies to the current PC.

*How it works* (TurboFan): Return-address encoding. Each entry maps:
  return_pc_offset  →  handler_pc_offset
  When an exception is thrown, the runtime walks the stack, finds the return address at each frame, looks it up in the handler table, and if found jumps to the handler. The
  HandlerPrediction bits (3 bits) tell the runtime whether a catch, finally, async_await, or uncaught handler is expected.

Handler table is about exceptions finding catch blocks.

| Field            | Size (bytes) | Description                                       |
|------------------+--------------+---------------------------------------------------|
| return_pc_offset | 4 (int32)    | PC offset of the call's return address            |
| handler_data     | 4 (int32)    | handler_offset [3:32] + HandlerPrediction [0:3]   |
| handler_sentry   | 16 (CHERI)   | Sealed capability to handler address (CHERI only) |


| HandlerPrediction value | Meaning       |
|-------------------------+---------------|
|                       0 | catch       |
|                       1 | finally     |
|                       2 | async_await |
|                       3 | Uncaught      |

** DeoptimizationData

|                      Index | Field                | Type                       |
|----------------------------+----------------------+----------------------------|
|                          0 | TranslationByteArray | ByteArray                  |
|                          1 | InlinedFunctionCount | Smi                        |
|                          2 | LiteralArray         | WeakFixedArray             |
|                          3 | OsrBytecodeOffset    | Smi                        |
|                          4 | OsrPcOffset          | Smi                        |
|                          5 | OptimizationId       | Smi                        |
|                          6 | SharedFunctionInfo   | Object                     |
|                          7 | InliningPositions    | PodArray<InliningPosition> |
|                          8 | DeoptExitStart       | Smi                        |
|                          9 | EagerDeoptCount      | Smi                        |
|                         10 | LazyDeoptCount       | Smi                        |
| 11 + i*kDeoptEntrySize + 0 | BytecodeOffsetRaw    | Smi                        |
| 11 + i*kDeoptEntrySize + 1 | TranslationIndex     | Smi                        |
| 11 + i*kDeoptEntrySize + 2 | Pc                   | Smi                        |
| 11 + i*kDeoptEntrySize + 3 | NodeId (DEBUG only)  | Smi                        |



** RelocInfo
| Mode                       | Value | GC Root? | Tag  | Data size | Description                                |
|----------------------------+-------+----------+------+-----------+--------------------------------------------|
| NO_INFO                    |     0 | No       | —    | 0B        | Placeholder / no relocation                |
| CODE_TARGET                |     1 | Strong   | 01   | 0B        | Call/jump to another Code object           |
| RELATIVE_CODE_TARGET       |     2 | Strong   | long | 0B        | PC-relative code reference                 |
| COMPRESSED_EMBEDDED_OBJECT |     3 | Strong   | long | 0B        | Compressed (32-bit) heap object pointer    |
| FULL_EMBEDDED_OBJECT       |     4 | Strong   | 00   | 0B        | Full 64-bit heap object pointer            |
| WASM_CALL                  |     5 | No       | long | 0B        | Wasm function call target                  |
| WASM_STUB_CALL             |     6 | No       | 10   | 0B        | Wasm stub call target                      |
| EXTERNAL_REFERENCE         |     7 | No       | long | ptr       | C++ runtime function address               |
| INTERNAL_REFERENCE         |     8 | No       | long | ptr       | Absolute reference within same code object |
| INTERNAL_REFERENCE_ENCODED |     9 | No       | long | ptr       | PC-relative internal reference             |
| OFF_HEAP_TARGET            |    10 | No       | long | 0B        | Off-heap embedded builtin (far)            |
| NEAR_BUILTIN_ENTRY         |    11 | No       | long | 0B        | Near call to off-heap builtin              |
| CONST_POOL                 |    12 | No       | long | 4B int    | ARM constant pool position marker          |
| VENEER_POOL                |    13 | No       | long | 4B int    | ARM64 veneer pool position marker          |
| DEOPT_SCRIPT_OFFSET        |    14 | No       | long | 4B int    | Source script offset for deopt             |
| DEOPT_INLINING_ID          |    15 | No       | long | 4B int    | Inlining ID for deopt frame reconstruction |
| DEOPT_REASON               |    16 | No       | long | 1B        | Why deoptimization was triggered           |
| DEOPT_ID                   |    17 | No       | long | 4B int    | Index into DeoptimizationData entries      |
| DEOPT_NODE_ID              |    18 | No       | long | 4B int    | Compiler node ID (debug builds only)       |
| PC_JUMP                    |    19 | No       | long | VLQ       | Internal: large PC delta overflow          |
| CAPABILITY_CONSTANT        |    20 | No       | long | cap       | CHERI capability constant (CHERI only)     |

- RelocInfo Encoding Format

| Form    | Byte 0                      | Byte 1           | Further bytes   | When used                                    |
|---------+-----------------------------+------------------+-----------------+----------------------------------------------|
| Short   | [6-bit PC delta][2-bit tag] | —                | —               | EMBEDDED_OBJECT, CODE_TARGET, WASM_STUB_CALL |
| Long    | [6-bit mode][0b11]          | [8-bit PC delta] | see data column | All other modes                              |
| PC_JUMP | [PC_JUMP mode][0b11]        | VLQ byte 0       | more VLQ bytes  | PC delta > 63 bytes                          |

- RelocInfo GC Interaction


| Mode                       | GC Root | Visitor method         | Effect on GC compaction       |
|----------------------------+---------+------------------------+-------------------------------|
| CODE_TARGET                | Strong  | VisitCodeTarget        | Pointer updated if code moves |
| RELATIVE_CODE_TARGET       | Strong  | VisitCodeTarget        | Relative offset patched       |
| COMPRESSED_EMBEDDED_OBJECT | Strong  | VisitEmbeddedPointer   | Compressed ptr updated        |
| FULL_EMBEDDED_OBJECT       | Strong  | VisitEmbeddedPointer   | Full ptr updated              |
| EXTERNAL_REFERENCE         | No      | VisitExternalReference | C++ addr, not heap-managed    |
| INTERNAL_REFERENCE         | No      | VisitInternalReference | Patched if IS object moves    |
| OFF_HEAP_TARGET            | No      | VisitOffHeapTarget     | Builtin blob, never moves     |
| NEAR_BUILTIN_ENTRY         | No      | VisitOffHeapTarget     | Builtin blob, never moves     |
| DEOPT_*                    | No      | (skipped by GC)        | Debug annotations only        |
| CONST_POOL / VENEER_POOL   | No      | (skipped by GC)        | Pool position markers only    |

* Pipeline
** Turbofan

*** CompileTurbofan
- set code instruction address:
#+begin_src cpp
MaybeHandle<Code> CompileTurbofan(Isolate* isolate, Handle<JSFunction> function,
                                  Handle<SharedFunctionInfo> shared,
                                  ConcurrencyMode mode,
                                  BytecodeOffset osr_offset,
                                  CompileResultBehavior result_behavior);
// -> NotConcurrent test
bool CompileTurbofan_Concurrent(Isolate* isolate,
                                std::unique_ptr<TurbofanCompilationJob> job);

#+end_src

*** AssembleCode

#+begin_src cpp
// ->
CompilationJob::Status OptimizedCompilationJob::ExecuteJob(
    RuntimeCallStats* stats, LocalIsolate* local_isolate);
// ->
PipelineCompilationJob::Status PipelineCompilationJob::ExecuteJobImpl(
    RuntimeCallStats* stats, LocalIsolate* local_isolate);

void CodeGenerator::AssembleCode();

void Emit(Instr instruction);

// masm_->buffer_start_
// display/32xw masm_->buffer_start_ + 0x...
#+end_src

*** FinalizeCode
#+begin_src cpp
// +
PipelineCompilationJob::Status PipelineCompilationJob::FinalizeJobImpl(
    Isolate* isolate);
// ->
MaybeHandle<Code> Factory::CodeBuilder::BuildInternal(
    bool retry_allocation_or_fail);
// ->
Handle<Code> Factory::CodeBuilder::NewCode(const NewCodeOptions& options);
#+end_src

* Exception
** Unwinding

#+begin_src cpp
Object Isolate::UnwindAndFindHandler();

int OptimizedFrame::LookupExceptionHandlerInTable();
#+end_src

* Calling Convention
** Call
#+begin_src cpp
Builtins_InterpreterEntryTrampoline
Builtins_JSEntryTrampoline
Builtins_JSEntry
v8::internal::(anonymous namespace)::Invoke
Builtins_ResumeGeneratorHandler
Builtins_InvokeIntrinsicHandler
Builtins_CallRuntimeHandler
Builtins_CallProperty0Handler
Builtins_CallProperty1Handler
Builtins_CallUndefinedReceiverHandler
Builtins_InterpreterPushUndefinedAndArgsThenCall
Builtins_CallUndefinedReceiver2Handler
Builtins_Call_ReceiverIsAny
Builtins_CallFunction_ReceiverIsAny
Builtins_CompileLazy

CallJSAndDispatch(function, context, args, receiver_mode);

V8_WARN_UNUSED_RESULT MaybeHandle<Object> Invoke(Isolate* isolate,
                                                 const InvokeParams& params);
// b v8::internal::(anonymous namespace)::Invoke
// b v8::internal::GeneratedCode<__uintcap_t, __uintcap_t, __uintcap_t, __uintcap_t, __uintcap_t, __intcap_t, __uintcap_t**>::Call

#+end_src

** Builtins
#+begin_src cpp
IGNITION_HANDLER(CallUndefinedReceiver, InterpreterJSCallAssembler) {
  JSCall(ConvertReceiverMode::kNullOrUndefined);
}
#+end_src


* Assembler

** jit codegen

#+begin_src backtrace
#0  v8::internal::Assembler::adr (this=0x530a4d70 [rwRW,0x530a4c00-0x530a5800], rd=..., imm21=0)
    at ../../src/codegen/arm64/assembler-arm64.cc:890
#1  0x0000000004f499b0 in v8::internal::MacroAssembler::ComputeCodeStartAddress (
    this=0x530a4d70 [rwRW,0x530a4c00-0x530a5800], rd=...)
    at ../../src/codegen/arm64/macro-assembler-arm64.cc:4592
#2  0x0000000005a93e3c in v8::internal::compiler::CodeGenerator::AssembleCodeStartRegisterCheck (
    this=0x530a4c00 [rwRW,0x530a4c00-0x530a5800])
    at ../../src/compiler/backend/arm64/code-generator-arm64.cc:987
#3  0x00000000052e61b0 in v8::internal::compiler::CodeGenerator::AssembleCode (
    this=0x530a4c00 [rwRW,0x530a4c00-0x530a5800])
    at ../../src/compiler/backend/code-generator.cc:221
#4  0x00000000059560f0 in v8::internal::compiler::AssembleCodePhase::Run (
    this=0xfffffff7aa18 [rwRW,0xfffffff7aa18-0xfffffff7aa19],
    data=0x530db2d0 [rwRW,0x530db000-0x530db700],
    temp_zone=0x48e8faa0 [rwRW,0x48e8faa0-0x48e8fb10]) at ../../src/compiler/pipeline.cc:2649
#5  0x0000000005945a74 in v8::internal::compiler::PipelineImpl::Run<v8::internal::compiler::AssembleCodePhase> (this=0x530db6b0 [rwRW,0x530db000-0x530db700])
    at ../../src/compiler/pipeline.cc:1361
#6  0x00000000059386f0 in v8::internal::compiler::PipelineImpl::AssembleCode (
    this=0x530db6b0 [rwRW,0x530db000-0x530db700],
    linkage=0x530a2d10 [rwRW,0x530a1000-0x530a3000]) at ../../src/compiler/pipeline.cc:4037
#7  0x0000000005937c7c in v8::internal::compiler::PipelineCompilationJob::ExecuteJobImpl (
    this=0x530db000 [rwRW,0x530db000-0x530db700], stats=0x0,
    local_isolate=0x48e46480 [rwRW,0x48e46480-0x48e46600]) at ../../src/compiler/pipeline.cc:1290
#8  0x0000000003bc42b8 in v8::internal::OptimizedCompilationJob::ExecuteJob (
    this=0x530db000 [rwRW,0x530db000-0x530db700], stats=0x0,
    local_isolate=0x48e46480 [rwRW,0x48e46480-0x48e46600]) at ../../src/codegen/compiler.cc:488
#9  0x0000000003bdf4e0 in v8::internal::(anonymous namespace)::CompileTurbofan_NotConcurrent (
    isolate=0x48e94000 [rwRW,0x48e94000-0x48eb4000], job=0x530db000 [rwRW,0x530db000-0x530db700])
    at ../../src/codegen/compiler.cc:1045
#10 0x0000000003bde9dc in v8::internal::(anonymous namespace)::CompileTurbofan (
    isolate=0x48e94000 [rwRW,0x48e94000-0x48eb4000], function=..., shared=...,
    mode=v8::internal::ConcurrencyMode::kSynchronous, osr_offset=...,
    result_behavior=v8::internal::(anonymous namespace)::CompileResultBehavior::kDefault)
    at ../../src/codegen/compiler.cc:1178
#11 0x0000000003bce8ac in v8::internal::(anonymous namespace)::GetOrCompileOptimized (
    isolate=0x48e94000 [rwRW,0x48e94000-0x48eb4000], function=...,
    mode=v8::internal::ConcurrencyMode::kSynchronous,
    code_kind=v8::internal::CodeKind::TURBOFAN, osr_offset=...,
    result_behavior=v8::internal::(anonymous namespace)::CompileResultBehavior::kDefault)
    at ../../src/codegen/compiler.cc:1323
#12 0x0000000003bd0014 in v8::internal::Compiler::CompileOptimized (
    isolate=0x48e94000 [rwRW,0x48e94000-0x48eb4000], function=...,
    mode=v8::internal::ConcurrencyMode::kSynchronous, code_kind=v8::internal::CodeKind::TURBOFAN)
    at ../../src/codegen/compiler.cc:2757
#13 0x0000000004d382f0 in v8::internal::__RT_impl_Runtime_CompileOptimized (args=...,
    isolate=0x48e94000 [rwRW,0x48e94000-0x48eb4000]) at ../../src/runtime/runtime-compiler.cc:140
#14 0x0000000004d37eac in v8::internal::Runtime_CompileOptimized (args_length=1,
    args_object=0xfffffff7c270 [rwRW,0xffffbff80000-0xfffffff80000],
    isolate=0x48e94000 [rwRW,0x48e94000-0x48eb4000]) at ../../src/runtime/runtime-compiler.cc:98
#15 0x00000000064e3ad8 in Builtins_CEntry_Return1_ArgvOnStack_NoBuiltinExit ()
#+end_src


** Compile OSR
#+begin_src backtrace
#0  v8::internal::compiler::CodeGenerator::AssembleCode (this=0x536a5000 [rwRW,0x536a5000-0x536a5c00]) at ../../src/compiler/backend/code-generator.cc:330
#1  0x0000000005955e30 in v8::internal::compiler::AssembleCodePhase::Run (this=0xfffffff7a7d8 [rwRW,0xfffffff7a7d8-0xfffffff7a7d9],
    data=0x530c42d0 [rwRW,0x530c4000-0x530c4700], temp_zone=0x48e874f0 [rwRW,0x48e874f0-0x48e87560]) at ../../src/compiler/pipeline.cc:2649
#2  0x00000000059457b4 in v8::internal::compiler::PipelineImpl::Run<v8::internal::compiler::AssembleCodePhase> (this=0x530c46b0 [rwRW,0x530c4000-0x530c4700])
    at ../../src/compiler/pipeline.cc:1361
#3  0x0000000005938430 in v8::internal::compiler::PipelineImpl::AssembleCode (this=0x530c46b0 [rwRW,0x530c4000-0x530c4700], linkage=0x48f9ed00 [rwRW,0x48f9d000-0x48f9f000])
    at ../../src/compiler/pipeline.cc:4037
#4  0x00000000059379bc in v8::internal::compiler::PipelineCompilationJob::ExecuteJobImpl (this=0x530c4000 [rwRW,0x530c4000-0x530c4700], stats=0x0,
    local_isolate=0x48e46480 [rwRW,0x48e46480-0x48e46600]) at ../../src/compiler/pipeline.cc:1290
#5  0x0000000003bc3cf8 in v8::internal::OptimizedCompilationJob::ExecuteJob (this=0x530c4000 [rwRW,0x530c4000-0x530c4700], stats=0x0,
    local_isolate=0x48e46480 [rwRW,0x48e46480-0x48e46600]) at ../../src/codegen/compiler.cc:488
#6  0x0000000003bdef20 in v8::internal::(anonymous namespace)::CompileTurbofan_NotConcurrent (isolate=0x48e8c000 [rwRW,0x48e8c000-0x48eac000],
    job=0x530c4000 [rwRW,0x530c4000-0x530c4700]) at ../../src/codegen/compiler.cc:1045
#7  0x0000000003bde41c in v8::internal::(anonymous namespace)::CompileTurbofan (isolate=0x48e8c000 [rwRW,0x48e8c000-0x48eac000], function=..., shared=...,
    mode=v8::internal::ConcurrencyMode::kSynchronous, osr_offset=..., result_behavior=v8::internal::(anonymous namespace)::CompileResultBehavior::kDefault)
    at ../../src/codegen/compiler.cc:1178
#8  0x0000000003bce2ec in v8::internal::(anonymous namespace)::GetOrCompileOptimized (isolate=0x48e8c000 [rwRW,0x48e8c000-0x48eac000], function=...,
    mode=v8::internal::ConcurrencyMode::kSynchronous, code_kind=v8::internal::CodeKind::TURBOFAN, osr_offset=...,
    result_behavior=v8::internal::(anonymous namespace)::CompileResultBehavior::kDefault) at ../../src/codegen/compiler.cc:1323
#9  0x0000000003bd5448 in v8::internal::Compiler::CompileOptimizedOSR (isolate=0x48e8c000 [rwRW,0x48e8c000-0x48eac000], function=..., osr_offset=...,
    mode=v8::internal::ConcurrencyMode::kSynchronous, code_kind=v8::internal::CodeKind::TURBOFAN) at ../../src/codegen/compiler.cc:3899
#10 0x0000000004d3cedc in v8::internal::(anonymous namespace)::CompileOptimizedOSR (isolate=0x48e8c000 [rwRW,0x48e8c000-0x48eac000], function=..., osr_offset=...)
    at ../../src/runtime/runtime-compiler.cc:454
#11 0x0000000004d3a338 in v8::internal::__RT_impl_Runtime_CompileOptimizedOSR (args=..., isolate=0x48e8c000 [rwRW,0x48e8c000-0x48eac000])
    at ../../src/runtime/runtime-compiler.cc:496
#12 0x0000000004d3a100 in v8::internal::Runtime_CompileOptimizedOSR (args_length=0, args_object=0xfffffff7c320 [rwRW,0xffffbff80000-0xfffffff80000],
    isolate=0x48e8c000 [rwRW,0x48e8c000-0x48eac000]) at ../../src/runtime/runtime-compiler.cc:487
#13 0x00000000064e38d8 in Builtins_CEntry_Return1_ArgvOnStack_NoBuiltinExit ()
#+end_src


* Instruction


** InstructionStartOf stacktrace

*** read-only-heap

#+begin_src backtrace
#0  v8::internal::EmbeddedData::InstructionStartOf (this=0xfffffff7db20 [rwRW,0xfffffff7db20-0xfffffff7db60], builtin=v8::internal::Builtin::kAddHandler)
    at ../../src/snapshot/embedded/embedded-data-inl.h:14
#1  0x0000000001cd9d08 in v8::internal::Deserializer<v8::internal::Isolate>::PostProcessNewObject (this=<optimized out>, map=..., obj=..., space=<optimized out>)
    at ../../src/snapshot/deserializer.cc:498
#2  0x0000000001cd7654 in v8::internal::Deserializer<v8::internal::Isolate>::ReadObject (this=<optimized out>, space=v8::internal::SnapshotSpace::kReadOnlyHeap)
    at ../../src/snapshot/deserializer.cc:681
#3  0x0000000001ce5b54 in v8::internal::Deserializer<v8::internal::Isolate>::ReadNewObject<v8::internal::SlotAccessorForRootSlots> (
    this=0xfffffff7e010 [rwRW,0xfffffff7e010-0xfffffff7e380], data=152 '\230', slot_accessor=...) at ../../src/snapshot/deserializer.cc:878
#4  0x0000000001cd9050 in v8::internal::Deserializer<v8::internal::Isolate>::ReadSingleBytecodeData<v8::internal::SlotAccessorForRootSlots> (
    this=0xfffffff7db20 [rwRW,0xfffffff7db20-0xfffffff7db60], data=138 '\212', slot_accessor=...) at ../../src/snapshot/deserializer.cc:798
#5  0x0000000001cd8b4c in v8::internal::Deserializer<v8::internal::Isolate>::ReadData (this=0xfffffff7e010 [rwRW,0xfffffff7e010-0xfffffff7e380], start=..., end=...)
    at ../../src/snapshot/deserializer.cc:787
#6  0x0000000001cf0c38 in v8::internal::RootVisitor::VisitRootPointer (this=0xfffffff7e010 [rwRW,0xfffffff7e010-0xfffffff7e380],
    root=v8::internal::Root::kReadOnlyObjectCache, description=0x0, p=...) at ../../src/objects/visitors.h:78
#7  v8::internal::ReadOnlyDeserializer::DeserializeIntoIsolate (this=0xfffffff7e010 [rwRW,0xfffffff7e010-0xfffffff7e380]) at ../../src/snapshot/read-only-deserializer.cc:103
#8  0x0000000001476ef4 in v8::internal::ReadOnlyHeap::DeserializeIntoIsolate (this=0x4477e000 [rwRW,0x4477e000-0x44781000], isolate=0x446a0000 [rwRW,0x446a0000-0x446c0000],
    read_only_snapshot_data=0xfffffff7f1d0 [rwRW,0xfffffff7f1d0-0xfffffff7f200], can_rehash=false) at ../../src/heap/read-only-heap.cc:122
#9  0x0000000001476888 in v8::internal::ReadOnlyHeap::SetUp (isolate=0x446a0000 [rwRW,0x446a0000-0x446c0000],
    read_only_snapshot_data=0xfffffff7f1d0 [rwRW,0xfffffff7f1d0-0xfffffff7f200], can_rehash=false) at ../../src/heap/read-only-heap.cc:80
#10 0x000000000126d550 in v8::internal::Isolate::Init (this=0x446a0000 [rwRW,0x446a0000-0x446c0000], startup_snapshot_data=<optimized out>,
    read_only_snapshot_data=0xfffffff7f1d0 [rwRW,0xfffffff7f1d0-0xfffffff7f200], shared_heap_snapshot_data=<optimized out>, can_rehash=<optimized out>)
    at ../../src/execution/isolate.cc:4486
#11 0x000000000126e794 in v8::internal::Isolate::InitWithSnapshot (this=0x446a0000 [rwRW,0x446a0000-0x446c0000],
    startup_snapshot_data=0xfffffff7f250 [rwRW,0xfffffff7f250-0xfffffff7f280], read_only_snapshot_data=0xfffffff7f1d0 [rwRW,0xfffffff7f1d0-0xfffffff7f200],
    shared_heap_snapshot_data=0xfffffff7f1a0 [rwRW,0xfffffff7f1a0-0xfffffff7f1d0], can_rehash=false) at ../../src/execution/isolate.cc:4162
#12 0x0000000001d0c53c in v8::internal::Snapshot::Initialize (isolate=0x446a0000 [rwRW,0x446a0000-0x446c0000]) at ../../src/snapshot/snapshot.cc:182
#13 0x0000000000ec000c in v8::Isolate::Initialize (v8_isolate=0x446a0000 [rwRW,0x446a0000-0x446c0000], params=...) at ../../src/api/api.cc:9523
#14 0x0000000000ec04a4 in v8::Isolate::New (params=...) at ../../src/api/api.cc:9559
#15 0x0000000000e4572c in v8::Shell::Main (argc=<optimized out>, argv=<optimized out>) at ../../src/d8/d8.cc:5904
#16 0x0000000043aabadc in __libc_start1 (argc=12667968, argv=0xffffbff7f500 [rwRW,0xffffbff7f500-0xffffbff7f550], env=0xffffbff7f550 [rwRW,0xffffbff7f550-0xffffbff7f6e0],
    cleanup=<optimized out>, mainX=0xe463f1 <main(int, char**)> [rxRE,0x3d0000-0x368c000] (sentry), data_cap=<optimized out>, code_cap=<optimized out>)
    at /home/yayu/cheri/cheribsd/lib/libc/csu/libc_start1.c:248
#17 0x0000000000e135d0 in _start (auxv=<optimized out>, cleanup=0x68e4f0 [rR,0x68e400-0x8ad400], obj=<optimized out>)
    at /home/yayu/cheri/cheribsd/lib/csu/aarch64c/crt1_c.c:145
#+end_src


*** isolate table
#+begin_src backtrace
#0  v8::internal::EmbeddedData::InstructionStartOf (this=0xfffffff7e460 [rwRW,0xfffffff7e460-0xfffffff7e4a0], builtin=v8::internal::Builtin::kAddHandler)
    at ../../src/snapshot/embedded/embedded-data-inl.h:14
#1  0x00000000010a36e8 in v8::internal::Builtins::InitializeIsolateDataTables (isolate=0x446a0000 [rwRW,0x446a0000-0x446c0000]) at ../../src/builtins/builtins.cc:353
#2  0x000000000126dab4 in v8::internal::Isolate::Init (this=0x446a0000 [rwRW,0x446a0000-0x446c0000],
    startup_snapshot_data=0xfffffff7f250 [rwRW,0xfffffff7f250-0xfffffff7f280], read_only_snapshot_data=<optimized out>,
    shared_heap_snapshot_data=0xfffffff7f1a0 [rwRW,0xfffffff7f1a0-0xfffffff7f1d0], can_rehash=<optimized out>) at ../../src/execution/isolate.cc:4645
#3  0x000000000126e794 in v8::internal::Isolate::InitWithSnapshot (this=0x446a0000 [rwRW,0x446a0000-0x446c0000],
    startup_snapshot_data=0xfffffff7f250 [rwRW,0xfffffff7f250-0xfffffff7f280], read_only_snapshot_data=0xfffffff7f1d0 [rwRW,0xfffffff7f1d0-0xfffffff7f200],
    shared_heap_snapshot_data=0xfffffff7f1a0 [rwRW,0xfffffff7f1a0-0xfffffff7f1d0], can_rehash=false) at ../../src/execution/isolate.cc:4162
#4  0x0000000001d0c53c in v8::internal::Snapshot::Initialize (isolate=0x446a0000 [rwRW,0x446a0000-0x446c0000]) at ../../src/snapshot/snapshot.cc:182
#5  0x0000000000ec000c in v8::Isolate::Initialize (v8_isolate=0x446a0000 [rwRW,0x446a0000-0x446c0000], params=...) at ../../src/api/api.cc:9523
#6  0x0000000000ec04a4 in v8::Isolate::New (params=...) at ../../src/api/api.cc:9559
#7  0x0000000000e4572c in v8::Shell::Main (argc=<optimized out>, argv=<optimized out>) at ../../src/d8/d8.cc:5904
#8  0x0000000043aabadc in __libc_start1 (argc=4360, argv=0xffffbff7f500 [rwRW,0xffffbff7f500-0xffffbff7f550], env=0xffffbff7f550 [rwRW,0xffffbff7f550-0xffffbff7f6e0],
    cleanup=<optimized out>, mainX=0xe463f1 <main(int, char**)> [rxRE,0x3d0000-0x368c000] (sentry), data_cap=<optimized out>, code_cap=<optimized out>)
    at /home/yayu/cheri/cheribsd/lib/libc/csu/libc_start1.c:248
#9  0x0000000000e135d0 in _start (auxv=<optimized out>, cleanup=0x45636e40 [rwRW,0x45636e40-0x45636ee0], obj=<optimized out>)
    at /home/yayu/cheri/cheribsd/lib/csu/aarch64c/crt1_c.c:145
#+end_src


* Compile V8
** args
#+begin_src conf
# Build arguments go here.
# See "gn args <out_dir> --list" for available build arguments.
is_component_build = true
is_debug = true
symbol_level = 2
target_cpu = "x64"
use_goma = false
v8_enable_backtrace = true
v8_enable_fast_mksnapshot = true
v8_enable_slow_dchecks = true
v8_optimized_debug = false
v8_expose_symbols = true
v8_symbol_level = 2
cppgc_enable_object_names = true
v8_enable_disassembler = true
v8_enable_gdbjit = true
#+end_src

** command
#+begin_src shell
gn args out/Debug
#----- add args -----#
gn gen out/Debug
#+end_src

** clangd
#+begin_src sh
cmake ~/cheri/morello-llvm-project/llvm/ -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra"
cmake --build ~/cheri/build/morello-llvm-project-build/ --target clangd
#+end_src


* Debug

https://v8.dev/docs/gdb
https://v8.dev/docs/gdb-jit

** debug builtins
#+begin_src gdb
(gdb) tb v8::internal::Isolate::Init
(gdb) r
(gdb) b Builtins_InterpreterEntryTrampoline
(gdb) b Builtins_BaselineOnStackReplacement
#+end_src

** debug jit
- object fields:
#+begin_src gdb
p *(uintptr_t*)((void*)code.ptr_ + Code::kInstructionStartOffset - kHeapObjectTag)
#+end_src

- AssembleCode
#+begin_src gdb
x/32wx masm_->buffer_start_
display/32xw masm_->buffer_start_ + 0x... (p/x masm_.pc_ - masm_.buffer_start_)
#+end_src


** gdbinit
=v8/tools/gdbinit=

*** functions

**** job
#+begin_src gdb
Print a v8 JavaScript object
Usage: job tagged_ptr
#+end_src

**** jh
#+begin_src gdb
Print content of a v8::internal::Handle
Usage: jh internal_handle
#+end_src

**** jlh
#+begin_src gdb
print-v8-local, jl, jlh
Print content of v8::Local handle.
#+end_src

**** jco
#+begin_src gdb
Print a v8 Code object from an internal code address
Usage: jco pc
#+end_src

**** jtt
#+begin_src gdb
Print the complete transition tree of the given v8 Map.
Usage: jtt tagged_ptr
#+end_src

**** jst
#+begin_src gdb
Print the current JavaScript stack trace
Usage: jst
#+end_src

**** jss
#+begin_src gdb
Skip the jitted stack on x64 to where we entered JS last.
Usage: jss
#+end_src

**** bta
#+begin_src gdb
Print stack trace with assertion scopes
Usage: bta
#+end_src

**** heap_find
#+begin_src gdb
Find the location of a given address in V8 pages.
Usage: heap_find address
#+end_src

**** cpcp
#+begin_src gdb
Prints compressed pointer (raw value) after decompression.
Usage: cpcp compressed_pointer
#+end_src

**** cpm
#+begin_src gdb
Prints member, compressed or not.
Usage: cpm member
#+end_src


** js debug commands
- ~DebugPrint~
- ~DebugTrace~
- ~SystemBreak~
- [[https://source.chromium.org/chromium/v8/v8.git/+/05720af2b09a18be5c41bbf224a58f3f0618f6be:src/runtime/runtime.h;l=574][full commands]]


* CHERI
- fork commit: 0c4044b7336787781646e48b2f98f0c7d1b400a5
** ISA
- c16:

*** Permission
- r: CHERI_PERM_LOAD
- R: CHERI_PERM_LOAD_CAP
- w: CHERI_PERM_STORE
- W: CHERI_PERM_STORE_CAP
- x: CHERI_PERM_EXECUTE
- E: CHERI_PERM_EXECUTIVE/ARM_CAP_PERMISSION_EXECUTIVE

- default
#+begin_src js
[
    CHERI_PERM_GLOBAL,
    CHERI_PERM_EXECUTE,
    CHERI_PERM_LOAD,
    CHERI_PERM_STORE,
    CHERI_PERM_LOAD_CAP,
    CHERI_PERM_STORE_CAP,
    CHERI_PERM_STORE_LOCAL_CAP,
    ARM_CAP_PERMISSION_EXECUTIVE,
    ARM_CAP_PERMISSION_MUTABLE_LOAD,
    ARM_CAP_PERMISSION_BRANCH_SEALED_PAIR
]
#+end_src


** CHERI-CFI
*** turbofan code

**** SEAL reuse

need to recheck if the inst is capability?

#+begin_src backtrace
#0  0x0000000004f94d5c in unsigned int v8::base::ReadUnalignedValue<unsigned int>(__uintcap_t) () at ../../src/base/memory.h:38
#1  0x0000000004f9678c in v8::internal::Instruction::InstructionBits() const () at ../../src/codegen/arm64/instructions-arm64.h:91
#2  0x0000000004f96760 in v8::internal::Instruction::Mask(unsigned int) const () at ../../src/codegen/arm64/instructions-arm64.h:112
#3  0x0000000004f96614 in v8::internal::Instruction::IsLdrLiteralC() const () at ../../src/codegen/arm64/instructions-arm64.h:232
#4  0x0000000004f95610 in v8::internal::Assembler::target_address_at(__uintcap_t, __uintcap_t) () at ../../src/codegen/arm64/assembler-arm64-inl.h:604
#5  0x0000000004f95f98 in v8::internal::RelocInfo::target_address() () at ../../src/codegen/arm64/assembler-arm64-inl.h:779
#6  0x000000000760788c in v8::internal::SetupIsolateDelegate::ReplacePlaceholders(v8::internal::Isolate*) () at ../../src/builtins/setup-builtins-internal.cc:246
#7  0x0000000007659894 in v8::internal::SetupIsolateDelegate::SetupBuiltinsInternal(v8::internal::Isolate*) () at ../../src/builtins/setup-builtins-internal.cc:358
#8  0x0000000006281f84 in v8::internal::SetupIsolateDelegate::SetupBuiltins(v8::internal::Isolate*, bool) () at ../../src/init/setup-isolate-full.cc:29
#9  0x00000000051bc130 in v8::internal::Isolate::Init(v8::internal::SnapshotData*, v8::internal::SnapshotData*, v8::internal::SnapshotData*, bool) ()
    at ../../src/execution/isolate.cc:4586
#10 0x00000000051baf90 in v8::internal::Isolate::InitWithoutSnapshot() () at ../../src/execution/isolate.cc:4152
#11 0x0000000004b7e4a8 in v8::SnapshotCreator::SnapshotCreator(v8::Isolate*, __intcap_t const*, v8::StartupData const*, bool) () at ../../src/api/api.cc:600
#12 0x000000000616ddc0 in v8::internal::CreateSnapshotDataBlobInternal(v8::SnapshotCreator::FunctionCodeHandling, char const*, v8::Isolate*) ()
    at ../../src/snapshot/snapshot.cc:774
#13 0x0000000004b1ca54 in (anonymous namespace)::CreateSnapshotDataBlob(v8::Isolate*, char const*) () at ../../src/snapshot/mksnapshot.cc:161
#14 0x0000000004b1c47c in main () at ../../src/snapshot/mksnapshot.cc:287
#+end_src


#+begin_src backtrace
#0  0x0000000003c8fce4 in v8::base::ReadUnalignedValue<unsigned int> (p=0x4ae02309 [rwxRWE,0x4ae00000-0x52e40000] (invalid,sentry) ) at ../../src/base/memory.h:38
#1  0x0000000003c91714 in v8::internal::Instruction::InstructionBits (this=0x4ae02309 [rwxRWE,0x4ae00000-0x52e40000] (invalid,sentry))
    at ../../src/codegen/arm64/instructions-arm64.h:91
#2  0x0000000003c916e8 in v8::internal::Instruction::Mask (this=0x4ae02309 [rwxRWE,0x4ae00000-0x52e40000] (invalid,sentry), mask=4290772992)
    at ../../src/codegen/arm64/instructions-arm64.h:112
#3  0x0000000003c9159c in v8::internal::Instruction::IsLdrLiteralC (this=0x4ae02309 [rwxRWE,0x4ae00000-0x52e40000] (invalid,sentry))
    at ../../src/codegen/arm64/instructions-arm64.h:232
#4  0x0000000003c90598 in v8::internal::Assembler::target_address_at (pc=0x4ae02309 [rwxRWE,0x4ae00000-0x52e40000] (invalid,sentry) , constant_pool=0x0 )
    at ../../src/codegen/arm64/assembler-arm64-inl.h:604
#5  0x0000000003c90d74 in v8::internal::RelocInfo::target_object (this=0xfffffff7acd0 [rwRW,0xfffffff7acb0-0xfffffff7ad20], cage_base=...)
    at ../../src/codegen/arm64/assembler-arm64-inl.h:831
#6  0x00000000059392b0 in v8::internal::compiler::PipelineImpl::CheckNoDeprecatedMaps (this=0x530db6b0 [rwRW,0x530db000-0x530db700], code=...)
    at ../../src/compiler/pipeline.cc:4139
#7  0x00000000059389d4 in v8::internal::compiler::PipelineCompilationJob::FinalizeJobImpl (this=0x530db000 [rwRW,0x530db000-0x530db700],
    isolate=0x48e94000 [rwRW,0x48e94000-0x48eb4000]) at ../../src/compiler/pipeline.cc:1309
#8  0x0000000003bc44d4 in v8::internal::OptimizedCompilationJob::FinalizeJob (this=0x530db000 [rwRW,0x530db000-0x530db700], isolate=0x48e94000 [rwRW,0x48e94000-0x48eb4000])
    at ../../src/codegen/compiler.cc:499
#9  0x0000000003bdf5a0 in v8::internal::(anonymous namespace)::CompileTurbofan_NotConcurrent (isolate=0x48e94000 [rwRW,0x48e94000-0x48eb4000],
    job=0x530db000 [rwRW,0x530db000-0x530db700]) at ../../src/codegen/compiler.cc:1055
#10 0x0000000003bde9dc in v8::internal::(anonymous namespace)::CompileTurbofan (isolate=0x48e94000 [rwRW,0x48e94000-0x48eb4000], function=..., shared=...,
    mode=v8::internal::ConcurrencyMode::kSynchronous, osr_offset=..., result_behavior=v8::internal::(anonymous namespace)::CompileResultBehavior::kDefault)
    at ../../src/codegen/compiler.cc:1178
#11 0x0000000003bce8ac in v8::internal::(anonymous namespace)::GetOrCompileOptimized (isolate=0x48e94000 [rwRW,0x48e94000-0x48eb4000], function=...,
    mode=v8::internal::ConcurrencyMode::kSynchronous, code_kind=v8::internal::CodeKind::TURBOFAN, osr_offset=...,
    result_behavior=v8::internal::(anonymous namespace)::CompileResultBehavior::kDefault) at ../../src/codegen/compiler.cc:1323
#12 0x0000000003bd0014 in v8::internal::Compiler::CompileOptimized (isolate=0x48e94000 [rwRW,0x48e94000-0x48eb4000], function=...,
    mode=v8::internal::ConcurrencyMode::kSynchronous, code_kind=v8::internal::CodeKind::TURBOFAN) at ../../src/codegen/compiler.cc:2757
#13 0x0000000004d382f0 in v8::internal::__RT_impl_Runtime_CompileOptimized (args=..., isolate=0x48e94000 [rwRW,0x48e94000-0x48eb4000])
    at ../../src/runtime/runtime-compiler.cc:140
#14 0x0000000004d37eac in v8::internal::Runtime_CompileOptimized (args_length=1, args_object=0xfffffff7c270 [rwRW,0xffffbff80000-0xfffffff80000],
    isolate=0x48e94000 [rwRW,0x48e94000-0x48eb4000]) at ../../src/runtime/runtime-compiler.cc:98
#15 0x00000000064e3ad8 in Builtins_CEntry_Return1_ArgvOnStack_NoBuiltinExit ()
#+end_src
*** TODO code flag

- change stack?

#+begin_src cpp
class V8_EXPORT_PRIVATE Frame : public ZoneObject {
#+end_src

** Test
#+begin_src sh
# v8/out/cfi
../../cheri/distribute_v8_tests.py -M ../../v8-test-machines -d . --v8-root ../../ --remote-v8-root ~/v8/ -v --suites unittests -b out/cfi
#+end_src

*** AccessorTest.CachedAccessorTurboFan
- off heap target misalignment?
- off heap target = pc_ + constant_pool_

#+begin_src backtrace
#0  0x0000000008bb51c8 in v8::base::ReadUnalignedValue<unsigned int> (p=0x534020d4 [xR,0x53400000-0x63440000] ) at ../../src/base/memory.h:38
#1  0x0000000008bb68dc in v8::internal::Instruction::InstructionBits (this=0x534020d4 [xR,0x53400000-0x63440000]) at ../../src/codegen/arm64/instructions-arm64.h:91
#2  0x0000000008bb68b0 in v8::internal::Instruction::Mask (this=0x534020d4 [xR,0x53400000-0x63440000], mask=4290772992) at ../../src/codegen/arm64/instructions-arm64.h:112
#3  0x0000000008bb6764 in v8::internal::Instruction::IsLdrLiteralC (this=0x534020d4 [xR,0x53400000-0x63440000]) at ../../src/codegen/arm64/instructions-arm64.h:232
#4  0x0000000008bb5924 in v8::internal::Assembler::target_address_at (pc=0x534020d4 [xR,0x53400000-0x63440000] , constant_pool=0x0 )
    at ../../src/codegen/arm64/assembler-arm64-inl.h:604
#5  0x0000000008bb642c in v8::internal::RelocInfo::target_off_heap_target (this=0xfffffff7b360 [rwRW,0xfffffff7b340-0xfffffff7b3b0])
    at ../../src/codegen/arm64/assembler-arm64-inl.h:888
#6  0x0000000008bb4f10 in v8::internal::RelocInfo::Verify (this=0xfffffff7b360 [rwRW,0xfffffff7b340-0xfffffff7b3b0], isolate=0x5207a000 [rwRW,0x5207a000-0x5209a000])
    at ../../src/codegen/reloc-info.cc:433
#7  0x0000000008d0d5a4 in v8::internal::InstructionStream::InstructionStreamVerify (this=0xfffffff7e430 [rwRW,0xfffffff7e430-0xfffffff7e440],
    isolate=0x5207a000 [rwRW,0x5207a000-0x5209a000]) at ../../src/diagnostics/objects-debug.cc:1207
#8  0x0000000008d0991c in v8::internal::HeapObject::HeapObjectVerify (this=0xfffffff7e790 [rwRW,0xfffffff7e790-0xfffffff7e7a0],
    isolate=0x5207a000 [rwRW,0x5207a000-0x5209a000]) at ../../src/diagnostics/objects-debug.cc:250
#9  0x0000000008d07a20 in v8::internal::Object::ObjectVerify (this=0xfffffff7e8f0 [rwRW,0xfffffff7e8f0-0xfffffff7e900], isolate=0x5207a000 [rwRW,0x5207a000-0x5209a000])
    at ../../src/diagnostics/objects-debug.cc:137
#10 0x0000000008fb4cbc in v8::internal::HeapVerification::VerifyObject (this=0xfffffff7f160 [rwRW,0xfffffff7f160-0xfffffff7f1c0], object=...)
    at ../../src/heap/heap-verifier.cc:381
#11 0x00000000091782b0 in v8::internal::PagedSpaceBase::Verify (this=0x51207f00 [rwRW,0x51207f00-0x51208200], isolate=0x5207a000 [rwRW,0x5207a000-0x5209a000],
    visitor=0xfffffff7f160 [rwRW,0xfffffff7f160-0xfffffff7f1c0]) at ../../src/heap/paged-spaces.cc:699
#12 0x0000000008fb4858 in v8::internal::HeapVerification::VerifySpace (this=0xfffffff7f160 [rwRW,0xfffffff7f160-0xfffffff7f1c0],
    space=0x51207f00 [rwRW,0x51207f00-0x51208200]) at ../../src/heap/heap-verifier.cc:353
#13 0x0000000008fb46f8 in v8::internal::HeapVerification::Verify (this=0xfffffff7f160 [rwRW,0xfffffff7f160-0xfffffff7f1c0]) at ../../src/heap/heap-verifier.cc:336
#14 0x0000000008fb5c0c in v8::internal::HeapVerifier::VerifyHeap (heap=0x52093410 [rwRW,0x5207a000-0x5209a000]) at ../../src/heap/heap-verifier.cc:699
#15 0x0000000008ffda3c in v8::internal::HeapVerifier::VerifyHeapIfEnabled (heap=0x52093410 [rwRW,0x5207a000-0x5209a000]) at ../../src/heap/heap-verifier.h:80
#16 0x0000000008fe9d58 in v8::internal::Heap::TearDownWithSharedHeap (this=0x52093410 [rwRW,0x5207a000-0x5209a000]) at ../../src/heap/heap.cc:5801
#17 0x0000000008db146c in v8::internal::Isolate::Deinit (this=0x5207a000 [rwRW,0x5207a000-0x5209a000]) at ../../src/execution/isolate.cc:3691
#18 0x0000000008db0f34 in v8::internal::Isolate::Delete (isolate=0x5207a000 [rwRW,0x5207a000-0x5209a000]) at ../../src/execution/isolate.cc:3432
#19 0x00000000087f7294 in v8::Isolate::Dispose (this=0x5207a000 [rwRW,0x5207a000-0x5209a000]) at ../../src/api/api.cc:9569
#20 0x0000000007c11710 in v8::IsolateWrapper::~IsolateWrapper (this=0x51232630 [rwRW,0x51232600-0x512326c0]) at ../../test/unittests/test-utils.cc:61
#21 0x0000000007391adc in v8::WithIsolateMixin<v8::WithDefaultPlatformMixin<testing::Test>, (v8::CountersMode)0>::~WithIsolateMixin (
    this=0x51232600 [rwRW,0x51232600-0x512326c0]) at ../../test/unittests/test-utils.h:120
#22 0x0000000007391984 in v8::WithIsolateScopeMixin<v8::WithIsolateMixin<v8::WithDefaultPlatformMixin<testing::Test>, (v8::CountersMode)0> >::~WithIsolateScopeMixin (
    this=0x51232600 [rwRW,0x51232600-0x512326c0]) at ../../test/unittests/test-utils.h:143
#23 0x00000000073918ac in v8::WithContextMixin<v8::WithIsolateScopeMixin<v8::WithIsolateMixin<v8::WithDefaultPlatformMixin<testing::Test>, (v8::CountersMode)0> > >::~WithContextMixin (this=0x51232600 [rwRW,0x51232600-0x512326c0]) at ../../test/unittests/test-utils.h:268
#24 0x00000000073d2f14 in AccessorTest_CachedAccessorTurboFan_Test::~AccessorTest_CachedAccessorTurboFan_Test (this=0x51232600 [rwRW,0x51232600-0x512326c0])
    at ../../test/unittests/api/accessor-unittest.cc:80
#25 0x00000000073d2f3c in AccessorTest_CachedAccessorTurboFan_Test::~AccessorTest_CachedAccessorTurboFan_Test (this=0x51232600 [rwRW,0x51232600-0x512326c0])
    at ../../test/unittests/api/accessor-unittest.cc:80
#26 0x000000000d02734c in testing::Test::DeleteSelf_ (this=0x51232600 [rwRW,0x51232600-0x512326c0]) at ../../third_party/googletest/src/googletest/include/gtest/gtest.h:318
#27 0x000000000d026ef4 in testing::internal::HandleExceptionsInMethodIfSupported<testing::Test, void> (object=0x51232600 [rwRW,0x51232600-0x512326c0],
    method=(void (testing::Test::*)(testing::Test * const)) 0xd027315 <testing::Test::DeleteSelf_()>, location=warning: (Error: pc 0x421d842 in address map, but not in symtab.)

    0x421d842 [rR,0x421d842-0x421d860] "the test fixture's destructor") at ../../third_party/googletest/src/googletest/src/gtest.cc:2653
#28 0x000000000d014934 in testing::TestInfo::Run (this=0x50842980 [rwRW,0x50842980-0x50842b00]) at ../../third_party/googletest/src/googletest/src/gtest.cc:2855
#29 0x000000000d015014 in testing::TestSuite::Run (this=0x50837a80 [rwRW,0x50837a80-0x50837c40]) at ../../third_party/googletest/src/googletest/src/gtest.cc:3008
#30 0x000000000d01fc4c in testing::internal::UnitTestImpl::RunAllTests (this=0x5082f000 [rwRW,0x5082f000-0x5082f400])
    at ../../third_party/googletest/src/googletest/src/gtest.cc:5866
#31 0x000000000d028ba0 in testing::internal::HandleExceptionsInMethodIfSupported<testing::internal::UnitTestImpl, bool> (object=0x5082f000 [rwRW,0x5082f000-0x5082f400],
    method=(bool (testing::internal::UnitTestImpl::*)(testing::internal::UnitTestImpl * const)) 0xd01f791 <testing::internal::UnitTestImpl::RunAllTests()>, location=warning: (Error: pc 0x3f22892 in address map, but not in symtab.)

    0x3f22892 [rR,0x3f22892-0x3f228c8] "auxiliary test code (environments or event listeners)") at ../../third_party/googletest/src/googletest/src/gtest.cc:2653
#32 0x000000000d01f768 in testing::UnitTest::Run (this=0xefe7500 <testing::UnitTest::GetInstance()::instance> [rwRW,0xefe7500-0xefe7550])
    at ../../third_party/googletest/src/googletest/src/gtest.cc:5440
#33 0x0000000007baa8b4 in RUN_ALL_TESTS () at ../../third_party/googletest/src/googletest/include/gtest/gtest.h:2284
#34 0x0000000007baa788 in main (argc=1, argv=0xffffbff7f420 [rwRW,0xffffbff7f420-0xffffbff7f4d0]) at ../../test/unittests/run-all-unittests.cc:55
#+end_src

*** DeoptimizationDisableConcurrentRecompilationTest.DeoptimizeBinaryOperationADD
- should return to 0x0000000053402c00 (+1 for sentry)?
- why $c30 changed
- ~../../test/unittests/deoptimizer/deoptimization-unittest.cc:389~
- =v8/out/cfi/gen/torque-generated/src/builtins/number-tq-csa.cc=

#+begin_src backtrace
0x000000000b5bc5b8 in Builtins_Add ()
(gdb)
0x000000000b5bc5bc in Builtins_Add ()
(gdb)
0x000000000b5bc5f8 in Builtins_Add ()
(gdb)
0x0000000053402e08 in ?? ()

-----
# display
display/xg 0xfffffff7d210
display/xg 0xfffffff7d220

1: x/xg 0xfffffff7d210  0xfffffff7d210: 0x0000fffffff7d270
2: x/xg 0xfffffff7d220  0xfffffff7d220: 0x0000000053402c01

-----
# changed on this call
0xb5bb404 <Builtins_Add+1092>   bl      0xb5339c0 <Builtins_NonPrimitiveToPrimitive_Default>

-----
0xb5342e8 <Builtins_NonPrimitiveToPrimitive_Default+2344>       bl      0xb536900 <Builtins_OrdinaryToPrimitive_Number_Inline>
0xb536b64 <Builtins_OrdinaryToPrimitive_Number_Inline+612>      bl      0xaea2a00 <Builtins_Call_ReceiverIsAny>

-----
void Builtins::Generate_Call(MacroAssembler* masm, ConvertReceiverMode mode) {

-----
0xaef9fec <Builtins_InterpreterEntryTrampoline+428>     blr     c2

-----
(gdb) bt
#0  v8::internal::PointerAuthentication::ReplacePC (pc_address=0xfffffff7d220 [rwRW,0xffffbff80000-0xfffffff80000], new_pc=0x53402e09 [rxR,0x53400000-0x63440000] )
    at ../../src/execution/pointer-authentication-dummy.h:31
#1  0x0000000008c981a4 in v8::internal::(anonymous namespace)::ActivationsFinder::VisitThread (this=0xfffffff7bd60 [rwRW,0xfffffff7bd60-0xfffffff7bd90],
    isolate=0x5248a000 [rwRW,0x5248a000-0x524aa000], top=0x5248a210 [rwRW,0x5248a000-0x524aa000]) at ../../src/deoptimizer/deoptimizer.cc:336
#2  0x0000000008c97ca0 in v8::internal::Deoptimizer::DeoptimizeMarkedCode (isolate=0x5248a000 [rwRW,0x5248a000-0x524aa000]) at ../../src/deoptimizer/deoptimizer.cc:394
#3  0x0000000008c98ac4 in v8::internal::Deoptimizer::DeoptimizeFunction (function=..., code=...) at ../../src/deoptimizer/deoptimizer.cc:438
#4  0x0000000009c7e0d0 in v8::internal::__RT_impl_Runtime_DeoptimizeFunction (args=..., isolate=0x5248a000 [rwRW,0x5248a000-0x524aa000])
    at ../../src/runtime/runtime-test.cc:185
#5  0x0000000009c7de24 in v8::internal::Runtime_DeoptimizeFunction (args_length=1, args_object=0xfffffff7cfb0 [rwRW,0xffffbff80000-0xfffffff80000],
    isolate=0x5248a000 [rwRW,0x5248a000-0x524aa000]) at ../../src/runtime/runtime-test.cc:176
#6  0x000000000b31d208 in Builtins_CEntry_Return1_ArgvInRegister_NoBuiltinExit ()
#7  0x0000000000000006 in ?? ()
#+end_src

-  [[file:~/cheri/v8/src/deoptimizer/deoptimizer.cc::Address new_pc = code.instruction_start() + trampoline_pc;]]

#+begin_src turbo
--- Raw source ---
f(7, new X());

--- Optimized code ---
optimization_id = 1
source_position = 0
kind = TURBOFAN
stack_slots = 8
compiler = turbofan
address = 0x23ddbddc44c1

Instructions (size = 1356)
0x534024c0     0  10000010       adr c16, #+0x0 (addr 0x534024c0)
0x534024c4     4  c2c0b211       gcseal x17, c16
0x534024c8     8  37000071       tbnz w17, #0, #+0xc (addr 0x534024d4)
0x534024cc     c  b2400211       orr x17, x16, #0x1
0x534024d0    10  c2d14210       scvalue c16, c16, x17
0x534024d4    14  37000050       tbnz w16, #0, #+0x8 (addr 0x534024dc)
0x534024d8    18  d4200000       brk #0x0
0x534024dc    1c  c2c0b051       gcseal x17, c2
0x534024e0    20  37000071       tbnz w17, #0, #+0xc (addr 0x534024ec)
0x534024e4    24  b2400051       orr x17, x2, #0x1
0x534024e8    28  c2d14042       scvalue c2, c2, x17
0x534024ec    2c  37000042       tbnz w2, #0, #+0x8 (addr 0x534024f4)
0x534024f0    30  d4200000       brk #0x0
0x534024f4    34  eb02021f       cmp x16, x2
0x534024f8    38  540002a0       b.eq #+0x54 (addr 0x5340254c)
0x534024fc    3c  d2c00821       movz x1, #0x4100000000
0x53402500    40  c2659f50       ldr c16, [c26, #19256]
0x53402504    44  c2c0b211       gcseal x17, c16
0x53402508    48  37000071       tbnz w17, #0, #+0xc (addr 0x53402514)
0x5340250c    4c  b2400211       orr x17, x16, #0x1
0x53402510    50  c2d14210       scvalue c16, c16, x17
0x53402514    54  37000050       tbnz w16, #0, #+0x8 (addr 0x5340251c)
0x53402518    58  d4200000       brk #0x0
0x5340251c    5c  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402520    60  c2c09200       gctag x0, c16
0x53402524    64  37000040       tbnz w0, #0, #+0x8 (addr 0x5340252c)
0x53402528    68  d4200000       brk #0x0
0x5340252c    6c  22c103ff       ldp c31, c0, [csp], #32
0x53402530    70  c2c0b211       gcseal x17, c16
0x53402534    74  37000071       tbnz w17, #0, #+0xc (addr 0x53402540)
0x53402538    78  b2400211       orr x17, x16, #0x1
0x5340253c    7c  c2d14210       scvalue c16, c16, x17
0x53402540    80  37000050       tbnz w16, #0, #+0x8 (addr 0x53402548)
0x53402544    84  d4200000       brk #0x0
0x53402548    88  c2c23200       blr x16
0x5340254c    8c  b8584350       ldur w16, [x26, #-124]
0x53402550    90  36e80270       tbz w16, #29, #+0x4c (addr 0x5340259c)
0x53402554    94                                             ;; off heap target
0x53402558    98  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x5340255c    9c  c2c0b220       gcseal x0, c17
0x53402560    a0  37000060       tbnz w0, #0, #+0xc (addr 0x5340256c)
0x53402564    a4  b2400220       orr x0, x17, #0x1
0x53402568    a8  c2c04231       scvalue c17, c17, x0
0x5340256c    ac  37000051       tbnz w17, #0, #+0x8 (addr 0x53402574)
0x53402570    b0  d4200000       brk #0x0
0x53402574    b4  22c103ff       ldp c31, c0, [csp], #32
0x53402578    b8  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x5340257c    bc  c2c0b220       gcseal x0, c17
0x53402580    c0  37000060       tbnz w0, #0, #+0xc (addr 0x5340258c)
0x53402584    c4  b2400220       orr x0, x17, #0x1
0x53402588    c8  c2c04231       scvalue c17, c17, x0
0x5340258c    cc  37000051       tbnz w17, #0, #+0x8 (addr 0x53402594)
0x53402590    d0  d4200000       brk #0x0
0x53402594    d4  22c103ff       ldp c31, c0, [csp], #32
0x53402598    d8  c2c21220       br x17
0x5340259c    dc  910003f0       mov x16, sp
0x534025a0    e0  f2400e1f       tst x16, #0xf
0x534025a4    e4  540001e0       b.eq #+0x3c (addr 0x534025e0)
0x534025a8    e8  52800760       movz w0, #0x3b
0x534025ac    ec                                             ;; external reference (abort_with_reason)
0x534025b0    f0  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x534025b4    f4  c2c09200       gctag x0, c16
0x534025b8    f8  37000040       tbnz w0, #0, #+0x8 (addr 0x534025c0)
0x534025bc    fc  d4200000       brk #0x0
0x534025c0   100  22c103ff       ldp c31, c0, [csp], #32
0x534025c4   104  c2c0b211       gcseal x17, c16
0x534025c8   108  37000071       tbnz w17, #0, #+0xc (addr 0x534025d4)
0x534025cc   10c  b2400211       orr x17, x16, #0x1
0x534025d0   110  c2d14210       scvalue c16, c16, x17
0x534025d4   114  37000050       tbnz w16, #0, #+0x8 (addr 0x534025dc)
0x534025d8   118  d4200000       brk #0x0
0x534025dc   11c  c2c23200       blr x16
0x534025e0   120  62bf7bfd       stp fp, lr, [csp, #-32 ]!
0x534025e4   124  c2c1d3fd       mov fp, csp
0x534025e8   128  62be03ff       stp c31, c0, [csp, #-64 ]!
0x534025ec   12c  42816fe1       stp c1, cp, [csp, #32 ]
0x534025f0   130  028083ff       sub csp, csp, #0x20 (32)
0x534025f4   134  a2540344       ldur c4, [c26, #-192]
0x534025f8   138  c2000bfb       str cp, [csp, #16]
0x534025fc   13c  eb2463ff       cmp sp, x4
0x53402600   140  54001389       b.ls #+0x270 (addr 0x53402870)
0x53402604   144                                             ;; object: 0x2c8f79046171 <String[1]: #f>
0x53402608   148  d2800000       movz x0, #0x0
0x5340260c   14c                                             ;; object: 0x0867828c0b51 <NativeContext[282]>
0x53402610   150                                             ;; off heap target
0x53402614   154  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402618   158  c2c09200       gctag x0, c16
0x5340261c   15c  37000040       tbnz w0, #0, #+0x8 (addr 0x53402624)
0x53402620   160  d4200000       brk #0x0
0x53402624   164  22c103ff       ldp c31, c0, [csp], #32
0x53402628   168  c2c0b211       gcseal x17, c16
0x5340262c   16c  37000071       tbnz w17, #0, #+0xc (addr 0x53402638)
0x53402630   170  b2400211       orr x17, x16, #0x1
0x53402634   174  c2d14210       scvalue c16, c16, x17
0x53402638   178  37000050       tbnz w16, #0, #+0x8 (addr 0x53402640)
0x5340263c   17c  d4200000       brk #0x0
0x53402640   180  c2c23200       blr x16
0x53402644   184  c20007e0       str c0, [csp, #8]
0x53402648   188                                             ;; object: 0x2c8f79045ed1 <String[1]: #X>
0x5340264c   18c  d2800080       movz x0, #0x4
0x53402650   190                                             ;; object: 0x0867828c0b51 <NativeContext[282]>
0x53402654   194                                             ;; off heap target
0x53402658   198  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x5340265c   19c  c2c09200       gctag x0, c16
0x53402660   1a0  37000040       tbnz w0, #0, #+0x8 (addr 0x53402668)
0x53402664   1a4  d4200000       brk #0x0
0x53402668   1a8  22c103ff       ldp c31, c0, [csp], #32
0x5340266c   1ac  c2c0b211       gcseal x17, c16
0x53402670   1b0  37000071       tbnz w17, #0, #+0xc (addr 0x5340267c)
0x53402674   1b4  b2400211       orr x17, x16, #0x1
0x53402678   1b8  c2d14210       scvalue c16, c16, x17
0x5340267c   1bc  37000050       tbnz w16, #0, #+0x8 (addr 0x53402684)
0x53402680   1c0  d4200000       brk #0x0
0x53402684   1c4  c2c23200       blr x16
0x53402688   1c8  910003f0       mov x16, sp
0x5340268c   1cc  f2400e1f       tst x16, #0xf
0x53402690   1d0  540001e0       b.eq #+0x3c (addr 0x534026cc)
0x53402694   1d4  52800760       movz w0, #0x3b
0x53402698   1d8                                             ;; external reference (abort_with_reason)
0x5340269c   1dc  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x534026a0   1e0  c2c09200       gctag x0, c16
0x534026a4   1e4  37000040       tbnz w0, #0, #+0x8 (addr 0x534026ac)
0x534026a8   1e8  d4200000       brk #0x0
0x534026ac   1ec  22c103ff       ldp c31, c0, [csp], #32
0x534026b0   1f0  c2c0b211       gcseal x17, c16
0x534026b4   1f4  37000071       tbnz w17, #0, #+0xc (addr 0x534026c0)
0x534026b8   1f8  b2400211       orr x17, x16, #0x1
0x534026bc   1fc  c2d14210       scvalue c16, c16, x17
0x534026c0   200  37000050       tbnz w16, #0, #+0x8 (addr 0x534026c8)
0x534026c4   204  d4200000       brk #0x0
0x534026c8   208  c2c23200       blr x16
0x534026cc   20c  028083ff       sub csp, csp, #0x20 (32)
0x534026d0   210  c20007ff       str c31, [csp, #8]
0x534026d4   214  c240db44       ldr c4, [c26, #432]
0x534026d8   218  c20003e4       str c4, [csp]
0x534026dc   21c  c2c1d001       mov c1, c0
0x534026e0   220  c2c1d003       mov c3, c0
0x534026e4   224  d2800020       movz x0, #0x1
0x534026e8   228                                             ;; object: 0x0867828c0b51 <NativeContext[282]>
0x534026ec   22c                                             ;; off heap target
0x534026f0   230  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x534026f4   234  c2c09200       gctag x0, c16
0x534026f8   238  37000040       tbnz w0, #0, #+0x8 (addr 0x53402700)
0x534026fc   23c  d4200000       brk #0x0
0x53402700   240  22c103ff       ldp c31, c0, [csp], #32
0x53402704   244  c2c0b211       gcseal x17, c16
0x53402708   248  37000071       tbnz w17, #0, #+0xc (addr 0x53402714)
0x5340270c   24c  b2400211       orr x17, x16, #0x1
0x53402710   250  c2d14210       scvalue c16, c16, x17
0x53402714   254  37000050       tbnz w16, #0, #+0x8 (addr 0x5340271c)
0x53402718   258  d4200000       brk #0x0
0x5340271c   25c  c2c23200       blr x16
0x53402720   260  910003f0       mov x16, sp
0x53402724   264  f2400e1f       tst x16, #0xf
0x53402728   268  540001e0       b.eq #+0x3c (addr 0x53402764)
0x5340272c   26c  52800760       movz w0, #0x3b
0x53402730   270                                             ;; external reference (abort_with_reason)
0x53402734   274  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402738   278  c2c09200       gctag x0, c16
0x5340273c   27c  37000040       tbnz w0, #0, #+0x8 (addr 0x53402744)
0x53402740   280  d4200000       brk #0x0
0x53402744   284  22c103ff       ldp c31, c0, [csp], #32
0x53402748   288  c2c0b211       gcseal x17, c16
0x5340274c   28c  37000071       tbnz w17, #0, #+0xc (addr 0x53402758)
0x53402750   290  b2400211       orr x17, x16, #0x1
0x53402754   294  c2d14210       scvalue c16, c16, x17
0x53402758   298  37000050       tbnz w16, #0, #+0x8 (addr 0x53402760)
0x5340275c   29c  d4200000       brk #0x0
0x53402760   2a0  c2c23200       blr x16
0x53402764   2a4  028103ff       sub csp, csp, #0x40 (64)
0x53402768   2a8  c2000fff       str c31, [csp, #24]
0x5340276c   2ac  d2c000e4       movz x4, #0x700000000
0x53402770   2b0  428083e4       stp c4, c0, [csp, #16 ]
0x53402774   2b4  c240db44       ldr c4, [c26, #432]
0x53402778   2b8  c20003e4       str c4, [csp]
0x5340277c   2bc  d2800060       movz x0, #0x3
0x53402780   2c0  c24017e1       ldr c1, [csp, #40]
0x53402784   2c4                                             ;; object: 0x0867828c0b51 <NativeContext[282]>
0x53402788   2c8                                             ;; off heap target
0x5340278c   2cc  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402790   2d0  c2c09200       gctag x0, c16
0x53402794   2d4  37000040       tbnz w0, #0, #+0x8 (addr 0x5340279c)
0x53402798   2d8  d4200000       brk #0x0
0x5340279c   2dc  22c103ff       ldp c31, c0, [csp], #32
0x534027a0   2e0  c2c0b211       gcseal x17, c16
0x534027a4   2e4  37000071       tbnz w17, #0, #+0xc (addr 0x534027b0)
0x534027a8   2e8  b2400211       orr x17, x16, #0x1
0x534027ac   2ec  c2d14210       scvalue c16, c16, x17
0x534027b0   2f0  37000050       tbnz w16, #0, #+0x8 (addr 0x534027b8)
0x534027b4   2f4  d4200000       brk #0x0
0x534027b8   2f8  c2c23200       blr x16
0x534027bc   2fc  f85d03a3       ldur x3, [x29, #-48]
0x534027c0   300  c2c1d3bf       mov csp, fp
0x534027c4   304  22c17bfd       ldp fp, lr, [csp], #32
0x534027c8   308  91000470       add x16, x3, #0x1 (1)
0x534027cc   30c  927ffa10       and x16, x16, #0xfffffffffffffffe
0x534027d0   310  b6f802b0       tbz x16, #63, #+0x54 (addr 0x53402824)
0x534027d4   314  d2c006e1       movz x1, #0x3700000000
0x534027d8   318  c2659f50       ldr c16, [c26, #19256]
0x534027dc   31c  c2c0b211       gcseal x17, c16
0x534027e0   320  37000071       tbnz w17, #0, #+0xc (addr 0x534027ec)
0x534027e4   324  b2400211       orr x17, x16, #0x1
0x534027e8   328  c2d14210       scvalue c16, c16, x17
0x534027ec   32c  37000050       tbnz w16, #0, #+0x8 (addr 0x534027f4)
0x534027f0   330  d4200000       brk #0x0
0x534027f4   334  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x534027f8   338  c2c09200       gctag x0, c16
0x534027fc   33c  37000040       tbnz w0, #0, #+0x8 (addr 0x53402804)
0x53402800   340  d4200000       brk #0x0
0x53402804   344  22c103ff       ldp c31, c0, [csp], #32
0x53402808   348  c2c0b211       gcseal x17, c16
0x5340280c   34c  37000071       tbnz w17, #0, #+0xc (addr 0x53402818)
0x53402810   350  b2400211       orr x17, x16, #0x1
0x53402814   354  c2d14210       scvalue c16, c16, x17
0x53402818   358  37000050       tbnz w16, #0, #+0x8 (addr 0x53402820)
0x5340281c   35c  d4200000       brk #0x0
0x53402820   360  c2c23200       blr x16
0x53402824   364  c2b073ff       add csp, csp, x16, lsl #4
0x53402828   368  910003f0       mov x16, sp
0x5340282c   36c  f2400e1f       tst x16, #0xf
0x53402830   370  540001e0       b.eq #+0x3c (addr 0x5340286c)
0x53402834   374  52800760       movz w0, #0x3b
0x53402838   378                                             ;; external reference (abort_with_reason)
0x5340283c   37c  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402840   380  c2c09200       gctag x0, c16
0x53402844   384  37000040       tbnz w0, #0, #+0x8 (addr 0x5340284c)
0x53402848   388  d4200000       brk #0x0
0x5340284c   38c  22c103ff       ldp c31, c0, [csp], #32
0x53402850   390  c2c0b211       gcseal x17, c16
0x53402854   394  37000071       tbnz w17, #0, #+0xc (addr 0x53402860)
0x53402858   398  b2400211       orr x17, x16, #0x1
0x5340285c   39c  c2d14210       scvalue c16, c16, x17
0x53402860   3a0  37000050       tbnz w16, #0, #+0x8 (addr 0x53402868)
0x53402864   3a4  d4200000       brk #0x0
0x53402868   3a8  c2c23200       blr x16
0x5340286c   3ac  c2c253c0       ret
0x53402870   3b0  d2c00e04       movz x4, #0x7000000000
0x53402874   3b4  910003f0       mov x16, sp
0x53402878   3b8  f2400e1f       tst x16, #0xf
0x5340287c   3bc  540001e0       b.eq #+0x3c (addr 0x534028b8)
0x53402880   3c0  52800760       movz w0, #0x3b
0x53402884   3c4                                             ;; external reference (abort_with_reason)
0x53402888   3c8  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x5340288c   3cc  c2c09200       gctag x0, c16
0x53402890   3d0  37000040       tbnz w0, #0, #+0x8 (addr 0x53402898)
0x53402894   3d4  d4200000       brk #0x0
0x53402898   3d8  22c103ff       ldp c31, c0, [csp], #32
0x5340289c   3dc  c2c0b211       gcseal x17, c16
0x534028a0   3e0  37000071       tbnz w17, #0, #+0xc (addr 0x534028ac)
0x534028a4   3e4  b2400211       orr x17, x16, #0x1
0x534028a8   3e8  c2d14210       scvalue c16, c16, x17
0x534028ac   3ec  37000050       tbnz w16, #0, #+0x8 (addr 0x534028b4)
0x534028b0   3f0  d4200000       brk #0x0
0x534028b4   3f4  c2c23200       blr x16
0x534028b8   3f8  028083ff       sub csp, csp, #0x20 (32)
0x534028bc   3fc  c20007ff       str c31, [csp, #8]
0x534028c0   400  c20003e4       str c4, [csp]
0x534028c4   404                                             ;; object: 0x0867828c0b51 <NativeContext[282]>
0x534028c8   408  d2800020       movz x0, #0x1
0x534028cc   40c                                             ;; external reference (Runtime::StackGuardWithGap)
0x534028d0   410                                             ;; off heap target
0x534028d4   414  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x534028d8   418  c2c09200       gctag x0, c16
0x534028dc   41c  37000040       tbnz w0, #0, #+0x8 (addr 0x534028e4)
0x534028e0   420  d4200000       brk #0x0
0x534028e4   424  22c103ff       ldp c31, c0, [csp], #32
0x534028e8   428  c2c0b211       gcseal x17, c16
0x534028ec   42c  37000071       tbnz w17, #0, #+0xc (addr 0x534028f8)
0x534028f0   430  b2400211       orr x17, x16, #0x1
0x534028f4   434  c2d14210       scvalue c16, c16, x17
0x534028f8   438  37000050       tbnz w16, #0, #+0x8 (addr 0x53402900)
0x534028fc   43c  d4200000       brk #0x0
0x53402900   440  c2c23200       blr x16
0x53402904   444  17ffff40       b #-0x300 (addr 0x53402604)
0x53402908   448  d503201f       nop
0x5340290c   44c  5800059f       constant pool begin (num_const = 44)    ;; constant pool
0x53402910   450  c2c233e0       constant
0x53402914   454  d503201f       constant
0x53402918   458  d503201f       constant
0x5340291c   45c  d503201f       constant
0x53402920   460  79046171       constant
0x53402924   464  00002c8f       constant
0x53402928   468  3f074003       constant
0x5340292c   46c  dc5f4000       constant
0x53402930   470  828c0b51       constant
0x53402934   474  00000867       constant
0x53402938   478  3f07c003       constant
0x5340293c   47c  dc5f4000       constant
0x53402940   480  79045ed1       constant
0x53402944   484  00002c8f       constant
0x53402948   488  3f074003       constant
0x5340294c   48c  dc5f4000       constant
0x53402950   490  08b3c52d       constant
0x53402954   494  00000000       constant
0x53402958   498  b7c6008a       constant
0x5340295c   49c  b05dc000       constant
0x53402960   4a0  09c00ea1       constant
0x53402964   4a4  00000000       constant
0x53402968   4a8  b7c6008a       constant
0x5340296c   4ac  b05dc000       constant
0x53402970   4b0  0aea2801       constant
0x53402974   4b4  00000000       constant
0x53402978   4b8  b7c6008a       constant
0x5340297c   4bc  b05dc000       constant
0x53402980   4c0  0aed5c81       constant
0x53402984   4c4  00000000       constant
0x53402988   4c8  b7c6008a       constant
0x5340298c   4cc  b05dc000       constant
0x53402990   4d0  0aeff481       constant
0x53402994   4d4  00000000       constant
0x53402998   4d8  b7c6008a       constant
0x5340299c   4dc  b05dc000       constant
0x534029a0   4e0  0b17d181       constant
0x534029a4   4e4  00000000       constant
0x534029a8   4e8  b7c6008a       constant
0x534029ac   4ec  b05dc000       constant
0x534029b0   4f0  0b31d601       constant
0x534029b4   4f4  00000000       constant
0x534029b8   4f8  b7c6008a       constant
0x534029bc   4fc  b05dc000       constant
0x534029c0   500  a25a0350       ldur c16, [c26, #-96]
0x534029c4   504  c2c0b211       gcseal x17, c16
0x534029c8   508  37000071       tbnz w17, #0, #+0xc (addr 0x534029d4)
0x534029cc   50c  b2400211       orr x17, x16, #0x1
0x534029d0   510  c2d14210       scvalue c16, c16, x17
0x534029d4   514  37000050       tbnz w16, #0, #+0x8 (addr 0x534029dc)
0x534029d8   518  d4200000       brk #0x0
0x534029dc   51c  c2c0b211       gcseal x17, c16
0x534029e0   520  37000071       tbnz w17, #0, #+0xc (addr 0x534029ec)
0x534029e4   524  b2400211       orr x17, x16, #0x1
0x534029e8   528  c2d14210       scvalue c16, c16, x17
0x534029ec   52c  37000050       tbnz w16, #0, #+0x8 (addr 0x534029f4)
0x534029f0   530  d4200000       brk #0x0
0x534029f4   534  c2c21200       br x16
0x534029f8   538  97fffff2       bl #-0x38 (addr 0x534029c0)
0x534029fc   53c  97fffff1       bl #-0x3c (addr 0x534029c0)
0x53402a00   540  97fffff0       bl #-0x40 (addr 0x534029c0)
0x53402a04   544  97ffffef       bl #-0x44 (addr 0x534029c0)
0x53402a08   548  97ffffee       bl #-0x48 (addr 0x534029c0)

Inlined functions (count = 0)

Deoptimization Input Data (deopt points = 5)
 index  bytecode-offset  node-id    pc
     0                0       15   184
     1                7       21   1c8
     2               11       26   260
     3               17       31   2fc
     4               -1        7   444

Safepoints (entries = 5, byte size = 38)
0x53402644    184  slots (sp->fp): 00100000  deopt      0 trampoline:    538
0x53402688    1c8  slots (sp->fp): 01100000  deopt      1 trampoline:    53c
0x53402720    260  slots (sp->fp): 01100000  deopt      2 trampoline:    540
0x534027bc    2fc  slots (sp->fp): 00100000  deopt      3 trampoline:    544
0x53402904    444  slots (sp->fp): 00100000  deopt      4 trampoline:    548

RelocInfo (size = 23)
0x53402554  off heap target
0x534025ac  external reference (abort_with_reason)  (0x8b3c52d)
0x53402604  full embedded object  (0x2c8f79046171 <String[1]: #f>)
0x5340260c  full embedded object  (0x0867828c0b51 <NativeContext[11a]>)
0x53402610  off heap target
0x53402648  full embedded object  (0x2c8f79045ed1 <String[1]: #X>)
0x534026ec  off heap target
0x53402788  off heap target
0x534028cc  external reference (Runtime::StackGuardWithGap)  (0x9c00ea1)
0x534028d0  off heap target
0x5340290c  constant pool (size b4)

--- End code ---
#+end_src


#+begin_src turbo
--- Raw source ---
(x, y) { return x + y; };

--- Optimized code ---
optimization_id = 2
source_position = 10
kind = TURBOFAN
name = f
stack_slots = 6
compiler = turbofan
address = 0x23ddbddc4771

Instructions (size = 912)
0x53402a80     0  10000010       adr c16, #+0x0 (addr 0x53402a80)
0x53402a84     4  c2c0b211       gcseal x17, c16
0x53402a88     8  37000071       tbnz w17, #0, #+0xc (addr 0x53402a94)
0x53402a8c     c  b2400211       orr x17, x16, #0x1
0x53402a90    10  c2d14210       scvalue c16, c16, x17
0x53402a94    14  37000050       tbnz w16, #0, #+0x8 (addr 0x53402a9c)
0x53402a98    18  d4200000       brk #0x0
0x53402a9c    1c  c2c0b051       gcseal x17, c2
0x53402aa0    20  37000071       tbnz w17, #0, #+0xc (addr 0x53402aac)
0x53402aa4    24  b2400051       orr x17, x2, #0x1
0x53402aa8    28  c2d14042       scvalue c2, c2, x17
0x53402aac    2c  37000042       tbnz w2, #0, #+0x8 (addr 0x53402ab4)
0x53402ab0    30  d4200000       brk #0x0
0x53402ab4    34  eb02021f       cmp x16, x2
0x53402ab8    38  540002a0       b.eq #+0x54 (addr 0x53402b0c)
0x53402abc    3c  d2c00821       movz x1, #0x4100000000
0x53402ac0    40  c2659f50       ldr c16, [c26, #19256]
0x53402ac4    44  c2c0b211       gcseal x17, c16
0x53402ac8    48  37000071       tbnz w17, #0, #+0xc (addr 0x53402ad4)
0x53402acc    4c  b2400211       orr x17, x16, #0x1
0x53402ad0    50  c2d14210       scvalue c16, c16, x17
0x53402ad4    54  37000050       tbnz w16, #0, #+0x8 (addr 0x53402adc)
0x53402ad8    58  d4200000       brk #0x0
0x53402adc    5c  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402ae0    60  c2c09200       gctag x0, c16
0x53402ae4    64  37000040       tbnz w0, #0, #+0x8 (addr 0x53402aec)
0x53402ae8    68  d4200000       brk #0x0
0x53402aec    6c  22c103ff       ldp c31, c0, [csp], #32
0x53402af0    70  c2c0b211       gcseal x17, c16
0x53402af4    74  37000071       tbnz w17, #0, #+0xc (addr 0x53402b00)
0x53402af8    78  b2400211       orr x17, x16, #0x1
0x53402afc    7c  c2d14210       scvalue c16, c16, x17
0x53402b00    80  37000050       tbnz w16, #0, #+0x8 (addr 0x53402b08)
0x53402b04    84  d4200000       brk #0x0
0x53402b08    88  c2c23200       blr x16
0x53402b0c    8c  b8584350       ldur w16, [x26, #-124]
0x53402b10    90  36e80270       tbz w16, #29, #+0x4c (addr 0x53402b5c)
0x53402b14    94                                             ;; off heap target
0x53402b18    98  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402b1c    9c  c2c0b220       gcseal x0, c17
0x53402b20    a0  37000060       tbnz w0, #0, #+0xc (addr 0x53402b2c)
0x53402b24    a4  b2400220       orr x0, x17, #0x1
0x53402b28    a8  c2c04231       scvalue c17, c17, x0
0x53402b2c    ac  37000051       tbnz w17, #0, #+0x8 (addr 0x53402b34)
0x53402b30    b0  d4200000       brk #0x0
0x53402b34    b4  22c103ff       ldp c31, c0, [csp], #32
0x53402b38    b8  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402b3c    bc  c2c0b220       gcseal x0, c17
0x53402b40    c0  37000060       tbnz w0, #0, #+0xc (addr 0x53402b4c)
0x53402b44    c4  b2400220       orr x0, x17, #0x1
0x53402b48    c8  c2c04231       scvalue c17, c17, x0
0x53402b4c    cc  37000051       tbnz w17, #0, #+0x8 (addr 0x53402b54)
0x53402b50    d0  d4200000       brk #0x0
0x53402b54    d4  22c103ff       ldp c31, c0, [csp], #32
0x53402b58    d8  c2c21220       br x17
0x53402b5c    dc  910003f0       mov x16, sp
0x53402b60    e0  f2400e1f       tst x16, #0xf
0x53402b64    e4  540001e0       b.eq #+0x3c (addr 0x53402ba0)
0x53402b68    e8  52800760       movz w0, #0x3b
0x53402b6c    ec                                             ;; external reference (abort_with_reason)
0x53402b70    f0  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402b74    f4  c2c09200       gctag x0, c16
0x53402b78    f8  37000040       tbnz w0, #0, #+0x8 (addr 0x53402b80)
0x53402b7c    fc  d4200000       brk #0x0
0x53402b80   100  22c103ff       ldp c31, c0, [csp], #32
0x53402b84   104  c2c0b211       gcseal x17, c16
0x53402b88   108  37000071       tbnz w17, #0, #+0xc (addr 0x53402b94)
0x53402b8c   10c  b2400211       orr x17, x16, #0x1
0x53402b90   110  c2d14210       scvalue c16, c16, x17
0x53402b94   114  37000050       tbnz w16, #0, #+0x8 (addr 0x53402b9c)
0x53402b98   118  d4200000       brk #0x0
0x53402b9c   11c  c2c23200       blr x16
0x53402ba0   120  62bf7bfd       stp fp, lr, [csp, #-32 ]!
0x53402ba4   124  c2c1d3fd       mov fp, csp
0x53402ba8   128  62be03ff       stp c31, c0, [csp, #-64 ]!
0x53402bac   12c  42816fe1       stp c1, cp, [csp, #32 ]
0x53402bb0   130  a2540342       ldur c2, [c26, #-192]
0x53402bb4   134  c20003fb       str cp, [csp]
0x53402bb8   138  eb2263ff       cmp sp, x2
0x53402bbc   13c  54000829       b.ls #+0x104 (addr 0x53402cc0)
0x53402bc0   140  c2401fe0       ldr c0, [csp, #56]
0x53402bc4   144  c24023e1       ldr c1, [csp, #64]
0x53402bc8   148                                             ;; object: 0x0867828c0b51 <NativeContext[282]>
0x53402bcc   14c                                             ;; off heap target
0x53402bd0   150  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402bd4   154  c2c09200       gctag x0, c16
0x53402bd8   158  37000040       tbnz w0, #0, #+0x8 (addr 0x53402be0)
0x53402bdc   15c  d4200000       brk #0x0
0x53402be0   160  22c103ff       ldp c31, c0, [csp], #32
0x53402be4   164  c2c0b211       gcseal x17, c16
0x53402be8   168  37000071       tbnz w17, #0, #+0xc (addr 0x53402bf4)
0x53402bec   16c  b2400211       orr x17, x16, #0x1
0x53402bf0   170  c2d14210       scvalue c16, c16, x17
0x53402bf4   174  37000050       tbnz w16, #0, #+0x8 (addr 0x53402bfc)
0x53402bf8   178  d4200000       brk #0x0
0x53402bfc   17c  c2c23200       blr x16
0x53402c00   180  f85d03a3       ldur x3, [x29, #-48]
0x53402c04   184  c2c1d3bf       mov csp, fp
0x53402c08   188  22c17bfd       ldp fp, lr, [csp], #32
0x53402c0c   18c  f1000c7f       cmp x3, #0x3 (3)
0x53402c10   190  5400004a       b.ge #+0x8 (addr 0x53402c18)
0x53402c14   194  d2800063       movz x3, #0x3
0x53402c18   198  91000470       add x16, x3, #0x1 (1)
0x53402c1c   19c  927ffa10       and x16, x16, #0xfffffffffffffffe
0x53402c20   1a0  b6f802b0       tbz x16, #63, #+0x54 (addr 0x53402c74)
0x53402c24   1a4  d2c006e1       movz x1, #0x3700000000
0x53402c28   1a8  c2659f50       ldr c16, [c26, #19256]
0x53402c2c   1ac  c2c0b211       gcseal x17, c16
0x53402c30   1b0  37000071       tbnz w17, #0, #+0xc (addr 0x53402c3c)
0x53402c34   1b4  b2400211       orr x17, x16, #0x1
0x53402c38   1b8  c2d14210       scvalue c16, c16, x17
0x53402c3c   1bc  37000050       tbnz w16, #0, #+0x8 (addr 0x53402c44)
0x53402c40   1c0  d4200000       brk #0x0
0x53402c44   1c4  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402c48   1c8  c2c09200       gctag x0, c16
0x53402c4c   1cc  37000040       tbnz w0, #0, #+0x8 (addr 0x53402c54)
0x53402c50   1d0  d4200000       brk #0x0
0x53402c54   1d4  22c103ff       ldp c31, c0, [csp], #32
0x53402c58   1d8  c2c0b211       gcseal x17, c16
0x53402c5c   1dc  37000071       tbnz w17, #0, #+0xc (addr 0x53402c68)
0x53402c60   1e0  b2400211       orr x17, x16, #0x1
0x53402c64   1e4  c2d14210       scvalue c16, c16, x17
0x53402c68   1e8  37000050       tbnz w16, #0, #+0x8 (addr 0x53402c70)
0x53402c6c   1ec  d4200000       brk #0x0
0x53402c70   1f0  c2c23200       blr x16
0x53402c74   1f4  c2b073ff       add csp, csp, x16, lsl #4
0x53402c78   1f8  910003f0       mov x16, sp
0x53402c7c   1fc  f2400e1f       tst x16, #0xf
0x53402c80   200  540001e0       b.eq #+0x3c (addr 0x53402cbc)
0x53402c84   204  52800760       movz w0, #0x3b
0x53402c88   208                                             ;; external reference (abort_with_reason)
0x53402c8c   20c  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402c90   210  c2c09200       gctag x0, c16
0x53402c94   214  37000040       tbnz w0, #0, #+0x8 (addr 0x53402c9c)
0x53402c98   218  d4200000       brk #0x0
0x53402c9c   21c  22c103ff       ldp c31, c0, [csp], #32
0x53402ca0   220  c2c0b211       gcseal x17, c16
0x53402ca4   224  37000071       tbnz w17, #0, #+0xc (addr 0x53402cb0)
0x53402ca8   228  b2400211       orr x17, x16, #0x1
0x53402cac   22c  c2d14210       scvalue c16, c16, x17
0x53402cb0   230  37000050       tbnz w16, #0, #+0x8 (addr 0x53402cb8)
0x53402cb4   234  d4200000       brk #0x0
0x53402cb8   238  c2c23200       blr x16
0x53402cbc   23c  c2c253c0       ret
0x53402cc0   240  d2c00a02       movz x2, #0x5000000000
0x53402cc4   244  910003f0       mov x16, sp
0x53402cc8   248  f2400e1f       tst x16, #0xf
0x53402ccc   24c  540001e0       b.eq #+0x3c (addr 0x53402d08)
0x53402cd0   250  52800760       movz w0, #0x3b
0x53402cd4   254                                             ;; external reference (abort_with_reason)
0x53402cd8   258  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402cdc   25c  c2c09200       gctag x0, c16
0x53402ce0   260  37000040       tbnz w0, #0, #+0x8 (addr 0x53402ce8)
0x53402ce4   264  d4200000       brk #0x0
0x53402ce8   268  22c103ff       ldp c31, c0, [csp], #32
0x53402cec   26c  c2c0b211       gcseal x17, c16
0x53402cf0   270  37000071       tbnz w17, #0, #+0xc (addr 0x53402cfc)
0x53402cf4   274  b2400211       orr x17, x16, #0x1
0x53402cf8   278  c2d14210       scvalue c16, c16, x17
0x53402cfc   27c  37000050       tbnz w16, #0, #+0x8 (addr 0x53402d04)
0x53402d00   280  d4200000       brk #0x0
0x53402d04   284  c2c23200       blr x16
0x53402d08   288  028083ff       sub csp, csp, #0x20 (32)
0x53402d0c   28c  c20007ff       str c31, [csp, #8]
0x53402d10   290  c20003e2       str c2, [csp]
0x53402d14   294                                             ;; object: 0x0867828c0b51 <NativeContext[282]>
0x53402d18   298                                             ;; external reference (Runtime::StackGuardWithGap)
0x53402d1c   29c  d2800020       movz x0, #0x1
0x53402d20   2a0                                             ;; off heap target
0x53402d24   2a4  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402d28   2a8  c2c09200       gctag x0, c16
0x53402d2c   2ac  37000040       tbnz w0, #0, #+0x8 (addr 0x53402d34)
0x53402d30   2b0  d4200000       brk #0x0
0x53402d34   2b4  22c103ff       ldp c31, c0, [csp], #32
0x53402d38   2b8  c2c0b211       gcseal x17, c16
0x53402d3c   2bc  37000071       tbnz w17, #0, #+0xc (addr 0x53402d48)
0x53402d40   2c0  b2400211       orr x17, x16, #0x1
0x53402d44   2c4  c2d14210       scvalue c16, c16, x17
0x53402d48   2c8  37000050       tbnz w16, #0, #+0x8 (addr 0x53402d50)
0x53402d4c   2cc  d4200000       brk #0x0
0x53402d50   2d0  c2c23200       blr x16
0x53402d54   2d4  17ffff9b       b #-0x194 (addr 0x53402bc0)
0x53402d58   2d8  d503201f       nop
0x53402d5c   2dc  5800039f       constant pool begin (num_const = 28)    ;; constant pool
0x53402d60   2e0  c2c233e0       constant
0x53402d64   2e4  d503201f       constant
0x53402d68   2e8  d503201f       constant
0x53402d6c   2ec  d503201f       constant
0x53402d70   2f0  828c0b51       constant
0x53402d74   2f4  00000867       constant
0x53402d78   2f8  3f07c003       constant
0x53402d7c   2fc  dc5f4000       constant
0x53402d80   300  08b3c52d       constant
0x53402d84   304  00000000       constant
0x53402d88   308  b7c6008a       constant
0x53402d8c   30c  b05dc000       constant
0x53402d90   310  09c00ea1       constant
0x53402d94   314  00000000       constant
0x53402d98   318  b7c6008a       constant
0x53402d9c   31c  b05dc000       constant
0x53402da0   320  0aeff481       constant
0x53402da4   324  00000000       constant
0x53402da8   328  b7c6008a       constant
0x53402dac   32c  b05dc000       constant
0x53402db0   330  0b31d601       constant
0x53402db4   334  00000000       constant
0x53402db8   338  b7c6008a       constant
0x53402dbc   33c  b05dc000       constant
0x53402dc0   340  0b5bafc1       constant
0x53402dc4   344  00000000       constant
0x53402dc8   348  b7c6008a       constant
0x53402dcc   34c  b05dc000       constant
0x53402dd0   350  a25a0350       ldur c16, [c26, #-96]
0x53402dd4   354  c2c0b211       gcseal x17, c16
0x53402dd8   358  37000071       tbnz w17, #0, #+0xc (addr 0x53402de4)
0x53402ddc   35c  b2400211       orr x17, x16, #0x1
0x53402de0   360  c2d14210       scvalue c16, c16, x17
0x53402de4   364  37000050       tbnz w16, #0, #+0x8 (addr 0x53402dec)
0x53402de8   368  d4200000       brk #0x0
0x53402dec   36c  c2c0b211       gcseal x17, c16
0x53402df0   370  37000071       tbnz w17, #0, #+0xc (addr 0x53402dfc)
0x53402df4   374  b2400211       orr x17, x16, #0x1
0x53402df8   378  c2d14210       scvalue c16, c16, x17
0x53402dfc   37c  37000050       tbnz w16, #0, #+0x8 (addr 0x53402e04)
0x53402e00   380  d4200000       brk #0x0
0x53402e04   384  c2c21200       br x16
0x53402e08   388  97fffff2       bl #-0x38 (addr 0x53402dd0)
0x53402e0c   38c  97fffff1       bl #-0x3c (addr 0x53402dd0)

Inlined functions (count = 0)

Deoptimization Input Data (deopt points = 2)
 index  bytecode-offset  node-id    pc
     0                2       17   180
     1               -1        9   2d4

Safepoints (entries = 2, byte size = 20)
0x53402c00    180  slots (sp->fp): 10000000  deopt      0 trampoline:    388
0x53402d54    2d4  slots (sp->fp): 10000000  deopt      1 trampoline:    38c

RelocInfo (size = 19)
0x53402b14  off heap target
0x53402b6c  external reference (abort_with_reason)  (0x8b3c52d)
0x53402bc8  full embedded object  (0x0867828c0b51 <NativeContext[11a]>)
0x53402bcc  off heap target
0x53402d18  external reference (Runtime::StackGuardWithGap)  (0x9c00ea1)
0x53402d20  off heap target
0x53402d5c  constant pool (size 74)

--- End code ---
#+end_src


#+begin_src turbo
--- Raw source ---
deopt = true;var result = f(7, new X());

--- Optimized code ---
optimization_id = 3
source_position = 0
kind = TURBOFAN
stack_slots = 8
compiler = turbofan
address = 0x23ddbddc5281

Instructions (size = 1736)
0x53402e80     0  10000010       adr c16, #+0x0 (addr 0x53402e80)
0x53402e84     4  c2c0b211       gcseal x17, c16
0x53402e88     8  37000071       tbnz w17, #0, #+0xc (addr 0x53402e94)
0x53402e8c     c  b2400211       orr x17, x16, #0x1
0x53402e90    10  c2d14210       scvalue c16, c16, x17
0x53402e94    14  37000050       tbnz w16, #0, #+0x8 (addr 0x53402e9c)
0x53402e98    18  d4200000       brk #0x0
0x53402e9c    1c  c2c0b051       gcseal x17, c2
0x53402ea0    20  37000071       tbnz w17, #0, #+0xc (addr 0x53402eac)
0x53402ea4    24  b2400051       orr x17, x2, #0x1
0x53402ea8    28  c2d14042       scvalue c2, c2, x17
0x53402eac    2c  37000042       tbnz w2, #0, #+0x8 (addr 0x53402eb4)
0x53402eb0    30  d4200000       brk #0x0
0x53402eb4    34  eb02021f       cmp x16, x2
0x53402eb8    38  540002a0       b.eq #+0x54 (addr 0x53402f0c)
0x53402ebc    3c  d2c00821       movz x1, #0x4100000000
0x53402ec0    40  c2659f50       ldr c16, [c26, #19256]
0x53402ec4    44  c2c0b211       gcseal x17, c16
0x53402ec8    48  37000071       tbnz w17, #0, #+0xc (addr 0x53402ed4)
0x53402ecc    4c  b2400211       orr x17, x16, #0x1
0x53402ed0    50  c2d14210       scvalue c16, c16, x17
0x53402ed4    54  37000050       tbnz w16, #0, #+0x8 (addr 0x53402edc)
0x53402ed8    58  d4200000       brk #0x0
0x53402edc    5c  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402ee0    60  c2c09200       gctag x0, c16
0x53402ee4    64  37000040       tbnz w0, #0, #+0x8 (addr 0x53402eec)
0x53402ee8    68  d4200000       brk #0x0
0x53402eec    6c  22c103ff       ldp c31, c0, [csp], #32
0x53402ef0    70  c2c0b211       gcseal x17, c16
0x53402ef4    74  37000071       tbnz w17, #0, #+0xc (addr 0x53402f00)
0x53402ef8    78  b2400211       orr x17, x16, #0x1
0x53402efc    7c  c2d14210       scvalue c16, c16, x17
0x53402f00    80  37000050       tbnz w16, #0, #+0x8 (addr 0x53402f08)
0x53402f04    84  d4200000       brk #0x0
0x53402f08    88  c2c23200       blr x16
0x53402f0c    8c  b8584350       ldur w16, [x26, #-124]
0x53402f10    90  36e80270       tbz w16, #29, #+0x4c (addr 0x53402f5c)
0x53402f14    94                                             ;; off heap target
0x53402f18    98  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402f1c    9c  c2c0b220       gcseal x0, c17
0x53402f20    a0  37000060       tbnz w0, #0, #+0xc (addr 0x53402f2c)
0x53402f24    a4  b2400220       orr x0, x17, #0x1
0x53402f28    a8  c2c04231       scvalue c17, c17, x0
0x53402f2c    ac  37000051       tbnz w17, #0, #+0x8 (addr 0x53402f34)
0x53402f30    b0  d4200000       brk #0x0
0x53402f34    b4  22c103ff       ldp c31, c0, [csp], #32
0x53402f38    b8  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402f3c    bc  c2c0b220       gcseal x0, c17
0x53402f40    c0  37000060       tbnz w0, #0, #+0xc (addr 0x53402f4c)
0x53402f44    c4  b2400220       orr x0, x17, #0x1
0x53402f48    c8  c2c04231       scvalue c17, c17, x0
0x53402f4c    cc  37000051       tbnz w17, #0, #+0x8 (addr 0x53402f54)
0x53402f50    d0  d4200000       brk #0x0
0x53402f54    d4  22c103ff       ldp c31, c0, [csp], #32
0x53402f58    d8  c2c21220       br x17
0x53402f5c    dc  910003f0       mov x16, sp
0x53402f60    e0  f2400e1f       tst x16, #0xf
0x53402f64    e4  540001e0       b.eq #+0x3c (addr 0x53402fa0)
0x53402f68    e8  52800760       movz w0, #0x3b
0x53402f6c    ec                                             ;; external reference (abort_with_reason)
0x53402f70    f0  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402f74    f4  c2c09200       gctag x0, c16
0x53402f78    f8  37000040       tbnz w0, #0, #+0x8 (addr 0x53402f80)
0x53402f7c    fc  d4200000       brk #0x0
0x53402f80   100  22c103ff       ldp c31, c0, [csp], #32
0x53402f84   104  c2c0b211       gcseal x17, c16
0x53402f88   108  37000071       tbnz w17, #0, #+0xc (addr 0x53402f94)
0x53402f8c   10c  b2400211       orr x17, x16, #0x1
0x53402f90   110  c2d14210       scvalue c16, c16, x17
0x53402f94   114  37000050       tbnz w16, #0, #+0x8 (addr 0x53402f9c)
0x53402f98   118  d4200000       brk #0x0
0x53402f9c   11c  c2c23200       blr x16
0x53402fa0   120  62bf7bfd       stp fp, lr, [csp, #-32 ]!
0x53402fa4   124  c2c1d3fd       mov fp, csp
0x53402fa8   128  62be03ff       stp c31, c0, [csp, #-64 ]!
0x53402fac   12c  42816fe1       stp c1, cp, [csp, #32 ]
0x53402fb0   130  028083ff       sub csp, csp, #0x20 (32)
0x53402fb4   134  a2540345       ldur c5, [c26, #-192]
0x53402fb8   138  c2000bfb       str cp, [csp, #16]
0x53402fbc   13c  eb2563ff       cmp sp, x5
0x53402fc0   140  54001c69       b.ls #+0x38c (addr 0x5340334c)
0x53402fc4   144  910003f0       mov x16, sp
0x53402fc8   148  f2400e1f       tst x16, #0xf
0x53402fcc   14c  540001e0       b.eq #+0x3c (addr 0x53403008)
0x53402fd0   150  52800760       movz w0, #0x3b
0x53402fd4   154                                             ;; external reference (abort_with_reason)
0x53402fd8   158  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53402fdc   15c  c2c09200       gctag x0, c16
0x53402fe0   160  37000040       tbnz w0, #0, #+0x8 (addr 0x53402fe8)
0x53402fe4   164  d4200000       brk #0x0
0x53402fe8   168  22c103ff       ldp c31, c0, [csp], #32
0x53402fec   16c  c2c0b211       gcseal x17, c16
0x53402ff0   170  37000071       tbnz w17, #0, #+0xc (addr 0x53402ffc)
0x53402ff4   174  b2400211       orr x17, x16, #0x1
0x53402ff8   178  c2d14210       scvalue c16, c16, x17
0x53402ffc   17c  37000050       tbnz w16, #0, #+0x8 (addr 0x53403004)
0x53403000   180  d4200000       brk #0x0
0x53403004   184  c2c23200       blr x16
0x53403008   188  028083ff       sub csp, csp, #0x20 (32)
0x5340300c   18c                                             ;; object: 0x23ddbddc4b31 <FixedArray[1]>
0x53403010   190                                             ;; object: 0x23ddbddc4c81 <JSFunction (sfi = 0x23ddbddc4a31)>
0x53403014   194  428017e6       stp c6, c5, [csp ]
0x53403018   198                                             ;; external reference (Runtime::DeclareGlobals)
0x5340301c   19c  d2800040       movz x0, #0x2
0x53403020   1a0  c24013fb       ldr cp, [csp, #32]
0x53403024   1a4                                             ;; off heap target
0x53403028   1a8  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x5340302c   1ac  c2c09200       gctag x0, c16
0x53403030   1b0  37000040       tbnz w0, #0, #+0x8 (addr 0x53403038)
0x53403034   1b4  d4200000       brk #0x0
0x53403038   1b8  22c103ff       ldp c31, c0, [csp], #32
0x5340303c   1bc  c2c0b211       gcseal x17, c16
0x53403040   1c0  37000071       tbnz w17, #0, #+0xc (addr 0x5340304c)
0x53403044   1c4  b2400211       orr x17, x16, #0x1
0x53403048   1c8  c2d14210       scvalue c16, c16, x17
0x5340304c   1cc  37000050       tbnz w16, #0, #+0x8 (addr 0x53403054)
0x53403050   1d0  d4200000       brk #0x0
0x53403054   1d4  c2c23200       blr x16
0x53403058   1d8  c240e740       ldr c0, [c26, #456]
0x5340305c   1dc                                             ;; object: 0x23ddbddc1d51 <String[5]: #deopt>
0x53403060   1e0  d2800004       movz x4, #0x0
0x53403064   1e4  c2400bfb       ldr cp, [csp, #16]
0x53403068   1e8                                             ;; off heap target
0x5340306c   1ec  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53403070   1f0  c2c09200       gctag x0, c16
0x53403074   1f4  37000040       tbnz w0, #0, #+0x8 (addr 0x5340307c)
0x53403078   1f8  d4200000       brk #0x0
0x5340307c   1fc  22c103ff       ldp c31, c0, [csp], #32
0x53403080   200  c2c0b211       gcseal x17, c16
0x53403084   204  37000071       tbnz w17, #0, #+0xc (addr 0x53403090)
0x53403088   208  b2400211       orr x17, x16, #0x1
0x5340308c   20c  c2d14210       scvalue c16, c16, x17
0x53403090   210  37000050       tbnz w16, #0, #+0x8 (addr 0x53403098)
0x53403094   214  d4200000       brk #0x0
0x53403098   218  c2c23200       blr x16
0x5340309c   21c                                             ;; object: 0x2c8f79046171 <String[1]: #f>
0x534030a0   220  d2800080       movz x0, #0x4
0x534030a4   224                                             ;; object: 0x0867828c0b51 <NativeContext[282]>
0x534030a8   228                                             ;; off heap target
0x534030ac   22c  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x534030b0   230  c2c09200       gctag x0, c16
0x534030b4   234  37000040       tbnz w0, #0, #+0x8 (addr 0x534030bc)
0x534030b8   238  d4200000       brk #0x0
0x534030bc   23c  22c103ff       ldp c31, c0, [csp], #32
0x534030c0   240  c2c0b211       gcseal x17, c16
0x534030c4   244  37000071       tbnz w17, #0, #+0xc (addr 0x534030d0)
0x534030c8   248  b2400211       orr x17, x16, #0x1
0x534030cc   24c  c2d14210       scvalue c16, c16, x17
0x534030d0   250  37000050       tbnz w16, #0, #+0x8 (addr 0x534030d8)
0x534030d4   254  d4200000       brk #0x0
0x534030d8   258  c2c23200       blr x16
0x534030dc   25c  c20007e0       str c0, [csp, #8]
0x534030e0   260                                             ;; object: 0x2c8f79045ed1 <String[1]: #X>
0x534030e4   264  d2800100       movz x0, #0x8
0x534030e8   268                                             ;; object: 0x0867828c0b51 <NativeContext[282]>
0x534030ec   26c                                             ;; off heap target
0x534030f0   270  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x534030f4   274  c2c09200       gctag x0, c16
0x534030f8   278  37000040       tbnz w0, #0, #+0x8 (addr 0x53403100)
0x534030fc   27c  d4200000       brk #0x0
0x53403100   280  22c103ff       ldp c31, c0, [csp], #32
0x53403104   284  c2c0b211       gcseal x17, c16
0x53403108   288  37000071       tbnz w17, #0, #+0xc (addr 0x53403114)
0x5340310c   28c  b2400211       orr x17, x16, #0x1
0x53403110   290  c2d14210       scvalue c16, c16, x17
0x53403114   294  37000050       tbnz w16, #0, #+0x8 (addr 0x5340311c)
0x53403118   298  d4200000       brk #0x0
0x5340311c   29c  c2c23200       blr x16
0x53403120   2a0  910003f0       mov x16, sp
0x53403124   2a4  f2400e1f       tst x16, #0xf
0x53403128   2a8  540001e0       b.eq #+0x3c (addr 0x53403164)
0x5340312c   2ac  52800760       movz w0, #0x3b
0x53403130   2b0                                             ;; external reference (abort_with_reason)
0x53403134   2b4  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53403138   2b8  c2c09200       gctag x0, c16
0x5340313c   2bc  37000040       tbnz w0, #0, #+0x8 (addr 0x53403144)
0x53403140   2c0  d4200000       brk #0x0
0x53403144   2c4  22c103ff       ldp c31, c0, [csp], #32
0x53403148   2c8  c2c0b211       gcseal x17, c16
0x5340314c   2cc  37000071       tbnz w17, #0, #+0xc (addr 0x53403158)
0x53403150   2d0  b2400211       orr x17, x16, #0x1
0x53403154   2d4  c2d14210       scvalue c16, c16, x17
0x53403158   2d8  37000050       tbnz w16, #0, #+0x8 (addr 0x53403160)
0x5340315c   2dc  d4200000       brk #0x0
0x53403160   2e0  c2c23200       blr x16
0x53403164   2e4  028083ff       sub csp, csp, #0x20 (32)
0x53403168   2e8  c20007ff       str c31, [csp, #8]
0x5340316c   2ec  c240db45       ldr c5, [c26, #432]
0x53403170   2f0  c20003e5       str c5, [csp]
0x53403174   2f4  c2c1d001       mov c1, c0
0x53403178   2f8  c2c1d003       mov c3, c0
0x5340317c   2fc  d2800020       movz x0, #0x1
0x53403180   300                                             ;; object: 0x0867828c0b51 <NativeContext[282]>
0x53403184   304                                             ;; off heap target
0x53403188   308  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x5340318c   30c  c2c09200       gctag x0, c16
0x53403190   310  37000040       tbnz w0, #0, #+0x8 (addr 0x53403198)
0x53403194   314  d4200000       brk #0x0
0x53403198   318  22c103ff       ldp c31, c0, [csp], #32
0x5340319c   31c  c2c0b211       gcseal x17, c16
0x534031a0   320  37000071       tbnz w17, #0, #+0xc (addr 0x534031ac)
0x534031a4   324  b2400211       orr x17, x16, #0x1
0x534031a8   328  c2d14210       scvalue c16, c16, x17
0x534031ac   32c  37000050       tbnz w16, #0, #+0x8 (addr 0x534031b4)
0x534031b0   330  d4200000       brk #0x0
0x534031b4   334  c2c23200       blr x16
0x534031b8   338  910003f0       mov x16, sp
0x534031bc   33c  f2400e1f       tst x16, #0xf
0x534031c0   340  540001e0       b.eq #+0x3c (addr 0x534031fc)
0x534031c4   344  52800760       movz w0, #0x3b
0x534031c8   348                                             ;; external reference (abort_with_reason)
0x534031cc   34c  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x534031d0   350  c2c09200       gctag x0, c16
0x534031d4   354  37000040       tbnz w0, #0, #+0x8 (addr 0x534031dc)
0x534031d8   358  d4200000       brk #0x0
0x534031dc   35c  22c103ff       ldp c31, c0, [csp], #32
0x534031e0   360  c2c0b211       gcseal x17, c16
0x534031e4   364  37000071       tbnz w17, #0, #+0xc (addr 0x534031f0)
0x534031e8   368  b2400211       orr x17, x16, #0x1
0x534031ec   36c  c2d14210       scvalue c16, c16, x17
0x534031f0   370  37000050       tbnz w16, #0, #+0x8 (addr 0x534031f8)
0x534031f4   374  d4200000       brk #0x0
0x534031f8   378  c2c23200       blr x16
0x534031fc   37c  028103ff       sub csp, csp, #0x40 (64)
0x53403200   380  c2000fff       str c31, [csp, #24]
0x53403204   384  d2c000e5       movz x5, #0x700000000
0x53403208   388  428083e5       stp c5, c0, [csp, #16 ]
0x5340320c   38c  c240db45       ldr c5, [c26, #432]
0x53403210   390  c20003e5       str c5, [csp]
0x53403214   394  d2800060       movz x0, #0x3
0x53403218   398  c24017e1       ldr c1, [csp, #40]
0x5340321c   39c                                             ;; object: 0x0867828c0b51 <NativeContext[282]>
0x53403220   3a0                                             ;; off heap target
0x53403224   3a4  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53403228   3a8  c2c09200       gctag x0, c16
0x5340322c   3ac  37000040       tbnz w0, #0, #+0x8 (addr 0x53403234)
0x53403230   3b0  d4200000       brk #0x0
0x53403234   3b4  22c103ff       ldp c31, c0, [csp], #32
0x53403238   3b8  c2c0b211       gcseal x17, c16
0x5340323c   3bc  37000071       tbnz w17, #0, #+0xc (addr 0x53403248)
0x53403240   3c0  b2400211       orr x17, x16, #0x1
0x53403244   3c4  c2d14210       scvalue c16, c16, x17
0x53403248   3c8  37000050       tbnz w16, #0, #+0x8 (addr 0x53403250)
0x5340324c   3cc  d4200000       brk #0x0
0x53403250   3d0  c2c23200       blr x16
0x53403254   3d4                                             ;; object: 0x23ddbddc1d21 <String[6]: #result>
0x53403258   3d8  d2800284       movz x4, #0x14
0x5340325c   3dc  c2400bfb       ldr cp, [csp, #16]
0x53403260   3e0                                             ;; off heap target
0x53403264   3e4  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53403268   3e8  c2c09200       gctag x0, c16
0x5340326c   3ec  37000040       tbnz w0, #0, #+0x8 (addr 0x53403274)
0x53403270   3f0  d4200000       brk #0x0
0x53403274   3f4  22c103ff       ldp c31, c0, [csp], #32
0x53403278   3f8  c2c0b211       gcseal x17, c16
0x5340327c   3fc  37000071       tbnz w17, #0, #+0xc (addr 0x53403288)
0x53403280   400  b2400211       orr x17, x16, #0x1
0x53403284   404  c2d14210       scvalue c16, c16, x17
0x53403288   408  37000050       tbnz w16, #0, #+0x8 (addr 0x53403290)
0x5340328c   40c  d4200000       brk #0x0
0x53403290   410  c2c23200       blr x16
0x53403294   414  c240e740       ldr c0, [c26, #456]
0x53403298   418  f85d03a3       ldur x3, [x29, #-48]
0x5340329c   41c  c2c1d3bf       mov csp, fp
0x534032a0   420  22c17bfd       ldp fp, lr, [csp], #32
0x534032a4   424  91000470       add x16, x3, #0x1 (1)
0x534032a8   428  927ffa10       and x16, x16, #0xfffffffffffffffe
0x534032ac   42c  b6f802b0       tbz x16, #63, #+0x54 (addr 0x53403300)
0x534032b0   430  d2c006e1       movz x1, #0x3700000000
0x534032b4   434  c2659f50       ldr c16, [c26, #19256]
0x534032b8   438  c2c0b211       gcseal x17, c16
0x534032bc   43c  37000071       tbnz w17, #0, #+0xc (addr 0x534032c8)
0x534032c0   440  b2400211       orr x17, x16, #0x1
0x534032c4   444  c2d14210       scvalue c16, c16, x17
0x534032c8   448  37000050       tbnz w16, #0, #+0x8 (addr 0x534032d0)
0x534032cc   44c  d4200000       brk #0x0
0x534032d0   450  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x534032d4   454  c2c09200       gctag x0, c16
0x534032d8   458  37000040       tbnz w0, #0, #+0x8 (addr 0x534032e0)
0x534032dc   45c  d4200000       brk #0x0
0x534032e0   460  22c103ff       ldp c31, c0, [csp], #32
0x534032e4   464  c2c0b211       gcseal x17, c16
0x534032e8   468  37000071       tbnz w17, #0, #+0xc (addr 0x534032f4)
0x534032ec   46c  b2400211       orr x17, x16, #0x1
0x534032f0   470  c2d14210       scvalue c16, c16, x17
0x534032f4   474  37000050       tbnz w16, #0, #+0x8 (addr 0x534032fc)
0x534032f8   478  d4200000       brk #0x0
0x534032fc   47c  c2c23200       blr x16
0x53403300   480  c2b073ff       add csp, csp, x16, lsl #4
0x53403304   484  910003f0       mov x16, sp
0x53403308   488  f2400e1f       tst x16, #0xf
0x5340330c   48c  540001e0       b.eq #+0x3c (addr 0x53403348)
0x53403310   490  52800760       movz w0, #0x3b
0x53403314   494                                             ;; external reference (abort_with_reason)
0x53403318   498  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x5340331c   49c  c2c09200       gctag x0, c16
0x53403320   4a0  37000040       tbnz w0, #0, #+0x8 (addr 0x53403328)
0x53403324   4a4  d4200000       brk #0x0
0x53403328   4a8  22c103ff       ldp c31, c0, [csp], #32
0x5340332c   4ac  c2c0b211       gcseal x17, c16
0x53403330   4b0  37000071       tbnz w17, #0, #+0xc (addr 0x5340333c)
0x53403334   4b4  b2400211       orr x17, x16, #0x1
0x53403338   4b8  c2d14210       scvalue c16, c16, x17
0x5340333c   4bc  37000050       tbnz w16, #0, #+0x8 (addr 0x53403344)
0x53403340   4c0  d4200000       brk #0x0
0x53403344   4c4  c2c23200       blr x16
0x53403348   4c8  c2c253c0       ret
0x5340334c   4cc  d2c00e05       movz x5, #0x7000000000
0x53403350   4d0  910003f0       mov x16, sp
0x53403354   4d4  f2400e1f       tst x16, #0xf
0x53403358   4d8  540001e0       b.eq #+0x3c (addr 0x53403394)
0x5340335c   4dc  52800760       movz w0, #0x3b
0x53403360   4e0                                             ;; external reference (abort_with_reason)
0x53403364   4e4  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x53403368   4e8  c2c09200       gctag x0, c16
0x5340336c   4ec  37000040       tbnz w0, #0, #+0x8 (addr 0x53403374)
0x53403370   4f0  d4200000       brk #0x0
0x53403374   4f4  22c103ff       ldp c31, c0, [csp], #32
0x53403378   4f8  c2c0b211       gcseal x17, c16
0x5340337c   4fc  37000071       tbnz w17, #0, #+0xc (addr 0x53403388)
0x53403380   500  b2400211       orr x17, x16, #0x1
0x53403384   504  c2d14210       scvalue c16, c16, x17
0x53403388   508  37000050       tbnz w16, #0, #+0x8 (addr 0x53403390)
0x5340338c   50c  d4200000       brk #0x0
0x53403390   510  c2c23200       blr x16
0x53403394   514  028083ff       sub csp, csp, #0x20 (32)
0x53403398   518  c20007ff       str c31, [csp, #8]
0x5340339c   51c  c20003e5       str c5, [csp]
0x534033a0   520                                             ;; object: 0x0867828c0b51 <NativeContext[282]>
0x534033a4   524  d2800020       movz x0, #0x1
0x534033a8   528                                             ;; external reference (Runtime::StackGuardWithGap)
0x534033ac   52c                                             ;; off heap target
0x534033b0   530  62bf03ff       stp c31, c0, [csp, #-32 ]!
0x534033b4   534  c2c09200       gctag x0, c16
0x534033b8   538  37000040       tbnz w0, #0, #+0x8 (addr 0x534033c0)
0x534033bc   53c  d4200000       brk #0x0
0x534033c0   540  22c103ff       ldp c31, c0, [csp], #32
0x534033c4   544  c2c0b211       gcseal x17, c16
0x534033c8   548  37000071       tbnz w17, #0, #+0xc (addr 0x534033d4)
0x534033cc   54c  b2400211       orr x17, x16, #0x1
0x534033d0   550  c2d14210       scvalue c16, c16, x17
0x534033d4   554  37000050       tbnz w16, #0, #+0x8 (addr 0x534033dc)
0x534033d8   558  d4200000       brk #0x0
0x534033dc   55c  c2c23200       blr x16
0x534033e0   560  17fffef9       b #-0x41c (addr 0x53402fc4)
0x534033e4   564  d503201f       nop
0x534033e8   568  5800083f       constant pool begin (num_const = 65)    ;; constant pool
0x534033ec   56c  c2c233e0       constant
0x534033f0   570  bddc4b31       constant
0x534033f4   574  000023dd       constant
0x534033f8   578  3f07c003       constant
0x534033fc   57c  dc5f4000       constant
0x53403400   580  bddc4c81       constant
0x53403404   584  000023dd       constant
0x53403408   588  3f07c003       constant
0x5340340c   58c  dc5f4000       constant
0x53403410   590  bddc1d51       constant
0x53403414   594  000023dd       constant
0x53403418   598  3f07c003       constant
0x5340341c   59c  dc5f4000       constant
0x53403420   5a0  79046171       constant
0x53403424   5a4  00002c8f       constant
0x53403428   5a8  3f074003       constant
0x5340342c   5ac  dc5f4000       constant
0x53403430   5b0  828c0b51       constant
0x53403434   5b4  00000867       constant
0x53403438   5b8  3f07c003       constant
0x5340343c   5bc  dc5f4000       constant
0x53403440   5c0  79045ed1       constant
0x53403444   5c4  00002c8f       constant
0x53403448   5c8  3f074003       constant
0x5340344c   5cc  dc5f4000       constant
0x53403450   5d0  bddc1d21       constant
0x53403454   5d4  000023dd       constant
0x53403458   5d8  3f07c003       constant
0x5340345c   5dc  dc5f4000       constant
0x53403460   5e0  08b3c52d       constant
0x53403464   5e4  00000000       constant
0x53403468   5e8  b7c6008a       constant
0x5340346c   5ec  b05dc000       constant
0x53403470   5f0  09c00ea1       constant
0x53403474   5f4  00000000       constant
0x53403478   5f8  b7c6008a       constant
0x5340347c   5fc  b05dc000       constant
0x53403480   600  09c668c1       constant
0x53403484   604  00000000       constant
0x53403488   608  b7c6008a       constant
0x5340348c   60c  b05dc000       constant
0x53403490   610  0aea2801       constant
0x53403494   614  00000000       constant
0x53403498   618  b7c6008a       constant
0x5340349c   61c  b05dc000       constant
0x534034a0   620  0aed5c81       constant
0x534034a4   624  00000000       constant
0x534034a8   628  b7c6008a       constant
0x534034ac   62c  b05dc000       constant
0x534034b0   630  0aeff481       constant
0x534034b4   634  00000000       constant
0x534034b8   638  b7c6008a       constant
0x534034bc   63c  b05dc000       constant
0x534034c0   640  0b138481       constant
0x534034c4   644  00000000       constant
0x534034c8   648  b7c6008a       constant
0x534034cc   64c  b05dc000       constant
0x534034d0   650  0b17d181       constant
0x534034d4   654  00000000       constant
0x534034d8   658  b7c6008a       constant
0x534034dc   65c  b05dc000       constant
0x534034e0   660  0b31d601       constant
0x534034e4   664  00000000       constant
0x534034e8   668  b7c6008a       constant
0x534034ec   66c  b05dc000       constant
0x534034f0   670  a25a0350       ldur c16, [c26, #-96]
0x534034f4   674  c2c0b211       gcseal x17, c16
0x534034f8   678  37000071       tbnz w17, #0, #+0xc (addr 0x53403504)
0x534034fc   67c  b2400211       orr x17, x16, #0x1
0x53403500   680  c2d14210       scvalue c16, c16, x17
0x53403504   684  37000050       tbnz w16, #0, #+0x8 (addr 0x5340350c)
0x53403508   688  d4200000       brk #0x0
0x5340350c   68c  c2c0b211       gcseal x17, c16
0x53403510   690  37000071       tbnz w17, #0, #+0xc (addr 0x5340351c)
0x53403514   694  b2400211       orr x17, x16, #0x1
0x53403518   698  c2d14210       scvalue c16, c16, x17
0x5340351c   69c  37000050       tbnz w16, #0, #+0x8 (addr 0x53403524)
0x53403520   6a0  d4200000       brk #0x0
0x53403524   6a4  c2c21200       br x16
0x53403528   6a8  97fffff2       bl #-0x38 (addr 0x534034f0)
0x5340352c   6ac  97fffff1       bl #-0x3c (addr 0x534034f0)
0x53403530   6b0  97fffff0       bl #-0x40 (addr 0x534034f0)
0x53403534   6b4  97ffffef       bl #-0x44 (addr 0x534034f0)
0x53403538   6b8  97ffffee       bl #-0x48 (addr 0x534034f0)
0x5340353c   6bc  97ffffed       bl #-0x4c (addr 0x534034f0)
0x53403540   6c0  97ffffec       bl #-0x50 (addr 0x534034f0)
0x53403544   6c4  97ffffeb       bl #-0x54 (addr 0x534034f0)

Inlined functions (count = 0)

Deoptimization Input Data (deopt points = 8)
 index  bytecode-offset  node-id    pc
     0                6       17   1d8
     1               13       23   21c
     2               19       28   25c
     3               26       34   2a0
     4               30       39   338
     5               36       44   3d4
     6               41       48   414
     7               -1        7   560

Safepoints (entries = 8, byte size = 56)
0x53403058    1d8  slots (sp->fp): 00100000  deopt      0 trampoline:    6a8
0x5340309c    21c  slots (sp->fp): 00100000  deopt      1 trampoline:    6ac
0x534030dc    25c  slots (sp->fp): 00100000  deopt      2 trampoline:    6b0
0x53403120    2a0  slots (sp->fp): 01100000  deopt      3 trampoline:    6b4
0x534031b8    338  slots (sp->fp): 01100000  deopt      4 trampoline:    6b8
0x53403254    3d4  slots (sp->fp): 00100000  deopt      5 trampoline:    6bc
0x53403294    414  slots (sp->fp): 00100000  deopt      6 trampoline:    6c0
0x534033e0    560  slots (sp->fp): 00100000  deopt      7 trampoline:    6c4

RelocInfo (size = 2d)
0x53402f14  off heap target
0x53402f6c  external reference (abort_with_reason)  (0x8b3c52d)
0x5340300c  full embedded object  (0x23ddbddc4b31 <FixedArray[1]>)
0x53403010  full embedded object  (0x23ddbddc4c81 <JSFunction (sfi = 0x23ddbddc4a31)>)
0x53403018  external reference (Runtime::DeclareGlobals)  (0x9c668c1)
0x53403024  off heap target
0x5340305c  full embedded object  (0x23ddbddc1d51 <String[5]: #deopt>)
0x53403068  off heap target
0x5340309c  full embedded object  (0x2c8f79046171 <String[1]: #f>)
0x534030a4  full embedded object  (0x0867828c0b51 <NativeContext[11a]>)
0x534030a8  off heap target
0x534030e0  full embedded object  (0x2c8f79045ed1 <String[1]: #X>)
0x53403184  off heap target
0x53403220  off heap target
0x53403254  full embedded object  (0x23ddbddc1d21 <String[6]: #result>)
0x534033a8  external reference (Runtime::StackGuardWithGap)  (0x9c00ea1)
0x534033e8  constant pool (size 108)

--- End code ---
#+end_src


*** DeoptimizationTest.DeoptimizeSimpleNested
- size not correct?
- ZapCodeObject: start_address + size_in_bytes does not contain the safepoint trampoline?

#+begin_src backtrace

Thread 1 hit Hardware watchpoint 8: *0x53402c90

Old value = 1396714545
New value = 195936478
(gdb) bt
#0  v8::internal::Heap::ZapCodeObject (this=0x520a3410 [rwRW,0x5208a000-0x520aa000],
    start_address=0x53402c80 [rwxRWE,0x53400000-0x63440000] , size_in_bytes=1472) at ../../src/heap/heap.cc:4497
#1  0x0000000008f93adc in v8::internal::HeapAllocator::AllocateRaw<(v8::internal::AllocationType)2> (
    this=0x520a3440 [rwRW,0x5208a000-0x520aa000], size_in_bytes=1472, origin=v8::internal::AllocationOrigin::kRuntime,
    alignment=v8::internal::kTaggedAligned) at ../../src/heap/heap-allocator-inl.h:138
#2  0x0000000008f92df8 in v8::internal::HeapAllocator::AllocateRaw (this=0x520a3440 [rwRW,0x5208a000-0x520aa000], size_in_bytes=1472,
    type=v8::internal::AllocationType::kCode, origin=v8::internal::AllocationOrigin::kRuntime, alignment=v8::internal::kTaggedAligned)
    at ../../src/heap/heap-allocator-inl.h:166
#3  0x0000000008fb25ec in v8::internal::HeapAllocator::AllocateRawWithLightRetrySlowPath (this=0x520a3440 [rwRW,0x5208a000-0x520aa000],
    size=1472, allocation=v8::internal::AllocationType::kCode, origin=v8::internal::AllocationOrigin::kRuntime,
    alignment=v8::internal::kTaggedAligned) at ../../src/heap/heap-allocator.cc:78
#4  0x0000000008f7b294 in v8::internal::HeapAllocator::AllocateRawWith<(v8::internal::HeapAllocator::AllocationRetryMode)0> (
    this=0x520a3440 [rwRW,0x5208a000-0x520aa000], size=1472, allocation=v8::internal::AllocationType::kCode,
    origin=v8::internal::AllocationOrigin::kRuntime, alignment=v8::internal::kTaggedAligned) at ../../src/heap/heap-allocator-inl.h:225
#5  0x0000000008f58834 in v8::internal::Factory::CodeBuilder::AllocateInstructionStream (
    this=0xfffffff7ac20 [rwRW,0xfffffff7ac20-0xfffffff7acd0], retry_allocation_or_fail=false) at ../../src/heap/factory.cc:289
#6  0x0000000008f58310 in v8::internal::Factory::CodeBuilder::NewInstructionStream (
    this=0xfffffff7ac20 [rwRW,0xfffffff7ac20-0xfffffff7acd0], retry_allocation_or_fail=false) at ../../src/heap/factory.cc:122
#7  0x0000000008f58fdc in v8::internal::Factory::CodeBuilder::BuildInternal (this=0xfffffff7ac20 [rwRW,0xfffffff7ac20-0xfffffff7acd0],
    retry_allocation_or_fail=false) at ../../src/heap/factory.cc:161
#8  0x0000000008f599bc in v8::internal::Factory::CodeBuilder::TryBuild (this=0xfffffff7ac20 [rwRW,0xfffffff7ac20-0xfffffff7acd0])
    at ../../src/heap/factory.cc:348
#9  0x000000000a175f64 in v8::internal::compiler::CodeGenerator::FinalizeCode (this=0x513f5000 [rwRW,0x513f5000-0x513f5c00])
    at ../../src/compiler/backend/code-generator.cc:516
#10 0x000000000a7a62c0 in v8::internal::compiler::FinalizeCodePhase::Run (this=0xfffffff7ae9f [rwRW,0xfffffff7ae9f-0xfffffff7aea0],
    data=0x635282d0 [rwRW,0x63528000-0x63528700], temp_zone=0x511fe700 [rwRW,0x511fe700-0x511fe770])
    at ../../src/compiler/pipeline.cc:2661
#11 0x000000000a7973f4 in v8::internal::compiler::PipelineImpl::Run<v8::internal::compiler::FinalizeCodePhase> (
    this=0x635286b0 [rwRW,0x63528000-0x63528700]) at ../../src/compiler/pipeline.cc:1365
#12 0x000000000a78a51c in v8::internal::compiler::PipelineImpl::FinalizeCode (this=0x635286b0 [rwRW,0x63528000-0x63528700],
    retire_broker=true) at ../../src/compiler/pipeline.cc:4061
#13 0x000000000a78a1ec in v8::internal::compiler::PipelineCompilationJob::FinalizeJobImpl (
    this=0x63528000 [rwRW,0x63528000-0x63528700], isolate=0x5208a000 [rwRW,0x5208a000-0x520aa000])
    at ../../src/compiler/pipeline.cc:1301
#14 0x0000000008af558c in v8::internal::OptimizedCompilationJob::FinalizeJob (this=0x63528000 [rwRW,0x63528000-0x63528700],
    isolate=0x5208a000 [rwRW,0x5208a000-0x520aa000]) at ../../src/codegen/compiler.cc:499
#15 0x0000000008b10674 in v8::internal::(anonymous namespace)::CompileTurbofan_NotConcurrent (
    isolate=0x5208a000 [rwRW,0x5208a000-0x520aa000], job=0x63528000 [rwRW,0x63528000-0x63528700]) at ../../src/codegen/compiler.cc:1055
--Type <RET> for more, q to quit, c to continue without paging--
#16 0x0000000008b0faac in v8::internal::(anonymous namespace)::CompileTurbofan (isolate=0x5208a000 [rwRW,0x5208a000-0x520aa000],
    function=..., shared=..., mode=v8::internal::ConcurrencyMode::kSynchronous, osr_offset=...,
    result_behavior=v8::internal::(anonymous namespace)::CompileResultBehavior::kDefault) at ../../src/codegen/compiler.cc:1178
#17 0x0000000008aff974 in v8::internal::(anonymous namespace)::GetOrCompileOptimized (isolate=0x5208a000 [rwRW,0x5208a000-0x520aa000],
    function=..., mode=v8::internal::ConcurrencyMode::kSynchronous, code_kind=v8::internal::CodeKind::TURBOFAN, osr_offset=...,
    result_behavior=v8::internal::(anonymous namespace)::CompileResultBehavior::kDefault) at ../../src/codegen/compiler.cc:1323
#18 0x0000000008afeeec in v8::internal::Compiler::Compile (isolate=0x5208a000 [rwRW,0x5208a000-0x520aa000], function=...,
    flag=v8::internal::Compiler::KEEP_EXCEPTION, is_compiled_scope=0xfffffff7cfc0 [rwRW,0xfffffff7cfc0-0xfffffff7cfe0])
    at ../../src/codegen/compiler.cc:2632
#19 0x0000000009bdeca0 in v8::internal::__RT_impl_Runtime_CompileLazy (args=..., isolate=0x5208a000 [rwRW,0x5208a000-0x520aa000])
    at ../../src/runtime/runtime-compiler.cc:64
#20 0x0000000009bde88c in v8::internal::Runtime_CompileLazy (args_length=1,
    args_object=0xfffffff7d2f0 [rwRW,0xffffbff80000-0xfffffff80000], isolate=0x5208a000 [rwRW,0x5208a000-0x520aa000])
    at ../../src/runtime/runtime-compiler.cc:45
#21 0x000000000b31ef98 in Builtins_CEntry_Return1_ArgvOnStack_NoBuiltinExit ()
#+end_src


*** v8/test/mjsunit/compiler/regress-1196185.js

- ~mark-compact~ !!!

#+begin_src backtrace
Thread 1 hit Breakpoint 3, 0x000000004bc42e14 in ?? ()
1: $c16 = () 0x6284d81 <Builtins_KeyedLoadICTrampoline_Megamorphic> [rxRE,0x108000-0x7528000] (sentry)
#+end_src

#+begin_src backtrace
#
# Fatal error in ../../src/codegen/arm64/assembler-arm64-inl.h, line 610
# Debug check failed: instr->IsBranchAndLink() || instr->IsUnconditionalBranch().
#
#
#
#FailureMessage Object: 0xfffffff74090
==== C stack trace ===============================

    0x6c4305d <v8::base::debug::StackTrace::StackTrace(void)+0x24> at /home/yayu/v8/out/cfi/d8
    0x6c3cd55 <v8::platform::(anonymous namespace)::PrintStackTrace(void)+0x100038> at /home/yayu/v8/out/cfi/d8
    0x6c130d5 <V8_Fatal(char const*, int, char const*, ...)+0x12c> at /home/yayu/v8/out/cfi/d8
    0x6c12c6d <_ZN2v84base17PrintCheckOperandIcEENSt3__19enable_ifIXaaaantsr3std11is_functionINS2_14remove_pointerIT_E4typeEEE5valuentsr3std7is_enumIS5_EE5valuesr19has_output_operatorIS5_NS0_18CheckMessageStreamEEE5valueENS2_12basic_stringIcNS2_11char_traitsIcEENS2_9allocatorIcEEEEE4typeES5_> at /home/yayu/v8/out/cfi/d8
    0x6c1323d <V8_Dcheck(char const*, int, char const*)+0x34> at /home/yayu/v8/out/cfi/d8
    0x3c14139 <v8::internal::Assembler::target_address_at(__uintcap_t, __uintcap_t)+0xa4> at /home/yayu/v8/out/cfi/d8
    0x3c148a5 <v8::internal::RelocInfo::target_object(v8::internal::PtrComprCageBase)+0x1b4> at /home/yayu/v8/out/cfi/d8
    0x3c1323d <v8::internal::RelocInfo::Verify(v8::internal::Isolate*)+0x138> at /home/yayu/v8/out/cfi/d8
    0x3d85d55 <v8::internal::InstructionStream::InstructionStreamVerify(v8::internal::Isolate*)+0x3f0> at /home/yayu/v8/out/cfi/d8
    0x3d820cd <v8::internal::HeapObject::HeapObjectVerify(v8::internal::Isolate*)+0x1d68> at /home/yayu/v8/out/cfi/d8
    0x3d801d1 <v8::internal::Object::ObjectVerify(v8::internal::Isolate*)+0xe0> at /home/yayu/v8/out/cfi/d8
    0x40543dd <v8::internal::HeapVerification::VerifyObject(v8::internal::HeapObject)+0x12c> at /home/yayu/v8/out/cfi/d8
    0x4222df5 <v8::internal::PagedSpaceBase::Verify(v8::internal::Isolate*, v8::internal::SpaceVerificationVisitor*) const+0x544> at /home/yayu/v8/out/cfi/d8
    0x4053f79 <v8::internal::HeapVerification::VerifySpace(v8::internal::BaseSpace*)+0xa0> at /home/yayu/v8/out/cfi/d8
    0x4053e19 <v8::internal::HeapVerification::Verify(void)+0x458> at /home/yayu/v8/out/cfi/d8
    0x405532d <v8::internal::HeapVerifier::VerifyHeap(v8::internal::Heap*)+0x170> at /home/yayu/v8/out/cfi/d8
    0x409f129 <v8::internal::HeapVerifier::VerifyHeapIfEnabled(v8::internal::Heap*)+0x30> at /home/yayu/v8/out/cfi/d8
    0x4078f21 <v8::internal::Heap::PerformGarbageCollection(v8::internal::GarbageCollector, v8::internal::GarbageCollectionReason, char const*)+0x5e4> at /home/yayu/v8/out/cfi/d8
    0x4075f41 <v8::internal::Heap::CollectGarbage(v8::internal::AllocationSpace, v8::internal::GarbageCollectionReason, v8::GCCallbackFlags)+0x61c> at /home/yayu/v8/out/cfi/d8
    0x4050465 <v8::internal::HeapAllocator::AllocateRawWithLightRetrySlowPath(int, v8::internal::AllocationType, v8::internal::AllocationOrigin, v8::internal::AllocationAlignment)+0x118> at /home/yayu/v8/out/cfi/d8
    0x4050619 <v8::internal::HeapAllocator::AllocateRawWithRetryOrFailSlowPath(int, v8::internal::AllocationType, v8::internal::AllocationOrigin, v8::internal::AllocationAlignment)+0xa8> at /home/yayu/v8/out/cfi/d8
    0x4014fcd <_ZN2v88internal13HeapAllocator15AllocateRawWithILNS1_19AllocationRetryModeE1EEENS0_10HeapObjectEiNS0_14AllocationTypeENS0_16AllocationOriginENS0_19AllocationAlignmentE+0x19c> at /home/yayu/v8/out/cfi/d8
    0x3ff3a1d <v8::internal::AllocationType v8::internal::Factory::AllocateRawWithAllocationSite(v8::internal::Handle<v8::internal::Map>(v8::internal::Factory::AllocateRawWithAllocationSite<v8::internal::AllocationSite>)+0x190> at /home/yayu/v8/out/cfi/d8
    0x3ff9559 <v8::internal::AllocationType v8::internal::Factory::NewJSObjectFromMap(v8::internal::Handle<v8::internal::Map>(v8::internal::Factory::NewJSObjectFromMap<v8::internal::AllocationSite>)+0x204> at /home/yayu/v8/out/cfi/d8
    0x46cfb85 <int v8::internal::Factory::NewFastOrSlowJSObjectFromMap(v8::internal::Handle<v8::internal::Map>(v8::internal::AllocationType, v8::internal::Factory::NewFastOrSlowJSObjectFromMap<v8::internal::AllocationSite>)+0x134> at /home/yayu/v8/out/cfi/d8
    0x4799c1d <v8::internal::JSObject::New<v8::internal::JSReceiver> v8::internal::JSObject::New(v8::internal::Handle<v8::internal::JSFunction>(v8::internal::JSObject::New<v8::internal::AllocationSite>)+0x370> at /home/yayu/v8/out/cfi/d8
    0x3e85e49 <v8::internal::Isolate*<v8::internal::Object> v8::internal::ErrorUtils::Construct(v8::internal::Isolate*, v8::internal::Handle<v8::internal::JSFunction>(v8::internal::Object, v8::internal::Object, v8::internal::FrameSkipMode, v8::internal::Object, v8::internal::ErrorUtils::StackTraceCollection)+0x430> at /home/yayu/v8/out/cfi/d8
    0x3e86a5d <v8::internal::Object v8::internal::MessageTemplate v8::internal::ErrorUtils::MakeGenericError(v8::internal::Isolate*, v8::internal::Handle<v8::internal::JSFunction>(v8::internal::Isolate*<v8::internal::Object>(v8::internal::Object, v8::internal::FrameSkipMode)+0x34c> at /home/yayu/v8/out/cfi/d8
    0x400175d <v8::internal::Object v8::internal::MessageTemplate v8::internal::Factory::NewError(v8::internal::Handle<v8::internal::JSFunction>(v8::internal::Factory::NewError<v8::internal::Object>(v8::internal::Object)+0x290> at /home/yayu/v8/out/cfi/d8
    0x4001d85 <v8::internal::Object v8::internal::Factory::NewReferenceError(v8::internal::MessageTemplate, v8::internal::Handle<v8::internal::Object>(v8::internal::Object)+0xf0> at /home/yayu/v8/out/cfi/d8
    0x42ccee9 <v8::internal::IC::ReferenceError(v8::internal::Handle<v8::internal::Name>)+0x158> at /home/yayu/v8/out/cfi/d8
    0x42ce1f9 <v8::internal::LoadIC::Load<v8::internal::Name> v8::internal::LoadIC::Load(v8::internal::Handle<v8::internal::Object>(bool, v8::internal::Object)+0xab8> at /home/yayu/v8/out/cfi/d8
    0x42cf5c1 <bool v8::internal::LoadGlobalIC::Load(v8::internal::Handle<v8::internal::Name>()+0x700> at /home/yayu/v8/out/cfi/d8
    0x42e1b61 <_ZN2v88internalL35__RT_impl_Runtime_LoadGlobalIC_MissENS0_9ArgumentsILNS0_13ArgumentsTypeE0EEEPNS0_7IsolateE+0x100564> at /home/yayu/v8/out/cfi/d8
#+end_src

** Print Error
./v8_unittests --random-seed=1974085256 --nohard-abort --verify-heap --enable-slow-asserts --testing-d8-test-runner --gtest_filter=DeoptimizationDisableConcurrentRecompilationTest.DeoptimizeBinaryOperationADD --gtest_random_seed=1974085256 --gtest_print_time=0 --print-opt-code --trace-turbo --print-builtin-code

