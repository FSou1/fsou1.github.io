---
title: How to create nested folders within Azure Blob container
tags: .NET Azure
redirect_from: "/Nested_folders_with_azure_blob_storage/"
---

Do you know that by default Azure Blob storage does not give you an opportunity to group your blobs with folders? But this post is intended to share with you one workaround. Feel free to find out the source codes at the end.

Let me give to those who do not have experience with this solution a few words about that: Blob storage is a massively scalable object storage for unstructured data. With exabytes of capacity and massive scalability, Blob Storage stores from hundreds to billions of objects in hot, cool, or archive tiers, depending on how often data access is needed. Store any type of unstructured data—images, videos, audio, documents, and more—easily and cost-effectively.

It's a great choice for the right use-case - as always. But lets return to the topic and show you the issue you'll face with very soon if won't organize your blobs within container:

![blobs-mess-in-container](/assets/images/posts/blobs-mess-in-container.png)

It looks like a mess, isn't it? I could also argue that whether you have different blobs you need to create additional containers. It makes sense and I can't agree with you more, since you won't have a necessity to group your blobs by some condition (let it be a year or a month of created date). And here is a way how to solve it: just use the folder name as a part of your blob's name. Amazing, but it works.

```csharp
blobName = $"photos/photo-{i}.png";
cloudBlockBlob = cloudBlobContainer.GetBlockBlobReference(blobName);
await cloudBlockBlob.UploadTextAsync("Blob content");
```

In first case lets group our blobs by the blob type (photo, log, request):

![blobs-by-type](/assets/images/posts/blobs-by-type.png)

Well, it's already good enough, but what about additional grouping by year? Taking into account the above it's not a big deal as well:

```csharp
blobName = $"photos/2017/photo-{i}.png";
cloudBlockBlob = cloudBlobContainer.GetBlockBlobReference(blobName);
await cloudBlockBlob.UploadTextAsync("Blob content");
```

![blobs-by-type-by-year](/assets/images/posts/blobs-by-type-by-year.png)

One more additional feature in this case is that your blob's uri will also contains nesting levels: `https://your-storage.blob.core.windows.net/nested-blob-container/photos/2017/photo-1.png`.

Thanks for reading and feel free to let me know whether it was useful for you.

Links:

1. [GitHub source codes](https://github.com/FSou1/AzureSamples/tree/master/Storage.Blob.SubContainers);
2. [Blob storage](https://azure.microsoft.com/en-us/services/storage/blobs/);
3. [Quickstart: Upload, download, and list blobs using .NET](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-dotnet?tabs=windows).
