# ppx_version

`ppx_version` contains OCaml extension points (ppxs) meant to assure
the stability of types and their `Bin_prot` serializations. With such
stability, data can be persisted and restored, or communicated over
networks reliably, even as software evolves.

To define a stable type:

```ocaml
[%%versioned
  module Stable = struct
    module V1 = struct
	  type t = int * string

      let to_latest (n,s) = (n,s)
    end
  end]
```

The use of `%%versioned` generates a `Bin_prot` typeclass instance in
the same way that annotating the type with `[@@deriving bin_io]` would
do. A module alias `Latest` is also generated; it alias the
highest-numbered versioned module (in this example, `V1`). Whenever a
new versioned module is created, the `to_latest` functions for earlier
modules need to be updated. For a module `Vn`, the `to_latest` function
has type `Vn.t -> Latest.t`.

The `Stable` module contains the generated function
```ocaml
val deserialize_binary_opt : Bin_prot.Common.buf -> Stable.Latest.t option
```
Given a buffer containing a serialization of a type from any of the
versioned modules, this function will return an instance of `Stable.Latest.t`.
The return type is an option, because the serialization may be from an
unknown version.

The variant `%%versioned_asserted` can be used when the types used in
the definition are not themselves versioned, as when they're
from some external library. In that case, you're required to
provide a `Tests` module. You should provide tests that verify that
serializations of the types in the versioned module don't change.

The variant `%%versioned_binable` can be used when the functor
`Binable.Of_binable` (or `Of_binable1`, `Of_binable2`), or
`Binable.Of_stringable` are used to provide serialization.

Another place you may want stable types is in the definition of
Jane Street `async_rpc_kernel` versioned RPC calls. The
idiom is:
```ocaml
module V1 = struct
  module T = struct
    type query = int [@@deriving bin_io, version {rpc}]

    type response = string [@@deriving bin_io, version {rpc}]

    let query_of_caller_model = ...

    let callee_model_of_query = ...

    let response_of_callee_model = ...

    let caller_model_of_response = ...
  end

  include T
  include Register (T)
  end
```
See Jane Street's `Async` library for details on how to define
and use versioned RPC calls.

For both ordinary versioned modules, and versioned RPC modules, the
linter in `ppx_version` checks that all the types used in these
definitions are themselves versioned types, or OCaml builtin types,
such as `int` and `string`.  Therefore, these types are versioned
all-the-way-down.

The linter also enforces a number of other rules.

One linter rule is that stable-versioned modules cannot be contained
in the result of a functor application.  The reason is that the types
might depend on the functor arguments, so that distinct functor
applications yield different types with different serializations. An
exception to that rule is when the functor takes no arguments.

Another rule is that types from stable-versioned modules with a
specific version can only be used to define other types in
stable-versioned modules. In other settings, use `Stable.Latest.t`.
Similarly, modules passed as arguments to functions cannot
be a specific-versioned module; use `Stable.Latest` instead.

The linter does not examine code in tests that use the `let%test`,
`let%test_unit` or `let%test_module` forms used with the Jane Street
`ppx_inline_tests` package. Anything goes in tests.

A serializable type may not need to be versioned, because the
serialization is neither persisted nor shared with different software.
In that case, annotate the type with `[@@deriving bin_io_unversioned]`.
