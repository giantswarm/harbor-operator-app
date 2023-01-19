**Things to avoid**

1. Don't replicate an image without specifying the tag you want.
   This will mean you pull all images and if you cancel this process it
   may cause errors when you try to pull in the future.

2. Watch out for images not being pulled from the proxy cache. This is a
   known issue: https://github.com/goharbor/harbor/issues/14791 that
   means you won't be able to to see your images in harbor. They will
   still retain in your S3.
