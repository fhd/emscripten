.. _Optimizing-Code:

===============
Optimizing Code
===============

Generally you should first compile and run your code without optimizations (the default). Once you are sure that the code runs correctly, you can use the techniques in this article to make it load and run faster.

How to optimize code
====================

Code is optimized by specifying :ref:`optimization flags <emcc-compiler-optimization-options>` when running :ref:`emcc <emccdoc>`. The levels include: :ref:`-O0 <emcc-O0>` (no optimization), :ref:`-O1 <emcc-O1>`, :ref:`-O2 <emcc-O2>`, :ref:`-Os <emcc-Os>`, :ref:`-Oz <emcc-Oz>`, and :ref:`-O3 <emcc-O3>`. 

For example, to compile with optimization level ``-O2``:

.. code-block:: bash

	emcc -O2 file.cpp

The higher optimization levels introduce progressively more aggressive optimization, resulting in improved performance and code size at the cost of increased compilation time. The levels can also highlight different issues related to undefined behavior in code.

The optimization level you should use depends mostly on the current stage of development: 

- When first porting code, run *emcc* on your code using the default settings (without optimization). Check that your code works and :ref:`debug <Debugging>` and fix any issues before continuing.
- Build with lower optimization levels during development for a shorter compile/test iteration cycle (``-O0`` or ``-O1``).
- Build with ``-O2`` or ``-O3`` when releasing your code.  ``-O3`` builds are even more optimized than ``-O2``, but at the cost of significantly longer compilation time.
- Other optimizations are discussed in the following sections.

In addition to the ``-Ox`` options, there are separate compiler options that can be used to control the JavaScript optimizer (:ref:`js-opts <emcc-js-opts>`), LLVM optimizations (:ref:`llvm-opts <emcc-llvm-opts>`) and LLVM link-time optimizations (:ref:`llvm-lto <emcc-llvm-lto>`).

.. note::

	-  The meanings of the *emcc* optimization flags (``-O1, -O2`` etc.) are different to the similarly-named options in *gcc*, *clang*, and other compilers, because optimizing JavaScript is very different to optimizing native code. The mapping of the *emcc* levels to the LLVM bitcode optimization levels is documented in the reference.
	-  If you compile several files into a single JavaScript output, be sure to specify the **same** optimization flags when compiling sources into objects, and objects into JavaScript or HTML. See :ref:`Building-Projects` for more details.


Advanced compiler settings
==========================

There are several flags you can :ref:`pass to the compiler <emcc-s-option-value>` to affect code generation, which will also affect performance — for example :ref:`DISABLE_EXCEPTION_CATCHING <optimizing-code-exception-catching>`. These are documented in `src/settings.js <https://github.com/kripken/emscripten/blob/master/src/settings.js>`_. Some of these will be directly affected by the optimization settings (you can find out which ones by searching for ``apply_opt_level`` in `tools/shared.py <https://github.com/kripken/emscripten/blob/master/tools/shared.py#L906>`_).

A few useful flags are:

- 
	.. _optimizing-code-no-exit-runtime:
	
	``NO_EXIT_RUNTIME``: Building with ``-s NO_EXIT_RUNTIME=1`` lets the compiler know that you don't want to shut down the runtime environment after the ``main()`` function finishes. This allows it to discard the ``atexit`` and global destructor calls it would otherwise make, improving code size and startup speed.

	This is useful if your ``main()`` function finishes but you still want to execute code, for example in an app that uses a :ref:`main loop function <emscripten-runtime-environment-main-loop>`. 
	
	.. note:: Emscripten will not shut down the runtime if it detects :c:func:`emscripten_set_main_loop`, but it is better to optimise away the unnecessary code.


	
Very large projects
===================

This section describes optimisations and issues that are only relevant to very large projects.

.. _optimizing-code-memory-initialization:

Memory initialization
---------------------

By default Emscripten emits the static memory initialization code inside the **.js** file. This can cause the JavaScript file to be very large, which will slow down startup. It can also cause problems in JavaScript engines with limits on array sizes, resulting in errors like ``Array initializer too large`` or ``Too much recursion``. 

The ``--memory-init-file 1`` :ref:`emcc option <emcc-memory-init-file>` causes the compiler to emit this code in a separate binary file with suffix **.mem**. The **.mem** file is loaded (asynchronously) by the main **.js** file before ``main()`` is called and compiled code is able to run. 

.. note: From Emscripten 1.21.1 this setting is enabled by default for ``-O2`` builds (and above). 


.. _optimizing-code-oz-os:

Trading off code size and performance
-------------------------------------
You may wish to build the less performance-sensitive source files in your project using :ref:`-Os <emcc-Os>` or :ref:`-Oz <emcc-Oz>` and the remainder using :ref:`-O2 <emcc-O2>` (:ref:`-Os <emcc-Os>` and :ref:`-Oz <emcc-Oz>` are similar to :ref:`-O2 <emcc-O2>`, but reduce code size at the expense of performance. :ref:`-Oz <emcc-Oz>` reduces code size more than :ref:`-Os <emcc-Os>`.) 

.. note:: This only matters when compiling the source to bitcode. There are currently no JavaScript-specific optimization flags for ``-Os`` or ``-Oz``, and these map to ``-O2`` in the bitcode-to-JavaScript phase.

Code size
---------

Tips for reducing code size include:

- Define a separate memory initialization file (as :ref:`mentioned above <optimizing-code-memory-initialization>`).
- Use ``-Os`` or ``-Oz``  (as :ref:`mentioned above <optimizing-code-oz-os>`).
- Build bitcode to JavaScript with :ref:`-O3 <emcc-O3>`. This runs the expensive variable reuse pass (``registerizeHarder``). It is even more effective than ``-O2`` but slower to compile.
- Use :ref:`llvm-lto <emcc-llvm-lto>` when compiling from bitcode to JavaScript: ``--llvm-lto 1``. This can break some code as the LTO code path is less tested.
- Disable :ref:`optimizing-code-inlining`: ``-s INLINING_LIMIT=1``. Compiling with -Os or -Oz generally avoids inlining too.
- Use :ref:`closure <emcc-closure>` on the outside non-asm.js code: ``--closure 1``. This can break code that doesn't use `closure annotations properly <https://developers.google.com/closure/compiler/docs/api-tutorial3>`_.

.. _optimizing-code-outlining:

Outlining
---------

JavaScript engines will often compile very large functions slowly (relative to their size), and fail to optimize them effectively (or at all). One approach to this problem is to use "outlining": breaking them into smaller functions that can be compiled and optimized more effectively. 

Outlining increases overall code size, and can itself make some code less optimised. Despite this, outlining can sometimes improve both startup and runtime speed. For more information see For more information read `Outlining: a workaround for JITs and big functions <http://mozakai.blogspot.com/2013/08/outlining-workaround-for-jits-and-big.html>`_.

The ``OUTLINING_LIMIT`` setting defines the function size at which Emscripten will try to break large functions into smaller ones. Search for this setting in `settings.js <https://github.com/kripken/emscripten/blob/master/src/settings.js>`_ for information on how to determine what functions may need to be outlined and how to choose an appropriate function size.


.. _optimizing-code-aggressive-variable-elimination:

Aggressive variable elimination
-------------------------------

Aggressive variable elimination attempts to remove variables whenever possible, even at the cost of increasing code size by duplicating expressions. This can improve speed in cases where you have extremely large functions. For example it can make sqlite (which has a huge interpreter loop with thousands of lines in it) 7% faster. 

You can enable aggressive variable elimination with ``-s AGGRESSIVE_VARIABLE_ELIMINATION=1``. 

.. note:: This setting can be harmful in some cases. Test before using it.


Other optimization issues
=========================

.. _optimizing-code-exception-catching:

C++ exceptions
--------------

C++ exceptions are turned off by default in ``-O1`` (and above). This prevents the generation of ``try-catch`` blocks, which lets the code run much faster, and also makes the code smaller. 

To re-enable exceptions in optimized code, run *emcc* with ``-s DISABLE_EXCEPTION_CATCHING=0`` (see `src/settings.js <https://github.com/kripken/emscripten/blob/master/src/settings.js>`_).

.. _optimizing-code-inlining:

Memory Growth
-------------

Building with ``-s ALLOW_MEMORY_GROWTH=1`` allows the total amount of memory used to change depending on the demands of the application. This is useful for apps that don't know ahead of time how much they will need, but it disables some optimizations. (Work is ongoing to improve this.)

Inlining
--------

`Inlining <http://en.wikipedia.org/wiki/Inline_expansion>`_ often generates large functions, as these allow the compiler's optimizations to be more effective. Unfortunately large functions can be slower at runtime than multiple smaller functions because JavaScript engines often either don't optimize big functions (for fear of long JIT times), or they do optimize them resulting in noticeable pauses. 

.. note:: ``-O1`` and ``-O2`` inline functions by default. Ironically, this can actually decrease performance in some cases!

You can try to avoid this issue by disabling inlining (in specific files or everywhere), or by using :ref:`optimizing-code-outlining`.

Viewing code optimization passes
--------------------------------

Enable :ref:`debugging-EMCC_DEBUG` to output files for each JavaScript optimization pass.

.. _optimizing-code-unsafe-optimisations:

Unsafe optimizations
====================

A few **UNSAFE** optimizations you might want to try are:

- ``-s FORCE_ALIGNED_MEMORY=1``: Makes all memory accesses fully aligned. This can break on code that actually requires unaligned accesses.
- ``--llvm-lto 1``: This enables LLVM's link-time optimizations, which can help in some cases. However there are known issues with these optimizations, so code must be extensively tested. See :ref:`llvm-lto <emcc-llvm-lto>` for information about the other modes.
- ``--closure 1``: This can help with reducing the size of the non-generated (support/glue) code, and with startup. However it can break if you do not do proper :term:`Closure Compiler` annotations and exports.

.. _optimizing-code-profiling:

Profiling
=========

Modern browsers have JavaScript profilers that can help find the slower parts in your code. As each browser's profiler has limitations, profiling in multiple browsers is highly recommended. 

To ensure that compiled code contains enough information for profiling, build your project with :ref:`profiling <emcc-profiling>` as well as optimization and other flags:

.. code-block:: bash

	emcc -O2 --profiling file.cpp


Troubleshooting poor performance
================================

Emscripten-compiled code can currently achieve approximately half the speed of a native build. If the performance is significantly poorer than expected, you can also run through the additional troubleshooting steps below:

-  :ref:`Building-Projects` is a two-stage process: compiling source code files to LLVM **and** generating JavaScript from LLVM. Did you build using the same optimization values in **both** steps (``-O2`` or ``-O3``)?
-  Test on multiple browsers. If performance is acceptable on one browser and significantly poorer on another, then :ref:`file a bug report <bug-reports>`, noting the problem browser and other relevant information.
- Does the code *validate* in Firefox (look for "Successfully compiled asm.js code" in the web console). If you see a validation error when using an up-to-date version of Firefox and Emscripten then please :ref:`file a bug report <bug-reports>`.

