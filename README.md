# go-archetype

go-archetype is a tool for creating project archetypes.

If you or your company create new projects and you want all these projects to preserve a similar structure or simply create a good starting point for other develoeprs, use go-archetype to define project archtypes (templates).

## Concept

Two roles are involved:

* Archetype creator (typically a senior developer/architect)
* Developer creating a new project

### Archetype creator

As the archetype creator you create a template (aka blueprint) project. This project is fully functional, valid code written in Golang, JavaScript, Java, Python etc. (go-archetype is agnostic)
And then you define a set of transformaitons. E.g. ask the user for the project name and replace here, here and here.

### Developer using the archetype

As a developer creating a new project based on the archetype, you run `go-archetype` with a few parameters, you answer a few questions (such as what if you new project's name, these questions are defined by the archetype creator) and your new code is generated.

### Summary

Here are the steps:

1. Architect creates an actual working project. This project is used as a blueprint or an archetype. It's a barebones project, no templating involved.
1. Architect defines a set of transformations in transformations.yml. These transformations are used for simple search-and-replace in the blueprint. (and there are more compicated types, see below)
1. Developer runs go-archetype to create a new project based on the blueprint.

## Usage

    go-archetype --transformaitons=transformations.yml --source=. --destination=/path/to/your/new/project

## The transformaitons.yml file

The transformaitons file contains two main sections:

* inputs
* transformations

The *inputs* section defines user inputs, see description below.
The *transformations* file defines the list of ordered transformations. also see below.

See the file transformations.yml in this very project as an example.

## Requesting for user input

It is common to request user inputs in order to apply a set of transformations. For example you might want to request for the project name, description, wheather to include a readme file etc.

There are two type of inputs: `text` and `yesno`.  Text provide simple, single-line text inputs. While yesno provides for a boolean [y/N] question.

Example:

```yml
inputs:
  - id: ProjectName
    text: What is the project name?
    type: text
  - id: IncludeReadme
    text: Would you like to include the readme file?
    type: yesno
```

The `id` must be unique and is later also use for performing the transformations (see below).

A user may provide the required inputs interactively when promped to when running go-archetype. The user may also provide the input as CLI arguments, which is useful for automation.

Interactive question example: (whent he user sees when runnin go-archetype)

    ? Would you like to include the readme file? (y/N)

Providing the input as CLI args:

```sh
go-archetype transform --transformations=transformations.yml \
    --source=. \
    --destination=.tmp/go/my-go-project \
    -- \
    --ProjectName my-go-project
    --IncludeReadme yes
```

To seperate program args from user input we use `--`. After the `--` the list of user inputs is provided.

## Templating user input

After accepting user input you might want to transform and tempalte it.

We use go templates. For full overview of the templating language see [text/template](https://golang.org/pkg/text/template/)
Following is a list of several examples to get you started.

* User arewrapped in `{{` and `}}`
* User inputs are prepended with `.`, so for example `{{.ProjectName}}`
* You may combine user input with constats, for example `{{.ProjectName}} - {{.ProjectDescription}}

Pipelines and functions are a useful concept. For example you might request the project's name and then display this project name
in uppercase in the readme file and lowercase in source files. To that end we provide a host of **template pipelines**.

    {{ .ProjectName | upper }} # ProjectName is provided by the user and we transform it to all uppercase. with the upper pipeline
    {{ wrap 80 .Description }} # Wordwrap the project description by 80 characters

We include out the the box the [sprig](http://masterminds.github.io/sprig/strings.html) library, which includes many differnet string manipulation functions, to list a few: `trim`, `trimAll`, `trimSuffix`, `trimPrefix`, `upper`, `lower`, `title`, `wrap`, `plural`, `snakecase`, `camelcase`, `kebabcase` etc.

## Transformers

There are two types of transformers: *include* and *replace*.

### The *include* transformer

  TODO: high level

All files that pass the global ignore filter are included by default. Unless one or more of the *include* rules with an empty `region_marker` applies to them, in which case the files are included only if the condition evaluates to true.

Condition is a go template condition as can be used in an `{{if}}` expression. As such a simple boolean is expressed as simply `.var` where `var` is typically a user input.

For example with the following user input:

```yml
inputs:
  - id: IncludeReadme
    text: Would you like to include the readme file?
    type: yesno
```

It is possbible to define the following include transformer:

```yml
transformations:
  - name: include the readme file
    type: include
    region_marker: # When there's no marker, the entire file(s) is included
    condition: .IncludeReadme
    files: ["README.md"]
```

Note that when there's no `region_marker` that simply means that the entire file is included/excluded based on the user's input.

More sophisticated expresions could also be utilized, such as boolean algebra, using `and .x .y`, `or .x .y` etc (`x` and `y` are user inputs). For a complete reference, see [go templates](https://golang.org/pkg/text/template/).

#### Don't forget the dot.

Keep in mind that go templates require a dot (`.`) to prepend a value. So when utilizing user input, for example such as `IncludeReadme` be sure to prepend the dot, e.g. `.IncludeReadme` whenever used in conditions or replacements.

One caveat to that is that for simplicity go-archetype allows dot-less conditions when they are very simple, e.g. only `variable`, such as `IncludeReadme`. So the following conditions are actually equivalent:

```
condition: .IncludeReadme
```

and

```
condition: IncludeReadme # No dot here
```

This is done in order to simplify the simple single-operand conditions. However, with more complex conditions be sure to prepend the dot.
For example the following condition is valid where x and y are user inputs:

```
condition: and .x .y
```

But the following is not valid:

```
condition: and x y # Not valid. x and y need to be prepended by a dot .
```

### The *replace* transformer

  TODO

### Recepies

#### Always ignore

To always exclude some parts of the output regardless of user input, do as follows:

```yml
transformers:
  - name: do not include template code in the final output
    type: include
    region_marker: __DO_NOT_INCLUDE__
    condition: false
    files: ["**"]
```

And then in your source file(s):

```
# BEGIN __DO_NOT_INCLUDE__
... Code that should neven be included in the final output, such as
... templating specific scripts, makefiles etc
# END __DO_NOT_INCLUDE__
```

#### Conditional replace

Sometimes you need to conditionally replace a pattern. The condition may depend on user input. For example you may ask the user whether they'd like to includ gRPC functionality in the project or not and based on that render a different makefile.

Since the underlying rendering engine uses [go templates](https://golang.org/pkg/text/template/) it is possible to utilize the following condition:

```
{{ if .include_grpc }}build: build-grpc{{ else }}build:{{end}}
```

This condition will render `build:` if `.include_grpc` is false and `build: build-grpc` if `.include_grpc` is true. `include_grpc` is a user input in this case.

The complete example would look as follows then:

```yml
inputs:
  - id: include_grpc
    text: Should gRPC functionality be included?
    type: yesno
transformations:
  - name: build with grpc or not
    type: replace
    pattern: "build: build-grpc"
    replacement: "{{ if .include_grpc }}build: build-grpc{{ else }}build:{{end}}"
    files: ["Makefile"]
```

## Order of execution

Transformatinos are executed by the order they appear inside the transformations.yml file. The output of the first transformation is then piped into the input of the second transformation and so forth.
That means that the order is important such that if you're pattern needs to match certain text, you need to make sure that no previous transformation had changed this text. That's why it's wise to start with the more specific replacements and then move on to the more generic replacements.

Example:

```yml
transformations:
  - name: project long description
    type: replace
    pattern: Use go-archetype to transform project archetypes into existing live projects
    replacement: "{{ wrap 80 .ProjectDescription }}"
    files: ["cmd/root.go"]
  - name: project name
    type: replace
    pattern: go-archetype
    replacement: "{{ .ProjectName }}"
    files: ["*.go", "**/*.go"]
```

`project long description` should be placed before `project name`. If it weren't so then after applying ProjetName replacement on all occurences of the string `"go-archetype"` then the sentence `"Use go-archetype to transform project archetypes into existing live projects"` would have become `"Use my-project-name to transform project archetypes into existing live projects"` and then the replacement would not have been matched.

## Before and After

The `before` and `after` hooks allos you to run arbitrary shell command just before running all transformations or just after then.

They are provided with useful context that can be used in the actual command, which includes:

* `source` (Used as `{{ .source }}`)
* `destination` (Used as `{{ .destination }}`)
* As well as all user inputs

## Operations and debugging

The view detailed logs, run with `LOG_LEVEL=debug`
