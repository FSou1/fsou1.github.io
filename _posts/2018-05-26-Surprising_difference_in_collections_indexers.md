---
title: Surprising difference in array and list indexers behaviour
tags: .NET
redirect_from: "/Surprising_difference_in_collections_indexers/"
---

If the existence of a difference between using an array's and list's indexers in .NET makes surprised not only me but you as well, then let's try to puzzle it out.

# Sample

Here is a little squiz which I'd like to discuss. What is the expected output (tip: unexpected)?

```csharp
public struct Person
{
    public int Age { get; set; }

    public void IncreaseAge()
    {
        Age = Age + 1;
    }
}

class Program
{
    static void Main(string[] args)
    {
        Person[] array = new Person[] { new Person() };
        array[0].IncreaseAge();
        Console.WriteLine("Array's: " + array[0].Age); // ?

        List<Person> list = new List<Person>() { new Person() };
        list[0].IncreaseAge();
        Console.WriteLine("List's: " + list[0].Age); // ?
    }
}
```

Well, let me do not spoiler for a while and provide the IL code to you.

# Usage of the array's indexer

```
IL_0001:  ldc.i4.1
IL_0002:  newarr     ArrayIndexersSample.Person
IL_0007:  stloc.0
IL_0008:  ldloc.0
IL_0009:  ldc.i4.0
IL_000a:  ldelema    ArrayIndexersSample.Person
IL_000f:  call       instance void ArrayIndexersSample.Person::IncreaseAge()
```

I'd like to pay your attention on the `ldelema` OpCode.

> Loads the address of the array element at a specified array index onto the top of the evaluation stack as type & (managed pointer).

Taking into account the above we could realize that the array's element will be retrieved and updated by reference.

## Usage of the list's indexer

```
IL_004d:  ldc.i4.0
IL_004e:  callvirt   instance !0 class [System.Collections]System.Collections.Generic.List`1<valuetype ArrayIndexersSample.Person>::get_Item(int32)
IL_0053:  stloc.2
IL_0054:  ldloca.s   V_2
IL_0056:  call       instance void ArrayIndexersSample.Person::IncreaseAge()
```

In this case the element's retrieve was performed by a `get_Item(int32)` method. My assumption: dispite the fact that under the hood a `List` type stores elements in array and access them in exactly the same manner with the `ldelema` OpCode, the managed pointer stays at the `get_Item(int32)` method stack and the result returns by value, which further update has absolutely no effect on the storing element.

Well, now I hope it's not a big deal to realize the final output:

```
Array's: 1
List's: 0
```

Links:

1. [OpCodes.Ldelema Field](https://msdn.microsoft.com/en-us/library/system.reflection.emit.opcodes.ldelema.aspx);
2. [Performance traps of ref locals and ref returns in C#](https://blogs.msdn.microsoft.com/seteplia/2018/04/11/performance-traps-of-ref-locals-and-ref-returns-in-c/).
