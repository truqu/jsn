jsn
===

[![Build Status](https://secure.travis-ci.org/nalundgaard/jsn.png?branch=master)](http://travis-ci.org/nalundgaard/jsn)

jsn is a tool for working with JSON representations in erlang--complex, nested
JSON objects in particular.

In the spirit of [ej][ej], it supports the common formats output by JSON
decoders such as [jsonx][jsonx], [jiffy][jiffy], and [mochijson2][mochijson2].
Unlike [ej][ej], however, it supports _all three_ common JSON representations
in Erlang:

* `proplist` (**default**)(common to [jsx][jsx] and [jsonx][jsonx])
* `eep18` (common to [jiffy][jiffy] and [jsonx][jsonx])
* `struct` (common to [mochijson2][mochijson2]) 

In addition to supporting the additional `proplist` format, jsn's path
input structure is somewhat more flexible, allowing for input of
period-delimited binary strings or atoms to indicate a path through a
deeply nested structure. This support is similar to [kvc][kvc]'s path format,
and also likely to be familiar to users of [erlson][erlson].

This code base was originally developed as a wrapper around [ej][ej], adding
support for the 'syntactic sugar' of the period-delimited keys. However, a
need arose for the library to be proplist-compatible, so it has been refactored
to be a nearly standalone library.

Running 
-------

To run this library locally, build the source code:

```sh
make deps all
```

Then start an erlang shell:

```sh
./erl.sh
```

jsn is an OTP library, and does not really need to be started as such. 

Application Integration
-----------------------

To add `jsn` to your Erlang OTP application, simply add it to your
`rebar.config`:

```erlang
{deps, [
    %% ... your deps ...
    {jsn, "", {git, "git@github.com:nalundgaard/jsn.git", {branch, master}}}

]}.
```

After you get and compile deps, you will have full access to jsn from your
local console.

Paths & Indexes
---------------

Paths are pointers into a (potentially nested) jsn object. An object may contain
sub-objects or arrays at any layer, and as such, a path may include both keys
(as binary strings and sometimes atoms) as well as array indexes. Indexes can be 
provided as positive integers (i.e., `1` is the first element of an array) or as
the shortcut atoms `'first'` and `'last'`.

There are 3 different supported path styles, each with different tradeoffs:

1. List of binary/atom keys, e.g. `[<<"person">>, 'id']`.
   Mixing and matching atom and binary string keys is supported, but using
   binary keys only is most performant (and matches the 'native' path format).
   However, this format **does not** support array indexes.
2. Tuple of binary keys and array indexes, e.g., `{<<"users">>, last}`.
   Atom keys are not supported due to potential ambiguity. This is the
   only possible path format to use if you want to leverage the array
   index feature.
3. Period-delimited atom or binary string, e.g. `<<"user.id">>` or `'user.id'`.
   This format is the most compact and readable, but only supports keys
   (no array indexes).

Library Functions
-----------------

jsn provides functions to create, append, delete, and transform objects in all
supported formats (`proplist`, `eep18`, and `struct`). This section contains a
reference for the primary library functions available.

### `new/0,1,2` - Create a new object

* `new()` - Create an empty object in the default format.
* `new(TupleOrTupleList)` - Given a `{Path, Value}` tuple or a list of such
  tuples, return a new object (in the default format) containing the given
  path(s) and value(s).
* `new(TupleOrTupleList, Options)` - Identical to new/1, but with the addition
  of Options, which allow passing a specific format.

Examples:

```erlang
% create an empty object
jsn:new().
% []

% create an object using a single path, value pair.
jsn:new({'user.id', <<"123">>}).
% [{<<"user">>,[{<<"id">>,<<"123">>}]}]

% create an object using a list of path, value pairs.
jsn:new([{'user.id', <<"123">>}, {<<"user.name">>, <<"John">>}]).
% [{<<"user">>,
%   [{<<"id">>,<<"123">>},{<<"name">>,<<"John">>}]}]

% create a jsn object in eep18 format
jsn:new([{'user.id', <<"123">>},
         {<<"user.name">>, <<"John">>}], [{format, eep18}]).
% {[{<<"user">>,
%    {[{<<"id">>,<<"123">>},{<<"name">>,<<"John">>}]}}]}

% create a jsn object in struct (mochijson2) format
jsn:new([{'user.id', <<"123">>},
         {<<"user.name">>, <<"John">>}], [{format, struct}]).
% {struct,[{<<"user">>,
%           {struct,[{<<"id">>,<<"123">>},{<<"name">>,<<"John">>}]}}]}
```

### `get/2,3`, `get_list/2,3`, and `find/3,4` - Extract data from objects

* `get(Path, Object)` - Return the value at Path in the Object, or `undefined`
  if it is missing.
* `get(Path, Object, Default)` - Identical to `get/2`, but returns `Default`
  instead of undefined.
* `get_list(PathList, Object)` - Identical to `get/2`, but expects a list of
  Paths, and returns a corresponding list of values.
* `get_list(PathList, Object, Default)` - Identical to `get/3`, but expects a
  list of Paths, and returns a corresponding list of values.
* `find(Path, SearchTerm, Objects)` - Find all the elements of the Objects list
  where the given Path in the element matches the SearchTerm, and return them.
* `find(Path, SubPath, SearchTerm, Object)` - Get the Path from the given
  Object (should be a list of objects), and find all the elements in the list
  where the given SubPath in the element matches SearchTerm, and return them.

Examples:

```erlang
User = jsn:new([{'user.id', <<"123">>},
                {'user.activated', true},
                {'user.name.first', <<"Jane">>},
                {'user.name.last', <<"Doe">>}]).
% [{<<"user">>,
%   [{<<"id">>,<<"123">>},
%    {<<"activated">>,true},
%    {<<"name">>,
%     [{<<"first">>,<<"Jane">>},{<<"last">>,<<"Doe">>}]}]}]

% get the user id
UserId = jsn:get('user.id', User).
% <<"123">>

% get a non-existant field, with and without a custom default
jsn:get(<<"user.deleted">>, User).
% undefined
jsn:get([<<"user">>, <<"deleted">>, User, false).
% false

% get several fields in a single call:
jsn:get_list(['user.name.first', 'user.name.last', 'user.name.middle'], User).
% [<<"Jane">>,<<"Doe">>,undefined]
jsn:get_list(['user.activated', 'user.deleted'], User, false).
% [true,false]

User2 = jsn:new([{'user.id', <<"456">>},
                 {'user.name.first', <<"Eve">>},
                 {'user.name.middle', <<"L.">>},
                 {'user.name.last', <<"Doer">>}]).
% [{<<"user">>,
%   [{<<"id">>,<<"456">>},
%    {<<"name">>,
%     [{<<"first">>,<<"Eve">>},
%      {<<"middle">>,<<"L.">>},
%      {<<"last">>,<<"Doer">>}]}]}]

% find the first user by id:
[User] = jsn:find({<<"user">>, <<"id">>}, <<"123">>, [User, User2]).
% [[{<<"user">>,
%    [{<<"id">>,<<"123">>},
%     {<<"activated">>,true},
%     {<<"name">>,
%      [{<<"first">>,<<"Jane">>},{<<"last">>,<<"Doe">>}]}]}]]
```

### `set/3` and `set_list/2` - Add to and update existing objects

* `set(Path, Object, Value)` - Append (or update) the Object by setting Path
  to Value, and return the result.
* `set_list(TupleList, Object)` - Given a list of `{Path, Value}` tuples, apply
  the `set/3` function to the each Path and Value using the given object, and
  return the result.

Examples:

```erlang
User = jsn:new([{'user.id', <<"123">>},
                {'user.activated', true},
                {'user.name.first', <<"Jane">>},
                {'user.name.last', <<"Doe">>}]).
% [{<<"user">>,
%   [{<<"id">>,<<"123">>},
%    {<<"activated">>,true},
%    {<<"name">>,
%     [{<<"first">>,<<"Jane">>},{<<"last">>,<<"Doe">>}]}]}]

% Set Jane's middle name
jsn:set([<<"user">>, <<"name">>, <<"middle">>], User, <<"Jacqueline">>).
% [{<<"user">>,
%   [{<<"id">>,<<"123">>},
%    {<<"activated">>,true},
%    {<<"name">>,
%     [{<<"first">>,<<"Jane">>},
%      {<<"last">>,<<"Doe">>},
%      {<<"middle">>,<<"Jacqueline">>}]}]}]

% Deactivate Jane's User, and change her middle name
jsn:set_list([{'user.activated', false},
              {'user.name.middle', <<"Jay">>}], User).
% [{<<"user">>,
%   [{<<"id">>,<<"123">>},
%    {<<"activated">>,false},
%    {<<"name">>,
%     [{<<"first">>,<<"Jane">>},
%      {<<"last">>,<<"Doe">>},
%      {<<"middle">>,<<"Jay">>}]}]}]
```

### `delete/2`, `delete_list/2`, and `delete_if_equal/2` - Remove data from existing objects

* `delete(Path, Object)` - Given a Path, remove it from the Object, if it
  exists, and return the result.
* `delete_list(PathList, Object)` - Given a list of paths, apply `delete/2` to
  Object and return the result.
* `delete_if_equal(Paths, ValueOrValues, Object)` - Given a list of paths and a
  Value or list of Values, check each path Value/Values, and if equal, remove the
  matching Path, Value pair from the object, and return the result.

Examples:

```erlang
Company = jsn:new([{'company.name', <<"Foobar, Inc.">>},
                   {'company.created.by', <<"00000000">>},
                   {'company.created.at', 469778436},
                   {'company.location', <<"U.S. Virgin Islands">>},
                   {{<<"company">>, <<"employees">>, 1, <<"id">>}, <<"00000000">>},
                   {{<<"company">>, <<"employees">>, 1, <<"name">>}, <<"Alice">>},
                   {{<<"company">>, <<"employees">>, 1, <<"position">>}, <<"CEO">>},
                   {{<<"company">>, <<"employees">>, 2, <<"id">>}, <<"00000001">>},
                   {{<<"company">>, <<"employees">>, 2, <<"name">>}, <<"Bob">>},
                   {{<<"company">>, <<"employees">>, 2, <<"position">>}, <<"CTO">>}]).
% [{<<"company">>,
%   [{<<"name">>,<<"Foobar, Inc.">>},
%    {<<"created">>,
%     [{<<"by">>,<<"00000000">>},{<<"at">>,469778436}]},
%    {<<"location">>,<<"U.S. Virgin Islands">>},
%    {<<"employees">>,
%     [[{<<"id">>,<<"00000000">>},
%       {<<"name">>,<<"Alice">>},
%       {<<"position">>,<<"CEO">>}],
%      [{<<"id">>,<<"00000001">>},
%       {<<"name">>,<<"Bob">>},
%       {<<"position">>,<<"CTO">>}]]}]}]

% remove the location from Company
jsn:delete('company.location', Company).
% [{<<"company">>,
%   [{<<"name">>,<<"Foobar, Inc.">>},
%    {<<"created">>,
%     [{<<"by">>,<<"00000000">>},{<<"at">>,469778436}]},
%    {<<"employees">>,
%     [[{<<"id">>,<<"00000000">>},
%       {<<"name">>,<<"Alice">>},
%       {<<"position">>,<<"CEO">>}],
%      [{<<"id">>,<<"00000001">>},
%       {<<"name">>,<<"Bob">>},
%       {<<"position">>,<<"CTO">>}]]}]}]

% delete Bob and the location in one call
jsn:delete_list(['company.location', {<<"company">>, <<"employees">>, last}], Company).
% [{<<"company">>,
%   [{<<"name">>,<<"Foobar, Inc.">>},
%    {<<"created">>,
%     [{<<"by">>,<<"00000000">>},{<<"at">>,469778436}]},
%    {<<"employees">>,
%     [[{<<"id">>,<<"00000000">>},
%       {<<"name">>,<<"Alice">>},
%       {<<"position">>,<<"CEO">>}]]}]}]

% conditionally delete the company Location
SecretLocations = [<<"Nevada">>, <<"Luxembourg">>, <<"U.S. Virgin Islands">>].
jsn:delete_if_equal('company.location', SecretLocations, Company).
% [{<<"company">>,
%   [{<<"name">>,<<"Foobar, Inc.">>},
%    {<<"created">>,
%     [{<<"by">>,<<"00000000">>},{<<"at">>,469778436}]},
%    {<<"employees">>,
%     [[{<<"id">>,<<"00000000">>},
%       {<<"name">>,<<"Alice">>},
%       {<<"position">>,<<"CEO">>}],
%      [{<<"id">>,<<"00000001">>},
%       {<<"name">>,<<"Bob">>},
%       {<<"position">>,<<"CTO">>}]]}]}]
```

### `copy/3,4` and `transform/2` - Re-shaping existing objects.

* `copy(PathList, Src, Dst)` - Given a list of paths, a source object, and one
  or more destination objects, copy the paths and values from Src to the
  destination(s). This function always returns a list of objects.
* `copy(PathList, Src, Dst, Mutator)` - Like above, but pass the value through
  a function/1 which mutates the set value.
* `transform(Transforms, Object)` - Given a list of `{Path, fun/1}` tuples,
  apply the fun to the value at path and modify the given object. Return the
  new object.

Examples:

```erlang
Source = jsn:new([{'key1', <<"value1">>},
                  {'key2', <<"value2">>},
                  {'key3', <<"value3">>},
                  {'key3', <<"value3">>}]).
% [{<<"key1">>,<<"value1">>},
%  {<<"key2">>,<<"value2">>},
%  {<<"key3">>,<<"value3">>}]

Destination = jsn:new({'key4', <<"value4">>}).
% [{<<"key4">>,<<"value4">>}]

% copy some of the paths from source to destination
[NewDestination] = jsn:copy(['key1', 'key2'], Source, Destination).
% [[{<<"key4">>,<<"value4">>},
%   {<<"key1">>,<<"value1">>},
%   {<<"key2">>,<<"value2">>}]]

T1 = fun(<<"value", N/binary>>) -> N end.

% transform all the keys of NewDestination
jsn:transform([{key1, T1},{key2, T1},{key4, T1}], NewDestination).
% [{<<"key4">>,<<"4">>},
%  {<<"key1">>,<<"1">>},
%  {<<"key2">>,<<"2">>}]
```

### `encode/1` and `decode/1,2` - Encoding/decoding JSON for interaction with jsn

jsn features convenience functions for encoding to and decoding from JSON.
These functions are thin wrappers around [jsonx][jsonx], which supports
decoding into all three jsn-supported formats (`proplist`, `eep18`, `struct`).

* `encode(Object)` - Encode the given object (in any supported format)
  as a binary string.
* `decode(Json)` - Decode the given binary string into a jsn object, in the
  default proplist format.
* `decode(Json, Options)` - Decode the given binary string into a json object,
  in the specified format.

Examples:

```erlang
Json = <<"{\"key\": {\"inner_key\":\"value\"}}">>.
% <<"{\"key\": {\"inner_key\":\"value\"}}">>

Proplist = jsn:decode(Json).
% [{<<"key">>,[{<<"inner_key">>,<<"value">>}]}]

Eep18 = jsn:decode(Json, [{format, eep18}]).
% {[{<<"key">>,{[{<<"inner_key">>,<<"value">>}]}}]}

Struct = jsonx:decode(Json, [{format, struct}]).
% {struct,[{<<"key">>,
%           {struct,[{<<"inner_key">>,<<"value">>}]}}]}
```

### `equal/3,4` - Path-wise object comparison

* `equal(Paths, OriginalObject, OtherObjectOrObjects)` - Given a list of paths,
  an Original object, and a single or list of objects, verify that each path
  in each of the other object(s) has the same value as the original does at the
  same path, for each path in the list of paths. If so, return ok; otherwise,
  return an error tuple with the error type and a summary of mismatches for the
  first mismatched object.
* `equal(Paths, OriginalObject, OtherObjectOrObjects, Mode)` - Identical to
  `equal/3`, but with Mode explicitly passed in. There are two modes available:
  * `hard` - paths that do not exist in the objects to be tested are treated as
    errors.  This is the default.
  * `soft`  - missing paths are ignored. Only values that exist are checked for
    equality.

Examples:

```erlang
Object1 = jsn:new([{<<"path1">>, <<"thing1">>},
                   {<<"path2">>, <<"thing2">>},
                   {<<"path3">>, <<"thing3">>}]).
% {[{<<"path1">>,<<"thing1">>},
%   {<<"path2">>,<<"thing2">>},
%   {<<"path3">>,<<"thing3">>}]}

Object2 = jsn:new([{<<"path1">>, <<"thing1">>},
                   {<<"path2">>, <<"notthing2">>}]).
% {[{<<"path1">>,<<"thing1">>},{<<"path2">>,<<"notthing2">>}]}

% by path1, these objects are equal
jsn:equal([<<"path1">>], Object1, Object2).
% ok

% by path2, not so much
jsn:equal([<<"path2">>], Object1, Object2).
% {error,{not_equal,<<"mismatch of requested and existing field(s): path2">>}}

% same story with path3
jsn:equal([<<"path3">>], Object1, Object2).
% {error,{not_equal,<<"mismatch of requested and existing field(s): path3">>}}

% but if you use soft mode, path 3 works (because it's missing in Object2)
jsn:equal([<<"path3">>], Object1, Object2, soft).
% ok

% you can also use a list of objects for the 3rd argument.
jsn:equal([<<"path1">>, <<"path3">>], Object1, [Object1, Object2], soft).
% ok
```

### `is_equal/2` and `is_subset/2` - Boolean object comparison

* `is_equal(A, B)` - Compare a pair of objects (in proplist, eep18, or struct
  format), returning `true` if all values and/or key-value pairs are matched
  across both objects.
* `is_subset(A, B)` - Compare a pair of objects or json terms (in proplist,
  eep18, or struct format), returning `true` if all values, key-value pairs,
  and array members in the first input are present in the second input.

Examples:

```erlang
Object1 = jsn:new([{<<"path1">>, <<"thing1">>},
                   {<<"path2">>, <<"thing2">>}]).
% [{<<"path1">>,<<"thing1">>},{<<"path2">>,<<"thing2">>}]

Object2 = jsn:new([{<<"path1">>, <<"thing1">>},
                   {<<"path2">>, <<"thing2">>}], [{format, struct}]).
% {struct,[{<<"path1">>,<<"thing1">>},
%          {<<"path2">>,<<"thing2">>}]}

Object3 = [{path1, <<"thing1">>},
           {path2, <<"thing2">>}].
% [{path1,<<"thing1">>},{path2,<<"thing2">>}]

jsn:is_equal(Object1, Object2).
% true

jsn:is_equal(Object1, Object3).
% true

Object4 = jsn:set(path1, Object1, 1).
% {[{<<"path1">>,1},{<<"path2">>,<<"thing2">>}]}

jsn:is_equal(Object1, Object4).
% false

Object5 = jsn:set_list([{path3, <<"thing3">>}], Object1).
% {[{<<"path3">>,<<"thing3">>},
%   {<<"path1">>,<<"thing1">>},
%   {<<"path2">>,<<"thing2">>}]}

Object6 = jsn:set_list([{path3, <<"thing3">>}], Object2).
% {struct,[{<<"path3">>,<<"thing3">>},
%          {<<"path1">>,<<"thing1">>},
%          {<<"path2">>,<<"thing2">>}]}

jsn:is_subset(Object1, jsn:new(Object1)).
% true

jsn:is_subset(Object1, Object2).
% true

jsn:is_subset(Object1, Object3).
% true

jsn:is_subset(Object3, Object1).
% false
```


[ej]: https://github.com/seth/ej
[kvc]: https://github.com/etrepum/kvc
[erlson]: https://github.com/alavrik/erlson
[jsx]: https://github.com/talentdeficit/jsx
[jsonx]: https://github.com/iskra/jsonx
[jiffy]: https://github.com/davisp/jiffy
[mochijson2]: https://github.com/mochi/mochiweb/blob/master/src/mochijson2.erl
