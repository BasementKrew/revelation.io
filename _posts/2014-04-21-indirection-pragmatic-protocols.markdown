---
layout: post
title:  "Indirection with Pragmatic Protocols"
date:   2014-04-21 08:00:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
author_image: "http://www.gravatar.com/avatar/2fdc7b889f35118a7334187b15c5b957.png?r=x&amp;s=320"
summary: "Mixins and abstract classes and protocols, oh my! This week we cover indirection from the dynamic side of the house and see how it measures up against it's static cousins in the world of interfaces."
tags: interfaces, indirection, golang, Go, Java, ruby, Objective-C, objc, python, protocols, mixin, modules, abstract
---

Following up on our interface article from last week (check it out [here](indirection-idiomatic-interfaces.html) if you missed it), today we cover the dynamic cousin of interfaces, protocols. Now it is important to note that in this article we use the term "protocol" to encompass all the forms of indirection in different dynamic languages. Ruby has mixins and modules, Python has abstract classes and multiple inheritance and Objective-C has protocols. Now those are a lot of terms, to keep things clear we will just refer to all of them as protocols (because I am a Objective-C guy!).

Alright, now that we got our vernacular covered, what is the difference between an interface and a protocol? Both provide a form of indirection which allows us to do some pretty powerful abstraction as we outlined last week. The major difference is due to the runtime of these dynamic languages. Dynamic languages delay method binding until runtime. This means methods aren't required by the compiler to exist before running. It will just send a message (which is the method to run) to the object and expects it to respond. We can of course add checks to make sure that object responds before dispatching the message (that would be just irresponsible otherwise). This delay gives us quite a bit of flexibility and makes creating indirection in dynamic languages very simple. Onward to the examples!

Here is a ruby module

``` ruby
#hmmm, do these examples look familiar..?
#For animals that like to chew.
module Chewable
   def chew
    puts "chewing away!"
   end
end

#For animals that can soar through the sky.
module Flying
   def fly
    puts "Look ma, I'm Flying"
   end
end

#Nice base animal class. Giving us some common methods for all animals.
class Animal
  def run
    puts "running!!!"
  end
end

#Dog class. Well dogs do chew. As we all probably know.
class Dog < Animal
  include Chewable
end
#Bird class. Of course bird's don't chew.
class Bird < Animal
  include Flying
  def run
    puts "running, psh try flying sometime!"
  end
end

#Create a dog and bird.
dog = Dog.new
bird = Bird.new

#Both can run.
dog.run
bird.run

#Each are running their respective module methods.
dog.chew
bird.fly
```

Pretty simple, eh? This is the same example we used for our interface article last week. Note that the code does not need or have the concept of a contract of methods that will be fulfilled. Simply just include the module in the class and viola that class can now use those methods. This example shows the simplest and most flexible (it is Ruby after all!) use of a dynamic runtime. Next up Python!

``` python

#For animals that like to chew.
class Chewable(object):
   def chew(self):
    print "chewing away!"

#For animals that can soar through the sky.
class Flying(object):
   def fly(self):
    print "Look ma, I'm Flying"

#Nice base animal class. Giving us some common methods for all animals.
class Animal(object):
  def run(self):
    print "running!!!"

#Dog class. Well dogs do chew. As we all probably know.
class Dog( Animal, Chewable):
  pass

#Bird class. Of course bird's don't chew.
class Bird(Animal, Flying):
  def run(self):
    print "running, psh try flying sometime!"

#Create a dog and bird.
dog = Dog()
bird = Bird()

#Both can run.
dog.run()
bird.run()

#Each are running their respective class methods.
dog.chew()
bird.fly()
```

Python's abstract classes and multiple inheritance is a nod to C++. Multiple inheritance is not personally one my favorite implementation of class design (to be clear, I don't dislike Python as a whole, it a solid language), I believe it is important to showcase the different ways protocols are implemented. Now on to my boy Objective-C!

```objc

//For animals that like to chew.
@protocol Chewable <NSObject>

-(void)chew;

@optional
-(void)bark;

@end

//For animals that can soar through the sky.
@protocol Flying <NSObject>

-(void)fly;

@end

//Nice base animal class. Giving us some common methods for all animals.
@interface Animal : NSObject

-(void)run;

@end

@implementation Animal

-(void)run
{
  NSLog(@"running!!!");
}

@end

//Dog class. Well dogs do chew. As we all probably know.
@interface Dog : Animal<Chewable>

@end

@implementation Dog

-(void)chew
{
  NSLog(@"chewing away!");
}

@end

//Bird class. Of course bird's don't chew.
@interface Bird : Animal<Flying>

@end

@implementation Bird

-(void)run
{
  NSLog(@"running, psh try flying sometime!");
}

-(void)fly
{
  NSLog(@"Look ma, I'm Flying!");
}

@end

//Using a OS X app delegate, just to give OS X some love!
- (void)applicationDidFinishLaunching:(NSNotification *)aNotification
{
  //Create a dog and bird.
  Dog *dog = [[Dog alloc] init];
  Bird *bird = [[Bird alloc] init];

  //Both can run.
  dog.run();
  bird.run();

  //Each running something specific to that animal, via their protocol.
  dog.chew();
  bird.fly();
}
```

Notice how Objective-C is much more like an interface. It, like other dynamic languages, does not require you to have a method defined to make a method call, but the protocol helps define what the expected use case is to the complier (and other programmers!). Also note that we have an optional method of `bark()` that we didn't implement (poor dog). This is a simple example of how powerful a dynamic runtime is. One important side note, Objective-C is a complied language unlike Ruby and Python, so a little more goes into creating a protocol.


I bet you might be asking "why don't we just do this in every language?". It is the age-old discussion between flexibility and speed. Protocols are generally much slower than interfaces, so it is a matter of your needs. Every language is a tool in your toolbox. All tools are designed with a purpose in mind and have strengths and weakness that make each useful. We all have tools that we prefer to use, but at the end of the day, the important part of solving a problem is using the right tool for the task at hand. It is up to us as computer scientists to make sure we choose the right one.

As always, questions, comments, feedback, and random emojis expressing your thoughts on this article, are appreciated.

Twitter: [@daltoniam](https://twitter.com/daltoniam).

