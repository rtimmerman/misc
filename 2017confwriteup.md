# PHP Conference 2017: Jumps in PHP
## Based on Derick Rethams' Talk: It's All About The Goto
### Roderick Timmerman

### Abstract
Amongst the various talks held at the PHP Conference, a significant number of talks focused on the core internals of PHP. The workings of PHP's internal structure are a generally deemed mysterious and dangerous ground owing to the lack of documentation concerning them; so talks like "It's all About the Goto" by Derick Rethams (maintainer of extensions such as VLD, Xdebug, and more) are bound to be insightful.

Retham's conference talk, essentially a primer for the VLD extension, demonstrated how PHP opcodes could be inspected with it, and that all control structures and loops are expressed by various types of jump statements (née GOTO statements). The talk also went on to point out uses of VLD for debugging alongside other tools like XDebug.  Further to this, it becomes possible to see ways of optimising the use of PHP's various constructs and built-ins allowing for faster and cleaner code.

Disclaimer: Neither the talk or this document recommend the use of the `goto` statement itself in PHP!

### How PHP Runs Code
One way to understand PHP's inner workings is to know how the PHP executable works when executing code. PHP can execute code in different ways depending on the context within which it is being used, this achieved through the use of the SAPI (Server Application Programming Interface) some examples of which are: `cgi (common gateway interface)`: the traditional web server api, `fpm-fcgi (php-fpm)`: the api for using a resident instance of php running within a server context, `cli (command line interface)`: used when executing php on command line, and so on. For the purpose of this write-up let's focus on the `cli` SAPI. The `cli` executes PHP code in the following order:

1. Load Core PHP executable
2. MINIT (Initialise compiled modules)
3. Choose the SAPI (SAPI ready)
4. RINIT (request start up)
5. Run php code
6. RSHUTDOWN (request shutdown)
7. MSHUTDOWN (module shudwon)
9. exit

#### Tokenisation
Step 5, where the php code is run can be shown in terms of the Tokens:

This php code
```php
<?php
echo "hello, world";
```
could be represented like this:
```
T_OPEN_TAG
T_ECHO "hello, world"
```

All of the tokens php has available to it are found within `zend_language_scanner`. Once tokenised, the script is parsed by the `zend_language_parser` into a state machine which is then exectued: an instance of Zend AST (Abstract Syntax Tree). This abstract tree is then converted into byte code before being executed by the PHP core.

### PHP Bytecode Primer
Tools such as the VLD (Vulcan Logic Disassembler) allow programmers to examine the actual byte code of php application.  The tool itself can be installed directly from PEAR or compiled from github.  Install from PECL:

```bash
pecl install vld-beta
printf "extension=vld.so\n" > /etc/php.d/vld.so
```

In it's simplest form VLD will output the bytecode for a given php program to the screen:
```
php -d vld.active=1 hello.php

Finding entry points
Branch analysis from position: 0
Jump found. (Code = 62) Position 1 = -2
filename:       /home/roderick/scrapbook/php/hello.php
function name:  (null)
number of ops:  3
compiled vars:  none
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   2     0  E >   EXT_STMT                                                 
         1        ECHO                                                     'Hello%2C+world%0A'
   3     2      > RETURN                                                   1

branch: #  0; line:     2-    3; sop:     0; eop:     2; out1:  -2
path #1: 0, 
Hello, world
```
The table displayed in the output contains the following columns:

| Column   | description                                                                                    |
|----------|:-----------------------------------------------------------------------------------------------|
|`line`    | The actual PHP code line number being executed (this will not compensate for comments in code) |
| `#*`     | Opcode number                                                                                  |
| `E`      | Marks entry (e.g. into a function)                                                             |
| `I`      | [Function] Input?                                                                              |
| `O`      | [Function] Output?                                                                             |
| `op`     | Opcode name                                                                                    |
| `fetch`  | Unknown, left blank                                                                            |
| `ext`    | Unknown, left blank                                                                            |
| `return` | States where the opcode returns its output                                                     |

### PHP Bytecode Structures
With this tool lets examine the output of an if structure:

```php
<?php
$a = readline();
if ($a == 42) {
	echo "Correct!";
}
```

```
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   2     0  E >   EXT_STMT                                                 
         1        INIT_FCALL                                               'readline'
         2        EXT_FCALL_BEGIN                                          
         3        DO_FCALL                                      0  $1      
         4        EXT_FCALL_END                                            
         5        ASSIGN                                                   !0, $1
   3     6        EXT_STMT                                                 
         7        IS_EQUAL                                         ~3      !0, 42
         8      > JMPZ                                                     ~3, ->11
   4     9    >   EXT_STMT                                                 
        10        ECHO                                                     'Correct%21'
   7    11    > > RETURN                                                   1

```

Now the byte codelooks even more akin to what might be in seen in a processor's assembly code. The value for `$a` is provided between lines 1 and 4 where the built function readline assigns the data to variable $1.
Line 5: `ASSIGN` allocates `$1` to parameter `!0`.  Line 6 marks the beginning of the if statement block with the comparator on line 7 `IS_EQUAL`.  The `IS_EQUAL` call will return a non-zero if the condition is met (i.e. evaluates to `true`) thus the JMPZ (the first "goto" shown) will cause the runtime to go to line 11 `RETURN` if the condition fails otherwise printing the statement "Correct!".

The same thing can be done to shown how statements like `if...else`, `elseif`, `case`, and even `try...catch` statements work in a similar way by using other variants of `JMP`. Loops are similar in nature, apart from the where the `JMP` statments redirect their code here is a while loop (selection statements like if prohibit JMPs that loop, but loops permit this):

```php
<?php
$a = 0;
while ($a++ < 10) {
	echo $a;
}
```

```
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   1     0  E >   ASSIGN                                                   !0, 0
         1      > JMP                                                      ->3
         2    >   ECHO                                                     !0
         3    >   POST_INC                                         ~2      !0
         4        IS_SMALLER                                       ~3      ~2, 10
         5      > JMPNZ                                                    ~3, ->2
         6    > > RETURN                                                   null
```

Here we see a jump straight into the conditional section of the loop, where `POST_INC` occurs, with the loop only continuing to the echo when the condition is met (`JMPNZ` back to line 2, the `ECHO`).

#### Code Optimisation
The efficiency of built in iterator functions vs user-defined ones can thus been shown. Consider this userland iterator that multiplies each array element value by 2.


##### Foreach loop code
```php
<?php
$numbers = [1, 3, 5, 8];
foreach ($numbers as &$number) {
	$number *= 2;
}
print_r($numbers);
```

```
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   1     0  E >   ASSIGN                                                   !0, <array>
         1      > FE_RESET_RW                                      $3      !0, ->5
         2    > > FE_FETCH_RW                                              $3, !1, ->5
         3    >   ASSIGN_MUL                                    0          !1, 2
         4      > JMP                                                      ->2
         5    >   FE_FREE                                                  $3
         6        INIT_FCALL                                               'print_r'
         7        SEND_VAR                                                 !0
         8        DO_FCALL                                      0          
         9      > RETURN                                                   null
```

##### Array map code
```php
<?php
$numbers = [1, 3, 5, 8];
$numbers = array_map(
	function ($number) {
		return $number * 2;
	},
	$numbers
);
print_r($numbers);
```

```
function name:  (null)
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   1     0  E >   ASSIGN                                                   !0, <array>
         1        INIT_FCALL                                               'array_map'
         2        DECLARE_LAMBDA_FUNCTION                                  '%00%7Bclosure%7DCommand+line+code0xb526b22d'
         3        SEND_VAL                                                 ~2
         4        SEND_VAR                                                 !0
         5        DO_FCALL                                      0  $3      
         6        ASSIGN                                                   !0, $3
         7        INIT_FCALL                                               'print_r'
         8        SEND_VAR                                                 !0
         9        DO_FCALL                                      0          
        10      > RETURN                                                   null
...
...
Function %00%7Bclosure%7DCommand+line+code0xb526b22d:
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
         0  E >   RECV                                             !0      
         1        MUL                                              ~1      !0, 2
         2      > RETURN                                                   ~1
         3*     > RETURN                                                   null
```

Here, note the expected `JMP` type statements in the `foreach` code, essentially the code block will executed in this way for the number elements in the array.  The `array_map` variant achieves the same result but uses lambda function definitions and no bytecode jumps, the jumps instead are carried out within the zend-engine's implementation (so as native processor jumps and so on), but with a larger collection of code including two function definitions.  Programmers are at liberty to choose the solution which best suits them but it is worth noting that performing as few jumps in the bytecode could allow for faster code.

### Conclusion

By examining the bytecode using tools like the VLD extension generated by various PHP functions ways of making code faster can be seen. Carrying out such analysis could also match other uses such as testing which new features would benefit legacy code the most when migrating and debugging troublesome code.

It is also worth considering other types code analysis when using this technique.  Dead code analysis (as seen Xdebug) can be useful since PHP doesn't optimise code because of it's nature as an intepreted language.

The VLD extension, together with the the Zend AST allow programmers to begin start to unravel the inner workings of the way PHP runs code.
