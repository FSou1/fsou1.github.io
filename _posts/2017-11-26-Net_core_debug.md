---
title: Debugging .NET Core CLR afterword
tags: .NET Core
redirect_from: "/Net_core_debug/"
---

Some time ago I decided to sort out internal details of .NET Core CLR and today I'd like to share with you the most surprising things that I found during this simple investigation.

## Starting to debug .NET Core CLR source codes is pretty easy

Microsoft took care of developers who are interested in development of their products. The repository documentation is not only very detailed and well-designed but also keeps up to date and contains lots of how-to information.

Depending on the developer's OS it contains particular build instructions and requirements ([this one](https://github.com/dotnet/coreclr/blob/master/Documentation/building/windows-instructions.md) is for Windows for example). As you could notice there are IDE and necessary component requirements, secondary packages and environment setup which should be performed for a successful building.

The building process itself is pretty straightforward and contains the only one step: execute `build.cmd` with optional arguments. Detailed [debugging instructions](https://github.com/dotnet/coreclr/blob/master/Documentation/building/debugging-instructions.md) on a different OS could also be found on a separate page.

In my case the building process took about 40 minutes (including tests) and didn't contain any issues.

## Implicit relationship between C++ and C# in memory layout

Once I've found it I was really surprised but some time later it became really obvious. For a better explanation I'd like to show you the code:

![csharp_cpp_memory_relationship](/assets/images/posts/csharp_cpp_memory_relationship.png)

Have you also already realized it? The left part of the code (C#) has exactly the same memory layout as the right one (C++). And if we think for a while actually there is nothing to wonder. Furthermore we gain some great opportunities which I'd like to bring it up shortly (we need to go deeper).

## Two-way interaction

I won't go into low-level details how a call actually works, but instead I'd like to provide a small part of a diagram which confirms the current thesis:

![csharp_cpp_two_way](/assets/images/posts/csharp_cpp_twoway.png)

The left part contains C++ code which is used for the application domain setup. During this process CoreRun initializes an instance of `AppDomain`'s type and makes a `setupDomain.Call` which passes an execution control to C#'s `AppDomain` implementation. When the C# part of work will be completed the execution control passes once again to C++ with the `nCreateContext()` method call.

As I've already mentioned before the root cause of this opportunity is the same memory layout (probably, it's better to say 'exactly the same mapping between the type design and the memory structure regardless programming language'). In reality we allocate the same size of memory and every property or method has the same offset in both languages. Why shouldn't it be possible to make such kind of calls? It's quite real and obvious.

## C# access modifiers do not affect availability from outside

Once again an attentive reader was able to figure out the confirmation of this thesis with the previous image. Haven't you noticed it yet? Ok, I'm going to clarify a little bit:

```csharp
private static object PrepareDataForSetup(
    String friendlyName,
    AppDomainSetup setup,
    string[] propertyNames,
    string[] propertyValues)
{
    //..
}
```

The signature of this method contains the access modifier `private` but it's not a barrier to make a call from C++ part. And once again it becomes quite obvious if we start to think from memory point of view and pointers management (actually it's also even available to call `private` method from a C# via reflection but this is a completely different story).

## Contributing to Microsoft repository is easier than you think

Last but not least I'd like to share today: contributing to microsoft repository is pretty easy, straightforward and really exciting. If you have any suggestions, issues or questions feel free to create a new issue with an explanation in details and be sure that a competent feedback will be posted very soon.

In my case I found that an implementation of a several methods was missed and @jkotas suggested me to be responsible for this improvement. Honestly it's a pretty simple improvement and my contribution wasn't too important, but I can't describe how exciting it was.

![coreclr_oss](/assets/images/posts/coreclr_oss.png)

Finally I'd like to recommend an awesome video- [Adam Sitnik — My awesome journey with Open Source](https://www.youtube.com/watch?v=2HSPKyAyuik) for everyone who hasn't seen it before. In this video Adam shares his experience and gives a lot of useful advice and recommendations for people who have been planning to start contributing but still do not know how to do it.

Thanks everyone for reading and feel free to comment.
