---
title: 如何为Clang编译器写插件实现自动化重构
toc: true
top: false
original: true
date: 2020-08-07 19:38:37
tags:
- C/C++
categories:
- 编程
---

## 动机
最近在项目中采用`DCI`理论进行重构，核心是各个`Role`之间的交互，现存系统中有很多定义的抽象类型，若统一转成`ROLE`定义，形式、语义上也能够更加统一。举个例子：
```cpp
class IFoo {
public:
    virtual ~IFoo() = default;
    virtual const void* foo(int, int, char) const noexcept = 0;
    virtual const int* foo3(int, int, char) = 0;
    virtual int foo4(int, int = 0) const= 0;
    virtual int foo5() const = 0;
};
```

诸如此类，统一重构成如下形式：
```cpp
DEFINE_ROLE(IFoo) {
    ABSTRACT(foo(int, int, char) const noexcept -> const void*);
    ABSTRACT(foo3(int, int, char) -> const int*);
    ABSTRACT(foo4(int, int = 0) const -> int);
    ABSTRACT(foo5() const -> int);
};
```

还有后续需要将项目中的**数据表**重构成**结构体**，那么原来的`db.GetData(KEYID)`就需要重构成`tbl.key`形式。

由于我用过`clang-tidy`做过多次自动化重构，很自然地想到可以对`clang`编译器打补丁，定义规则让编译器自动帮我完成重构，而程序员的乐趣就是将此类**重复易错工作**转换成更有趣的方式，既能增长知识，又不失效率：）

考虑到这个任务比较简单，而量比较大，若用脚本匹配，得考虑各种情况：

- 匹配一个`class`
- 这个`class`必须是抽象类（除了析构函数外只有纯虚接口）
- 这个`class`不能被其他class嵌套
- 这个`class`不能被继承
- 成员函数各种修饰：`const/voliatile/noexception/&&`
- 识别返回类型后置
- 默认参数处理，比如匹配`=0`，别匹配到默认参数去了
- 空格处理
- 成员函数可能换行
- 删掉`public`等访问修饰词
- 删掉默认析构函数，因为`DEFINE_ROLE`已经定义了
- ...

简单的脚本是很难完成这个任务的，简单起见，我采用直接写插件的形式，而不是基于`clang-tidy`开发。

核心思想就是用`Clang AST DSL`匹配需要的部分，然后通过`MatchCallBack`对匹配部分利用`Rewriter`进行修改。

## Clang AST
`Clang AST`主要有这几种结点：`Decl`，`Stmt`和`Type`，进而衍生出很多其他类型的节点，有意思的是`AST`节点并没有公共基类，遍历`AST`相当于遍历图而不是简单的树。而`clang`提供的`DSL`更能很好融入到`cpp`中。更多信息可以参考[参考资料](#参考资料)：

通过`clang-check`工具可以把语法树`dump`出来：
```cpp
$ clang-check -ast-dump -ast-dump-filter IFoo testdecl.cpp
Dumping IFoo:
CXXRecordDecl 0x208cb90 <testdecl.cpp:8:1, line:15:1> line:8:7 class IFoo definition
|-DefinitionData polymorphic abstract has_constexpr_non_copy_move_ctor can_const_default_init
| |-DefaultConstructor exists non_trivial constexpr needs_implicit defaulted_is_constexpr
| |-CopyConstructor simple non_trivial has_const_param needs_implicit implicit_has_const_param
| |-MoveConstructor
| |-CopyAssignment non_trivial has_const_param implicit_has_const_param
| |-MoveAssignment
| `-Destructor non_trivial user_declared
|-CXXRecordDecl 0x208cca8 <col:1, col:7> col:7 implicit referenced class IFoo
|-AccessSpecDecl 0x208cd38 <line:9:1, col:7> col:1 public
|-CXXDestructorDecl 0x208cdf8 <line:10:5, col:29> col:13 ~IFoo 'void ()' virtual default noexcept-unevaluated 0x208cdf8
|-CXXMethodDecl 0x208d188 <line:11:5, col:62> col:25 foo 'const void *(int, int, char) const noexcept' virtual pure
| |-ParmVarDecl 0x208cef8 <col:29> col:32 'int'
| |-ParmVarDecl 0x208cf78 <col:34> col:37 'int'
| `-ParmVarDecl 0x208cff0 <col:39> col:43 'char'
|-CXXMethodDecl 0x208d440 <line:12:5, col:47> col:24 foo3 'const int *(int, int, char)' virtual pure
| |-ParmVarDecl 0x208d258 <col:29> col:32 'int'
| |-ParmVarDecl 0x208d2d8 <col:34> col:37 'int'
| `-ParmVarDecl 0x208d350 <col:39> col:43 'char'
|-CXXMethodDecl 0x208d670 <line:13:5, col:66> col:17 foo4 'int (int, int) const' virtual pure
| |-ParmVarDecl 0x208d510 <col:22, col:26> col:26 'int'
| `-ParmVarDecl 0x208d590 <col:40, col:56> col:44 'int' cinit
|   `-IntegerLiteral 0x20bc2f0 <col:56> 'int' 0
|-CXXMethodDecl 0x208d770 <line:14:5, col:32> col:17 foo5 'int () const' virtual pure
`-CXXMethodDecl 0x208d868 <line:8:7> col:7 implicit operator= 'IFoo &(const IFoo &)' inline default noexcept-unevaluated 0x208d868
  `-ParmVarDecl 0x208d978 <col:7> col:7 'const IFoo &'
```

对于我们这个例子，首先需要用`AST`将抽象类匹配出来，通过`clang-query`可以`Read Eval Print Loop`得到：
```bash
$ clang-query test/testdecl.cpp
clang-query> m cxxRecordDecl(isClass(), isDefinition())
Match #1:
testdecl.cpp:8:1: note: "root" binds here
class IFoo {
^~~~~~~~~~~~
1 match.
...
clang-query> m cxxRecordDecl(isClass(), isDefinition(), unless(isDerivedFrom(anything())), forEach(cxxMethodDecl(isPure())), unless(hasParent(cxxRecordDecl())))
Match #1:
testdecl.cpp:8:1: note: "root" binds here
class IFoo {
^~~~~~~~~~~~
1 match.
```

经过不断约束、尝试，最终得到如下`AST DSL`：
```cpp
cxxRecordDecl(isClass(),
    isDefinition(),
    unless(isDerivedFrom(anything())),
    unless(hasParent(cxxRecordDecl())),
    forEach(cxxMethodDecl(isPure(), unless(hasTrailingReturn()))))
```

从如上`dsl`可以得到的信息是：

- 匹配一个`class`
- 该`class`必须被定义
- 该`class`没有继承任何东西
- 该`class`没有被其他类型嵌套
- 该类必须包含纯虚接口

然而通过上述信息，无法判断一个`class`是否是抽象的，需要通过进一步判断。

## `ASTConsumer`
而匹配过程，是通过`ASTConsumer`类完成的，需要重写其提供的`HandleTranslationUnit`接口。 而`DSL`提供的`bind`接口可以供后续`MatchCallBack`获取使用。
```cpp
struct RewriteDeclConsumer: ASTConsumer {
    RewriteDeclConsumer(Rewriter &R) : RDHandler(R) {
        DeclarationMatcher DeclMatcher = cxxRecordDecl(
                isClass(),
                isDefinition(),
                unless(isDerivedFrom(anything())),
                unless(hasParent(cxxRecordDecl())),
                forEach(cxxMethodDecl(isPure(), unless(hasTrailingReturn())))
                ).bind("classdef");
        Finder.addMatcher(DeclMatcher, &RDHandler);
    }
    void HandleTranslationUnit(ASTContext &Ctx) override {
        Finder.matchAST(Ctx);
    }
private:
    MatchFinder Finder;
    RewriteDeclMatcher RDHandler;
};
```

## `MatchCallBack`
经过上述步骤后，实现该类`run`接口，就能对匹配的`AST`进行修改。
```cpp
struct RewriteDeclMatcher: MatchFinder::MatchCallback {
    RewriteDeclMatcher(Rewriter &Rewriter) : RewriteDeclWriter(Rewriter) {}
    void run(const MatchFinder::MatchResult &) override;
    void onEndOfTranslationUnit() override {
        // Output to stdout
        RewriteDeclWriter.getEditBuffer(RewriteDeclWriter.getSourceMgr().getMainFileID())
            .write(llvm::outs());
        // Replace in place
        // RewriteDeclWriter.overwriteChangedFiles();
    }
private:
    Rewriter RewriteDeclWriter;
};
```

来看看`run`的实现，首先将`ASTContext`和`SourceManager`存起来，方便后续使用：
```cpp
// ASTContext is used to retrieve the source location
ASTContext *Ctx = Result.Context;
const SourceManager& SM = Ctx->getSourceManager();
Rewriter::RewriteOptions rmOps;
rmOps.RemoveLineIfEmpty = true; // RemoveText的时候若空行，则删除整行
```

接下来判断该类是否为一个抽象类，若有一个接口不是纯虚接口，则退出：
```cpp
// check if class is abstract
const auto ClassDecl = Result.Nodes.getNodeAs<CXXRecordDecl>("classdef");
for (auto&& F: ClassDecl->methods()) {
    if (dyn_cast<CXXConstructorDecl>(F) ||
            dyn_cast<CXXDestructorDecl>(F) ||
            F->isImplicit()) {
        continue;
    }
    if (! F->isPure()) return;
}
```

将`class IFoo {`替换成`DEFINE_ROLE(IFOO) {`:
```cpp
{
    auto ClassName = ClassDecl->getDeclName();
    RewriteDeclWriter.ReplaceText(ClassDecl->getBeginLoc(), strlen("class "), "DEFINE_ROLE(");
    RewriteDeclWriter.InsertText(
        ClassDecl->getLocation().getLocWithOffset(ClassName.getAsString().length()), ")");
}
```

后面就可以遍历每个成员函数了，这里其实可以用`Vistor`实现，避免自己写迭代器`dyn_cast`了：
```cpp
for (auto&& decls: ClassDecl->decls()) {
    // ...
}
```

删除`public:`等访问修饰符：
```cpp
if (auto Access = dyn_cast<AccessSpecDecl>(decls); Access) {
    RewriteDeclWriter.RemoveText(Access->getSourceRange(), rmOps);
}
```

获取成员方法：
```cpp
if (auto F = dyn_cast<CXXMethodDecl>(decls); F) {
    // ...
}
```

删掉构造、析构函数：
```cpp
if (!F->isImplicit() && (dyn_cast<CXXDestructorDecl>(F) || dyn_cast<CXXConstructorDecl>(F))) {
    auto R = F->getSourceRange();
    R.setEnd(R.getEnd().getLocWithOffset(1)); // eat ';'
    RewriteDeclWriter.RemoveText(R, rmOps);
}
```

之后是处理纯虚接口，判断若为返回类型后置形式，则不处理。而匹配函数修饰符的时候，没有找到好的方案，只能通过词法模块逐个关键字处理（处理`const/voliatile/&&/noexcept`这类）。找到函数右括号位置作为箭头位置，而返回类型和`virtual`可以改成`ABSTRACT(`，`=0`替换成`)`：
```cpp
if (F->isPure()) {
    const TypeSourceInfo* TSI = F->getTypeSourceInfo();
    if (auto Proto = TSI->getType()->getAs<FunctionProtoType>();
            Proto && Proto->hasTrailingReturn()) { continue; }
    auto FTL = TSI->getTypeLoc().getAs<FunctionTypeLoc>();
    auto TrailingPos = Lexer::getLocForEndOfToken(FTL.getRParenLoc(), 0, SM, Ctx->getLangOpts());
    SourceRange EqualZero;
    // skip trailing cv && noexception
    {
        auto&& [FileId, offset] = SM.getDecomposedLoc(TrailingPos);
        StringRef File = SM.getBufferData(FileId);
        Lexer lexer(SM.getLocForStartOfFile(FileId),
                Ctx->getLangOpts(),
                File.begin(),
                File.data() + offset,
                File.end());
        Token T;
        bool find = false;
        while (! lexer.LexFromRawLexer(T) && !find) {
            if (T.is(tok::raw_identifier)) {
                auto& Info = Ctx->Idents.get(StringRef(SM.getCharacterData(T.getLocation()), T.getLength()));
                T.setKind(Info.getTokenID());
            }
            switch (T.getKind()) {
                case tok::amp:
                case tok::ampamp:
                case tok::kw_const:
                case tok::kw_volatile:
                case tok::kw_noexcept:
                    TrailingPos = T.getEndLoc();
                    break;
                case tok::equal:
                    EqualZero.setBegin(T.getLocation().getLocWithOffset(T.hasLeadingSpace() ? -1 : 0));
                    break;
                case tok::numeric_constant:
                    EqualZero.setEnd(T.getLocation());
                    find = true;
                    break;
                default:
                    break;
            }
        }
    }
    auto ReturnTypeRange = F->getReturnTypeSourceRange();
    SourceLocation VirtualLoc = F->getBeginLoc();
    ReturnTypeRange.setBegin(VirtualLoc.getLocWithOffset(strlen("virtual")));
    StringRef ReturnType = Lexer::getSourceText(
            Lexer::getAsCharRange(ReturnTypeRange, SM, Ctx->getLangOpts()),
            SM, Ctx->getLangOpts());
    RewriteDeclWriter.ReplaceText(ReturnTypeRange, "");
    RewriteDeclWriter.ReplaceText(VirtualLoc, strlen("virtual "), "ABSTRACT(");
    RewriteDeclWriter.ReplaceText(EqualZero, ")");
    RewriteDeclWriter.InsertText(TrailingPos, " ->" + ReturnType.str());
}
```

## 整合到一块
最后将我们的`ASTConsumer`注册到`clang`插件体系中去，也很简单，实现`PluginASTAction`的`CreateASTConsumer`接口即可：
```cpp
struct RewriteDecl: PluginASTAction {
    bool ParseArgs(const CompilerInstance &,
            const std::vector<std::string> &) override {
        return true;
    }
    std::unique_ptr<ASTConsumer> CreateASTConsumer(CompilerInstance &CI,
            StringRef file) override {
        RewriteDeclWriter.setSourceMgr(CI.getSourceManager(), CI.getLangOpts());
        return std::make_unique<RewriteDeclConsumer>(RewriteDeclWriter);
    }
private:
    Rewriter RewriteDeclWriter;
};

static FrontendPluginRegistry::Add<RewriteDecl> X
("RewriteDecl", "Refactor Interface by DEFINE_ROLE");
```

## 工程构建
新建一个CMakeLists.txt用于构建这项工程，更多细节请参考：[https://github.com/netcan/Practice/tree/master/cpp/clang-plugin](https://github.com/netcan/Practice/tree/master/cpp/clang-plugin)

```cmake
find_package(LLVM REQUIRED CONFIG)
find_package(Clang REQUIRED CONFIG)

message(STATUS "Found LLVM ${LLVM_PACKAGE_VERSION}")
message(STATUS "Using LLVMConfig.cmake in: ${LLVM_DIR}")

include_directories(${LLVM_INCLUDE_DIRS})
add_definitions(${LLVM_DEFINITIONS})

add_library(
    RefactorDecl
    SHARED
    RefactorDecl.cpp)

target_link_libraries(RefactorDecl
    "$<$<PLATFORM_ID:Darwin>:-undefined dynamic_lookup>")

add_executable(
    RefactorDeclMain
    RefactorDeclMain.cpp)

target_link_libraries(RefactorDeclMain
    PRIVATE
    clangTooling)
```

## 执行自动化重构
插件方式：
```cpp
$ clang++ -Xclang -load -Xclang libRewriteDecl.so -Xclang -plugin -Xclang RewriteDecl testdecl.cpp
DEFINE_ROLE(IFoo) {
    ABSTRACT(foo(int, int, char) const volatile noexcept -> const void*);
    ABSTRACT(foo3(int, int, char) && -> const int*);
    ABSTRACT(foo4(int, int = 0) const -> int);
    ABSTRACT(foo5() const -> int);
};
```

二进制执行方式：
```cpp
$ ./bin/RefactorDeclMain testdecl.cpp
DEFINE_ROLE(IFoo) {
    ABSTRACT(foo(int, int, char) const volatile noexcept -> const void*);
    ABSTRACT(foo3(int, int, char) && -> const int*);
    ABSTRACT(foo4(int, int = 0) const -> int);
    ABSTRACT(foo5() const -> int);
};
```

## 参考资料
- [https://s3.amazonaws.com/connect.linaro.org/yvr18/presentations/yvr18-223.pdf](https://s3.amazonaws.com/connect.linaro.org/yvr18/presentations/yvr18-223.pdf)
- [https://clang.llvm.org/docs/IntroductionToTheClangAST.html](https://clang.llvm.org/docs/IntroductionToTheClangAST.html)
- [https://clang.llvm.org/docs/LibASTMatchersReference.html#narrowing-matchers](https://clang.llvm.org/docs/LibASTMatchersReference.html#narrowing-matchers)

