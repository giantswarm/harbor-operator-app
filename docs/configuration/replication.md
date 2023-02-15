**Things to avoid**

1. Don't replicate an image without specifying the tag you want. Specifically,
   through the harbor UI as this will mean you pull all versions of the images. Currently, it is not possible to specify a range of versions and you will have to give the tag for weach version you want. If you cancel this process it may cause errors when you try to pull in the future.

   Because of this you should only replicate images that you want a specfic tag from. We would need to look more into this bug to determine exactly what is happening.

2. Watch out for images not being pulled from the proxy cache. This is a
   known issue: https://github.com/goharbor/harbor/issues/14791 that
   means you won't be able to to see your images in harbor. They will
   still retain in your S3.
