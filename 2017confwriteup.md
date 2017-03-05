# PHP Conference 2017: Jumps in PHP
## Based on Derick Rethams' Talk: It's All About The Goto
### Roderick Timmerman

### Abstract
This year I attended the PHP Conference with the objectives of attending talks discussing the PHP language's internal structure, and dev ops.  The PHP's workings of PHP's internal structure are a generally deemed mysterious and dangerous ground owing to the lack of documentation concerning them; so talk's such as this one, by Derick Rethams, are bound to be interesting.

This conference talk demonstrated that it is possible to view some of the inner workings of PHP using the available VLD extension, that all control structures and loops are expressed by various types of jump statements (neé GOTO statements).  Further to this it thus possible to see ways of optimising the use of various constructs in PHP allowing for faster and cleaner code.

### How PHP Gets to Byte Code
The key to understanding the inner workings is to know the way the PHP executable works when executing code. PHP can execute code in different depending on the context within which it is being used, this achieved through the use of the SAPI (Server Application Programming Interface) some examples of which are: `cgi (common gateway interface)`, `fpm-fcgi (php-fpm)`, `cli (command line interface)`, and so on. For the purpose of this writeup let's focus on the `cli` SAPI. The `cli` executes PHP code in the following order:

1. Load Core PHP executable
2. MINIT (Initialise compiled modules)
3. Choose the SAPI (SAPI ready)
4. RINIT (request start up)
5. Run php code
6. RSHUTDOWN (request shutdown)
7. MSHUTDOWN (module shudwon)
9. exit

Step 5, where the php code is run can be shown in terms of the token's php will actually use to process the code:

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

All of the tokens php has available to it are found within `zend_language_scanner`. Once tokenised, the script is parsed by the `zend_language_parser` into a state machine which then exectued: an instance of Zend AST (Abstract Syntax Tree). The abstract tree is then converted into byte code.

### Examining the Byte Code
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

With this tool available to us, lets examine the output of an if structure:

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

Now the byte code looks more akin to what might be in seen in assembly code. The value for $a is provided between lines 1 and 4 where the built function readline assigns the data to variable $1.
Line 5: `ASSIGN` allocates $1 to parameter !0.  Line 6 is the start of if statement with the comparator on line 7 `IS_EQUAL`.  The `IS_EQUAL` call will return a non-zero if the condition is met thus the JMPZ (the first "goto" shown) will cause the runtime to go to line 11 `RETURN` if the condition fails otherwise printing the statement "Correct!".

The same thing can be done to shown how statements like `if...else`, `elseif`, `case`, and even `try...catch` statements work in a similar way by using other variants of `JMP`. Loops are similar in nature, apart from the where the `JMP` statments redirect their code here is a while loop:

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

Thus the efficiency of built in iterator functions vs user-defined ones can thus been shown. Consider these this userland iterator that multiplies each array element value by 2.

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

Here note the expected `JMP` type statements in the `foreach` code, essentially the code block will executed in this way for the number elements in the array.  The `array_map` variant achieves the same result but uses lambda function definitions and no bytecode jumps, the jumps instead are carried out within the zend-engine's implementation (so as native processor jumps and so on), but with a larger collection of code including two function definitions.  Programmers are at liberty to choose the solution which best suites them but perform as few jumps in the bytecode could allow for faster code.

### Conclusion

By examining the bytecode using tools like the VLD extension generated by various PHP functions ways of making code faster can be seen. Carrying out such analysis could also aid in which new features would benefit legacy code the most.  Other types analysis such as dead code analysis which just as important, since PHP doesn't optimise code because of it's nature as an intepreter, should be used in conjuction with this technique.  The VLD extension, together with the the Zend AST allow programmers to start to unravel the inner workings of PHP.
