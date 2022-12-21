---
layout: post
title: The challenges of writing safe Lua abstractions in Rust
tags: coding
---

For the past few months, as an exercise of writing low-level Rust/C interop code, I've been trying to write a safe high-level interface for the [Lua virtual machine][1] in Rust.
I may or may not publish my code as a crate eventually, but regardless, here are a few challenges I've faced.

### Ensuring stack space

Most Lua APIs interact with the stack, and the [Lua documentation][2] emphasizes that you (the caller) must ensure there is enough stack space before calling a function that modifies the stack.
The stack is a FIFO collection, and the API offers many functions to push, pop and inspect the elements inside it.
The API does not perform bounds checking however, so if you don't ensure enough space for an operation, it will trigger **undefined behavior**.

Writing safe Rust code means it can never trigger UB.
Under no circumstances should safe code ever be able to trigger UB, which means our safe abstraction must be able to prevent every edge case that may trigger UB.
If safe code somehow manages to trigger UB, we can only ever hope for a nice segfault.

This is a deceptively hard problem to solve. One naive solution is to simply call [`lua_checkstack`][3] before all calls that push to the stack.

```c
assert(lua_checkstack(L, 1)); // ensure stack space for the number
lua_pushnumber(L, 1.2);
```

We can then create a safe abstraction for pushing to a stack, so that the caller won't have to worry about stack spaces.

```rust
struct Stack(*mut lua_State);

impl Stack {
    fn push_number(&self, n: f64) {
        unsafe {
            assert!(lua_checkstack(self.0, 1) != 0);
            lua_pushnumber(self.0, n);
        }
    }
}
```

However, calling `lua_checkstack` before every stack operation seems rather inefficient.
Although Rust incurs essentially no performance penalty for calling C functions, and the function itself should be a quick pointer comparison most of the time, it's more efficient to perform one stack check and reallocation for a large set of operations if possible.
Also, as I'll discuss later, this combining of stack checks actually becomes necessary to support proper error handling without significant overhead.
For example, the following two snippets are equivalent, but the latter is clearly more efficient:

```c
// lua equivalent: { foo = "bar" }
assert(lua_checkstack(L, 1)); // ensure stack space for the table
lua_newtable(L);
assert(lua_checkstack(L, 1)); // ensure stack space for the value
lua_pushstring(L, "bar");
lua_setfield(L, -2, "foo"); // set the field

// optimized equivalent:
assert(lua_checkstack(L, 2)); // ensure stack space for both the table and value
lua_newtable(L);
lua_pushstring(L, "bar");
lua_setfield(L, -2, "foo"); // set the field
```

How could we group these related operations together and predict the total required stack space in advance?
One elegant solution is to represent each stack operation as an _expression_.
The creation of a table is an expression, and the pushing of a string is also an expression.
The setting of a field is a _composite expression_ taking three child expressions — the table, key and value — forming an expression tree.
We can then take advantage of Rust's expressive type system and static analysis to accurately predict the required stack space at **compile time**.

For simplicity and illustrative purposes, let's assume that evaluating an expression pushes exactly one value onto the stack.
This would be more complex if we were to support [multi-valued expressions][4].
We can create an abstraction for expressions using _traits_, and implement the trait for every type that can be evaluated on the stack.

```rust
trait Expression {
    /// The number of stack slots required for this expression.
    const STACK_SIZE: c_int;

    /// Evaluates this expression on the stack.
    ///
    /// # Safety
    ///
    /// Caller must ensure there is enough stack space.
    unsafe fn eval(&self, stack: *mut lua_State);
}

struct NewTable;
struct PushString(String);
struct SetField<T: Expression, K: Expression, V: Expression>(T, K, V);
```

Implementing `Expression` for `NewTable` and `PushString` is straightforward; both require only one stack space for the push operation.

```rust
impl Expression for NewTable {
    const STACK_SIZE: c_int = 1;

    unsafe fn eval(&self, stack: *mut lua_State) {
        lua_newtable(stack);
    }
}

impl Expression for PushString {
    const STACK_SIZE: c_int = 1;

    unsafe fn eval(&self, stack: *mut lua_State) {
        lua_pushlstring(stack, self.0.as_ptr() as *const c_char, self.0.len());
    }
}
```

Implementing `Expression` for `SetField` is less straightforward, but it is intuitive.
Each operand of the expression can be arbitrarily complex themselves, but we know that evaluating them will push exactly one value onto the stack each as a result.
Thus the required stack space for a `SetField` is the maximum of its operands' required stack space plus whatever is already on the stack from evaluating the previous operands.

```rust
impl<T: Expression, K: Expression, V: Expression> Expression for SetField<T, K, V> {
    const STACK_SIZE: c_int = max(
        T::STACK_SIZE,
        1 + K::STACK_SIZE, // +1 to account for the table on the stack
        2 + V::STACK_SIZE, // +2 to account for the table and key
    );

    unsafe fn eval(&self, stack: *mut lua_State) {
        self.0.eval(stack); // push table
        self.1.eval(stack); // push key
        self.2.eval(stack); // push value
        lua_settable(stack, -3); // pop key and value into field
        // the table is left on the stack
    }
}
```

Then it is simply a matter of composing the expressions and inspecting the stack size constant in order to evaluate them safely.
We can create arbitrarily complex expressions and the compiler will always be able to predict the exact amount of required stack space (well, as far as [`recursion_limit`][11] allows).

```rust
impl Stack {
    fn eval<E: Expression>(expr: &E) {
        unsafe {
            // ensure stack space for the entire expression
            assert!(lua_checkstack(self.0, E::STACK_SIZE) != 0);
            expr.eval(self.0);
            lua_pop(self.0);
        }
    }
}

// compose and evaluate complex expression
let stack: Stack = ...;
let table = NewTable;
let key = PushString("foo".into());
let value = PushString("bar".into());
stack.eval(SetField(table, key, value)); // { foo = "bar" }
```

Finally, a safe high-level Lua abstraction with little overhead... Right?

### Error handling

Unfortunately, it turns out that the code above is completely unsafe due to the lack of error handling.
Many Lua APIs can throw _exceptions_, for example if the allocation for the new table or string fails.
Reading and writing table fields can also throw exceptions because the table may have _metamethods_.
However, whereas a C++ runtime would be able to throw _real_ exceptions, Lua is written in C which doesn't have real exceptions; instead it emulates them using [`longjmp`][5].

A `longjmp` is essentially a `goto` that is not restricted to a function scope.
It can jump up _anywhere_ there is a `setjmp`.
Lua uses these functions to create a _protected context_.
Before calling some fallible code, it creates a recovery point with a `setjmp` and then calls the fallible code.
If the fallible code wants to throw an exception, it pushes the exception onto the stack and then performs a `longjmp` back to the recovery point.
This achieves an effect similar to a C++ `try`/`catch` block.

At this point, it's probably necessary to clarify some terminologies.
There is the _Lua stack_ which is a heap-allocated structure that contains Lua values.
_Lua code_ uses this stack for passing function arguments and return values, holding local variables, etc.
A _Lua stack_ are tied to a _Lua thread_, and in practice these are the same thing.
Lua threads are also called _coroutines_ or _green threads_; there seems to be no consensus on how to call this thing.

There is also the _call stack_ which is a block of raw memory where _native code_ stores things like local variables, return addresses, etc.
Both _C code_ and _Rust code_ share the same call stack.
A _call stack_ is tied to a real _OS thread_.

With that in mind, let's imagine what might happen when the Lua runtime encounters some Lua code that wants to throw an exception.
It pushes the exception value onto the Lua stack, then performs a `longjmp` back to the recovery point.
This is like traversing _up_ the call stack with a `goto` until the recovery point, _forgetting_ all stack frames between the `longjmp` and `setjmp` points.
The recovery point then detects that a `longjmp` had occurred and pops the exception value from the Lua stack.

This is perfectly fine in the context of plain old C code.
However, where it becomes problematic is when _Rust_ code is intermingled with C code.
The issue with `longjmp` is that it is completely unaware of _destructors_.
Unlike real exception mechanisms that properly unwind the call stack and call destructors for the values on every stack frame, `longjmp` just restores the call stack to the point of `setjmp` as if all stack frames that occurred between the two points didn't even exist.
If one of those [disappearing stack frames][12] happened to be _Rust's_ stack frame, then any Rust value implementing `Drop` in that frame would not be deallocated properly.

It's interesting to note that forgetting stack frames like this is technically **safe**.
Safe Rust does not actually guarantee that `Drop` is called!
If we imagine the stack frames as being Rust values, doing a `longjmp` is like [calling `mem::forget` on the stack frames][6]. `mem::forget` is a **safe** function.

However, it's clearly in our best interests to prevent stack frames from getting forgotten, because not calling destructors will obviously leak memory.
That means we need to wrap every fallible code in a protected context so that the exception is caught before it touches Rust stack frames.
Fortunately, Lua provides `lua_pcall` to do exactly this.

### Lua 5.1 and LuaJIT complications

To fix the earlier examples, we could try wrapping the code for allocating the table and string using `lua_pushcfunction`.
This creates a Lua function, that calls back into Rust code, that calls back into Lua to create a new table, and we can call _that_ wrapper function in a protected context using `lua_pcall`.

```rust
impl Expression for NewTable {
    const STACK_SIZE: c_int = 1; // one for the callback function

    unsafe fn eval(&self, stack: *mut lua_State) {
        unsafe extern "C" fn cb_newtable(L: *mut lua_State) -> c_int {
            // lua guarantees 20 stack space; no checkstack necessary
            lua_newtable(L);
            1 // return the new table
        }

        lua_pushcfunction(stack, cb_newtable);
        assert!(lua_pcall(stack, 0, 1, 0) == 0, "out of memory");
        // the table is now on the stack
    }
}
```

This code should be perfectly safe. Problem solved?
Unfortunately in Lua 5.1 and LuaJIT implementations, [`lua_pushcfunction` can also throw a memory error][7] if allocation for the Lua wrapper for the C function fails.

We could use `lua_cpcall` instead, which only exists in Lua 5.1 and LuaJIT.
It guarantees that it will never throw, but there is a catch:
The callback called by `lua_cpcall` cannot return any values.
Why does this limitation exist? I'm not sure.
One way to circumvent this is to create a separate temporary thread to perform the fallible operation on, and then move the result to the main stack.

```rust
impl Expression for NewTable {
    const STACK_SIZE: c_int = 2; // two for the thread and table

    unsafe fn eval(&self, stack: *mut lua_State) {
        // create a temporary thread for fallible operation
        // this also pushes the thread onto the main stack
        let temp = lua_newthread(stack);

        unsafe extern "C" fn cb_newtable(L: *mut lua_State) -> c_int {
            // pointer to the main stack given as the first argument
            let stack = lua_topointer(L, 1) as *mut lua_State;
            // lua guarantees 20 stack space; no checkstack necessary
            lua_newtable(L);
            // move the new table to the main stack
            lua_xmove(L, stack, 1);

            0
        }

        // create the table on the temporary thread
        assert!(lua_cpcall(temp, cb_newtable, stack as *mut c_void) == 0, "out of memory");

        // the table is now on the stack; remove temporary thread
        lua_replace(stack, -2);

        // the table is left on the stack
    }
}
```

Okay, now this code should be perfectly safe. Problem solved?
Unfortunately, `lua_newthread` can also throw a memory error if allocation fails.
To create a temporary thread to catch exceptions, we need to create another temporary thread to safely create the temporary thread, like a chicken and egg problem.
One way to circumvent this is to create the temporary thread on runtime initialization, cache it to the registry and retrieve it when necessary, since reading a field from a table will never throw (without metamethods, at least).

As you can see, even the deceptively simple act of creating a new empty table can get quite convoluted when creating a safe abstraction that can handle every edge case.
It seems impossible to create a safe "zero-cost" abstraction for Lua when the API does not provide any way of handling errors in a zero-cost manner.

There is another edge case we need to handle when interacting with Lua 5.1 and LuaJIT: `lua_checkstack` can throw a memory error.
That's right, the very function responsible for checking if the stack can be resized, will also throw if resizing the stack fails, so it must also be wrapped in a protected context...

```rust
impl Stack {
    fn eval<E: Expression>(expr: &E) {
        unsafe {
            unsafe extern "C" fn cb_checkstack<E: Expression>(L: *mut lua_State) -> c_int {
                // this can throw since we are protected
                luaL_checkstack(L, E::STACK_SIZE, b"out of memory".as_ptr() as *const c_char);
                0
            }

            assert!(lua_cpcall(self.0, cb_checkstack::<E>, ptr::null_mut()) == 0, "out of memory");
            expr.eval(self.0);
            lua_pop(self.0);
        }
    }
}
```

This circles back to the previously discussed point about combining stack checks for a large set of operations.
A simple `lua_checkstack` may be cheap and unsafe, but wrapping that in `lua_cpcall` for safety probably adds a non-negligible overhead.
Combining the stack checks is a nice compromise that achieves both safety and performance.

### Zero-cost error handling using C++ exceptions

Everything I've discussed so far relied on `lua_pcall` to create a protected context.
This is reasonable because a Rust library that interacts with a Lua host cannot assume anything about how the exception mechanism is actually implemented under the hood, and on most systems that would be `setjmp`/`longjmp`.
However when Rust code is the host of Lua code, we have the option of compiling Lua to use _real_ C++ exceptions as its exception mechanism.

This may allow us to ditch `lua_pcall` and all the dance around temporary threads, but Rust code still cannot catch C++ exceptions directly, or so it seems.
Rust panics are _probably_ implemented using the same unwinding mechanism (libunwind) as Lua (if compiled in C++ with Clang?) which could allow us to catch exceptions directly in Rust using `catch_unwind`, but the documentation explicitly declares this as [undefined behavior][8].
We could also try using the [`try` compiler intrinsic][9] directly, but this nightly-only API will never be stabilized and there seems to be very little documentation around its usage.

As far as I can tell, the only safe and stable way of catching C++ exceptions in Rust that doesn't rely on black magic is manually writing C++ wrapper functions for every Lua function that may throw and linking them into Rust.
It would be a massive PITA, but there shouldn't be any overhead from calling C++ wrapper functions if [link-time optimizations][10] are enabled.

I've yet to explore this area further, but the possibility of safe zero-cost abstractions makes exploring it seem worthwhile in the future.

[1]: https://www.lua.org
[2]: http://www.lua.org/manual/5.1/manual.html#3.2
[3]: http://www.lua.org/manual/5.1/manual.html#lua_checkstack
[4]: https://www.lua.org/pil/5.1.html
[5]: https://en.wikipedia.org/wiki/Setjmp.h
[6]: https://github.com/rust-lang/unsafe-code-guidelines/issues/210
[7]: http://lua-users.org/lists/lua-l/2021-11/msg00077.html
[8]: https://doc.rust-lang.org/std/panic/fn.catch_unwind.html#notes
[9]: https://doc.rust-lang.org/std/intrinsics/fn.try.html
[10]: https://doc.rust-lang.org/cargo/reference/profiles.html#lto
[11]: https://doc.rust-lang.org/reference/attributes/limits.html#the-recursion_limit-attribute
[12]: https://blog.rust-lang.org/inside-rust/2021/01/26/ffi-unwind-longjmp.html
