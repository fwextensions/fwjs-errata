# Adobe Fireworks JavaScript Engine Errata

This document is a compendium of the bugs, hidden features and other weirdnesses lurking inside the JavaScript engine used to run extensions in Adobe Fireworks.  Feel free to fork this repo and contribute back your own findings.

To explore the JS examples here, it's helpful to install the [Fireworks Console](http://johndunning.com/fireworks/about/FWConsole) extension.  You can then enter some JS, evaluate it, and see the output.

If you’re a real masochist, you can take a look at the source code for the JavaScript interpreter, which is installed with the app in *Configuration\Third Party Source Code\JavaScript Interpreter*.  Doing so was how I discovered some of the undocumented features of the JS engine.  It appears to be based on version 1.5 of the Mozilla interpreter.  The version string in the source says `"JavaScript-C 1.5 pre-release 3 2001-03-07"`, so it looks like the code hasn’t been updated since **2001**.


# Contents

- [Native objects](#native-objects)
	- [FwDict and FwArray](#fwdict-and-fwarray)
	- [in and hasOwnProperty() with native objects](#in-and-hasownproperty-with-native-objects)
	- [customData](#customdata)
	- [dom.pngText](#dompngtext)
	- [Files.readLine() is limited to 2047 characters](#filesreadline-is-limited-to-2047-characters)
- [Bugs](#bugs)
	- [Assignment in if statements](#assignment-in-if-statements)
	- [Function.toString() reformats the source code](#functiontostring-reformats-the-source-code)
	- [Regular expressions](#regular-expressions)
	- [decodeURI(),decodeURIComponent(),encodeURI(), and encodeURIComponent() crash Fireworks](#decodeuridecodeuricomponentencodeuri-and-encodeuricomponent-crash-fireworks)
	- [[].sort() returns undefined](#sort-returns-undefined)
	- [Naming a variable nodes in auto shape code causes an error](#naming-a-variable-nodes-in-auto-shape-code-causes-an-error)
	- [System.osName is wrong](#systemosname-is-wrong)
	- [fw.appName is wrong](#fwappname-is-wrong)
	- [dom.setElementLocked() and dom.setElementVisible() index elements differently than dom.frames[0].layers[0].elements](#domsetelementlocked-and-domsetelementvisible-index-elements-differently-than-domframes0layers0elements)
	- [Setting the locked, visible or disclosure property of a sub-layer throws an exception](#setting-the-locked-visible-or-disclosure-property-of-a-sub-layer-throws-an-exception)
	- [dom.frames[m].layers[n] always returns the layer from the current frame](#domframesmlayersn-always-returns-the-layer-from-the-current-frame)
	- [The frame index in all dom methods is ignored](#the-frame-index-in-all-dom-methods-is-ignored)
- [Undocumented features](#undocumented-features)
	- [toSource()](#tosource)
	- [File class](#file-class)
	- [const](#const)
	- [Getters and setters](#getters-and-setters)
	- [watch() and unwatch()](#watch-and-unwatch)
	- [`__call__` and `__parent__`](#__call__-and-__parent__)
	- [dom.topLayers](#domtoplayers)
	- [fw.takeScreenshot()](#fwtakescreenshot)
	- [fw.getFWLaunchLanguage()](#fwgetfwlaunchlanguage)
	- [fw.commonLibraryDir](#fwcommonlibrarydir)
	- [fw.reloadPatterns() and fw.reloadTextures()](#fwreloadpatterns-and-fwreloadtextures)
	- [fw.browseForFileURL()](#fwbrowseforfileurl)
	- [fw.setDocumentImageSize()](#fwsetdocumentimagesize)
	- [Tools](#tools)
	- [$](#)
	- [fw.shiftKeyDown(), fw.ctrlCmdKeyDown(), fw.altOptKeyDown()](#fwshiftkeydown-fwctrlcmdkeydown-fwaltoptkeydown)
	- [fw.undo() and fw.redo()](#fwundo-and-fwredo)
	- [fw.uiOK](#fwuiok)


# Native objects

In addition to standard JS objects, the Fireworks environment exposes access to various Fireworks-specific objects, which don’t always behave like normal JS objects. 
 

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

Another weirdness occurs if you try to store an object on `customData` that contains the properties `top`, `right`, `bottom`, and `left`.  Fireworks will force the values of those properties to be numbers, even if they’re set to a different value:

```JavaScript
el.customData.foo = { left: "a", top: "b", right: "c", bottom: "d", bar: 42 };
log(el.customData.foo);
```

This leaves `customData.foo` actually set to `{ bottom: 0, left: 0, right: 0, top: 0 }`.  It appears that Fireworks treats any object on `customData` with those properties as a bounds object, forces all the values to be numeric and strips off any properties beyond the four sides.  The only way to work around this is to change the names of your properties so they don’t trigger this special handling.  

The same thing happens with an object containing `x` and `y` properties:

```JavaScript
el.customData.foo = { x: "a", y: "b" };
log(el.customData.foo); // { x: 0, y: 0 }
```


## `dom.pngText`

The `pngText` property of Fireworks documents is an `FwDict` that can be used to persistently store arbitrary data in the document.  While somewhat similar to `customData`, it's even more limited, as it supports storing only string values. So if you do:

```JavaScript
dom.pngText.foo = [1, 2, 3];
```

and then save the file, the next time you reopen it, the array value will have been turned into the source string version of that value: `"[1,2,3]"`. 

Even more confusing, `dom.pngText.foo` will appear to still be an array immediately after you set it.  It's only after the document is saved, closed and reopened do you discover it's been converted into a string.

And to make matters worse, each property on `dom.pngText` is limited to 1023 characters.  So if you think you can just stringify some data and store it, think again.  It will get cut off if it's too long.

The [`DomStorage`](http://htmlpreview.github.com/?https://raw.github.com/fwextensions/fwlib/master/docs/a321565296.html) class offers a way to more easily store arbitrary data on `dom.pngText` by turning it into a JSON string and then automatically chunking it up into 1023-character strings.  When the data is restored, the chunks are combined and then evaluated. 

Note that like `customData`, there’s no way to delete a property from `dom.pngText`.  You can only set the property to `null`.

Also note that each page in a Fireworks document has its own independent `pngText` property.  The easiest way to deal with this in an extension is just to always store data on the first page's `pngText`.  


## `Files.readLine()` is limited to 2047 characters

`Files.readLine()` will return only the first 2047 characters of a line of text in a file, so if a file has lines longer than that, your JS code will not be able to successfully read all of it.  This may particularly be an issue with JSON files, if you don’t take care to break the lines up when initially saving it.


# Bugs

There are some flat-out bugs in the JS engine, most of which can’t really be worked around.


## Assignment in `if` statements

Although usually unintended, the JavaScript syntax allows you to assign a value to a variable within the expression of an `if` statement.  However, the Fireworks JS engine interprets an assignment as a check for equality and doesn’t perform the assignment:

```JavaScript
var a = 0; 
if (a = 1) {
	log("a!"); 
}
a; // 0
```

Usually you don’t want to write code like this, but some JS minifiers, like Google's Closure Compiler, do make this micro-optimization, so the output from such compressors won’t work correctly in Fireworks.  (Fortunately, minification isn’t a big deal for Fireworks extensions, since the code isn’t being loaded from the internet.)


## `Function.toString()` reformats the source code

Most JS implementations seem to store the original source text and use that to return a function's string representation, which means that comments within the function are visible in the returned string.  The JS engine in Fireworks, however, seems to decompile the bytecodes to produce the string representation of a function, which means that comments are lost.  Other changes are made to the source, like adding semi-colons where they’re optional and reformatting and re-indenting the code.

For example, this code:

```JavaScript
function foo(
	a,
	b) 
{
	// this comment won't appear

if (a) log("a!")  // note the lack of braces
	else if (b) log("b!")
}
foo.toString();
```

Produces this output:

```JavaScript
'
function foo(a, b) {
    if (a) {
        log("a!");
    } else {
        if (b) {
            log("b!");
        }
    }
}
'
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

then calling `toString()` on that function returns code with a syntax error:

```JavaScript
"
function foo() {
    var o = {bar baz:[42]};
}
"
```

Note that the `"bar baz"` property isn’t quoted in the output, which it should be, so if you try to `eval()` the output, you’ll get an error.


## Regular expressions

There are a number of annoying bugs in the JS engine's handling of regular expressions.  This code:

```JavaScript
"foo bar baz".match(/(?:\w+\s*){1,5}/)
```

will return `["foo bar baz"]` in a browser but only `["foo "]` in Fireworks.  

In a browser, this expression:

```JavaScript
"http://api.twitter.com/oauth/request_token".match(/^([^:\/?#]+?:\/\/)*([^\/:?#]*)?(:[^\/?#]*)*([^?#]*)(\?[^#]*)?(#(.*))*/)
```

returns 

```JavaScript
["http://api.twitter.com/oauth/request_token", "http://", "api.twitter.com", undefined, "/oauth/request_token", undefined, undefined, undefined]
```

But in Fireworks, it just returns:

```JavaScript
["http://api.twitter.com/oauth/request_token", "http://", "", "", ""]
```

Using a captured group token, like `$1`, in the replacement string in a call to `replace()` will fail if the token is immediately followed by a number.  So a call like `"***".replace(/(\*)/g, "$12|")` will return `"*2|*2|*2|"` in a browser but `"|||"` in Fireworks.  It seems like the parser is interpreting the token in `"$12|"` as `$12`, rather than `$1` followed by a 2.


## `decodeURI()`,`decodeURIComponent()`,`encodeURI()`, and `encodeURIComponent()` crash Fireworks 

Calling any of these functions will either make the JS engine unstable or cause Fireworks to crash completely.  The only workaround is to use `escape()` and `unescape()`, which aren’t perfect replacements, or write your own versions of those functions.  


## `[].sort()` returns `undefined`

In browsers, the `sort()` method returns the sorted array, but in Fireworks, it returns `undefined`:

```JavaScript
[3, 2, 1].sort() instanceof Array; // false
```

The array does get sorted, but you just can’t chain the call.


## Naming a variable `nodes` in auto shape code causes an error

In auto shape code, you often want to refer to the nodes in one of the shape's path contours.  You might think of storing a reference to the nodes in a variable called `nodes`, like `var nodes`.  But you’d be sadly mistaken.  Doing so will cause an exception to be thrown.  You have to use a different name for the variable, like `nds`.  Senocular also notes this bug at the end of his list of [auto shape gotchas](http://senocular.com/fireworks/tutorials/autoshapes/?page=4).

There is a special circle of hell reserved for whoever is responsible for this bug. 


## `System.osName` is wrong

Checking `System.osName` on Windows 7 returns `"Windows XP"`.


## `fw.appName` is wrong

`fw.appName` returns `"Fireworks 10"` in Fireworks CS6, even though CS6 is version 12.


## `dom.setElementLocked()` and `dom.setElementVisible()` index elements differently than `dom.frames[0].layers[0].elements`

The `elements` array of a layer is indexed from the top-most element in the layer downwards.  When you call `dom.setElementLocked()`, the third parameter is supposed to be the index of the element you want to lock or unlock.  But passing in `0` actually toggles the lock state of the bottom-most element.  

So it seems like `dom.setElementLocked()` indexes the layer's elements in the opposite direction from the `elements` array.  To convert from the `elements` index to the index used by `dom.setElementLocked()`, subtract it from the length of the array minus 1.  For example, if there are 5 elements on the layer and you want to change the locked state of the second one from the top (with index 1), you’d do `(5 – 1) – 1 == 3`.

The `dom.setElementVisible()` method has the same bug. 


## Setting the `locked`, `visible` or `disclosure` property of a sub-layer throws an exception

You can lock or hide a layer by doing something like `dom.layers[1].frames[0].visible = true`.  But if layer 1 is actually a sub-layer, then this code will throw an exception that says *Could not run the script. A parameter was incorrect.*.  The workaround is to use the `dom.setLayerLocked()`, `dom.setLayerVisible()` and `dom.setLayerDisclosure()` methods instead.  

Just accessing the `locked`, `visible` or `disclosure` properties without changing them works fine for sub-layers.  Note that the `disclosure` property is accessed directly on the layer, like `dom.layers[1].disclosure`.


## `dom.frames[m].layers[n]` always returns the layer from the current frame

If your document has multiple frames and sub-layers, then the layer named `"Content"`, say, may have a different index on different frames.  This is because sub-layers don’t appear on every frame, yet they’re included in the `layers` array.  

However, if your document has 3 frames and `dom.currentFrameNum` is `0`, then `dom.frames[1].layers[2].name` doesn’t return the name of the third layer in the `layers` array on the second frame.  It actually returns the third layer on the first frame.  No matter which index you use for `dom.frames[]`, the `layers` array for that frame always lists the layers from the current frame.  The only way to work around this is to change `dom.currentFrameNum` to the desired frame before accessing that frame's layers. 


## The frame index in all dom methods is ignored

Methods like `dom.setLayerLocked()` have a parameter that is supposed to specify which frame the given layer is on, since layers can have different locked and visible states on different frames.  Unfortunately, the frame parameter seems to be ignored.  If you call `dom.setLayerLocked(1, 2, true, false)` to lock layer 1 on frame 2 while `dom.currentFrameNum` is 0, then layer 1 on frame 0 will get locked instead.  The only work around is to change `dom.currentFrameNum` to the desired frame before calling the method.  

The buggy methods include:

* `dom.setLayerLocked()`
* `dom.setLayerVisible()`
* `dom.setElementLocked()`
* `dom.setElementName()`
* `dom.setElementVisible()`


# Undocumented features

Many of these undocumented features aren’t unique to Fireworks.  They were standard features in version 1.5 of the Mozilla JS engine, but not necessarily standard across other browsers.  


## `toSource()`

Most browsers don’t have the `toSource()` method, but it's incredibly useful.  It returns a JavaScript representation of the object you call it on.  It's basically a poor-man's JSON, without the quoted properties.

```JavaScript
var foo = {
	bar: [42]
};
foo.toSource(); // "({bar:[42]})"
```

Since you can take the string returned by `toSource()` and pass it to `eval()` to recreate the original object, this method provides a simple way of making a deep copy of an object:

```JavaScript
function copyObject(
	inObject)
{
	if (typeof inObject == "object" && inObject !== null) {
		return eval("(" + inObject.toSource() + ")");
	} else {
		return inObject;
	}
}
```


## `File` class

There’s a documented `Files` object in Fireworks, which provides methods for working with files.  Unfortunately, this object doesn’t offer any way to get a file's size or modification date.  Fortunately, there’s also an undocumented `File` class lurking in the JS engine: 

```JavaScript
var f = new File("C:\\Projects\\test.js");
log(f.exists); // true if the file exists
```

Note that unlike the methods of the documented `Files` object, the path passed to the `File` constructor must be a standard OS path, not a `file://` URL.  So on Windows it would contain backslashes, and on the Mac it would contain forward slashes. 

A `File` instance has the following properties and methods:

* `exists`: flag indicating whether the file at the path exists.
* `length`: the size of the file in bytes.
* `created`: the creation date of the file, as a JavaScript `Date`.
* `modified`: the last modified date of the file, as a JavaScript `Date`.
* `open(mode)`: pass `"append"` to open the file for appending, which is otherwise impossible to do.
* `writeln(text)`: writes the text to the file *without* appending a newline, despite the method name.
* `write(text)`: writes the text to the file *with* appending a newline, despite the method name.

There’s a `read()` method as well, though it doesn’t work very well.  Its first parameter should be the number of bytes to read, but it only ever seems to read one byte.  It's better to use `Files.open()` to open a file for reading and then call `readline()` to read in the whole file. 

Note that the `modified` and `created` dates have a bug where they’re exactly a month in the future compared to the actual date.  The `files` module in the [fwlib repo](https://github.com/fwextensions/fwlib) provides `getCreatedDate()` and `getModifiedDate()` methods that correct this bug.  


## `const`

This statement defines a read-only constant value in the current function scope:

```JavaScript
const foo = 42;
foo = "bar";
log(foo); // still 42
```

Assignments to a `const` are silently ignored.  This statement has been available within [Firefox](https://developer.mozilla.org/en-US/docs/JavaScript/Reference/Statements/const) for a long time. 


## Getters and setters

The cutting edge ECMAScript 5 includes getters and setters as part of the specification, but it turns out that Fireworks has had these features for years:

```JavaScript
var o =
  {
    get property() { log("gotten!"); return "get"; },
    set property(v) { log("sotten!  " + v); }
  };
var v = o.property; // prints "gotten!", v === "get"
o.property = "new"; // prints "sotten!  new"
```

There are two other syntaxes for adding getters and setters:

```JavaScript
var o = {};
o.property getter = function() { log("gotten!"); return "get" };
o.property setter = function() { log("sotten!"); };
var v = o.property; // prints "gotten!", v === "get"
o.property = "new"; // prints "sotten!"

var o = {}; 
o.__defineGetter__("property", function(n, o) { log("gotten!"); return "get"; });
o.__defineSetter__("property", function(n, o) { log("sotten!"); });
var v = o.property; // prints "gotten!", v === "get"
o.property = "new"; // prints "sotten!"
```

Unfortunately, there doesn’t appear to be a way to get access to the getter or setter function via `__lookupGetter__` or `__lookupSetter__` methods that were added in later versions of the Mozilla engine.  There also is no way to control whether the getters/setters are enumerable.  If you use the `__defineGetter__` syntax, the resulting property seems to not be enumerable, whereas if you use the object literal syntax, it is.  

See this [blog post](http://whereswalden.com/2010/04/16/more-spidermonkey-changes-ancient-esoteric-very-rarely-used-syntax-for-creating-getters-and-setters-is-being-removed/) for more information on this old getter/setter syntax, which has since been removed from the Mozilla engine.


## `watch()` and `unwatch()`

Calling the `watch()` method on an object and passing the name of a property on that object and a function handler will cause that function to be called whenever the property's value changes.  This works for pure JS objects, but it unfortunately doesn’t work with the native Fireworks objects, which makes it a lot less useful.  The Mozilla developer site has [more information](https://developer.mozilla.org/en-US/docs/JavaScript/Reference/Global_Objects/Object/watch).

The `console.watch()` and `console.unwatch()` methods of the Fireworks Console use these functions. 


## `__call__` and `__parent__`

The `__call__` property of a function is an object containing a property for every parameter and local variable in the function.  This lets a called function actually peek into the state of the function that called it, via `arguments.callee.caller.__call__`.  In fact, the `__call__` property is mutable, which lets you reach into a function and actually create local variables:

```JavaScript
 (function() {
    function foo()
    {
        bar();
        log(baz);
    }

    function bar()
    {
        arguments.callee.caller.__call__.baz = "hello, foo";
    }

    foo(); // hello, foo
})();
```

In this example, even though the `baz` variable isn’t defined in the `foo()` function, it exists after the call to `bar()`, which modifies `foo()`'s `__call__` property. 

The somewhat related `__parent__` property of objects provides access to the scope that contains that object.  For a global variable, this would be the global scope, but for a function defined within another function, the inner function's `__parent__` would be the outer function.  The property contains an attribute for every variable or function defined within that scope.

While these properties are fairly arcane, they were crucial for adding the [`trace()`](https://github.com/fwextensions/trace#how-trace-works) method to the Fireworks Console.


## `dom.topLayers`

This property is an array of the top-level layers (no sub-layers), which is otherwise a pain to figure out from the `dom.layers` array.  This can also be accessed via a frame object, like `dom.frames[0].topLayers`. 


## `fw.takeScreenshot()`

Displays a dialog telling you to switch to the window you want to take a screenshot of.  Once you have, you click the *OK* button to switch to a crosshair mode.  You can then drag out a rectangle over the area you want to capture.  When you release the mouse, that part of the screen is copied to the clipboard as an image.  


## `fw.getFWLaunchLanguage()`

Returns a string with the code for the language and locale in which Fireworks is displaying its UI, like `"en_US"`.


## `fw.commonLibraryDir`

Returns the path to the *Common Library* directory, which is *<installDrive>:\Users\<username>\Appdata\Roaming\Adobe\Fireworks CS6\Common Library* on Windows and *<HD>/Library/Users/<username>/Library/Application Support/Adobe/Fireworks CS6/Common Library* on OS X.  This was added in Fireworks CS6.


## `fw.reloadPatterns()` and `fw.reloadTextures()`

These methods reload the patterns and textures directories, respectively, so that the drop-downs in the *Properties* panel update to show the currently available files.  This is useful if your extension creates a new pattern or texture and you want it to be immediately available to the user without restarting Fireworks.  This was added in Fireworks CS6.


## `fw.browseForFileURL()`

In Fireworks CS6, this method takes a fourth parameter that specifies the root directory for the dialog box that appears.  


## `fw.setDocumentImageSize()`

In Fireworks CS6, this method takes a fourth parameter that specifies the interpolation method to use when the document is resized.  The parameter can be a number from 1 to 4, which corresponds to these interpolation methods:

1. Bicubic
2. Bilinear
3. Soft
4. Nearest neighbor 


## `Tools`

In Fireworks CS6, the global `Tools` object is a hash that maps a language-independent name for tools, like `"marquee"` or `"paintBucket"`, to the localized string that’s used for the tool in the UI, like `"Marquee"` or `"Paint Bucket"`.  The only reason this is necessary is that the `fw.activeTool` property returns the localized name of the currently selected tool.  So if you want your extension to do something only when `fw.activeTool == "Paint Bucket"`, that check will work in the English version of Fireworks but not the Japanese version.  

To make the extension work with different languages, you’d want to check if `fw.activeTool == Tools.paintBucket`.  It would have probably been more useful to have the hash go the other way, so that you could check if `Tools[fw.activeTool] == "paintBucket"`, but it's easy enough to build your own object with those mappings.  

Interestingly, `Tools` is an instance of `Object`, rather than `FwDict`.


## `$`

This global object contains a number of Fireworks-specific properties, some of which are useful and some which are not:

* `build`: returns the detailed build version of Fireworks, e.g. `"12.0.0.236"`.
* `buildDate`: this seems to return the same string as `$.build`.
* `fileName`: returns `"not implemented"`.
* `locale`: returns `"English"` for a US build of Fireworks. 
* `os`: returns `"Windows XP"` for Windows 7, so this is not reliable. 
* `version`: returns `"12.0"` for Fireworks CS6, which is the correct version number, unlike `fw.appName`.
* `sleep()`: pass in the number of milliseconds the script should sleep before continuing.  Unfortunately, this method blocks the UI, so there’s no way to use it to cause code to run after a delay.  


## `fw.shiftKeyDown()`, `fw.ctrlCmdKeyDown()`, `fw.altOptKeyDown()`

These undocumented methods return the current state of the shift, ctrl/command and alt/option keys, respectively.  They could be used to make a command behave differently depending on whether a key was held down while it was selected from the *Commands* menu.  But you wouldn’t be able to tell whether the command was run from the menu or via a keyboard shortcut that included the given key.


## `fw.undo()` and `fw.redo()`

These `undo()` and `redo()` methods seem to behave the same as the equivalent methods on the `dom` object.


## `fw.uiOK`

This can be set to `true` or `false`, but it's not clear what effect it has on anything.
