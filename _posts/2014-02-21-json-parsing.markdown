---
layout: post
title:  "JSON Serializing with a Dynamic Runtime"
date:   2014-02-21 08:00:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
summary: "Despite the addition of NSJSONSerialization in iOS 5 and OSX 10.7, a lot of boilerplate can go into mapping the Foundation objects values to proper object properties. Lucky for us, there is a dynamic runtime to help us out."
tags: JSON, foundation, serialize, parse, NSJSONSerialization
---

JSON is all the rage nowadays. It seems every major SaaS, framework, and language supports JSON and Objective-c is no exception. [Programmable Web](http://blog.programmableweb.com/2011/05/25/1-in-5-apis-say-bye-xml/) reported in 2011 that 1 in 5 new APIs are designed with JSON only. 

![](http://blog.programmableweb.com/wp-content/jsononly.png)

Fast forward to 2014, clearly JSON has become the dominant format. NSJSONSerialization gives us a great start, yet still a lot of boilerplate can go into creating and mapping JSON to proper objects. Let's take this JSON as an example: 

{% highlight javascript %}
{
    "id" : 1
    "first_name": "John",
    "last_name": "Smith",
    "age": 25,
    "address": {
        "id": 1
        "street_address": "21 2nd Street",
        "city": "New York",
        "state": "NY",
        "postal_code": 10021
     }

}
{% endhighlight %}  

Now we would want to map this to a model class to the tune of:

{% highlight objective-c %}

@interface Address : NSObject

@property(nonatomic,strong)NSNumber *objID;
@property(nonatomic,copy)NSString *streetAddress;
@property(nonatomic,copy)NSString *city;
@property(nonatomic,copy)NSString *state;
@property(nonatomic,strong)NSNumber *postalCode;

@end

@interface User : NSObject

@property(nonatomic,strong)NSNumber *objID;
@property(nonatomic,copy)NSString *firstName;
@property(nonatomic,copy)NSString *lastName;
@property(nonatomic,strong)NSNumber *age;
@property(nonatomic,strong)Address *address;

@end

{% endhighlight %}  

Now a normal boilerplate heavy way this would be accomplished would be like so:

{% highlight objective-c %}

//data is a NSData version of our JSON string above.
NSError *error = nil;
NSDictionary* response = [NSJSONSerialization JSONObjectWithData:data options:0 error:&error];
User *john = [[User alloc] init];
john.objID = response[@"id"];
john.firstName = response[@"first_name"];
john.lastName = response[@"last_name"];
john.age = response[@"age"];
//now for the address
john.address = [[Address alloc] init];
NSDictionary* address = response[@"address"];
john.address.objID = address[@"id"];
john.address.streetAddress = address[@"street_address"];
john.address.city = address[@"city"];
john.address.state = address[@"state"];
john.address.postalCode = address[@"postal_code"];

{% endhighlight %}  

that is a **LOT** of boilerplate and error prone due to the many string key access. Thanks to the dynamic runtime we can eliminate this completely. First we need to get the properties of the User object to know which JSON keys we need to map to the User object's properties.

{% highlight objective-c %}
//need to import to do some runtime work
#import <objc/runtime.h>

-(NSArray*)getPropertiesOfClass:(Class)objectClass
{
    unsigned int outCount, i;
    objc_property_t *properties = class_copyPropertyList(objectClass, &outCount);
    NSMutableArray *gather = [NSMutableArray arrayWithCapacity:outCount];
    for(i = 0; i < outCount; i++)
    {
        objc_property_t property = properties[i];
        NSString* propName = [NSString stringWithUTF8String:property_getName(property)];
        const char *type = property_getAttributes(property);
        
        NSString *typeString = [NSString stringWithUTF8String:type];
        NSArray *attributes = [typeString componentsSeparatedByString:@","];
        NSString *typeAttribute = [attributes objectAtIndex:0];
        
        if ([typeAttribute hasPrefix:@"T@"] && [typeAttribute length] > 3)
        {
            NSString * typeClassName = [typeAttribute substringWithRange:NSMakeRange(3, [typeAttribute length]-4)];  //turns @"NSDate" into NSDate
            Class typeClass = NSClassFromString(typeClassName);
            if(!self.propertyClasses)
                self.propertyClasses = [[NSMutableDictionary alloc] init];
            [self.propertyClasses setObject:typeClass forKey:propName];
        }
        [gather addObject:propName];
    }
    free(properties);
    if([objectClass superclass] && [objectClass superclass] != [NSObject class])
        [gather addObjectsFromArray:[self getPropertiesOfClass:[objectClass superclass]]];
    return gather;
}
{% endhighlight %} 

Let's break down what is happening. Objective-C is a dynamic language as previously mentioned. This differs from more common languages like C++ or Java. We could spend a whole article (and probably will!) on the differences between dynamic and static languages, but for now let's just settle on a 10,000 foot explanation. A dynamic language is where method execution is done at runtime. This means a method does not technically have to exist before we start running the application. Due to this design, we get a lot of power to easily inspect the object at runtime and find what properties it has. If you want to learn more about the runtime, [Mark Dalrymple](https://twitter.com/borkware) at [Big Nerd Ranch](http://bignerdranch.com) has a great set of [articles](http://blog.bignerdranch.com/2833-inside-the-bracket-part-1-open-for-business/) that explain this subject in depth.

Now for the code. `objc_property_t` is a C structure which as the header comments define as: 
{% highlight objective-c %}
An opaque type that represents an Objective-C declared property.
{% endhighlight %} 
`class_copyPropertyList` returns us a C array of these structures. We next iterate over them with a standard for loop and process each structure in the array. First we get the name of the property via `property_getName`. Next we pull the attributes of the property via `property_getAttributes` and separate them out to get the ones we want. We are primarily interested in the first attribute which we can use to get the class of the property. Using NSClassFromString we can convert the string name of the class into the class representation. Lastly we clean up our C array with `free()` and check if we need to get our super class(es) properties.

Ok great, now we have both the properties names and their classes. Let's our list looks something like this:

{% highlight objective-c %}
objID =  NSNumber
firstName = NSString
lastName = NSString
age = NSNumber
address = Address
{% endhighlight %}

armed with information we can easily map the JSON object to our User object and even validate that our properties classes are the expected type from the JSON. The only thing left we need to do is match the different naming conventions of JSON and Objective-C, so the snake casing of JSON matches the camel case of Objective-C. That can be accomplished like so:

{% highlight objective-c %}
+(NSString*)convertToJsonName:(NSString*)propName start:(NSInteger)start
{
    NSRange range = [propName rangeOfString:@"[a-z.-][^a-z .-]" options:NSRegularExpressionSearch range:NSMakeRange(start, propName.length-start)];
    if(range.location != NSNotFound && range.location < propName.length)
    {
        unichar c = [propName characterAtIndex:range.location+1];
        propName = [propName stringByReplacingOccurrencesOfString:[NSString stringWithFormat:@"%c",c]
                                                       withString:[[NSString stringWithFormat:@"_%c",c] lowercaseString]
                                                          options:0 range:NSMakeRange(start, propName.length-start)];
        return [self convertToJsonName:propName start:range.location+1];
    }
    return propName;
}
{% endhighlight %} 

This method takes our property name of `FirstName` and converts it to `first_name` so that when we search in the NSDictonary returned from NSJSONSerialization we will find the correct JSON key name. After that it is a simple loop to get the values matched up:

{% highlight objective-c %}
User *john = [[User alloc] init];
for(NSString* propName in propArray) //propArray is a NSArray of our property names
{
  NSString* jsonName = [self convertToJsonName:propName];
  id value = jsonDict[jsonName]; 
  [john setValue:value forKey:propName];
}
{% endhighlight %} 

much better. All we had to do we loop through our property array and convert the property name to the JSON name and use KVC to set the property value. The boilerplate has been eliminated and our code is now much less error prone. This example left out out how get the child object of address into it's proper object, but that is fairly straight forward with a little recursion. If you are interested in seeing that implementation I wrote a library that does this. You can check out [JSONJoy here](https://github.com/daltoniam/JSONJoy). With JSONJoy the same example from above can be shorten to a single line.

{% highlight objective-c %}
//responseObject is a NSDictonary from NSJSONSerialization or a NSString of the JSON content.
User *john = [User objectWithJoy:responseObject];
//JSON is fully parsed and mapped to our john User object.
{% endhighlight %} 

As you can see, with a bit of creativity and powerful dynamic runtime, we can take a common, mundane, and error prone task, and turn it into a concise and joyful experience. JSON never looked so good!

Thoughts? Questions? Feedback? Hit me up on Twitter at [@daltoniam](http://twitter.com/daltoniam). 


