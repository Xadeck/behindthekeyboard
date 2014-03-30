Here is a nice little C macro that will let you inline Lua code into a C/C++ program.
[sourcecode language="cpp"]
#ifndef __lua_inline_H
#define __lua_inline_H
#include 

extern void lua_inline_error(lua_State *,const char *,int);

#define LUA_INLINE(L,...) \
do { \
  if (luaL_dostring(L,#__VA_ARGS__)!=0)\
    lua_inline_error(L,__FILE__,__LINE__);\
} while (0);

#endif
[/sourcecode]

<!--more-->
The <tt>do while (0)</tt> construct is a classical hack so you can use the macro in any circumstances. There are some good explanations on the post <a href="http://stackoverflow.com/questions/257418/do-while-0-what-is-it-good-for">do { … } while (0) what is it good for?</a> and also  <a href='http://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Multi-statement_Macro'>in the wikibook for C++ idioms</a>.

The whole trick is done by the variational args of the preprocessor (which handles comas in the inlined lua code), and the stringification operator (pound sign) of the preprocessor. Here is a little program that shows usage.
[sourcecode language="cpp"]
#include &quot;./lua_inline.h&quot;
#include &lt;lua.hpp&gt;
#include &lt;iostream&gt;    

using namespace std;

void lua_inline_error(lua_State *L,const char * file,int line)
{
  cerr &lt;&lt; file &lt;&lt; &quot;, &quot; &lt;&lt; line &lt;&lt; &quot;: inlined lua error\n&quot;;
  cerr &lt;&lt; &quot;\t&quot; &lt;&lt; lua_tostring(L,-1) &lt;&lt; &quot;\n&quot;;
  lua_pop(L,1);	
}                             

int main(int argc,char **argv)
{
	lua_State *L = luaL_newstate();
	luaL_openlibs(L);
	LUA_INLINE(L,
		local a = 3
		local t = {1,2,3,4,5}
		for _,x in pairs(t) do
			print(x*a)
		end
	)	
	lua_close(L);	
	return 0;
}
[/sourcecode]
The interest of this macro is that some code is much easier to write in lua (in particular when iterating over tables) than with the C api.