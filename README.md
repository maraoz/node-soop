node-soop
=========

Soop up your JavaScript. This is a simple tool to enable a kind of dependency injection
pattern that is a bit cleaner and easier to understand in the normal cases were you 
want the dependencies wired up in a default manner.  When you wish to override the 
dependencies (i.e. for running unit tests with mocks and stubs), the pattern is also
easy to comprehend.  It also provides a few helper methods for creating inheritance 
chains and explicitly invoking inherited behavior (aka a "super send"). Finally, it 
also gives you the easy ability to load multiple versions of a module (with different
bindings or even different inheritance chains) concurrently.

A few details of the implementation are hacky (i.e. the use of a global for passing
imports to loaded modules), but it works and it was highly desirable to avoid modifying
the nodejs module system or requiring a special build of nodejs. Perhaps if this 
scheme finds widespread use, a new special variable called "imports" could be directly
supported by the nodejs module system (along with new forms of calling require() to 
allow imports to be specified).

With soop, Behaviors (classes if you will) are created in the usual way, but with 
some additional care when using externally provided objects that you wish to have
the ability to override.  Also, when exporting the constructor, we call a function
to add an inherit() and super() method it.  Below is a simple example that would
allow the use of the global "Date" to be overridden:

    var imports = require('soop').imports();
    var Date = imports.Date || global.Date;

    function Person() {
      this._name = null;
      this.lastUpdate = Date.now();
    };

    Person.prototype.name = function(aString) {
      if(!aString) return this._name;
      this._name = aString;
      this.lastUpdate = Date.now();
    };

    module.exports = require('soop')(Person);

The use of the "||" operator in the import binding statement for Date ensures
that if there is a binding for Date, the code to the right of the "||" will
not execute (as a result of the short circuit evaluation attribute of 
JavaScript's "||" operator).  This construct allows you to require() other 
modules of code while not actually loading that code in situations where a 
binding for an import has been provided.

This module allows a Person to be loaded and used in idiomatic nodejs style:

    var Person = require('./Person');
    var me = new Person();

You can access a cached, default instance of a Person as follows:

    var me = require('./Person').default();

If you wanted to use this module with a substitute for the global Date object, you
would do that as follows:

    var MockDate = {now: function(){return 'a split second ago';}};
    var Person = require('soop').load('./Person', {Date: MockDate});
    var me = new Person();
    me.name('George');
    console.log(me.lastUpdate);

Unlike require(), the load() function does not maintain a cache.  If that is
desirable (i.e. for resolving a cyclic reference), you would need to create that 
cache explicitly.

In the following example, we create a Coder that by default inherits from our
Person. Soop will automatically setup inheritance to the object specified in
the parent slot of the constructor. You can also explicity setup inheritance
using the inherit() method (i.e. "Coder.inherit(Person);").  Note the use of
imports for setting the parent which allows for the inherited object to be 
specified using the load() function.

Finally, we demonstrate the use of super(), which allows the method of a parent
to be explicitly invoked (even if the child has overridden that method).  There
are two slightly different forms shown, one that invokes the parent constructor,
and one that invokes a normal method on the parent.

    var imports = require('soop').imports();

    function Coder() {
      Coder.super(this, arguments);
      this.favoriteLanguage = 'JavaScript';
    };
    Coder.parent = imports.parent || require('./Person');

    Coder.prototype.name = function(aString) {
      if(!aString) return Coder.super(this, 'name', arguments);
      return Coder.super(this, 'name', [aString+'(coder)']);
    };

    module.exports = require('soop')(Coder);
