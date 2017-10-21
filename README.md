<img src="http://38.media.tumblr.com/db32471b7c8870cbb0b2cc173af283bb/tumblr_inline_nm9x9u6u261rw7ney_540.gif" height="170" width="100%" />


# Shards [![Build Status](https://travis-ci.org/cabol/shards.svg?branch=master)](https://travis-ci.org/cabol/shards)

ETS tables on steroids!

[**Shards**](https://github.com/cabol/shards) is an **Erlang/Elixir** library/tool
compatible with the ETS API, that implements [Sharding/Partitioning](https://en.wikipedia.org/wiki/Partition_(database)) support
on top of ETS totally transparent and out-of-box. **Shards** might be probably
the **simplest** option to scale-out ETS tables.

[Additional documentation on cabol.github.io](http://cabol.github.io/posts/2016/04/14/sharding-support-for-ets.html).


## Introduction

Why might we need **Sharding** on ETS tables? Well, the main reason is to keep
the lock contention under control, in order to scale-out ETS tables (linearly)
and support higher levels of concurrency without lock issues; specially
write-locks – which most of the cases might cause significant performance
degradation.

Therefore, one of the most common and proven strategies to deal with these problems
is [Sharding/Partitioning](https://en.wikipedia.org/wiki/Partition_(database)) –
the principle is pretty similar to [DHTs](https://en.wikipedia.org/wiki/Distributed_hash_table).

Here is where **Shards** comes in. **Shards** makes very easy to achieve
all these features with **zero** effort. It provides an API compatible with
[ETS](http://erlang.org/doc/man/ets.html) – with few exceptions. You can check
the list of compatible ETS functions that **Shards** provides [HERE](https://github.com/cabol/shards/issues/1).


## Installation

### Erlang

In your `rebar.config`:

```erlang
{deps, [
  {shards, "0.5.0"}
]}.
```

### Elixir

In your `mix.exs`:

```elixir
def deps do
  [{:shards, "~> 0.5"}]
end
```


## Getting Started!

### Build

    $ git clone https://github.com/cabol/shards.git
    $ cd shards
    $ make

 > **NOTE:** `shards` comes with a helper **Makefile** but it's a simple wrapper on top of `rebar3`,
   therefore, you can do everything using `rebar3` directly as well.

Now you can start an Erlang console with `shards` running:

    $ make shell

### Creating shards

```erlang
% let's create a table, such as you would create it with ETS, with 4 shards
> shards:new(mytab1, [{n_shards, 4}]).
mytab1
```

Exactly as ETS, `shards:new/2` function receives 2 arguments, the name of the table and
the options. With `shards` there are additional options:

Option | Description | Default
-------|-------------|--------
`{n_shards, pos_integer()}` | Allows to set the desired number of shards. | By default, the number of shards is calculated from the total online schedulers: `erlang:system_info(schedulers_online)`
`{scope, l | g}` | Defines `shards` scope, in other words, if sharding will be applied locally (`l`) or global/distributed (`g`) | `l`
`{restart_strategy, one_for_one | one_for_all}` | Allows to configure the restart strategy for `shards_owner_sup`. | `one_for_one`
`{pick_shard_fun, pick_fun()}` | Function to pick the **shard** on which the `key` will be handled locally – used by `shards_local`. See [shards_state](./src/shards_state.erl). | `shards_lib:pick/3`
`{pick_node_fun, pick_fun()}` | Function to pick the **node** on which the `key` will be handled globally/distributed – used by `shards_dist`. See [shards_state](./src/shards_state.erl). | `shards_lib:pick/3`
`{nodes, [node()]}` | Allows to set a list of nodes to auto setup a distributed table – the table is created in all given nodes and then all nodes are joined. This option only has effect if the option `{scope, g}` has been set.  | `[]`
`{sup_name, atom()}` | Allows to define a custom name for `shards_sup`. | `shards_sup`

> **NOTE:** By default `shards` uses a built-in functions to pick the **shard** (local scope)
  and the **node** (distributed scope) on which the key will be handled. BUT you can override
  them and set your own functions, they are totally configurable by table, so you can have
  different tables with different pick-functions each.

Furthermore, when a new table is created, the [<i class="icon-refresh"></i> **State**](#state)
is created for that table as well. The purpose of the **State** is to store information
related to that table, such as: number of shards, pick functions, etc. For more
information go to the [<i class="icon-refresh"></i> **State Section**](#state).

You can get the **State** at any time you want:

```erlang
> shards_state:get(mytab1).
{state,shards_local,shards_sup,4,#Fun<shards_local.pick.3>,
       #Fun<shards_local.pick.3>}

% this is a wrapper on top of shards_state:get/1
> shards:state(mytab1).
{state,shards_local,shards_sup,4,#Fun<shards_local.pick.3>,
       #Fun<shards_local.pick.3>}
```

> **NOTE:** For more information about `shards:new/2` see [shards](./src/shards.erl).

Let's continue:

```erlang
% create another one with a default number of shards, which is the total of online
% schedulers; in my case is 8 (4 cores, 2 threads each).
% This value is calculated calling: erlang:system_info(schedulers_online)
> shards:new(mytab2, []).
mytab2

% now open the observer so you can see what happened
> observer:start().
ok
```

You will see the process tree of `shards` application. When you create a new "table,"
what happens behind is: `shards` creates a supervision tree dedicated only to that
group of shards that will represent your logical table in multiple physical ETS
tables and everything is handled auto-magically by `shards`, you only have to use
the API like if you were working with a common ETS table.

### Playing with shards

Now let's execute some write/read operations against the created `shards`:

```erlang
% inserting some objects
> shards:insert(mytab1, [{k1, 1}, {k2, 2}, {k3, 3}]).
true

% let's check those objects
> shards:lookup(mytab1, k1).
[{k1,1}]
> shards:lookup(mytab1, k2).
[{k2,2}]
> shards:lookup(mytab1, k3).
[{k3,3}]
> shards:lookup(mytab1, k4).
[]

% delete an object and then check
> shards:delete(mytab1, k3).
true
> shards:lookup(mytab1, k3).
[]

% now let's find all stored objects using select
> MatchSpec = ets:fun2ms(fun({K, V}) -> {K, V} end).
[{{'$1','$2'},[],[{{'$1','$2'}}]}]
> shards:select(mytab1, MatchSpec).
[{k1,1},{k2,2}]
```

As you may have noticed, it's extremely easy, because almost all ETS functions are
implemented by shards, it's only matters of replacing `ets` module by `shards`.

### Deleting shards

**Shards** behaves in an elastic way, as you saw, more shards can be added/removed dynamically:

```erlang
> shards:delete(mytab1).
true

> observer:start().
ok
```

See how `shards` gets shrinks.


## State

In the previous section we saw something about the `state`, how it can be fetched
at any time. But, what is the `state`?

There are different properties that have to be stored somewhere for `shards` to work
correctly. Remember, `shards` has a logic on top of `ETS`, for example, compute the
shard/node where the key/value pair goes, and to do that, it needs the number of shards,
the function to pick the shard or node (in case of global scope), the table type and
of course, the module to use depending on the scope (`shards_local` or `shards_dist`).

Because of that, when a new table is created using `shards`, a new supervision tree
is created as well to represent that table. The supervisor is `shards_owner_sup` and
it has a control ETS table to save the `state` so it can be fetched later at any time.

The `shards` state is defined as:

```erlang
-record(state, {
  module         = shards_local          :: module(),
  sup_name       = shards_sup            :: atom(),
  n_shards       = ?N_SHARDS             :: pos_integer(),
  pick_shard_fun = fun shards_lib:pick/3 :: pick_fun(),
  pick_node_fun  = fun shards_lib:pick/3 :: pick_fun()
}).
```

But this record is totally transparent for you, `shards` provides a dedicated module
to handle the `state`: [shards_state](./src/shards_state.erl). With this utility module,
you can fetch the state, get any property and also other functions.


## Using shards_local directly

The module `shards` is a wrapper on top of two main modules:

 * [shards_local](./src/shards_local.erl): Implements Sharding on top of ETS tables,
   but locally (on a single Erlang node).
 * [shards_dist](./src/shards_dist.erl): Implements Sharding but across multiple
   distributed Erlang nodes, which must run `shards` locally, since `shards_dist`
   uses `shards_local` internally. We'll cover the distributed part later.

When you use `shards` on top of `shards_local`, a call to the control ETS table
owned by `shards_owner_sup` must be done, in order to recover the
[<i class="icon-refresh"></i> State](#state), mentioned previously.

Most of the `shards_local` functions receive the **State** as a parameter, so it
must be fetched before to call it. You can check how the `shards` module is
implemented [HERE](./src/shards.erl).

If any microsecond matters to you, you can skip the call to the control ETS
table by calling `shards_local` directly – there are two options to do it.

### Option 1

The first option is getting the `state`, and passing it as an argument. Now the
question is: how to get the **State**? Well, it's extremely easy, you can get
the `state` when you call `shards:new/2` for the first time, or you can call
`shards:state/1`, `shards_state:get/1` or `shards_state:new/0,1,2,3,4` at any
time you want, and then it might be stored within the calling process, or
wherever you want. E.g.:

```erlang
% take a look at the 2nd element of the returned tuple, that is the state
> shards:new(mytab, [{n_shards, 4}]).
mytab

% remember you can get the state at any time you want
> State = shards:state(mytab).
{state,shards_local,shards_sup,4,#Fun<shards_local.pick.3>,
       #Fun<shards_local.pick.3>}

% now you can call shards_local directly
> shards_local:insert(mytab, {1, 1}, State).
true
> shards_local:lookup(mytab, 1, State).
[{1,1}]

% in this case, only the n_shards is different from default, so you
% can do this:
> shards_local:lookup(mytab, 1, shards_state:new(4)).
[{1,1}]
```

### Option 2

The second option is to call `shards_local` directly without the `state`, but
this is only possible if you have created a table with default `shards` options
– such as `n_shards`, `pick_shard_fun` and `pick_node_fun`. If you can take this
option, it might be significantly better, since in this case no additional calls
are needed, not even to recover the `state` (like in the previous option),
because a new `state` is created with default values. Therefore, the call is
mapped directly to an **ETS** function. E.g.:

```erlang
% create a table without set n_shards, pick_shard_fun or pick_node_fun
> shards:new(mytab, []).
mytab

% call shards_local without the state
> shards_local:insert(mytab, {1, 1}).
true
> shards_local:lookup(mytab, 1).     
[{1,1}]
```

Most of the cases this is not necessary, `shards` wrapper is more than enough,
it adds only a few microseconds of latency. In conclusion, **Shards** gives you
the flexibility to do it, but it's your call!


## Distributed Shards

So far, we've seen how **Shards** works locally, now let's see how **Shards**
works but distributed.

**1.** Let's start 3 Erlang consoles running shards:

Node `a`:

```
$ rebar3 shell --sname a@localhost
```

Node `b`:

```
$ rebar3 shell --sname b@localhost
```

Node `c`:

```
$ rebar3 shell --sname c@localhost
```

**2.** Create a table with global scope (`{scope, g}`) on each node and then join them.

There are two ways to achieve this:

 1. Manually, create the table on each node and then from any of them join the rest.

    Create the table on each node:

    ```erlang
    % when a tables is created with {scope, g}, the module shards_dist is used
    % internally by shards
    > shards:new(mytab, [{scope, g}]).
    mytab
    ```

    Join them. From node `a`, join `b` and `c` nodes:

    ```erlang
    > shards:join(mytab, ['b@localhost', 'c@localhost']).
    [a@localhost,b@localhost,c@localhost]
    ```

    Let's check that all nodes have the same nodes running next function on each node:

    ```erlang
    > shards:get_nodes(mytab).
    [a@localhost,b@localhost,c@localhost]
    ```

 2. The easier way, call `shards:new/3` but passing the option `{nodes, Nodes}`,
    where `Nodes` is the list of nodes you want to join.

    From Node `a`:

    ```erlang
    > shards:new(mytab, [{scope, g}, {nodes, ['b@localhost', 'c@localhost']}]).
    mytab
    ```

    Let's check again on all nodes:

    ```erlang
    > shards:get_nodes(mytab).
    [a@localhost,b@localhost,c@localhost]
    ```

**3.** Now **Shards** cluster is ready, let's do some basic operations:

From node `a`:

```erlang
> shards:insert(mytab, [{k1, 1}, {k2, 2}]).
true
```

From node `b`:

```erlang
> shards:insert(mytab, [{k3, 3}, {k4, 4}]).
true
```

From node `c`:

```erlang
> shards:insert(mytab, [{k5, 5}, {k6, 6}]).
true
```

Now, from any of previous nodes:

```erlang
> [shards:lookup_element(mytab, Key, 2) || Key <- [k1, k2, k3, k4, k5, k6]].
[1,2,3,4,5,6]
```

All nodes should return the same result.

Let's do some deletions, from any node:

```erlang
> shards:delete(mytab, k6).
true
```

And again, let's check it out from any node:

```erlang
% as you can see 'k6' was deleted
> shards:lookup(mytab, k6).
[]

% check remaining values
> [shards:lookup_element(mytab, Key, 2) || Key <- [k1, k2, k3, k4, k5]].    
[1,2,3,4,5]
```

> **NOTE**: This module is still under continuous development. So far, only few
  basic functions have been implemented.


## Important links

* [Blog Post – including load tests results](http://cabol.github.io/posts/2016/04/14/sharding-support-for-ets.html).

* [ExShards](https://github.com/cabol/ex_shards) – Elixir wrapper for
  `shards` with extra and nicer functions.

* [Nebulex](https://github.com/cabol/nebulex) – Local and Distributed Caching
  Tool for Elixir.

* [KVX](https://github.com/cabol/kvx) – Simple/basic Elixir in-memory Key/Value
  Store using `shards` (default adapter).

* [Cacherl](https://github.com/ferigis/cacherl) uses `shards` to implement a
  Distributed Cache.


## Running Tests

    $ make test

You can find tests results in `_build/test/logs`, and coverage in `_build/test/cover`.


## Building Edoc

    $ make doc

> **Note:** Once you run the previous command, a new folder `doc` is created,
  and you'll have a pretty nice HTML documentation.


## Copyright and License

Copyright (c) 2016 Carlos Andres Bolaños R.A.

**Shards** source code is licensed under the [MIT License](LICENSE.md).
