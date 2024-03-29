+++
title = "Diagrams and Images in Doxygen"
date = "2015-06-28"
slug = "2015/06/28/diagrams-and-images-in-doxygen"
Categories = [ "development" ]
+++

Is your technical documentation hard to read? Diagrams and images liven up technical documentation and help the reader to better understand the subject. In the last article of the Doxygen miniseries we'll go over a couple of options how to include diagrams and images in Doxygen documentation.

<!--more-->

In order to show the graphical capabilities of Doxygen I created a sample project. You can check out the project source code and the generated HTML and PDF output at:

* [Source code on GitHub](https://github.com/noseka1/diagrams-and-images-in-doxygen "diagrams-and-images-in-doxygen")
* [The generated HTML output](http://noseka1.github.com/diagrams-and-images-in-doxygen)
* [The generated PDF output](http://noseka1.github.com/diagrams-and-images-in-doxygen/refman.pdf)

Doxygen on its own doesn't implement graphical operations. However, it can include diagrams and images generated by external tools.

# DOT graphs

The DOT language allows for simple definition of graphs. You can find a great documentation with many examples of DOT graphs in the manual [Drawing graphs with dot](http://www.graphviz.org/Documentation/dotguide.pdf "Drawing graphs with dot"). Doxygen tag `@dot` allows for embedding the DOT graph definition directly into your documentation. The nodes of the graph can be made hyperlinks as it is demonstrated in the sample project. Doxygen itself uses DOT graphs to generate the class inheritance and call graph diagrams. In order to generate the DOT diagrams you need to have `dot` utility installed. On most distributions the `dot` utility can be found in the `graphviz` package.

# MSC sequence diagrams

[Mscgen](http://www.mcternan.me.uk/mscgen/ "Mscgen") is a handy utility for generating sequence diagrams. On many Linux distributions you can find it in the `mscgen` package. Similarly to DOT graphs, the parts of the mscgen diagrams can be made clickable, too. In Doxygen, you can include a MSC diagram by using `@msc` tag.

# PlantUML diagrams

All kinds of UML diagrams can be created with [PlantUML](http://plantuml.sourceforge.net/ "PlantUML"). To learn about the different PlantUML features you can refer to the great reference guide [Drawing UML with PlantUML](http://plantuml.sourceforge.net/PlantUML_Language_Reference_Guide.pdf "Drawing UML with PlantUML"). PlatUML is implemented in Java. You'll need to download the jar file `plantuml.jar` and tell the jar location to Doxygen in your `Doxyfile`. The definition of a PlantUML diagram in Doxygen must be enclosed in the `@startuml` and `@enduml` tags.

# Dia diagrams

If you prefer drawing your diagrams directly instead of defining them as a plain text the [Dia Diagram Editor](http://dia-installer.de/ "Dia Diagram Editor") can be a good fit for you. On many Linux distributions come with a `dia` package. After you have created your dia file you insert it into the Doxygen documentation using the `@diafile` Doxygen tag.

# Inserting images

Last thing that our sample project illustrates is inserting images into Doxygen documentation. You'll need to have your images ready in several graphical formats (`.svg`, `.eps`, ...) depending on which Doxygen ouput you're willing to generate. For example, for *Latex* output the images must be provided in Encapsulated PostScript (`.eps`). Example of including the same image in multiple formats for *HTML* and *Latex* outputs:

{{< highlight plaintext "linenos=table" >}}
@image html people.svg
@image latex people.eps "People image" width=\textwidth
{{< / highlight >}}
