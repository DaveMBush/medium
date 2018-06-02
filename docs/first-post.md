# Upload an In Memory Image as a File using Angular

This past week, I needed to be able to upload an image in my application to the server as a file so that I could crop it and upload it.

Now, uploading an image that you pulled up using the file upload control is relatively straight forward. But, in our case, 
the image we want to be able to upload didn't always come from the user's file system. This causes two problems.

First, you can't crop an image you retrieved from a different URL using the HTML Canvas because of Cross Origin restrictions 
and second, you can't upload the file using the standard file upload mechanism because you didn't get it from the file system.


## Image to Data without Canvas

Now, the standard way of converting an Image to a data URL is to
- create a new Image,
- set the onload event handler to a function
- set the Image src attribute to the file 
- in the onload function,
  - draw the image onto the canvas
  - call canvas.toDataUrl(mimeType) to get the data url.

It is pretty trivial code:

``` typescript
const canvas = document.createElement("canvas");
context = canvas.getContext('2d');
const base_image = new Image();
base_image.src = 'img/base.png';
base_image.onload = function(){
  context.drawImage(base_image, 100, 100);
  let dataUrl = canvas.toDataURL('image/jpeg');
}
```

But, the trouble starts when you use a URL that doesn't originate from the file system, or the same domain as the application 
you are running.

The trick is to read the image using an XMLHttpRequest for foriegn URLs and then use a FileReader object and call 
readAsDataURL passing the result of the XMLHttpRequest.read.

``` typescript
const image = new Image();
const self = this;
const reader = new FileReader();
reader.onloadend = function () {
    reader.onloadend = null;
    // make sure the image is loaded before we go
    // after width and height;
    image.src = null;
    image.onload = () => {
        image.onload = null;
        // do something with the new image here
    }
    image.src = reader.result;
};

image.onload = () => {
    image.onload = null;
    const xhr = new XMLHttpRequest();
    xhr.onload = function () {
        reader.readAsDataURL(xhr.response);
    };
    xhr.open('GET', image.src);
    xhr.responseType = 'blob';
    xhr.send();
};

if(action.payload instanceof File) {
    // won't be using image.onload so we need to turn it off
    image.onload = null;
    reader.readAsDataURL(action.payload);
} else {
    // this triggers image.onload which triggers reader.readAsDataURL
    image.src = action.payload;
}
```

Now we can put the image using the data URL on the canvas and the canvas doesn't know we got it from a foreign URL 
any more so we no longer get a cross origin URL.

The entry point for this code is on line 27. You see that if we are working with a File, we just use the FileReader directly. 
But if we are using http, we go through the XMLHttpRequest.

## Fake File Upload

The only problem with this is that now we no longer have the file point, so if we want to upload the file, as a file, to the server we have come up with some way of creating a fake file object. And believe it or not, that is a lot easier than you might think.

You see, a File object is just a kind of Blob object. So, all we really need to do is to create a Blob. But, we have one additional issue. Our image is in base 64 and we need to convert it to a binary byte array.

``` typescript
b64toFile(dataURI): File {
    // convert the data URL to a byte string
    const byteString = atob(dataURI.split(',')[1]);

    // pull out the mime type from the data URL
    const mimeString = dataURI.split(',')[0].split(':')[1].split(';')[0]

    // Convert to byte array
    const ab = new ArrayBuffer(byteString.length);
    const ia = new Uint8Array(ab);
    for (let i = 0; i < byteString.length; i++) {
        ia[i] = byteString.charCodeAt(i);
    }

    // Create a blob that looks like a file.
    const blob = new Blob([ab], { 'type': mimeString });
    blob['lastModifiedDate'] = (new Date()).toISOString();
    blob['name'] = 'file';
        
    // Figure out what extension the file should have
    switch(blob.type) {
        case 'image/jpeg':
            blob['name'] += '.jpg';
            break;
        case 'image/png':
            blob['name'] += '.png';
            break;
    }
    // cast to a File
    return <File>blob;
}
```

You can use the resulting "File" anywhere you would use a File you had retrieved from the file system.

## Upload via Http

The last bit of this is that we need to upload this via the Http service. This is going to be harder to show with code because it depends on what you need to do.

In my case, I needed to just upload the file, so the post was pretty straight forward. I used an Http.post() and passed the returned "file" as the data parameter.

But, you may need to upload it by wrapping the file in a Form object and specifying varying headers.

## The End

I know it is a relatively short post, but I hope it is helpful to someone. There are a lot of examples out there of how to do this in JavaScript and jQuery, but I was unable to find anything that was specific to TypeScript and Angular.
