# Refactorings


Guava, as a library, deprecates APIs with noticeable frequency, though we [no
longer delete] APIs not annotated `@Beta`. Inside Google, we deprecate and
delete APIs much more aggressively, which we can make work because we have
infrastructure in place to refactor the entire Google codebase away from APIs
which we wish to eliminate. (Some background about this infrastructure can be
found [here][monolithic].)

To reduce the pain of upgrading versions of Guava, and help users migrate away
from APIs we consider deprecated, we are experimenting with releasing some of
our internal refactorings. Currently, these refactorings use
[Refaster][refaster], a refactoring tool we developed to refactor based on
simple, before-and-after code examples in Java.

Currently, Refaster is released as part of Google's Java compiler tools project,
[Error Prone]. Our external support for
Refaster is still somewhat experimental, but we hope you'll be interested in
experimenting with it. The documentation can be found [here][refaster-docs]. The
[refactorings released with Guava][refactorings] should be applicable by
directly following those instructions.

This is an experiment, and we hope you will find it useful. If these
refactorings work for you — or if they don't! — please let us know.

[monolithic]: https://cacm.acm.org/magazines/2016/7/204032-why-google-stores-billions-of-lines-of-code-in-a-single-repository/fulltext
[refaster]: https://research.google.com/pubs/archive/41876.pdf
[refaster-docs]: https://errorprone.info/docs/refaster
[refactorings]: https://github.com/google/guava/tree/master/refactorings
[no longer delete]: https://groups.google.com/forum/?utm_medium=email&utm_source=footer#!msg/guava-discuss/rX-QXo-67ZU/gLEvfV4CAwAJ
[Error Prone]: https://github.com/google/error-prone
