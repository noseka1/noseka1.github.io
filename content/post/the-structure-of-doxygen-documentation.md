+++
title = "The Structure of Doxygen Documentation"
date = "2015-06-21"
slug = "2015/06/21/the-structure-of-doxygen-documentation"
Categories = [ "development" ]
+++

Let's review some basic means that Doxygen provides to structure your documentation. If you're a newcomer to Doxygen this blogpost might be useful for you.

<!--more-->

In order to demonstrate Doxygen features I created a sample project. You can check out the project source code and the generated HTML ouput at:

* [Source code on GitHub](https://github.com/noseka1/structure-of-doxygen-doc "structure-of-doxygen-doc")
* [The generated HTML output](http://noseka1.github.com/structure-of-doxygen-doc/)

The tree view in the generated HTML output looks as follows:

{{< figure src="/images/posts/doxygen_treeview.png" class="center" >}}

## Doxygen pages

Pages in Doxygen are used for documentation that is not directly attached to the source code entity like class, file or member. They will typically contain a longer description of your project. You can refer to any source code entity from within the page if required.

There's always a project main page created by the Doxygen tag `@mainpage`. In our example, the title of the main page is `My Project`. All other pages listed under the main page are created using the Doxygen tag `@page`. In our example, we're using Markdown files where the `@page` tag is assumed and you're not required to write it. Doxygen automatically generates a page for every file with the `.markdown` extension. We talked about Markdown support in Doxygen in my [previous blogpost](/blog/2015/06/14/technical-documentation-with-doxygen/ "Technical Documentation with Doxygen").
The page hierarchy is created by the repetitive use of the Doxygen tag `@subpage`. The `@subpage` tag creates a parent-child relationship between two pages. It generates a reference (link) to the subpage at the same time. Example:

{{< highlight plaintext "linenos=table" >}}
# The List of subpages:

* Page @subpage subpage_1
* Page @subpage subpage_2
* Page @subpage subpage_3
{{< / highlight >}}

## Doxygen modules

A larger software project typically consists of multiple modules. A module implements a particular project functionality. Doxygen cannot know which classes, files or namespaces belong to which module in your project. If you want to structure your documentation based on modules you'll need to tag each class, file or namespace with the module name. This allows Doxygen to understand your modularized design and generate the documentation accordingly.

You can start with a definition of your modules and their parent-child relationship in a separate Doxygen file. A module is defined using `@defgroup` Doxygen tag. In our example, we're defining a parent module `My Project Modules` with two submodules `Math library` and `Misc library`:

{{< highlight-caption lang="c" linenos="table" title="doxygen/group_defs.dox" >}}
/**
 * @defgroup group_main My Project Modules
 * @brief All project modules
 *
 * @{
 */

/** @defgroup group_math Math library */
/** @defgroup group_misc Misc library */

/** @} */
{{< / highlight-caption >}}

Later in the source code, you can associate a class, file or a namespace with a module by using a Doxygen tag `@ingroup`:

{{< highlight-caption lang="c" linenos="table" title="src/math/factorial.h" >}}
#ifndef MYPROJECT_MATH_FACTORIAL_H
#define MYPROJECT_MATH_FACTORIAL_H

/** @ingroup group_math */
namespace myproject {
    namespace math {

        /** @ingroup group_math */
        int fact(int n);
    }
}
#endif
{{< / highlight-caption >}}

## Namespaces, classes and files

Doxygen automatically generates a list of namespaces, classes and files for you.
