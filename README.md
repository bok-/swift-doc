# swift-doc

A package for generating documentation for Swift projects.

**This project is under active development
and is expected to change significantly before its first stable release.**

Given a directory of Swift files,
`swift-doc` generates CommonMark (Markdown) files
for each class, structure, enumeration, and protocol
as well as top-level type aliases, functions, and variables.

For an example of generated documentation,
[check out the Wiki for our fork of Alamofire][alamofire wiki]. 

> **Note**:
> Output is currently limited to CommonMark,
> but the plan is to support HTML and other formats as well.

## Usage

`swift-doc` can be used from the command-line 
or in a [GitHub Actions][github actions] workflow.

### Command-Line Utility

To run `swift-doc` from the command-line,
clone the repository
and do `swift run swift-doc` from within the project directory.

```terminal
$ git clone https://github.com/SwiftDocOrg/swift-doc.git
$ cd swift-doc
$ swift run swift-doc path/to/SwiftProject/Sources --output Documentation
$ tree Documentation
$ Documentation/
├── Home
├── (...)
├── _Footer.md
└── _Sidebar.md
```

`swift-doc` takes one or more paths and enumerates them recursively,
collecting all Swift files into a single "module"
and generating documentation accordingly.
By default,
output files are written to `.build/documentation`,
but you can change that with the `--output` option flag.

### GitHub Action

This repository also hosts a [GitHub Action][github actions]
that you can incorporate into your project's workflow.

The CommonMark files generated by `swift-doc` 
are formatted for publication to your project's [GitHub Wiki][github wiki],
which you can do with 
[github-wiki-publish-action][github-wiki-publish-action].
Alternatively,
you could publish `swift-doc`-generated documentation to GitHub Pages,
or bundle them into a release artifact.

#### Inputs

- `inputs`: 
  One or more paths to Swift files in your workspace. 
  (Default: `"./Sources"`)
- `output`:
  The path for generated output. 
  (Default: `"./.build/documentation"`)

#### Example Workflow

```yml
name: Documentation

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v1
      - name: Generate Documentation
        uses: SwiftDocOrg/swift-doc@master
        with:
          inputs: "Source"
          output: "Documentation"
      - name: Upload Documentation to Wiki
        uses: SwiftDocOrg/github-wiki-publish-action@master
        with:
          path: "Documentation"
        env:
          GITHUB_PERSONAL_ACCESS_TOKEN: ${{ secrets.GITHUB_PERSONAL_ACCESS_TOKEN }}
```

* * *

In addition to `swift-doc`,
this project currently ships with several, experimental executables
that offer complementary functionality.
It's unclear how all everything will ultimately fit together,
but for now, 
they're incubating in a shared monorepo
(the intent is for each of them to eventually become 
an option, subcommand, or plugin of `swift-doc`).

* * *

### swift-dcov

`swift-dcov` generates documentation coverage statistics for Swift files.

```terminal
$ git clone https://github.com/SwiftDocOrg/SwiftSemantics.git

$ swift run swift-dcov SwiftSemantics/Sources/ | jq ".data.totals"
{
  "count": 207,
  "documented": 199,
  "percent": 96.1352657004831
}

$ swift run swift-dcov SwiftSemantics/Sources/ | jq ".data.symbols[] | select(.documented == false)"
{
  "file": "SwiftSemantics/Supporting Types/GenericRequirement.swift",
  "line": 67,
  "column": 6,
  "name": "GenericRequirement.init?(_:)",
  "type": "Initializer",
  "documented": false
}
...
```

While there are plenty of tools for assessing test coverage for code,
we weren't able to find anything analogous for documentation coverage.
To this end,
we've contrived a simple JSON format 
[inspired by llvm-cov](https://reviews.llvm.org/D22651#change-xdePaVfBugps).

If you know of an existing standard 
that you think might be better suited for this purpose,
please reach out by [opening an Issue][open an issue]!

### swift-api-inventory

`swift-api-inventory` lists the public API symbols of Swift files.

```terminal
$ git clone https://github.com/SwiftDocOrg/SwiftSemantics.git

$ swift run swift-api-inventory SwiftSemantics/Sources | less
struct AssociatedType: Declaration, Hashable, Codable, ExpressibleBySyntax
var AssociatedType.attributes { get }
var AssociatedType.context { get }
var AssociatedType.keyword { get }
var AssociatedType.modifiers { get }
var AssociatedType.name { get }
...

$ swift run swift-api-inventory SwiftSemantics/Sources | wc
     207    1023    8993
```

Because each symbol is printed on its own line,
you can feed the output of `swift-api-inventory` to conventional diffing tools
to determine API changes between different releases of a project.

For example,
here's an API diff between the first beta and latest release candidate of 
[Alamofire 5](https://forums.swift.org/t/alamofire-5-one-year-in-the-making-now-in-beta/18865):

```terminal
$ git clone https://github.com/Alamofire/Alamofire.git
$ (cd Alamofire; git co 5.0.0-beta.1; swift run swift-api-inventory Source > ../Alamofire-5.0.0-beta.1.txt)
$ (cd Alamofire; git co 5.0.0-rc.3; swift run swift-api-inventory Source > ../Alamofire-5.0.0-rc.3.txt)
$ diff -u Alamofire-5.0.0-beta.1.txt Alamofire-5.0.0-rc.3.txt | diffstat
 Alamofire-5.0.0-rc.3.txt |  346 ++++++++++++++++++++++++++++++++—————
 1 file changed, 238 insertions(+), 108 deletions(-)
```

<details>

<summary>Example diff between Alamofire 5 RC3 and RC1</summary>

```terminal
$ diff -u Alamofire-5.0.0-rc.1.txt Alamofire-5.0.0-rc.3.txt
```

```diff
— Alamofire-5.0.0-rc.1.txt
+++ Alamofire-5.0.0-rc.3.txt
@@ -77,6 +77,7 @@
 case AFError.ServerTrustFailureReason.revocationCheckFailed(output: Output, options: RevocationTrustEvaluator.Options)
 case AFError.ServerTrustFailureReason.revocationPolicyCreationFailed
 case AFError.ServerTrustFailureReason.settingAnchorCertificatesFailed(status: OSStatus, certificates: [SecCertificate])
+case AFError.ServerTrustFailureReason.trustEvaluationFailed(error: Error?)
 enum AFError.URLRequestValidationFailureReason
 case AFError.URLRequestValidationFailureReason.bodyDataInGETRequest(_: Data)
 case AFError.createURLRequestFailed(error: Error)
@@ -613,13 +614,14 @@
 case URLEncodedFormEncoder.SpaceEncoding.percentEscaped
 case URLEncodedFormEncoder.SpaceEncoding.plusReplaced
 var URLEncodedFormEncoder.allowedCharacters { get set }
+var URLEncodedFormEncoder.alphabetizeKeyValuePairs { get }
 var URLEncodedFormEncoder.arrayEncoding { get }
 var URLEncodedFormEncoder.boolEncoding { get }
 var URLEncodedFormEncoder.dataEncoding { get }
 var URLEncodedFormEncoder.dateEncoding { get }
 func URLEncodedFormEncoder.encode(_: Encodable) throws -> String
 func URLEncodedFormEncoder.encode(_: Encodable) throws -> Data
-init URLEncodedFormEncoder(arrayEncoding: ArrayEncoding, boolEncoding: BoolEncoding, dataEncoding: DataEncoding, dateEncoding: DateEncoding, keyEncoding: KeyEncoding, spaceEncoding: SpaceEncoding, allowedCharacters: CharacterSet)
+init URLEncodedFormEncoder(alphabetizeKeyValuePairs: Bool, arrayEncoding: ArrayEncoding, boolEncoding: BoolEncoding, dataEncoding: DataEncoding, dateEncoding: DateEncoding, keyEncoding: KeyEncoding, spaceEncoding: SpaceEncoding, allowedCharacters: CharacterSet)
```

</details>

### swift-api-diagram

`swift-dcov` generates a graph of APIs in [DOT format][dot]
that can be rendered by [GraphViz][graphviz] into a diagram.

```terminal
$ swift run swift-api-diagram Alamofire/Source > graph.dot
$ head graph.dot
digraph Anonymous {
  "Session" [shape=box];
  "NetworkReachabilityManager" [shape=box];
  "URLEncodedFormEncoder" [shape=box,peripheries=2];
  "ServerTrustManager" [shape=box];
  "MultipartFormData" [shape=box];
  
  subgraph cluster_Request {
    "DataRequest" [shape=box];
    "Request" [shape=box];

$ dot -T svg graph.dot > graph.svg
```

Here's an excerpt of the graph generated for Alamofire:

![Excerpt of swift-doc-api Diagram for Alamofire](https://user-images.githubusercontent.com/7659/73189318-0db0e880-40d9-11ea-8895-341a75ce873c.png)


## Motivation

From its earliest days,
Swift has been fortunate to have [Jazzy][jazzy],
which is a fantastic tool for generating documentation
for both Swift and Objective-C projects.
Over time, however,
the way we write Swift code —
and indeed the language itself —
has evolved to incorporate patterns and features 
that are difficult to understand using 
the same documentation standards that served us well for Objective-C.

Whereas in Objective-C,
you could get a complete view of a type's functionality from its class hierarchy,
Swift code today tends to layer and distribute functionality across 
[a network of types][swift number protocols diagram].
While adopting a
[protocol-oriented paradigm][protocol-oriented programming]
can make Swift easier and more expressive to write,
it can also make Swift code more difficult to understand.

Our primary goal for `swift-doc`
is to make Swift documentation more useful
by surfacing the information you need to understand how an API works
and presenting it in a way that can be easily searched and accessed.
We want developers to be empowered to use Swift packages to their full extent,
without being reliant on (often outdated) blog posts or Stack Overflow threads. 
We want documentation coverage to become as important as test coverage:
a valuable metric for code quality,
and an expected part of first-rate open source projects.

Jazzy styles itself after Apple's official documentation circa 2014
(code-named "Jazz", as it were),
which was well-suited to understanding Swift code as we wrote it back then
when it was more similar to Objective-C.
But this design is less capable of documenting the behavior of
generically-constrained types,
default implementations,
[dynamic member lookup][se-0195],
[property wrappers][se-o258], or
[function builders][se-xxxx].
(Alas,
Apple's [most recent take][apple documentation] on reference documentation
hasn't improved matters,
having instead focused on perceived cosmetic issues.)

Without much in the way of strong design guidance,
we're not entirely sure what Swift documentation _should_ look like.
But we do think plain text is a good place to start.
We look forward to soliciting feedback and ideas from everyone
to identify what those needs are and come up with the best ways to meet them.

In the meantime,
we've set ourselves up for success
by investing in the kind of foundation we'll need
to build whatever we decide best solves the problems at hand.
`swift-doc` is built on top of a constellation of projects
that take advantage of modern infrastructure and tooling:

- [SwiftSemantics][swiftsemantics]:
  Parses Swift code into its constituent declarations
  using [SwiftSyntax][swiftsyntax]
- [SwiftMarkup][swiftmarkup]:
  Parses Swift documentation comments into structured entities
  using [CommonMark][commonmark]
- [github-wiki-publish-action][github-wiki-publish-action]:
  Publishes the contents of a directory to your project's wiki

These new technologies have already yielded some promising results.
`swit-doc` is built in Swift,
and can be installed both macOS and Linux as a small, standalone binary.
Because it relies only on a syntactic reading of Swift source code,
without needing code first to be compiled,
`swift-doc` is quite fast.
As a baseline,
compare its performance to Jazzy 
when generating documentation for [SwiftSemantics][swiftsemantics]:

```terminal
$ cd SwiftSemantics

$ time swift-doc Sources
        0.21 real         0.16 user         0.02 sys

$ time jazzy # fresh build
jam out ♪♫ to your fresh new docs in `docs`
       67.36 real        98.76 user         8.89 sys


$ time jazzy # with build cache
jam out ♪♫ to your fresh new docs in `docs`
       17.70 real         2.17 user         0.88 sys
```

Of course,
some of that is simply Jazzy doing more,
generating HTML, CSS, and a search index instead of just text.
Compare its [generated HTML output][jazzy swiftsemantics]
to [a GitHub wiki generated with `swift-doc`][swift-doc swiftsemantics].

## What About [SwiftDoc.org][swiftdoc.org]?

**tl;dr:** 
We're currently working on updating SwiftDoc.org for Swift 5,
and hope to have it released later this week.

SwiftDoc.org,
[originally "Swifter"](http://natecook.com/blog/2014/09/introducing-swifter/),
was created by Nate Cook ([@natecook1000][@natecook1000]) 
in September 2014.
At the time,
Swift tooling was still in its infancy,
so Nate actually 
[wrote a parser (from scratch!)](https://github.com/SwiftDocOrg/swiftdoc-parser)
to pull symbols and documentation from the Swift standard library.
Nate became managing editor of [NSHipster][nshipster] in 2015,
bringing SwiftDoc with him as an affiliated project.
When Mattt took over NSHipster duties for Nate in 2018,
he inherited SwiftDoc along with it.

After the hand-off,
we were able to get the site updated for Swift 4.2 without too much trouble.
But when it came time to regenerate the site for Swift 5,
we found ourselves deep in ["dependency hell"][dependency hell]
(something to do with the [regular expression][pcre] library 
that Nate had used for the parser).
After begging and pleading with 
the spirits possessing our `node_modules` directory to no avail,
we decided to roll up our sleeves and get started on a long-term replacement —
this time, written in Swift.

_Thanks for all of your encouragement about the site over the years
and your patience throughout this whole process.
We're sorry it took so long to get around to getting it updated,
but we hope this all will have been worth the wait!_ 🙇‍♂️

## Project Roadmap

_(Coming soon!)_

## License

MIT

## Contact

Mattt ([@mattt](https://twitter.com/mattt))

[alamofire wiki]: https://github.com/SwiftDocOrg/Alamofire/wiki
[github wiki]: https://help.github.com/en/github/building-a-strong-community/about-wikis
[github actions]: https://github.com/features/actions
[swiftsyntax]: https://github.com/apple/swift-syntax
[swiftsemantics]: https://github.com/SwiftDocOrg/SwiftSemantics
[swiftmarkup]: https://github.com/SwiftDocOrg/SwiftMarkup
[commonmark]: https://github.com/SwiftDocOrg/CommonMark
[github-wiki-publish-action]: https://github.com/SwiftDocOrg/github-wiki-publish-action
[open an issue]: https://github.com/SwiftDocOrg/swift-doc/issues/new
[jazzy]: https://github.com/realm/jazzy
[swift number protocols diagram]: https://nshipster.com/propertywrapper/#swift-number-protocols
[protocol-oriented programming]: https://asciiwwdc.com/2015/sessions/408
[apple documentation]: https://developer.apple.com/documentation
[se-0195]: https://github.com/apple/swift-evolution/blob/master/proposals/0195-dynamic-member-lookup.md
[se-o258]: https://github.com/apple/swift-evolution/blob/master/proposals/0258-property-wrappers.md
[se-xxxx]: https://github.com/apple/swift-evolution/blob/9992cf3c11c2d5e0ea20bee98657d93902d5b174/proposals/XXXX-function-builders.md
[swiftdoc.org]: https://swiftdoc.org
[jazzy swiftsemantics]: https://swift-semantics-jazzy.netlify.com
[swift-doc swiftsemantics]: https://github.com/SwiftDocOrg/SwiftSemantics/wiki
[@natecook1000]: https://github.com/natecook1000
[nshipster]: https://nshipster.com
[dependency hell]: https://github.com/apple/swift-package-manager/tree/master/Documentation#dependency-hell
[pcre]: https://en.wikipedia.org/wiki/Perl_Compatible_Regular_Expressions
[dot]: https://en.wikipedia.org/wiki/DOT_(graph_description_language)
[graphviz]: https://www.graphviz.org
