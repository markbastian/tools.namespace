clojure.tools.namespace
========================================

Tools for managing namespaces in Clojure. Parse `ns` declarations from
source files, extract their dependencies, build a graph of namespace
dependencies within a project, update that graph as files change, and
reload files in the correct order.

This is only about namespace dependencies **within** a single project.
It has nothing to do with Leiningen, Maven, JAR files, or
repositories.



Releases and Dependency Information
----------------------------------------

* [Latest stable release is 0.2.11](https://github.com/clojure/tools.namespace/tree/tools.namespace-0.2.11)

* [All Released Versions](http://search.maven.org/#search%7Cgav%7C1%7Cg%3A%22org.clojure%22%20AND%20a%3A%22tools.namespace%22)

[Leiningen](http://leiningen.org/) dependency information:

    [org.clojure/tools.namespace "0.2.11"]

[Maven](http://maven.apache.org/) dependency information:

    <dependency>
      <groupId>org.clojure</groupId>
      <artifactId>tools.namespace</artifactId>
      <version>0.2.11</version>
    </dependency>


### Development Snapshots ###

* Git master branch is at **0.2.12-SNAPSHOT**

* [All Snapshot Versions](https://oss.sonatype.org/content/groups/public/org/clojure/tools.namespace/)

Leiningen dependency information for development snapshots:

    :dependencies [[org.clojure/tools.namespace "0.2.12-SNAPSHOT"]]
    :repositories {"sonatype-oss-public"
                   "https://oss.sonatype.org/content/groups/public/"}

See also [Maven Settings and Repositories](http://dev.clojure.org/display/doc/Maven+Settings+and+Repositories) on dev.clojure.org.



Overview
----------------------------------------

[API Documentation](http://clojure.github.com/tools.namespace/)

tools.namespace consists of several parts:

**clojure.tools.namespace.parse:** A parser for namespace declarations
in Clojure source files. Given a stream of characters from a Clojure
source file, it can find the `ns` declaration and parse the `:require`
and `:use` clauses to find the names of other namespaces that file
depends on. This is all syntactic analysis: it does not 
evaluate any code.

**clojure.tools.namespace.find:** Utilities to search for Clojure
namespaces on the filesystem, in directories or JAR files. Combined
with [java.classpath](http://clojure.github.com/java.classpath/), it
can search for namespaces on the Java classpath. This namespace
contains most of the functions in clojure.tools.namespace version
0.1.x.

**clojure.tools.namespace.repl:** Utilities to load and reload code
based on the namespace dependency graph. This takes some explaining,
see below. c.t.n.repl is built out of smaller parts:

* c.t.n.dependency - generic dependency graph data structure
* c.t.n.track - namespace dependency tracker
* c.t.n.file - file-reader extension to tracker
* c.t.n.dir - directory-scanner extension to tracker
* c.t.n.reload - namespace-reloading extension to tracker

You can recombine these parts in other ways, but c.t.n.repl is the
primary public entry-point to their functionality.

**clojure.tools.namespace.move:** Utilities to aid in moving and
renaming Clojure namespaces. This code is still ALPHA, and it modifies
your source files, so be careful.



Reloading Code: Motivation
----------------------------

c.t.n.repl is a smarter way to reload code. 

The traditional way to reload Clojure code without restarting the JVM
is `(require ... :reload)` or `:reload-all` or an editor/IDE feature
that does the same thing. This has several problems:

* If you modify two namespaces which depend on each other, you must
  remember to reload them in the correct order to avoid compilation
  errors.

* If you remove definitions from a source file and then reload it,
  those definitions are still available in memory. If other code
  depends on those definitions, it will continue to work but will
  break the next time you restart the JVM.

* If the reloaded namespace contains `defmulti`, you must also reload
  all of the associated `defmethod` expressions.

* If the reloaded namespace contains `defprotocol`, you must also
  reload any records or types implementing that protocol and replace
  any existing instances of those records/types with new instances.

* If the reloaded namespace contains macros, you must also reload any
  namespaces which use those macros.

* If the running program contains functions which close over values in
  the reloaded namespace, those closed-over values are not updated.
  (This is common in web applications which construct the "handler
  stack" as a composition of functions.)

Often the only surefire way to reload Clojure code is to restart the
JVM. A large Clojure application can take 20 seconds or more just to
compile. I wrote tools.namespace to help speed up this development
cycle.

For more detail on how I use tools.namespace in my development
workflow, see the article [My Clojure Workflow, Reloaded](http://thinkrelevance.com/blog/2013/06/04/clojure-workflow-reloaded).



Reloading Code: Usage
-----------------------

There's only one important function, `refresh`:

    user=> (require '[clojure.tools.namespace.repl :refer [refresh]])
    nil

    user=> (refresh)
    :reloading (com.example.util com.example.app com.example.app-test)
    :ok

The `refresh` function will scan all the directories on the classpath
for Clojure source files, read their `ns` declarations, build a graph
of their dependencies, and load them in dependency order. (You can
change the directories it scans with `set-refresh-dirs`.)

Later on, after you have changed and saved a few files in your editor,
run it again:

    user=> (refresh)
    :reloading (com.example.app com.example.app-test)
    :ok

Based on file modification timestamps and the graph of dependencies,
the `refresh` function will reload *only* the namespaces that have
changed, in dependency order. But first, it will *unload* (remove) the
namespaces that changed to clear out any old definitions.

This is quite unlike `(require ... :reload)`. Calling `refresh` will
*blow away your old code*. Sometimes this is helpful: it can catch
trivial mistakes like deleting a function that another piece of code
depends on. But sometimes it hurts when you have built-up application
state stored in a Var that got deleted by `refresh`.

This brings us to the next section:



Reloading Code: Preparing Your Application
--------------------------------------------

Being able to safely destroy and reload namespaces without breaking
your application requires some discipline and careful design. It won't
"just work" on any Clojure project.


### No Global State

The first rule for making your application reload-safe is **no global
state**. That means you should avoid things like this:

    (def state-of-world (ref {}))
    (def object-handle (atom nil))

c.t.n.repl/refresh will destroy those Vars when it reloads the
namespace (even if you used `defonce`).

Instead of storing your state in global Vars, store it *locally* in an
object that represents the running state of your application. Then
provide a constructor function to initialize that state:

    (defn create-application []
      {:state-of-world (ref {})
       :object-handle (atom nil)})

You can choose what representation works best for your application:
map, vector, record, or even just a single Ref by itself.

Typically you'll still need one global `def` somewhere, perhaps in the
REPL itself, to hold the current application instance. See the next
section.


### Managed Lifecycle

The second rule for making your application reload-safe is to have a
consistent way to **start and stop the entire system**.

The "start" function should:

- Acquire stateful resources such as sockets, files, and database
  connections

- Start threads or background processes

- Initialize application state such as caches or counters

- Return an object encapsulating the state of the application

The "stop" function should take the state returned by "start" as an
argument and do the opposite:

- Close or release stateful resourecs

- Stop all background processes

- Clear out application state

It might take a few tries to get it right, but once you have working
start and stop functions you can have a workflow like this:

Step 1. Start up a REPL.

Step 2. Load the app:

    user=> (require '[clojure.tools.namespace.repl :refer [refresh]])
    user=> (refresh)
    user=> (def my-app (start-my-app))

Step 3. Test it out.

Step 4. Modify some source files.

Step 5. Restart:

    user=> (stop-my-app my-app)
    user=> (refresh)
    user=> (def my-app (start-my-app))

(You could also combine all those steps in a single utility function,
but see warnings below.)

After that, you've got a squeaky-clean new instance of your app
running, in a fraction of the time it takes to restart the JVM.


### Handling Errors

If an exception is thrown while loading a namespace, `refresh` stops,
prints the namespace that caused the exception, and returns the
exception. You can print the rest of the stacktrace with
`clojure.repl/pst`; the exception itself is bound to `*e`.

    user=> (refresh)
    :reloading (com.example.app com.example.app-test)
    :error-while-loading com.example.app
    #<IllegalArgumentException java.lang.IllegalArgumentException:
      Parameter declaration cond should be a vector>

    user=> (clojure.repl/pst)
    IllegalArgumentException Parameter declaration cond should be a vector
            clojure.core/assert-valid-fdecl (core.clj:6567)
            clojure.core/sigs (core.clj:220)
            clojure.core/defn (core.clj:294)
            clojure.lang.Var.invoke (Var.java:427)
            ...

Remember that any namespaces which depend on the namespace that caused
the exception **do not exist** at this point: they have been removed
but not yet reloaded.

After you fix the problem, call `refresh` again and it will resume
reloading where it left off. 

**NOTE:** If your current REPL namespace is one of those that has not
yet been reloaded, then none of the functions you defined in that
namespace will exist! Starting with version 0.2.8, tools.namespace
will *attempt* to restore aliases to the namespaces which were
successfully loaded.

So, for example, if your current REPL namespace is named `dev` and
contains this ns declaration:

    (ns dev
      (:require [com.example.foo :as foo]
                [com.example.bar :as bar]
                [clojure.tools.namespace.repl :as tns]))

And you get an error on refresh like this:

    dev=> (tns/refresh)
    :reloading (com.example.bar dev)
    :error-while-loading com.example.bar
    #<CompilerException ... compiling:(com/example/bar.clj:1:21)>

Then the functions in `com.example.foo` should still be available in
the `dev` namespace via the alias `foo`.



Warnings and Potential Problems
-------------------------------

**`ns` syntax:** Clojure's `ns` macro is notoriously lax in what
syntax it accepts. tools.namespace.parse is somewhat liberal, but it
cannot handle every possible variation of syntax that `ns` does. Stick
to the docstrings of [ns] and [require] and everything should be fine.

**AOT-compilation:** Reloading code does not work in the presence of
[AOT-compiled] namespaces. If you are using AOT-compilation in your
project, make sure it is disabled and you have deleted any
AOT-compiled `.class` files before starting a REPL development
session. (In Leiningen, run `lein clean`.)

Note that the presence of `:main` in project.clj triggers
AOT-compilation in some versions of Leiningen.

**Conflicts:** Other libraries which also do code-reloading may
conflict with tools.namespace. One known example is ring-devel (as of
[Ring] version 1.1.6) which uses [ns-tracker], which uses an older
version of tools.namespace.

**REPL namespace:** Be careful when reloading the namespace in which
you run your REPL. Because namespaces are removed when reloading, all
your past definitions are lost. Either keep your REPL in a namespace
which has no file associated with it, such as `user`, or put all your
REPL definitions in a file so that they can be reloaded.

**Fully-qualified names:** Be careful when using fully-qualified
symbol names without namespace aliases (`require` with no `:as`). If
the namespace happens to be loaded already, it will not necessarily
cause an error if you forget to `require` it, but the dependency graph
of namespaces will be incorrect.

**Old definitions:** Beware of code which has references to old
definitions, especially references to things you created in the REPL.

**Rolling your own:** If you create your own instance of the
dependency tracker, do not store it in a namespace which gets
reloaded.

[AOT-compiled]: http://clojure.org/compilation
[Ring]: https://github.com/ring-clojure/ring
[ns-tracker]: https://github.com/weavejester/ns-tracker
[ns]: http://clojure.github.io/clojure/clojure.core-api.html#clojure.core/ns
[require]: http://clojure.github.io/clojure/clojure.core-api.html#clojure.core/require


### Warnings for Helper Functions

Be careful defining a helper function in a namespace which calls
`refresh` if that namespace also could get reloaded. For example, you
might try to combine the stop-refresh-start code from the "Managed
Lifecycle" section into a single function:

    (def my-app nil)

    (defn restart []
      (stop-my-app my-app)
      (refresh)
      (alter-var-root #'my-app (constantly (start-my-app))))

This won't work if the namespace containing `restart` could get
reloaded. After `refresh`, the namespace containing `restart` has been
dropped, but the function continues to run in the *old* namespace and
refer to old Vars.

If you want to run some code after `refresh`, you can pass an option
naming a function you want to run *after* a successful reload. The
value of this option must be a symbol, and it must be fully
namespace-qualified. The previous example could be correctly written
(assuming these functions are defined in the `dev` namespace):

    (def my-app nil)

    (defn start []
      (alter-var-root #'my-app (constantly (start-my-app))))

    (defn restart []
      (stop-my-app my-app)
      (refresh :after 'dev/start))


### Warnings for Aliases

Namespace aliases created at the REPL will still refer to the *old* namespace  after `refresh`. For example:

    user=> (require '[com.example.foo :as foo])
    
    user=> foo/bar

    user=> (refresh)
    :reloading (com.example.foo)
    :ok

    user=> foo/bar   ; this is the *old* foo/bar

If you try to recreate the alias with the new namespace, you will get an error:

    user=> (require '[com.example.foo :as foo])
    IllegalStateException Alias foo already exists in
    namespace user, aliasing com.example.foo
    clojure.lang.Namespace.addAlias (Namespace.java:224)

The only way out is to remove the alias before recreating it:

    user=> (ns-unalias *ns* 'foo)
    nil
    user=> (alias 'foo 'com.example.foo)


### Warnings for Protocols

When reloading namespaces which contain protocols, be careful that you
do not leave any old instances of records or types implementing those
protocols.

For example, if you have a namespace like this:

    (ns com.example.foo)
    
    (defprotocol IFoo
      (foo [this]))
    
    (defrecord FooRecord []
      IFoo (foo [this] nil))

And you do something like the following at the REPL:

    user=> (def my-foo (->FooRecord))
    user=> (clojure.tools.namespace.repl/refresh)
    user=> (foo my-foo)

You will get a confusing error message like this:

    IllegalArgumentException
    No implementation of method: :foo
    of protocol: #'com.example.foo/IFoo
    found for class: com.example.foo.FooRecord
    clojure.core/-cache-protocol-fn (core_deftype.clj:527)

That's because `my-foo` is an **instance** of the **old** version of
`FooRecord`, implementing the **old** version of `IFoo`. As far as the
JVM is concerned, the old `IFoo` and the new `IFoo` are completely
different classes.

To avoid this problem, always create new instances of records after a
refresh.


### Warnings for Multimethods

Calling `prefer-method` is a global side-effect. If you modify a call
to `prefer-method` and reload the namespace containing it, Clojure may
throw "java.lang.IllegalStateException: Preference conflict in multimethod."
The workaround is to call `remove-method` before reloading.
tools.namespace cannot detect this situation automatically. See [TNS-23].


### Heap Usage and PermGen (JDK 1.7 and before)

In rare cases, reloading a lot of code may lead to out-of-memory
errors from the JVM like `java.lang.OutOfMemoryError: PermGen space`.

You may be able to mitigate this by increasing the size of the
"Permanent Generation" where the JVM stores compiled classes. To do
this, add the following command-line argument to your JVM startup:

    -XX:MaxPermSize=<N>

where `<N>` is a number with a suffix like `m` for megabytes.

To find the default MaxPermSize for your JDK, run
`java -XX:+PrintFlagsFinal` and search the results for "MaxPermSize".
Try doubling it.

The Permanent Generation was removed in JDK 1.8 ([JEP 122]) so this
section no longer applies.

In some older JDKs (1.5) the default garbage collector did not collect
the Permanent Generation at all unless it was explicitly enabled with
`-XX:+CMSPermGenSweepingEnabled`.



Disabling Refresh In a Namespace
--------------------------------

Some projects have a "project REPL" or a "scratch" namespace where you
want keep state during development. You can use the functions
`disable-unload!` and `disable-reload!` in
`clojure.tools.namespace.repl` to prevent `refresh` from automatically
un/reloading those namespaces.

Use this feature sparingly: it exists as a development-time
convenience, not a work-around for code that is not reload-safe. Also,
see the warnings about aliases, below. Aliases to reloaded namespaces
will break if the namespace *containing* the alias is not reloaded
also.

After an error, `refresh` will **not** attempt to recover symbol
mappings and aliases for namespaces with `disable-unload!` or
`disable-reload!` set.



Developer Information
----------------------------------------

* [GitHub project](https://github.com/clojure/tools.namespace)

* [How to contribute](http://dev.clojure.org/display/community/Contributing)

* [Bug Tracker](http://dev.clojure.org/jira/browse/TNS)

* [Continuous Integration](http://build.clojure.org/job/tools.namespace/)

* [Compatibility Test Matrix](http://build.clojure.org/job/tools.namespace-test-matrix/)



Change Log
----------------------------------------

### Version 0.2.12-SNAPSHOT

  * In development, current Git master branch

### Version 0.2.11 on 19-Jun-2015

  * [TNS-34] Allow reader conditionals in parsed source files

### Version 0.2.10 on 26-Feb-2015

  * Widen existing functions to handle both clj and cljc files in
    advance of reader conditional support in Clojure 1.7.

### Version 0.2.9 on 31-Jan-2015

  * Fix [TNS-20]: Undefined 'unload' order after namespaces are first
    added to an new, empty tracker.

  * Improvement [TNS-21]: Support `ns` clauses which use vectors
    instead of lists for clauses, contrary to docs.

  * Improvement [TNS-32]: Support `ns` clauses which use symbols as
    clause heads instead of keywords, contrary to docs.

### Version 0.2.8 on 19-Dec-2014

  * Improvement [TNS-31]: Specific error message when `:after` symbol
    passed to `refresh` cannot be resolved.

  * Fix [TNS-26]: namespace alias recovery after failed reload did not
    work due to local binding shadowing global Var

### Version 0.2.7 on 19-Sept-2014

  * [Revert bad commit](https://github.com/clojure/tools.namespace/commit/27194f2edfe3f5f9e1343f993beca4b43f0bafe8)
    mistakenly included in 0.2.6 which could cause the tracker's
    'unload' order to be incorrect. See discussion at [TNS-20].

### **BROKEN** Version 0.2.6 on 7-Sept-2014 **DO NOT USE**

  * `clojure.tools.namespace.parse/read-ns-decl` asserts that its
    argument is a PushbackReader, instead of silently returning nil

  * Fix [TNS-22]: broken `clojure.string/replace` with Windows path
    separator

### Version 0.2.5 on 15-Jul-2014

  * New `clojure.tools.namespace.repl/clear` empties the state of the
    REPL dependency tracker. This can help repair the dependency
    tracker after a failed load or a circular dependency error.

  * Enhancement [TNS-19]: `deps-from-ns-decl` should return an empty
    set instead of nil. This may be a breaking change for some but
    is consistent with the original docstring.

  * Enhancement [TNS-18]: Compute transitive dependencies in linear time.

  * Enhancement [TNS-17]: The `ns` form doesn't need to be the first
    top-level form in a file.

  * Fix [TNS-16]: Don't depend on specific hash ordering in tests.
    Exposed by new hash functions in Clojure 1.6.0.

  * Fix [TNS-15]: Handle spaces in classpath directories (old
    `clojure.tools.namespace`)

  * Fix [TNS-12]: Duplicate definition of `jar-file?`

### Version 0.2.4 on 19-Jul-2013

  * Fix [TNS-10]: Forbid circular dependency when a namespace depends
    on itself

  * Fix [TNS-9] and [TNS-11]: support other prefix-list forms

  * Fix [TNS-8]: In `move-ns`, do not modify files whose contents does
    not change

### Version 0.2.3 on 01-Apr-2013

  * New: Attempt recovery of aliases/refers in REPL after error

### Version 0.2.2 on 14-Dec-2012

  * New: Add `:after` option to `refresh`

  * New: Add `clojure.tools.namespace.move`

  * Fix [TNS-4], reflection warnings

### Version 0.2.1 on 26-Oct-2012

  * Fix: Restore deprecated 0.1.x APIs in `clojure.tools.namespace`

  * Fix [TNS-3], actually use `refresh-dirs`

### Version 0.2.0 on 05-Oct-2012

  * **Not recommended for use**: this release introduced breaking API
    changes (renaming core namespaces and functions) without
    backwards-compatibility. Applications with dependencies on both
    the 0.2.x and 0.1.x APIs cannot use this version.

  * New dependency tracking & reloading features

  * Eliminate dependency on [java.classpath]

### Version 0.1.3 on 24-Apr-2012

  * Fix [TNS-1] Workaround for Clojure 1.2 reader bug

### Version 0.1.2 on 10-Feb-2012

  * Fix: Eliminate reflection warnings

### Version 0.1.1 on 18-May-2011

### Version 0.1.0 on 24-Apr-2011

  * Source-compatible with clojure.contrib.find-namespaces in old
    clojure-contrib 1.2.0

[TNS-1]: http://dev.clojure.org/jira/browse/TNS-1
[TNS-3]: http://dev.clojure.org/jira/browse/TNS-3
[TNS-4]: http://dev.clojure.org/jira/browse/TNS-4
[TNS-8]: http://dev.clojure.org/jira/browse/TNS-8
[TNS-9]: http://dev.clojure.org/jira/browse/TNS-9
[TNS-10]: http://dev.clojure.org/jira/browse/TNS-10
[TNS-11]: http://dev.clojure.org/jira/browse/TNS-11
[TNS-12]: http://dev.clojure.org/jira/browse/TNS-12
[TNS-15]: http://dev.clojure.org/jira/browse/TNS-15
[TNS-16]: http://dev.clojure.org/jira/browse/TNS-16
[TNS-17]: http://dev.clojure.org/jira/browse/TNS-17
[TNS-18]: http://dev.clojure.org/jira/browse/TNS-18
[TNS-19]: http://dev.clojure.org/jira/browse/TNS-19
[TNS-20]: http://dev.clojure.org/jira/browse/TNS-20
[TNS-21]: http://dev.clojure.org/jira/browse/TNS-21
[TNS-22]: http://dev.clojure.org/jira/browse/TNS-22
[TNS-23]: http://dev.clojure.org/jira/browse/TNS-23
[TNS-24]: http://dev.clojure.org/jira/browse/TNS-24
[TNS-25]: http://dev.clojure.org/jira/browse/TNS-25
[TNS-26]: http://dev.clojure.org/jira/browse/TNS-26
[TNS-27]: http://dev.clojure.org/jira/browse/TNS-27
[TNS-28]: http://dev.clojure.org/jira/browse/TNS-28
[TNS-29]: http://dev.clojure.org/jira/browse/TNS-29
[TNS-30]: http://dev.clojure.org/jira/browse/TNS-30
[TNS-31]: http://dev.clojure.org/jira/browse/TNS-31
[TNS-32]: http://dev.clojure.org/jira/browse/TNS-32
[TNS-33]: http://dev.clojure.org/jira/browse/TNS-33
[TNS-34]: http://dev.clojure.org/jira/browse/TNS-34
[java.classpath]: https://github.com/clojure/java.classpath
[JEP 122]: http://openjdk.java.net/jeps/122



Copyright and License
----------------------------------------

Copyright © 2012 Stuart Sierra All rights reserved. The use and
distribution terms for this software are covered by the
[Eclipse Public License 1.0] which can be found in the file
epl-v10.html at the root of this distribution. By using this software
in any fashion, you are agreeing to be bound by the terms of this
license. You must not remove this notice, or any other, from this
software.

[Eclipse Public License 1.0]: http://opensource.org/licenses/eclipse-1.0.php
