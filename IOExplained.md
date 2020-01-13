# I/O utilities

## `ByteStreams` and `CharStreams`

Guava uses the term "stream" to refer to a `Closeable` stream for I/O data which
has positional state in the underlying resource. The term "`byte` stream" refers
to an `InputStream` or `OutputStream`, while "`char` stream" refers to a
`Reader` or `Writer` (though their supertypes `Readable` and `Appendable` are
often used as method parameter types). Corresponding utilities are divided into
the utility classes [`ByteStreams`] and [`CharStreams`].

Most Guava stream-related utilities deal with an entire stream at a time and/or
handle buffering themselves for efficiency. Also note that Guava methods that
take a stream do _not_ close the stream: closing streams is generally the
responsibility of the code that opens the stream.

Some of the methods provided by these classes include:

| **`ByteStreams`**                                   | **`CharStreams`**                      |
| :-------------------------------------------------- | :------------------------------------- |
| [`byte[] toByteArray(InputStream)`][ByteStreams.toByteArray] | [`String toString(Readable)`] |
| N/A                                                 | [`List<String> readLines(Readable)`]   |
| [`long copy(InputStream, OutputStream)`]            | [`long copy(Readable, Appendable)`]    |
| [`void readFully(InputStream, byte[])`][ByteStreams.readFully] | N/A                         |
| [`void skipFully(InputStream, long)`]               | [`void skipFully(Reader, long)`]       |
| [`OutputStream nullOutputStream()`]                 | [`Writer nullWriter()`]                |

## Sources and sinks

It's common to create I/O utility methods that help you to avoid dealing with
streams at all when doing basic operations. For example, Guava has
`Files.toByteArray(File)` and `Files.write(File, byte[])`. However, you end up
with similar methods scattered all over, each dealing with a different kind of
_source_ of data or _sink_ to which data can be written. For example, Guava has
`Resources.toByteArray(URL)` which does the same thing as
`Files.toByteArray(File)`, but using a `URL` as the source of data rather than a
file.

To address this, Guava has a set of abstractions over different types of data
sources and sinks. A source or sink is a resource of some sort that you know how
to open a new stream to, such as a `File` or `URL`. Sources are readable, while
sinks are writable. Additionally, sources and sinks are broken down according to
whether you are dealing with `byte` or `char` data.

**Operations** | **Bytes**      | **Chars**
:------------- | :------------- | :-------------
**Reading**    | [`ByteSource`] | [`CharSource`]
**Writing**    | [`ByteSink`]   | [`CharSink`]

The advantage of these APIs is that they provide a common set of operations.
Once you've wrapped your data source as a `ByteSource`, for example, you get the
same set of methods no matter what that source happens to be.

#### Creating sources and sinks

Guava provides a number of source and sink implementations:

**Bytes**                                       | **Chars**
:---------------------------------------------- | :--------
[`Files.asByteSource(File)`]                    | [`Files.asCharSource(File, Charset)`]
[`Files.asByteSink(File, FileWriteMode...)`]    | [`Files.asCharSink(File, Charset, FileWriteMode...)`]
[`MoreFiles.asByteSource(Path, OpenOption...)`] | [`MoreFiles.asCharSource(Path, Charset, OpenOption...)`]
[`MoreFiles.asByteSink(Path, OpenOption...)`]   | [`MoreFiles.asCharSink(Path, Charset, OpenOption...)`]
[`Resources.asByteSource(URL)`]                 | [`Resources.asCharSource(URL, Charset)`]
[`ByteSource.wrap(byte[])`][ByteSource.wrap]    | [`CharSource.wrap(CharSequence)`]
[`ByteSource.concat(ByteSource...)`]            | [`CharSource.concat(CharSource...)`]
[`ByteSource.slice(long, long)`]                | N/A
[`CharSource.asByteSource(Charset)`]            | [`ByteSource.asCharSource(Charset)`]
N/A                                             | [`ByteSink.asCharSink(Charset)`]

In addition, you can extend the source and sink classes yourself to create new
implementations.

**Note:** While it can be tempting to create a source or sink that wraps an
_open_ stream (such as an `InputStream`), this should be avoided. Your
source/sink should instead open a new stream each time its `openStream()` method
is called. This allows the source or sink to control the full lifecycle of that
stream and allows it to be usable multiple times rather that becoming unusable
the first time any method on it is called. Additionally, if you're opening the
stream before creating the source or sink you may still have to deal with
ensuring that the stream is closed correctly if an exception is thrown elsewhere
in your code, which defeats many of the advantages of using a source or sink in
the first place.

#### Using Sources and Sinks

Once you have a source or sink instance, you have access to a number of
operations for reading or writing.

##### Common operations

All sources and sinks provide the ability to open a new stream for reading or
writing. By default, other operations are all implemented by calling one of
these methods to get a stream, doing something, and then ensuring that the
stream is closed.

These methods are all named:

*   `openStream()` - returns an `InputStream`, `OutputStream`, `Reader` or
    `Writer` depending on the type of source or sink.
*   `openBufferedStream()` - returns an `InputStream`, `OutputStream`,
    `BufferedReader` or `Writer` depending on the type of source or sink. The
    returned stream is guaranteed to be buffered if necessary. For example, a
    source that reads from a byte array has no need for additional buffering in
    memory. This is why the methods do not return `BufferedInputStream` etc.
    except in the case of `BufferedReader`, because it defines the `readLine()`
    method.

##### Source operations

| **`ByteSource`**                          | **`CharSource`**                          |
| :---------------------------------------- | :---------------------------------------- |
| [`byte[] read()`][ByteSource.read]        | [`String read()`]                         |
| N/A                                       | [`ImmutableList<String> readLines()`]     |
| N/A                                       | [`String readFirstLine()`]                |
| [`long copyTo(ByteSink)`]                 | [`long copyTo(CharSink)`]                 |
| [`long copyTo(OutputStream)`]             | [`long copyTo(Appendable)`]               |
| [`Optional<Long> sizeIfKnown()`]          | [`Optional<Long> lengthIfKnown()`]        |
| [`long size()`]                           | [`long length()`]                         |
| [`boolean isEmpty()`][ByteSource.isEmpty] | [`boolean isEmpty()`][CharSource.isEmpty] |
| [`boolean contentEquals(ByteSource)`]     | N/A                                       |
| [`HashCode hash(HashFunction)`]           | N/A                                       |

##### Sink operations

| **`ByteSink`**                         | **`CharSink`**                                                |
| :------------------------------------- | :------------------------------------------------------------ |
| [`void write(byte[])`][ByteSink.write] | [`void write(CharSequence)`]                                  |
| [`long writeFrom(InputStream)`]        | [`long writeFrom(Readable)`]                                  |
| N/A                                    | [`void writeLines(Iterable<? extends CharSequence>)`]         |
| N/A                                    | [`void writeLines(Iterable<? extends CharSequence>, String)`] |

#### Examples

```java
// Read the lines of a UTF-8 text file
ImmutableList<String> lines = Files.asCharSource(file, Charsets.UTF_8)
    .readLines();

// Count distinct word occurrences in a file
Multiset<String> wordOccurrences = HashMultiset.create(
    Splitter.on(CharMatcher.whitespace())
        .trimResults()
        .omitEmptyStrings()
        .split(Files.asCharSource(file, Charsets.UTF_8).read()));

// SHA-1 a file
HashCode hash = Files.asByteSource(file).hash(Hashing.sha1());

// Copy the data from a URL to a file
Resources.asByteSource(url).copyTo(Files.asByteSink(file));
```

## `Files`

In addition to methods for creating file sources and sinks, the `Files` class
contains a number of convenience methods that you might be interested in.

| Method                              | Description                                                       |
| :---------------------------------- | :---------------------------------------------------------------- |
| [`createParentDirs(File)`]          | Creates necessary but nonexistent parent directories of the file. |
| [`getFileExtension(String)`]        | Gets the file extension of the file described by the path.        |
| [`getNameWithoutExtension(String)`] | Gets the name of the file with its extension removed              |
| [`simplifyPath(String)`]            | Cleans up the path. Not always consistent with your filesystem; test carefully! |
| [`fileTraverser()`]                 | Returns a `Traverser` that can traverse file trees                |

[`ByteStreams`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteStreams.html
[`CharStreams`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharStreams.html
[ByteStreams.toByteArray]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteStreams.html#toByteArray-java.io.InputStream-
[`String toString(Readable)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharStreams.html#toString-java.lang.Readable-
[`List<String> readLines(Readable)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharStreams.html#readLines-java.lang.Readable-
[`long copy(InputStream, OutputStream)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteStreams.html#copy-java.io.InputStream-java.io.OutputStream-
[`long copy(Readable, Appendable)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharStreams.html#copy-java.lang.Readable-java.lang.Appendable-
[ByteStreams.readFully]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteStreams.html#readFully-java.io.InputStream-byte:A-
[`void skipFully(InputStream, long)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteStreams.html#skipFully-java.io.InputStream-long-
[`void skipFully(Reader, long)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharStreams.html#skipFully-java.io.Reader-long-
[`OutputStream nullOutputStream()`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteStreams.html#nullOutputStream--
[`Writer nullWriter()`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharStreams.html#nullWriter--
[`ByteSource`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSource.html
[`CharSource`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSource.html
[`ByteSink`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSink.html
[`CharSink`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSink.html
[`Files.asByteSource(File)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/Files.html#asByteSource-java.io.File-
[`Files.asCharSource(File, Charset)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/Files.html#asCharSource-java.io.File-java.nio.charset.Charset-
[`MoreFiles.asByteSource(Path, OpenOption...)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/MoreFiles.html#asByteSource-java.nio.file.Path-java.nio.file.OpenOption...-
[`MoreFiles.asCharSource(Path, Charset, OpenOption...)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/MoreFiles.html#asCharSource-java.nio.file.Path-java.nio.charset.Charset-java.nio.file.OpenOption...-
[`Files.asByteSink(File, FileWriteMode...)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/Files.html#asByteSink-java.io.File-com.google.common.io.FileWriteMode...-
[`MoreFiles.asByteSink(Path, OpenOption...)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/MoreFiles.html#asByteSink-java.nio.file.Path-java.nio.file.OpenOption...-
[`Files.asCharSink(File, Charset, FileWriteMode...)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/Files.html#asCharSink-java.io.File-java.nio.charset.Charset-com.google.common.io.FileWriteMode...-
[`MoreFiles.asCharSink(Path, Charset, OpenOption...)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/MoreFiles.html#asCharSink-java.nio.file.Path-java.nio.charset.Charset-java.nio.file.OpenOption...-
[`Resources.asByteSource(URL)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/Resources.html#asByteSource-java.net.URL-
[`Resources.asCharSource(URL, Charset)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/Resources.html#asCharSource-java.net.URL-java.nio.charset.Charset-
[ByteSource.wrap]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSource.html#wrap-byte:A-
[`CharSource.wrap(CharSequence)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSource.html#wrap-java.lang.CharSequence-
[`ByteSource.concat(ByteSource...)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSource.html#concat-com.google.common.io.ByteSource...-
[`CharSource.concat(CharSource...)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSource.html#concat-com.google.common.io.CharSource...-
[`ByteSource.slice(long, long)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSource.html#slice-long-long-
[`CharSource.asByteSource(Charset)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSource.html#asByteSource-java.nio.charset.Charset-
[`ByteSource.asCharSource(Charset)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSource.html#asCharSource-java.nio.charset.Charset-
[`ByteSink.asCharSink(Charset)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSink.html#asCharSink-java.nio.charset.Charset-
[ByteSource.read]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSource.html#read--
[`String read()`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSource.html#read--
[`ImmutableList<String> readLines()`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSource.html#readLines--
[`String readFirstLine()`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSource.html#readFirstLine--
[`long copyTo(ByteSink)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSource.html#copyTo-com.google.common.io.ByteSink-
[`long copyTo(CharSink)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSource.html#copyTo-com.google.common.io.CharSink-
[`long copyTo(OutputStream)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSource.html#copyTo-java.io.OutputStream-
[`long copyTo(Appendable)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSource.html#copyTo-java.lang.Appendable-
[`Optional<Long> sizeIfKnown()`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSource.html#sizeIfKnown--
[`Optional<Long> lengthIfKnown()`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSource.html#lengthIfKnown--
[`long size()`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSource.html#size--
[`long length()`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSource.html#length--
[ByteSource.isEmpty]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSource.html#isEmpty--
[CharSource.isEmpty]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSource.html#isEmpty--
[`boolean contentEquals(ByteSource)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSource.html#contentEquals-com.google.common.io.ByteSource-
[`HashCode hash(HashFunction)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSource.html#hash-com.google.common.hash.HashFunction-
[ByteSink.write]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSink.html#write-byte:A-
[`void write(CharSequence)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSink.html#write-java.lang.CharSequence-
[`long writeFrom(InputStream)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/ByteSink.html#writeFrom-java.io.InputStream-
[`long writeFrom(Readable)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSink.html#writeFrom-java.lang.Readable-
[`void writeLines(Iterable<? extends CharSequence>)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSink.html#writeLines-java.lang.Iterable-
[`void writeLines(Iterable<? extends CharSequence>, String)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/CharSink.html#writeLines-java.lang.Iterable-java.lang.String-
[`createParentDirs(File)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/Files.html#createParentDirs-java.io.File-
[`getFileExtension(String)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/Files.html#getFileExtension-java.lang.String-
[`getNameWithoutExtension(String)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/Files.html#getNameWithoutExtension-java.lang.String-
[`simplifyPath(String)`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/Files.html#simplifyPath-java.lang.String-
[`fileTraverser()`]: http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/io/Files.html#fileTraverser--
