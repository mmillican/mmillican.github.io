---
layout: post
title: 'Process Images with AWS Lambda and .NET Core'
subtitle: "Process and resize images in AWS S3 using a Lambda Function and ASP.NET Core"
date: 2020-12-18 00:00:00
author: 'Matt Millican'
header-img: 'img/blog/macbook-with-smartphone.jpg'
permalink: blog/:slug
---

Generating less-than-full-sized images of user-uploaded content to an 
application such as a CMS, social platform, or 
blog. Smaller image sizes enable better performance for your users and save on 
bandwidth costs when full resolution isn't a necessity.

However, resizing an image to potentially multiple smaller sizes or doing other 
processing can cause your uploading/publishing user experience to suffer 
as it is often not an immediate response. If you are uploading images to a 
service such as Amazon S3, you can enlist the help of a Lambda Function to 
resize those images out of process, thus enhancing the upload experience.

AWS Lambda is a "serverless" service which allows you to run code without having 
to worry about provisioning or maintaining the servers behind the scenes. You can 
essentially run "any" code in a Lambda function, including Node.js, Python, .NET, 
and many more<sup>1</sup>.

In this post, we will walk through how to create a Lambda function using 
.NET Core and C# to resize images that are uploaded to an S3 bucket. 
For purposes of this post, I will assume you have the .NET Core SDK installed, and 
are familiar with what an S3 bucket is and how to upload the images to the bucket.

> Note: Lambda does not [yet] support .NET 5, so this post uses ASP.NET Core 3.1

### Getting Started

To start, install the 
[AWS .NET Lambda templates](https://www.nuget.org/packages/Amazon.Lambda.Templates)
which include project templates to "bootstrap" creating Lambda functions in .NET. 
You can install these by running the following command in your favorite CLI:

```
$ dotnet new -i Amazon.Lambda.Templates
```

> Note: for production uses, I likely would _not_ use these templates as my 
> image processor would be part of a larger application managed by CloudFormation.

If you run `dotnet new` in your CLI, you'll notice several new Lambda-related 
project templates available. For this project, we will use the **lambda.S3** template.
Run the following command to create your project (within the directory you want your 
project to be created).

```
$ dotnet new lambda.S3
```

Once this is finished, open the project in VS Code (or your favorite editor) using 
`code .`. Your new project contains two folders, `src`, and `test`, each with a 
single project in them. Make sure to run `dotnet restore` to restore the Nuget packages.

You will notice a `aws-lambda-tools-defaults.json` file in the `src/LambdaImage` 
directory. This is a configuration file that has some default settings used with 
the .NET Lambda tools. If you have multiple AWS credential profiles on your 
machine, you'll want to set the `profile` attribute to your desired profile.

> We will cover Lambda deployment in a separate post

## The Code

Enough introduction, let's get into the code. To begin, we need to install 
an image processing library for Nuget. My personal favorite is Six Labors' 
[Image Sharp](https://www.nuget.org/packages/SixLabors.ImageSharp/) as it is easy 
to use and supports a variety of image types and processing functions.

To install the package, run the following command from your project directory:

```
$ dotnet add package SixLabors.ImageSharp
```

The `FunctionHandler` method takes an `S3Event` parameter that contains 
information about the S3 object we are being notified about (the image that was just 
uploaded). We will use this to get the actual object from S3 for processing. 

In the `try` block, replace the templated code with:

```c#
// First, copy the original image out of the "upload" folder and into permanent storage
var originalSizeKey = GetResizedFileKey(s3Event.Object.Key);
await _s3Client.CopyObjectAsync(s3Event.Bucket.Name, s3Event.Object.Key, s3Event.Bucket.Name, originalSizeKey);

// Now, get the image data to use for resizing
byte[] imageBytes = null;
var contentType = "";

using (var objectResponse = await _s3Client.GetObjectAsync(s3Event.Bucket.Name, s3Event.Object.Key))
using (var ms = new MemoryStream())
{
    contentType = objectResponse.Headers.ContentType;
    await objectResponse.ResponseStream.CopyToAsync(ms);
    imageBytes = ms.ToArray();
}
```

> I made a [helper method](#image-name-helper) to make renaming the files easier.

First, because our Lambda will be listening to events for specific key prefixes, we 
need to copy the image out of that "folder" and into our permanent storage 
(S3 doesn't have true folders). Next, we need to retrieve the object by its key 
("file path") and save the content type and the bytes to use for processing. 

Before we do that though, let's create a simple object to represent our image sizes:

```c#
public class ImageSize
{
    public string Key { get; set; }
    public int Width { get; set; }
    public int Height { get; set; }

    public ImageSize(string key, int width, int height)
    {
        Key = key;
        Width = width;
        Height = height;
    }
}
```

Now, time for the resizing! Back in the `try/catch` block, add the following code: 

```c#
var imageSizes = new List<ImageSize>
{
    new ImageSize("small", 400, 400),
    new ImageSize("medium", 1000, 1000),
    new ImageSize("large", 1600, 1600)
};

foreach (var imgSize in imageSizes)
{
    context.Logger.LogLine($"... Resizing to {imgSize.Key}");

    var resizedFileKey = GetResizedFileKey(s3Event.Object.Key, imgSize.Key);

    IImageFormat imageFormat;
    using (var image = Image.Load(imageBytes, out imageFormat))
    {
        using (var outStream = new MemoryStream())
        {
            image.Mutate(x => x.Resize(new ResizeOptions
            {
                Mode = ResizeMode.Max, // TODO: might want to make this a property on each image size
                Position = AnchorPositionMode.Center,
                Size = new Size(imgSize.Width, imgSize.Height)
            }));

            image.Save(outStream, imageFormat);

            var putObjectRequest = new PutObjectRequest
            {
                Key = resizedFileKey,
                BucketName = s3Event.Bucket.Name,
                ContentType = imageContentType,
                InputStream = outStream
            };
            await _s3Client.PutObjectAsync(putObjectRequest);
        }
    }

    context.Logger.LogLine($"... Resized and saved file '{s3Event.Object.Key}' to '{imgSize.Key}'");
}

// Delete the original
await _s3Client.DeleteObjectAsync(s3Event.Bucket.Name, s3Event.Object.Key);

return "Ok"; // Lambda expects a string to be returned to indicate success
```

Let's parse through what our code is doing. For each image size in our list, we 
are generating a new filename using the [naming helper](#image-name-helper). 
Then, using the byte array created earlier, load the image using ImageSharp and 
detect its format. Using the Mutate API, we are resizing the image to the dimensions 
defined in our `ImageSize` and save it to a new stream. Once that is complete, 
we can save it back to the bucket with its new key. Be sure to save it outside of 
your original "upload" folder so you don't find yourself in a never-ending loop of 
resizing. Finally, delete the original image from the upload path.

#### Image name helper

This is a helper method I made to add the "size key" to the image filename:

```c#
private string GetResizedFileKey(string originalKey, string size = null)
{
    var filename = Path.GetFileName(originalKey);
    var newFileKeyPrefix = originalKey.Replace(OriginalKeyPrefix, "").Replace(filename, "");

    if (!string.IsNullOrEmpty(size))
    {
        filename = $"{size}-{filename}";
    }

    return $"{newFileKeyPrefix}{filename}";
}
```

### Wrapping up

So there we have it. You've completed your Lambda function to resize images 
uploaded to an S3 bucket when it gets notified of objects being created. We still 
need to deploy the function to Lambda and set up the notifications on the bucket, 
which we'll do in our next post. 

Have any tips or comments for image processing or other S3-related lambda functions? 
Feel free to share them in the comments below.

PS: This post is a part of the <a href="https://www.csadvent.christmas" class="christmassy-yay"><span>2</span><span>0</span><span>2</span><span>0</span><span> </span><span>C</span><span>#</span><span> </span><span>A</span><span>d</span><span>v</span><span>e</span><span>n</span><span>t</span><span> </span><span>C</span><span>a</span><span>l</span><span>e</span><span>n</span><span>d</span><span>a</span><span>r</span></a> 
Be sure to check it out!

PPS. Thanks to [@q](https://twitter.com/quangdaon) for proofing this post, providing 
feedback and the fancy Christmas effect.

<hr />

<sup>1</sup> [https://aws.amazon.com/lambda/](https://aws.amazon.com/lambda/)

<span>Header photo by <a href="https://unsplash.com/@maxcodes?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Maxwell Nelson</a> on <a href="https://unsplash.com/photos/taiuG8CPKAQ?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>


<style>
  .christmassy-yay span {
    text-decoration: underline;
  }

  .christmassy-yay span:nth-child(1n){
    color: #34a65f;
  }
  .christmassy-yay span:nth-child(2n){
    color: #cc231e;
  }
</style>