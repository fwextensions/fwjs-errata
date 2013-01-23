# Adobe Fireworks JavaScript Engine Errata

This document is a compendium of the bugs, hidden features and other weirdnesses lurking inside the JavaScript engine used to run extensions in Adobe Fireworks.  Feel free to fork this repo and contribute back your own findings.

To explore the JS examples here, it's helpful to install the [Fireworks Console](http://johndunning.com/fireworks/about/FWConsole) extension.  You can then enter some JS, evaluate it, and see the output.

If you’re a real masochist, you can take a look at the source code for the JavaScript interpreter, which is installed with the app in *Configuration\Third Party Source Code\JavaScript Interpreter*.  Doing so was how I discovered some of the undocumented features of the JS engine.  It appears to be based on version 1.5 of the Mozilla interpreter.  The version string in the source says `"JavaScript-C 1.5 pre-release 3 2001-03-07"`, so it looks like the code hasn’t been updated since **2001**.


# Native objects


## `FwDict` and `FwArray`

A number of Fireworks objects that are exposed in the JS environment look like arrays but are actually instances of the `FwDict` and `FwArray` class:

```JavaScript
fw.selection instanceof Array; // false
fw.selection instanceof FwArray; // true

fw.selection[0].customData instanceof FwDict; // true

fw.selection[0].customData.foo = [42]; 
fw.selection[0].customData.foo instanceof Array; // false
fw.selection[0].customData.foo instanceof FwArray; // true

dom.frames instanceof FwArray; // true
```

This is why the `fw.selection` object doesn’t have any useful `Array` methods, like `slice()`:

```JavaScript
typeof fw.selection.slice; // "undefined"
```

Both classes inherit from `Object` and seem to be roughly interchangeable.  


## `in` and `hasOwnProperty()` with native objects

Unfortunately, the `in` operator and `hasOwnProperty()` method don't seem to work with instances of native Fireworks objects.  In some cases, neither work, in others, just `hasOwnProperty()` doesn’t work:

```JavaScript
fw.selection[0].customData.foo = [42]; 
"foo" in fw.selection[0].customData; // false

dom.layers[0].name; // "Layer 1"
"name" in dom.layers[0]; // true
dom.layers[0].hasOwnProperty("name"); // false
```


## `customData`

The `customData` property on elements in a Fireworks document is extremely useful for storing extra data about the element that can then be used by a JS extension.  Unfortunately, there are some quirks to using `customData` that can trip you up if you’re not careful.  

One surprising thing is that if you add an object or array to an element's `customData` and then duplicate that element, the data is shared among the duplicates.  

```JavaScript
dom.addNewRectangle({ left: 0, top: 0, right: 100, bottom: 100 }, 0);
fw.selection[0].customData.foo = [42];
dom.cloneSelection();
fw.selection[0].customData.foo[1] = "bar"; // foo is now [42, "bar"]
dom.selectAll();
log(fw.selection[0].customData.foo, fw.selection[1].customData.foo);
```

This outputs `[42, "bar"] [42, "bar"]`, showing that the `customData` on both the original and the duplicate are pointing at the same array.  If you save and reopen the file, however, the two elements will be pointing at different arrays that can be updated independently.

The `delete` operator also doesn’t work with `customData` properties:

```JavaScript
fw.selection[0].customData.foo = [42];
delete fw.selection[0].customData.foo;
log(fw.selection[0].customData.foo); // still shows [42]
```

To remove a property from `customData`, the best you can do is simply set it to `null`.


## `dom.pngText`


## `Files.readLine()` is limited to 2047 characters

`Files.readLine()` will return only the first 2047 characters of a line of text in a file, so if a file has lines longer than that, your JS code will not be able to successfully read all of it.  This may particularly be an issue with JSON files, if you don’t take care to break the lines up.


## nodes


## Rect objects


# Bugs

## Assignment in `if` statements

Although normally an error, it should be possible to assign a value to a variable within the expression of an `if` statement.  However, the Fireworks JS engine interprets an assignment as a check for equality and doesn’t perform the assignment:

```JavaScript
var a = 0; 
if (a = 1) {
	log("a!"); 
}
a; // 0
```

Usually you don’t want to write code like this, but some JS minifiers, like Google's Closure Compiler, do make this micro-optimization, so the output from such compressors won’t work correctly in Fireworks.  


## `Function.toString()` reformats the source code

Most JS implementations seem to store the original source text and use that to return a function's string representation, which means that comments within the function are visible in the returned string.  The JS engine in Fireworks, however, seems to decompile the bytecodes to produce the string representation of a function, which means that comments are lost.  Other changes are made to the source, like adding semi-colons where they’re optional and reformatting and re-indenting the code.

For example, this code:

```JavaScript
function foo(
	a,
	b) 
{
	// this comment won't appear

if (a) log("a!")
	else 
	if (b) log("b!")
}
foo.toString();
```

Produces this output:

```JavaScript
"
function foo(a, b) {
    if (a) {
        log("a!");
    } else {
        if (b) {
            log("b!");
        }
    }
}
"
```

This normally isn’t a big deal, but some code libraries may expect to be able to see comments in a function's source code.

There is one case `Function.toString()` actually returns incorrect code.  If a function contains an object with a quoted property name:

```JavaScript
function foo() 
{
	var o = {
		"bar baz": [42]
	};
};
foo.toString();
```

Then calling `toString()` on that function returns code with a syntax error:

```JavaScript
"
function foo() {
    var o = {bar baz:[42]};
}
"
```

Note that the `"bar baz"` property isn’t quoted in the output, which it should be.


## Regular expressions

There are a number of annoying bugs in the JS engine's handling of regular expressions.  This code:

```JavaScript
"foo bar baz".match(/(?:\w+\s*){1,5}/)
```

will return `["foo bar baz"]` in a browser but only `["foo "]` in Fireworks.  

This expression:

```JavaScript
"http://api.twitter.com/oauth/request_token".match(/^([^:\/?#]+?:\/\/)*([^\/:?#]*)?(:[^\/?#]*)*([^?#]*)(\?[^#]*)?(#(.*))*/)
```

returns `["http://api.twitter.com/oauth/request_token", "http://", "api.twitter.com", undefined, "/oauth/request_token", undefined, undefined, undefined]` in a browser but `["http://api.twitter.com/oauth/request_token", "http://", "", "", ""]` in Fireworks.  

Using a captured group token, like `$1`, in the replacement string in a call to `replace()` will fail if the token is immediately followed by a number.  So a call like `"***".replace(/(\*)/g, "$12|")` will return `"*2|*2|*2|"` in a browser but `"|||"` in Fireworks.  It seems like the parser is interpreting the token in `"$12|"` as `$12`, rather than `$1` followed by a 2.


## `encodeURIComponent()`


## `[].sort()` returns `undefined`

In browsers, the `sort()` method returns the sorted array, but in Fireworks, it returns `undefined`:

```JavaScript
[3, 2, 1].sort() instanceof Array; // false
```

The array does get sorted, but you just can’t chain the call.


# Undocumented features

## `toSource()`

## `const`

## `watch()` and `unwatch()`

## Getters and setters

## `arguments.callee.call`

## __proto__

```JavaScript
```
