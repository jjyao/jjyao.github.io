---
layout: post
title: "JIT in Action"
date: 2021-01-16 22:23:27 -0800
comments: true
categories:
---

[JIT](https://en.wikipedia.org/wiki/Just-in-time_compilation) is a way of executing computer code that involves compilation during execution of a program at **runtime**. This post shows how to execute [Brainfuck](https://en.wikipedia.org/wiki/Brainfuck) programs using JIT in various ways.

1. [Interpreter](#interpreter)
2. [Transpilation](#transpilation)
3. [LLVM](#llvm)
4. [DynASM](#dynasm)
<!-- more -->

## <a id="interpreter"></a>Interpreter
Before showing how to use JIT to run a Brainfuck program, lets see how it can be run by an interpreter.

``` cpp
#include <cstdio>
#include <cstdlib>
#include <string>
#include <vector>
#include <fstream>
#include <streambuf>

class BrainfuckInterpreter {
  public:
    BrainfuckInterpreter(const std::string& program)
      : program(program) {}

    void run();

  private:
    const std::string program;
};

void BrainfuckInterpreter::run() {
  char data[30000] = {0};
  char* ptr = data;
  std::vector<size_t> stack;
  size_t skips = 0;

  for (size_t i = 0; i < program.size(); ++i) {
    switch(program[i]) {
      case '>':
        if (!skips) {
          ++ptr;
        }
        break;
      case '<':
        if (!skips) {
          --ptr;
        }
        break;
      case '+':
        if (!skips) {
          ++*ptr;
        }
        break;
      case '-':
        if (!skips) {
          --*ptr;
        }
        break;
      case '.':
        if (!skips) {
          putchar(static_cast<int>(*ptr));
        }
        break;
      case ',':
        if (!skips) {
          *ptr = static_cast<char>(getchar());
        }
        break;
      case '[':
        stack.emplace_back(i);
        if (!*ptr) {
          ++skips;
        }
        break;
      case ']':
        if (*ptr) {
          i = *(stack.end() - 1);
        } else {
          stack.pop_back();
        }
        if (skips) {
          --skips;
        }
        break;
    }
  }
}

int main(int argc, char** argv) {
  std::ifstream ifs(argv[1]);
  std::string program((std::istreambuf_iterator<char>(ifs)), (std::istreambuf_iterator<char>()));
  BrainfuckInterpreter interpreter(program);
  interpreter.run();
  return 0;
}
```

```
g++ -std=c++11 -o brainfuck brainfuck.cc

./brainfuck program.bf
```

For a complicated Brainfuck program like [mandelbrot.bf](https://www.google.com/search?q=mandelbrot.bf), running in an interpreter will be slow.

## <a id="transpilation"></a>Transpilation
One way to generate and run native code for a Brainfuck program is transpilaiton. The first step is generating the equivalent C/C++ code for the Brainfuck program and then using a conventional compiler to generate a shared object. The second step is using `dlopen()` to load the shared object and run it.

``` cpp
#include <dlfcn.h>
#include <cstdio>
#include <cstdlib>
#include <string>
#include <vector>
#include <fstream>
#include <iostream>
#include <streambuf>

class BrainfuckTranspiler {
  public:
    BrainfuckTranspiler(const std::string& program)
      : program(program) {}

    void run();

  private:
    const std::string program;
};

void BrainfuckTranspiler::run() {
  std::string ccname = std::tmpnam(nullptr);
  ccname += ".cc";
  std::ofstream ofs;
  ofs.open(ccname);

  ofs << R"(
#include <cstdio>
#include <vector>

extern "C" void _run() {
  char data[30000] = {0};
  char* ptr = data;
  )";

  for (size_t i = 0; i < program.size(); ++i) {
    switch(program[i]) {
      case '>':
        ofs << "++ptr;" << std::endl;
        break;
      case '<':
        ofs << "--ptr;" << std::endl;
        break;
      case '+':
        ofs << "++*ptr;" << std::endl;
        break;
      case '-':
        ofs << "--*ptr;" << std::endl;
        break;
      case '.':
        ofs << "putchar(static_cast<int>(*ptr));" << std::endl;
        break;
      case ',':
        ofs << "*ptr = static_cast<char>(getchar());" << std::endl;
        break;
      case '[':
        ofs << "while (*ptr) {" << std::endl;
        break;
      case ']':
        ofs << "}" << std::endl;
        break;
    }
  }
  ofs << "}";
  ofs.close();

  std::string soname = std::tmpnam(nullptr);
  soname += ".so";
  std::string compile = "g++ " + ccname + " -o " + soname + " -shared -fPIC";
  system(compile.c_str());

  void* handle = dlopen(soname.c_str(), RTLD_NOW);
  if (!handle) {
    std::cerr << dlerror() << std::endl;
  }
  dlerror();
  void (*_run)();
  _run = (void (*)()) dlsym(handle, "_run");
  char* error = dlerror();
  if (error) {
    std::cerr << error << std::endl;
  }
  (*_run)();
  dlclose(handle);

  remove(ccname.c_str());
  remove(soname.c_str());
}

int main(int argc, char** argv) {
  std::ifstream ifs(argv[1]);
  std::string program((std::istreambuf_iterator<char>(ifs)), (std::istreambuf_iterator<char>()));
  BrainfuckTranspiler transpiler(program);
  transpiler.run();
  return 0;
}
```

```
g++ -std=c++11 -ldl -o brainfuck brainfuck.cc

./brainfuck program.bf
```

## <a id="llvm"></a>LLVM
Besides using an external compiler process to generate the native code, we can also use LLVM mcjit by first generating the equivalent LLVM IR for the Brainfuck program. We can generate the IR manually or using clang frontend. Here I'm using clang to generate the IR but LLVM has an [example](https://github.com/llvm/llvm-project/tree/main/llvm/examples/BrainF) of manually generating it. The LLVM version I'm using is 7.0.1.

``` cpp
#include <cstdlib>
#include <string>
#include <vector>
#include <sstream>
#include <fstream>
#include <iostream>
#include <streambuf>
#include <clang/Driver/Compilation.h>
#include <clang/Driver/Driver.h>
#include <clang/CodeGen/CodeGenAction.h>
#include <clang/Frontend/FrontendOptions.h>
#include <clang/Frontend/CompilerInstance.h>
#include <clang/Frontend/CompilerInvocation.h>
#include <clang/Frontend/TextDiagnosticPrinter.h>
#include <llvm/IR/Module.h>
#include <llvm/ADT/StringRef.h>
#include <llvm/IR/LLVMContext.h>
#include <llvm/Support/Host.h>
#include <llvm/Support/TargetSelect.h>
#include <llvm/ExecutionEngine/ExecutionEngine.h>
#include <llvm/ExecutionEngine/GenericValue.h>

class BrainfuckLLVM {
  public:
    BrainfuckLLVM(const std::string& program, const std::string& clangexe)
      : program(program), clangexe(clangexe) {}

    void run();

  private:
    const std::string program;
    const std::string clangexe;
};

void BrainfuckLLVM::run() {
  std::string ccname = std::tmpnam(nullptr);
  ccname += ".cc";
  std::ofstream ofs;
  ofs.open(ccname);

  ofs << R"(
#include <cstdio>
#include <vector>

extern "C" void _run() {
  char data[30000] = {0};
  char* ptr = data;
  )";

  for (size_t i = 0; i < program.size(); ++i) {
    switch(program[i]) {
      case '>':
        ofs << "++ptr;" << std::endl;
        break;
      case '<':
        ofs << "--ptr;" << std::endl;
        break;
      case '+':
        ofs << "++*ptr;" << std::endl;
        break;
      case '-':
        ofs << "--*ptr;" << std::endl;
        break;
      case '.':
        ofs << "putchar(static_cast<int>(*ptr));" << std::endl;
        break;
      case ',':
        ofs << "*ptr = static_cast<char>(getchar());" << std::endl;
        break;
      case '[':
        ofs << "while (*ptr) {" << std::endl;
        break;
      case ']':
        ofs << "}" << std::endl;
        break;
    }
  }
  ofs << "}";
  ofs.close();

  // Generate LLVM IR using clang frontend
  clang::IntrusiveRefCntPtr<clang::DiagnosticOptions> dopts = new clang::DiagnosticOptions();
  clang::TextDiagnosticPrinter* tdp = new clang::TextDiagnosticPrinter(llvm::errs(), &*dopts);
  clang::IntrusiveRefCntPtr<clang::DiagnosticIDs> dids(new clang::DiagnosticIDs());
  clang::DiagnosticsEngine dengine(dids, &*dopts, tdp);

  llvm::Triple triple(llvm::sys::getProcessTriple());
  clang::driver::Driver driver(clangexe, triple.str(), dengine);
  driver.setTitle("clang");
  driver.setCheckInputsExist(false);

  llvm::SmallVector<const char*, 16> args;
  args.push_back("clang");
  args.push_back(ccname.c_str());
  args.push_back("-fsyntax-only");
  std::unique_ptr<clang::driver::Compilation> compilation(driver.BuildCompilation(args));
  const clang::driver::JobList& jobs = compilation->getJobs();
  const clang::driver::Command& command = static_cast<clang::driver::Command>(*jobs.begin());
  const clang::driver::ArgStringList& ccargs = command.getArguments();
  std::unique_ptr<clang::CompilerInvocation> cinvocation(new clang::CompilerInvocation());
  clang::CompilerInvocation::CreateFromArgs(*cinvocation, const_cast<const char**>(ccargs.data()),
      const_cast<const char**>(ccargs.data()) + ccargs.size(), dengine);
  clang::CompilerInstance cinstance;
  cinstance.setInvocation(std::move(cinvocation));
  cinstance.createDiagnostics();

  llvm::LLVMContext context;
  clang::EmitLLVMOnlyAction action(&context);
  cinstance.ExecuteAction(action);
  std::unique_ptr<llvm::Module> module = action.takeModule();

  // Use mcjit to generate the native code from IR and run it
  llvm::InitializeNativeTarget();
  llvm::InitializeNativeTargetAsmPrinter();
  llvm::Function* _run = module->getFunction("_run");
  llvm::ExecutionEngine* ee = llvm::EngineBuilder(std::move(module)).create();
  llvm::GenericValue gv = ee->runFunction(_run, std::vector<llvm::GenericValue>());
  fflush(stdout);

  remove(ccname.c_str());
}

int main(int argc, char** argv) {
  std::ifstream ifs(argv[1]);
  std::string program((std::istreambuf_iterator<char>(ifs)), (std::istreambuf_iterator<char>()));
  BrainfuckLLVM llvm(program, argv[2]);
  llvm.run();
  return 0;
}
```

```
g++ -std=c++11 `/path/to/llvm-config --cxxflags --ldflags` -lclangASTMatchers -lclangFrontendTool -lclangFrontend -lclangDriver -lclangSerialization -lclangCodeGen -lclangParse -lclangSema -lclangToolingInclusions -lclangToolingCore  -lclangFormat -lclangIndex -lclangCrossTU -lclangStaticAnalyzerFrontend -lclangStaticAnalyzerCheckers -lclangStaticAnalyzerCore -lclangAnalysis -lclangARCMigrate -lclangRewriteFrontend -lclangRewrite -lclangEdit -lclangAST -lclangLex -lclangBasic `/path/to/llvm-config --libs` -o brainfuck brainfuck.cc

./brainfuck program.bf /path/to/clang
```

## <a id="dynasm"></a>DynASM
Another way of generating the native code is using [DynASM](https://luajit.org/dynasm.html) from LuaJIT and there is an [example](https://corsix.github.io/dynasm-doc/tutorial.html) online.
