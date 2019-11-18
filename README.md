# A hacked-up, totally-not-safe-for-upstream LLVM fork

I found a very weird performance regression bug in Xcode that has been driving
me crazy for months. _So crazy_ that I started messing with `clang` myself to
see if I could figure out what's going on.

You can find the bug demonstrated in [liscio/SubFrameworks](https://github.com/liscio/SubFrameworks).

## How to use this fork/branch?

If you're interested in playing along, here's how I got everything going:

``` bash
# In some working folder, grab this branch/fork
git clone https://github.com/liscio/llvm-project
cd llvm-project
git checkout cl-more-module-remarks

# Now, create a build folder to work in
mkdir build
cd build

# Run cmake
cmake -G 'Unix Makefiles' \
    -DLLVM_ENABLE_PROJECTS="clang;libcxx;libcxxabi" \
    -DCMAKE_BUILD_TYPE=RelWithDebInfo -DLLVM_CREATE_XCODE_TOOLCHAIN=ON \
    ../llvm

# Build the Xcode toolchain
make -j40 install-xcode-toolchain

# Create a symlink so Xcode can see the new toolchain
sudo ln -s /usr/local/Toolchains/LLVM10.0.0git.xctoolchain /Library/Developer/Toolchains/LLVM10.0.0git.xctoolchain


```

## Using the custom toolchain

With `xcodebuild`, you can specify this toolchain at the command-line to avoid conflicts with your normal Xcode working setup.

For instance, using my reproducing test case, you can run this:

``` bash
xcodebuild -scheme MainFramework -derivedDataPath testDerivedData \
  -toolchain LLVM10.0.0git clean build \
  COMPILER_INDEX_STORE_ENABLE=0
```

The key bits are specifying the `-toolchain`, and
`COMPILER_INDEX_STORE_ENABLE=0`. The latter works around the fact that Xcode
relies on Apple's own fork of `clang` which has some slight differences of its
own.

## Filtering the output

I slapped timestamps on the remarks so that you could see how multiple `clang`
processes are contending with one another for access to the module cache.

To get usable output, I have been relying on my Vim-fu and using the following
commands:

```
g!/remark/d
```

This only shows the remarks.

```
%s/\(.*\) @ '\(\d\+\)'/\2 \1/g
```

This puts the timestamp at the beginning of each line.

```
%sort
```

And this sorts the line in the file.

(Yes, I could probably chain these together using `sed`, but I've already spent
enough time on this. :P)

## Sample Output

This is further filtered to aid readability, but the following contains output
from my sample project.

In a nutshell, it looks like all the `clang` processes find the dirtied module
in the cache (because of the clean rebuild). Then, they fight for the right to
build the new item in the cache.

The first process wins the lock, and starts building the module.

The next process comes along, but when it loads the just-built module, it is
mis-identified as dirty!

This repeats for each `clang` invocation that depends on the same module. Ugh.

```
20191118112954791694000 SubFrameworks/SubC.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
20191118112954792319000 SubFrameworks/SubA.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
20191118112954792519000 SubFrameworks/SubB.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
20191118112954841476000 SubFrameworks/SubA.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
20191118112954841521000 SubFrameworks/SubB.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
20191118112954841526000 SubFrameworks/SubC.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
20191118112954859879000 SubFrameworks/SubC.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
20191118112954860010000 SubFrameworks/SubB.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
20191118112954860233000 SubFrameworks/SubA.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
20191118112954996015000 MainFramework/MainB.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
20191118112954997451000 MainFramework/MainD.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
20191118112954997610000 MainFramework/MainG.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
20191118112954998312000 MainFramework/MainF.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
20191118112954999188000 MainFramework/MainA.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
20191118112954999794000 MainFramework/MainE.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
20191118112955120641000 MainFramework/MainB.h:10:9: remark: finished building module 'SubFrameworks' [-Rmodule-build]
20191118112955122549000 MainFramework/MainB.h:10:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
20191118112955125256000 MainFramework/MainG.h:10:9: remark: [compileAndLoadModule] Module 'SubFrameworks' (which we waited for another process to build) was marked out of date [-Rmodule-build]
20191118112955125290000 MainFramework/MainG.h:10:9: remark: [compileAndLoadModule] Loop iteration '2' for pid '90724' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
20191118112955125584000 MainFramework/MainG.h:10:9: remark: [compileAndLoadModule] taking on responsibility of building 'SubFrameworks' [-Rmodule-build]
20191118112955125728000 MainFramework/MainG.h:10:9: remark: building module 'SubFrameworks' as 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
20191118112955127248000 SubFrameworks/SubFrameworks.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
20191118112955133979000 SubFrameworks/SubFrameworks.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
20191118112955149600000 SubFrameworks/SubFrameworks.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
20191118112955171072000 MainFramework/MainG.h:10:9: remark: finished building module 'SubFrameworks' [-Rmodule-build]
20191118112955173989000 MainFramework/MainG.h:10:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
20191118112955178635000 MainFramework/MainE.h:10:9: remark: [compileAndLoadModule] Module 'SubFrameworks' (which we waited for another process to build) was marked out of date [-Rmodule-build]
20191118112955178672000 MainFramework/MainE.h:10:9: remark: [compileAndLoadModule] Loop iteration '2' for pid '90727' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
20191118112955179067000 MainFramework/MainE.h:10:9: remark: [compileAndLoadModule] taking on responsibility of building 'SubFrameworks' [-Rmodule-build]
20191118112955179227000 MainFramework/MainE.h:10:9: remark: building module 'SubFrameworks' as 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
20191118112955180907000 SubFrameworks/SubFrameworks.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
20191118112955187664000 SubFrameworks/SubFrameworks.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
20191118112955202886000 SubFrameworks/SubFrameworks.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
20191118112955220128000 MainFramework/MainD.h:10:9: remark: [compileAndLoadModule] The lock owner died for module 'SubFrameworks' [-Rmodule-build]
20191118112955220166000 MainFramework/MainD.h:10:9: remark: [compileAndLoadModule] Loop iteration '2' for pid '90723' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
20191118112955223613000 MainFramework/MainE.h:10:9: remark: finished building module 'SubFrameworks' [-Rmodule-build]
20191118112955225891000 MainFramework/MainE.h:10:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
20191118112955228772000 MainFramework/MainD.h:10:9: remark: [compileAndLoadModule] Module 'SubFrameworks' (which we waited for another process to build) was marked out of date [-Rmodule-build]
20191118112955228804000 MainFramework/MainD.h:10:9: remark: [compileAndLoadModule] Loop iteration '3' for pid '90723' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
20191118112955229090000 MainFramework/MainD.h:10:9: remark: [compileAndLoadModule] taking on responsibility of building 'SubFrameworks' [-Rmodule-build]
20191118112955229228000 MainFramework/MainD.h:10:9: remark: building module 'SubFrameworks' as 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
20191118112955230801000 SubFrameworks/SubFrameworks.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
20191118112955237161000 SubFrameworks/SubFrameworks.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
20191118112955243649000 MainFramework/MainC.h:10:9: remark: [compileAndLoadModule] The lock owner died for module 'SubFrameworks' [-Rmodule-build]
20191118112955243689000 MainFramework/MainC.h:10:9: remark: [compileAndLoadModule] Loop iteration '2' for pid '90728' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
20191118112955251529000 SubFrameworks/SubFrameworks.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
20191118112955258655000 MainFramework/MainA.h:10:9: remark: [compileAndLoadModule] The lock owner died for module 'SubFrameworks' [-Rmodule-build]
20191118112955258694000 MainFramework/MainA.h:10:9: remark: [compileAndLoadModule] Loop iteration '2' for pid '90726' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
20191118112955272420000 MainFramework/MainD.h:10:9: remark: finished building module 'SubFrameworks' [-Rmodule-build]
20191118112955274820000 MainFramework/MainD.h:10:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
20191118112955306748000 MainFramework/MainA.h:10:9: remark: [compileAndLoadModule] Module 'SubFrameworks' (which we waited for another process to build) was marked out of date [-Rmodule-build]
20191118112955306785000 MainFramework/MainA.h:10:9: remark: [compileAndLoadModule] Loop iteration '3' for pid '90726' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
20191118112955307094000 MainFramework/MainA.h:10:9: remark: [compileAndLoadModule] taking on responsibility of building 'SubFrameworks' [-Rmodule-build]
20191118112955307235000 MainFramework/MainA.h:10:9: remark: building module 'SubFrameworks' as 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
20191118112955308928000 SubFrameworks/SubFrameworks.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
20191118112955315252000 SubFrameworks/SubFrameworks.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
20191118112955315474000 MainFramework/MainF.h:10:9: remark: [compileAndLoadModule] The lock owner died for module 'SubFrameworks' [-Rmodule-build]
20191118112955315513000 MainFramework/MainF.h:10:9: remark: [compileAndLoadModule] Loop iteration '2' for pid '90725' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
20191118112955325327000 MainFramework/MainC.h:10:9: remark: [compileAndLoadModule] The lock owner died for module 'SubFrameworks' [-Rmodule-build]
20191118112955325366000 MainFramework/MainC.h:10:9: remark: [compileAndLoadModule] Loop iteration '3' for pid '90728' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
20191118112955329311000 SubFrameworks/SubFrameworks.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
20191118112955349386000 MainFramework/MainA.h:10:9: remark: finished building module 'SubFrameworks' [-Rmodule-build]
20191118112955351444000 MainFramework/MainA.h:10:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
20191118112955353470000 MainFramework/MainF.h:10:9: remark: [compileAndLoadModule] Module 'SubFrameworks' (which we waited for another process to build) was marked out of date [-Rmodule-build]
20191118112955353502000 MainFramework/MainF.h:10:9: remark: [compileAndLoadModule] Loop iteration '3' for pid '90725' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
20191118112955353784000 MainFramework/MainF.h:10:9: remark: [compileAndLoadModule] taking on responsibility of building 'SubFrameworks' [-Rmodule-build]
20191118112955353924000 MainFramework/MainF.h:10:9: remark: building module 'SubFrameworks' as 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
20191118112955355357000 SubFrameworks/SubFrameworks.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
20191118112955361820000 SubFrameworks/SubFrameworks.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
20191118112955377044000 SubFrameworks/SubFrameworks.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
20191118112955398247000 MainFramework/MainF.h:10:9: remark: finished building module 'SubFrameworks' [-Rmodule-build]
20191118112955400540000 MainFramework/MainF.h:10:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
20191118112955534232000 MainFramework/MainC.h:10:9: remark: [compileAndLoadModule] Module 'SubFrameworks' (which we waited for another process to build) was marked out of date [-Rmodule-build]
20191118112955534298000 MainFramework/MainC.h:10:9: remark: [compileAndLoadModule] Loop iteration '4' for pid '90728' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
20191118112955534763000 MainFramework/MainC.h:10:9: remark: [compileAndLoadModule] taking on responsibility of building 'SubFrameworks' [-Rmodule-build]
20191118112955534977000 MainFramework/MainC.h:10:9: remark: building module 'SubFrameworks' as 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
20191118112955536652000 SubFrameworks/SubFrameworks.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
20191118112955542624000 SubFrameworks/SubFrameworks.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
2019111811295554376000 MainFramework/MainB.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
20191118112955557052000 SubFrameworks/SubFrameworks.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
2019111811295556048000 MainFramework/MainD.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
2019111811295556588000 MainFramework/MainG.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
2019111811295556820000 MainFramework/MainA.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
2019111811295557255000 MainFramework/MainF.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
20191118112955576951000 MainFramework/MainC.h:10:9: remark: finished building module 'SubFrameworks' [-Rmodule-build]
2019111811295557773000 MainFramework/MainE.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
20191118112955579030000 MainFramework/MainC.h:10:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295558081000 MainFramework/MainC.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
20191118112955679000 MainFramework/MainC.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
2019111811295571834000 MainFramework/MainB.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
2019111811295572725000 MainFramework/MainB.h:10:9: remark: [CompilerInstance::loadModule] Sourcing module 'SubFrameworks' from cache [-Rmodule-build]
2019111811295573106000 MainFramework/MainB.h:10:2: remark: [ASTReader::getInputFile] file 'testDerivedData/Build/Intermediates.noindex/SubFrameworks.build/Debug/SubFrameworks.build/module.modulemap' has been modified since the module file 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' was built: mtime changed [-Rmodule-build]
2019111811295573165000 MainFramework/MainB.h:10:2: remark: [ASTReader::ReadControlBlock] control block file out of date 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295573198000 MainFramework/MainB.h:10:2: remark: [ASTReader::ReadASTCore] control block returned out of date 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295573234000 MainFramework/MainB.h:10:2: remark: [ASTReader::ReadAST] removing out of date module 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295573298000 MainFramework/MainB.h:10:9: remark: [CompilerInstance::loadModule] cached module 'SubFrameworks' was out of date [-Rmodule-build]
2019111811295573376000 MainFramework/MainB.h:10:9: remark: [compileAndLoadModule] Loop iteration '1' for pid '90722' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
2019111811295574351000 MainFramework/MainB.h:10:9: remark: [compileAndLoadModule] taking on responsibility of building 'SubFrameworks' [-Rmodule-build]
2019111811295574546000 MainFramework/MainB.h:10:9: remark: building module 'SubFrameworks' as 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295576739000 SubFrameworks/SubFrameworks.h:9:9: remark: [CompilerInstance::loadModule] Sourcing module 'Foundation' from cache [-Rmodule-build]
2019111811295577513000 MainFramework/MainD.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
2019111811295578408000 MainFramework/MainD.h:10:9: remark: [CompilerInstance::loadModule] Sourcing module 'SubFrameworks' from cache [-Rmodule-build]
2019111811295578761000 MainFramework/MainD.h:10:2: remark: [ASTReader::getInputFile] file 'testDerivedData/Build/Intermediates.noindex/SubFrameworks.build/Debug/SubFrameworks.build/module.modulemap' has been modified since the module file 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' was built: mtime changed [-Rmodule-build]
2019111811295578812000 MainFramework/MainD.h:10:2: remark: [ASTReader::ReadControlBlock] control block file out of date 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295578868000 MainFramework/MainD.h:10:2: remark: [ASTReader::ReadASTCore] control block returned out of date 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295578922000 MainFramework/MainD.h:10:2: remark: [ASTReader::ReadAST] removing out of date module 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295578972000 MainFramework/MainD.h:10:9: remark: [CompilerInstance::loadModule] cached module 'SubFrameworks' was out of date [-Rmodule-build]
2019111811295579030000 MainFramework/MainD.h:10:9: remark: [compileAndLoadModule] Loop iteration '1' for pid '90723' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
2019111811295579064000 MainFramework/MainG.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
2019111811295579407000 MainFramework/MainA.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
2019111811295580057000 MainFramework/MainG.h:10:9: remark: [CompilerInstance::loadModule] Sourcing module 'SubFrameworks' from cache [-Rmodule-build]
2019111811295580337000 MainFramework/MainA.h:10:9: remark: [CompilerInstance::loadModule] Sourcing module 'SubFrameworks' from cache [-Rmodule-build]
2019111811295580409000 MainFramework/MainF.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
2019111811295580445000 MainFramework/MainG.h:10:2: remark: [ASTReader::getInputFile] file 'testDerivedData/Build/Intermediates.noindex/SubFrameworks.build/Debug/SubFrameworks.build/module.modulemap' has been modified since the module file 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' was built: mtime changed [-Rmodule-build]
2019111811295580511000 MainFramework/MainG.h:10:2: remark: [ASTReader::ReadControlBlock] control block file out of date 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295580542000 MainFramework/MainG.h:10:2: remark: [ASTReader::ReadASTCore] control block returned out of date 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295580573000 MainFramework/MainG.h:10:2: remark: [ASTReader::ReadAST] removing out of date module 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295580628000 MainFramework/MainG.h:10:9: remark: [CompilerInstance::loadModule] cached module 'SubFrameworks' was out of date [-Rmodule-build]
2019111811295580684000 MainFramework/MainG.h:10:9: remark: [compileAndLoadModule] Loop iteration '1' for pid '90724' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
2019111811295580724000 MainFramework/MainA.h:10:2: remark: [ASTReader::getInputFile] file 'testDerivedData/Build/Intermediates.noindex/SubFrameworks.build/Debug/SubFrameworks.build/module.modulemap' has been modified since the module file 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' was built: mtime changed [-Rmodule-build]
2019111811295580789000 MainFramework/MainA.h:10:2: remark: [ASTReader::ReadControlBlock] control block file out of date 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295580803000 MainFramework/MainE.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
2019111811295580825000 MainFramework/MainA.h:10:2: remark: [ASTReader::ReadASTCore] control block returned out of date 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295580862000 MainFramework/MainA.h:10:2: remark: [ASTReader::ReadAST] removing out of date module 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295580913000 MainFramework/MainA.h:10:9: remark: [CompilerInstance::loadModule] cached module 'SubFrameworks' was out of date [-Rmodule-build]
2019111811295580927000 MainFramework/MainC.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
2019111811295580969000 MainFramework/MainA.h:10:9: remark: [compileAndLoadModule] Loop iteration '1' for pid '90726' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
2019111811295581292000 MainFramework/MainF.h:10:9: remark: [CompilerInstance::loadModule] Sourcing module 'SubFrameworks' from cache [-Rmodule-build]
2019111811295581665000 MainFramework/MainF.h:10:2: remark: [ASTReader::getInputFile] file 'testDerivedData/Build/Intermediates.noindex/SubFrameworks.build/Debug/SubFrameworks.build/module.modulemap' has been modified since the module file 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' was built: mtime changed [-Rmodule-build]
2019111811295581731000 MainFramework/MainF.h:10:2: remark: [ASTReader::ReadControlBlock] control block file out of date 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295581732000 MainFramework/MainE.h:10:9: remark: [CompilerInstance::loadModule] Sourcing module 'SubFrameworks' from cache [-Rmodule-build]
2019111811295581763000 MainFramework/MainF.h:10:2: remark: [ASTReader::ReadASTCore] control block returned out of date 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295581767000 MainFramework/MainC.h:10:9: remark: [CompilerInstance::loadModule] Sourcing module 'SubFrameworks' from cache [-Rmodule-build]
2019111811295581819000 MainFramework/MainF.h:10:2: remark: [ASTReader::ReadAST] removing out of date module 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295581881000 MainFramework/MainF.h:10:9: remark: [CompilerInstance::loadModule] cached module 'SubFrameworks' was out of date [-Rmodule-build]
2019111811295581946000 MainFramework/MainF.h:10:9: remark: [compileAndLoadModule] Loop iteration '1' for pid '90725' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
2019111811295582147000 MainFramework/MainC.h:10:2: remark: [ASTReader::getInputFile] file 'testDerivedData/Build/Intermediates.noindex/SubFrameworks.build/Debug/SubFrameworks.build/module.modulemap' has been modified since the module file 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' was built: mtime changed [-Rmodule-build]
2019111811295582180000 MainFramework/MainE.h:10:2: remark: [ASTReader::getInputFile] file 'testDerivedData/Build/Intermediates.noindex/SubFrameworks.build/Debug/SubFrameworks.build/module.modulemap' has been modified since the module file 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' was built: mtime changed [-Rmodule-build]
2019111811295582203000 MainFramework/MainC.h:10:2: remark: [ASTReader::ReadControlBlock] control block file out of date 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295582234000 MainFramework/MainC.h:10:2: remark: [ASTReader::ReadASTCore] control block returned out of date 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295582243000 MainFramework/MainE.h:10:2: remark: [ASTReader::ReadControlBlock] control block file out of date 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295582268000 MainFramework/MainC.h:10:2: remark: [ASTReader::ReadAST] removing out of date module 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295582274000 MainFramework/MainE.h:10:2: remark: [ASTReader::ReadASTCore] control block returned out of date 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295582309000 MainFramework/MainE.h:10:2: remark: [ASTReader::ReadAST] removing out of date module 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/SubFrameworks-3U5BOV1R8307A.pcm' [-Rmodule-build]
2019111811295582316000 MainFramework/MainC.h:10:9: remark: [CompilerInstance::loadModule] cached module 'SubFrameworks' was out of date [-Rmodule-build]
2019111811295582355000 MainFramework/MainE.h:10:9: remark: [CompilerInstance::loadModule] cached module 'SubFrameworks' was out of date [-Rmodule-build]
2019111811295582411000 MainFramework/MainC.h:10:9: remark: [compileAndLoadModule] Loop iteration '1' for pid '90728' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
2019111811295582412000 MainFramework/MainE.h:10:9: remark: [compileAndLoadModule] Loop iteration '1' for pid '90727' while compiling/loading module 'SubFrameworks' [-Rmodule-build]
2019111811295585072000 SubFrameworks/SubFrameworks.h:9:2: remark: [ASTReader::ReadAST] successfully loaded 'testDerivedData/ModuleCache.noindex/4ZNU7QMPSADY/Foundation-2PSPYZLNW14KI.pcm' [-Rmodule-build]
2019111811295599758000 SubFrameworks/SubFrameworks.h:9:9: remark: [CompilerInstance::loadModule] successfully read 'Foundation' from the cache [-Rmodule-build]
```

## Other notes

I have also *messed around* with other bits of the code, and hopefully by the
time you see this I have backed out the more dangerous of those hacks that I
made in an attempt to influence `clang`'s behavior.

Some things I tried:

* "Fixing" the exponential backoff strategy in the `LockFileManager` to
  introduce some randomness in the delay times. (Yes, I am aware that I
  am using terrible RNG code.)

* Forcing the `ASTReader` to check file contents when the `mtime` has changed.

Clearly I am *way* out of my lane here, but I figured I'd poke the bear a
little bit to see how it changed its behavior.


## Closing thoughts

I hope that my random prodding about will somehow help the developer(s)
tasked with resolving these issues. 

There are probably other things I should have been doing instead of poking at
`clang`, but this is a *really nasty issue* that is costing me significant
amounts of wasted time waiting for builds to complete, so I hoped that perhaps
I could speed up the effort by shining a spotlight on this.

Also, it's very likely that Xcode (or the file system) is responsible for this
error, and not `clang`. The fact that "dirtiness" seems to be defined as the
source `modulemap` file's `mtime` changing after every successful dependency
module build raises a red flag for me.

Good luck!
