+++
categories = [ "clang", "llvm", "C", "C++" ]
date = "2016-12-06T10:40:49-05:00"
description = "Using Clang libtooling for LLVM to generate C interface from C++ source code."
keywords = [ "Clang", "libtooling", "LLVM", "C", "C++", "source code generation", "source to source translation"]
title = "Generate C interface from C++ source code using Clang libtooling"
+++
I developed a [concurrent user-thread library](https://github.com/samanbarghi/uThreads) using C++, and was looking for a way to provide a C interface and make the functionality available in C. In order to achieve this goal, I started writting a C wrapper interface manually where each public method of a class is mapped to a function in C with the class name followed by the function name. Also, constructors are mapped to class name followed by _\_create_ and destructors are mapped to the class name followed by _\_destory_. To create an object, memory is allocated using _new_ in C++ and the pointer is casted to a struct pointer in C using _reinterpret\_cast_. The casted pointer in C is passed to member functions of that object and is used to call the member function of the instance. To delete the object, the pointer is passed to the _\_destroy_ function and _delete_ is used to call the deconstructor. For example, for the calss Rectangle below stored in `Rectangle.h`:

```
class Rectangle{
  private:
    int length;
    int width;
  public:
  Rectangle(int l, int w): length(l), width(w){};
  int area(){ return length*width;};
  perimeter() { return 2*(lendth+width);};
  ~Rectangle(){};
}
```

Two files `cwrapper.h` and `cwrapper.cpp` are created to map the Rectangle class to C:

```
#ifdef __cplusplus
extern "C"{
#endif

/* Wrapper struct to hold a pointer to
   Rectangle object in C  */
struct      WRectangle;             
typedef     struct WRectangle WRectangle;

/* Wrapper for the constructor */
WRectangle*   Rectangle_create(int l, int w);     
/* Wrapper for destructor */
void  Rectangle_destroy(WRectangle* self);
/* Wrapper for method area */
int Rectangle_area(WRectangle* self);
/* Wrapper for method perimeter */
int Rectangle_perimeter(WRectangle* self);
#ifdef __cplusplus
}
#endif
```

```
#include "Rectangle.h"
#include "cwrapper.h"

#ifdef __cplusplus
extern "C"{
#endif

WRectangle*   Rectangle_create(int l, int w){
  //calling the constructor and casting the pointer to WRectangle
  return reinterpret_cast<WRectangle*>( new Rectangle(l,w) );
}
void  Rectangle_destroy(WRectangle* self){
  //calling the destructor
  delete reinterpret_cast<Rectangle*>(self);
}
int Rectangle_area(WRectangle* self){
    return reinterpret_cast<Rectangle*>(self)->area();
}
int Rectangle_perimeter(WRectangle* self){
    return reinterpret_cast<Rectangle*>(self)->perimeter();
}
#ifdef __cplusplus
}
#endif
```

The wrapper functions can be used in C source code to interact with the C++ objects. I manually created two files [cwrapper.h](https://github.com/samanbarghi/uThreads/blob/v0.3.0/src/cwrapper.h) and [cwrapper.cpp](https://github.com/samanbarghi/uThreads/blob/v0.3.0/src/cwrapper.cpp) for uThreads library which make the library available in C. These files were maintainable when the library was small and there were not many classes and functions to map. However, as the library grew it became harder and harder to maintain the cwrapper functions for all the classes. Thus, I started exploring other options to generate such interface automatically, where the C++ source code is parsed and analysed and various information is extracted automatically, or to word it better I needed a compiler frontend!

[Clang libtooling](http://clang.llvm.org/docs/LibTooling.html) supports writing standalone tools based on Clang. To generate the C interface, I use [Clang AST Matcher](http://clang.llvm.org/docs/LibASTMatchersReference.html) to find public methods of specific classes and generate the C functions based on that. To get familiar with Clang AST you can refer to [this articls](http://clang.llvm.org/docs/IntroductionToTheClangAST.html).

For this tool to work, you need LLVM and Clang installed on your system. You can either [build them from source](http://clang.llvm.org/get_started.html), which will take a long time to finish, or you can install precompiled packages for your operating system. I am using Calng version 4.0.0 and LLVM version 4.0.0, but I tested the code with Clang and LLVM v3.8 and it works fine. To install Clang and LLVM under ubuntu issue `sudo apt-get install llvm clang`.

I am going to explain how I used Clang libtooling to generate the cwrapper files for uThreads. The finished code can be found [here](https://github.com/samanbarghi/CPP2C). This is not used to generate the actual cwrapper files under uThreads, as it still needs some improvements before it can be used in production. Before I explain the code, let's go through some rules and issues for generating the cwrapper functions:

* Cwrapper functions should only be generated for static and public member functions of each class. There is no point to map private member functions of classes to C.
* I am not going to map operator overloading, copy constructor and move constructor functions to C, as uThreads classes do not have any of that
* uThreads does not have a namespace yet (it will be added in the future), so there should be a way of figuring out the name of the classes under uThreads that I want to map to C. If there was a namespace, I could simply find all the classes under that namespace and use them to achieve the goal. But for now, I am using a list of class names to map to C stored in `std::vector`.
* Since function overloading is not supported in C, there should be a way to map overloaded functions to C. For now I use numbering to map the overloaded functions to C. e.g., uThread_create and uThread_create_1 are constructors for uThread class.
* Template functions are not supported in the current version and will be supported in the future.
* Here, all the functions only accepts builtin types and references to uThreads objects, thus make it easy to generate the C counterparts. However, if objects from outside libraries or even standard libraries are passed to or returned from the functions, it needs additional effort to make it work. Since those classes should be listed separately and a struct in C should be generated for them and a pointer to them stored in the C code.

Let's also take a look at the generated AST by Clang. If you pass `-Xclang -ast-dump` to clang, you can see the AST that is generated by clang. So issueing the following command in the root directory of uThreads will print out the AST in a simple text format:
```
clang++ -Xclang -ast-dump -fsyntax-only -I./src -std=c++11 include/uThreads.h
```
`include/uThreads.h` is the file that includes all the header files in uThreads, thus all uThreads classes and functions are present when passing the file to clang. Also, since many stdlib and system files are included the resulting AST is going to be very large, as it covers all the included files as well. Here are AST nodes related to constructor, destructor, and method `accept` from class Connection in uThreads:

[{{< figure src="/images/2016-12-06/ast-dump.png" title="AST Dump" >}}](/images/2016-12-06/ast-dump.png)

AST node related to constructor is of type `CXXConstructorDecl`, AST node related to the destructor is of type `CXXDestructorDecl`, and the accept method is of type `CXXMethodDecl`. [CXXConstructorDecl](http://clang.llvm.org/doxygen/classclang_1_1CXXConstructorDecl.html) and [CXXDestructorDecl](http://clang.llvm.org/doxygen/classclang_1_1CXXDestructorDecl.html) are both childern of class `CXXMethodDecl`. Hence, by finding all method declarations, i.e. `CXXMethodDecl`, in AST we meet all the requirments to generate the C interface.

[AST matchers](http://clang.llvm.org/docs/LibASTMatchersReference.html) are used to find and match AST nodes and perform a specific task based on the matched node. To understand the idea further, let's use `clang-query` which can be used to test AST matchers. For instance to query the uThreads AST, issue the following command:
```
clang-query include/uThreads.h -- -I./src -std=c++11
```
It enters `clang-query` console and you can issue `help` to see the available commands. Let's use the `match` command to find all the `CXXMethodDecl` in the AST:
```
clang-query> match cxxMethodDecl()
```

As before, since many files are included in the source code, the output is going to be large and includes all the methods, constructors and destructors from uThreads and all other classes included. Here is part of the output for class `Connection`:
```
Match #6987:

./src/io/Network.h:61:5: note: "root" binds here
    Connection() :
    ^~~~~~~~~~~~~~

Match #6988:

./src/io/Network.h:72:5: note: "root" binds here
    Connection(int fd) :
    ^~~~~~~~~~~~~~~~~~~~

Match #6989:

./src/io/Network.h:87:5: note: "root" binds here
    Connection(int domain, int type, int protocol) throw(std::system_error);
    ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Match #6990:

./src/io/Network.h:89:5: note: "root" binds here
    ~Connection();
    ^~~~~~~~~~~~~

Match #6991:

./src/io/Network.h:99:5: note: "root" binds here
    int accept(Connection *conn, struct sockaddr *addr, socklen_t *addrlen);
    ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

However, for the C interface we are only interested in public methods. So let's change the query to reflect that:
```
clang-query> match cxxMethodDecl(isPublic())
```

This time only the public methods are printed to the output. Also, since we are only interested in classes defined in uThreads library, I am going to use specific class names to limit the output. For example, to see all the public methods from class `Connection` issue the following commands:

```
clang-query> set output dump
clang-query> match cxxMethodDecl(isPublic(), ofClass(hasName("Connection")))
```

This time I changed the output to show AST dump instead of diagnostics, which results in the following:
```
Binding for "root":
CXXConstructorDecl 0x3712520 <./src/io/Network.h:61:5, line:64:5> line:61:5 Connection 'void (void)'
|-CXXCtorInitializer Field 0x3712178 'pd' 'class PollData *'
| `-CXXDefaultInitExpr 0x37162d8 <col:5> 'class PollData *'
|-CXXCtorInitializer Field 0x37121d0 'fd' 'int'
| `-UnaryOperator 0x3716260 <line:62:16, col:17> 'int' prefix '-'
|   `-IntegerLiteral 0x3716240 <col:17> 'int' 1
|-CompoundStmt 0x37163a8 <col:19, line:64:5>
| `-CXXMemberCallExpr 0x3716380 <line:63:9, col:14> 'void'
|   `-MemberExpr 0x3716348 <col:9> '<bound member function type>' ->init 0x3712298
|     `-CXXThisExpr 0x3716330 <col:9> 'class Connection *' this
`-FullComment 0x3940930 <line:56:7, line:59:47>
  |-ParagraphComment 0x3940800 <line:56:7>
  | `-TextComment 0x39407d0 <col:7> Text=" "
  |-BlockCommandComment 0x3940820 <col:8, col:52> Name="brief"
  | `-ParagraphComment 0x3940880 <col:14, col:52>
  |   `-TextComment 0x3940850 <col:14, col:52> Text=" Create a Connection that does not have"
  `-ParagraphComment 0x39408f0 <line:58:7, line:59:47>
    |-TextComment 0x39408a0 <line:58:7, col:65> Text=" This is useful for accept or socket functions that require"
    `-TextComment 0x39408c0 <line:59:7, col:47> Text=" a Connection object without fd being set"


Match #2:

Binding for "root":
CXXConstructorDecl 0x37126a0 <./src/io/Network.h:72:5, line:75:5> line:72:5 Connection 'void (int)'
|-ParmVarDecl 0x3712618 <col:16, col:20> col:20 used fd 'int'
|-CXXCtorInitializer Field 0x3712178 'pd' 'class PollData *'
| `-CXXDefaultInitExpr 0x3716460 <col:5> 'class PollData *'
|-CXXCtorInitializer Field 0x37121d0 'fd' 'int'
| `-ImplicitCastExpr 0x3716420 <line:73:16> 'int' <LValueToRValue>
|   `-DeclRefExpr 0x37163c8 <col:16> 'int' lvalue ParmVar 0x3712618 'fd' 'int'
|-CompoundStmt 0x3716530 <col:20, line:75:5>
| `-CXXMemberCallExpr 0x3716508 <line:74:9, col:14> 'void'
|   `-MemberExpr 0x37164d0 <col:9> '<bound member function type>' ->init 0x3712298
|     `-CXXThisExpr 0x37164b8 <col:9> 'class Connection *' this
`-FullComment 0x3940ba0 <line:66:7, line:70:45>
  |-ParagraphComment 0x39409d0 <line:66:7>
  | `-TextComment 0x39409a0 <col:7> Text=" "
  |-BlockCommandComment 0x39409f0 <col:8, line:67:7> Name="brief"
  | `-ParagraphComment 0x3940a70 <line:66:14, line:67:7>
  |   |-TextComment 0x3940a20 <line:66:14, col:61> Text=" Create a connection object with the provided fd"
  |   `-TextComment 0x3940a40 <line:67:7> Text=" "
  |-ParamCommandComment 0x3940a90 <col:8, col:16> [in] implicitly Param="fd" ParamIndex=0
  | `-ParagraphComment 0x3940af0 <<invalid sloc>>
  `-ParagraphComment 0x3940b60 <line:69:7, line:70:45>
    |-TextComment 0x3940b10 <line:69:7, col:71> Text=" If the connection is already established by other means, set the"
    `-TextComment 0x3940b30 <line:70:7, col:45> Text=" fd and add it to the polling structure"


Match #3:

Binding for "root":
CXXConstructorDecl 0x37129b0 <./src/io/Network.h:87:5, col:75> col:5 Connection 'void (int, int, int) throw(std::system_error)'
|-ParmVarDecl 0x3712798 <col:16, col:20> col:20 domain 'int'
|-ParmVarDecl 0x3712808 <col:28, col:32> col:32 type 'int'
|-ParmVarDecl 0x3712878 <col:38, col:42> col:42 protocol 'int'
`-FullComment 0x3940ec0 <line:78:7, line:86:5>
  |-ParagraphComment 0x3940c40 <line:78:7>
  | `-TextComment 0x3940c10 <col:7> Text=" "
  |-BlockCommandComment 0x3940c60 <col:8, line:79:7> Name="brief"
  | `-ParagraphComment 0x3940ce0 <line:78:14, line:79:7>
  |   |-TextComment 0x3940c90 <line:78:14, col:65> Text=" Same as socket syscall adds | SOCK_NONBLOCK to type"
  |   `-TextComment 0x3940cb0 <line:79:7> Text=" "
  |-BlockCommandComment 0x3940d00 <col:8, col:37> Name="return"
  | `-ParagraphComment 0x3940d60 <col:15, col:37>
  |   `-TextComment 0x3940d30 <col:15, col:37> Text=" same as socket syscall"
  |-ParagraphComment 0x3940e20 <line:81:7, line:84:29>
  | |-TextComment 0x3940d80 <line:81:7, col:69> Text=" Throws a std::system_error exception. Do not call from C code."
  | |-TextComment 0x3940da0 <line:82:7, col:68> Text=" The unerlying socket is always nonbelocking. This is achieved"
  | |-TextComment 0x3940dc0 <line:83:7, col:60> Text=" by adding a  (| SOCK_NONBLOCK) to type, thus requires"
  | `-TextComment 0x3940de0 <line:84:7, col:29> Text=" linux kernels > 2.6.27"
  `-ParagraphComment 0x3940e70 <line:86:1, col:5>
    `-TextComment 0x3940e40 <col:1, col:5> Text="     "


Match #4:

Binding for "root":
CXXDestructorDecl 0x3712ad0 <./src/io/Network.h:89:5, col:17> col:5 ~Connection 'void (void)' noexcept-unevaluated 0x3712ad0

```

We can use the final version of the matcher to find the public methods of interest and generate C source code based on that. We start by creating an option category which is applied to all command-line options. Also you can define new options for the command line tool using option tables, so arguments can be passed to your program. I am not going to use that feature in the current implementation, but included "-o" option just to show how it's done:

```
/** Options **/
static cl::OptionCategory CPP2CCategory("CPP2C options");
static std::unique_ptr<opt::OptTable> Options(createDriverOptTable());
static cl::opt<std::string> OutputFilename("o", cl::desc(Options->getOptionHelpText((options::OPT_o))));
```
I also create a struct with custom string streams which is used to generate the final C `.h` and `.c` files, a vector of the name of the classes from uThreads to be explored and a map of function names to an integer, which is used to keep track of functions and detect overleaded functions.

```
/** Classes to be mapped to C **/
struct OutputStreams{
	string headerString;
	string bodyString;

	llvm::raw_string_ostream HeaderOS;
	llvm::raw_string_ostream BodyOS;

	OutputStreams() : headerString(""), bodyString(""), HeaderOS(headerString), BodyOS(bodyString){};
};


vector<string> ClassList = {"uThread", "kThread", "Cluster", "Connection", "Mutex", "OwnerLock", "ConditionVariable", "Semaphore", "uThreadPool"};

map<string, int> funcList;
```
Next, we write the main function:

```
int main(int argc, const char **argv) {
	// parse the command-line args passed to your code
	CommonOptionsParser op(argc, argv, CPP2CCategory);
	// create a new Clang Tool instance (a LibTooling environment)
	ClangTool Tool(op.getCompilations(), op.getSourcePathList());


	// run the Clang Tool, creating a new FrontendAction
	return Tool.run(newFrontendActionFactory<MyFrontendAction>().get());
}
```
Whithin the main function, we pass the OptionCategory to the `CommonOptionsParser` along with argc and argv to parse the arguments passed to the code. Next, We create a `ClangTool` instance and pass the Compilation and source path lists to it. Finally, the ClangTool needs a new FrontendAction for each translation unit we run
on.  Thus, it takes a FrontendActionFactory as parameter. To create a `FrontendActionFactory` from a given FrontendAction type, we call `newFrontendActionFactory<MyFrontendAction>()`. `MyFrontendAction` is a `ASTFrontendAction` which is used for AST consumer-based frontend actions, and since we are going to rely on AST matchers we use the `ASTFrontendAction`.

```
// For each source file provided to the tool, a new FrontendAction is created.
 class MyFrontendAction: public ASTFrontendAction {
public:
	MyFrontendAction() {
		OS.HeaderOS << 	"#ifndef UTHREADS_CWRAPPER_H\n"
						"#define UTHREADS_CWRAPPER_H_\n"
						"#include <pthread.h>\n"
						"#include <sys/types.h>\n"
						"#include <sys/socket.h>\n"
						"#include <inttypes.h>\n\n"

						"#ifdef __cplusplus\n"
						"extern \"C\"{\n"
						"#endif\n"
						"#include <stdbool.h>\n";
		OS.BodyOS << "#include \"generic/basics.h\"\n"
					"#include \"cwrapper.h\"\n"
					"#include \"runtime/uThread.h\"\n"
					"#include \"runtime/uThreadPool.h\"\n"
					"#include \"runtime/kThread.h\"\n"
					"#include \"io/Network.h\"\n"
						"#ifdef __cplusplus\n"
						"extern \"C\"{\n"
						"#endif\n";

	}

	void EndSourceFileAction() override {

		StringRef headerFile("cwrapper.h");
		StringRef bodyFile("cwrapper.cpp");

        // Open the output file
        std::error_code EC;
        llvm::raw_fd_ostream HOS(headerFile, EC, llvm::sys::fs::F_None);
        if (EC) {
            llvm::errs() << "while opening '" << headerFile<< "': "
            << EC.message() << '\n';
            exit(1);
        }
        llvm::raw_fd_ostream BOS(bodyFile, EC, llvm::sys::fs::F_None);
        if (EC) {
            llvm::errs() << "while opening '" << bodyFile<< "': "
            << EC.message() << '\n';
            exit(1);
        }


		OS.HeaderOS << "#ifdef __cplusplus\n"
						"}\n"
						"#endif\n"
						"#endif /* UTHREADS_CWRAPPER_H_ */\n";

		OS.BodyOS<< "#ifdef __cplusplus\n"
						"}\n"
						"#endif\n";

		OS.HeaderOS.flush();
		OS.BodyOS.flush();
		HOS<< OS.headerString << "\n";
		BOS<< OS.bodyString << "\n";

	}

	std::unique_ptr<ASTConsumer> CreateASTConsumer(CompilerInstance &CI,
			StringRef file) override {

		return llvm::make_unique<MyASTConsumer>(OS);
	}

private:
	OutputStreams OS;
};
```

`MyFrontendAction` includes an instance of `OutputStreams` that has two string streams, one for the header file and the other one for the `.cpp` file. Both streams are intialized with the required include statements and `#ifdef` statements to be placed at the beginning of the file. The `EndSourceFileAction` is a callback at the end of processing a single input, and is used to append required strings to the end of the streams and save the streams to `cwrapper.h` and `cwrapper.cpp` files. Finally, `CreateASTConsumer` is called to create the AST consumer object for this action. Here an instance of `MyASTConsumer` is created and returned, we also pass the `OutputStreams` instance by reference, so later it can be passed to handlers. AST consumer, in our case, is responsible for registering matchers and run them over the AST. Here, the list of classes is used along with the `cxxMethodDecl` matcher we came up with earlier to match the methods of interest.

```
// Implementation of the ASTConsumer interface for reading an AST produced
// by the Clang parser. It registers a couple of matchers and runs them on
// the AST.
class MyASTConsumer: public ASTConsumer {
public:
	MyASTConsumer(OutputStreams& os) : OS(os),
			HandlerForClassMatcher(os){
		// Add a simple matcher for finding 'if' statements.

		for(string& className : ClassList){
			OS.HeaderOS << "struct      W"<< className << "; \n"
						   "typedef     struct W"<< className << " W"<< className << ";\n";
			//oss.push_back(std::move(os))

			DeclarationMatcher classMatcher = cxxMethodDecl(isPublic(), ofClass(hasName(className))).bind("publicMethodDecl");
			Matcher.addMatcher(classMatcher, &HandlerForClassMatcher);
		}

	}

	void HandleTranslationUnit(ASTContext &Context) override {
		// Run the matchers when we have the whole TU parsed.
		Matcher.matchAST(Context);
	}

private:
	OutputStreams& OS;
	classMatchHandler HandlerForClassMatcher;

	MatchFinder Matcher;
};
```
`MyASTConsumer` has an instance of a `classMatchHandler` and a `MatchFinder`. `MatchFinder` is used to register matchers and handlers and to apply the matchers to AST Context and call the related handlers when a match is found. In the constructor, we add all the matchers using `addMatcher`, and under the `HandleTranslationUnit` we call `matchAST` to perform the matches and do the callbacks. We also bind the match to a string (`publicMethodDecl`), which is used later to differentiate among matchers. Also, notice that for each class a struct is generated with name `WclassName` along with a typedef with the same name. As explained eariler, this struct is used to hold a pointer to the instance of the class in C.

Class `classMatchHandler` is a child of `MatchFinder::MatchCallBack` class, and is used to perform the required task after a match is found. We also override member function `run`, that accepts a `MatchResult` and is called upon finding a match. The first step is to check whether the matched node with the provided string `publicMethodDecl` is of type `cxxMethodDecl`.

```
if (const CXXMethodDecl *cmd = Result.Nodes.getNodeAs<CXXMethodDecl>("publicMethodDecl")){
			string methodName = "";
			string className = cmd->getParent()->getDeclName().getAsString();
			string returnType = "";
			string returnCast = "";
			bool shouldReturn, isPointer;
			string self = "W" + className + "* self";
			string separator = ", ";
			string bodyEnd = "";

			std::stringstream functionBody;
```
If so, we create a few of strings variables in order to generate the final C method. `className` captures the name of the parent class of the method, i.e. the name used to find the method in the matcher, by getting the declaration name of the parent class as string. To map member functions to C, there is a need to send the pointer to the object as the first argument, so the string `self` contains the string that represents the pointer to the struct in C that is named as _self_. The rest of the variables are explained later.

As explained eariler, operator overloading is ignored and not mapped to C, thus:
```
//ignore operator overloadings
if(cmd->isOverloadedOperator())
  return;
```
Next, we check whether the matched method is a constructor:
```
//constructor
if (const CXXConstructorDecl* ccd = dyn_cast<CXXConstructorDecl>(cmd)) {
  if(ccd->isCopyConstructor() || ccd->isMoveConstructor()) return;
  methodName = "_create";
  returnType = "W" + className + "*";
  self = "";
  separator = "";
  functionBody << "return reinterpret_cast<"<< returnType << ">( new " << className << "(";
  bodyEnd += "))";
  ```

  If `cmd` can be dynamically casted to `CXXConstructorDecl`, it means the method is a constructor. If the constructor is a copy constructor or move constructor we are not going to map it to C as explained above. The final method name should be `className_create`, so we set the `methodName` to be `_create` and later concatenate it to the calss name. Also, since the constructor creates an instance of the object and return a pointer to the created object, the return type should be the same as the struct type that holds the pointer to the object in C. `self` string will be empty, since there is no instance of the object yet to be passed to the constructor. `separator` is used to separate function arguments in the string, and since there is no self, the next argument do not need a comma before it. `functionBody` is a string stream that holds the information related to the function definition. For the constructor, we know that it returns the casted pointer to the object created by `new`, so the related string is generated accordingly. `bodyEnd` holds the string to complete the function definition, and since there are two open parentheses, they should be closed after arguments are placed in between the parentheses.

  Next, we check wether the method is a destructor:
  ```
  }else if (isa<CXXDestructorDecl>(cmd)) {
  methodName = "_destroy";
  returnType = "void";
  functionBody << " delete reinterpret_cast<"<<className << "*>(self)";
  ```
If the method declaration `isa<CXXDestructorDecl>`, it means the method is a destructor. Thus, the `methodName` should be `_destroy`, `returnType` should be void, and `functionBody` just calls `delete` on the casted pointer.

If the method is not a constructor, destructor, or overloaded operator, it is a member or static function:
```
}else{
  methodName = "_" + cmd->getNameAsString();
  const QualType qt = cmd->getReturnType();
  std::tie(returnType, returnCast, isPointer, shouldReturn) = determineCType(qt);

  //should this function return?
  if(shouldReturn)
  	functionBody << "return ";

  if(returnCast != ""){
  	//if not pointer and it needs to be casted, then return the pointer
  	if(!isPointer)
  		functionBody << "&";
  	functionBody << "reinterpret_cast<"<< returnType << ">(";
  	bodyEnd += ")";
  }

  //if Static call it properly
  if(cmd->isStatic())
  	functionBody <<className << "::" << cmd->getNameAsString() << "(";
  //if not  use the passed object to call the method
  else
  	functionBody << "reinterpret_cast<"<<className<<"*>(self)->" << cmd->getNameAsString() << "(";

  bodyEnd += ")";
}
```
The `methodName` in this case is the same as the name of the method with prefix `_`. Next step is to determine what is the return type of the method, for that purpose we use `getReturnType` to find the `QualType`. Then, we pass the `QualType` to the helper function `determineCType`, in order to find the type, whether it needs casting, whether it's a pointer and whether it's a void type. Here is the definition of the `determineCType` function:
```
tuple<string, string, bool, bool> determineCType(const QualType& qt){

  string CType = "";
  string CastType = ""; //whether this should be casted or not
  bool 	isPointer = false;
  bool 	shoulReturn = true;

  //if it is builtin type use it as is
  if(qt->isBuiltinType() || (qt->isPointerType() && qt->getPointeeType()->isBuiltinType())){
    CType = qt.getAsString();
    if(qt->isVoidType())
      shoulReturn = false;
  //if it is a CXXrecordDecl then return a pointer to WName*
  }else if(qt->isRecordType()){
    const CXXRecordDecl* crd = qt->getAsCXXRecordDecl();
    string recordName = crd->getNameAsString();
    CType = "W" + recordName + "*";
    CastType = recordName+ "*";

  }else if( (qt->isReferenceType() || qt->isPointerType()) && qt->getPointeeType()->isRecordType()){
    isPointer = true; //to properly differentiate among cast types
    const CXXRecordDecl* crd = qt->getPointeeType()->getAsCXXRecordDecl();
    string recordName = crd->getNameAsString();
    if ( std::find(ClassList.begin(), ClassList.end(), recordName) != ClassList.end() ){
      CType = "W" + recordName + "*";
      CastType = recordName + "*";
    }else{
      CType = recordName+"*";
    }

  }
  return make_tuple(CType, CastType, isPointer, shoulReturn);

}
```
This function returns 4 variables; one that reperesents the type that should be returned, one that shows what the return type should be casted from, and two booleans reperesenting whether the type is a pointer and whether the function needs a `return` statement before the function call. First, if the type is a builtin type, or is a pointer to a builtin type, there is no need to cast it (castType is left empty), and if this builtin type is `void` there is no need to put the `return` keyword before calling the function. Otherwise, if the type is a record type (class or struct), since in uThreads all functions only return record types that is defined whitin uThreads library, we can be sure that by casting the return type to `WclassName` struct, we are returning the instance of the object (it can be problematic if the returned object needs to be copied, since the original memory might be freed at the end of the function and the returned pointer is not valid anymore. But since in uThreads all objects are returned either by reference or only pointers are being returned, this condition does not apply to uThreads. Thus, it should be modified if you need to use it in another projects. This part is mainly relevant for function arguments, when later `determineCType` is called to determine the type of the passed arguments). Finally, if the return type is a pointer or reference to a record type, and the pointer is to a class under the uThreads list of classes, return the `WclassName*`, otherwise return the pointer itself.

Continuing on determining the return type, if the `shouldRetran` flag is true, a `return` keyword is added before the function call. If the return variable requires casting, insert `reinterpret_cast` along with the C return type before calling the function. Finally, generate the function call: if the function is static, call it by using `className::methodName`, and if not use `reinterpret_cast` to cast the passed `self` pointer to the object pointer and call the function by  `self->methodName`.

Next, let's generate the function declaration in C by concatenating the related strings together:
```
std::stringstream funcname;
funcname << returnType << " " << className << methodName;

auto it = funcList.find(funcname.str());

if(it != funcList.end()){
	it->second++;
	funcname << "_" << it->second ;
}else{
	funcList[funcname.str()] = 0;
}

funcname << "(" << self;
```

Function name is of form `returnType className_methodName` in C. In addition, as explained earlier, if there is any function overloading, a number is added to the end of the function name. Thus, a map of function names to integer is used to keep track of the number of times the function of the specified class appeared. If it appears more than one time, a suffix `_n` is added to the end of the function, where `n` is the number of times the function appeared. Finally, we add a parentheses to the end of the function name, along with the `self` string, that reperesnts whether a pointer to the object is passed to the function or not, and start generating the argument list:
```
for(unsigned int i=0; i<cmd->getNumParams(); i++)
{
  const QualType qt = cmd->parameters()[i]->getType();
  std::tie(returnType, returnCast, isPointer, shouldReturn) = determineCType(qt);
  funcname << separator << returnType << " ";
  funcname << cmd->parameters()[i]->getQualifiedNameAsString() << "";

  if(i !=0 )
    functionBody << separator;
  if(returnCast == "")
    functionBody << cmd->parameters()[i]->getQualifiedNameAsString();
  else{
    if(!isPointer)
      functionBody << "*";
    functionBody << "reinterpret_cast<" << returnCast << ">("<< cmd->parameters()[i]->getQualifiedNameAsString() << ")";

  }

  string separator = ", ";
}
```

To generate the argument list in C, we iterate through the method arguments by using `getNumParams` and `parameters` array. The type of the argument in C is determined by calling to `determineCType`, and based on the tpe of the argument it might require casting. After generating the list of the arguments, we simply close the parentheses and print the generated function name and body in header and body string streams:
```
funcname << ")";

OS.HeaderOS << funcname.str() << ";\n";

OS.BodyOS << funcname.str() << "{\n    ";
OS.BodyOS << functionBody.str();
OS.BodyOS << bodyEnd << "; \n}\n" ;
```

As explained earlier, `BodyOS` and `HeaderOS` streams are flushed to related files in the end. To build and install the binary simply clone the repo and issue the following commands (you require cmake and make to be installed):
```
git clone https://github.com/samanbarghi/CPP2C
cd CPP2C
mkdir build
cmake ..
make
sudo make install
```
The binary is installed under `/usr/local/bin` and you can use it under the uThreads directory as follows:
```
git clone https://github.com/samanbarghi/uThreads
cd uThreads
cpp2c include/uThreads.h -- -I./src -std=c++11
```
`cpp2c` creates two files `cwrapper.h` and `cwrapper.cpp` that holds the generated C interface. You can find the generated file [here](https://github.com/samanbarghi/CPP2C/tree/master/output).


Another issue with current approach is in cases where there is no public constructor (e.g., uThread class does not have a public constructor and uThread instances are created using uThreads::create function) the content of the constructor is going to be wrong and need to be updated. A possible fix that can further improve the current approach is to annotate the C++ source code with the intended C interface, and in cases the annotation is used simply use the annotation instead of generating the function.

 Although the C interface generator still has some shortcomings when it comes to return types, dealing with template functions and out of uThreads classes, this practice shows the potential of using such approach to automate the generation of C interfaces from C++ source code, specially for large projects.
