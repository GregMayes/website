---
title: Importing Global Functions in Namespaced PHP Code
tags: PHP
---

You may have seen namespaced PHP code where global functions have been imported via `use function` or have been fully-qualified using a `\` character. Or perhaps you use a static analysis tool that routinely bugs you to import global functions and you don't really understand why. In this short article, I'm going to explain the "why" behind this particular micro-optimisation.

I've used the `str_repeat` function in the example below but it could have been another internal PHP function or a globally scoped function declared in something like a `helpers.php` file.

{% highlight php %}

<?php

namespace Foo\Bar\Baz;

use function str_repeat;

echo str_repeat('teststring', 3);

{% endhighlight %}

I'm going to run the code snippet through an Opcode dumper called [Vulcan Logic Dumper (VLD)](https://pecl.php.net/package/vld). If you'd like to play around with VLD without installing it on your local environment, you can easily run it using [3v4l.org](https://3v4l.org). After you `eval();` your code, you should see a tab called 'VLD' at the top of the results box. You can also look at the performance metrics of the executed code by using the 'Performance' tab.

In the Opcode dump below, we can see that on line 0 that there's an `INIT_FCALL` Opcode for the invocation of `str_repeat`. On lines 1 and 2, there are `SEND_VAL` Opcodes on for passing the arguments to the function.

{% highlight plaintext %}

number of ops:  6
compiled vars:  none
line      #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
    7     0  E >   INIT_FCALL                                               'str_repeat'
          1        SEND_VAL                                                 'teststring'
          2        SEND_VAL                                                 3
          3        DO_ICALL                                         $0      
          4        ECHO                                                     $0
          5      > RETURN                                                   1

{% endhighlight %}

You might think that this is how it always looks when you call a function such as `str_repeat` within namespaced code. However, that `INIT_FCALL` Opcode is only being used here because of the explicit importing of  `str_repeat` function from the global namespace. This would also be the case if you removed the `use function` statement and called the function as `\str_repeat('teststring', 3)` instead.

If I run that snippet again without the import, I get slightly different results this time. The Opcode on line 0 in the below dump is now `INIT_NS_FCALL_BY_NAME` rather than `INIT_FCALL`. This means that PHP is looking for the function `str_repeat` inside the namespace `Foo\Bar\Baz`, which we know doesn't exist. It will then traverse the namespace tree (`Foo\Bar`, `Foo`) until it hits the global namespace and finds the internal PHP function `str_repeat`. The performance implications of this are hard to quantify and will vary greatly depending on your application, but I think we can all agree that fewer operations to produce the same result is preferable.


{% highlight plaintext %}

number of ops:  6
compiled vars:  none
line      #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
    5     0  E >   INIT_NS_FCALL_BY_NAME                                    'Foo%5CBar%5CBaz%5Cstr_repeat'
          1        SEND_VAL_EX                                              'teststring'
          2        SEND_VAL_EX                                              3
          3        DO_FCALL                                      0  $0      
          4        ECHO                                                     $0
          5      > RETURN                                                   1

{% endhighlight %}


Avoiding namespace traversal is just one reason to import global functions within namespaced code. I'll explore the other reason by calling the `strlen` function with an explicit import; I've made a crude example function called `isGreaterThanFiveCharacters` that returns whether an ASCII string is greater than 5 characters or not.

{% highlight php %}

<?php

namespace Foo\Bar\Baz;

use function strlen;

function isGreaterThanFiveCharacters(string $input): bool
{
    return strlen($input) > 5;
}

isGreaterThanFiveCharacters('test string');

{% endhighlight %}

On line 1 of the below Opcode dump, the Opcode is `STRLEN` rather than `INIT_FCALL`. This is because `strlen` is one of a limited number of internal PHP functions that have their own Opcodes. The `STRLEN` Opcode means that rather than the Zend Engine having to call a function to calculate the length of the string, it can instead access the `len` property from the [zend_string](https://www.phpinternalsbook.com/php7/internal_types/strings/zend_strings.html) struct. To my knowledge there isn't an official list of the internal functions that have an accompanying Opcode, but [this method from PHP-CS-Fixer](https://github.com/PHP-CS-Fixer/PHP-CS-Fixer/blob/master/src/Fixer/FunctionNotation/NativeFunctionInvocationFixer.php#L327) looks to cover them all.

{% highlight plaintext %}

function name:  Foo\Bar\Baz\isGreaterThanFiveCharacters
number of ops:  7
compiled vars:  !0 = $input
line      #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
    7     0  E >   RECV                                             !0      
    9     1        STRLEN                                           ~1      !0
          2        IS_SMALLER                                       ~2      5, ~1
          3        VERIFY_RETURN_TYPE                                       ~2
          4      > RETURN                                                   ~2
   10     5*       VERIFY_RETURN_TYPE                                       
          6*     > RETURN                                                   null


{% endhighlight %}

What if I was to run the `isGreaterThanFiveCharacters` function again, but this time without importing the `strlen` function?

{% highlight plaintext %}

function name:  Foo\Bar\Baz\isGreaterThanFiveCharacters
number of ops:  9
compiled vars:  !0 = $input
line      #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
    5     0  E >   RECV                                             !0      
    7     1        INIT_NS_FCALL_BY_NAME                                    'Foo%5CBar%5CBaz%5Cstrlen'
          2        SEND_VAR_EX                                              !0
          3        DO_FCALL                                      0  $1      
          4        IS_SMALLER                                       ~2      5, $1
          5        VERIFY_RETURN_TYPE                                       ~2
          6      > RETURN                                                   ~2
    8     7*       VERIFY_RETURN_TYPE                                       
          8*     > RETURN                                                   null

{% endhighlight %}

I now get the `INIT_NS_FCALL_BY_NAME` Opcode instead of `STRLEN`, followed by `SEND_VAR_EX` and `DO_FCALL`. Huh, what's going on here? Well, it turns out that internal PHP functions with accompanying Opcodes are only able to utilise them inside a namespace if the function has been imported or has been fully-qualified. This mechanic doesn't apply to non-namespaced code, where the aforementioned functions will always use their accompanying Opcodes.

If you were to call that `isGreaterThanFiveCharacters` function hundreds of thousands of times in a performance sensitive loop inside namespaced code without having imported `strlen`, you're going to suffer two performance penalties. The first being that your application is going to spend time needlessly traversing namespaces to find `strlen`. The second is that PHP will have to invoke the `strlen` function each time, when it could get the `len` property from the `zend_string` struct instead, and skip the function call entirely.