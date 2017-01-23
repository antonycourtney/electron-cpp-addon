# electron-cpp-addon

This is a minimal repro to demonstrate an issue with building native addons for Electron that use standard C++ headers and libraries.

It's based on the simplest ["hello_world" addon](https://github.com/nodejs/node-addon-examples/tree/master/1_hello_world/nan) from [node-addon-examples](https://github.com/nodejs/node-addon-examples) with a one-line change to include the C++11 standard include file `<regex>`.

# Issue

## What Happens:

On my system (A Mac running macOS Sierra 10.12.2) this addon compiles with no warnings or errors when using `node-gyp` to build for node.js v7.1.0.

Rebuilding this addon for electron.js following the instructions in [Using Native Node Modules](https://github.com/electron/electron/blob/master/docs/tutorial/using-native-node-modules.md) fails with an error indicating the the standard C++ header file `<regex>` is not found.

## What I'd expect to happen:

I would expect this addon to compile without warnings or errors when building for electron.

# Instructions to Reproduce the Issue

## Clone This Repo and and Build for Node.js

```sh
$ git clone git@github.com:antonycourtney/electron-cpp-addon.git
[...]
$ cd electron-cpp-addon
$ npm install
```

The final command should produce output something like the following:

```sh
> hello_world@0.0.0 install /private/tmp/electron-cpp-addon
> node-gyp rebuild

  CXX(target) Release/obj.target/hello/hello.o
  SOLINK_MODULE(target) Release/hello.node
hello_world@0.0.0 /private/tmp/electron-cpp-addon
├── bindings@1.2.1
└── nan@2.5.0
$
```
Confirm that this built succesfully:

```sh
$ node --version
v7.1.0
$ node hello.js
world
$
```

( You can also run the commmand `node-gyp rebuild` manually from the shell any time after `npm install` to recompile the addon for node.)

## Attempt to Build for Electron

Following the approach recommended in [Using Native Node Modules](https://github.com/electron/electron/blob/master/docs/tutorial/using-native-node-modules.md), attempt to run `node-gyp` to rebuild this addon for Electron:

```sh
$ HOME=~/.electron-gyp node-gyp rebuild --target=1.4.15 --arch=x64
```

On my machine, this produces the following output:

```
gyp info it worked if it ends with ok
gyp info using node-gyp@3.5.0
gyp info using node@7.1.0 | darwin | x64
gyp info spawn /Users/antony/anaconda/bin/python2
gyp info spawn args [ '/usr/local/lib/node_modules/node-gyp/gyp/gyp_main.py',
gyp info spawn args   'binding.gyp',
gyp info spawn args   '-f',
gyp info spawn args   'make',
gyp info spawn args   '-I',
gyp info spawn args   '/private/tmp/electron-cpp-addon/build/config.gypi',
gyp info spawn args   '-I',
gyp info spawn args   '/usr/local/lib/node_modules/node-gyp/addon.gypi',
gyp info spawn args   '-I',
gyp info spawn args   '/Users/antony/.electron-gyp/.node-gyp/iojs-1.4.15/common.gypi',
gyp info spawn args   '-Dlibrary=shared_library',
gyp info spawn args   '-Dvisibility=default',
gyp info spawn args   '-Dnode_root_dir=/Users/antony/.electron-gyp/.node-gyp/iojs-1.4.15',
gyp info spawn args   '-Dnode_gyp_dir=/usr/local/lib/node_modules/node-gyp',
gyp info spawn args   '-Dnode_lib_file=iojs.lib',
gyp info spawn args   '-Dmodule_root_dir=/private/tmp/electron-cpp-addon',
gyp info spawn args   '--depth=.',
gyp info spawn args   '--no-parallel',
gyp info spawn args   '--generator-output',
gyp info spawn args   'build',
gyp info spawn args   '-Goutput_dir=.' ]
gyp info spawn make
gyp info spawn args [ 'BUILDTYPE=Release', '-C', 'build' ]
  CXX(target) Release/obj.target/hello/hello.o
../hello.cc:2:10: fatal error: 'regex' file not found
#include <regex>
         ^
1 error generated.
make: *** [Release/obj.target/hello/hello.o] Error 1
gyp ERR! build error
gyp ERR! stack Error: `make` failed with exit code: 2
gyp ERR! stack     at ChildProcess.onExit (/usr/local/lib/node_modules/node-gyp/lib/build.js:276:23)
gyp ERR! stack     at emitTwo (events.js:106:13)
gyp ERR! stack     at ChildProcess.emit (events.js:191:7)
gyp ERR! stack     at Process.ChildProcess._handle.onexit (internal/child_process.js:215:12)
gyp ERR! System Darwin 16.3.0
gyp ERR! command "/usr/local/bin/node" "/usr/local/bin/node-gyp" "rebuild" "--target=1.4.15" "--arch=x64"
gyp ERR! cwd /private/tmp/electron-cpp-addon
gyp ERR! node -v v7.1.0
gyp ERR! node-gyp -v v3.5.0
gyp ERR! not ok
$
```

# Analysis and Workaround

If we run the `node-gyp rebuild` command manually to build for node.js, we see that the `common.gypi` file used by node-gyp when building for node (7.1.0 on my system) is `~/.node-gyp/7.1.0/include/node/common.gypi
` whereas when we build for electron, we are using `~/.electron-gyp/.node-gyp/iojs-1.4.14/common.gypi`.


digging in to these two files and comparing the differences, we see that around line 387 in node's common.gypi we have:

```
['clang==1', {
  'xcode_settings': {
    'GCC_VERSION': 'com.apple.compilers.llvm.clang.1_0',
    'CLANG_CXX_LANGUAGE_STANDARD': 'gnu++0x',  # -std=gnu++0x
    'CLANG_CXX_LIBRARY': 'libc++',
  },
}],
```

The corresponding fragment from electron's version of the same file is almost identical, but missing the `CLANG_CXX_LIBRARY` definition:

```
['clang==1', {
  'xcode_settings': {
    'GCC_VERSION': 'com.apple.compilers.llvm.clang.1_0',
    'CLANG_CXX_LANGUAGE_STANDARD': 'gnu++0x',  # -std=gnu++0x
  },
}],
```

## Workaround

We can get this to compile if we add the following fragment to `binding.gyp` in this project:

```
"conditions": [
  [ 'OS=="mac"', {
    'xcode_settings': {
      'CLANG_CXX_LIBRARY': 'libc++'
    }
}]]
```

This should be added as a property of the object in the `targets` array, so that the complete `binding.gyp` becomes:

```
{
  "targets": [
    {
      "target_name": "hello",
      "sources": [ "hello.cc" ],
      "include_dirs": [
        "<!(node -e \"require('nan')\")"
      ],
      "conditions": [
        [ 'OS=="mac"', {
          'xcode_settings': {
            'CLANG_CXX_LIBRARY': 'libc++'
          }
      }]]
    }
  ]
}
```
