Asynchronous IndexedDB plugin for Cordova
================================

[![Dependencies](https://img.shields.io/david/ABB-Austin/cordova-plugin-indexeddb-async.svg)](https://david-dm.org/ABB-Austin/cordova-plugin-indexeddb-async)
[![npm](http://img.shields.io/npm/v/cordova-plugin-indexeddb-async.svg)](https://www.npmjs.com/package/cordova-plugin-indexeddb-async)
[![License](https://img.shields.io/npm/l/cordova-plugin-indexeddb-async.svg)](LICENSE)

* Uses [IndexedDBShim](https://github.com/axemclion/IndexedDBShim) to polyfill devices that don't support IndexedDB
* Uses the [__asynchronous__ WebSql plugin](https://github.com/Thinkwise/cordova-plugin-websql) on Windows devices
* Can _optionally_ override native IndexedDB on devices with [buggy implementations](http://www.raymondcamden.com/2014/9/25/IndexedDB-on-iOS-8--Broken-Bad)
* This plugin is basically an IndexedDB-to-WebSql adapter


Installation
--------------------------
Install via the [Cordova CLI](https://cordova.apache.org/docs/en/edge/guide_cli_index.md.html).

For __Cordova CLI 4.x__, use the GIT URL syntax:

````bash
cordova plugin add https://github.com/ABB-Austin/cordova-plugin-indexeddb-async.git
````

For __Cordova CLI 5.x__, use the [new npm syntax](https://github.com/cordova/apache-blog-posts/blob/master/2015-04-15-plugins-release-and-move-to-npm.md):

````bash
cordova plugin add cordova-plugin-indexeddb-async
````


Using the plugin
--------------------------
Cordova will automatically load the plugin and run it.  So all you need to do is use IndexedDB just like normal.

````javascript
// Open (or create) the database
var open = indexedDB.open("MyDatabase", 1);

// Create the schema
open.onupgradeneeded = function() {
    var db = open.result;
    var store = db.createObjectStore("MyObjectStore", {keyPath: "id"});
    var index = store.createIndex("NameIndex", ["name.last", "name.first"]);
};

open.onsuccess = function() {
    // Start a new transaction
    var db = open.result;
    var tx = db.transaction("MyObjectStore", "readwrite");
    var store = tx.objectStore("MyObjectStore");
    var index = store.index("NameIndex");

    // Add some data
    store.put({id: 12345, name: {first: "John", last: "Doe"}, age: 42});
    store.put({id: 67890, name: {first: "Bob", last: "Smith"}, age: 35});
    
    // Query the data
    var getJohn = store.get(12345);
    var getBob = index.get(["Smith", "Bob"]);

    getJohn.onsuccess = function() {
        console.log(getJohn.result.name.first);  // => "John"
    };

    getBob.onsuccess = function() {
        console.log(getBob.result.name.first);   // => "Bob"
    };

    // Close the db when the transaction is done
    tx.oncomplete = function() {
        db.close();
    };
}
````


Overriding Native IndexedDB
--------------------------
Even if a device natively supports IndexedDB, you may still want to use this shim instead.  Some native IndexedDB implementations are [very buggy](http://www.raymondcamden.com/2014/9/25/IndexedDB-on-iOS-8--Broken-Bad).  Others are [missing certain features](http://codepen.io/cemerick/pen/Itymi).  

There are also many minor inconsistencies between different implementations of IndexedDB, such as how errors are handled, how transaction timing works, how records are sorted, how cursors behave, etc.  Using this shim will ensure consistent behavior across all browsers.

To force the plugin to replace the browser's native IndexedDB, add this line to your app:

````javascript
window.shimIndexedDB.__useShim()
````


Known Issues
--------------------------
#### iOS
Due to a [bug in WebKit](https://bugs.webkit.org/show_bug.cgi?id=137034), the `window.indexedDB` property is read-only and cannot be overridden by IndexedDBShim.  There are two possible workarounds for this:

1. Use `window.shimIndexedDB` instead of `window.indexedDB` 
2. Create an `indexedDB` variable in your closure<br>
By creating a variable named `indexedDB`, all the code within that closure will use the variable instead of the `window.indexedDB` property.  For example:

````javascript
(function() {
    // This works on all browsers, and only uses IndexedDBShim as a final fallback 
    var indexedDB = window.indexedDB || window.mozIndexedDB || window.webkitIndexedDB || window.msIndexedDB || window.shimIndexedDB;

    // This code will use the native IndexedDB, if it exists, or the shim otherwise
    indexedDB.open("MyDatabase", 1);
})();
````

