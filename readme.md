# Improved client side video editing

By: Yash Khandelwal & Greg Whitworth

## Introduction
Currently editing video within the browser is a very complex task as there isn't any 
straight forward approach to decoding the encoded video file to produce a raw stream that 
can do common video editing capabilities such as trimming or concatenation. Normally, web developers will do client side editing in three potential ways:

1. They will create their own client-side pipeline to decode the video file(s) to access the stream(s); do whatever edits they need to and then recode the video into the desired format.
2. They allow the user to do artificial edits keeping JS based data structure of the edits doing creative work to move the player position to give the illusion that the adjustments that you've made have occurred on the client. Then upon saving this document this document of events is sent to the server where the actual video editing occurs.
3. They will capture video content via the [MediaRecorder](https://www.w3.org/TR/mediastream-recording/#mediarecorder-api) which provides a Blob and then utilize the slice method to trim the content where desired.

All of these approaches have their pros and cons, the first one requires either knowing the video formats that the application will be working with or bundle a full zipped version of library, such as ffmpeg, in WASM to handle multiple codecs. This can result in large file sizes (at times up to 7MB zipped) to enable client side editing. This does however unlock all of the necessary needs of trimming and concatenation.

With the second approach, this likewise has the benefit of being able to handle the use cases denoted above without having to download the larger files. The negative implications of this approach is that the server side solution can produce bottlenecks in an editing queue and costs associated with having dedicated servers for doing the video edits. Additionally, this may result in numerous redundant edits in the queue since upon saving it adds the editing to the queue. This can lead to increased server side costs and a slow turn around time for the end user.

The final approach, allows you to avoid needing to download a large file or send it to a server, but it requires that the editing occurs at 1x speed. For example, if you have a 60 minute video and want to trim it to 20 minutes, you'll need to wait 20 minutes for the new blob to be created. With our early prototypes, this same work can be done in less than 3 seconds.

This API is a starting point to enable video editing on the client that not only enables the capabilities listed above without the need to handle all of above overhead for the most common web based video editing scenarios. 

We have worked with [Flipgrid](https://www.flipgrid.com) to validate that this approach tackles their video editing needs and significantly improves their user experience.

## Proposed Solution

We're proposing a `MediaBlob` that extends the regular [blob](https://w3c.github.io/FileAPI/#blob-section) and a `MediaBlobOperation` which will be used to batch the proposed media editing operations. Based on initial feedback from customers that have a need for this technology, they needed concatenation and trimming capabilities, as such that is what we started with.

## MediaBlob
```
[Exposed=(Window,Worker), Serializable]
interface MediaBlob : Blob {
    constructor(Blob blob);
    readonly attribute long long duration;
};
```

### Constructor MediaBlob
When the `MediaBlob` constructor is invoked, the User Agent MUST run the following steps:
1. Let *blob* be the constructors first argument
2. Run the steps in *[Handling MimeTypes](#handling-mimetypes)*
    * If the return value is *true*, return the new *MediaBlob*
    * else throw the [DOMException](https://heycam.github.io/webidl/#idl-DOMException) that was returned.

### Duration Property
1. When the *duration* property is called the User Agent MUST return the length of the *Blob* in milliseconds

```
let mediaBlob = new MediaBlob(blob); // blob is a Blob object for a valid media
console.log(mediaBlob.duration) // Outputs 480000 = 8 minutes
```

## MediaBlobOperation
```
[Exposed=(Window,Worker), Serializable]
interface MediaBlobOperation {
    constructor(MediaBlob mediaBlob);

    void trim(long long startTime, long long endTime);
    void split(long long time);
    void concat(<Sequence<MediaBlob>);
    Promise<Sequence<MediaBlob>> finalize(optional DOMString mimeType);
};
```

### Constructor MediaBlobOperation
When the `MediaBlobOperation` constructor is invoked, the User Agent MUST run the following steps:
1. Let *mediaBlob* be the constructors first argument
2. If *mediaBlob* is not undefined, return the new *MediaBlobOperation*
3. else throw a "[DataError](https://heycam.github.io/webidl/#dataerror)" [DOMException](https://heycam.github.io/webidl/#idl-DOMException)

### Batching
The `MediaBlobOperation` methods *Trim*, *Concat* and *Split* will not modify the MediaBlob when invoked. These methods will be tracked and executed only when *Finalize* is called. The benefit of batching these operations is to save memory and provide efficiency. Due to the nature of *Split* operation, it should always be the last method if called before calling *Finalize*.

### Trim Method
The *trim* method is utilized to create the segment of time that the author would like to keep;
the remaining content on either end, if any, is removed.

#### Parameter Definitions
* `startTime`: The starting time position in milliseconds **Required**
* `endTime`: The ending time position in milliseconds **Required**

#### Trim Algorithm
1. Let *x* be the byte-order position, with the zeroth position representing the first byte
2. Let *O* represent the blob to be trimmed

The User Agent will execute the following when *finalize* is called.

3. *[Check for errors](#error-handling-in-finalize)*
5. Move *x* to the *startTime* within *O*
6. Consume all of the bytes between the *startTime* and the *endTime* and place these bytes in a new MediaBlob object *trimmedBlob*

```
let mbo = new MediaBlobOperation(new MediaBlob(blob));
mbo.trim(240000, 360000);
mbo.finalize().then(function(mediaBlobs) {
    // mediaBlobs[0] will be the trimmed blob of 2 min duration
});
```

### Split Method
The split method allows the author to split a *blob* into two separate MediaBlobs at a given time. Due to the nature of this operation, it should be the last
operation before calling finalize().

#### Parameter Definitions
* `time`: The time, in milliseconds, at which the blob is to be split into two separate MediaBlobs.

##### Split Algorithm
1. Let *time* represent the *split location*
2. Let *O* represent the blob to be split

The User Agent will execute the following when *finalize* is called.

3. *[Check for errors](#error-handling-in-finalize)*
4. Consume all of the content prior to the *split location* and place into *mediaBlob1*
5. Place the remaining content into *mediaBlob2*
6. Place both *mediaBlob1* and *mediaBlob2* into a sequence

```
let mbo = new MediaBlobOperation(new MediaBlob(blob));
mbo.split(2000);
mbo.finalize().then(function(mediaBlobs) {
    // mediaBlobs will be an array of two MediaBlobs split at 2 seconds 
});
```

### Concat Method
This method allows you to take two *MediaBlob* blobs and concatenate them.

#### Parameter Definitions
* `blob`: This is the *MediaBlob* to concatenate with the current *MediaBlob*

#### Concat Algorithm
1. Let *m1* represent the first *MediaBlob* which will be the *MediaBlob* from the *MediaBlobOperation* that has the *concat* method called upon
2. Let *m2* represent the second *MediaBlob* which will be the *MediaBlob* that will be concatenated with *m1*

The User Agent will execute the following when *finalize* is called.

3. *[Check for errors](#error-handling-in-finalize)*
4. Produce a new *MediaBlob* and copy the bytes from *m1* into this new blob, followed by *m2*

```
let mbo = new MediaBlobOperation(new MediaBlob(blob1));
mbo.concat(new MediaBlob(blob2));
mbo.finalize().then(function(mediaBlobs) {
    // mediaBlobs[0] will be a concatenated MediaBlob of blob1 and blob2 
});
```

### Finalize Method
This method will execute all the tracked operations and return an array of MediaBlob object based on the *mimeType* value.

#### Parameter Definitions
* `mimeType`: DOMString representation of the mimetype [[RFC2046]](https://tools.ietf.org/html/rfc2046) as the expected output

#### Finalize Method
1. Let *O* be the *MediaBlobOperation* context object on which the *finalize* method is being called.
2. The User Agent will perform *[error checking](#error-handling-in-finalize)*.
3. If mimeType is provided, run the steps in *[Handling MimeTypes](#handling-mimetypes)*
    * If the return value is *true*, continue
    * else reject the  promise with the [DOMException](https://heycam.github.io/webidl/#idl-DOMException) that was returned.
4. If no errors, the User Agent will execute all the tracked operations and get a sequence of MediaBlobs.
    * The operations will be executed in a sequential order in which they are added and it is up to web developers to batch the operations in the most optimized way. 
    * This is necessary to provide better *[error handling](#error-handling-in-finalize)*.
5. The User Agent will create a new sequence of *MediaBlob* based on the mime type provided.
6. Resolve the promise with the sequence of *MediaBlob*

```
// let the mimeType of the blob be 'video/webm; codecs=vp8,opus;'
let mbo = new MediaBlobOperation(new MediaBlob(blob))
mbo.finalize('video/mp4; codecs=h264,aac;').then(function(mediaBlobs) {
    // mediaBlobs[0] will be a MediaBlob object encoded with H.264 video codec and AAC audio codec
});
```

### Example with multiple operations
```
let mbo = new MediaBlobOperation(new MediaBlob(blob));
mbo.trim(4000, 360000);
mbo.concat(new MediaBlob(blob2));
mbo.finalize().then(function(mediaBlobs) {
    // mediaBlobs[0] will be a concatenated MediaBlob of blob1 (which will be trimmed) and blob2 
});
```

### Error Handling in finalize
When *[finalize()](#finalize-method)* is called, the User Agent will perform these basic checks for the operations that are batched. This error checking should be done before executing any of the operations.

For *[trim()](#trim-method)*
1. Let *O* represent the blob to be trimmed
2. If *startTime* is less than 0 **OR** *endTime* is greater than the *O*.duration **OR** *startTime* is greater than the *endTime*:
    * Reject promise with a "[InvalidStateError](https://heycam.github.io/webidl/#invalidstateerror)" [DOMException](https://heycam.github.io/webidl/#idl-DOMException)

For *[split()](#split-method)*
1. Let *O* represent the blob to be split
2. If *time* is less than 0 **OR** is greater than *O*.duration **OR** this is not the last operation before finalize() was called
    * Reject promise with a "[InvalidStateError](https://heycam.github.io/webidl/#invalidstateerror)" [DOMException](https://heycam.github.io/webidl/#idl-DOMException)

For *[concat()](#concat-method)*
1. Let *m1* represent the first *MediaBlob* which will be the *MediaBlob* from the *MediaBlobOperation* that has the *concat* method called upon
2. Let *m2* represent the *MediaBlob* that is passed in to *concat* method to be concatenated with *m1*
3. If the mimeType of m1 does not equal the mimeType of m2:
    * Reject promise with a "[InvalidStateError](https://heycam.github.io/webidl/#invalidstateerror)" [DOMException](https://heycam.github.io/webidl/#idl-DOMException)

The DOMException.message must contain:
1. Operation name
2. The sequence number indicating the position of the operation

Example:
```
let mbo = new MediaBlobOperation(new MediaBlob(blob));
mbo.trim(0,5000);  // Trim from 0 to 5 secs
mbo.split(7000);  // Split the MediaBlob at 7 secs
mbo.finalize().then(function(mediaBlobs) { })
.catch((error) => {
    // sample error.message: "Split called on sequence 2: The time provided is greater than the duration of the MediaBlob."
});


```

### Handling MimeTypes
The *Finalize* method can take a *DOMString* of the mime-type the author desires to have returned from the method. 
To determine if the mime-type is supported, do the following:

1. Determine the mime type of the blob by using [MIME sniffing](https://mimesniff.spec.whatwg.org/#sniffing-in-an-audio-or-video-context)
2. If the mime type is not a valid mime type
3. OR the mime type contains a media type or media subtype that the UserAgent can not render:
   * return a "[NotSupportedError](https://heycam.github.io/webidl/#notsupportederror)" [DOMException](https://heycam.github.io/webidl/#idl-DOMException)
4. else 
   * return *true*

<p class="note">mimeType specifies the media type and container format for the recording via a type/subtype combination, with the codecs and/or profiles parameters [<a href="https://tools.ietf.org/html/rfc6381" target="_blank">RFC6381</a>] 
specified where ambiguity might arise. Individual codecs might have further optional specific parameters.</p>

---
[Related issues](https://github.com/MicrosoftEdge/MSEdgeExplainers/labels/MediaBlob) | [Open a new issue](https://github.com/MicrosoftEdge/MSEdgeExplainers/issues/new?title=%5BMediaBlob%5D)
