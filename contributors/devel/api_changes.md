*This document is oriented at developers who want to change existing APIs.
A set of API conventions, which applies to new APIs and to changes, can be
found at [API Conventions](api-conventions.md).

**Table of Contents**
<!-- BEGIN MUNGE: GENERATED_TOC -->

- [So you want to change the API?](#so-you-want-to-change-the-api)
  - [Operational overview](#operational-overview)
  - [On compatibility](#on-compatibility)
  - [Backward compatibility gotchas](#backward-compatibility-gotchas)
  - [Incompatible API changes](#incompatible-api-changes)
  - [Changing versioned APIs](#changing-versioned-apis)
    - [Edit types.go](#edit-typesgo)
    - [Edit defaults.go](#edit-defaultsgo)
    - [Edit conversion.go](#edit-conversiongo)
  - [Changing the internal structures](#changing-the-internal-structures)
    - [Edit types.go](#edit-typesgo-1)
  - [Edit validation.go](#edit-validationgo)
  - [Edit version conversions](#edit-version-conversions)
  - [Generate protobuf objects](#generate-protobuf-objects)
  - [Edit json (un)marshaling code](#edit-json-unmarshaling-code)
  - [Making a new API Version](#making-a-new-api-version)
  - [Making a new API Group](#making-a-new-api-group)
  - [Update the fuzzer](#update-the-fuzzer)
  - [Update the semantic comparisons](#update-the-semantic-comparisons)
  - [Implement your change](#implement-your-change)
  - [Write end-to-end tests](#write-end-to-end-tests)
  - [Examples and docs](#examples-and-docs)
  - [Alpha, Beta, and Stable Versions](#alpha-beta-and-stable-versions)
    - [Adding Unstable Features to Stable Versions](#adding-unstable-features-to-stable-versions)

<!-- END MUNGE: GENERATED_TOC -->

# So you want to change the API?

Before attempting a change to the API, you should familiarize yourself with a
number of existing API types and with the [API conventions](api-conventions.md).
If creating a new API type/resource, we also recommend that you first send a PR
containing just a proposal for the new API types, and that you initially target
the extensions API (pkg/apis/extensions).

The Kubernetes API has two major components - the internal structures and
the versioned APIs. The versioned APIs are intended to be stable, while the
internal structures are implemented to best reflect the needs of the Kubernetes
code itself.

What this means for API changes is that you have to be somewhat thoughtful in
how you approach changes, and that you have to touch a number of pieces to make
a complete change.  This document aims to guide you through the process, though
not all API changes will need all of these steps.

## Operational overview

It is important to have a high level understanding of the API system used in
Kubernetes in order to navigate the rest of this document.

As mentioned above, the internal representation of an API object is decoupled
from any one API version. This provides a lot of freedom to evolve the code,
but it requires robust infrastructure to convert between representations. There
are multiple steps in processing an API operation - even something as simple as
a GET involves a great deal of machinery.

The conversion process is logically a "star" with the internal form at the
center. Every versioned API can be converted to the internal form (and
vice-versa), but versioned APIs do not convert to other versioned APIs directly.
This sounds like a heavy process, but in reality we do not intend to keep more
than a small number of versions alive at once. While all of the Kubernetes code
operates on the internal structures, they are always converted to a versioned
form before being written to storage (disk or etcd) or being sent over a wire.
Clients should consume and operate on the versioned APIs exclusively.

To demonstrate the general process, here is a (hypothetical) example:

   1. A user POSTs a `Pod` object to `/api/v7beta1/...`
   2. The JSON is unmarshalled into a `v7beta1.Pod` structure
   3. Default values are applied to the `v7beta1.Pod`
   4. The `v7beta1.Pod` is converted to an `api.Pod` structure
   5. The `api.Pod` is validated, and any errors are returned to the user
   6. The `api.Pod` is converted to a `v6.Pod` (because v6 is the latest stable
version)
   7. The `v6.Pod` is marshalled into JSON and written to etcd

Now that we have the `Pod` object stored, a user can GET that object in any
supported api version. For example:

   1. A user GETs the `Pod` from `/api/v5/...`
   2. The JSON is read from etcd and unmarshalled into a `v6.Pod` structure
   3. Default values are applied to the `v6.Pod`
   4. The `v6.Pod` is converted to an `api.Pod` structure
   5. The `api.Pod` is converted to a `v5.Pod` structure
   6. The `v5.Pod` is marshalled into JSON and sent to the user

The implication of this process is that API changes must be done carefully and
backward-compatibly.

## On compatibility

Before talking about how to make API changes, it is worthwhile to clarify what
we mean by API compatibility.  Kubernetes considers forwards and backwards
compatibility of its APIs a top priority.

An API change is considered forward and backward-compatible if it:

   * adds new functionality that is not required for correct behavior (e.g.,
does not add a new required field)
   * does not change existing semantics, including:
     * default values *and behavior*
     * interpretation of existing API types, fields, and values
     * which fields are required and which are not
     * mutable fields do not become immutable
     * valid values do not become invalid

Put another way:

1. Any API call (e.g. a structure POSTed to a REST endpoint) that worked before
your change must work the same after your change.
2. Any API call that uses your change must not cause problems (e.g. crash or
degrade behavior) when issued against servers that do not include your change.
3. It must be possible to round-trip your change (convert to different API
versions and back) with no loss of information.
4. Existing clients need not be aware of your change in order for them to
continue to function as they did previously, even when your change is utilized.

If your change does not meet these criteria, it is not considered strictly
compatible, and may break older clients, or result in newer clients causing
undefined behavior.

Let's consider some examples. In a hypothetical API (assume we're at version
v6), the `Frobber` struct looks something like this:

```go
// API v6.
type Frobber struct {
  Height int    `json:"height"`
  Param  string `json:"param"`
}
```

You want to add a new `Width` field. It is generally safe to add new fields
without changing the API version, so you can simply change it to:

```go
// Still API v6.
type Frobber struct {
  Height int    `json:"height"`
  Width  int    `json:"width"`
  Param  string `json:"param"`
}
```

The onus is on you to define a sane default value for `Width` such that rule #1
above is true - API calls and stored objects that used to work must continue to
work.

For your next change you want to allow multiple `Param` values. You can not
simply change `Param string` to `Params []string` (without creating a whole new
API version) - that fails rules #1 and #2. You can instead do something like:

```go
// Still API v6, but kind of clumsy.
type Frobber struct {
  Height int           `json:"height"`
  Width  int           `json:"width"`
  Param  string        `json:"param"`  // the first param
  ExtraParams []string `json:"extraParams"` // additional params
}
```

Now you can satisfy the rules: API calls that provide the old style `Param`
will still work, while servers that don't understand `ExtraParams` can ignore
it. This is somewhat unsatisfying as an API, but it is strictly compatible.

Part of the reason for versioning APIs and for using internal structs that are
distinct from any one version is to handle growth like this. The internal
representation can be implemented as:

```go
// Internal, soon to be v7beta1.
type Frobber struct {
  Height int
  Width  int
  Params []string
}
```

The code that converts to/from versioned APIs can decode this into the somewhat
uglier (but compatible!) structures. Eventually, a new API version, let's call
it v7beta1, will be forked and it can use the clean internal structure.

We've seen how to satisfy rules #1 and #2. Rule #3 means that you can not
extend one versioned API without also extending the others. For example, an
API call might POST an object in API v7beta1 format, which uses the cleaner
`Params` field, but the API server might store that object in trusty old v6
form (since v7beta1 is "beta"). When the user reads the object back in the
v7beta1 API it would be unacceptable to have lost all but `Params[0]`. This
means that, even though it is ugly, a compatible change must be made to the v6
API.

However, this is very challenging to do correctly. It often requires multiple
representations of the same information in the same API resource, which need to
be kept in sync in the event that either is changed. For example, let's say you
decide to rename a field within the same API version. In this case, you add
units to `height` and `width`. You implement this by adding duplicate fields:

```go
type Frobber struct {
  Height         *int          `json:"height"`
  Width          *int          `json:"width"`
  HeightInInches *int          `json:"heightInInches"`
  WidthInInches  *int          `json:"widthInInches"`
}
```

You convert all of the fields to pointers in order to distinguish between unset
and set to 0, and then set each corresponding field from the other in the
defaulting pass (e.g., `heightInInches` from `height`, and vice versa), which
runs just prior to conversion. That works fine when the user creates a resource
from a hand-written configuration -- clients can write either field and read
either field, but what about creation or update from the output of GET, or
update via PATCH (see
[In-place updates](https://kubernetes.io/docs/user-guide/managing-deployments/#in-place-updates-of-resources))?
In this case, the two fields will conflict, because only one field would be
updated in the case of an old client that was only aware of the old field (e.g.,
`height`).

Say the client creates:

```json
{
  "height": 10,
  "width": 5
}
```

and GETs:

```json
{
  "height": 10,
  "heightInInches": 10,
  "width": 5,
  "widthInInches": 5
}
```

then PUTs back:

```json
{
  "height": 13,
  "heightInInches": 10,
  "width": 5,
  "widthInInches": 5
}
```

The update should not fail, because it would have worked before `heightInInches`
was added.

## Backward compatibility gotchas

* A single feature/property cannot be represented using multiple spec fields in the same API version
  simultaneously, as the example above shows. Only one field can be populated in any resource at a time, and the client
  needs to be able to specify which field they expect to use (typically via API version),
  on both mutation and read. Old clients must continue to function properly while only manipulating
  the old field. New clients must be able to function properly while only manipulating the new
  field.

* A new representation, even in a new API version, that is more expressive than an
  old one breaks backward compatibility, since clients that only understood the
  old representation would not be aware of the new representation nor its
  semantics. Examples of proposals that have run into this challenge include
  [generalized label selectors](http://issues.k8s.io/341) and [pod-level security context](http://prs.k8s.io/12823).

* Enumerated values cause similar challenges. Adding a new value to an enumerated set
  is *not* a compatible change. Clients which assume they know how to handle all possible
  values of a given field will not be able to handle the new values. However, removing a
  value from an enumerated set *can* be a compatible change, if handled properly (treat the
  removed value as deprecated but allowed). For enumeration-like fields that expect to add
  new values in the future, such as `reason` fields, please document that expectation clearly
  in the API field descriptions. Clients should treat such sets of values as potentially
  open-ended.

* For [Unions](api-conventions.md#unions), sets of fields where at most one should
  be set, it is acceptable to add a new option to the union if the [appropriate
  conventions](api-conventions.md#objects) were followed in the original object.
  Removing an option requires following the [deprecation process](https://kubernetes.io/docs/deprecation-policy/).
  
* Changing any validation rules always has the potential of breaking some client, since it changes the
  assumptions about part of the API, similar to adding new enum values. Validation rules on spec fields can
  neither be relaxed nor strengthened. Strengthening cannot be permitted because any requests that previously
  worked must continue to work. Weakening validation has the potential to break other consumers and generators
  of the API resource. Status fields whose writers are under our control (e.g., written by non-pluggable
  controllers), may potentially tighten validation, since that would cause a subset of previously valid
  values to be observable by clients.
  
* Do not add a new API version of an existing resource and make it the preferred version in the same
  release, and do not make it the storage version. The latter is necessary so that a rollback of the
  apiserver doesn't render resources in etcd undecodable after rollback.

## Incompatible API changes

There are times when this might be OK, but mostly we want changes that meet this
definition. If you think you need to break compatibility, you should talk to the
Kubernetes team first.

Breaking compatibility of a beta or stable API version, such as v1, is
unacceptable. Compatibility for experimental or alpha APIs is not strictly
required, but breaking compatibility should not be done lightly, as it disrupts
all users of the feature. Experimental APIs may be removed. Alpha and beta API
versions may be deprecated and eventually removed wholesale, as described in the
[versioning document](../design-proposals/versioning.md).

If your change is going to be backward incompatible or might be a breaking
change for API consumers, please send an announcement to
`kubernetes-dev@googlegroups.com` before the change gets in. If you are unsure,
ask. Also make sure that the change gets documented in the release notes for the
next release by labeling the PR with the "release-note-action-required" github label.

If you found that your change accidentally broke clients, it should be reverted.

In short, the expected API evolution is as follows:

* `extensions/v1alpha1` ->
* `newapigroup/v1alpha1` -> ... -> `newapigroup/v1alphaN` ->
* `newapigroup/v1beta1` -> ... -> `newapigroup/v1betaN` ->
* `newapigroup/v1` ->
* `newapigroup/v2alpha1` -> ...

While in extensions we have no obligation to move forward with the API at all
and may delete or break it at any time.

While in alpha we expect to move forward with it, but may break it.

Once in beta we will preserve forward compatibility, but may introduce new
versions and delete old ones.

v1 must be backward-compatible for an extended length of time.

## Changing versioned APIs

For most changes, you will probably find it easiest to change the versioned
APIs first. This forces you to think about how to make your change in a
compatible way. Rather than doing each step in every version, it's usually
easier to do each versioned API one at a time, or to do all of one version
before starting "all the rest".

### Edit types.go

The struct definitions for each API are in
`staging/src/k8s.io/api/<group>/<version>/types.go`. Edit those files to reflect
the change you want to make. Note that all types and non-inline fields in
versioned APIs must be preceded by descriptive comments - these are used to
generate documentation. Comments for types should not contain the type name; API
documentation is generated from these comments and end-users should not be
exposed to golang type names.

For types that need the generated
[DeepCopyObject](https://github.com/kubernetes/kubernetes/commit/8dd0989b395b29b872e1f5e06934721863e4a210#diff-6318847735efb6fae447e7dbf198c8b2R3767)
methods, usually only required by the top-level types like `Pod`, add this line
to the comment
([example](https://github.com/kubernetes/kubernetes/commit/39d95b9b065fffebe5b6f233d978fe1723722085#diff-ab819c2e7a94a3521aecf6b477f9b2a7R30)): 

```golang 
  // +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
```

Optional fields should have the `status,omitempty` json tag; fields are interpreted as
being required otherwise.

### Edit defaults.go

If your change includes new fields for which you will need default values, you
need to add cases to `pkg/apis/<group>/<version>/defaults.go` (the core v1 API
is special, its defaults.go is at `pkg/api/v1/defaults.go`. For simplicity, we
will not mention this special case in the rest of the article). Of course, since
you have added code, you have to add a test:
`pkg/apis/<group>/<version>/defaults_test.go`.

Do use pointers to scalars when you need to distinguish between an unset value
and an automatic zero value.  For example,
`PodSpec.TerminationGracePeriodSeconds` is defined as `*int64` the go type
definition.  A zero value means 0 seconds, and a nil value asks the system to
pick a default.

Don't forget to run the tests!

### Edit conversion.go

Given that you have not yet changed the internal structs, this might feel
premature, and that's because it is. You don't yet have anything to convert to
or from. We will revisit this in the "internal" section. If you're doing this
all in a different order (i.e. you started with the internal structs), then you
should jump to that topic below. In the very rare case that you are making an
incompatible change you might or might not want to do this now, but you will
have to do more later. The files you want are
`pkg/apis/<group>/<version>/conversion.go` and
`pkg/apis/<group>/<version>/conversion_test.go`.

Note that the conversion machinery doesn't generically handle conversion of
values, such as various kinds of field references and API constants. [The client
library](https://github.com/kubernetes/client-go/blob/v4.0.0-beta.0/rest/request.go#L352)
has custom conversion code for field references. You also need to add a call to
`AddFieldLabelConversionFunc` of your scheme with a mapping function that
understands supported translations, like this
[line](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/pkg/api/v1/conversion.go#L165).

## Changing the internal structures

Now it is time to change the internal structs so your versioned changes can be
used.

### Edit types.go

Similar to the versioned APIs, the definitions for the internal structs are in
`pkg/apis/<group>/types.go`. Edit those files to reflect the change you want to
make. Keep in mind that the internal structs must be able to express *all* of
the versioned APIs.

Similar to the versioned APIs, you need to add the `+k8s:deepcopy-gen` tag to
types that need generated DeepCopyObject methods.

## Edit validation.go

Most changes made to the internal structs need some form of input validation.
Validation is currently done on internal objects in
`pkg/apis/<group>/validation/validation.go`. This validation is the one of the
first opportunities we have to make a great user experience - good error
messages and thorough validation help ensure that users are giving you what you
expect and, when they don't, that they know why and how to fix it. Think hard
about the contents of `string` fields, the bounds of `int` fields and the
optionality of fields.

Of course, code needs tests - `pkg/apis/<group>/validation/validation_test.go`.

## Edit version conversions

At this point you have both the versioned API changes and the internal
structure changes done.  If there are any notable differences - field names,
types, structural change in particular - you must add some logic to convert
versioned APIs to and from the internal representation.  If you see errors from
the `serialization_test`, it may indicate the need for explicit conversions.

Performance of conversions very heavily influence performance of apiserver.
Thus, we are auto-generating conversion functions that are much more efficient
than the generic ones (which are based on reflections and thus are highly
inefficient).

The conversion code resides with each versioned API. There are two files:

   - `pkg/apis/<group>/<version>/conversion.go` containing manually written
     conversion functions
   - `pkg/apis/<group>/<version>/zz_generated.conversion.go` containing
     auto-generated conversion functions

Since auto-generated conversion functions are using manually written ones,
those manually written should be named with a defined convention, i.e. a
function converting type X in pkg a to type Y in pkg b, should be named:
`convert_a_X_To_b_Y`.

Also note that you can (and for efficiency reasons should) use auto-generated
conversion functions when writing your conversion functions.

Adding manually written conversion also requires you to add tests to
`pkg/apis/<group>/<version>/conversion_test.go`.

Once all the necessary manually written conversions are added, you need to
regenerate auto-generated ones. To regenerate them run:

```sh
make clean && make generated_files
```

`make clean` is important, otherwise the generated files might be stale, because
the build system uses custom cache.

`make all` will invoke `make generated_files` as well.

The `make generated_files` will also regenerate the `zz_generated.deepcopy.go`,
`zz_generated.defaults.go`, and `api/openapi-spec/openapi-spec`.

If regeneration is somehow not possible due to compile errors, the easiest
workaround is to remove the files causing errors and rerun the command.

## Generate Code 

Apart from the `defaulter-gen`, `deepcopy-gen`, `conversion-gen` and
`openapi-gen`, there are a few other generators:
 - `go-to-protobuf`
 - `client-gen`
 - `lister-gen`
 - `informer-gen`
 - `codecgen` (for fast json serialization with ugorji codec)

Many of the generators are based on
[`gengo`](https://github.com/kubernetes/gengo) and share common
flags. The `--verify-only` flag will check the existing files on disk
and fail if they are not what would have been generated.

The generators that create go code have a `--go-header-file` flag
which should be a file that contains the header that should be
included. This header is the copyright that should be present at the
top of the generated file and should be checked with the
[`repo-infra/verify/verify-boilerplane.sh`](https://github.com/kubernetes/repo-infra/blob/master/verify/verify-boilerplate.sh)
script at a later stage of the build.

To invoke these generators, you can run `make generated`, which runs a bunch of
[scripts](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/hack/update-all.sh#L63-L78).
Please continue to read the next a few sections, because some generators have
prerequisites, also because they introduce how to invoke the generators
individually if you find `make update` takes too long to run.

### Generate protobuf objects

For any core API object, we also need to generate the Protobuf IDL and marshallers.
That generation is invoked with

```sh
hack/update-generated-protobuf.sh
```

The vast majority of objects will not need any consideration when converting
to protobuf, but be aware that if you depend on a Golang type in the standard
library there may be additional work required, although in practice we typically
use our own equivalents for JSON serialization. The `pkg/api/serialization_test.go`
will verify that your protobuf serialization preserves all fields - be sure to
run it several times to ensure there are no incompletely calculated fields.

### Generate Clientset

`client-gen` is a tool to generate clientsets for top-level API objects.

`client-gen` requires the `// +genclient=true` annotation on each
exported type in both the internal `pkg/apis/<group>/types.go` as well as each
specifically versioned `staging/src/k8s.io/api/<group>/<version>/types.go`.

If the apiserver hosts your API under a different group name than the `<group>`
in the filesystem, (usually this is because the `<group>` in the filesystem
omits the "k8s.io" suffix, e.g., admission vs. admission.k8s.io), you can
instruct the `client-gen` to use the correct group name by adding the `//
+groupName=` annotation in the `doc.go` in both the internal
`pkg/apis/<group>/doc.go` as well as in each specifically versioned
`staging/src/k8s.io/api/<group>/<version>/types.go`.

Once you added the annotations, generate the client with

```sh
hack/update-codegen.sh
```

client-gen is flexible. See [this document](generating-clientset.md) if you need
client-gen for non-kubernetes API.

### Generate Listers

`lister-gen` is a tool to generate listers for a client. It reuses the
`//+genclient=true` and the `// +groupName=` annotations, so you do not need to
specify extra annotations.

Your previous run of `hack/update-codegen.sh` has invoked `lister-gen`.

### Generate Informers

`informer-gen` generates the very useful Informers which watch API
resources for changes. It reuses the `//+genclient=true` and the
`//+groupName=` annotations, so you do not need to specify extra annotations.

Your previous run of `hack/update-codegen.sh` has invoked `informer-gen`.

### Edit json (un)marshaling code

We are auto-generating code for marshaling and unmarshaling json representation
of api objects - this is to improve the overall system performance.

The auto-generated code resides with each versioned API:

   - `staging/src/k8s.io/api/<group>/<version>/types.generated.go`

To regenerate them run:

```sh
hack/update-codecgen.sh
```

## Making a new API Version

This section is under construction, as we make the tooling completely generic.

If you are adding a new API version to an existing group, you can copy the
structure of the existing `pkg/apis/<group>/<existing-version>` and
`staging/src/k8s.io/api/<group>/<existing-version>` directories.

Due to the fast changing nature of the project, the following content is probably out-dated:
* You can control if the version is enabled by default by update
[pkg/master/master.go](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/pkg/master/master.go#L381).
* You must add the new version to
  [pkg/apis/group_name/install/install.go](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/pkg/apis/apps/install/install.go).
* You must add the new version to
  [hack/lib/init.sh#KUBE_AVAILABLE_GROUP_VERSIONS](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/hack/lib/init.sh#L53).
* You must add the new version  to
  [staging/src/k8s.io/kube-gen/cmd/go-to-protobuf/protobuf/cmd.go](https://github.com/kubernetes/kubernetes/blob/2e6be8583d00916f1896d2b53e550162f1558ccf/staging/src/k8s.io/kube-gen/cmd/go-to-protobuf/protobuf/cmd.go#L64)
  to generate protobuf IDL and marshallers.
* You must add the new version  to
  [cmd/kube-apiserver/app#apiVersionPriorities](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/cmd/kube-apiserver/app/aggregator.go#L172)
  to let the aggregator list it. This list will be removed before release 1.8.
* You must setup storage for the new version in
  [pkg/registry/group_name/rest](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/pkg/registry/authentication/rest/storage_authentication.go)

You need to regenerate the generated code as instructed in the sections above.

## Making a new API Group

You'll have to make a new directory under `pkg/apis/` and
`staging/src/k8s.io/api`; copy the directory structure of an existing API group,
e.g. `pkg/apis/authentication` and `staging/src/k8s.io/api/authentication`;
replace "authentication" with your group name and replace versions with your
versions; replace the API kinds in
[versioned](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/staging/src/k8s.io/api/authentication/v1/register.go#L47)
and
[internal](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/pkg/apis/authentication/register.go#L47)
register.go, and
[install.go](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/pkg/apis/authentication/install/install.go#L43)
with your kinds.

You'll have to add your API group/version to a few places in the code base, as
noted in [Making a new API Version](#making-a-new-api-version) section.

You need to regenerate the generated code as instructed in the sections above.

## Update the fuzzer

Part of our testing regimen for APIs is to "fuzz" (fill with random values) API
objects and then convert them to and from the different API versions. This is
a great way of exposing places where you lost information or made bad
assumptions. If you have added any fields which need very careful formatting
(the test does not run validation) or if you have made assumptions such as
"this slice will always have at least 1 element", you may get an error or even
a panic from the `serialization_test`. If so, look at the diff it produces (or
the backtrace in case of a panic) and figure out what you forgot. Encode that
into the fuzzer's custom fuzz functions. Hint: if you added defaults for a
field, that field will need to have a custom fuzz function that ensures that the
field is fuzzed to a non-empty value.

The fuzzer can be found in `pkg/api/testing/fuzzer.go`.

## Update the semantic comparisons

VERY VERY rarely is this needed, but when it hits, it hurts. In some rare cases
we end up with objects (e.g. resource quantities) that have morally equivalent
values with different bitwise representations (e.g. value 10 with a base-2
formatter is the same as value 0 with a base-10 formatter). The only way Go
knows how to do deep-equality is through field-by-field bitwise comparisons.
This is a problem for us.

The first thing you should do is try not to do that. If you really can't avoid
this, I'd like to introduce you to our `apiequality.Semantic.DeepEqual` routine.
It supports custom overrides for specific types - you can find that in
`pkg/api/helper/helpers.go`.

There's one other time when you might have to touch this: `unexported fields`.
You see, while Go's `reflect` package is allowed to touch `unexported fields`,
us mere mortals are not - this includes `apiequality.Semantic.DeepEqual`.
Fortunately, most of our API objects are "dumb structs" all the way down - all
fields are exported (start with a capital letter) and there are no unexported
fields. But sometimes you want to include an object in our API that does have
unexported fields somewhere in it (for example, `time.Time` has unexported fields).
If this hits you, you may have to touch the `apiequality.Semantic.DeepEqual`
customization functions.

## Implement your change

Now you have the API all changed - go implement whatever it is that you're
doing!

## Write end-to-end tests

Check out the [E2E docs](e2e-tests.md) for detailed information about how to
write end-to-end tests for your feature.

## Examples and docs

At last, your change is done, all unit tests pass, e2e passes, you're done,
right? Actually, no. You just changed the API. If you are touching an existing
facet of the API, you have to try *really* hard to make sure that *all* the
examples and docs are updated. There's no easy way to do this, due in part to
JSON and YAML silently dropping unknown fields. You're clever - you'll figure it
out. Put `grep` or `ack` to good use.

If you added functionality, you should consider documenting it and/or writing
an example to illustrate your change.

Make sure you update the swagger and OpenAPI spec by running:

```sh
hack/update-swagger-spec.sh
hack/update-openapi-spec.sh
```

The API spec changes should be in a commit separate from your other changes.

## Alpha, Beta, and Stable Versions

New feature development proceeds through a series of stages of increasing
maturity:

- Development level
  - Object Versioning: no convention
  - Availability: not committed to main kubernetes repo, and thus not available
in official releases
  - Audience: other developers closely collaborating on a feature or
proof-of-concept
  - Upgradeability, Reliability, Completeness, and Support: no requirements or
guarantees
- Alpha level
  - Object Versioning: API version name contains `alpha` (e.g. `v1alpha1`)
  - Availability: committed to main kubernetes repo;  appears in an official
release; feature is disabled by default, but may be enabled by flag
  - Audience: developers and expert users interested in giving early feedback on
features
  - Completeness: some API operations, CLI commands, or UI support may not be
implemented; the API need not have had an *API review* (an intensive and
targeted review of the API, on top of a normal code review)
  - Upgradeability: the object schema and semantics may change in a later
software release, without any provision for preserving objects in an existing
cluster; removing the upgradability concern allows developers to make rapid
progress; in particular, API versions can increment faster than the minor
release cadence and the developer need not maintain multiple versions;
developers should still increment the API version when object schema or
semantics change in an [incompatible way](#on-compatibility)
  - Cluster Reliability: because the feature is relatively new, and may lack
complete end-to-end tests, enabling the feature via a flag might expose bugs
with destabilize the cluster (e.g. a bug in a control loop might rapidly create
excessive numbers of object, exhausting API storage).
  - Support: there is *no commitment* from the project to complete the feature;
the feature may be dropped entirely in a later software release
  - Recommended Use Cases: only in short-lived testing clusters, due to
complexity of upgradeability and lack of long-term support and lack of
upgradability.
- Beta level:
  - Object Versioning: API version name contains `beta` (e.g. `v2beta3`)
  - Availability: in official Kubernetes releases, and enabled by default
  - Audience: users interested in providing feedback on features
  - Completeness: all API operations, CLI commands, and UI support should be
implemented; end-to-end tests complete; the API has had a thorough API review
and is thought to be complete, though use during beta may frequently turn up API
issues not thought of during review
  - Upgradeability: the object schema and semantics may change in a later
software release; when this happens, an upgrade path will be documented; in some
cases, objects will be automatically converted to the new version; in other
cases, a manual upgrade may be necessary; a manual upgrade may require downtime
for anything relying on the new feature, and may require manual conversion of
objects to the new version; when manual conversion is necessary, the project
will provide documentation on the process (for an example, see [v1 conversion
tips](../api.md#v1-conversion-tips))
  - Cluster Reliability: since the feature has e2e tests, enabling the feature
via a flag should not create new bugs in unrelated features; because the feature
is new, it may have minor bugs
  - Support: the project commits to complete the feature, in some form, in a
subsequent Stable version; typically this will happen within 3 months, but
sometimes longer; releases should simultaneously support two consecutive
versions (e.g. `v1beta1` and `v1beta2`; or `v1beta2` and `v1`) for at least one
minor release cycle (typically 3 months) so that users have enough time to
upgrade and migrate objects
  - Recommended Use Cases: in short-lived testing clusters; in production
clusters as part of a short-lived evaluation of the feature in order to provide
feedback
- Stable level:
  - Object Versioning: API version `vX` where `X` is an integer (e.g. `v1`)
  - Availability: in official Kubernetes releases, and enabled by default
  - Audience: all users
  - Completeness: same as beta
  - Upgradeability: only [strictly compatible](#on-compatibility) changes
allowed in subsequent software releases
  - Cluster Reliability: high
  - Support: API version will continue to be present for many subsequent
software releases;
  - Recommended Use Cases: any

### Adding Unstable Features to Stable Versions

When adding a feature to an object which is already Stable, the new fields and
new behaviors need to meet the Stable level requirements. If these cannot be
met, then the new field cannot be added to the object.

For example, consider the following object:

```go
// API v6.
type Frobber struct {
  Height int    `json:"height"`
  Param  string `json:"param"`
}
```

A developer is considering adding a new `Width` parameter, like this:

```go
// API v6.
type Frobber struct {
  Height int    `json:"height"`
  Width  int    `json:"height"`
  Param  string `json:"param"`
}
```

However, the new feature is not stable enough to be used in a stable version
(`v6`). Some reasons for this might include:

- the final representation is undecided (e.g. should it be called `Width` or
`Breadth`?)
- the implementation is not stable enough for general use (e.g. the `Area()`
routine sometimes overflows.)

The developer cannot add the new field until stability is met. However,
sometimes stability cannot be met until some users try the new feature, and some
users are only able or willing to accept a released version of Kubernetes. In
that case, the developer has a few options, both of which require staging work
over several releases.


A preferred option is to first make a release where the new value (`Width` in
this example) is specified via an annotation, like this:

```go
kind: frobber
version: v6
metadata:
  name: myfrobber
  annotations:
    frobbing.alpha.kubernetes.io/width: 2
height: 4
param: "green and blue"
```

This format allows users to specify the new field, but makes it clear that they
are using a Alpha feature when they do, since the word `alpha` is in the
annotation key.

Another option is to introduce a new type with an new `alpha` or `beta` version
designator, like this:

```
// API v6alpha2
type Frobber struct {
  Height int    `json:"height"`
  Width  int    `json:"height"`
  Param  string `json:"param"`
}
```

The latter requires that all objects in the same API group as `Frobber` to be
replicated in the new version, `v6alpha2`. This also requires user to use a new
client which uses the other version. Therefore, this is not a preferred option.

A related issue is how a cluster manager can roll back from a new version
with a new feature, that is already being used by users. See
https://github.com/kubernetes/kubernetes/issues/4855.

<!-- BEGIN MUNGE: GENERATED_ANALYTICS -->
[![Analytics](https://kubernetes-site.appspot.com/UA-36037335-10/GitHub/docs/devel/api_changes.md?pixel)]()
<!-- END MUNGE: GENERATED_ANALYTICS -->
