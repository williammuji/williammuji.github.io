---
layout: post
title:  "Async signal safe"
date: 2020-07-14 11:03:07 +0800
categories: jekyll update
---

1. async-signal-safe
2. boost stacktrace
3. redis stacktrace 
4. glog stacktrace
5. absl stacktrace
6. folly stacktrace
7. ELF
8. references

## 1.async-signal-safe

signal-safety - async-signal-safe functions

async-signal-safe函数要求在一个信号处理(signal handler)中能够安全的被调用。
大部分函数都不是async-signal-safe，一般来说，不可重入的函数都不是async-signal-safe。

举例:
一般对文件操作都是buffered I/O，stdio函数一般都会维护一个data buffer，还有计数器和索引，
用来记录数据大小和当前buffer内数据偏移。所以如果有个业务功能函数正在调用printf，此时来了个signal，
那么就会进入signal handler，假设这个signal handler也调用了printf，那么就会干扰了第一次printf的buffer，
引起数据错乱。假设对printf加锁，那么重入的话就会造成死锁。

有两个选择可以解决：
1. 在signal handler里只可以调用async-signal-safe函数，且对于全局变量保证signal handler是可重入的。
2. 如果执行unsafe函数或者操作全局数据时候，需要block signal delivery。
相对来说，第二个选择比较复杂，一般采用第一种选择。那么就需要保证async-signal-safe是可重入的，或是原子操作的。

POSIX.1规定了一些函数是async-signal-safe的(write/send/strncpy ...)

[signal-safety(7) — Linux manual page](https://man7.org/linux/man-pages/man7/signal-safety.7.html)

[Signal Handlers and Async-Signal Safety](https://docs.oracle.com/cd/E19455-01/806-5257/gen-26/index.html)


## 2.boost stacktrace

Writing a signal handler requires high attention! 

Only a few system calls allowed in signal handlers, so there's no cross platform way to print a stacktrace without a risk of deadlocking. 

The only way to deal with the problem - dump raw stacktrace into file/socket and parse it on program restart.

对于线上coredump，我们需要及时根据信息来调试追踪，重启进程来查看stacktrace不可接受的。 

boost使用low-level的async-signal-safe函数来dump堆栈，而且使用二进制序列化，所以可以通过指令
'od -tx8 -An stacktrace_dump_failename' 或者通过 boost::stacktrace::stacktrace::from_dump来显示堆栈。

[Handle terminates, aborts and Segmentation Faults](https://www.boost.org/doc/libs/1_65_1/doc/html/stacktrace/getting_started.html#stacktrace.getting_started.handle_terminates_aborts_and_seg)

```CPP
#include <signal.h>     // ::signal, ::raise
#include <boost/stacktrace.hpp>

/// safe_dump_to is a low-level async-signal-safe function for dumping call stacks. 
/// Dumps are binary serialized arrays of `void*`, so you could read them by using 'od -tx8 -An stacktrace_dump_failename' Linux command
/// or using boost::stacktrace::stacktrace::from_dump functions.

void my_signal_handler(int signum) {
    ::signal(signum, SIG_DFL);
    boost::stacktrace::safe_dump_to("./backtrace.dump");
    ::raise(SIGABRT);
}
```

```CPP
::signal(SIGSEGV, &my_signal_handler);
::signal(SIGABRT, &my_signal_handler);
```

```CPP
if (boost::filesystem::exists("./backtrace.dump")) {
    // there is a backtrace
    std::ifstream ifs("./backtrace.dump");

    boost::stacktrace::stacktrace st = boost::stacktrace::stacktrace::from_dump(ifs);
    std::cout << "Previous run crashed:\n" << st << std::endl;

    // cleaning up
    ifs.close();
    boost::filesystem::remove("./backtrace.dump");
}
```

```CPP
Previous run crashed:
 0# 0x00007F2EC0A6A8EF
 1# my_signal_handler(int) at ../example/terminate_handler.cpp:37
 2# 0x00007F2EBFD84CB0
 3# 0x00007F2EBFD84C37
 4# 0x00007F2EBFD88028
 5# 0x00007F2EC0395BBD
 6# 0x00007F2EC0393B96
 7# 0x00007F2EC0393BE1
 8# bar(int) at ../example/terminate_handler.cpp:18
 9# foo(int) at ../example/terminate_handler.cpp:22
10# bar(int) at ../example/terminate_handler.cpp:14
11# foo(int) at ../example/terminate_handler.cpp:22
12# main at ../example/terminate_handler.cpp:84
13# 0x00007F2EBFD6FF45
14# 0x0000000000402209
```


## 3.redis stacktrace

```CPP
void sigsegvHandler(int sig, siginfo_t *info, void *secret) {
	ucontext_t *uc = (ucontext_t*) secret;
	logStackTrace(uc);
}
```

```CPP
/* Logs the stack trace using the backtrace() call. This function is designed
 * to be called from signal handlers safely. */
void logStackTrace(ucontext_t *uc) {
    void *trace[101];
    int trace_size = 0, fd = openDirectLogFiledes();

    if (fd == -1) return; /* If we can't log there is anything to do. */

    /* Generate the stack trace */
    trace_size = backtrace(trace+1, 100);

    if (getMcontextEip(uc) != NULL) {
        char *msg1 = "EIP:\n";
        char *msg2 = "\nBacktrace:\n";
        if (write(fd,msg1,strlen(msg1)) == -1) {/* Avoid warning. */};
        trace[0] = getMcontextEip(uc);
        backtrace_symbols_fd(trace, 1, fd);
        if (write(fd,msg2,strlen(msg2)) == -1) {/* Avoid warning. */};
    }

    /* Write symbols to log file */
    backtrace_symbols_fd(trace+1, trace_size, fd);

    /* Cleanup */
    closeDirectLogFiledes(fd);
}
```

sigsegvHandler调用logStackTrace打印stacktrace，但是logStackTrace所调用的只有backtrace_symbols_fd是async-signal-safe
backtrace并不是async-signal-safe，所以redis的实现并不是async-signal-safe

Function: int backtrace (void \*\*buffer, int size)	线程安全 非异步信号安全

Preliminary: | MT-Safe | AS-Unsafe init heap dlopen plugin lock | AC-Unsafe init mem lock fd | See POSIX Safety Concepts. 

Function: void backtrace_symbols_fd (void *const *buffer, int size, int fd)	线程安全 异步信号安全

Preliminary: | MT-Safe | AS-Safe | AC-Unsafe lock | See POSIX Safety Concepts. 

[Backtraces](http://www.gnu.org/software/libc/manual/html_node/Backtraces.html)


## 4.glog stacktrace

src/signalhandler.cc

src/demangle.cc

src/symbolize.cc

src/stacktrace_x86_64-inl.h


通过libgcc调用_Unwind_Backtrace获得堆栈信息
然后通过src/symbolizes.cc获取program counter所对应的symbol names.

```CPP
// Produce stack trace using libgcc
// If you change this function, also change GetStackFrames below.
int GetStackTrace(void** result, int max_depth, int skip_count) {
  if (!ready_to_run)
    return 0;
  
  trace_arg_t targ;
  
  skip_count += 1;         // Do not include the "GetStackTrace" frame
  
  targ.result = result;
  targ.max_depth = max_depth;
  targ.skip_count = skip_count;
  targ.count = 0;

  _Unwind_Backtrace(GetOneFrame, &targ);
  
  return targ.count;
}

```

```CPP
void FailureSignalHandler(int signal_number,
                          siginfo_t *signal_info,
                          void *ucontext)
{
#ifdef HAVE_STACKTRACE
  // Get the stack traces.
  void *stack[32];
  // +1 to exclude this function.
  const int depth = GetStackTrace(stack, ARRAYSIZE(stack), 1);
# ifdef HAVE_SIGACTION
  DumpSignalInfo(signal_number, signal_info);
# endif
  // Dump the stack traces.
  for (int i = 0; i < depth; ++i) {
    DumpStackFrameInfo("    ", stack[i]);
  }
#endif

}

// The algorithm used in Symbolize() is as follows.
//
//   1. Go through a list of maps in /proc/self/maps and find the map
//   containing the program counter.
//
//   2. Open the mapped file and find a regular symbol table inside.
//   Iterate over symbols in the symbol table and look for the symbol
//   containing the program counter.  If such a symbol is found,
//   obtain the symbol name, and demangle the symbol if possible.
//   If the symbol isn't found in the regular symbol table (binary is
//   stripped), try the same thing with a dynamic symbol table.
//
// Note that Symbolize() is originally implemented to be used in
// FailureSignalHandler() in base/google.cc.  Hence it doesn't use
// malloc() and other unsafe operations.  It should be both
// thread-safe and async-signal-safe.
```

1 首先通过/proc/self/maps找到包含program counter的文件，

2 然后通过该文件的regular symbol，找到包含该program counter的symbol，如果找到了就通过src/demangle.cc demangle it.

3 如果没有找到就通过动态symbol表重复2操作

代码实现如下：

```CPP
// The implementation of our symbolization routine.  If it
// successfully finds the symbol containing "pc" and obtains the
// symbol name, returns true and write the symbol name to "out".
// Otherwise, returns false. If Callback function is installed via
// InstallSymbolizeCallback(), the function is also called in this function,
// and "out" is used as its output.
// To keep stack consumption low, we would like this function to not
// get inlined.
static ATTRIBUTE_NOINLINE bool SymbolizeAndDemangle(void *pc, char *out,
                                                    int out_size) {
    object_fd = OpenObjectFileContainingPcAndGetStartAddress(
        pc0, start_address, base_address, out + 1, out_size - 1);
}
```

```CPP
// Searches for the object file (from /proc/self/maps) that contains
// the specified pc.  If found, sets |start_address| to the start address
// of where this object file is mapped in memory, sets the module base
// address into |base_address|, copies the object file name into
// |out_file_name|, and attempts to open the object file.  If the object
// file is opened successfully, returns the file descriptor.  Otherwise,
// returns -1.  |out_file_name_size| is the size of the file name buffer
// (including the null-terminator).
static ATTRIBUTE_NOINLINE int OpenObjectFileContainingPcAndGetStartAddress(
    uint64_t pc, uint64_t &start_address, uint64_t &base_address,
    char *out_file_name, int out_file_name_size) {
  int object_fd;

  int maps_fd;
  NO_INTR(maps_fd = open("/proc/self/maps", O_RDONLY));
  FileDescriptor wrapped_maps_fd(maps_fd);
  if (wrapped_maps_fd.get() < 0) {
    return -1;
  }

  int mem_fd;
  NO_INTR(mem_fd = open("/proc/self/mem", O_RDONLY));
  FileDescriptor wrapped_mem_fd(mem_fd);
  if (wrapped_mem_fd.get() < 0) {
    return -1;
  }

  // Iterate over maps and look for the map containing the pc.  Then
  // look into the symbol tables inside.
  char buf[1024];  // Big enough for line of sane /proc/self/maps
  int num_maps = 0;
  LineReader reader(wrapped_maps_fd.get(), buf, sizeof(buf), 0);
  while (true) {
    num_maps++;
    const char *cursor;
    const char *eol;
    if (!reader.ReadLine(&cursor, &eol)) {  // EOF or malformed line.
      return -1;
    }

    // Start parsing line in /proc/self/maps.  Here is an example:
    //
    // 08048000-0804c000 r-xp 00000000 08:01 2142121    /bin/cat
    //
    // We want start address (08048000), end address (0804c000), flags
    // (r-xp) and file name (/bin/cat).

    // Read start address.
    cursor = GetHex(cursor, eol, &start_address);
    if (cursor == eol || *cursor != '-') {
      return -1;  // Malformed line.
    }
    ++cursor;  // Skip '-'.

    // Read end address.
    uint64_t end_address;
    cursor = GetHex(cursor, eol, &end_address);
    if (cursor == eol || *cursor != ' ') {
      return -1;  // Malformed line.
    }
    ++cursor;  // Skip ' '.

    // Read flags.  Skip flags until we encounter a space or eol.
    const char *const flags_start = cursor;
    while (cursor < eol && *cursor != ' ') {
      ++cursor;
    }
    // We expect at least four letters for flags (ex. "r-xp").
    if (cursor == eol || cursor < flags_start + 4) {
      return -1;  // Malformed line.
    }

    // Determine the base address by reading ELF headers in process memory.
    ElfW(Ehdr) ehdr;
    // Skip non-readable maps.
    if (flags_start[0] == 'r' &&
        ReadFromOffsetExact(mem_fd, &ehdr, sizeof(ElfW(Ehdr)), start_address) &&
        memcmp(ehdr.e_ident, ELFMAG, SELFMAG) == 0) {
      switch (ehdr.e_type) {
        case ET_EXEC:
          base_address = 0;
          break;
        case ET_DYN:
          // Find the segment containing file offset 0. This will correspond
          // to the ELF header that we just read. Normally this will have
          // virtual address 0, but this is not guaranteed. We must subtract
          // the virtual address from the address where the ELF header was
          // mapped to get the base address.
          //
          // If we fail to find a segment for file offset 0, use the address
          // of the ELF header as the base address.
          base_address = start_address;
          for (unsigned i = 0; i != ehdr.e_phnum; ++i) {
            ElfW(Phdr) phdr;
            if (ReadFromOffsetExact(
                    mem_fd, &phdr, sizeof(phdr),
                    start_address + ehdr.e_phoff + i * sizeof(phdr)) &&
                phdr.p_type == PT_LOAD && phdr.p_offset == 0) {
              base_address = start_address - phdr.p_vaddr;
              break;
            }
          }
          break;
        default:
          // ET_REL or ET_CORE. These aren't directly executable, so they don't
          // affect the base address.
          break;
      }
    }

    // Check start and end addresses.
    if (!(start_address <= pc && pc < end_address)) {
      continue;  // We skip this map.  PC isn't in this map.
    }

    // Check flags.  We are only interested in "r*x" maps.
    if (flags_start[0] != 'r' || flags_start[2] != 'x') {
      continue;  // We skip this map.
    }
    ++cursor;  // Skip ' '.

    // Read file offset.
    uint64_t file_offset;
    cursor = GetHex(cursor, eol, &file_offset);
    if (cursor == eol || *cursor != ' ') {
      return -1;  // Malformed line.
    }
    ++cursor;  // Skip ' '.

    // Skip to file name.  "cursor" now points to dev.  We need to
    // skip at least two spaces for dev and inode.
    int num_spaces = 0;
    while (cursor < eol) {
      if (*cursor == ' ') {
        ++num_spaces;
      } else if (num_spaces >= 2) {
        // The first non-space character after skipping two spaces
        // is the beginning of the file name.
        break;
      }
      ++cursor;
    }
    if (cursor == eol) {
      return -1;  // Malformed line.
    }

    // Finally, "cursor" now points to file name of our interest.
    NO_INTR(object_fd = open(cursor, O_RDONLY));
    if (object_fd < 0) {
      // Failed to open object file.  Copy the object file name to
      // |out_file_name|.
      strncpy(out_file_name, cursor, out_file_name_size);
      // Making sure |out_file_name| is always null-terminated.
      out_file_name[out_file_name_size - 1] = '\0';
      return -1;
    }
    return object_fd;
  }
}
```

glog的实现应该是确保了async-signal-safe

另外还实现了一个MinimalFormatter

// The class is used for formatting error messages.  We don\'t use printf() 

// as it\'s not async signal safe.

class MinimalFormatter


## 5.absl stacktrace

absl/debugging/failure_signal_handler.h

absl/debugging/stacktrace.h

absl/debugging/symbolize.h

```CPP
static void AbslFailureSignalHandler(int signo, siginfo_t*, void* ucontext) {
  // First write to stderr.
  WriteFailureInfo(signo, ucontext, WriteToStderr);

  // Riskier code (because it is less likely to be async-signal-safe)
  // goes after this point.
  if (fsh_options.writerfn != nullptr) {
    WriteFailureInfo(signo, ucontext, fsh_options.writerfn);
  }

}
```

```CPP
// Called by AbslFailureSignalHandler() to write the failure info. It is
// called once with writerfn set to WriteToStderr() and then possibly
// with writerfn set to the user provided function.
static void WriteFailureInfo(int signo, void* ucontext,
                             void (*writerfn)(const char*)) {
  WriterFnStruct writerfn_struct{writerfn};
  WriteSignalMessage(signo, writerfn);
  WriteStackTrace(ucontext, fsh_options.symbolize_stacktrace, WriterFnWrapper,
                  &writerfn_struct);
}

static void WriteSignalMessage(int signo, void (*writerfn)(const char*)) {
  char buf[64];
  const char* const signal_string =
      debugging_internal::FailureSignalToString(signo);
  if (signal_string != nullptr && signal_string[0] != '\0') {
    snprintf(buf, sizeof(buf), "*** %s received at time=%ld ***\n",
             signal_string,
             static_cast<long>(time(nullptr)));  // NOLINT(runtime/int)
  } else {
    snprintf(buf, sizeof(buf), "*** Signal %d received at time=%ld ***\n",
             signo, static_cast<long>(time(nullptr)));  // NOLINT(runtime/int)
  }
  writerfn(buf);
}

```

```CPP
// Convenient wrapper around DumpPCAndFrameSizesAndStackTrace() for signal
// handlers. "noinline" so that GetStackFrames() skips the top-most stack
// frame for this function.
ABSL_ATTRIBUTE_NOINLINE static void WriteStackTrace(
    void* ucontext, bool symbolize_stacktrace,
    void (*writerfn)(const char*, void*), void* writerfn_arg) {
  constexpr int kNumStackFrames = 32;
  void* stack[kNumStackFrames];
  int frame_sizes[kNumStackFrames];
  int min_dropped_frames;
  int depth = absl::GetStackFramesWithContext(
      stack, frame_sizes, kNumStackFrames,
      1,  // Do not include this function in stack trace.
      ucontext, &min_dropped_frames);
  absl::debugging_internal::DumpPCAndFrameSizesAndStackTrace(
      absl::debugging_internal::GetProgramCounter(ucontext), stack, frame_sizes,
      depth, min_dropped_frames, symbolize_stacktrace, writerfn, writerfn_arg);
}

void DumpPCAndFrameSizesAndStackTrace(
    void* pc, void* const stack[], int frame_sizes[], int depth,
    int min_dropped_frames, bool symbolize_stacktrace,
    void (*writerfn)(const char*, void*), void* writerfn_arg) {
  if (pc != nullptr) {
    // We don't know the stack frame size for PC, use 0.
    if (symbolize_stacktrace) {
      DumpPCAndFrameSizeAndSymbol(writerfn, writerfn_arg, pc, pc, 0, "PC: ");
    } else {
      DumpPCAndFrameSize(writerfn, writerfn_arg, pc, 0, "PC: ");
    }
  }
  for (int i = 0; i < depth; i++) {
    if (symbolize_stacktrace) {
      // Pass the previous address of pc as the symbol address because pc is a
      // return address, and an overrun may occur when the function ends with a
      // call to a function annotated noreturn (e.g. CHECK). Note that we don't
      // do this for pc above, as the adjustment is only correct for return
      // addresses.
      DumpPCAndFrameSizeAndSymbol(writerfn, writerfn_arg, stack[i],
                                  reinterpret_cast<char*>(stack[i]) - 1,
                                  frame_sizes[i], "    ");
    } else {
      DumpPCAndFrameSize(writerfn, writerfn_arg, stack[i], frame_sizes[i],
                         "    ");
    }
  }
  if (min_dropped_frames > 0) {
    char buf[100];
    snprintf(buf, sizeof(buf), "    @ ... and at least %d more frames\n",
             min_dropped_frames);
    writerfn(buf, writerfn_arg);
  }
}

// Print a program counter, its stack frame size, and its symbol name.
// Note that there is a separate symbolize_pc argument. Return addresses may be
// at the end of the function, and this allows the caller to back up from pc if
// appropriate.
static void DumpPCAndFrameSizeAndSymbol(void (*writerfn)(const char*, void*),
                                        void* writerfn_arg, void* pc,
                                        void* symbolize_pc, int framesize,
                                        const char* const prefix) {
  char tmp[1024];
  const char* symbol = "(unknown)";
  if (absl::Symbolize(symbolize_pc, tmp, sizeof(tmp))) {
    symbol = tmp;
  }
  char buf[1024];
  if (framesize <= 0) {
    snprintf(buf, sizeof(buf), "%s@ %*p  (unknown)  %s\n", prefix,
             kPrintfPointerFieldWidth, pc, symbol);
  } else {
    snprintf(buf, sizeof(buf), "%s@ %*p  %9d  %s\n", prefix,
             kPrintfPointerFieldWidth, pc, framesize, symbol);
  }
  writerfn(buf, writerfn_arg);
}

bool Symbolize(const void *pc, char *out, int out_size) {
  // Symbolization is very slow under tsan.
  ABSL_ANNOTATE_IGNORE_READS_AND_WRITES_BEGIN();
  SAFE_ASSERT(out_size >= 0);
  debugging_internal::Symbolizer *s = debugging_internal::AllocateSymbolizer();
  const char *name = s->GetSymbol(pc);
  bool ok = false;
  if (name != nullptr && out_size > 0) {
    strncpy(out, name, out_size);
    ok = true;
    if (out[out_size - 1] != '\0') {
      // strncpy() does not '\0' terminate when it truncates.  Do so, with
      // trailing ellipsis.
      static constexpr char kEllipsis[] = "...";
      int ellipsis_size =
          std::min(implicit_cast<int>(strlen(kEllipsis)), out_size - 1);
      memcpy(out + out_size - ellipsis_size - 1, kEllipsis, ellipsis_size);
      out[out_size - 1] = '\0';
    }
  }
  debugging_internal::FreeSymbolizer(s);
  ABSL_ANNOTATE_IGNORE_READS_AND_WRITES_END();
  return ok;
}


// The implementation of our symbolization routine.  If it
// successfully finds the symbol containing "pc" and obtains the
// symbol name, returns pointer to that symbol. Otherwise, returns nullptr.
// If any symbol decorators have been installed via InstallSymbolDecorator(),
// they are called here as well.
// To keep stack consumption low, we would like this function to not
// get inlined.
const char *Symbolizer::GetSymbol(const void *const pc) {
  const char *entry = FindSymbolInCache(pc);
  if (entry != nullptr) {
    return entry;
  }
  symbol_buf_[0] = '\0';

  ObjFile *const obj = FindObjFile(pc, 1);
  ptrdiff_t relocation = 0;
  int fd = -1;
  if (obj != nullptr) {
    if (MaybeInitializeObjFile(obj)) {
      if (obj->elf_type == ET_DYN &&
          reinterpret_cast<uint64_t>(obj->start_addr) >= obj->offset) {
        // This object was relocated.
        //
        // For obj->offset > 0, adjust the relocation since a mapping at offset
        // X in the file will have a start address of [true relocation]+X.
        relocation = reinterpret_cast<ptrdiff_t>(obj->start_addr) - obj->offset;
      }

      fd = obj->fd;
    }
    if (GetSymbolFromObjectFile(*obj, pc, relocation, symbol_buf_,
                                sizeof(symbol_buf_), tmp_buf_,
                                sizeof(tmp_buf_)) == SYMBOL_FOUND) {
      // Only try to demangle the symbol name if it fit into symbol_buf_.
      DemangleInplace(symbol_buf_, sizeof(symbol_buf_), tmp_buf_,
                      sizeof(tmp_buf_));
    }
  } else {
#if ABSL_HAVE_VDSO_SUPPORT
    VDSOSupport vdso;
    if (vdso.IsPresent()) {
      VDSOSupport::SymbolInfo symbol_info;
      if (vdso.LookupSymbolByAddress(pc, &symbol_info)) {
        // All VDSO symbols are known to be short.
        size_t len = strlen(symbol_info.name);
        ABSL_RAW_CHECK(len + 1 < sizeof(symbol_buf_),
                       "VDSO symbol unexpectedly long");
        memcpy(symbol_buf_, symbol_info.name, len + 1);
      }
    }
#endif
  }

  if (g_decorators_mu.TryLock()) {
    if (g_num_decorators > 0) {
      SymbolDecoratorArgs decorator_args = {
          pc,       relocation,       fd,     symbol_buf_, sizeof(symbol_buf_),
          tmp_buf_, sizeof(tmp_buf_), nullptr};
      for (int i = 0; i < g_num_decorators; ++i) {
        decorator_args.arg = g_decorators[i].arg;
        g_decorators[i].fn(&decorator_args);
      }
    }
    g_decorators_mu.Unlock();
  }
  if (symbol_buf_[0] == '\0') {
    return nullptr;
  }
  symbol_buf_[sizeof(symbol_buf_) - 1] = '\0';  // Paranoia.
  return InsertSymbolInCache(pc, symbol_buf_);
}

```

可疑的是WriteSignalMessage调用了snprintf，所以abseil实现应该不是async-signal-safe。

## 6.folly

folly/experimental/symbolizer/SignalHandler.cpp

folly/experimental/symbolizer/Symbolizer.cpp

folly/experimental/symbolizer/StackTrace.cpp

```CPP
// Here be dragons.
void innerSignalHandler(int signum, siginfo_t* info, void* /* uctx */) {
  // First, let's only let one thread in here at a time.
  pthread_t myId = pthread_self();

  pthread_t prevSignalThread = kInvalidThreadId;
  while (!gSignalThread.compare_exchange_strong(prevSignalThread, myId)) {
    if (pthread_equal(prevSignalThread, myId)) {
      // First time here. Try to dump the stack trace without symbolization.
      // If we still fail, well, we're mightily screwed, so we do nothing the
      // next time around.
      if (!gInRecursiveSignalHandler.exchange(true)) {
        print("Entered fatal signal handler recursively. We're in trouble.\n");
        gStackTracePrinter->printStackTrace(false); // no symbolization
      }
      return;
    }

    // Wait a while, try again.
    timespec ts;
    ts.tv_sec = 0;
    ts.tv_nsec = 100L * 1000 * 1000; // 100ms
    nanosleep(&ts, nullptr);

    prevSignalThread = kInvalidThreadId;
  }

  dumpTimeInfo();
  dumpSignalInfo(signum, info);
  gStackTracePrinter->printStackTrace(true); // with symbolization

  // Run user callbacks
  gFatalSignalCallbackRegistry->run();
}

```

```CPP
// Note: not thread-safe, but that's okay, as we only let one thread
// in our signal handler at a time.
//
// Leak it so we don't have to worry about destruction order
SafeStackTracePrinter* gStackTracePrinter = new SafeStackTracePrinter();

void SafeStackTracePrinter::printStackTrace(bool symbolize) {
  SCOPE_EXIT {
    flush();
  };

  // Skip the getStackTrace frame
  if (!getStackTraceSafe(*addresses_)) {
    print("(error retrieving stack trace)\n");
  } else if (symbolize) {
    // Do our best to populate location info, process is going to terminate,
    // so performance isn't critical.
    Symbolizer symbolizer(&elfCache_, Dwarf::LocationInfoMode::FULL);
    symbolizer.symbolize(*addresses_);

    // Skip the top 2 frames:
    // getStackTraceSafe
    // SafeStackTracePrinter::printStackTrace (here)
    //
    // Leaving signalHandler on the stack for clarity, I think.
    printer_.println(*addresses_, 2);
  } else {
    print("(safe mode, symbolizer not available)\n");
    AddressFormatter formatter;
    for (size_t i = 0; i < addresses_->frameCount; ++i) {
      print(formatter.format(addresses_->addresses[i]));
      print("\n");
    }
  }
}

void Symbolizer::symbolize(
    const uintptr_t* addresses,
    SymbolizedFrame* frames,
    size_t addrCount) {
  size_t remaining = 0;
  for (size_t i = 0; i < addrCount; ++i) {
    auto& frame = frames[i];
    if (!frame.found) {
      ++remaining;
      frame.clear();
    }
  }

  if (remaining == 0) { // we're done
    return;
  }

  if (_r_debug.r_version != 1) {
    return;
  }

  char selfPath[PATH_MAX + 8];
  ssize_t selfSize;
  if ((selfSize = readlink("/proc/self/exe", selfPath, PATH_MAX + 1)) == -1) {
    // Something has gone terribly wrong.
    return;
  }
  selfPath[selfSize] = '\0';

  for (auto lmap = _r_debug.r_map; lmap != nullptr && remaining != 0;
       lmap = lmap->l_next) {
    // The empty string is used in place of the filename for the link_map
    // corresponding to the running executable.  Additionally, the `l_addr' is
    // 0 and the link_map appears to be first in the list---but none of this
    // behavior appears to be documented, so checking for the empty string is
    // as good as anything.
    auto const objPath = lmap->l_name[0] != '\0' ? lmap->l_name : selfPath;

    auto const elfFile = cache_->getFile(objPath);
    if (!elfFile) {
      continue;
    }

    // Get the address at which the object is loaded.  We have to use the ELF
    // header for the running executable, since its `l_addr' is zero, but we
    // should use `l_addr' for everything else---in particular, if the object
    // is position-independent, getBaseAddress() (which is p_vaddr) will be 0.
    auto const base =
        lmap->l_addr != 0 ? lmap->l_addr : elfFile->getBaseAddress();

    for (size_t i = 0; i < addrCount && remaining != 0; ++i) {
      auto& frame = frames[i];
      if (frame.found) {
        continue;
      }

      auto const addr = addresses[i];
      if (symbolCache_) {
        // Need a write lock, because EvictingCacheMap brings found item to
        // front of eviction list.
        auto lockedSymbolCache = symbolCache_->wlock();

        auto const iter = lockedSymbolCache->find(addr);
        if (iter != lockedSymbolCache->end()) {
          frame = iter->second;
          continue;
        }
      }

      // Get the unrelocated, ELF-relative address.
      auto const adjusted = addr - base;

      if (elfFile->getSectionContainingAddress(adjusted)) {
        frame.set(elfFile, adjusted, mode_);
        --remaining;
        if (symbolCache_) {
          // frame may already have been set here.  That's ok, we'll just
          // overwrite, which doesn't cause a correctness problem.
          symbolCache_->wlock()->set(addr, frame);
        }
      }
    }
  }
}


```

## 7.ELF(Executable and Linkable Format)
可执行与可链接格式 （英语：Executable and Linkable Format，缩写为ELF），常被称为ELF格式，在计算机科学中，是一种用于可执行文件、目标文件、共享库和核心转储的标准文件格式。 

- DESCRIPTION
>The  header  file <elf.h> defines the format of ELF executable binary files.  Amongst these files are normal executable files, relocatable object files, core files, and
>shared objects.
>
>An executable file using the ELF file format consists of an ELF header, followed by a program header table or a section header table, or both.  The ELF header is always
>at offset zero of the file.  The program header table and the section header table's offset in the file are defined in the ELF header.  The two tables describe the rest
>of the particularities of the file.
>
>This header file describes the above mentioned headers as C structures and also includes structures for dynamic sections, relocation sections and symbol tables.

- Program header (Phdr)
>An  executable  or  shared  object file's program header table is an array of structures, each describing a segment or other information the system needs to prepare the
>program for execution.  An object file segment contains one or more sections.  Program headers are meaningful only for executable and shared object files.  A file spec‐
>ifies  its  own  program  header  size  with the ELF header's e_phentsize and e_phnum members.  The ELF program header is described by the type Elf32_Phdr or Elf64_Phdr
>depending on the architecture:

- Section header (Shdr)
>A file's section header table lets one locate all the file's sections.  The section header table is an array of Elf32_Shdr or Elf64_Shdr structures.  The  ELF  header's
>e_shoff member gives the byte offset from the beginning of the file to the section header table.  e_shnum holds the number of entries the section header table contains.
>e_shentsize holds the size in bytes of each entry.

- struct r_debug
>//usr/include/link.h
>
>Data structure for communication from the run-time dynamic linker for loaded ELF shared objects.

## 8.references

[Symbol Table Section](https://docs.oracle.com/cd/E23824_01/html/819-0690/chapter6-79797.html)

[ELF_Format](http://www.skyfree.org/linux/references/ELF_Format.pdf)

[Object File Format](https://docs.oracle.com/cd/E23824_01/html/819-0690/chapter6-46512.html#scrolltoc)

[Load-time relocation of shared libraries](https://eli.thegreenplace.net/2011/08/25/load-time-relocation-of-shared-libraries/)
