# CMake kernel module template

My unsatiated need for autocomplete has lead me to this.


# Usage
1. Replace the names, and maybe the kernel header path in `CMakeLists.txt`
2. Create a build directory. For example `mkdir build`
3. `cd` into the build directory
4. Run `cmake ..`
5. Run `make`. This should build the dummy library. From now on autocomplete should be working.
6. Run `make module`. This will build your module.
7. Happy hacking!


# References
Shamelessly frankensteining these two resources:
- [https://github.com/enginning/cmake-kernel-module]
- [https://musteresel.github.io/posts/2020/02/cmake-template-linux-kernel-module.html]
