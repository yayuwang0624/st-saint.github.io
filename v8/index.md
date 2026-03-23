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


* Safepoints

- depot

* Pipeline
** Turbofan
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
// ->
CompilationJob::Status OptimizedCompilationJob::ExecuteJob(
    RuntimeCallStats* stats, LocalIsolate* local_isolate);
// ->
PipelineCompilationJob::Status PipelineCompilationJob::ExecuteJobImpl(
    RuntimeCallStats* stats, LocalIsolate* local_isolate);
// +
PipelineCompilationJob::Status PipelineCompilationJob::FinalizeJobImpl(
    Isolate* isolate);
// ->
MaybeHandle<Code> Factory::CodeBuilder::BuildInternal(
    bool retry_allocation_or_fail);
// ->
Handle<Code> Factory::CodeBuilder::NewCode(const NewCodeOptions& options);
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


* Compile
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

