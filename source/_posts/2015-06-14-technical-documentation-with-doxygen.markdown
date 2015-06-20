---
layout: post
title: "Technical Documentation with Doxygen"
date: 2015-06-14 22:04:36 -0700
comments: true
categories: development
---

Do you create project documentation in your company's internal wiki? I did it for quite some time until I realized that a good old Doxygen combined with Git can do a much better job.

<!-- more -->

[Doxygen](http://www.stack.nl/~dimitri/doxygen/ "Doxygen") is a well-known tool for generating documentation directly out of the annotated source code. However, it can be utilized to create technical documentation, too. In version 1.8.0, [Markdown support](http://www.stack.nl/~dimitri/doxygen/manual/markdown.html "Markdown support") was introduced to Doxygen. [Markdown](http://daringfireball.net/projects/markdown/ "Markdown") is a popular markup language. Plain text files written in Markdown format can be converted into HTML or many other formats. With Markdown support, creating technical documentation in Doxygen gets really easy.

Doxygen sample project
----------------------

To get started, I read a [very useful article](http://svenax.net/site/2013/07/creating-user-documentation-with-doxygen/ "Creating user documentation with Doxygen") describing how to enable support of plain Markdown files in Doxygen and a couple of gotchas one might encounter. Gotchas? No worries! In record time I was able to create sample technical documentation. As sample content I used a couple of articles from my blog which were also coded in Markdown. Now I'd like to share the project with you:

* [Source code on GitHub](https://github.com/noseka1/tech-doc-with-doxygen "tech-doc-with-doxygen")
* [The generated HTML output](http://noseka1.github.com/tech-doc-with-doxygen/)
* [The generated PDF output](http://noseka1.github.com/tech-doc-with-doxygen/refman.pdf)
* [The generated RTF output](http://noseka1.github.com/tech-doc-with-doxygen/refman.rtf)

The project shows a basic document structure. If you want to learn more about Doxygen I recommend downloading the Doxygen source distribution [tarball](http://www.stack.nl/~dimitri/doxygen/download.html) or clone the Doxygen's project [Git repository](https://github.com/doxygen/doxygen). After extracting the tar archive go to the `doc` directory where you can find the sources of the Doxygen's [webpage](http://www.stack.nl/~dimitri/doxygen/index.html "Doxygen"). You can learn a lot by reading through them.

Doxygen provides good support for creating graphs and diagrams in your technical document. For instance, you can insert graphs in [DOT](https://en.wikipedia.org/wiki/DOT_(graph_description_language "DOT") format with `@dot` or `@dotfile` tags. You can include [MSC](http://www.mcternan.me.uk/mscgen/ "MSC") sequence diagrams with `@msc` or `@mscfile` tags. To insert decent looking [PlantUML](http://plantuml.sourceforge.net/ "PlantUML") UML diagrams use `@startuml` tag. If you like to draw your diagrams in [Dia](http://dia-installer.de/ "Dia") you can insert them to Doxygen via `@diafile` tag.

Conclusion
----------

Doxygen is a very powerful documentation tool. Chances are that you're already using it to generate a documentation from your source code. It makes a lot of sense to me to use Doxygen for project documentation instead of creating it in the Wiki. If you combine Doxygen with a good source control (e.g. Git) you gain the following benefits:

* History tracking and branching that no Wiki can provide
* The same development workflow for your documentation as well as for your source code
* Project documentation and source code at one place and possibly referencing each other
