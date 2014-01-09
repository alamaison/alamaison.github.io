---
layout: post
title: "Proposal: Generic filesystem API"
comments: true
---

<script src="//cdn.jsdelivr.net/coliru/0.1.0/coliru.min.js"></script>
<script>
    window.onload = coliru.addRunButtonsToCodeBlocks;
    coliru.linkLibraries = ['boost_system', 'boost_filesystem'];
</script>

**\[FIRST DRAFT\]**

C++ streams provide a common interface to operate on file-like data,
regardless of how that data it is actually stored.  However, programs
often need to operate not just on data from different file sources,
but also on entirely separate filesystems.

## The problem

There is, currently, no way to operate on different filesystems
generically, if Boost.Filesystem is to be one of those filesystems.
The local-filesystem operations in the Boost.Filesystem API make it
difficult to call them generically because they, inadvertently, defeat
ADL, so generic code cannot resolve them to specific
implementations.

For example, how do you write code that calls
[`path temp_directory_path();`](http://www.boost.org/doc/libs/1_55_0/libs/filesystem/doc/reference.html#temp_directory_path)
and can operate on both the local filesystem, via Boost.Filesystem,
and on a filesystem over FTP, via an FTP library?  If both libraries
declare the function, how do you resolve the correct implementation?
You can't use a namespace as a type.

Normally, ADL is the solution to the problem; it resolves the correct
namespace based on the namespace of the operation's argument(s).  But
[`temp_directory_path()`](http://www.boost.org/doc/libs/1_55_0/libs/filesystem/doc/reference.html#temp_directory_path)
doesn't take an argument.

The source of the problem is a misconception that the free-functions
performing FS operations are part of the [`class
path`](http://www.boost.org/doc/libs/1_55_0/libs/filesystem/doc/reference.html#class-path)
API, when really they are part of the API of an implicit
local-filesystem object. The [Boost.Filesystem FAQ
asks](http://www.boost.org/doc/libs/1_55_0/libs/filesystem/doc/faq.htm):

> **Why are paths sometimes manipulated by member functions and
>   sometimes by non-member functions?**
>
> The design rule is that purely lexical operations are supplied as
> class path member functions, while operations performed by the
> operating system are provided as free functions.

This is wrong because the majority of the non-member operations don't
manipulate paths at all.  They are functions that manipulate the
filesystem, some of which use a path, and, therefore, are really part
of the API of an implicit local filesystem object.  The [proposed
solution](#proposed-solution) just makes this explicit, in a
backward-compatible way.

## Is modifying Boost.Filesystem necessary?

Before we discuss our proposal, let us explore how much can be
achieved without any changes to Boost.Filesystem, using the typical
ADL approach to generic programming.

### Limited solution: ADL on filesystem-specific path

If each filesystem were to declare a `path` class conforming to the
[`boost::filesystem::path`
interface](http://www.boost.org/doc/libs/1_55_0/libs/filesystem/doc/reference.html#class-path),
ADL could resolve a *subset* of the filesystem operations.

The example below implements a simple generic algorithm,
`remove_if_temporary` over both Boost.Filesystem and an imaginary
`ftp_filesystem`.

```c++
#include <iostream>
#include <boost/filesystem.hpp>

namespace ftp_filesystem {

    class path
    {
    public:

        path(const std::string& server, const std::string& location)
            : server_(server), location_(location) {}

        std::string string() const
        { return location_; }

        /* ... */
    private:
        std::string server_;
        std::string location_;
    };

    void remove(const path& target)
    {
        std::cout << "Removing " << target.string()
                  << " over FTP" << std::endl;
    };

}

template<typename Path>
void remove_if_temporary(const Path& file_path)
{
    if (file_path.string()[file_path.string().size() - 1] == '~')
    {
        remove(file_path);
    }
}

int main()
{
    remove_if_temporary(boost::filesystem::path("/tmp/bob"));
    remove_if_temporary(boost::filesystem::path("/home/bob~"));

    remove_if_temporary(
        ftp_filesystem::path("ftp.example.com", "/tmp/bob"));
    remove_if_temporary(
        ftp_filesystem::path("ftp.example.com", "/home/bob~"));
}
```

The key to this solution is that each filesystem's namespace maintains
a separate `path` type.  The algorithm is instantiated with different
path types, allowing ADL to resolve the necessary implementations.

However, this is not a good solution because it is continues the
pretence that operations are part of `class path`, but this pretence
only stretches so far.  The result is that:

1. Operations that don't take a `path` argument can't be used in generic
   algorithms.  Some operations, such as `temp_directory_path()` just
   *return* a `path`.  ADL won't resolve those.
2. `path` becomes coupled to a particular filesystem.
   `boost::filesystem::path` is a currently a good abstraction of many
   filesystem's path representation, but making it as the ADL key that
   resolves operations to their implementation means that other
   filesystems can't use it as their path representation (would
   `BOOST_STRONG_TYPEDEF` solve this?).
3. `path` must carry filesystem instance data.  The local
   filesystem is a singleton so its path only needs the
   location-on-disk string.  But other types of filesystem may be
   instantiated with host names, authentication information, running
   network sessions, etc.  The path instances have to carry this
   extra baggage.

In addition, important parts of the API, namely file opening/creation
and directory iteration, can still not be used generically, without
changes to Boost.Filesystem, because they are *classes* constructed
with a `path`, rather than free functions taking a `path` argument.
Solving this will need additional factory functions for each class so
that ADL can resolve them.

**In summary**: generic filesystem programming is possible with no
changes to Boost.Filesystem if generic algorithms don't open files,
create files, iterate directories or use any nullary filesystem
operations.  Even accepting these limitations, the API is not ideal
because of undesirable coupling between `path` and the specifics of
a filesystem.

<a name="proposed-solution"></a>
### Proposed solution: Add local filesystem object to Boost.Filesystem

The main problem with the above solution is that nullary operations
cannot be resolved by ADL.  In effect, these operations take a hidden
local filesystem argument, but, as the argument is implicit, ADL
cannot use it to resolve them

The suggestion in this proposal is to make the filesystem object
explicit so that:

- ADL can resolve all the operations using the namespace of this
  object; or
- the operations can be resolved as methods of this object.

Either change is backward-compatible. The existing functions remain in
place, and code that only uses the local filesystem need never refer
to the local filesystem object explicitly.

The example below demonstrates the second option.  This is my
preferred option because all filesystems, except the local one, are
likely to implement their operations in terms of private data in the
filesystem object.  Passing this object to free functions, instead of
calling a method, means this private data must be exposed.

The example below demonstrates a partial implementation of the
additional new `class filesystem`, and shows how to use it generically.

```c++
#include <boost/filesystem.hpp>

namespace ftp_filesystem {

    class filesystem
    {
    public:

        typedef boost::filesystem::path path;

        filesystem(const std::string& server)
            : server_(server) {}

        path temp_directory_path() const
        {
            std::cout << "Querying " << server_
                      << " for $TMPDIR\n";
            return "/tmp";
        }

        void remove(const path& target)
        {
            std::cout << "Removing " << target.string()
                      << " on " << server_
                      << " over FTP" << std::endl;
        };

    private:
        std::string server_;
    };

}

/* Added to existing Boost.Filesystem API */
namespace boost { namespace filesystem {

    class filesystem
    {
    public:

        typedef boost::filesystem::path path;

        path temp_directory_path() const
        {
            return temp_directory_path();
        }

        void remove(const path& target)
        {
            remove(target);
        };

    private:
        filesystem() {}

        friend boost::filesystem::filesystem& local_filesystem();
    };

    /* Local filesystem is singleton */
    inline boost::filesystem::filesystem& local_filesystem()
    {
        static filesystem instance;
        return instance;
    }
}}

template<typename Filesystem>
void remove_if_temporary(
    Filesystem& fs, const typename Filesystem::path& file_path)
{
    std::string name = file_path.filename().string();

    if (name[name.size() - 1] == '~' ||
        file_path.parent_path() == fs.temp_directory_path())
    {
        fs.remove(file_path);
    }
}

int main()
{
    ftp_filesystem::filesystem ftp_fs("ftp.example.com");

    boost::filesystem::filesystem& local_fs =
        boost::filesystem::local_filesystem();

    boost::filesystem::path in_temp_dir("/tmp/bob");
    boost::filesystem::path temp_by_name("/home/bob~");

    remove_if_temporary(ftp_fs, in_temp_dir);
    remove_if_temporary(ftp_fs, temp_by_name);

    remove_if_temporary(local_fs, in_temp_dir);
    remove_if_temporary(local_fs, temp_by_name);
}
```

As well as the technical reasons for introducing this change, I
believe it better reflects the division of responsibilities between
`class path` and the filesystem.  Paths are not coupled to the
filesystem and can be shared between supporting filesystems.  If a
particular filesystem needs a different path implementation, that is
supported too and it is up to the generic algorithm whether it works
with all path implementations or just `boost::filesystem::path`
instances.  Filesystem details are encapsulated in the filesystem
object alone and it performs all the operations using those internal
details.

<div class="message">
    The example above implements <code>class filesystem</code> in
    terms of the existing operations functions so that it will compile
    with no further changes.  A better idea may be to implement the
    existing operations in terms of the new filesystem object's
    methods because that better reflects their meaning.
</div>

## Impact on Boost.Filesystem

The impact on existing users of Boost.Filesystem should be minimal.
The only change they may notice is extra documentation for generic use
of the API.

### Backward compatibility

No existing code using Boost.Filesystem will need any changes.  The
existing free functions remain in place as a sensible convenience for
the common case where only the local filesystem is needed.

### Additions

A new object `boost::filesystem::filesystem` is introduced, along with
its singleton accessor function, `local_filesystem`.

Depending which of the two API options is chosen:

- the operation free-functions are overloaded to take an instance of
  `boost::filesystem::filesystem`; or
- `class boost::filesystem::filesystem` is given methods that perform
  each of the operations currently in free functions.

Even functions `absolute()` and `canonical()`, which may seem to be
part of the path API, are really part of the filesystem object's API:
they can be implemented solely in terms on `class path`'s public API
but not solely in terms of `class filesystem`'s API.

Finally, the classes in the API that perform operations on the
filesystem, e.g. `fstream`, `directory_iterator` require one of the
following changes:

- Add factory functions for each class to `namespace
  boost::filesystem`.  Allows them to be resolved by ADL.  Generic
  callers use auto as return type.
- Add static factory functions for each class to the `path`
  class. Allows them to be resolved by ADL.  Generic callers use auto
  as return type.
- Add typedefs for each class to `path` class. Callers construct
  classes directly.

Factory functions for streams will require a workaround in C++03
because the standard streams are not movable.

## TODO

- Define filesystem-path concept
- Define filesystem concept