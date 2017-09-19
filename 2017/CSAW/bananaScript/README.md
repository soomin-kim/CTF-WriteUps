# bananaScript - Reversing 450

==============================

## Problem Description

Not too sure how to Interpret this, the lab member who wrote this "forgot" to
write any documentation. This shit, and him, is bananas. B, A-N-A-N-A-S.

## Given Materials

- [banana.script](banana.script)
- [monkeyDo](monkeyDo)
- [test1.script](test1.script)
- [test2.script](test2.script)

## Write Up

### Set the starting point

They gave a weird, consisting of only bananas, script language. It's interpreter
is `monkeyDo`. The main script including flag is `banana.script`, and
`test1.script` and `test2.script` are two very tiny examples.

If I opened the interpreter in the IDA pro and decompiled with Hex-ray plugin,
there were more than 3000 lines of decompiled codes. So it looked not a good
idea to analyze the entire binary statically from the first line to the last
line.

Instead, I ran the example scripts directly to see what happends.

Outcome of `test1.script`:
```
./monkeyDo test1.script
aa
aa
```
Program waits for a user to give input string, and if the input length is 2, it
prints the given input string.

Outcome of `test2.script`:
```
./monkeyDo test2.script
soomink@ubuntu
bananas
BANANAS
bafafaa
```

With these observations, I thought these examples represented basic I/O example
for this language. And both two examples have only six lines, I decided to start
with these examples. As I mentioned earlier, it seemed impossible to analyze
whole binary statically. So I used `gdb` to trace the interpreter in test-driven
manner.

// Actually I'm not good at English, I'll only use present tense after this
// point

### Analyze test2.script

I choose `test2.script` for the very first sample, because it does not interact
with users.

In the `main` function, there is a for-loop like below, so I think it is quite
reasonable to treat that loop as an interpreting routine. And if my assumption
is right, then the function `sub_40FAD8` should retrun the number of lines in
the input script file. And maybe the argument has a kind of structure that holds
some parsed inputs.
```
  for ( i = 0LL; sub_40FAD8(&v554) > i; ++i )
```

* The first line:
`banANAS baNanas BANANAs BANANAS BANanAs BANANAS BANanAs BANANAS BAnANaS`

There are 4 allocations for words, `bananaAS`, `bananaNAS`, `bananas`, and
`bananaS`. Then, there are two consecutive function calls of `sub_40FB10` and
`sub_40FB3A`. I can observe this sequence of function calls many times  within
`main` function. And they are just about pointer operations. After inspecting
the contents of return pointers, I conclude that `sub_40FB10` returns a
structure for each line of script, and `sub_40FB3A` returns a structure for each
word in the line. So at first, I get `banANAS` from these calls. Next, I meet an
If-branch which condition is the return value of `sub_40FB5A` and it seems to
compare two strings, one is the first word of a line, and the other is a
previously allocated word, `bananAS` at the beginning of for-loop. Maybe the
very first those words have special meanings.

Since this case does not belong to any of these cases, I just follow the
execution flow. Then I can find this code pattern.
```
        v355 = return_line(&v554, i);
        v356 = return_word(v355, 0LL);
        v363 = 0;
        if ( *std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::operator[](v356, 0LL) == 'b' )
        {
          v357 = return_line(&v554, i);
          v358 = return_word(v357, 0LL);
          if ( *std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::operator[](v358, 1LL) == 'a' )
          {
            v359 = return_line(&v554, i);
            v360 = return_word(v359, 0LL);
            if ( *std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::operator[](v360, 2LL) == 'n' )
            {
              v361 = return_line(&v554, i);
              v362 = return_word(v361, 0LL);
              if ( *std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::operator[](v362, 3LL) == 'A' )
                v363 = 1;
            }
          }
        }
```
You can often find this pattern of codes. It sets a variable to 1 if the target
word starts with `banA`. After, I realize `banA` is a kind of indicator for
variables. Then it checks whether the third word also starts with `banA`. Since
the third word does not start with `banA`, I jump to the next target code line.

It checks whether the second word is `baNanas` and if so, it concatenates words
from the 2nd word to the last word of the line and stores to appropriate
pointer. I think `baNanas` is a kind of assignment operator.

The second line looks similar, so from here, the content of the script is like
```
var0 = BANANAs BANANAS BANanAs BANANAS BANanAs BANANAS BAnANaS
var1 = BAnaNas BAnaNaS BaNAnas BAnaNaS BaNAnas BAnaNaS BaNanAS
```

* The third line:
`bananas banANAS`

When the first word of line is `bananas`, it checks whether the second word is
an indicator for variable. And if so, it calls `sub_405A9E` to convert bananas'
to ascii characters(And you may realize that the bananas values look like binary
numbers). Then it prints the converted string to `cout`.

The fourth line is similar to the third line, so we omit the analsis process and
so far, the interpreted result is (including the sixth line)
```
var0 = BANANAs BANANAS BANanAs BANANAS BANanAs BANANAS BAnANaS
var1 = BAnaNas BAnaNaS BaNAnas BAnaNaS BaNAnas BAnaNaS BaNanAS
print var0
print var1
...???
print var0
```

* The fifth line:
`banANAS baNaNas banANAs`

Now we only have the fifth line of the `test2.script`. It consists of only 3
words and the first and third words are variables. The second word is a kind of
operator between two variabes. The first word is a destination operand, and the
third word is a source operand.

The operator `baNaNas` works like this:
It scans two operands strings from the back of the string. And if two
corresponding characters are lower-case, then the result is lower-case.
Otherwise, the result is upper-case. So if we treat the upper-cases as 0s, it is
like `logical and` operation.

In summary, `test2.script` looks like:
```
var0 = 'enc[ 1].enc[ 0].enc[13].enc[ 0].enc[13].enc[ 0].enc[18]'
var1 = 'enc[27].enc[26].enc[39].enc[26].enc[39].enc[26].enc[44]'
print var0
print var1
var0 and= var1
print var0
```
