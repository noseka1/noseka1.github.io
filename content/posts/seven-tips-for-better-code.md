+++
title = "Seven Tips for Better Code"
date = "2015-06-06"
slug = "2015/06/06/seven-tips-for-better-code"
Categories = []
+++
It's a pleasure to read and maintain your code, right? Software practice teaches that code is written once but read many times. Your teammates will appreciate it if you put your effort into writing a good looking and understandable code. Needless to say that when you'll read your code one year later you'll appreciate it, too. Let's discuss a couple of basic guidelines to write a better code.

<!-- more -->

1) Respect the pre-existing coding style
----------------------------------------
You've just joined a software project that run for a while. The coding conventions used on the project are weird and you really don't like them. Perhaps, they force you to put parenthesis on a new line or let you write variable names with the first letter in capital while you'd prefer a small letter. They even violate the existing Java or C# language conventions. And the [Hungarian notation](http://en.wikipedia.org/wiki/Hungarian_notation "Hungarian notation") looks just awful!

As a professional software developer, you'll find yourself in this situation many times. Unless you can reformat the code of the entire project it's better to stick with the coding style that's already in place. In other words, you should follow the already established coding style regardless whether  you like it or not. Perhaps, you can propose a change of coding style to your teammates when you're starting to work on a new project.

2) Tabs versus spaces
---------------------
Is it better to use tabs or spaces to indent your code? Well, pick one but stick with it throughout your entire project. I prefer spaces because they let my code look the same in all editors. My editor is configured to convert tabs into spaces so that I don't have to tap on the space bar too much when indenting.

3) Break your code down into methods
------------------------------------

It's a good practice to break down a complicated program logic into multiple methods. By doing so, you make it easier for the reader to understand the purpose of your code. Look at the following code example. Can you explain what the code is doing?

{% codeblock lang:java %}
    // Bloated program logic, don't do this!
    String retrieveData(String key) {
        String data = cache.get(key);
        if (data == null) {
            BufferedReader br = null;
            try {
                br = new BufferedReader(new FileReader("file.txt"));
                data = br.readLine();
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                try {
                    if (br != null ) {
                        br.close();
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            cache.put(key, data);
        }
        return data;
    }
{% endcodeblock %}

The aim of the code is pretty simple. When retrieving data, look if the requested data is in the cache first and if not fetch it from the file. If the data was read from the file, put it into the cache to speed up the succeeding lookups. Let's rewrite the code example to make this logic apparent to the reader:

{% codeblock lang:java %}
    String retrieveData(String key) {
        String data = cache.get(key);
        if (data == null) {
            data = readDataFromFile(key);
            cache.put(key, data);
        }
        return data;
    }

    String readDataFromFile(String key) {
        String data = null;
        BufferedReader br = null;
        try {
            br = new BufferedReader(new FileReader("file.txt"));
            data = br.readLine();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (br != null ) {
                    br.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return data;
    }
{% endcodeblock %}

The caching logic is now separate from the details of how to read the data from the file. Splitting the logic up into two methods greatly increased readability.

4) Avoid excessive access to your instance variables
----------------------------------------------------

Object-oriented programming introduced a *class* abstraction. Class groups the data with the code that can modify the data. Instead of using global variables that are accessible from any part of your program you can encapsulate your data with the code into a class. The scope where your data is accessible is now limited to the class members. However, your data is accessible by *any* class member. Assuming a very large class definition with a lot of members the data members can start feeling as a new form of global variables. I'd like to suggest to you avoiding excessive access to your data members. Look at the following example:

{% codeblock lang:java %}
class Number {

    private int number = 0;
    private int increment = 1;

    public int increment() {
        incrementNumber();
        return number;
    }

    private void incrementNumber() {
        // direct access to instance variable "number". Not too good.
        number = number + increment;
    }
}
{% endcodeblock %}
In the code above the instance variable `number` is accessed from within two methods: `increment` and `incrementNumber`. Should it happen that the instance variable `number` is not set to the value you expected you'll need to review both methods because they're both accessing the problematic variable `number`.

The following version of the same code accesses the instance variable `number` only from within the method `increment`. The method `incrementNumber` is working only with a copy of the `number` variable created on the stack. When searching for a problem you'll start reviewing the `increment` method and possibly follow the data flow into the `incrementNumber` method when needed. Also, if you'd like to reuse the code of the method `incrementNumber` it's easier to do now. The method `incrementNumber` doesn't access any instance variables and so you can separate it out into a *class* method right away.

{% codeblock lang:java %}
class Number {

    private int number = 0;
    private int increment = 1;

    public int increment() {
        number = incrementNumber(number, increment);
        return number;
    }

    private int incrementNumber(int num, int inc) {
        // no access to instance variables
        return num + inc;
    }
}
{% endcodeblock %}

5) Prevent maintenance issues
-----------------------------
Look at the following code. Can you see a problem?

{% codeblock lang:java %}
    if (value + 1 <= 10)
        value++;
    else
        value = 0;
        log.error("Value overflow");
{% endcodeblock %}

The intention of the maintainer was to add an error message for the case that the value overflowed number 10. However, the code as it is written will log the error message in any case. When coding think about the future maintenance and use parenthesis to make very clear what the `if` and `else` parts are:
{% codeblock lang:java %}
    if (value + 1 <= 10) {
        value++;
    } else {
        value = 0;
    }
{% endcodeblock %}
When the maintainer tries to add an error message to this code he cannot introduce the bug illustrated in the previous code example. Out of curiosity, the Linux kernel project advocates no parenthesis around the if-else statement if not necessary. As always, the existing coding conventions take precedence.

6) Use braces to make expressions more readable
-----------------------------------------------
Programming languages define operator precedence and it's great that you're aware of it. Not every programmer has mastered the operator precedence and associativity table, though. Be kind and code your expressions very clearly. Braces can help you:

{% codeblock lang:java %}
// relying on operator precedence
if (a && b || a && c) {
    // ...
}

// making very clear how the expression is evaluated
if ((a && b) || (a && c)) {
    // ...
}
{% endcodeblock %}

7) Learn your programming language idioms
-----------------------------------------
There's no perfect programming language. Idioms get invented to work around the quirks of a specific programming language. Using idioms improves the maintanability of the code. You should learn and use them wherever possible. For Java idioms you can refer to [Java Idioms](http://c2.com/ppr/wiki/JavaIdioms/JavaIdioms.html "Java Idioms") and [Good Java Idioms](http://www.nayuki.io/page/good-java-idioms "Good Java Idioms").
