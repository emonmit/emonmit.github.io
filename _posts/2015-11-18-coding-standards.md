---
layout: post
title: php7 coding standards
date: 2015-11-19
categories: php
tags: [php, coding standards]
description: php7源码中的编程规范文档
---
========================<br>
  PHP Coding Standards<br>
========================<br>
<br>
This file lists several standards that any programmer adding or changing code in PHP should follow.  Since this file was added at a very late stage of the development of PHP v3.0, the code base does not (yet) fully
follow it, but it's going in that general direction.  Since we are now well into version 5 releases, many sections have been recoded to use these rules.<br>
<br>
Code Implementation<br>
-------------------<br>
<br>
0.  Document your code in source files and the manual. [tm]<br>
<br>
1.  Functions that are given pointers to resources should not free them<br>
<br>
For instance, ``function int mail(char *to, char *from)`` should NOT free to and/or from.<br>
<br>
Exceptions:<br>
<br>
- The function's designated behavior is freeing that resource.  E.g. efree()<br>

- The function is given a boolean argument, that controls whether or not the function may free its arguments (if true - the function must free its arguments, if false - it must not)<br>

- Low-level parser routines, that are tightly integrated with the token cache and the bison code for minimum memory copying overhead.<br>
<br>
2.  Functions that are tightly integrated with other functions within the same module, and rely on each other non-trivial behavior, should be documented as such and declared 'static'.  They should be avoided if possible.<br>
<br>
3.  Use definitions and macros whenever possible, so that constants have meaningful names and can be easily manipulated.  The only exceptions to this rule are 0 and 1, when used as false and true (respectively). Any other use of a numeric constant to specify different behavior or actions should be done through a #define.<br>
<br>
4.  When writing functions that deal with strings, be sure to remember that PHP holds the length property of each string, and that it shouldn't be calculated with strlen().  Write your functions in such a way so that they'll take advantage of the length property, both for efficiency and in order for them to be binary-safe. Functions that change strings and obtain their new lengths while doing so, should return that new length, so it doesn't have to be recalculated with strlen() (e.g. php_addslashes())<br>
<br>
5.  NEVER USE strncat().  If you're absolutely sure you know what you're doing, check its man page again, and only then, consider using it, and even then, try avoiding it.<br>
<br>
6.  Use ``PHP_*`` macros in the PHP source, and ``ZEND_*`` macros in the Zend part of the source. Although the ``PHP_*`` macro's are mostly aliased to the ``ZEND_*`` macros it gives a better understanding on what kind of macro you're calling.<br>
<br>
7.  When commenting out code using a #if statement, do NOT use 0 only. Instead use "<git username here>_0". For example, #if FOO_0, where FOO is your git user foo.  This allows easier tracking of why code was commented out, especially in bundled libraries.<br>
<br>
8.  Do not define functions that are not available.  For instance, if a library is missing a function, do not define the PHP version of the function, and do not raise a run-time error about the function not existing.  End users should use function_exists() to test for the existence of a function<br>
<br>
9.  Prefer emalloc(), efree(), estrdup(), etc. to their standard C library counterparts.  These functions implement an internal "safety-net" mechanism that ensures the deallocation of any unfreed memory at the end of a request.  They also provide useful allocation and overflow information while running in debug mode.<br>

    In almost all cases, memory returned to the engine must be allocated
    using emalloc().<br>

    The use of malloc() should be limited to cases where a third-party
    library may need to control or free the memory, or when the memory in
    question needs to survive between multiple requests.<br>
<br>
User Functions/Methods Naming Conventions<br>
------------------
<br>
1.  Function names for user-level functions should be enclosed with in the PHP_FUNCTION() macro. They should be in lowercase, with words underscore delimited, with care taken to minimize the letter count. Abbreviations should not be used when they greatly decrease the readability of the function name itself::<br>

    Good:<br>
    'mcrypt_enc_self_test'<br>
    'mysql_list_fields'<br>
<br>
    Ok:<br>
    'mcrypt_module_get_algo_supported_key_sizes'<br>
    (could be 'mcrypt_mod_get_algo_sup_key_sizes'?)<br>
    'get_html_translation_table'<br>
    (could be 'html_get_trans_table'?)<br>
<br>
    Bad:<br>
    'hw_GetObjectByQueryCollObj'<br>
    'pg_setclientencoding'<br>
    'jf_n_s_i'<br>
<br>
2.  If they are part of a "parent set" of functions, that parent should be included in the user function name, and should be clearly related to the parent program or function family. This should be in the form of ``parent_*``::<br>

    A family of 'foo' functions, for example:<br>
    <br>
    Good:<br>
    'foo_select_bar'<br>
    'foo_insert_baz'<br>
    'foo_delete_baz'<br>
<br>
    Bad:<br>
    'fooselect_bar'<br>
    'fooinsertbaz'<br>
    'delete_foo_baz'<br>
<br>
3.  Function names used by user functions should be prefixed with ``_php_``, and followed by a word or an underscore-delimited list of words, in lowercase letters, that describes the function.  If applicable, they should be declared 'static'.<br>
<br>
4.  Variable names must be meaningful.  One letter variable names must be avoided, except for places where the variable has no real meaning or a trivial meaning (e.g. for (i=0; i<100; i++) ...).<br>
<br>
5.  Variable names should be in lowercase.  Use underscores to separate between words.<br>
<br>
6.  Method names follow the 'studlyCaps' (also referred to as 'bumpy case' or 'camel caps') naming convention, with care taken to minimize the letter count. The initial letter of the name is lowercase, and each letter that starts a new 'word' is capitalized::<br>
<br>
    Good:<br>
    'connect()'<br>
    'getData()'<br>
    'buildSomeWidget()'<br>
<br>
    Bad:<br>
    'get_Data()'<br>
    'buildsomewidget'<br>
    'getI()'<br>
<br>
7.  Classes should be given descriptive names. Avoid using abbreviations where possible. Each word in the class name should start with a capital letter, without underscore delimiters (CamelCaps starting with a capital letter). The class name should be prefixed with the name of the 'parent set' (e.g. the name of the extension)::<br>
<br>
    Good:<br>
    'Curl'<br>
    'FooBar'<br>
<br>
    Bad:<br>
    'foobar'<br>
    'foo_bar'<br>
<br>
Internal Function Naming Convensions<br>
----------------------<br>
<br>
1.  Functions that are part of the external API should be named 'php_modulename_function()' to avoid symbol collision. They should be in lowercase, with words underscore delimited. Exposed API must be defined in 'php_modulename.h'.<br>

    PHPAPI char *php_session_create_id(PS_CREATE_SID_ARGS);<br>

    Unexposed module function should be static and should not be defined in 'php_modulename.h'.<br>

    static int php_session_destroy(TSRMLS_D)<br>
<br>
2.  Main module source file must be named 'modulename.c'.<br>
<br>
3.  Header file that is used by other sources must be named 'php_modulename.h'.<br>

<br>
Syntax and indentation<br>
----------------------<br>
<br>
1.  Never use C++ style comments (i.e. // comment).  Always use C-style comments instead.  PHP is written in C, and is aimed at compiling under any ANSI-C compliant compiler.  Even though many compilers accept C++-style comments in C code, you have to ensure that your code would compile with other compilers as well. The only exception to this rule is code that is Win32-specific, because the Win32 port is MS-Visual C++ specific, and this compiler is known to accept C++-style comments in C code.<br>
<br>
2.  Use K&R-style.  Of course, we can't and don't want to force anybody to use a style he or she is not used to, but, at the very least, when you write code that goes into the core of PHP or one of its standard modules, please maintain the K&R style.  This applies to just about everything, starting with indentation and comment styles and up to function declaration syntax. Also see Indentstyle.<br>

    Indentstyle: http://www.catb.org/~esr/jargon/html/I/indent-style.html<br>
<br>
3.  Be generous with whitespace and braces.  Keep one empty line between the variable declaration section and the statements in a block, as well as between logical statement groups in a block.  Maintain at least one empty line between two functions, preferably two.  Always prefer::<br>
<br>
    if (foo) {
        bar;
    }

    to:

    if(foo)bar;
<br>
4.  When indenting, use the tab character.  A tab is expected to represent four spaces.  It is important to maintain consistency in indenture so that definitions, comments, and control structures line up correctly.<br>
<br>
5.  Preprocessor statements (#if and such) MUST start at column one. To indent preprocessor directives you should put the # at the beginning of a line, followed by any number of whitespace.<br>
<br>
Testing<br>
-------<br>
<br>
1.  Extensions should be well tested using *.phpt tests. Read about that in README.TESTING.<br>
<br>
Documentation and Folding Hooks<br>
-------------------------------<br>
<br>
In order to make sure that the online documentation stays in line with the code, each user-level function should have its user-level function prototype before it along with a brief one-line description of what the function does.  It would look like this::<br>

    /* \{\{\{ proto int abs(int number) 
    Returns the absolute value of the number */
    PHP_FUNCTION(abs)
    {
       ...
    }
    /* }}} */
<br>

    The \{\{\{ symbols are the default folding symbols for the folding mode in Emacs and vim (set fdm=marker).  Folding is very useful when dealing with large files because you can scroll through the file quickly and just unfold the function you wish to work on.  The }}} at the end of each function marks the end of the fold, and should be on a separate line.
<br>
The "proto" keyword there is just a helper for the doc/genfuncsummary script which generates a full function summary.  Having this keyword in front of the function prototypes allows us to put folds elsewhere in the code without messing up the function summary.<br>
<br>
Optional arguments are written like this::<br>

    /* \{\{\{ proto object imap_header(int stream_id, int msg_no [, int from_length [, int subject_length [, string default_host]]])
       Returns a header object with the defined parameters */
    /* }}} */
<br>
And yes, please keep the prototype on a single line, even if that line is massive.<br>
<br>
New and Experimental Functions<br>
-----------------------------------<br><br>
To reduce the problems normally associated with the first public implementation of a new set of functions, it has been suggested that the first implementation include a file labeled 'EXPERIMENTAL' in the function directory, and that the functions follow the standard prefixing conventions during their initial implementation.<br>

The file labelled 'EXPERIMENTAL' should include the following information::<br>

    Any authoring information (known bugs, future directions of the module).
    Ongoing status notes which may not be appropriate for Git comments.
<br>
In general new features should go to PECL or experimental branches until there are specific reasons for directly adding it to the core distribution.<br>
<br>
Aliases & Legacy Documentation<br>
-----------------------------------<br><br>
You may also have some deprecated aliases with close to duplicate names, for example, somedb_select_result and somedb_selectresult. For documentation purposes, these will only be documented by the most current name, with the aliases listed in the documentation for the parent function. For ease of reference, user-functions with completely different names, that alias to the same function (such as highlight_file and show_source), will be separately documented. The proto should still be included, describing which function is aliased.<br>

Backwards compatible functions and names should be maintained as long as the code can be reasonably be kept as part of the codebase. See /phpdoc/README for more information on documentation.<br>

