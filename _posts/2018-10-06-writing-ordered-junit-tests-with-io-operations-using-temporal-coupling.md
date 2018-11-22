---
layout: single
classes: wide
title:  "Writing ordered JUnit tests for IO operations using temporal coupling"
date:   2018-05-10 17:40 -0500
tags:
  - Java
  - JUnit
  - Tests
  - Temporal Coupling
  - IO Tests
 
categories: java tests
excerpt: "How to assure your tests are going to be executed in a particular order so you take
 advantage of previously generated states? Lets see how temporal coupling can be useful for IO tests."
comments: true
---

## When temporal coupling is a necessary evil
While writing a library or a core part of a system that relies mostly in IO operations you might need that the 
state of the previous IO operations works as setup for the tests that follows. Nonetheless, its good to mention that 
every test should be independent from each other and at the beginning of it you can setup certain state if needed. Also 
its important to point out that [mocks][1] are the solution for test IO operations for most scenarios. But, when you are
creating an IO library and you really want their codebase be reliable you really must automate tests to try out its
functionality in real use case scenarios; e.g.

> When delete a remote cloud file check that only that file was deleted and not a folder with the same name.

 Before that test you probably had to create that folder and also created that file. So for doing so you have 
 two choices:
 
 1. Independent tests: Create a test for creating a folder and other for creating a file. Then our aforementioned 
 test that does what the other 2 before do and something else. Most of test libraries like JUnit fail tests independently, 
 so when the test for creating the file fails then all others that do that same functionality fail as well. In such case, 
 you have to figure out that the failing feature is the one related to the simplest test of all failing ones: because it 
 has the minimum common functionality among them, so that functionality must be what is failing, i.e. creating a file. 
 Nonetheless that minimum common functionality is not very clear in some contexts, mostly because there might not be a 
 single test just for that failing functionality.
 
 2. Temporal coupled tests: Create tests in order; so you first create a file, then create a folder with the same name 
 and for the third you want to test that the function `deleteFile` only deletes the file and not the folder (which on 
 purpose has that same name). If you make these tests to run sequentially, despite the fact that tests might continue 
 running although previous ones have failed, you can pretty much understand in your unit test logs which functionality 
 might be the one that causes the failure. E.g.
 
 ```txt
 test_create_file_A                                     [FAILED]
 test_create_folder_A                                   [Ok]
 test_delete_file_A_should_not_delete_folder_A          [FAILED]
``` 

So in this case you might quickly suspect that what is failing is `test_create_file` because you can see it was the first
who was executed. If that is the case, when you fix that functionality the test `test_delete_file_should_not_delete_folder` 
will be fixed as well. If `test_delete_file_should_not_delete_folder` keeps failing then you know its core functionality 
is the one failing because it has no precedent failing tests. This approach has some advantages in comparison to the 
independent test one:
1. The code for each tests are smaller, so they have less boilerplate, are easier to understand and run faster.
1. When IO operations have a cost, e.g. PaaS operations in the cloud use to have cost, then you are saving money or 
keeping your project
in the free layer threshold.
1. Its easier to understand that the first failing test is buggy because it has no predecessor failing tests and probably 
is the cause of subsequent failing tests.

> Note: This situation, where the order of execution matters, is refereed by Uncle Bob as temporal coupling. He recommends
 a technique called "pass the block" to deal with it, but it applies to cases when before/after the core functionality 
 the actions are well established. e.g. open and close a session. Sadly in our case the tests depends on previously 
 generated states, where most of the previous functions are involved in.

## How to do temporal coupling in JUnit tests in a way that makes sense
Being that said, lets see how we can do temporal coupled tests using JUnit/Java the best way possible.

```java
import org.junit.*;
import org.junit.runners.MethodSorters;

@FixMethodOrder(MethodSorters.NAME_ASCENDING)                               //#1
public class BasicActionsTest {

    @BeforeClass                                                            //#2
    public static void setup() {
        //Your setup login. 
        //E.g. Login in a Cloud storage provider
    }

    @AfterClass                                                             //#2
    public static void logout() {
        //Close that session
    }

    @Test
    public void stage00_test_create_fileA() {                               //#3
        ...
    }

    @Test
    public void stage00_test_create_folderA() {                             //#4
        ...
    }
    
    @Test
    public void stage01_test_delete_fileA_should_not_delete_folderA(){      //#5
        ...
    }
}
```
Lets break this down:

1. `@FixMethodOrder(MethodSorters.NAME_ASCENDING)`: It guarantees that tests are going to be executed in alphabetical 
order. Thanks to this we don't have to add any additional test dependency to do what we need; junit will be enough.
To do so we attach a prefix that assures the wanted alphabetical order, i.e. `stage_xx_`, so whatever be the test that 
follows it will be executed following that order. 
2. `@BeforeClass` and `@AfterClass` allows you to run code at the beginning and at the end of the unit tests. E.g.
starting a session and closing it in a third party service your library works with.
3. Functionality that checks if you can create a file. 
4. Functionality that checks if you can create a folder.
5. Functionality that checks if after deleting that file A the folder A still exists.

Things to have in count:
* The letter `A` is used to refers the previously created resources and the fact they are named the same.
* Test #3 and #4 have both the same prefix `stage00_` which means that it doesn't matter the order between those 2 
functions, they are in the same stage and can even work in parallel (although JUnit wont do so).
* Test #5 will be executed at last for having `stage001_` as prefix, which is ordered after the tests which 
have prefixed with `stage00_`; so if one of the previous tests fails it fails as well.

You must be thinking right now. What if you want to add a new tests that runs between `stage00_` and `stage01_`? Should
I have to rename those after the `stage00_` to an upper number, i.e. rename `stage01_` to `stage02_`, etc.? 

No, that is a terrible idea, because that number of subsequent tests to rename can be really large and that's not good 
for your health.
Imagine that you want to check the functions `fileExists` and a `folderExists`. The easier place would be after the 
`stage00_` tests, you can do it like this.

  
```java
import org.junit.*;
import org.junit.runners.MethodSorters;

@FixMethodOrder(MethodSorters.NAME_ASCENDING)                               
public class BasicActionsTest {

    @BeforeClass                                                            
    public static void setup() {
        //Your setup login. 
        //E.g. Login in a Cloud storage provider
    }

    @AfterClass                                                            
    public static void logout() {
        //Close that session
    }

    @Test
    public void stage00_test_create_fileA() {                              
        ...
    }

    @Test
    public void stage00_test_create_folderA() {                         
        ...
    }
    
    @Test
    public void stage001_test_fileA_exits() {                              
        ...
    }

    @Test
    public void stage001_test_folderA_exists() {                         
        ...
    }
    
    @Test
    public void stage01_test_delete_fileA_should_not_delete_folderA(){
        ...
    }
}
```

These new tests:
* Did not required any change or renaming in the previously existent tests.
* Will run after the stage `stage00` and before the `stage01`.
* If they fail and the `stage00` tests don't fail it means that what is failing is the code for checking if a file
exists and probably `stage01_test_delete_fileA_should_not_delete_folderA` will fail as well because it checks that
`folderA` exists.
* If they fail and some of the `stage00` tests fail as well it means that the problem is in one of `stage00` tests and 
needs to be fixed.

## Conclusions
Temporal coupled tests are not the way of normally create tests or any code whatsoever. Try to always avoid temporal 
coupling and if its needed use the strategy of "passing the block". But there are cases like IO libraries where the 
whole key of the code is to do IO operations and mocking that code will give you a really bad code coverage. In such 
cases use this ordered test strategy and you can be sure your library or project will work as intended. If your code is 
used as a dependency in another project, that project will be able to mock all the functionality you created because 
you already took care of it.

## See more
 * A [real unit test](https://github.com/EliuX/MEGAcmd4J/blob/master/src/test/java/com/github/eliux/mega/cmd/BasicActionsTest.java) 
 for a library I created for the Mega storage services.
 * [Coupling in Computer Programming](https://en.wikipedia.org/wiki/Coupling_(computer_programming)). 
 * ["Clean Code" Robert C. Martin notes](https://monami555.github.io/2016/01/13/clean-code-robert-c-martin-notes.html). 
 Nonetheless search more about temporal coupling because all really good  references and videos about it from Uncle Bob 
 have copyright. I personally recommend the Clean Code course you can find in Safari Books which looks pretty much the 
 same as [Function Structure: Clean Code, episode 4](https://cleancoders.com/episode/clean-code-episode-4/show).
 * [Avoid temporal Coupling](http://we-press-buttons.blogspot.com/2016/04/avoiding-temporal-coupling-part-12.html)
 
 [1]: http://www.vogella.com/tutorials/Mockito/article.html