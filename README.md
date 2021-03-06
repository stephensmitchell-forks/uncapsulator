﻿Uncapsulator
=============

Uncapsulator provides a fluent API for .NET reflection that lets you easily access private members of an object or type, by employing a dynamic proxy that implements `IDynamicMetaObjectProvider`.

Uncapsulator is also useful when you want to dynamically invoke (public) members of an interface. Ordinary dynamic binding is unreliable in this scenario, because it fails with explicitly implemented interface members. 

Uncapsulator is like [ReflectionMagic](https://github.com/ReflectionMagic/ReflectionMagic) on steroids, and is based on the built-in feature in [LINQPad](http://www.linqpad.net), written by Joseph Albahari.

## Getting Started

Uncapsulator can be downloaded via the NuGet package of the same name.

To get started, add the following imports to your source:

```csharp
using Uncapsulator
using static Uncapsulator.TypeUncapsulator
```

Then to reflect over an object, call Uncapsulate():

```csharp
static void Main()
{
    var demo = new Demo();
    int privateValue = demo.Uncapsulate()._private;
}

class Demo
{
    int _private = 123;
}
```

You can keep 'dotting' in to access more private members:

```csharp
static void Main()
{
    var demo = new Demo();
    int privateValue = demo.Uncapsulate().SomeProp.SomeMethod()._private;
}

class Demo
{
    int _private = 123;
    Demo SomeProp => new Demo();
    Demo SomeMethod() => new Demo();
}
```

## Unwrapping the value

In the end, you'll probably need to extract an underlying value.
You can do so with either an implicit or explicit cast:

```csharp
static void Main()
{
    var uncap = new Demo().Uncapsulate();
    
    DateTime thisWorks = uncap.Now;             // Implicit cast to DateTime
    var thisAlsoWorks = (DateTime) uncap.Now;   // Explicit cast to DateTime
    
    // If you don't know the type and just want a System.Object, call .ToObject():
    
    object obj = uncap.Now.ToObject();
}

class Demo
{
    private DateTime Now => DateTime.Now;
}
```
## Calling methods and indexers

Unencapsulator takes care of method overload resolution, numeric conversions, ref/out marshaling, optional parameters, and generic methods:

```CSharp
static void Main()
{
    // Calling an object's private methods is easy with Uncapsulate():
    var demo = new Demo().Uncapsulate();
    string result1 = demo.PrivateMethod (100);

    // The uncapsulator will perform implicit numeric conversions for you automatically:
    string result2 = demo.PrivateMethod ((byte)100);

    // It will also perform overload resolution:
    string result3 = demo.PrivateMethod ("some string");

    // ...even with ref and out parameters:
    demo.PrivateMethod (out string s);
    
    // Optional parameters are also supported:
    demo.OptionalParamMethod (100);
    
    // as are indexers:
    string result4 = demo[123]; 
    
    // You can also call generic methods, as long as you're able to specify type parameters:
    demo.GenericMethod<DateTime>();
}

class Demo
{
    string PrivateMethod (int x) => "Private method! (int) " + x;

    string PrivateMethod (string s) => "Private method! (string) " + s;

    void PrivateMethod (out string x) => x = "Passing by reference works!";
    
    void OptionalParamMethod (int x = 1, int y = 2, int z = 3)
        => Console.WriteLine (new { x, y, z }));
        
    string this [int index] => index.ToString();

    void GenericMethod<T>() => Console.WriteLine (typeof (T).FullName);
}
```

## Static members

You can access static members of a type via methods on the TypeEncapsulator class:

```CSharp
using Uncapsulator;
using static Uncapsulator.TypeUncapsulator;   // This makes the Uncapsulate function easy to call.

static void Main()
{
    string result1 = Uncapsulate<Demo>()._privateField;

    // If the type that you want to access is private, you can specify the type name as a string:
    string result2 = Uncapsulate ("Demo", Assembly.GetExecutingAssembly())._privateField;

    // Use the + symbol to denote a nested class:
    string result3 = Uncapsulate ("Demo+NestedPrivate", Assembly.GetExecutingAssembly())._privateField;

    // Or if the containing class is accessible:
    Uncapsulate<Demo>().NestedPrivate._privateField.Dump();
}

class Demo
{
    static string _privateField = "static private value";

    class NestedPrivate
    {
        static string _privateField = "static private nested value";
    }    
}
```

## Constructing a new object

Constructing a new object is just like calling a static member (see preceding sample) whose name is @new

```CSharp
static void Main()
{
    Demo myClass = Uncapsulate<Demo>().@new (1);
    
    // or if the type is inaccessible:
    var myClass2 = Uncapsulate ("Demo").@new (2);    
    
    // You can also use @new as an instance method (i.e., on an existing object).
    // It will then instantiate a new object of the same type.
    myClass.Uncapsulate().@new (3);
}

class Demo
{
    static string _privateField = "static private value";

    private Demo (int foo) => Console.WriteLine ("Private constructor! " + foo);
}
```

## Interfaces and casts

 To call an explicitly implemented interface method, first cast to the interface with CastTo()

```CSharp
static void Main()
{
    new Demo().Uncapsulate().CastTo<ISomeInterface>().Test();
    
    // OR:
    new Demo().Uncapsulate().CastTo (typeof (ISomeInterface)).Test();

    // If the interface is private, specify its name as a string:
    new Demo().Uncapsulate().CastTo ("ISomeInterface").Test();

    // To cast to a generic type, specify the type name with a backtick:
    int count = new int[123].Uncapsulate().CastTo ("IList`1").Count;
}

interface ISomeInterface       { void Test();                  }
class Demo : ISomeInterface    { void ISomeInterface.Test() {} }
```

Uncapsulate() is useful even with public interfaces, as an alternative to C#'s standard dynamic binding. This is because the latter does not let you call explicitly implemented members:

```CSharp
void Main()
{

    dynamic foo = new Demo();
    try                  { foo.Test();               }
    catch (Exception ex) { ex.Dump ("Not allowed!"); }
    
    // The same thing works nicely with Uncapsulate:
    var foo2 = new Demo().Uncapsulate();
    foo2.CastTo<ISomeInterface>().Test();     // Works!	
    foo2.CastTo("ISomeInterface").Test();     // Works!	
}

public interface ISomeInterface       { void Test();                  }
public class Demo : ISomeInterface    { void ISomeInterface.Test() {} }
```

## Calling members hidden by a subtype

Should a subclass and base class define members with the same name, the subclass will normally "win" and hide the base class's member. To access the base class member, use the special `@base` member:

```CSharp
static void Main()
{    
    var sub = new SubClass();
        
    string s1 = sub.Uncapsulate()._x;          // "subclass"
    string s2 = sub.Uncapsulate().@base._x;    // "base class"

    // You can also access the base member with a cast:	
    string s3 = sub.Uncapsulate().CastTo ("BaseClass")._x;
}

class BaseClass
{
    string _x = "base class";
}

class SubClass : BaseClass
{
    string _x = "subclass";
}
```

## Using LINQ

To query a collection of private objects, call ToDynamicSequence() on the collection and assign the result to IEnumerable<dynamic>:

```CSharp
static void Main()
{
    IEnumerable<dynamic> sequence = new Demo().Uncapsulate().Customers.ToDynamicSequence();
    
    // Now we have something that we can run LINQ queries over:    
    var query =
        from item in sequence
        orderby (string) item._lastName   // Remember to cast the elements!
        select item;    
}

class Demo
{
    Customer[] Customers => Enumerable.Range (0, 6).Select (_ => new Customer()).ToArray();

    class Customer
    {
        string _firstName = "Joe" + _random.Next (10);
        string _lastName = "Bloggs" + _random.Next (10);

        static Random _random = new Random();
    }
}
```

## Caching

If you repeatedly call the same method with the same argument types, the underlying reflection operations (including overload resolution) are cached.
    
This caching works as long as you either re-use the same uncapsulated instance, or call Uncapsulate with useGlobalCache:true.

```CSharp
Demo demo = new Demo();

void Slow()    // Calling Uncapsulate over and over will clear the cache
{
    for (int i = 0; i < 100000; i++)
        demo.Uncapsulate().SomeMethodCall (i, "test", DateTime.Now);
}

void Fast()    // Calling Uncapsulate just once means the cache will work
{
    var uncap = demo.Uncapsulate();
    
    for (int i = 0; i < 100000; i++)
        uncap.SomeMethodCall (i, "test", DateTime.Now);
}

void AlsoFast()    // Always caches if you call Uncapsulate(useGlobalCache:true)
{
    for (int i = 0; i < 100000; i++)
        demo.Uncapsulate (true).SomeMethodCall (i, "test", DateTime.Now);
}

class Demo
{
    void SomeMethodCall (int x, string s, DateTime d) { }
    void SomeMethodCall (string x, string s, DateTime d) { }
    void SomeMethodCall (char x, string s, DateTime d) { }
    void SomeMethodCall (int x, string s, TimeSpan ts) { }
    void SomeMethodCall (bool x, string s, DateTime d) { }
    void SomeMethodCall<T> (bool x, T s, DateTime d) { }
    void SomeMethodCall (ref bool x, string s, DateTime d) { }
}
```

The static cache can be cleared as follows (to release memory and allow ALCs to unload):

```CSharp
Uncapsulator.Extensions.ClearCache();
```