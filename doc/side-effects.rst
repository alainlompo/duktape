============
Side effects
============

Overview
========

Duktape is a single threaded interpreter, so when the internal C code deals
with memory allocations, pointers, and internal data structures it is safe
to assume, for example, that pointers are stable while they're being used and
that internal state and data structures are not modified simultaneously from
other threads.

However, many internal operations trigger quite extensive side effects, such
as resizing the value stack (invalidating any pointers to it) or clobbering
the current heap error handling (longjmp) state.  There are a few primary
causes for the side effects, such as memory management reallocating data
structures, finalizer invocation, and Proxy trap invocation.  The primary
causes are also triggered by a lot of secondary causes.  The practical effect
is that any internal helper should be assumed to potentially invoke arbitrary
side effects unless there's a specific reason to assume otherwise.

Some of the side effects can be surprising when simply looking at the calling
code, which makes side effects an error prone factor when maintaining Duktape
internals.  Incorrect call site assumptions can cause immediate issues like
segfaults, assert failures, or valgrind warnings.  But it's also common for
an incorrect assumption to work out fine in practice, only to be triggered by
rare conditions like voluntary mark-and-sweep happening in just the right
place.  Such bugs have crept into the code base several times -- they're easy
to make and hard to catch with tests or code review.

This document describes the different side effects, how they may be triggered,
what mechanisms are in place to deal with them internally, and how the error
prone nature of side effects can be covered by tests.

Basic side effect categories
============================

Side effects are ultimately caused by:

* A refcount dropping to zero, causing a "refzero cascade" where a set of
  objects is refcount finalized and freed.  If any objects in the cascade
  have finalizers, the finalizer calls have a lot of side effects.  Object
  freeing is nearly side effect free, but does invalidate any pointers to
  unreachable but not-yet-freed objects which are held at times.

* Mark-and-sweep similarly frees objects, and can make finalizer calls.
  Mark-and-sweep may also resize/compact the string table and object property
  tables.

Any operation doing a DECREF may thus have side effects; any operation doing
anything to cause a mark-and-sweep (like allocating memory) may similarly have
side effects.  Finalizers cause the most wide ranging side effects, but even
with finalizers disabled there are significant side effects in mark-and-sweep.

Full side effects
-----------------

The most extensive type of side effect is arbitrary code execution, caused
by e.g. a finalizer or a Proxy trap call (and a number of indirect causes).
The potential side effects are very wide:

* Because a call is made, value, call, and catch stacks may be grown (but
  not shrunk) and their base pointers may change.  As a result, any duk_tval
  pointers to the value stack, duk_activation pointers to the call stack, and
  duk_catcher pointers to the catch stack are (potentially) invalidated.

* An error throw may happen, clobbering heap longjmp state.  This is a
  problem particularly in error handling where we're dealing with a previous
  throw.

* A new thread may be resumed and yielded from.  The resumed thread may even
  duk_suspend().

* A native thread switch may occur, for an arbitrarily long time, if any
  function called uses duk_suspend() and duk_resume().  This is not currently
  supported for finalizers, but may happen for Proxy trap calls legitimately.

* Because called code may operate on any object (except those we're certain
  not to be reachable), objects may undergo arbitrary mutation.  For example,
  object properties may be added, deleted, or modified; dynamic and external
  buffer data pointers may change; external buffer length may change.  An
  object's property table may be resized and its base pointer may change,
  invalidating both pointers to the property table.  Object property slot
  indices may also be invalidated due to object resize/compaction.

The following will be stable at all times:

* All heap object (duk_heaphdr) pointers are valid and stable regardless of
  any side effects, provided that the objects in question are reachable and
  correctly refcounted for.

* In particular, while duk_tval pointers to the value stack may change, if
  an object is encapsulated in a duk_tval, the pointer to the actual object
  is still stable.

* All string data pointers, including external strings.  String data is
  immutable, and can't be reallocated or relocated.

* All fixed buffer data pointers, because fixed buffer data follows the stable
  duk_heaphdr directly.  Dynamic and external buffer data pointers are not
  stable.

Side effects without finalizers, but with mark-and-sweep allowed
----------------------------------------------------------------

If code execution side effects (finalizer calls, Proxy traps, etc) are
avoided, most of the side effects are avoided.  In particular, refzero
situations are then side effect free because object freeing has no side
effects beyond memory free calls.

The following side effects still remain:

* Refzero processing still frees objects whose refcount reaches zero.
  Any pointers to such objects will thus be invalidated.  This may happen
  e.g. when a borrowed pointer is used and that pointer loses its backing
  reference.

* Mark-and-sweep may reallocate/compact the string table.  This affects
  the string table data structure pointers and indices/offsets into them.
  Strings themselves are not affected (but unreachable strings may be freed).

* Mark-and-sweep may reallocate/compact object property tables.  All property
  keys and values will remain reachable, but pointers and indices to an object
  property table may be invlidated.  This mostly affects property code which
  often finds a property's "slot index" and then operates on the index.

* Mark-and-sweep may free unreachable objects, invalidating any pointers to
  them.  This affects only objects which have been allocated and added to
  heap_allocated list.  Objects not on heap_allocated list are not affected
  because mark-and-sweep isn't aware of them; such objects are thus safe from
  collection, but at risk for leaking if an error is thrown, so such
  situations are usually very short lived.

Other side effects don't happen with current mark-and-sweep implementation.
For example, the following don't happen (but could, if mark-and-sweep scope
and side effect lockouts are changed):

* Thread value stack, call stack, and catch stack are never reallocated
  and all pointers to duk_tvals, duk_activations, and duk_catchers remain
  valid.  (This could easily change if mark-and-sweep were to "compact"
  the stacks on an emergency GC.)

The mark-and-sweep side effects listed above are not fundamental to the
engine and could be removed if they became inconvenient.  For example, it's
nice that emergency GC can compact objects in an attempt to free memory, but
it's not a critical feature (and many other engines don't do it either).

Side effects with finalizers and mark-and-sweep disabled
--------------------------------------------------------

When both finalizers and mark-and-sweep are disabled, the only remaining side
effects come from DECREF (plain or NORZ):

* Refzero processing still frees objects whose refcount reaches zero.
  Any pointers to such objects will thus be invalidated.  This may happen
  e.g. when a borrowed pointer is used and that pointer loses its backing
  reference.

When DECREF operations happen during mark-and-sweep they get handled specially:
the refcounts are updated normally, but the objects are never freed or even
queued to refzero_list.  This is done because mark-and-sweep will free any
unreachable objects; DECREF still gets called because mark-and-sweep finalizes
refcounts of any freed objects (or rather other objects they point to) so that
refcounts remain in sync.

Controls in place
=================

Finalizer execution, pf_prevent_count
-------------------------------------

Objects with finalizers are queued to finalize_list and are processed later
by duk_heap_process_finalize_list().  The queueing doesn't need any side
effect protection as it is side effect free.

duk_heap_process_finalize_list() is guarded by heap->pf_prevent_count.  If
the count is zero, finalize_list is processed until it becomes empty (new
finalizable objects may be queued while the list is being processed).  If
the count is non-zero, the call is a no-op.

The count prevents recursive finalize_list processing: the first call site
to call duk_heap_process_finalize_list() with a zero prevent count will
process the queue until it becomes empty.  When finalizers run, new objects
may be queued to finalize_list (an arbitrary number of objects may be queued)
but the list is only processed by the first call site.

The count can also be bumped upwards to prevent finalizer execution in the
first place, even if no call site is currently processing finalizers.  If the
count is bumped, there must be a reliable mechanism of unbumping the count or
finalizer execution is prevented permanently.

Mark-and-sweep prevent count, ms_prevent_count
----------------------------------------------

Stacking counter to prevent mark-and-sweep.  Also used to prevent recursive
mark-and-sweep entry.

Mark-and-sweep running, ms_running
----------------------------------

This flag is set only when mark-and-sweep is actually running, and doesn't
stack because recursive mark-and-sweep is not allowed.

The flag is used by DECREF macros to detect that mark-and-sweep is running
and that objects must not be queued to refzero_list or finalize_list; their
refcounts must still be updated.

Mark-and-sweep flags
--------------------

Mark-and-sweep base flags from duk_heap are ORed to mark-and-sweep argument
flags.  This allows a section of code to avoid e.g. object compaction
regardless of how mark-and-sweep gets triggered.

Using the base flags is useful when mark-and-sweep by itself is desirable
but e.g. object compaction is not.  Finalizers are prevented using a
separate flag.

Creating an error object, handling_error
----------------------------------------

This flag is set when Duktape internals are creating an error to be thrown.
If an error happens during that process (which includes a user errCreate()
callback), the flag is set and avoids recursion.  A pre-allocated "double
error" object is thrown instead.

Call stack unwind or handling an error, error_not_allowed
---------------------------------------------------------

This flag is only enabled when using assertions.  It is set in code sections
which must be protected against an error being thrown.  This is a concern
because currently the error state is global in duk_heap and doesn't stack,
so an error throw (even a caught and handled one) clobbers the state which
may be fatal in code sections working to handle an error.

DECREF NORZ (no refzero) macros
-------------------------------

DECREF NORZ (no refzero) macro variants behave the same as plain DECREF macros
except that they don't trigger side effects.  Since Duktape 2.1 NORZ macros
will handle refzero cascades inline (freeing all the memory directly); however,
objects with finalizers will be placed in finalize_list without finalizer
calls being made.

Once a code segment with NORZ macros is complete, DUK_REFZERO_CHECK_{SLOW,FAST}()
should be called.  The macro checks for any pending finalizers and processes
them.  No error catcher is necessary: error throw path also calls the macros and
processes pending finalizers.  (The NORZ name is a bit of a misnomer since
Duktape 2.1 reworks.)

Mitigation, test coverage
=========================

There are several torture test options to exercise side effect handling:

* Triggering a mark-and-sweep for every allocation (and in a few other places
  like DECREF too).

* Causing a simulated finalizer run with error throwing and call side effects
  every time a finalizer might have executed.

Operations causing side effects
===============================

The main reasons and controls for side effects are covered above.  Below is
a (non-exhaustive) list of common operations with side effects.  Any internal
helper may invoke some of these primitives and thus also have side effects.

DUK_ALLOC()

* May trigger voluntary or emergency mark-and-sweep, with arbitrary
  code execution side effects.

DUK_REALLOC()

* May trigger voluntary or emergency mark-and-sweep, with arbitrary
  code execution side effects.

* In particular, if reallocating e.g. the value stack, the triggered
  mark-and-sweep may change the base pointer being reallocated!
  To avoid this, the DUK_REALLOC_INDIRECT() call queries the base pointer
  from the caller for every realloc() attempt.

DUK_FREE()

* No side effects at present.

Property read, write, delete, existence check

* May invoke Proxy traps with arbitrary code execution side effects.

* Property write/delete operations may allocate memory and DECREF with
  arbitrary code execution side effects.

Value stack pushes

* Pushing to the value stack is side effect free as such.  The space must be
  allocated beforehand, and a pushed value is INCREF'd if it isn't primitive.
  INCREF is side effect free.

Value stack pops

* Popping a value may invoke a finalizer, and thus may cause arbitrary code
  execution side effects.

Value stack coercions

* Value stack type coercions may, depending on the coercion, may invoke
  methods like .toString() and .valueOf(), and thus have arbitrary code
  execution side effects.  Even failed attempts may cause side effects due
  to memory allocation attempts.

* In specific cases it may be safe to conclude that a coercion is side effect
  free; for example, doing a ToNumber() conversion on a plain string is side
  effect free at present.  (This may not always be the case in the future,
  e.g. if numbers become heap allocated.)

* Some coercions not involving an explicit method call may require an
  allocation call -- which may then trigger a voluntary or emergency
  mark-and-sweep leading to arbitrary code execution side effects.

INCREF

* No side effects at present.  Object is never freed or queued anywhere.

DECREF_NORZ

* No side effects other than freeing one or more objects, strings, and
  buffers.  The freed objects don't have finalizers.

* Queries finalizer existence which is side effect free.

* When mark-and-sweep is running, DECREF_NORZ adjusts target refcount but
  won't do anything else like queue object to refzero_list or free it; that's
  up to mark-and-sweep.

DECREF

* If refcount doesn't reach zero, no side effects.

* If refcount reaches zero, one or more objects, strings, and buffers are
  freed.  Objects with finalizers are queued to finalize_list, and the list
  is processed when the cascade of objects without finalizers has been freed.

* Queries finalizer existence which is side effect free.

* Finalizer execution phase has arbitrary code execution side effects.

* When mark-and-sweep is running, DECREF adjusts target refcount but won't
  do anything else.

duk__refcount_free_pending()

* As of Duktape 2.1 no side effects, just frees objects without a finalizer
  until refzero_list is empty.

* Recursive entry is prevented; first caller processes a cascade until it's
  done.  Pending finalizers are executed after the refzero_list is empty
  (unless prevented).  Finalizers are executed outside of refzero_list
  processing protection so DECREF freeing may happen normally during finalizer
  execution.

Mark-and-sweep

* Queries finalizer existence which is side effect free.

* Object compaction.

* String table compaction.

* Recursive entry prevented.

* Executes finalizers after mark-and-sweep is complete (unless prevented),
  which has arbitrary code execution side effects.  Finalizer execution
  happens outside of mark-and-sweep protection, but currently finalizer
  execution explicitly prevents mark-and-sweep to avoid incorrect rescue/free
  decisions when the finalize_list is only partially processed.

Error throw

* Overwrites heap longjmp state, so an error throw while handling a previous
  one is a fatal error.

* Because finalizer calls may involve error throws, finalizers cannot be
  executed in error handling (at least without storing/restoring longjmp
  state).

* Error handling must also never throw (without catching) for sandboxing
  reasons: the error handling path of a protected call is assumed to never
  throw.

* Mark-and-sweep without error throwing or call side effects is OK.

Debugger message writes

* Code writing a debugger message to the current debug client transport
  must ensure, somehow, that it will never happen when another function
  is doing the same (including nested call to itself).

* If nesting happens, memory unsafe behavior won't happen, but the debug
  connection becomes corrupted.

* There are some current issues for debugger message handling, e.g. debugger
  code uses duk_safe_to_string() which may have side effects or even busy
  loop.

Error handling and side effects
===============================

Side effects prevented:

* Finalizer execution.

* Because finalizers are prevented, longjmp state is safe from clobbering.

Side effects allowed:

* DECREF refzero free cascades.

* Mark-and-sweep, string table and object compactions.

Error code must be very careful not to throw an error in any part of the
error unwind process.  Otherwise sandboxing/protected call guarantees are
broken, and some of the side effect prevention changes are not correctly
undone (e.g. pf_prevent_count is bumped again!).  There are asserts in place
for the entire critical part (heap->error_not_allowed).

Call sites needing side effect protection
=========================================

Error throw and resulting unwind

* Must protect against another error: longjmp state doesn't nest.

* Prevent finalizers, avoid proxy traps.

* Avoid out-of-memory error throws, trial allocation is OK.

* Refzero with pure memory freeing is OK.

* Mark-and-sweep without finalizer execution is OK.  Object and string
  table compaction is OK:

Success unwind

* Must protect against error throws: otherwise state may be only
  partially unwound.  Same issues as with error unwind.

String table resize

* String table resize must be protected against string interning.

* Prevent finalizers, avoid Proxy traps.

* Avoid any throws, so that state is not left incomplete.

* Refzero with pure memory freeing is OK.

* Mark-and-sweep without finalizer execution is OK.  As of Duktape 2.1
  string interning is OK because it no longer causes a recursive string
  table resize regardless of interned string count.  String table itself
  protects against recursive resizing, so both object and string table
  compaction attempts are OK.

Object property table resize

* Prevent compaction of the object being resized.

* In practice, prevent finalizers (they may mutate objects) and proxy
  traps.  Prevent compaction of all objects because there's no fine
  grained control now (could be changed).

JSON fast path

* Prevent all side effects affecting property tables which are walked
  by the fast path.

* Prevent object and string table compaction, mark-and-sweep otherwise OK.

Object property slot updates (e.g. data -> accessor conversion)

* Property slot index being modified must not change.

* Prevent finalizers and proxy traps/getters (which may operate on the object).

* Prevent object compaction which affects slot indices even when properties
  are not deleted.

* In practice, use NORZ macros which avoids all relevant side effects.
