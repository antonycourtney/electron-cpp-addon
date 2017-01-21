# electron-cpp-addon

This is a minimal repro to demonstrate an issue with building native addons for Electron that use standard C++ headers and libraries.

It's based on the simplest ["hello_world" addon](https://github.com/nodejs/node-addon-examples/tree/master/1_hello_world/nan) from [node-addon-examples](https://github.com/nodejs/node-addon-examples]) with a one-line change to include the C++11 standard include file `<regex>`.

# Issue

## What Happens:

On my system (A Mac running macOS Sierra 10.12.2) this addon compiles with no warnings or errors when using `node-gyp` to build for node.js v7.1.0.

Rebuilding this addon for electron.js following the instructions in [Using Native Node Modules](https://github.com/electron/electron/blob/master/docs/tutorial/using-native-node-modules.md) fails with an error indicating the the standard C++ header file `<regex>` is not found.

## What I'd expect to happen:

I would expect this addon to compile without warnings or errors when building for electron.

# Instructions to Reproduce the Issue

## Build for Node.js

## Attempt to Build for Electron

# Analysis and Workaround
