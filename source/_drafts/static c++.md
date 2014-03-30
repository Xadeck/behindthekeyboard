Many design-patterns works by letting the user <em>register</em> objects or functions, that extend or enrich a functionality. It can for example be a new factory, or a listener. It usuallly works in two steps:
<ol>
  <li>first you derive from an interface and implement your custom behavior, as a function or a class;</li>
  <li>then you make an explicit call to register the fonction, or an instance of the class.</li>
</ol>
My concern is that it requires you to work in two places (the .h and .cpp file in which you perform step 1, and the place in the program initialization where you perform step 2). I wanted to keep this localized. That is, I wanted my fellow co-workers to be able to extend the system by simply adding a .cpp file to the build system. I wanted the mere fact of compiling this file and linking it to the executable to automatically add the coded functionalities to the system.

<!--more-->
The right answer to do this is to implement a plugin system: when run, the main program browse a plugin directory, load all the plugins it found, which performs step 2. Hence, the programmer works locally, creating its own little project that just needs to be compiled into a dynamic library. Qt has a very cool mechanism to implement this in a C++ program. Furthermore, the icing on the cake  is that, with plugins, you can extend the main program without even recompiling it.

Unfortunately, there are situations where you do not want to use plugins. That was my case on gaming consoles. Dynamic libraries are not so trivial to use on these platforms, and high level mechanisms such as the one proposed by Qt are not available, so you need to recode them. Besides, these platforms have requirements about disk access, which leads you to situations where you must absolutely avoid reading from the disk. Finally, when you develop for these platforms, you know by advance what your program will need, and in most case, you do not care about the possibility to later on dynamically extend your program (it is even often not possible, like on the Nintendo Wii). So I needed another mechanism. After some thoughts, I reformulated the problem like this:

<strong>How do you get code to be executed by just compiling and linking some source code into an executable?</strong>

More precisely, when you program runs, it usually executes a specific function, known as the <em>entry point</em>. For C++ it is generally int main(int,char**), but it can vary, like on Windows with the WinMain stuff. If you want code to be executed, there must be an instruction in that entry point that leads to the code somehow. So the exact problem  is that of executing code before the entry point.

The solution I came up with is to use what I coined <em>static registerers</em>. You create a class whose constructor executes the code you want to execute, and you create a static instance of it in your file. Simple idea. We will see in a minute when it comes gory. But for the moment, let's see a concrete example.

I needed an encapsulation of a Lua interpreter that could be enriched with binded functions. I wanted my co-workers to be able to write these functions in standalone .cpp files. Here is my solution. First, let's see only the interface of the <tt>LuaState</tt> class:
[sourcecode language="cpp"]
#ifndef __LuaState_h
#define __LuaState_h

#include &quot;./Singleton.h&quot;
#include &lt;lua.hpp&gt;
#include &lt;map&gt;
#include &lt;string&gt;

class LuaState : public Singleton
{
public:
  LuaState();
  virtual ~LuaState();
  operator lua_State*() const;

  struct Registerer
  {
    Registerer(const std::string &amp;amp;amp;amp;n,lua_CFunction f);
  };
private:
  static std::map&lt;std::string,lua_CFunction&gt;  registeredFunctions();
  lua_State *_L;
};

#define REGISTER_FUNCTION(f) LuaState::Registerer f##_registerer(#f,f);

#endif
[/sourcecode]
Apart from the registerer stuff, which you can ignore for the moment, it is quite simple. The only notable fact is that we used a singleton approach (the class can be instanciated only once, and if instanciated, a pointer to the unique instance is accessible via a class method). We also implemented a conversion operator so we can use that instance as a standard lua state. Here is a very small Lua interpreter that executes the lua code passed as command line arguments.
[sourcecode language="cpp"]
#include &quot;./LuaState.h&quot;

int main(int argc,char **argv)
{
  (void) new LuaState();
  for (int i=1;i &lt; argc;++i)
  {
    luaL_dostring(LuaState::singleton(),argv[i]);
  }
  delete LuaState::singletonPtr();
  return 0;
}
[/sourcecode]
Quite simple too. Notice that we use the singleton API to avoid storing the created LuaState in a variable. Now for the interesting part. I wanted my coders to be able to add new C-coded functions to the Lua interpreter. For example, a <tt>power</tt> function, which makes much more sense to code in C. With registerers, all they have to do is write the following file and add it to the build solution.
[sourcecode language="cpp"]
#include &quot;./LuaState.h&quot;

int power(lua_State *L)
{
  float x = luaL_checknumber(L,-2);
  int   n = luaL_checkint(L,-1);
  lua_pop(L,2);
  float r = 1;
  for (int i=0;i &lt; n;++i) r *= x;
  lua_pushnumber(L,r);
  return 1;
}

REGISTER_FUNCTION(power);
[/sourcecode]
So how does it work, and what are the nuts and bolts? Here is the implementation of <tt>LuaState</tt>:
[sourcecode language="cpp"]
#include &quot;./LuaState.h&quot;

using namespace std;

template &lt;&gt; LuaState *Singleton::_singleton = NULL;

LuaState::Registerer::Registerer(const string &amp;amp;amp;amp;n,lua_CFunction f)
{
  LuaState::registeredFunctions()[n] = f;
}
LuaState::LuaState()
{
  _L = luaL_newstate();
  luaL_openlibs(_L);
  for (map::const_iterator
      i=registeredFunctions().begin(),
      ii=registeredFunctions().end();
    i != ii;++i)
  {
    lua_pushcfunction(_L,i-&gt;second);
    lua_setglobal(_L,i-&gt;first.c_str());
  }
}
LuaState::~LuaState()
{
  lua_close(_L);
}
LuaState::operator lua_State*() const
{
  return _L;
}

map&lt;string,lua_CFunction&gt; LuaState::registeredFunctions()
{
  static map&lt;string,lua_CFunction&gt; l;
  return l;
}
[/sourcecode]
When the <tt>LuaState</tt> instance is created, it retrieves a static map of registered pointer to functions (the <tt>lua_CFunction</tt> is a typedef to functions with signature <tt>void (*)(lua_State*)</tt> and push them into the Lua interpreter with the given name. The <tt>REGISTER_FUNCTION()</tt> macro uses the preprocessor stringification mechanism (the pound sign) to use the same name than the C function.

The only subtle part is the <tt>registeredFunctions()</tt>. One may think that we could have used directly a static map in the <tt>LuaState</tt> class. But this would be dangerous. Indeed, the C++ standard does not guarantee any order for initialization of the static variables. So we could be in a situation where the static registerers are initialized before the static map, leading to crashes (this happened to me and was hard to diagnose). The good news is that the C++ standard states that static inside functions are initialized upon first call of the function. Which solves nicely our problem.

For completness, here is a simple singleton code, inspired from Ogre's implementation. For more information on singletons, you can check Alexandrescu's book "Modern Design in C++".
[sourcecode language="cpp"]#ifndef _Singleton_h__
#define _Singleton_h__

#include &lt;cassert&gt;

template &lt;class T&gt;  class Singleton
{
public:
  Singleton()
  {
      assert(!_singleton);
    _singleton = static_cast&lt;T*&gt;(this);
  }
  ~Singleton()
  {
    assert(_singleton);
    _singleton = 0;
  }
  static T&amp;amp; singleton()
  {
    assert(_singleton);
    return *_singleton;
  }
  static T* singletonPtr()
  {
    return _singleton;
  }
protected:
  static T* _singleton;
};

#endif
[/sourcecode]