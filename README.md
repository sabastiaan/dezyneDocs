# dezyneDocs

##This document is meant to describe the path of getting any dzn file compiled on an raspberry pi
I will not assume any knowlegde of how UNIX (the OS category which raspbian, the os of your raspberry pi, belong to)
nor will I assume any other pre-knowlegde besides basic OOP which was given in programming 1.
However I will assume that you are able to install raspbian on your pi, if not go ask somebody/ask the internet, they can help you better
than I would. 

You can structure your workflow in a multitude of ways, there are other ways possible than I will describe.
At the bottom I will expand on some assumptions I made.
This tutorial will use your pi to compile your programs. 

Grab your terminal and let's get started shall we! :D 

## SSH into your pi
Sorry, you have to google this one

### Getting dzn to run

First we need to install dzn onto your pi. 

#### NPM 
Dezyne cli needs npm 6.10 installed onto your system, otherwise it will not work.
The default repository has a differnt version, so will use [nvm (node version manager)](https://github.com/nvm-sh/nvm)
```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.3/install.sh | bash
source ~/.bashrc

nvm install 6.10
nvm use 6.10
nvm default alias 6.10

echo prefix=~/.npm > ~/.npmrc
PATH=~/.npm/bin:$PATH

source ~/.bashrc
nvm use --delete-prefix v6.10.3 --silent
source ~/.bashrc
```

Next up we will download dezyne from here:
```
npm install -g https://dezyne.verum.com/download/npm/dzn-2.9.1.tar.gz
```

Dezyne sends all of your files to their servers to validate them over there, hence we always need to autheticate first.
If we will not do this we will just get some js/node errors:


```
dzn -s https://dezyne.verum.com/service/2.9 -u "< user_email >" -p hello
```

Test and see if ```dzn --version``` lets you know you are running dzn 2.9.1


In case you haven't you should also have ```make``` and ```g++``` on your system. You probably also want wiring pi:  ```apt install g++ make wiringpi```
 
### Using dezyne

Most likely you have made your dezyne model on your local machine. 
Maybe you have generated code but didn't compile it yet (and if you did most likely didn't do it with a cross-compiler).
If not don't worry, we will do it all again on your pi!

I heavily suggest using git, go ahead and pull your dezyne code onto your machine.

You are now in your code directory with a bunch of dzn files.
This is the place where you might start using the make file found in their tutorials. However in this repository there is a slightly
altered Makefile which keeps things cleaner.

#### Why should we use makefiles?
Makefiles can been as generators of a bunch of shell commands that deal with setting up everything so you can compile with just a single command. Which is something that shouldn't be taken for granted!!! Copy your desired makefile into the root of your directory. Make files are required to be named ```[M/m]akefile``` and can be invoked with ```make <command>```. It is customary the invoking make without any command should build your executable (and all of that what it requires to do so).

The Dezyne makefile contains a few commands

```make runtime```
This will download the required runtime for dezyne

```make generate```
This will take all of the .dzn files in your current directory and will generate c++ code in the src/ (or if you are using the edited makefile into src_dzn)

```make```
Just using make will try to build the application

```make clean```
This will remove compile artifacts

```make clean_generated```
This will remove dzn generation junk, as well as the runtime.


### Compiling a pure dezyne program
Execute the first 3 commands listed above

Eventually you will need to integrate the native c++ code you wrote into design. 
To let design know you want to use your own implementation you need to have a dezyne ```component``` with an empty ```behaviour```
Example:
```
interface ISystem{
  in void enable();
}

component System{
  provides ISystem sys;
  
  behaviour{
  }
}
```
It is important to prefix your interface with an I due to how dezyne chooses names for their generated files (citiation needed).

For each given dezyne file it will generate an <name>.hh and <name>.cc
Inside of the .hh you can find the declaration of for example your component (and hence the interface).
Typically the .cc will be filled with code if you gave the behaviour an implementation. However if it body is empty
 it will also generate an empty .cc file. You shouldn't put code in here because of the risk of being overwritten by the code generator.


Now create a header file (ending with .hh) with the same name as your interface, but without the I prefix in your source directory.
Include the dezyne file with ```#include "[src/src_dzn]/ISystem.hh"```
Only the header file needs to be included. Keep in mind that your implementation might be included more than once!
Because of this you want to wrap your file with [header guards](https://stackoverflow.com/a/4767305/8668536) 

#### Implementing functions declared in dezyne with c++

Create a class with the same name as your component and inherit the dezyne generated class.
``` class System : skell::System { } ``` 
By convention dezyne will generate all their code in the skell namespace. A namespace is like a special name prefix but then baked into the language so you can do more fancy stuff with it, just ignore it for now.

If you would compile your program now it would not work. 
This is because the new class misses still a few things:
1. the class should have a constructor accepting the dzn locator 
2. it *requires* you to implement all of the functions specified in the interface

The first one is eassily fixed:
``` class System : skell::System { 
public:
  System(const dzn::locator& loc); 
} 
``` 
The second one be done in two ways:

1. inline
```
//System.hh
#ifndef __System.hh__
#define __System.hh__

class System : skell::System { 
private:
 void enable() { /* implementation */ } 
 // the next function does not need the private modifier again

public:
  System(const dzn::locator& loc); 
} 
#endif
``` 

2. split up
C and C++ allow you to declare and implement a function in two different places as long as the declaration precedes it.
```
int sum(int a, int b); //No body

int sum(int a, int b) { return a +b } // might be a different file
```
This means we can also split up our implementation of our code by adding an extra file with a ```.cc``` extension like so

```
//System.cc
//no header guards are required because this file won't be included by others
#include "System.hh"

System::enable() {

}
```


What is left is the code initialization of dezyne, which is best explained [here](https://www.verum.com/dzndoc/tutorials-code-integration-tutorial-expanding-the-alarmsystem/)
