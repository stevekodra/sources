# Developer guide to the Declarative Pipeline plugins

## Docker tests on OS X

Docker for Mac out of the box will not be able to run the Docker tests due
to mount issues. To fix this, make sure the following are present in Docker
Preferences -> File Sharing: `/private`, `/tmp`, `/var/folders`.
All three must be there for the tests to pass.

If any of them are missing, do the following in Docker
Preferences -> File Sharing:

1. If `/private`, `/tmp`, or `/var/folders` are present, remove them.
2. Add `/tmp`
   * Click the "+" plus symbol below the list.
   * Navigate to `/tmp` and click "Open". This will add the path `/private/tmp` to the  list.
   * Double-click on `/private/tmp` and change it to just `/tmp`.
3. Add `/var/folders`
   * Click the "+" plus symbol below the list.
   * Navigate to `/var/folders` and click "Open". This will add the path `/private/var/folders` to the list.
   * Double-click on `/private/var/folders` and change it to just `/var/folders`.
4. Add `/private`
   * Click the "+" plus symbol below the list.
   * Navigate to `/private` and click "Open".

**IMPORTANT:** `/private` must be removed and then re-added after `/tmp` and `/var/folders`
have been added since they are both actually symlinks under `/private`.
However, once added, the items in the list will be sorted to appear in the order
`/private`, `/tmp`, `/var/folders`.

This has been tested with Docker for Mac 2017.03 and 2017.09, and should keep
working going forward. You can verify that Docker tests work by running
`AgentTest#agentDocker` in the `pipeline-model-definition` module.

## AST Parsing, Validation, and Transformation

### Overview

During the compilation of a Declarative `Jenkinsfile`, there are three key
things happening, beyond the normal Pipeline CPS transformation and sandbox
transformation. These are the Declarative-specific AST parsing, validation,
and AST transformation to the runtime model. This all occurs during the
`SEMANTIC_ANALYSIS` compile phase, before CPS or sandbox transformation
occurs.

### AST Parsing

In order to validate the syntax and contents of a Declarative Pipeline, as
well as to enable transformation between `Jenkinsfile` Groovy syntax and the
JSON syntax utilized by the Blue Ocean Pipeline Editor, the `pipeline {}`
block is parsed into an AST-driven model of the Declarative configuration.
This is done by `ModelParser`, which is added to the compilation process by
`GroovyShellDecoratorImpl`. The AST model classes are in the
`pipeline-model-api` plugin's
`org.jenkinsci.plugins.pipeline.modeldefinition.ast` package, all extending
from `ModelASTElement`. Every one of these model elements can translate to
either Groovy or JSON, via the `toGroovy()` and `toJSON()` methods.

### Validation

Upon successful parsing and population of a `ModelASTPipelineDef` instance,
representing the `pipeline {}` block's contents, validation is performed.
This is done via the `ModelASTElement.validate(ModelValidator)` method. Each
extension of `ModelASTElement` is responsible first for calling
`ModelValidator.validateElement(this)` to validate themselves, and then for
calling `ModelASTElement.validate(ModelValidator)` for any child
`ModelASTElement`s it may contain. This ensures that the entire model is
validated.

The `ModelValidator.validateElement` methods perform the actual validation
of each `ModelASTElement` subclass. `ModelValidatorImpl` contains the
implementations. Validation can be performed against any `ModelASTElement`,
regardless of whether it was generated by parsing a `Jenkinsfile` or a JSON
file. Any validation failures are stored in an `ErrorCollector`, resulting
in the compilation process failing and those errors showing up as normal
Groovy compilation errors, with (hopefully!) useful error messages and
pointers to the line and column where the error was found. For validation
from JSON, a JSON array of messages with pointers to the JSON element
causing the error is returned instead.

### Runtime Transformation

Assuming that a `Jenkinsfile` contains a `pipeline {}` block, is
successfully parsed into a `ModelASTPipelineDef` instance, and no errors
are found by the validator, the `ModelASTPipelineDef` is passed to an
instance of `RuntimeASTTransformer`, which will return an AST expression
for a closure that returns a fully instantiated `Root` instance, which is
what's used at runtime to determine what stages, agents, steps, etc are to
be run.

Prior to version `1.2` of Declarative Pipelines, the transformation from the
`pipeline {}` block to a `Root` instance was done by calling the
`pipeline {}` block's closure with a number of special delegate classes for
taking the "method calls" that are in the block and mapping them to the
fields of `Root`, `Agent`, `Stages`, etc in the
`org.jenkisnci.plugins.pipeline.modeldefinition.model` package in the
`pipeline-model-definition` plugin. However, that process was always a bit
complicated and become more and more arcane and difficult to follow as
various features and fixes were made, particularly for `environment`
variables and `when { expression { ... } }`, which eventually bypassed the
closure translation entirely and were generated directly from
`ModelASTEnvironment` and `ModelASTWhenExpression` instances, combined with
repeated loops of `script.evaluate(...)` calls at runtime. This was needed
to support environment variable cross-referencing and other expected
behaviors, but the result was fragile, hard to follow, and very hard to
safely modify.

Therefore, in `1.2`, this was changed to use `RuntimeASTTransformer`
instead. Rather than taking the closure passed to `pipeline` and
interpreting it during runtime execution, that closure is instead
transformed at parse time into a new AST expression that results in
returning a fully populated `Root` instance. This drastically decreased
overall code complexity, and actually improved the runtime of Declarative
Pipelines by a second or so.

#### How and Why the Runtime Transformation Works the Way It Does

##### `environment` resolution

In order to support various behaviors of Declarative Pipelines, most
notably environment variable cross-references, some parts of the runtime
model can't be evaluated until they're actually relevant during runtime. In
order to enable lazy evaluation of environment variable values that may
contain references to other not-yet-evaluated environment variables, the
values for `environment` declarations are not stored as actual value
expressions, but as closures, which, when called, fetch and call the
closures for any other `environment` variables they may contain. At runtime,
this is managed by the `Environment.EnvironmentResolver` class. An
`Environment` instance has two `Environment.EnvironmentResolver` fields, one
for normal `environment` declarations, and one for `credentials`
invocations.

`Environment.EnvironmentResolver` contains a map of `String`s for key names to
closures. Those closures, as mentioned above, will get the actual value for the
key, calling the closures for any values contained within them as well. There
has been consideration of caching the values to avoid repeated calls to the
closures, but as of now, the decision has been that there's more value in being
able to lazily re-evaluate a value than in saving the computation.

The closures are generated by taking the `Expression` for an `environment`
value and visiting its child `Expression`s recursively, until an `Expression`
without a child `Expression` is encountered. If that's a `VariableExpression`
pointing to one of the other `environment` keys, the `VariableExpression` is
replaced with a `MethodCallExpression` to fetch and call the appropriate
closure. Otherwise, the original `Expression` is returned.

##### Closures within elements

A number of model elements, such as `StepsBlock`, `post` conditions, etc,
themselves contain closures of steps to evaluate at runtime. Due to vagaries of
CPS transformation, passing a closure to a constructor results in the closure
being evaluated in a non-CPS context, so that the first nested closure is
evaluated/returned rather than the entire closure. Therefore, we set closures
on those elements via a static method on `Utils`, which instantiates the
element and sets the closure. This may be improved in later versions of
`workflow-cps` than the current `2.36.1` - this will need to be monitored.

##### Instantiation of `Root` via a closure

Probably for similar reasons, attempts to pass a constructor call for `Root`
directly to the `pipeline {}` in place of the original closure result in CPS
transformed closures ejecting before they should. The work around for that is
to transform the original closure to a new closure, which when called returns
the result of a constructor call for `Root`. So in `ModelInterpreter.call`, we
take a `CpsClosure`, and then `call()` it to get the `Root` object. Again, this
should be monitored going forward so that when the underlying issue with
closures in constructors is resolved, we can switch to a simpler call.