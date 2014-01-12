﻿# Re2.Net

* [Introduction](#introduction)
    * [Platforms](#platforms)
    * [License](#license)
    * [Binaries](#binaries)
* [Re2.Net vs .NET Regex](#comparison)
    * [Theory](#theory)
    * [New in Re2.Net](#new)
    * [Different in Re2.Net](#different)
    * [Missing from Re2.Net](#missing)
    * [The Rest](#the-rest)
* [Performance](#performance)    
    * [Interpreting the Results](#interpreting-the-results)
    * [First Match](#first-match)
    * [All Matches](#all-matches)

    
## Introduction

Re2.Net is a .NET wrapper for Google's [RE2 regular expression library](http://code.google.com/p/re2/). It has the benefit of also being a mostly drop-in replacement for .NET's ``System.Text.RegularExpressions`` namespace.

What does 'drop-in replacement' mean? Simple. If you have a project that uses the .NET ``Regex``, ``Match``, ``Group``, ``Capture``, or associated classes, you can add a reference to ``Re2.Net.dll``, replace ``using System.Text.RegularExpressions`` with ``using Re2.Net``, and more often than not, It Just Works.

For a list of things that Re2.Net doesn't support (yet), see [below](#missing).


#### Platforms

Re2.Net targets .NET Framework 3.5 and is compiled using the Visual C++ 2008 runtime, so it has plenty of backwards compatibility for the enterprise environments where .NET is so popular. x86 and x64 platforms are both supported, but because the underlying RE2 library is unmanaged code, each platform requires its own DLL (this will change to a single DLL at some point).


#### License

Re2.Net is distributed under the [GNU Affero General Public License, Version 3](http://www.gnu.org/licenses/agpl-3.0.html) (RE2 by itself is distributed under a more permissive [BSD-style license](http://re2.googlecode.com/hg/LICENSE)). If you're interested in using Re2.Net but find the terms of the AGPLv3 too restrictive, contact me at <coltblackmore+re2.net@gmail.com>.


#### <a name="binaries"/>Binaries ([Download](https://drive.google.com/file/d/0B9Jv-lX2VrxfcksxenQzelk0aG8/edit?usp=sharing))

Building C++/CLI from source can be a finicky process, especially with older .NET versions (like 3.5) and VC++ compilers (like 2008). If you just want binaries, use the Download link above. The LICENSES file and XML documentation are included.

If you *are* interested in building from source, see the **C++/CLI Notes** and **Lib Requirements** sections at the top of ``Regex.h``. Note that despite targeting .NET 3.5 and compiling with VC++ 2008, the Re2.Net solution files are for Visual Studio 2012 (the free Express edition works just fine).


## <a name="comparison"/> Re2.Net vs .NET Regex

#### Theory

Russ Cox, one of the RE2 authors, has written a [stellar series](http://swtch.com/~rsc/regexp/) on the challenges of implementing regular expressions and the virtues of the RE2 approach. For a brief overview, see the description at the public [RE2 repository](http://code.google.com/p/re2/).

In short, RE2 is automata-driven and uses [NFAs](http://en.wikipedia.org/wiki/Nondeterministic_finite_automaton) and [DFAs](http://en.wikipedia.org/wiki/Deterministic_finite_automaton) to find matches. This method guarantees linear running time and bounded memory consumption, but doesn't support backreferences or generalized assertions (lookahead and lookbehind).

.NET, on the other hand, uses a typical [backtracking](http://www.codinghorror.com/blog/2006/01/regex-performance.html) implementation. This method guarantees neither running time (whence the ``Regex.MatchTimeout`` property) nor memory consumption, and can even result in [catastrophic failure](http://en.wikipedia.org/wiki/ReDoS), but it allows for backreferences and generalized assertions.


#### <a name="new"/> New in Re2.Net

* All ``IsMatch()``, ``Match()``, and ``Matches()`` methods accept ``byte[]`` in addition to ``string`` arguments. Note that when matching a ``byte[]``, ``Capture.Index`` and ``Capture.Length`` refer to the ``byte[]``, and not the ``string`` returned by ``Capture.Value``. In practice, this means that

    ```C#
    Debug.Assert(Regex.Match("水Ǆ", "水Ǆ").Length == 2);
    Debug.Assert(Regex.Match(Encoding.UTF8.GetBytes("水Ǆ"), "水Ǆ").Length == 5);
    ```

    will both pass, and in both cases ``Capture.Value`` is ``"水Ǆ"``.

* Re2.Net adds ``RegexOptions`` for explicit support of Latin-1 and ASCII, as well as a POSIX mode. See [Different in Re2.Net](#different) for more about encodings.

* By combining ``byte[]`` arguments with ``RegexOptions.Latin1`` (or ``RegexOptions.ASCII``, for byte values 0-127), Re2.Net provides true byte-level searching.

* Because RE2 is automata-driven, in Re2.Net ``Regex`` memory consumption is configurable using the ``maxMemory`` constructor parameter.


#### <a name="different"/> Different in Re2.Net

* Re2.Net constructors and static matching methods have no ``matchTimeout`` parameter, because RE2 is immune to the problem (catastrophic failure) that the ``matchTimeout`` parameter solves. The ``Regex.MatchTimeout`` property has accordingly been removed.

* RE2 accepts most, but not all, .NET Regex syntax. For the full list, see the [RE2 Syntax page](http://code.google.com/p/re2/wiki/Syntax). One notable difference between RE2 and .NET Regex is the format for named capturing groups. RE2 uses ``(?P<name>regex)``, while .NET Regex allows both ``(?<name>regex)`` and ``(?'name'regex)``. Neither accepts the other's syntax.

* RE2 doesn't allow Unicode in named capturing groups. ``(?<水>regex)`` is legal for .NET Regex, but RE2 won't accept it (or rather, won't accept ``(?P<水>regex)``).

* RE2 only retains the final ``Capture`` of each ``Group``. In other words, ``Regex.Match("abcd", "(ab|cd)+")`` captures ``"abcd"`` for the ``Match``, but only ``"cd"`` for the ``Group``. In .NET Regex, the ``Group`` would hold two captures: ``"ab"`` *and* ``"cd"``. (Of course, 99% of the time you only care about capturing the ``Match`` as a whole. Re2.Net provides ``RegexOptions.SingleCapture`` for just such occasions.)

* Although an implementation detail, it should be noted that RE2 only accepts UTF-8 and Latin-1 encodings, while .NET encodes all strings in UTF-16. Re2.Net resolves this discrepancy by converting the UTF-16 .NET strings to UTF-8, Latin-1, or ASCII (a subset of Latin-1), depending on the chosen ``RegexOptions``. In general there's no need to give any thought to this, but in the specific case of searching a ``byte[]`` for a UTF-16 pattern, Re2.Net will fail to find a match because the UTF-16 pattern has been converted to UTF-8 while the ``byte[]`` has not. In the future I may add ``RegexOptions.UTF16`` and ``RegexOptions.UTF8`` to allow users to make their intent clear, and, if ``RegexOptions.UTF16`` is combined with a ``byte[]`` argument, throw an exception. In the meantime, avoid combining ``byte[]`` arguments with patterns that you require to be UTF-16-encoded.


#### <a name="missing"/> Missing from Re2.Net

* Some ``RegexOptions`` from .NET Regex are not present in Re2.Net (and vice-versa).

* The ``Regex.Replace()`` and ``Match.Result()`` methods are not yet supported, but will be.

* Serialization is not currently supported. RE2 constructs its automata more efficiently than a .NET Regex can be deserialized, so there isn't much reason to add this other than as a compatibility layer.

* Re2.Net depends on native code and won't run anywhere that the native code can't run (Silverlight, Mono, etc.).

* Tests. A small C# benchmarking project is included in the source, but thorough tests haven't been written yet.

 
#### The Rest

Everything not listed above should line right up. Here are a few examples of .NET Regex quirks that Re2.Net emulates:

```C#
// Five matches, each with a value of "".
Debug.Assert(Regex.Matches("xxxx", "").Count == 5);

// Accessing an array (technically, the default Item property) with a negative index? Yup!
Debug.Assert(Regex.Matches("xxxx", "")[0].Groups[-1].Value == "");

// The returned Match containes a Group collection, and the first item in the Group collection is the Match.
var match = Regex.Match("abcd", "abcd");
Debug.Assert((Group)match == match.Groups[0]);
```


## Performance

The Re2.Net.Test project pits Re2.Net against .NET Regex where each is strongest: searching strings, for .NET Regex, and for Re2.Net, searching bytes. The testing method is as follows:

* *The Entire Project Gutenberg Works of Mark Twain* (``mtent12.txt``) is loaded as a byte array for Re2.Net and then encoded as a .NET string for .NET Regex.
* The encoding process isn't included in the measurement, so the comparison shows each library at its best. (In the real world, .NET Regex suffers from having to encode anything that isn't a .NET string, and Re2.Net suffers from having to encode anything that *is* a .NET string.)
* Performance is measured across 16 regular expressions, taken from [Performance comparison of regular expression engines](http://sljit.sourceforge.net/regex_perf.html) at the the PCRE-sljit project. These expressions aren't pathological -- if anything, they're overly simple.
* Each expression is tested 11 times (discontinuously, to flush the cache), and then the *median* (not the average) result is used.
* The test is performed once for finding the first match (``Regex.Match()``) and once for finding all matches (``Regex.Matches()``).


#### Interpreting the Results

* ``Twain`` (**First Match**) demonstrates the cost of switching between managed and unmanaged code. Because the first instance of ``Twain`` ends a mere 64 positions into the input, .NET Regex manages to find it before Re2.Net can even call into the underlying RE2 library. Search for ``Twain`` (**All Matches**), on the other hand, and Re2.Net recovers nicely, running twice as fast as .NET Regex.
* The above pattern repeats itself throughout the **First Match** test, so it's clear that for short inputs, Re2.Net often costs more in unmanaged overhead than it saves in algorithmic efficiency. (Of course, if the cost of encoding bytes into a .NET string is included, Re2.Net routinely wins by a factor of thousands, given an input of this size.)
* The **All Matches** test confirms that once the duration of a search exceeds Re2.Net's unmanaged overhead, it quickly outpaces .NET Regex, and by significant margins for non-trivial expressions.

**Conclusion**: Re2.Net excels at searching raw data, like files or scraped web pages. But for simple expressions and short inputs, .NET Regex is still the better option (assuming linear running time, bounded memory consumption, and immunity to pathological expressions aren't considerations).

===
#### First Match

|Regular Expression|Re2.Net|.NET Regex|Winner
|---|---:|---:|:---:
<code>Twain</code>|0.05 ms|0.003 ms|.NET Regex by **16.7x**
<code>^Twain</code>|22 ms|4.7 ms|.NET Regex by **4.6x**
<code>Twain$</code>|11 ms|26 ms|Re2.Net by **2.4x**
<code>Huck[a-zA-Z]+&#124;Finn[a-zA-Z]+</code>|20 ms|122 ms|Re2.Net by **6.1x**
<code>a[^x]{20}b</code>|0.5 ms|0.007 ms|.NET Regex by **69.9x**
<code>Tom&#124;Sawyer&#124;Huckleberry&#124;Finn</code>|0.2 ms|1.3 ms|Re2.Net by **5.7x**
<code>.{0,3}(Tom&#124;Sawyer&#124;Huckleberry&#124;Finn)</code>|0.7 ms|20 ms|Re2.Net by **26.2x**
<code>[a-zA-Z]+ing</code>|0.05 ms|0.05 ms|Re2.Net by **1.1x**
<code>^[a-zA-Z]{0,4}ing[^a-zA-Z]</code>|0.2 ms|2.3 ms|Re2.Net by **9.6x**
<code>[a-zA-Z]+ing$</code>|107 ms|7300 ms|Re2.Net by **68.2x**
<code>^[a-zA-Z ]{5,}$</code>|110 ms|4001 ms|Re2.Net by **36.4x**
<code>^.{16,20}$</code>|0.4 ms|0.3 ms|.NET Regex by **1.3x**
<code>([a-f]\(.[d-m].){0,2}[h-n]){2}</code>|0.2 ms|0.02 ms|.NET Regex by **9.1x**
<code>([A-Za-z]awyer&#124;[A-Za-z]inn)[^a-zA-Z]</code>|2.9 ms|135 ms|Re2.Net by **46.2x**
<code>"[^"]{0,30}[?!\.]"</code>|0.8 ms|0.1 ms|.NET Regex by **5.6x**
<code>Tom.{10,25}river&#124;river.{10,25}Tom</code>|46 ms|332 ms|Re2.Net by **7.2x**

===
#### All Matches

|Regular Expression|Re2.Net|.NET Regex|Winner
|---|---:|---:|:---:
<code>Twain</code>|14 ms|28 ms|Re2.Net by **1.9x**
<code>^Twain</code>|121 ms|26 ms|.NET Regex by **4.7x**
<code>Twain$</code>|11 ms|27 ms|Re2.Net by **2.5x**
<code>Huck[a-zA-Z]+&#124;Finn[a-zA-Z]+</code>|115 ms|651 ms|Re2.Net by **5.7x**
<code>a[^x]{20}b</code>|578 ms|436 ms|.NET Regex by **1.3x**
<code>Tom&#124;Sawyer&#124;Huckleberry&#124;Finn</code>|127 ms|689 ms|Re2.Net by **5.4x**
<code>.{0,3}(Tom&#124;Sawyer&#124;Huckleberry&#124;Finn)</code>|133 ms|21152 ms|Re2.Net by **158.9x**
<code>[a-zA-Z]+ing</code>|215 ms|7195 ms|Re2.Net by **33.4x**
<code>^[a-zA-Z]{0,4}ing[^a-zA-Z]</code>|122 ms|3104 ms|Re2.Net by **25.4x**
<code>[a-zA-Z]+ing$</code>|110 ms|7545 ms|Re2.Net by **68.3x**
<code>^[a-zA-Z ]{5,}$</code>|114 ms|4129 ms|Re2.Net by **36.3x**
<code>^.{16,20}$</code>|134 ms|3775 ms|Re2.Net by **28.2x**
<code>([a-f]\(.[d-m].){0,2}[h-n]){2}</code>|378 ms|3967 ms|Re2.Net by **10.5x**
<code>([A-Za-z]awyer&#124;[A-Za-z]inn)[^a-zA-Z]</code>|123 ms|5691 ms|Re2.Net by **46.3x**
<code>"[^"]{0,30}[?!\.]"</code>|29 ms|188 ms|Re2.Net by **6.4x**
<code>Tom.{10,25}river&#124;river.{10,25}Tom</code>|123 ms|921 ms|Re2.Net by **7.5x**
