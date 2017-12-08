---
title: CMake官方Tutorial中英对照翻译
date: 2015-01-08 00:00:00
tags:
  - tech
  - cmake
---

# CMake Tutorial

原版英文Tutorial：

http://www.cmake.org/cmake-tutorial/

Step1-Step7的源代码及CMakeLists：

http://public.kitware.com/cgi-bin/viewcvs.cgi/CMake/Tests/Tutorial/

> Below is a step-by-step tutorial covering common build system issues
> that CMake helps to address. Many of these topics have been introduced
> in Mastering CMake as separate issues but seeing how they all work
> together in an example project can be very helpful. This tutorial can
> be found in the Tests/Tutorial directory of the CMake source code
> tree. Each step has its own subdirectory containing a complete copy of
> the tutorial for that step

下面是涵盖CMake解决常见构建系统问题的一个手把手教程。通过单独的例子，阐述了CMake是如何协同工作的，这将会对你十分有帮助。这个教程可以在CMake源代码树的`Test/Tutorial`目录下找到。每个步骤有各自的子目录，且包含了一份关于这个步骤的完整教程。

# A Basic Starting Point (Step1)  简单的开始

> The most basic project is an executable built from source code files.
> For simple projects a two line CMakeLists file is all that is
> required. This will be the starting point for our tutorial. The
> CMakeLists file looks like:

最基本的工程是从源代码文件构建出可执行程序。对于简单的工程而言，一个两行的`CMakeLists`文件已经足够了。这将会是我们教程开始的第一步。`CMakeLists`文件看起来像这样：

```

cmake_minimum_required (VERSION 2.6)

project (Tutorial)

add_executable(Tutorial tutorial.cxx)

```

> Note that this example uses lower case commands in the CMakeLists
> file. Upper, lower, and mixed case commands are supported by CMake.
> The source code for tutorial.cxx will compute the square root of a
> number and the first version of it is very simple, as follows:

注意到这个例子在`CMakeLists`中用的是小写命令。无论是大写、小写、还是混合大小写，CMake都支持。
源代码文件`tutorial.cxx`将计算一个数的平方根，第一个版本十分简单：

    <!-- lang: cpp -->
    // A simple program that computes the square root of a number
    #include <stdio.h>
    #include <stdlib.h>
    #include <math.h>
    
    int main (int argc, char *argv[])
    {
      if (argc < 2)
        {
        fprintf(stdout,"Usage: %s number\n",argv[0]);
        return 1;
        }
      double inputValue = atof(argv[1]);
      double outputValue = sqrt(inputValue);
      fprintf(stdout,"The square root of %g is %g\n",
              inputValue, outputValue);
      return 0;
    }

## Adding a Version Number and Configured Header File 添加一个版本号和用于配置的头文件

> The first feature we will add is to provide our executable and project
> with a version number. While you can do this exclusively in the source
> code, doing it in the CMakeLists file provides more flexibility. To
> add a version number we modify the CMakeLists file as follows:

我们将为可执行程序和工程提供一个版本号作为第一个特性。当然你可以在源代码中专门这样做，但是使用`CMakeLists`文件将更加灵活。为了添加一个版本号，我们修改`CMakeLists`文件如下：

    <!-- lang: shell -->
    cmake_minimum_required (VERSION 2.6)
    project (Tutorial)
    # The version number.
    set (Tutorial_VERSION_MAJOR 1)
    set (Tutorial_VERSION_MINOR 0)
     
    # configure a header file to pass some of the CMake settings
    # to the source code
    configure_file (
      "${PROJECT_SOURCE_DIR}/TutorialConfig.h.in"
      "${PROJECT_BINARY_DIR}/TutorialConfig.h"
      )
     
    # add the binary tree to the search path for include files
    # so that we will find TutorialConfig.h
    include_directories("${PROJECT_BINARY_DIR}")
     
    # add the executable
    add_executable(Tutorial tutorial.cxx)

> Since the configured file will be written into the binary tree we must
> add that directory to the list of paths to search for include files.
> We then create a TutorialConfig.h.in file in the source tree with the
> following contents:

由于配置文件将被写入二进制树，我们必须要在路径列表中添加目录以搜索包含文件。
然后我们在源代码树中添加`TutorialConfig.h.in`文件：

    <!-- lang: cpp -->
    // the configured options and settings for Tutorial
    #define Tutorial_VERSION_MAJOR @Tutorial_VERSION_MAJOR@
    #define Tutorial_VERSION_MINOR @Tutorial_VERSION_MINOR@

> When CMake configures this header file the values for
> @Tutorial_VERSION_MAJOR@ and @Tutorial_VERSION_MINOR@ will be replaced
> by the values from the CMakeLists file. Next we modify tutorial.cxx to
> include the configured header file and to make use of the version
> numbers. The resulting source code is listed below.

CMake在配置过程中，将从`CMakeLists`文件中找到并替换头文件中的`@Tutorial_VERSION_MAJOR@`和`@Tutorial_VERSION_MINOR@`。之后我们修改`tutorial.cxx`，使它包含配置头文件，然后利用版本号。最终的源代码如下：

    <!-- lang: cpp -->
    // A simple program that computes the square root of a number
    #include <stdio.h>
    #include <stdlib.h>
    #include <math.h>
    #include "TutorialConfig.h"
     
    int main (int argc, char *argv[])
    {
      if (argc < 2)
        {
        fprintf(stdout,"%s Version %d.%d\n",
                argv[0],
                Tutorial_VERSION_MAJOR,
                Tutorial_VERSION_MINOR);
        fprintf(stdout,"Usage: %s number\n",argv[0]);
        return 1;
        }
      double inputValue = atof(argv[1]);
      double outputValue = sqrt(inputValue);
      fprintf(stdout,"The square root of %g is %g\n",
              inputValue, outputValue);
      return 0;
    }

> The main changes are the inclusion of the TutorialConfig.h header file
> and printing out a version number as part of the usage message.

主要的改变是添加了`TutorialConfig.h`头文件以及在使用帮助中打印出了版本号。

# Adding a Library (Step 2) 添加一个库

> Now we will add a library to our project. This library will contain
> our own implementation for computing the square root of a number. The
> executable can then use this library instead of the standard square
> root function provided by the compiler. For this tutorial we will put
> the library into a subdirectory called MathFunctions. It will have the
> following one line CMakeLists file:

现在我们将为工程添加一个库。这个库将包含计算一个数的平方根的实现代码。可执行程序可以使用这个库，来替代编译器提供的标准平方根函数。在这个教程中，我们将把库放进一个叫做`MathFunctions`的子目录中。`CMakeLists`文件将包含这样一行：

    <!-- lang: shell -->
    add_library(MathFunctions mysqrt.cxx)

> The source file mysqrt.cxx has one function called mysqrt that
> provides similar functionality to the compiler’s sqrt function. To
> make use of the new library we add an add_subdirectory call in the top
> level CMakeLists file so that the library will get built. We also add
> another include directory so that the MathFunctions/mysqrt.h header
> file can be found for the function prototype. The last change is to
> add the new library to the executable. The last few lines of the top
> level CMakeLists file now look like:

源代码文件`mysqrt.cxx`有一个叫做`mysqrt`的函数，提供了一个类似于编译器`sqrt`函数的功能。为了能利用这个新库，我们在`CMakeLists`文件的顶部添加一个`add_subdirectory`调用来使之能够被构建。同时，我们也添加一个包含目录使`MathFunctions/mysqrt.h`头文件提供函数原型。最后一个改变是给可执行程序添加新库。`CMakeLists`文件的最后几行看起来像这样：

    <!-- lang: shell -->
    include_directories ("${PROJECT_SOURCE_DIR}/MathFunctions")
    add_subdirectory (MathFunctions) 
    
    # add the executable
    add_executable (Tutorial tutorial.cxx)
    target_link_libraries (Tutorial MathFunctions)

> Now let us consider making the MathFunctions library optional. In this
> tutorial there really isn’t any reason to do so, but with larger
> libraries or libraries that rely on third party code you might want
> to. The first step is to add an option to the top level CMakeLists
> file.

现在让我们考虑使`MathFunctions`库可选。这个教程中确实没有任何理由这样做，但是对于更大的库或依赖于第三方代码的库来说，你可能更愿意这样做。第一步是在`CMakeLists`中添加一个选项：

    <!-- lang: shell -->
    # should we use our own math functions?
    option (USE_MYMATH 
            "Use tutorial provided math implementation" ON) 

> This will show up in the CMake GUI with a default value of ON that the
> user can change as desired. This setting will be stored in the cache
> so that the user does not need to keep setting it each time they run
> CMake on this project. The next change is to make the build and
> linking of the MathFunctions library conditional. To do this we change
> the end of the top level CMakeLists file to look like the following:

这将在`CMake GUI`中默认显示为ON，这样使用者可以根据需要来改变它。这个配置将被保存在缓存中，这样使用者不必在工程中使用CMake时每次都去配置它。下一个改变是使构建和链接`MathFunctions`库有条件化。为了达到目的我们改变`CMakeLists`文件如下：

    <!-- lang: shell -->
    # add the MathFunctions library?
    #
    if (USE_MYMATH)
      include_directories ("${PROJECT_SOURCE_DIR}/MathFunctions")
      add_subdirectory (MathFunctions)
      set (EXTRA_LIBS ${EXTRA_LIBS} MathFunctions)
    endif (USE_MYMATH)
     
    # add the executable
    add_executable (Tutorial tutorial.cxx)
    target_link_libraries (Tutorial  ${EXTRA_LIBS})

> This uses the setting of USE_MYMATH to determine if the MathFunctions
> should be compiled and used. Note the use of a variable (EXTRA_LIBS in
> this case) to collect up any optional libraries to later be linked
> into the executable. This is a common approach used to keep larger
> projects with many optional components clean. The corresponding
> changes to the source code are fairly straight forward and leave us
> with:

它使用了`USE_MYMATH`设置来决定`MathFunctions`是否应该被编译和使用。注意变量`EXTRA_LIBS`（本例中）用来收集接下来将被链接到可执行程序中的可选库。这是一个保持大型工程和许多可选组件干净的常用方法。在源代码中的相应改变非常简单：

    <!-- lang: cpp -->
    // A simple program that computes the square root of a number
    #include <stdio.h>
    #include <stdlib.h>
    #include <math.h>
    #include "TutorialConfig.h"
    #ifdef USE_MYMATH
    #include "MathFunctions.h"
    #endif
     
    int main (int argc, char *argv[])
    {
      if (argc < 2)
        {
        fprintf(stdout,"%s Version %d.%d\n", argv[0],
                Tutorial_VERSION_MAJOR,
                Tutorial_VERSION_MINOR);
        fprintf(stdout,"Usage: %s number\n",argv[0]);
        return 1;
        }
     
      double inputValue = atof(argv[1]);
     
    #ifdef USE_MYMATH
      double outputValue = mysqrt(inputValue);
    #else
      double outputValue = sqrt(inputValue);
    #endif
     
      fprintf(stdout,"The square root of %g is %g\n",
              inputValue, outputValue);
      return 0;
    }

> In the source code we make use of USE_MYMATH as well. This is provided
> from CMake to the source code through the TutorialConfig.h.in
> configured file by adding the following line to it:

在源代码中我们利用了`USE_MYMATH`。它是由`TutorialConfig.h.in`文件通过CMake提供的：

    <!-- lang: cpp -->
    #cmakedefine USE_MYMATH

# Installing and Testing (Step 3) 安装和测试

> For the next step we will add install rules and testing support to our
> project. The install rules are fairly straight forward. For the
> MathFunctions library we setup the library and the header file to be
> installed by adding the following two lines to MathFunctions’
> CMakeLists file:

下一步我们将给工程添加安装规则和测试支持。安装规则相当简单。

对于`MathFunctions`库，我们通过在`MathFunction`的`CMakeLists`文件中添加下面两行，来使它的库和头文件可以被安装：

    <!-- lang: shell -->
    install (TARGETS MathFunctions DESTINATION bin)
    install (FILES MathFunctions.h DESTINATION include)

> For the application the following lines are added to the top level
> CMakeLists file to install the executable and the configured header
> file:

对于应用程序，在`CMakeLists`中添加下面的命令来安装可执行程序和配置文件：

    <!-- lang: shell -->
    # add the install targets
    install (TARGETS Tutorial DESTINATION bin)
    install (FILES "${PROJECT_BINARY_DIR}/TutorialConfig.h"        
             DESTINATION include)

> That is all there is to it. At this point you should be able to build
> the tutorial, then type make install (or build the INSTALL target from
> an IDE) and it will install the appropriate header files, libraries,
> and executables. The CMake variable CMAKE_INSTALL_PREFIX is used to
> determine the root of where the files will be installed. Adding
> testing is also a fairly straight forward process. At the end of the
> top level CMakeLists file we can add a number of basic tests to verify
> that the application is working correctly.

就是这么一回事，现在你能够构建这个教程了，然后执行`make install`（或者通过IDE构建INSTALL目标），它将安装适当的头文件，库和可执行程序。CMake变量`CMAKE_INSTALL_PREFIX`用来决定所安装文件的根目录。

添加测试也非常简单。在顶级`CMakeLists`文件的最后，我们可以添加一系列基础测试来验证应用程序是否运行正常。

    <!-- lang: shell -->
    # does the application run
    add_test (TutorialRuns Tutorial 25)
     
    # does it sqrt of 25
    add_test (TutorialComp25 Tutorial 25)
     
    set_tests_properties (TutorialComp25 
      PROPERTIES PASS_REGULAR_EXPRESSION "25 is 5")
     
    # does it handle negative numbers
    add_test (TutorialNegative Tutorial -25)
    set_tests_properties (TutorialNegative
      PROPERTIES PASS_REGULAR_EXPRESSION "-25 is 0")
     
    # does it handle small numbers
    add_test (TutorialSmall Tutorial 0.0001)
    set_tests_properties (TutorialSmall
      PROPERTIES PASS_REGULAR_EXPRESSION "0.0001 is 0.01")
     
    # does the usage message work?
    add_test (TutorialUsage Tutorial)
    set_tests_properties (TutorialUsage
      PROPERTIES 
      PASS_REGULAR_EXPRESSION "Usage:.*number")

> The first test simply verifies that the application runs, does not
> segfault or otherwise crash, and has a zero return value. This is the
> basic form of a CTest test. The next few tests all make use of the
> PASS_REGULAR_EXPRESSION test property to verify that the output of the
> test contains certain strings. In this case verifying that the
> computed square root is what it should be and that the usage message
> is printed when an incorrect number of arguments are provided. If you
> wanted to add a lot of tests to test different input values you might
> consider creating a macro like the following:

第一个测试例子验证了程序运行，是否出现段错误或者崩溃，是否返回0值。这是一个`CTest`的基本测试。接下来的几个测试都利用了`PASS_REGULAR_EXPRESSION`来验证输出是否包含指定的字符串。在这种情况下，验证计算平方根是必要的，当提供了一个错误的参数时就打印用法信息。如果你想添加测试多个不同的输入值，可以考虑写一个像下面这样的宏：

    <!-- lang: shell -->
    #define a macro to simplify adding tests, then use it
    macro (do_test arg result)
      add_test (TutorialComp${arg} Tutorial ${arg})
      set_tests_properties (TutorialComp${arg}
        PROPERTIES PASS_REGULAR_EXPRESSION ${result})
    endmacro (do_test)
     
    # do a bunch of result based tests
    do_test (25 "25 is 5")
    do_test (-25 "-25 is 0")

> For each invocation of do_test, another test is added to the project
> with a name, input, and results based on the passed arguments.

对于每个`do_test`调用，都会根据传入其中的参数，在工程中添加名称、输入和结果。

# Adding System Introspection (Step 4) 添加系统反馈

> Next let us consider adding some code to our project that depends on
> features the target platform may not have. For this example we will
> add some code that depends on whether or not the target platform has
> the log and exp functions. Of course almost every platform has these
> functions but for this tutorial assume that they are less common. If
> the platform has log then we will use that to compute the square root
> in the mysqrt function. We first test for the availability of these
> functions using the CheckFunctionExists.cmake macro in the top level
> CMakeLists file as follows:

接下来让我们考虑在工程中添加一些代码，依赖目标平台上可能没有的特性。在这个例子中，我们将添加一些依赖于目标平台是否存在`log`和`exp`函数的代码。当然几乎所有的平台都有这些函数，但这个教程假设它们（这些函数）不常见。如果平台存在`log`，我们就在`mysqrt`中使用它来计算平方根。我们首先在顶级`CMakeLists`中使用`CheckFunctionExists.cmake`宏来测试这些函数的可用性：

    <!-- lang: shell -->
    # does this system provide the log and exp functions?
    include (CheckFunctionExists.cmake)
    check_function_exists (log HAVE_LOG)
    check_function_exists (exp HAVE_EXP)

> Next we modify the TutorialConfig.h.in to define those values if CMake
> found them on the platform as follows:

接下来我们修改`TutorialConfig.h.in`，如果CMake在平台上找到他们（函数），就定义这些值：

    <!-- lang: shell -->
    // does the platform provide exp and log functions?
    #cmakedefine HAVE_LOG
    #cmakedefine HAVE_EXP

> It is important that the tests for log and exp are done before the
> configure_file command for TutorialConfig.h. The configure_file
> command immediately configures the file using the current settings in
> CMake. Finally in the mysqrt function we can provide an alternate
> implementation based on log and exp if they are available on the
> system using the following code:

在为`TutorialConfig.h`执行`configure_file`命令之前完成对`log`和`exp`的测试非常重要。`configure_file`命令会使用CMake的当前设置立即配置这个文件。最后我们可以在`mysqrt`函数中提供基于`log`和`exp`的可选实现（如果他们在当前系统上可用）：

    <!-- lang: cpp -->
    // if we have both log and exp then use them
    #if defined (HAVE_LOG) && defined (HAVE_EXP)
      result = exp(log(x)*0.5);
    #else // otherwise use an iterative approach
      . . .

# Adding a Generated File and Generator (Step 5) 添加生成的文件和生成器

> In this section we will show how you can add a generated source file
> into the build process of an application. For this example we will
> create a table of precomputed square roots as part of the build
> process, and then compile that table into our application. To
> accomplish this we first need a program that will generate the table.
> In the MathFunctions subdirectory a new source file named
> MakeTable.cxx will do just that.

在这个单元中我们将为你展示如何在应用程序的构建过程中添加生成好的源文件。这个例子中我们将为构建过程造一个预计算平方根的表，然后将这个表编译进我们的应用程序。为了完成这个，我们首先需要一个可以生成这个表的程序。`MathFunctions`子目录下的`MakeTable.cxx`新源文件将完成这个工作：

    <!-- lang: cpp -->
    // A simple program that builds a sqrt table 
    #include <stdio.h>
    #include <stdlib.h>
    #include <math.h>
     
    int main (int argc, char *argv[])
    {
      int i;
      double result;
     
      // make sure we have enough arguments
      if (argc < 2)
        {
        return 1;
        }
      
      // open the output file
      FILE *fout = fopen(argv[1],"w");
      if (!fout)
        {
        return 1;
        }
      
      // create a source file with a table of square roots
      fprintf(fout,"double sqrtTable[] = {\n");
      for (i = 0; i < 10; ++i)
        {
        result = sqrt(static_cast<double>(i));
        fprintf(fout,"%g,\n",result);
        }
     
      // close the table with a zero
      fprintf(fout,"0};\n");
      fclose(fout);
      return 0;
    }

> Note that the table is produced as valid C++ code and that the name of
> the file to write the output to is passed in as an argument. The next
> step is to add the appropriate commands to MathFunctions’ CMakeLists
> file to build the MakeTable executable, and then run it as part of the
> build process. A few commands are needed to accomplish this, as shown
> below.

注意这个表格由C++代码产生，文件名作为参数传入以在里面输出结果。下一步是给`MathFunctions`的`CMakeLists`添加适当的命令来构建`MakeTable`程序。之后将作为构建过程的一部分来运行。需要很少的命令来完成这件事：

    <!-- lang: shell -->
    # first we add the executable that generates the table
    add_executable(MakeTable MakeTable.cxx)
     
    # add the command to generate the source code
    add_custom_command (
      OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/Table.h
      COMMAND MakeTable ${CMAKE_CURRENT_BINARY_DIR}/Table.h
      DEPENDS MakeTable
      )
     
    # add the binary tree directory to the search path for 
    # include files
    include_directories( ${CMAKE_CURRENT_BINARY_DIR} )
     
    # add the main library
    add_library(MathFunctions mysqrt.cxx ${CMAKE_CURRENT_BINARY_DIR}/Table.h  )

> First the executable for MakeTable is added as any other executable
> would be added. Then we add a custom command that specifies how to
> produce Table.h by running MakeTable. Next we have to let CMake know
> that mysqrt.cxx depends on the generated file Table.h. This is done by
> adding the generated Table.h to the list of sources for the library
> MathFunctions. We also have to add the current binary directory to the
> list of include directories so that Table.h can be found and included
> by mysqrt.cxx. When this project is built it will first build the
> MakeTable executable. It will then run MakeTable to produce Table.h.
> Finally, it will compile mysqrt.cxx which includes Table.h to produce
> the MathFunctions library. At this point the top level CMakeLists file
> with all the features we have added looks like the following:

首先，`MakeTable`像其他任何一个可执行程序一样被添加进入。然后，我们添加一个定制的命令来指定如何运行`MakeTable`来产生`Table.h`。之后，我们必须让CMake知道`mysqrt.cxx`依赖生成的`Table.h`文件。将生成的`Table.h`添加到`MathFunctions`源代码列表中来搞定它。同时我们也必须将当前二进制目录添加到包含目录列表中，使得`Table.h`可以被找到，以及被`mysqrt.cxx`包含。

但工程开始构建时，他首先会构建`MakeTable`程序，然后执行`MakeTable`来产生`Table.h`。最后，他会编译包含了`Table.h`的`mysqrt.cxx`来生成`MathFunctions`库。

至此，顶层的CMakeLists文件包含了我们之前添加的所有特性：

    <!-- lang: shell -->
    cmake_minimum_required (VERSION 2.6)
    project (Tutorial)
     
    # The version number.
    set (Tutorial_VERSION_MAJOR 1)
    set (Tutorial_VERSION_MINOR 0)
     
    # does this system provide the log and exp functions?
    include (${CMAKE_ROOT}/Modules/CheckFunctionExists.cmake)
     
    check_function_exists (log HAVE_LOG)
    check_function_exists (exp HAVE_EXP)
     
    # should we use our own math functions
    option(USE_MYMATH 
      "Use tutorial provided math implementation" ON)
     
    # configure a header file to pass some of the CMake settings
    # to the source code
    configure_file (
      "${PROJECT_SOURCE_DIR}/TutorialConfig.h.in"
      "${PROJECT_BINARY_DIR}/TutorialConfig.h"
      )
     
    # add the binary tree to the search path for include files
    # so that we will find TutorialConfig.h
    include_directories ("${PROJECT_BINARY_DIR}")
     
    # add the MathFunctions library?
    if (USE_MYMATH)
      include_directories ("${PROJECT_SOURCE_DIR}/MathFunctions")
      add_subdirectory (MathFunctions)
      set (EXTRA_LIBS ${EXTRA_LIBS} MathFunctions)
    endif (USE_MYMATH)
     
    # add the executable
    add_executable (Tutorial tutorial.cxx)
    target_link_libraries (Tutorial  ${EXTRA_LIBS})
     
    # add the install targets
    install (TARGETS Tutorial DESTINATION bin)
    install (FILES "${PROJECT_BINARY_DIR}/TutorialConfig.h"        
             DESTINATION include)
     
    # does the application run
    add_test (TutorialRuns Tutorial 25)
     
    # does the usage message work?
    add_test (TutorialUsage Tutorial)
    set_tests_properties (TutorialUsage
      PROPERTIES 
      PASS_REGULAR_EXPRESSION "Usage:.*number"
      )
     
     
    #define a macro to simplify adding tests
    macro (do_test arg result)
      add_test (TutorialComp${arg} Tutorial ${arg})
      set_tests_properties (TutorialComp${arg}
        PROPERTIES PASS_REGULAR_EXPRESSION ${result}
        )
    endmacro (do_test)
     
    # do a bunch of result based tests
    do_test (4 "4 is 2")
    do_test (9 "9 is 3")
    do_test (5 "5 is 2.236")
    do_test (7 "7 is 2.645")
    do_test (25 "25 is 5")
    do_test (-25 "-25 is 0")
    do_test (0.0001 "0.0001 is 0.01")

> TutorialConfig.h looks like:

`TutorialConfig.h`如下：

    <!-- lang: cpp -->
    // the configured options and settings for Tutorial
    #define Tutorial_VERSION_MAJOR @Tutorial_VERSION_MAJOR@
    #define Tutorial_VERSION_MINOR @Tutorial_VERSION_MINOR@
    #cmakedefine USE_MYMATH
     
    // does the platform provide exp and log functions?
    #cmakedefine HAVE_LOG
    #cmakedefine HAVE_EXP

> And the CMakeLists file for MathFunctions looks like:

`MathFunctions`的`CMakeLists`如下：

    <!-- lang: shell -->
    # first we add the executable that generates the table
    add_executable(MakeTable MakeTable.cxx)
    # add the command to generate the source code
    add_custom_command (
      OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/Table.h
      DEPENDS MakeTable
      COMMAND MakeTable ${CMAKE_CURRENT_BINARY_DIR}/Table.h
      )
    # add the binary tree directory to the search path 
    # for include files
    include_directories( ${CMAKE_CURRENT_BINARY_DIR} )
     
    # add the main library
    add_library(MathFunctions mysqrt.cxx ${CMAKE_CURRENT_BINARY_DIR}/Table.h)
     
    install (TARGETS MathFunctions DESTINATION bin)
    install (FILES MathFunctions.h DESTINATION include)

# Building an Installer (Step 6) 构建一个安装程序

> Next suppose that we want to distribute our project to other people so
> that they can use it. We want to provide both binary and source
> distributions on a variety of platforms. This is a little different
> from the instal we did previously in section Installing and Testing
> (Step 3), where we were installing the binaries that we had built from
> the source code. In this example we will be building installation
> packages that support binary installations and package management
> features as found in cygwin, debian, RPMs etc. To accomplish this we
> will use CPack to create platform specific installers as described in
> Chapter Packaging with CPack. Specifically we need to add a few lines
> to the bottom of our toplevel CMakeLists.txt file.

下面假设我们想发布我们的工程给其他人使用。我们想同时在多个平台上提供二进制程序和源代码。这和我们之前在第三部分`Installing and Testing`中做的安装（从源代码构建安装）有一点不同。这个例子中，我们将构建一个支持二进制安装和包管理特性的安装包，就像在`cygwin`、`debian`、`RPM`中找到的安装包一样。为了完成这件事，我们将利用在章节`Packaging with CPack`提到的`CPack`制作指定平台的安装程序。特别地，我们需要在顶层`CMakeLists`下面添加几行：

    <!-- lang: shell -->
    # build a CPack driven installer package
    include (InstallRequiredSystemLibraries)
    set (CPACK_RESOURCE_FILE_LICENSE  
         "${CMAKE_CURRENT_SOURCE_DIR}/License.txt")
    set (CPACK_PACKAGE_VERSION_MAJOR "${Tutorial_VERSION_MAJOR}")
    set (CPACK_PACKAGE_VERSION_MINOR "${Tutorial_VERSION_MINOR}")
    include (CPack)

> That is all there is to it. We start by including
> InstallRequiredSystemLibraries. This module will include any runtime
> libraries that are needed by the project for the current platform.
> Next we set some CPack variables to where we have stored the license
> and version information for this project. The version information
> makes use of the variables we set earlier in this tutorial. Finally we
> include the CPack module which will use these variables and some other
> properties of the system you are on to setup an installer. The next
> step is to build the project in the usual manner and then run CPack on
> it. To build a binary distribution you would run:

就是这样，先包含`InstallRequiredSystemLibraries`。这个模块将包含工程在当前平台需要的任何运行库。然后，我们设置一些`CPack`变量指向工程存放`license`和`version`信息的地方。版本信息利用了我们之前在教程里设置的变量。最后，我们包含的`CPack`模块将会使用这些变量，以及你所在系统的其他一些属性，来制作一个安装包。

下一步是按照平常的方式构建工程然后启动`CPack`。要发布一个二进制版，你需要执行：

    <!-- lang: shell -->
    cpack -C CPackConfig.cmake

> To create a source distribution you would type

要发布源代码，你需要执行：

    <!-- lang: shell -->
    cpack -C CPackSourceConfig.cmake

# Adding Support for a Dashboard (Step 7)  添加面板支持

> Adding support for submitting our test results to a dashboard is very
> easy. We already defined a number of tests for our project in the
> earlier steps of this tutorial. We just have to run those tests and
> submit them to a dashboard. To include support for dashboards we
> include the CTest module in our toplevel CMakeLists file.

为提交我们的测试结果添加面板支持十分简单。在之前的教程中，我们已经为工程定义了很多测试。我们仅仅需要执行这些测试以及把它们提交到面板。要包含面板支持我们需要在顶层`CMakeLists`中包含`CTest`模块。

    <!-- lang: shell -->
    # enable dashboard scripting
    include (CTest)

> We also create a CTestConfig.cmake file where we can specify the name
> of this project for the dashboard.

我们还为面板写了一个可以指定工程名的`CTestConfig.cmake`文件。

    <!-- lang: shell -->
    set (CTEST_PROJECT_NAME "Tutorial")

> CTest will read in this file when it runs. To create a simple
> dashboard you can run CMake on your project, change directory to the
> binary tree, and then run ctest –D Experimental. The results of your
> dashboard will be uploaded to Kitware's public dashboard here.

`CTest`将在运行时读取这个文件。要创建一个简单的面板，你可以在你的工程里运行CMake，将目录改变到二进制树，然后执行实验性的`ctest -D`。你面板上的结果将会被上传到`Kitware`的[公开面板][1]。

# 结束语

由于本人水平所限，译文和原文内容难免存在差异，甚至错误，还请各位读者评批指出！谢谢！

  [1]: http://www.cdash.org/CDash/index.php?project=PublicDashboard
