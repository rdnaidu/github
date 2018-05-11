---
layout: single
classes: wide
title:  "Meanful parsing in Java with regular expressions"
date:   2018-05-10 17:40 -0500
tags:
  - Java
  - Regular Expressions
  - Parsing
categories: java regular-expressions
excerpt: "Using Java regular expressions how to get a more meanful result using names for capturing the groups?"
---

## Simples case
Many times developers find theirselves in the situation where they need to analyze a string and convert it into a more meanful structure for their project; that's basically what [parsing][parsing-meaning] is all about. In Java that smart structure is probably an object which is instanced with the values we have gathered before.

Most of the time the content we need to parse has a separator; specially if its a program output. For instance if we have a <abr title="Comma-separated values">CSV</abr> entry we can use a simple split and infer each property based on its position, format it to the correct type and then construct the final object; e.g.

```java
final String[] tokens = "Aemon Targaryen;104;dead".split(";");

String name  = tokens[0];
Integer age  = Integer.parseInt(tokens[1]);
Status status  = Status.valueOf(tokens[2]);

return new Person(name, age, status);
```

Sometimes we have to make multiple splits, maybe with other separators, but basically the complexity of this kind of parsing is simple; it just takes more steps.


## More complex scenario
How to deal with a more complex pattern? Most of the times these are sources meant to be just read just by humans; some others, they have a not so well delimited structure. To deal with such scenarios you build a dedicated parser or if you are lucky solve it using regular expressions.

While developing an OSS project called [MegaCmd for Java][megacmd4j-project] I have faced many console parsing situations. Some of which I solved using splits-like solutions, but there was others where I had to parse a String like:

```
megacmd4j/level2 (folder, shared as exported permanent folder link: https://mega.nz/#F!APJXXXiJ!lfKu3tVd8pNceYYYH6qe_tA)
```

In this scenario, I needed to parse from that command output the remote folder (megacmd4j/level2) and the public link to be shared with other users    (https://mega.nz/#F!APJXXXiJ!lfKu3tVd8pNceYYYH6qe_tA). Using splits were not an option and it took me some minutes before getting with this regular expression pattern to get what I wanted:

```regexp
(\S+) \(.+link: (http[s]?://mega.nz/#.+)\)
```

1. `\S+` allowed me to catch any content at the beginning of the text before the first space.
2. ` \(.+link: ` allowed to detect the expected space and the opening parenthesis as delimiter of the irrelevant content to come. The irrelevant content finishes when the prefix of the public link is found: *link:* ...
3. `(http[s]?://mega.nz/#.+)\)` expects at the beginning an *http* or *https* url that belongs to the domain *mega.nz* and has a subpath that starts with a *#*. Outside this group it expects a closing parenthesis.

So far so good. I expected to have as the first group (`0`) the whole string. So basically what I wanted were the group `1` and `2`.

Thats how I arrived to this pure function:

```java
public static final ExportInfo parseExportListInfo(String exportInfoLine) {
    try {
        final Matcher matcher = Pattern.compile("(\\S+) \\(.+link: (http[s]?://mega.nz/#.+)\\)")
                .matcher(exportInfoLine);

        if(matcher.find()){
            final String remotePath = matcher.group(1);
            final String publicLink = matcher.group(2);
            return new ExportInfo(remotePath, publicLink);
        }
    } catch (IllegalStateException | IndexOutOfBoundsException ex) {
        //Don't let it go outside
    }

    throw new MegaInvalidResponseException(exportInfoLine);
}
```

These tests for catching expected failures:

```java
@Test(expected = MegaInvalidResponseException.class)
public void given_invalid_remotePath_when_parseExportListInfo_then_fail() {
    //Given
    String entryWithInvalidRemotePath = "megacmd4j/level2(folder, shared as exported " +
            " folder link: https://mega.nz/#F!APJmCbiJ!lfKu3tVd8pNceLoH6qe_tA)";

    //When
    ExportInfo.parseExportListInfo(entryWithInvalidRemotePath);
}

@Test(expected = MegaInvalidResponseException.class)
public void given_invalid_publicLinkPrefix_when_parseExportListInfo_then_fail() {
    //Given
    String entryWithInvalidRemotePath = "megacmd4j/level2 (folder, shared as exported " +
            " folder content: https://mega.nz/#F!APJmCbiJ!lfKu3tVd8pNceLoH6qe_tA)";

    //When
    ExportInfo.parseExportListInfo(entryWithInvalidRemotePath);
}

@Test(expected = MegaInvalidResponseException.class)
public void given_invalid_ending_when_parseExportListInfo_then_fail() {
    //Given
    String entryWithInvalidRemotePath = "megacmd4j/level2 (folder, shared as exported " +
            " folder content: https://mega.nz/#F!APJmCbiJ!lfKu3tVd8pNceLoH6qe_tA";

    //When
    ExportInfo.parseExportListInfo(entryWithInvalidRemotePath);
}

@Test(expected = MegaInvalidResponseException.class)
public void given_publicLinkWithIncorrectMegaUrl_when_parseExportListInfo_then_fail(){
    //Given
    String responseWithInvalidMegaUrl =
            "megacmd4j (folder, shared as exported permanent " +
                    "folder link: http://mega.com/#F!APJmCbiJ!lfKu3tVd8pNceLoH6qe_tA)";

    //When
    ExportInfo.parseExportListInfo(responseWithInvalidMegaUrl);
}
```

... and these others to verify successful cases:

```java
@Test
public void given_remotePathWithSubPaths_when_parseExportListInfo_then_success() {
    //Given
    String responseWithRemotePathWithSubPaths =
            "megacmd4j/level2/level3 (folder, shared as exported permanent " +
                    "folder link: https://mega.nz/#F!APJmCbiJ!lfKu3tVd8pNceLoH6qe_tA)";

    //When
    final ExportInfo exportInfo =
            ExportInfo.parseExportListInfo(responseWithRemotePathWithSubPaths);

    //Then
    Assert.assertEquals("megacmd4j/level2/level3", exportInfo.getRemotePath());
    Assert.assertEquals(
            "https://mega.nz/#F!APJmCbiJ!lfKu3tVd8pNceLoH6qe_tA",
            exportInfo.getPublicLink()
    );
}

@Test
public void given_remotePathWithSingleFolder_when_parseExportListInfo_then_success(){
    //Given
    String responseWithRemotePathWithSingleFolder =
            "megacmd4j (folder, shared as exported permanent " +
                    "folder link: https://mega.nz/#F!APJmCbiJ!lfKu3tVd8pNceLoH6qe_tA)";

    //When
    final ExportInfo exportInfo =
            ExportInfo.parseExportListInfo(responseWithRemotePathWithSingleFolder);

    //Then
    Assert.assertEquals("megacmd4j", exportInfo.getRemotePath());
    Assert.assertEquals(
            "https://mega.nz/#F!APJmCbiJ!lfKu3tVd8pNceLoH6qe_tA",
            exportInfo.getPublicLink()
    );
}

@Test
public void given_publicLinkNotHttps_when_parseExportListInfo_then_success(){
    //Given
    String responseWithHttp =
            "megacmd4j (folder, shared as exported permanent " +
                    "folder link: http://mega.nz/#F!APJmCbiJ!lfKu3tVd8pNceLoH6qe_tA)";

    //When
    final ExportInfo exportInfo =
            ExportInfo.parseExportListInfo(responseWithHttp);

    //Then
    Assert.assertEquals("megacmd4j", exportInfo.getRemotePath());
    Assert.assertEquals(
            "http://mega.nz/#F!APJmCbiJ!lfKu3tVd8pNceLoH6qe_tA",
            exportInfo.getPublicLink()
    );
}
```

## Taking result arguments with names
So far so good, but in the previous implementaton for capturing groups we are using numeric indexes generated by default. Its not so bad, but by simply naming groups we could turn the capture of values more legible. For instance, it allows to skip the temporary variables while being clear about which parameter of the constructor we are passing. 

For adding a name to a group the formula is simple:
```
(?<name>X)
```
The name of the group must be always specified at the begining, prefixed by `?` and inside `<>`. In this case it will be called `name` . `X` is the regular expression to capture the content of that group, which is basically what it has been used so far.

This will allow us to change the code to something more verbose like:

```regexp
(?<remotePath>\S+) \(.+link: (?<publicLink>http[s]?://mega.nz/#.+)\)
```

Taking this to the code, it results in:

```java
public static ExportInfo parseExportListInfo(String exportInfoLine) {
    try {
        final Matcher matcher = Pattern.compile(
                "(?<remotePath>\\S+) \\(.+link: (?<publicLink>http[s]?://mega.nz/#.+)\\)"
        ).matcher(exportInfoLine);

        if (matcher.find()) {
            return new ExportInfo(
                    matcher.group("remotePath"),
                    matcher.group("publicLink")
            );
        }
    } catch (IllegalStateException | IndexOutOfBoundsException ex) {
        //Don't let it go outside
    }

    throw new MegaInvalidResponseException(exportInfoLine);
}
```
Tests can be run again to check everything is working as before; but this time we know which are the parameters we are passing to the constructor and the purpose of the groups that are been captured; all thanks to the flexibility of named groups in regular expressions in Java. 

## Conclusions
Regular expressions are a powerful tool that can be used in almost any modern language. Patterns for capturing groups can be tricky to be understand, but with the help of named groups we can make it more meanful for yourself in the future and for other developers.

## See more

* [Class Pattern](https://docs.oracle.com/javase/8/docs/api/java/util/regex/Pattern.html)
* [Regular expressions in Java - Tutorial](http://www.vogella.com/tutorials/JavaRegularExpressions/article.html)
* [Regular Expression Test Page for Java](http://www.regexplanet.com/advanced/java/index.html) Helpful for checking if your regular expression is working before taking it to code.

[parsing-meaning]: https://en.wikipedia.org/wiki/Parsing
[megacmd4j-project]: https://github.com/EliuX/Megacmd4J