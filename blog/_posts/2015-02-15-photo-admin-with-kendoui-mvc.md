---
layout: post
title: 'Photo Gallery Admin with Kendo UI for MVC'
# subtitle: 'Build a Slackbot using Node JS and Botkit'
date: 2015-02-18 12:00:00
author: 'Matt Millican'
permalink: blog/photo-gallery-admin-with-kendo-ui-for-mvc/
disqus_identifier: photo-gallery-admin-with-kendo-ui-for-mvc
disqus_url: blog/post/photo-gallery-admin-with-kendo-ui-for-mvc
---

I am currently working on building a simple CMS for myself using ASP.NET MVC and [Kendo MVC wrappers](http://www.telerik.com/aspnet-mvc).  Within the CMS, I wanted a simple photo gallery, with an administrative UI for managing albums and photos.  Follow along as I share how I built the admin.

For purposes of keeping this post shorter than a book, I am going to skip over the part of creating and editing album properties.  Essentially all that is, is a Kendo Grid and Kendo Window to create a new album.  When you want to edit the album, there's a link that goes to the edit page, where the photo admin will live.

### Creating the Photo List

For the photo list, I used a Kendo ListView to show the photos in a grid.  I created a Kendo template as well to display the images with their title and some editing options.  

``` html
@(Html.Kendo().ListView()
    .Name("album-photos-list")
    .Deferred()
    .TagName("div")
    .ClientTemplateId("photo-template")
    .DataSource(ds => 
    {
       ds.Read("AlbumPhotos", "Albums", new { albumId = Model.Album.Id }); 
       ds.Sort(x => x.Add(s => s.SortOrder));
    })
    .HtmlAttributes(new { @class = "clearfix" }))
```

Even though this is pretty straight forward, I'll explain it a little. I set Deferred on the ListView so that the script would appear at the bottom of my page where I call Html.Kendo().DeferredScripts() (which is after I include jQuery and the Kendo script files).  Next is the TagName("div") and the ClientTemplateId("photo-template") which will reference the Kendo template for each photo item, as you'll see below.  After that, we have our DataSource to define where the data is coming from as well as the sorting for the photos.  Finally we have a simple HtmlAttributes() to add a clearfix CSS class since elements inside the list are going to be floated.

As part of the photo list, we have the Kendo template which defines how the photo items will look.

``` html
<script type="text/x-kendo-tmpl" id="photo-template">
    <div class="photo-item">
        <img src="@Url.Content(Model.ImageBaseUrl)#:FileName#?w=170&h=100&mode=crop" />
        <span class="overlay">
            <span class="title">#:Title#</span>
            <span class="options">
                <a href="\\#" class="glyphicon glyphicon-pencil pull-right photo-edit-button" style="color: white; text-decoration: none;" data-photo-id="#:Id#"></a>
            </span>
        </span>
    </div>
</script>
```

I think most of this is pretty simple, but the one thing I wanted to point out was the MVC I'm "injecting" into this.  This is basically to get the base image URL from the server-side which I'm passing through my model.  You'll also see "?w=170&h=100&mode=crop" which uses [ImageResizer](http://imageresizing.net/) to resize (and process) images on the fly as they are being served (you'll see ImageResizer being used more later).  Finally, some CSS to go with the above template:

``` css
.album-photos #album-photos-list { min-width: 938px; min-height: 329px; }
.album-photos .photo-item { float: left; position: relative; width: 170px; height: 100px; margin: 5px; }
    .album-photos .photo-item img { position: relative; }
    .album-photos .photo-item .overlay { position: absolute; visibility: hidden; left: 0; top: 81px; z-index: 100; width: 166px; height: 15px; padding: 2px; background: #000000; background: rgba(0, 0, 0, 0.75); font-size: x-small; color: #FFF; }
        .album-photos .photo-item .overlay .title { display: inline-block; width: 140px; height: 15px; overflow: hidden; white-space: nowrap; text-overflow: ellipsis; }
    .album-photos .photo-item:hover .overlay { visibility: visible; }
```

Let's move to the server side and look at the controller action being used to return the data for the list of images.

``` c#
public ActionResult AlbumPhotos(Guid albumId, [DataSourceRequest] DataSourceRequest request)
{
   var album = _photoService.GetAlbum(albumId);
   if (album == null)
   {
       Log.Error(string.Format("Could not find album {0}", albumId));
       return null;
   }

   var photos = Mapper.Map<IEnumerable<Photo>, IEnumerable<PhotoModel>>(album.Photos);
   return Json(photos.ToDataSourceResult(request));
}
```

This action is getting the album by it's ID (passed in from the UI) and returning the mapped (using AutoMapper) photos using the Kendo DataSourceResult.  

### Uploading Photos

Now time for the fun part - uploading new photos.  I am using the Kendo Upload widget, as shown below.

``` html
@(Html.Kendo().Upload()
     .Name("photos")
     .Deferred()
     .Multiple(true)
     .Async(async => async.Save("SavePhotos", "Albums", new { albumId = Model.Album.Id })
     .AutoUpload(true))
     .Events(evt => evt.Success("uploadCompleted")))
```

As before I am calling Deferred() so that it puts the script for the widget at the bottom of the page.  I'm defining that I want to allow multiple uploads at once with the Multiple(true) and then saying what controller action it should call when the photos are uploaded.  Along with that, I am enabling AutoUpload so as soon as the file(s) are selected, they are automatically uploaded.  Finally, I am referencing a javascript function that will refresh the list as you'll see next.

``` js
function uploadCompleted() {
    refreshPhotos();
}

function refreshPhotos() {
    $('#album-photos-list').data('kendoListView').dataSource.read();
}
```

You might ask why I didn't just call the code that's in the refreshPhotos() method directly - this is because I have other events that will trigger a refresh of the photo list.

Now for the saving of photos on the Controller side:

``` c#
public ActionResult SavePhotos(Guid albumId, IEnumerable<HttpPostedFileBase> photos)
{
    var album = _photoService.GetAlbum(albumId);
    EnsureAlbumFolderExists(album);
    var albumPath = GetAlbumPath(album);

    var resizeSettings = new ResizeSettings();
    resizeSettings.MaxWidth = PhotoSettings.MaxWidth;
    resizeSettings.MaxHeight = PhotoSettings.MaxHeight;
    resizeSettings.Quality = PhotoSettings.MaxQuality;
    resizeSettings.Mode = FitMode.Max; // TODO: change to use config setting

    foreach (var photoFile in photos)
    {
        var filename = Path.GetFileName(photoFile.FileName);
        // If we can't get the filename for some reason, just generate one using a guid - yuck
        if (string.IsNullOrEmpty(filename))
        {
            filename = string.Format("{0}{1}", Guid.NewGuid(), Path.GetExtension(filename));
        }

        filename = filename.ToLower();

        var path = Path.Combine(albumPath, filename);

        ImageBuilder.Current.Build(photoFile, path, resizeSettings);

        var photo = new Photo
        {
            Album = album,
            Title = filename,
            FileName = filename,
            SortOrder = 1
        };

        album.Photos.Add(photo);
    }

    _photoService.UpdateAlbum(album);

    return Content("");
}
```

What's this method doing?  First off, I'm getting the album that we are adding the photos to, ensuring that the folder for the photos exists on the server and then retrieving it's path.  For me, the path might look like "/media/photos/albums/{album-slug}/".  After that, I'm setting some [ImageResizer](http://imageresizing.net/) settings from settings in my web.config so that I can resize the photos on upload.  I'm then looping through all of the photos to get their file name (or generate one if I can't get it for some reason), and then processing and saving it with the image resizer.  Once the photo has been saved, I create a new record in the album's photo collection.  Once all the photos have been uploaded, I'm saving the album (which saves all it's photo records) and returning an empty content result as Kendo expects.  Once that happens, the success event on the file upload should trigger the photo list to be refreshed.

So, there you have it.  A simple photo uploading tool using [Kendo's ASP.NET MVC](http://www.telerik.com/aspnet-mvc) wrappers.  Try it out today and feel free to leave comments with questions.