---
title: "The Hidden Pitfalls of Regex You Cannot Afford to Ignore"
date: 2024-12-30T16:12:34Z
draft: false
images: 
- the-hidden-pitfalls-of-regex-you-cant-afford-to-ignore.gif
tags: ['regex', 'devops']
summary: "Regex. Great when it works, hilariously bad when it doesn't. A lot of regexes out in the wild don't account for international Unicode characters. It's incredibly simple to use up significant system resources if you're not careful. Lastly, it's easy to overlook the shenanigans that loosely typed languages can introduce."
---

Regex. Great when it works, hilariously bad when it doesn't. A lot of regexes out in the wild don't account for international Unicode characters. It's incredibly simple to use up significant system resources if you're not careful. Lastly, it's easy to overlook the shenanigans that loosely typed languages can introduce.

Even the worldwide outage caused by [Cloudstrike](https://www.crowdstrike.com/wp-content/uploads/2024/08/Channel-File-291-Incident-Root-Cause-Analysis-08.06.2024.pdf) back in 2024 was thanks to regex gone wrong.

![Cloudstrike outage in 2024](/the-hidden-pitfalls-of-regex-you-cant-afford-to-ignore.gif 'Cloudstrike outage in 2024')

Now with various AI tools trying to promote simple regex solutions for [deceptively hard problems](https://stackoverflow.com/questions/46155/how-can-i-validate-an-email-address-in-javascript), it's time we have a good, long talk about the problems with most regex solutions.

# Some quick history before we talk about modern regex

When computers were starting to get big back in the 1960s, it was mostly a US affair. That meant that characters such as the one used in this article were mostly all that was used. 

Computers were relatively new at the time, but [catching on quick](http://tronweb.super-nova.co.jp/characcodehist.html). Prior methods of character encoding, such as Baudot (which allowed for 55 printed characters) or Hollerith (69 characters) were falling short of new needs that required the entire English character set available via traditional typewriters. From that need came the ASCII character encoding and standard, which eventually made it possible to utilize 190 printed characters. 

ASCII stores characters as single bytes, and from this, it can have 255 different code points. How exactly this breaks down between standard ASCII and extended ASCII can be found on any [ASCII code chart](https://www.ascii-code.com/).

Once the internet kicked off, there was a huge problem. Communication across different countries became extremely difficult - everyone was using their own encodings! You'd fire an email off to a colleague overseas and get back nonsense. Your computer wouldn't be able to make sense of the characters, unless of course, you had the same character encoding as them. 

To fix this, Unicode was created. Unicode is backwards compatible with ASCII - the same spots in memory that ASCII lives is also where it lives in Unicode. However, Unicode allows for a virtually infinite amount of characters to be stored, so it easily holds every language out there and emojis. 

Single-byte unicode, UTF-8, is almost always the default standard across systems these days. Combined with an installed font that supports the charset, it is what allows you to read things such as Japanese and emoji, along with good ol' fashioned English. 

A really good video that explains Unicode, its origins, and its inner workings is this one by Tom Scott:

{{< youtube MijmeoH9LT4 >}}

# The glaring problem with most regex implementations 

With that bit of history, it should now be a bit more clear that these characters are not at all the same:

`A`, `Ａ`, `Á`

If you write the regex `A-Z`, it will only match to the first `A` character, and none of the others.

Strictly matching to ASCII ranges is okay for very few implementations. Anything related to DNS, such as [kubernetes dns subdomain name configs](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-subdomain-names) or domain names [themselves](https://en.wikipedia.org/wiki/Punycode), can and should be matched using `[A-Za-z0-9\_\.\-]`. 

Virtually everything else needs to be handled a bit differently. Especially when you're dealing with user input data.

For example, say you want to match the first 3 letters of any Latin-based string. You could write this:

`/^[A-Za-z]{3}/gm`

And that will only match to the following in this test case:

A hello world
Ａtest
**Gre**g
**Kim**
Ámi
**Bry**an
**Ame**lia
....
5452423
«¬®¯
平仮名
ひらがな

Problematically, roman text inputs from usually Asian users (in their [fullwidth forms](https://en.wikipedia.org/wiki/Halfwidth_and_Fullwidth_Forms_(Unicode_block))) and any accented ASCII character is not included.

A better way to handle this would be to instead refer to the [Unicode Latin Script](https://en.wikipedia.org/wiki/Latin_script_in_Unicode), which matches to any Latin script code point in Unicode. 

In node.js, such a regex would look like: 

`/^[\p{Script=Latin}]{3}/gmu`

And it would match to all of the following:

A hello world
**Ａte**st
**Gre**g
**Kim**
**Ámi**
**Bry**an
**Ame**lia
....
5452423
«¬®¯
平仮名
ひらがな

## Tangent: Sorting multibyte unicode data

Sorting ASCII data is fairly straightforward. 

Since all ASCII maps to 0-255, you can crumble that cookie multiple ways, one of which is achieved just by ordering it based on the decimal, octal, or hexidecimal values of its memory addresses. 

Multibyte data is not as straightforward. If we take our regex matches and sort them, we get the following:

```javascript
> let matches = [
    "Ａte",
    "Gre",
    "Kim",
    "Ámi",
    "Bry",
    "Ame"
]
> matches.sort()
[ 'Ame', 'Bry', 'Gre', 'Kim', 'Ámi', 'Ａte' ]
```

All the matches that start with Unicode get shoved to the end of the list, despite them clearly starting with an A character.

To fix this, we must use a (Collator)[https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Collator]. 

Language is a very human thing. One way we interpret proper alphabetic sorting in our languages is dependent on the locale. `en-US` may have different sorting preferences to `de-DE`. 

While Unicode themselves have a long winded [document]((https://www.unicode.org/reports/tr35/tr35-collation.html#Contents) detailing what all is possible with fine tuning your locale to fit your collation needs, it is up to your language of choice to [implement it](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl#locales_argument).

In order to properly sort Unicode, you need to pass the sort function the locale. 

```javascript
> matches.sort(new Intl.Collator('en-US').compare)
[ 'Ame', 'Ámi', 'Ａte', 'Bry', 'Gre', 'Kim' ]
```

# Evil regexes will eat your system resources while also being hard to find

Much like the classic nested for loop example that we all remember when learning Big O, regexes can eat up your system resources if written in a way that results in [catastrophic backtracking](https://www.regular-expressions.info/catastrophic.html). 

For example, let's say you have this regex:

`/(.+)+@/gm`

In testing, this matches both of these:

**hello.\_dfadfdsafdgdsfgf**_@_world.com
**woooo**_@_world.com

There's a big problem though. It's revisiting the captured group repeatedly to determine a match, causing the regex to fire off 15,347 steps that take 1.35 ms to correctly resolve a match. 
You end up with the classic `O(2^n)` exponential complexity as the test strings increase in length.

The non-greedy and correct version of that regex would be closer to `/.+?@/gm` - although email can be deceptive in regexes so perhaps a string split function would be a better idea.

Some absolute stellar examples of catastrophic backtracking can be found on this [stackoverflow](https://stackoverflow.com/questions/12841970/how-can-i-recognize-an-evil-regex) question.

Tools like [Regex101](https://regex101.com/) and its Regex Debugger can be utilized to help you find such evil regexes before they make their way into your production environment. 

# Regex problems in loosely typed languages

Based on Cloudflare's report, I'm going to take a guess at what maybe have happened that resulted in the [worldwide computer outage](https://en.wikipedia.org/wiki/2024_CrowdStrike-related_IT_outages) on July 19, 2024.

In the report, there's a paragraph that reads:

> The new IPC Template Type defined 21 input parameter fields, but the integration code that invoked the Content Interpreter with Channel File 291’s Template Instances supplied only 20 input values to match against. This parameter count mismatch evaded multiple layers of build validation and testing, as it was not discovered during the sensor release testing process, the Template Type (using a test Template Instance) stress testing or the first several successful deployments of IPC Template Instances in the field. In part, this was due to the use of wildcard matching criteria for the 21st input during testing and in the initial IPC Template Instances

Which to me, and I may be wrong on this, but it seems to be a a long winded way of saying, "we passed 21 config parameters to our build, one got turned into a null, and thanks to matching with a wildcard regex, we corrupted a script at the kernel layer of computers worldwide."

When you're working with a loosely typed language - aka one that does not force you to define the type of the variable you're working with - this kind of problem is really easy to achieve. Either because the parameter you've passed in is automagically interpreted as both what you'd expect _and_ a string, or because the language has adopted loose typing so it can try and automagically fix any parameters you pass to various functions. 

For javascript, in the following example, it ends up being the latter:

```javascript
> let regex = /^[A-Za-z\.\_\-]+$/g

// these results are what we'd expect
> regex.test("this-is-okay")
true
> regex.test("this is not okay!")
false

// and now for javascript to do the unexpected
> regex.test(null)
true
> regex.test(NaN)
true
```

What is happening here is `Regex.prototype.test()` is specified to do the [following](https://tc39.es/ecma262/multipage/text-processing.html#sec-regexp.prototype.test):

> 22.2.6.16 RegExp.prototype.test ( S )
> This method performs the following steps when called:
>
> 1. Let _R_ be the **this** value.
> 2. If _R_ is not an _Object_, throw a **TypeError** exception.
> 3. Let _string_ be ? _ToString(S)_.
> 4. Let _match_ be ? _RegExpExec(R, string)_.
> 5. If _match_ is not **null**, return **true**; else return **false**.

Somewhere in the process of running `test()`, javascript sets our parameter to a string. So suddenly, and with some warning if you're familiar with javascript's insanity around typing, `null` goes from being **null** to a **string**.

In order to avoid this, regexes in loosely typed language _must_ only be used after initial checks for other things, such as null values, are passed.

# Conclusion

Regexes are great, but you need to keep their pitfalls in mind when you use them. 

Code responsibly. ☕


