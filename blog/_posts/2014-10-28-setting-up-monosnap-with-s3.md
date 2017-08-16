---
layout: post
title: 'Setting up Monosnap with S3'
subtitle: 'An easy way to automatically upload screenshots with a shareable link'
date: 2014-10-28 12:00:00
author: 'Matt Millican'
permalink: blog/setting-up-monosnap-with-s3/
disqus_identifier: setting-up-monosnap-with-s3
disqus_url: /blog/post/setting-up-monosnap-with-s3
---

Originally, I had Monosnap saving to a folder on my Dropbox account.  This was fine and dandy, but still had some annoyances, like the link taking a bit to copy.  Monosnap has a built-in S3 integration which is pretty easy to set up.  Let's walk through that now.

## Setting up your S3 bucket

I'm assuming you've already created an Amazon AWS account and are in general familiar with how to navigate the console.  You may want to create a separate user for Monosnap to use, and give it full access to manage S3.  

1. To give an existing user access to manage your S3 buckets, go to IAM -> Users and select the user to edit.  In the "Permissions" section, click "Attach User Policy."  Search for "AmazonS3FullAccess" in the "Select Policy Template" list and click Select.

2. Next up is giving the user an Access Key.  Just below the permissions section, is a "Security Credentials" section.  Click "Manage Access Keys" to create a key.  Be sure to copy/paste these as you won't be able to see them once you close the modal (and will need them later).

3. Now, we to create the bucket.  Go to Services -> S3.  Click "Create Bucket" and in the modal, enter your preferred bucket name.  NOTE:  If you want to use a custom domain (ie s.mattmillican.com), you must enter that as the bucket name.  You can select the region that works best for you (I selected "US Standard."  That should be it for the S3 side.

## Configure DNS

I'm not going to go into too much detail here as every DNS provider is a little different.  Essentially though, you want to create a CNAME record for your subdomain (or domain if you're not using a sub) that points to "s3.amazonaws.com"  In my case I created a CNAME "s" that points to S3.  Keep in mind, your changes may take a while to propagate.

## Configuring Monosnap

Go to Monosnap Preferences.  If you haven't played around in here, I recommend you do so.  You can change things like the filename format (in "Advanced").  What we want to do now is set up the Account tab.

1. In the left side list, click "Amazon S3."  Go ahead and click the "default" button near the top to make sure this is always the default account for your uploads.
2. Enter your Access Key ID and Secret Access Key you got before in the S3 step.
3. Click Refresh to refresh your bucket list, and then select the bucket you created in the S3 console.
4. If you want to change the path (a sub-folder), go ahead and enter that.
5. If you're going to use a custom subdomain, enter that in the Base URL.  It can be HTTP or HTTPS.
6. Test settings to make sure you're good.

You should be good to go now to take screenshots and upload them to S3. 