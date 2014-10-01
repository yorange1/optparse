# Optparse

Optparse is a public domain, portable, reentrant, getopt-like option
parser. It's a single source and header file, so it can be trivially
dropped into any project. It supports POSIX getopt optstrings,
GNU-style long options, and subcommand processing.

## Why not getopt?

The POSIX getopt option parser has three fatal flaws. These flaws are
solved by Optparse.

1) The getopt parser state is stored entirely in global variables,
some of which are static and inaccessible. This means only one thread
can use getopt. It also means it's not possible to recursively parse
nested sub-arguments while in the middle of argument parsing. Optparse
fixes this by storing all state on a local struct.

2) The POSIX standard provides no way to properly reset the parser.
For portable code this means getopt is only good for one run, over one
argv with one optstring. It also means subcommand options cannot be
reliably processed with getopt(). Most implementations provide an
implementation-specific method to reset the parser, but this is not
portable. Optparse provides an `optparse_arg()` function for stepping
through non-option arguments, and parsing of options can continue
again at any time with a different optstring. The Optparse struct
itself could be passed around to subcommand handlers for additional
subcommand option parsing. If a full parser reset is needed,
`optparse_init()` can be called again.

3) In getopt, error messages are printed to stderr. This can be
disabled with opterr, but the messages themselves are still
inaccessible. Optparse solves this by writing the error message to its
errmsg field, which can be printed to anywhere. The downside to
Optparse is that this error message will always be in English rather
than the current locale.

## Drop-in Replacement

Optparse's interface should be familiar with anyone accustomed to
getopt. It's nearly a drop-in replacement. The optstring has the same
format and the parser struct fields have the same names as the getopt
global variables (optarg, optind, optopt).

The long option parser `optparse_long()` API is very similar to GNU's
`getopt_long()` and can serve as a portable, embedded replacement.

Optparse does not do any heap allocation, so no cleanup is needed.

See `optparse.h` for full API documentation.

## Example Usage

Here's a traditional getopt setup.

~~~c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
#include <getopt.h>

int main(int argc, char **argv)
{
    bool amend = false;
    bool brief = false;
    const char *color = "white";
    int delay = 0;

    int option;
    while ((option = getopt(argc, argv, "abc:d::")) != -1) {
        switch (option) {
        case 'a':
            amend = true;
            break;
        case 'b':
            brief = true;
            break;
        case 'c':
            color = optarg;
            break;
        case 'd':
            delay = optarg ? atoi(optarg) : 1;
            break;
        case '?':
            exit(EXIT_FAILURE);
        }
    }

    /* Print remaining arguments. */
    for (; optind < argc; optind++)
        printf("%s\n", argv[optind]);
    return 0;
}
~~~

Here's the same thing translated to Optparse.

~~~c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
#include "optparse.h"

int main(int argc, char **argv)
{
    bool amend = false;
    bool brief = false;
    const char *color = "white";
    int delay = 0;

    struct optparse options;
    optparse_init(&options, argc, argv);
    int option;
    while ((option = optparse(&options, "abc:d::")) != -1) {
        switch (option) {
        case 'a':
            amend = true;
            break;
        case 'b':
            brief = true;
            break;
        case 'c':
            color = options.optarg;
            break;
        case 'd':
            delay = options.optarg ? atoi(options.optarg) : 1;
            break;
        case '?':
            exit(EXIT_FAILURE);
        }
    }

    /* Print remaining arguments. */
    char *arg;
    while ((arg = optparse_arg(&options)))
        printf("%s\n", arg);
    return 0;
}
~~~

And here's a conversion to long options.

~~~c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
#include "optparse.h"

int main(int argc, char **argv)
{
    bool amend = false;
    bool brief = false;
    const char *color = "white";
    int delay = 0;

    struct optparse options;
    optparse_init(&options, argc, argv);
    struct optparse_long longopts[] = {
        {"amend", 'a', OPTPARSE_NONE},
        {"brief", 'b', OPTPARSE_NONE},
        {"color", 'c', OPTPARSE_REQUIRED},
        {"delay", 'd', OPTPARSE_OPTIONAL},
        {0}
    };
    int option;
    while ((option = optparse_long(&options, longopts, NULL)) != -1) {
        switch (option) {
        case 'a':
            amend = true;
            break;
        case 'b':
            brief = true;
            break;
        case 'c':
            color = optarg;
            break;
        case 'd':
            delay = optarg ? atoi(optarg) : 1;
            break;
        case '?':
            exit(EXIT_FAILURE);
        }
    }

    /* Print remaining arguments. */
    char *arg;
    while ((arg = optparse_arg(&options)))
        printf("%s\n", arg);
    return 0;
}
~~~