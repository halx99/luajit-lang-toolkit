LuaJIT Language Toolkit
===

The LuaJIT Language Toolkit is an implementation of the Lua programming language written in Lua itself.
It works by generating a LuaJIT's bytecode including the debug informations and use the LuaJIT's virtual machine to run the generated bytecode.

On itself the language toolkit does not do anything useful since LuaJIT itself does the same things natively.
The purpose of the language toolkit is to provide a starting point to implement a programming language that target the LuaJIT virtual machine.

With the LuaJIT Language Toolkit is easy to create a new language or modify the Lua language because the parser is cleanly separated from the bytecode generator and the virtual machine.

The toolkit implement actually a complete pipeline to parse a Lua program, generate an AST tree and generate the bytecode.

Lexer
---

Its role is to recognize lexical elements from the program text.
It does take the text of the program as input and does produce a flow of "tokens".

Using the language toolkit you can run the lexer only to examinate the flow of tokens:

```
luajit run-lexer.lua tests/test-1.lua
```

The command above generate for the following code fragment:

```lua
local x = {}
for k = 1, 10 do
    x[k] = k*k + 1
end
```

to obtain a list of the tokens:

    TK_local
    TK_name	x
    =
    {
    }
    TK_for
    TK_name	k
    =
    TK_number	1
    ,
    TK_number	10
    TK_do
    TK_name	x
    [
    TK_name	k
    ]
    =
    TK_name	k
    *
    TK_name	k
    +
    TK_number	1
    TK_end

Each line represent a token where the first element is the kind of token and the second element is its value, if any.

The Lexer's code is an almost literal translation of the LuaJIT's lexer.

Parser
---

The parser takes the flow of tokens as given by the lexer and forms the statements and expressions according to the language's grammar.
The parser is based on a list of parsing rules that are invoked each time a the input match a given rule.
When the input match a rule a corresponding function in the AST module is called to build an AST node.
The generated nodes in turns are passed as arguments to the other parsing rules until the whole program is parsed and a complete AST tree is built for the program text.

The AST tree is very useful since it does abstract the structure of the program and is more easy to manipulate.

What distinguish the language toolkit from LuaJIT is that the parser phase does generate an AST tree and the bytecode generation is done in a separate phase only when the AST tree is completely generated.

LuaJIT itself operates differently.
During the parsing phase it does not generate any AST but instead the bytecode is directly generated and loaded into the memory to be executed by the VM.
This means that LuaJIT's C implementation perform the three operations:

- parse the program text
- generate the bytecode
- load the bytecode into memory

in one single pass.
This approach is remarkable on itself and very efficient but it makes difficult to modify or extend the programming language.

### Parsing Rule example ###

To illustrate how the parsing work in the language toolkit let us make an example.
The grammar rule for the "return" statement is:

```
explist ::= {exp ','} exp

return_stmt ::= return [explist]
```

In this case the toolkit parser's rule will parse the optional expression list by calling the function `expr_list`.
Then, once the expressions are parsed the AST's rule `ast:return_stmt(exps, line)` will be invoked by passing the expressions list obtained before.

```lua
local function parse_return(ast, ls, line)
    ls:next() -- Skip 'return'.
    ls.fs.has_return = true
    local exps
    if EndOfBlock[ls.token] or ls.token == ';' then -- Base return.
        exps = { }
    else -- Return with one or more values.
        exps = expr_list(ast, ls)
    end
    return ast:return_stmt(exps, line)
end
```

As you can see the AST function are invoked using the `ast` object.

In addition the parser provides additional informations about:

* the function prototype
* the syntactic scope

The first is used to keep trace of some informations about the current function parsed.

The syntactic scope rules tell to the user's rule when a new syntactic block begins or end.
Currently this is not really used by the AST builder but it can be useful for other implementations.

The Abstract Syntax Tree (AST)
---

The abstract syntax tree represent the whole Lua program with all the informations.

One possible approach to implement a new programming language is to generate an AST tree that correspond to the target programming language and to transform the tree in a Lua's AST tree in a separate phase.

Another possible approach is to act from the parser itself and directly generate the appropriate Lua AST nodes.

Currently the language toolkit does not perform any transformation and just pass the AST tree to the bytecode generator module.

Bytecode Generator
---

Once the AST tree is generated it can be feeded to the bytecode generator module that will generate the corresponding LuaJIT bytecode.

The bytecode generator is based on the original work of Richard Hundt for the Nyanga programming language.
It was largely modified by myself to produce optimized code similar to what LuaJIT generate itself.
A lot of work was also done to ensure the correctness of the bytecode and of the debug informations.

Alternative Lua Code generator
------------------------------

Instead of passing the AST tree to the bytecode generator an alternative module can be used to generate Lua code.
The module is called "luacode-generator" and can be used exactly like the bytecode generator.

The Lua code generator has the advantage of being more simple and more safe as the code is parsed directly by LuaJIT ensuring from the beginning complete compatibility of the bytecode.

Currently the Lua Code Generator backend does not preserve the line numbers of the original source code. This is meant to be fixed in the future.

Use this backend instead of the bytecode generator if you prefer to have a more safe backend to convert the Lua AST to code.
The module can be used also to pretty-printing a Lua AST tree since the code itself is probably the most human readable representation of the AST tree.

C API
---

The language toolkit does provide a very simple set of C API to implement a custom language.
The functions provided by the C API are:

```c
/* The functions above are the equivalent of the luaL_* corresponding
   functions. */
extern int language_init(lua_State *L);
extern int language_report(lua_State *L, int status);
extern int language_loadbuffer(lua_State *L, const char *buff, size_t sz, const char *name);
extern int language_loadfile(lua_State *L, const char *filename);


/* This function push on the stack a Lua table with the functions:
   loadstring, loadfile, dofile and loader.
   The first three function can replace the Lua functions while the
   last one, loader, can be used as a customized "loader" function for
   the "require" function. */
extern int luaopen_langloaders(lua_State *L);
```

The functions above can be used to create a custom LuaJIT executable that use the language toolkit implementation.

When the function `language_*` is used, an independent `lua_State` is created behind the scenes and used to compile the bytecode.
Once the bytecode is generated it is loaded into the user's `lua_State` ready to be executed.
The approach of using a separate Lua's state ensure that the process of compiling does not interfere with the user's application.

It should be noted that even when an executable is created with the C API the lang/* Lua files need to be available at run time because they are used by the language toolkit's Lua state.

Running the Application
---

The application can be run with the following command:

```
luajit run.lua [lua-options] <filename>
```

The "run.lua" script will just invoke the complete pipeline of the lexer, parser and bytecode generator and it will pass the bytecode to luajit with "loadstring".

The language toolkit provide also a customized executable named `luajit-x` that use the language toolkit's toolchain instead of the native one.
Otherwise the program `luajit-x` works exactly as luajit itself and accept the same options.

This means that you can experiment with the language by modifying the Lua implementation of the language and test the changes immediately without recompiling anything by using `luajit-x` as a REPL.

### Generated Bytecode ###

You can inspect the bytecode generated by the language toolkit by using the "-b" options.
They can be invoked either with standard luajit by using "run.lua" or directly using the customized program `luajit-x`.

For example you can inspect the bytecode using the following command:

```
luajit run.lua -bl tests/test-1.lua
```

or in alternative:

```
./src/luajit-x -bl tests/test-1.lua
```

where we suppose that you are running `luajit-x` from the language toolkit's root directory.
This is somewhat *required* since the `luajit-x` programe needs to found the lang/* Lua modules when is executed.

Either way, when you use one of the two commands above to generate the bytecode you will obtain on the screen:

```
-- BYTECODE -- "test-1.lua":0-7
00001    TNEW     0   0
0002    KSHORT   1   1
0003    KSHORT   2  10
0004    KSHORT   3   1
0005    FORI     1 => 0010
0006 => MULVV    5   4   4
0007    ADDVN    5   5   0  ; 1
0008    TSETV    5   0   4
0009    FORL     1 => 0006
0010 => KSHORT   1   1
0011    KSHORT   2  10
0012    KSHORT   3   1
0013    FORI     1 => 0018
0014 => GGET     5   0      ; "print"
0015    TGETV    6   0   4
0016    CALL     5   1   2
0017    FORL     1 => 0014
0018 => RET0     0   1
```

You can compare it with the bytecode generated natively by LuaJIT using the command:

```
luajit -bl tests/test-1.lua
```

In the example above the generated bytecode will be *identical* to those generated by LuaJIT.
This is not an hazard since the Language Toolkit's bytecode generator is designed to produce the same bytecode that LuaJIT itself would generate.
Yet in some cases the generated code will differ but this is not considered a problem as long as the generated code is still correct.

### Bytecode Annotated Dump ###

In addition to the standard LuaJIT bytecode functions the language toolkit support also a special debug mode where the bytecode in printed byte-by-byte in hex format with some annotations on the right side of the screen.
The annotations will explain the meaning of each chunk of bytes and decode them as appropriate.

For example:

```
luajit run.lua -bx tests/test-1.lua
```

will print on the screen something like:

```
1b 4c 4a 01             | Header LuaJIT 2.0 BC
00                      | Flags: None
11 40 74 65 73 74 73 2f | Chunkname: @tests/test-1.lua
74 65 73 74 2d 31 2e 6c |
75 61                   |
                        | .. prototype ..
8a 01                   | prototype length 138
02                      | prototype flags PROTO_VARARG
00                      | parameters number 0
07                      | framesize 7
00 01 01 12             | size uv: 0 kgc: 1 kn: 1 bc: 19
31                      | debug size 49
00 07                   | firstline: 0 numline: 7
                        | .. bytecode ..
32 00 00 00             | 0001    TNEW     0   0
27 01 01 00             | 0002    KSHORT   1   1
27 02 0a 00             | 0003    KSHORT   2  10
27 03 01 00             | 0004    KSHORT   3   1
49 01 04 80             | 0005    FORI     1 => 0010
20 05 04 04             | 0006 => MULVV    5   4   4
14 05 00 05             | 0007    ADDVN    5   5   0  ; 1
39 05 04 00             | 0008    TSETV    5   0   4
4b 01 fc 7f             | 0009    FORL     1 => 0006
27 01 01 00             | 0010 => KSHORT   1   1
27 02 0a 00             | 0011    KSHORT   2  10
27 03 01 00             | 0012    KSHORT   3   1
49 01 04 80             | 0013    FORI     1 => 0018
34 05 00 00             | 0014 => GGET     5   0      ; "print"
36 06 04 00             | 0015    TGETV    6   0   4
3e 05 02 01             | 0016    CALL     5   1   2
4b 01 fc 7f             | 0017    FORL     1 => 0014
47 00 01 00             | 0018 => RET0     0   1
                        | .. uv ..
                        | .. kgc ..
0a 70 72 69 6e 74       | kgc: "print"
                        | .. knum ..
02                      | knum int: 1
                        | .. debug ..
01                      | pc001: line 1
02                      | pc002: line 2
02                      | pc003: line 2
02                      | pc004: line 2
02                      | pc005: line 2
...
```

This kind of output is especially useful for debugging the language toolkit itself because it does account for every byte of the bytecode and include all the sections of the bytecode.
For examples you will be able to inspect the `kgc` or `knum` sections where the prototype's constants are stored.
The output will include also the debug section in decoded form so that it can be easily inspected.

Current Status
---

Currently LuaJIT Language Toolkit should be considered as beta software.

The implementation is now complete in term of features and well tested, even for the most complex cases and a complete test suite is used to verify the correctness of the generated bytecode.

The language toolkit is currently capable of executing itself.
This means that the language toolkit is able to correctly compile and load all of its module and execute them correctly.

Yet some bugs are probably present and you should be cautious when you use LuaJIT language toolkit.
