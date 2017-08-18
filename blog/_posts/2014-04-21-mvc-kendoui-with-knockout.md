---
layout: post
title: 'MVC Application with Kendo UI and Knockout'
date: 2014-04-21 12:00:00
author: 'Matt Millican'
permalink: blog/mvc-application-with-kendo-ui-and-knockout/
disqus_identifier: mvc-application-with-kendo-ui-and-knockout
disqus_url: /blog/post/mvc-application-with-kendo-ui-and-knockout
redirect_from: /blog/post/mvc-application-with-kendo-ui-and-knockout
---

I've been a big fan of Kendo UI since before it was actually Kendo UI (fka "Telerik MVC Extensions").  I've recently began to like Knockout quite a bit and have been doing a lot more client side development.  Unfortunately though, a lot of my javascript was "spaghetti code."  I had some JS at the bottom of pages for some stuff and some in random external JS files of course without any rhyme or reason.  Further, I was doing a ton of things like $('#my-field').hide().  That's fine and all, but in large apps, I felt it getting hard to manage and keep track of.

I've been trying to find a way to combine three of my favorite tools: MVC, Kendo UI MVC, Knockout and Knockout Kendo (Knockout bindings for Kendo UI); for some time now.  Finally, I think I found a solution I'm content with (for now of course).  My solution comprises of AJAX bindings in the Kendo controls, using "modules" in JS and Knockout view models.

The example I'm going to show is a simplified version of a user manager for one of my projects.

### Kendo Grid (index.cshtml)

``` html
@(Html.Kendo().Grid<UserModel>()
    .Name("account-users")
    .Deferred()
    .DataSource(ds => ds
    .Ajax()
    .ServerOperation(false)
    .Model(mdl => mdl.Id(x => x.Id))
    .Read("GetAccountUsers", "Users")
    .Sort(sort =>
        {
            sort.Add(x => x.LastName);
            sort.Add(x => x.FirstName);
       }))
    .Sortable()
    .Filterable()
    .Pageable()
    .Editable()
    .Columns(cols =>
        {
            cols.Bound(x => x.LastName).Title("Last name").Width("15%");
            cols.Bound(x => x.FirstName).Title("First name").Width("15%");
            cols.Bound(x => x.Username).Title("Username").Width("15%");
            cols.Bound(x => x.EmailAddress).Title("Email address").Width("25%");
            cols.Bound(x => x.LastLogin).Title("Last login").Format("{0:d}").Width("15%");
            cols.Command(cmd => cmd.Custom("manage").Text("Edit").Click("Collab.users.editUser").SendDataKeys(true)).Width("15%");
        }))
```

The grid is pretty simple, but a few things to note: I am using the Ajax binding, going to a simple ActionResult method that returns the list of users for this account. The grid has filtering, sorting and pagination enabled. Since I want to direct the users to a different page to edit the user, I'm using a custom command event as seen on the last line of the column definitions. More on that below.

### Adding a new user, in a Kendo Window

Typically, when users are creating something new, a user, I like to let them enter the minimal information and then add more later, if they choose to.  Modal windows are the perfect place for these small(er) forms.  The Knockout Kendo bindings work wonderfully for this:

``` html
<div id="new-user-window" data-bind="kendoWindow: { isOpen: EditingUser.Show, width: 600, title: 'New user' }">
    @Html.Partial("_New")
</div>
```

This is a much cleaner way to create a Kendo window than the typical Razor code required.  It's displaying an MVC partial view which essentially just has all the markup for the form.  It's worth mentioning that I have another window just below that has a different form it and they operate nicely together.

### Opening the Window(s)

Above the grid, I have some action buttons, that are bound to the KO model, essentially just used for opening the windows.

``` html
<p class="actions">
    <a href="#" class="btn btn-sm btn-success" data-bind="click: AddUser">New user</a>
    <a href="#" class="btn btn-sm btn-success" data-bind="click: Invite">Invite user(s)</a>
</p>
```

### page js (creating & binding the view model)

This part is pretty basic, but because bindings can't overlap on the same page, I'm declaring the element that this binding should apply to.  This will allow you to have multiple alerts on a single page, for things like alerts, and other common parts of your page(s).  You'll see how I'm binding it to a specific element, in this case a wrapping `<div>` by the bold code.

``` html
<script>
    $(document).ready(function () {
    window.model = new UserListViewModel(@(Html.Raw(Model.SerializeModel())));
    ko.applyBindings(model, document.getElementById('userManager'));
    });
</script>
```

### Knockout view models

Now for the fun that is the KO [view] models.  Though these are pretty basic, you'll see where they tie in with the Knockout buttons and windows.

``` js
var UserListViewModel = function () {
    'use strict';
    var self = this;

    this.EditingUser = new EditUserViewModel();
    this.InviteUsers = new InviteUsersViewModel();
    this.AddUser = function () {
        self.EditingUser.Show(true);
        self.EditingUser.IsNew(true);
    };

    this.Invite = function() {
        self.InviteUsers.Show(true);
        self.InviteUsers.AddInvite();
    };
};

var EditUserViewModel = function() {
    var self = this;

    this.Show = ko.observable(false);

    this.IsNew = ko.observable(false);

    this.Id = ko.observable();

    // Properties removed for brevity

    this.SaveUser = function () {
        // removed for brevity
    };
};
```

Both the EditUserViewModel and the InviteUsersViewModel have the Show properties that define when to show the windows.

### app.users.js

As I mentioned above, I have what I'm calling "modules" that deal which various parts of the app.  These are basically to encapsulate methods that I call frequently from Kendo controls and KO models.

``` js
var app = app || {};
app.users = (function() {
    'use strict';

    var _refreshUsers = function() {
        var grid = $('#account-users').data('kendoGrid');
        grid.DataSource.read();
    };

    var _editUser = function (e) {
       e.preventDefault();
       var dataItem = this.dataItem($(e.currentTarget).closest('tr'));
       window.location = _.string.format('/users/edit/{0}', dataItem.Id);
    };

    return {
        refreshUsers: _refreshUsers,
        editUser: _editUser
    };
}());
```

There's a lot of different moving parts but so far the code has been able to stay pretty clean for the different areas of the app I've applied this to.  I used to be of the mentality that one solution has to do everything in the app.  Recently, thanks to some bright minds such as [John Papa](https://twitter.com/John_Papa), [Burke Holland](https://twitter.com/burkeholland) and many others, I've realized this isn't true.  It's more about finding tools and patterns that work for you (and your team) and best accomplish your needs.

I'd love feedback and ideas from others using the same (or different) tools.