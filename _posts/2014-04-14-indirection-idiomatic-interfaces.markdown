---
layout: post
title:  "Indirection with Idiomatic Interfaces"
date:   2014-04-14 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
summary: "Interfaces. Simply a language ideal or a powerful use of indirection? In this article we will explore the uses of interfaces and how different languages use them. We will also review how golang can use them to achieve some dynamic freedom in a static world."
tags: interfaces, indirection, golang, Go, Java, ruby, Objective-C
---

Alright, "Indirection with Idiomatic Interfaces", may be a little over doing it, but I needed a couple of cool "i" words to string together with "interfaces". On a more definitive note, interfaces do actually provide a nice layer of indirection or what most programmers would probably call abstraction. For those newer to computer science, those terms can seem a little vague so let's define them:

**indirection:** indirectness or lack of straightforwardness in action, speech, or progression.

**abstraction:** the process of considering something independently of its associations, attributes, or concrete accompaniments

Those definitions are still a little vague, therefore, a better way to describe it in a computer science terms, is if something is abstract or indirect, we are trying to design a system that is flexible, reusable and extendable, which are all wonderful ideals when it comes to designing great software. Interfaces are a construct to provide us with just that. Since talk is cheap and code is what us problem solvers crave, I would like to provide some examples of interfaces in different languages. In my oh so humble opinion, I believe the best place to start is with C.

```cpp
#include <stdio.h>

/* our animal structure! */
typedef struct {
  void (*run)(void); /* da function pointer */
} Animal;

/* dogs like to run */
void dog_run(void)
{
  printf("running!!!\n");
}

/* horses like to gallop! */
void horse_run(void)
{
  printf("galloping!!!\n");
}

int main(int argc, char *argv[])
{
  /* create a dog */
  Animal dog = {
    .run = dog_run
  };

  /* create a horse */
  Animal horse = {
    .run = horse_run
  };

  /* let our animal's run away! */
  dog.run();
  horse.run();

  return 0;
}
```

Now I know what you are thinking, C doesn't have interfaces and that code is just a function pointer! To that, I would say you are correct! I believe C is important to include as it is the base on which all software is built. This is a simple little example, but here we can see that we create an animal structure called dog and assign it a run function. We also create a horse and assign it a run function. Now they both can call run and print different results for how these animals run. The upshot to the function pointer is it provides some nice indirection and it is quite fast. We will cover the performance advantages in another article. On the downside, it is not as flexible as we might like and has some clumsy syntax. This leads us to our next contender, Java!

```java
/* For animals that like to chew */
interface Chewable {
  public void chew();
}

/* For animals that can soar through the sky */
interface Flying {
  public void fly();
}

/* nice base animal class. Giving us some common methods for all animals. */
class Animal
{
  public void run() {
    System.out.println("running!!!");
  }
}

/* Dog class. Well dogs do chew. As we all probably know. */
class Dog extends Animal implements Chewable
{
  public void chew() {
    System.out.println("chewing away!");
  }
}

/* Bird class. Of course bird's don't chew. */
class Bird extends Animal implements Flying
{
  public void run() {
    System.out.println("running, psh try flying sometime!");
  }

  public void fly() {
    System.out.println("Look ma, I'm Flying!");
  }
}

/* a place for our animals to strech their legs */
class Yard
{
  public static void main(String[] args) {
    Dog dog = new Dog();
    Bird bird = new Bird();

    /* both can run from our base class */
    dog.run();
    bird.run();

    /* each implement their own interface for something specific to that animal */
    dog.chew();
    bird.fly();
  }
}
```

Java is definitely a champion of interfaces within an [OOP](http://en.wikipedia.org/wiki/Object-oriented_programming) environment and what many languages "interface" implementation is compared against. As we see, it has the same indirection as the C structure. It is providing us the same functionality using classes, but with another layer of indirection from our interfaces. Looking at the code, our interfaces are able to provide each animal with their own abilities, according to what they can do. Without interfaces, we would have been left to our own devices with some really creative subclassing. With interfaces, we are able to keep a real straightforward class design, but have specific animal functionality added. Another nice thing about interfaces are that they can assure us that any classes in the future that implement a certain interface will have those methods available. Now for those who are more versed in dynamic runtime languages, such as Ruby, Python, Objective-C, etc. this probably seems a little strange. Without getting too far off topic, (since we are going to cover them next [week](indirection-pragmatic-protocols.html) know that in these languages, pretty much all methods are executed at runtime, so the concept of interfaces is different. Unlike static languages that need to know that methods are a part of a class at compile time, dynamic languages just send a message to a class and if it does not respond, it throws an exception. This doesn't mean the type of indirection that interfaces are providing us is lost. Ruby has mixins, Python has multiple inheritance and Objective-C has it's protocols that allow us the same flexibility that interface provide. I will provide a few good articles I found for your reading pleasure, if you are so inclined. Now that we had a nice little primer to what interfaces are and how can use them, I want to talk about my most recent language endeavors that prompted me to write this article. Interfaces in Go, Go's interfaces were a little strange to me at first. Coming from a C/Objective-C/Ruby background I hadn't had much exposure to them other than the little bit of Java I have done in school assignments. Without further ado, here is our example Go code.

```go
package main

import "fmt"

//For animals that like to chew
type Chewable interface {
  Chew()
}

//For animals that can soar through the sky
type Flying interface {
  Fly()
}

//Our base Animal type
type Animal struct{}

func (animal *Animal) Run() {
  fmt.Println("running!")
}

//Now here is a Dog type with an embedded Animal type
type Dog struct {
  Animal
}

//Here is the Dog type implementing the Chewable interface
func (dog *Dog) Chew() {
  fmt.Println("chewing away!")
}

//Now here is the Bird type that also embeds the Animal type
type Bird struct {
  Animal
}

//Here the Bird is "overriding" the Animal type function
func (bird *Bird) Run() {
  fmt.Println("running, psh try flying sometime!")
}

//Bird's can Fly... by implementing the Flying interface
func (bird *Bird) Fly() {
  fmt.Println("Look ma, I'm Flying!")
}

//Good ole main
func main() {
  dog := Dog{} // different init style
  bird := new(Bird)

  //Both can run from our embedded animal type
  dog.Run()
  bird.Run()

  //each implement their own interface for something specific to that animal
  dog.Chew()
  bird.Fly()
}
```

As we can see the Go code looks a lot like our Java code. It has interfaces for the different animal types which they implement and call in our main. One thing to note about Go is that our interfaces are implicitly satisfied. In smaller words, this basically means that there is nothing saying that a type implements an interface. Notice this is different from Java in the fact that each Java class has to declare that it is going to use a certain interface and you have to implement the methods accordingly. In Go you just implement the methods. This opens up a really cool feature of Go, **empty interfaces**. Since every type in Go implements at least zero methods, this gives us what dynamic language people call `duck typing`. Ruby and Python are both known for using this typing, but Go still has the compiler to catch obvious mistakes. I think a good example of this is the `Printf` source code of Go. If we jump over the docs we see `Printf(format string, a ...interface{})`. Simply put, this method takes a format string and a variadic number of empty interface arguments. To those familiar with C's variadic arguments, this is a pretty normal thing, but to those less so, it basically means you can take a variable number of arguments. In C, `printf` is achieved through the use of a void pointer, which in many senses is what a empty interface is doing. If we dig through the `Printf` source, after we jump through the `Fprintf` method, we find the `doPrintf` method. There is a lot going on in there to build up a format string, but the part we are interested is `reflect.TypeOf(arg).String()`. Notice that a little bit above that, we are looping through our number of arguments and writing those into our string buffer based on the string value that is being returned from the `reflect.TypeOf().String()`. If we review the reflect package document we see that this `Typeof` method returns a `Type struct` and the String method converts that Type value to a string representation of that type. This way we can put it into our string buffer. I will include the documentation for these so you can give them a look. My illustration shows how Go is able to use the interface construct to provide us some dynamic power out of a generally static, compiled, language. This implementation is quite flexible and abstract (not to mention fast), making it a no brainer as to why Go has seen such rapid growth in it's very short lifetime.


[Objective-C Protocols](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/WorkingwithProtocols/WorkingwithProtocols.html)

[Python Multiple Inheritance](https://docs.python.org/2/tutorial/classes.html#multiple-inheritance)

[Ruby Mixin & Modules](http://www.tutorialspoint.com/ruby/ruby_modules.htm)

[Go Reflect Docs](http://golang.org/pkg/reflect/)

[Printf Source](http://golang.org/src/pkg/fmt/print.go?s=5879:5942#L1194)

[In depth review of Go Interfaces](http://research.swtch.com/interfaces)

[More on Go Interfaces](http://jordanorelli.com/post/32665860244/how-to-use-interfaces-in-go)

With that, we end our little journey through interfaces. As always feel free to hit me up on [Twitter](https://twitter.com/AC_Macalister)