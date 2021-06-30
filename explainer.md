# WebXR Image Tracking API - explainer DRAFT

## DISCLAIMER

**WARNING:** This is an early draft of a proposed API that's still under active discussion, see for example [immersive-web/administrivia#135](https://github.com/immersive-web/administrivia/issues/135). **DO NOT** quote this draft as a specification, and expect incompatible changes in the future.

## Introduction

Augmented reality platforms that are based on camera mapping often have the capability to recognize user-specified images in the environment and provide tracking information for them. The Image Tracking API makes this feature available to WebXR applications.

For an example of this functionality in a native application, see:

  https://developers.google.com/ar/develop/java/augmented-images

Since detecting and tracking these images happens locally on the device, this functionality can be implemented without providing camera images to the application.

## Use Cases

- augmenting a physical tabletop game by recognizing components and adding 3D models to them when viewed through a smartphone WebXR application

- creating shared anchor points for a multi-user AR experience by placing trackable images in the environment

- using a poster as a portal into an immersive experience for an art installation

## Using the API

Request image tracking as a required or optional feature using its feature descriptor and a list of images to track:

```js
const img = document.getElementById('img);
// Ensure the image is loaded and ready for use
// FIXME: does createImageBitmap do this implicitly?
await img.decode();
const imgBitmap = createImageBitmap(img);

const session = await navigator.xr.requestSession('immersive-ar', {
  requiredFeatures: ['image-tracking'],
  trackedImages: [
    {
      image: imgBitmap,
      widthInMeters: 0.2
    }
  ]
});
```

The `image` attribute must be an ImageBitmap, it can be created from various image sources such as HTMLImageElements using [createImageBitmap](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/createImageBitmap). The image content is a snapshot at the time the `requestSession` call is made, and later changes to the image such as redrawing a canvas source image have no effect on tracking.

For effective tracking, images must contain sufficient detail, must not have rotational symmetry, and should avoid repeating patterns or flat single-colored areas.

If multiple images are being tracked, results can be unpredictable if the images share common features such as a logo appearing on multiple images. In that case, the system may return tracking results for the wrong image.

The `widthInMeters` attribute specifies the expected measured width of the image in the real world. This is required, but can be an estimate. If the actual size doesn't match the expected size, the initial reported pose when the image is first recognized is likely to be inaccurate, and may remain inaccurate if the system can't reliably detect its true 3D position. When viewed from a fixed camera position, a half-sized image at half the distance looks identical to a full-sized image, and the tracking system can't differentiate these cases without additional context about the environment.

Once the session is active, the application can call `getTrackedImageScores()` on the XRSession. This returns a promise with information about the expected ability to use the provided images for tracking. The argument is an array containing one XRTrackedImageScore enum value per image, in the same order as the `trackedImages` array provided to `requestSession`. The enum value `untrackable` means that the image is not usable for tracking, for example due to having insufficient distinctive feature points, and this image will never appear in tracking results. The value `trackable` means that the image is potentially detectable. (Future versions of this API may define additional more granular values with quality estimates for trackable images. A value other than `untrackable` should be considered to be a potentially trackable image.)

```js
async function onSessionStarted(session) {
  const scores = await session.getTrackedImageScores();
  let trackableImages = 0;
  for (let index = 0; index < scores.length; ++index) {
    if (scores[index] == 'untrackable') {
      MarkImageUntrackable(index);
    } else {
      ++trackableImages;
    }
  }
  if (trackableImages == 0) {
    WarnUser("No trackable images");
  }
}
```

Once the session is active, in requestAnimationFrame, query XRFrame for the current state of tracked images:

```js
const results = frame.getImageTrackingResults();
for (const result of results) {
  // The result's index is the image's position in the trackedImages array specified at session creation
  const imageIndex = result.index;

  // Get the pose of the image relative to a reference space.
  const pose = frame.getPose(result.imageSpace, referenceSpace);

  const state = result.trackingState;

  if (state == "tracked") {
    HighlightImage(imageIndex, pose);
  } else if (state == "emulated") {
    FadeImage(imageIndex, pose);
  }
}
```

The `trackingState` attribute provides information about the tracked image:

* `tracked` means the image was recognized and is currently being actively tracked in 3D space, and is at least partially visible to a tracking camera. (This does not necessarily mean that it's visible in the user's viewport in case that differs from the tracking camera field of view.)
* `emulated` means that the image was recognized and tracked recently, but may currently be out of camera view or obscured, and the reported pose is based on assuming that the object remains at the same position and orientation as when it was last seen. This pose is likely to be adequate for a poster attached to a wall, but may be unhelpful for an image attached to a moving object.

The `imageSpace` origin is the center point of the tracked image. The +x axis points toward the right edge of the image and +y toward the top of the image. The +z axis is orthogonal to the picture plane, pointing toward the viewer when the image's front is in view.

The returned image tracking data also includes a `measuredWidthInMeters` value as measured by the tracking system. This is zero if this is unknown, for example due to the image being detected but not yet firmly located in 3D space. The measurement is updated on a best-effort basis as the image is being tracked, but may remain zero if the implementation is unable to provide such a measurement. If the tracking state is `tracked`, drawing a rectangle of the measured width at the imageSpace's pose in 3D space should ideally be a close match to the image as seen by the tracking camera. If the actual size differs from the initially specified size and hasn't been accurately measured yet, drawing this rectangle on a 2D screen should still visually appear at the expected screen position and size, but may be at the wrong depth, leading to incorrect occlusion compared to other scene objects.

The result list only contains entries for actively tracked images. The order of results is arbitrary, but each result's `index` attribute provides its location in the `trackedImages` array used with the initial session request.

If tracking is lost, the image stops appearing in the results for future frames. If an application needs to take action on tracking loss, it can do so by saving information about the previous frame's tracked images, for example:

```js
let imagesTrackedPreviousFrame = {};

function onAnimationLoop(frame) {
  let imagesTrackedThisFrame = {};
  for (const result of frame.getImageTrackingResults()) {
    imagesTrackedThisFrame[result.index] = true;
  }

  for (const index in imagesTrackedPreviousFrame) {
    if (!imagesTrackedThisFrame[index]) {
      HideImage(index);
    }
  }
  imagesTrackedPreviousFrame = imagesTrackedThisFrame;
}
```

## Alternatives Considered

Overall, the goal for this API is to provide useful functionality to
applications that support specific use cases, while not granting the
site full access to the user's camera feed. The API is intended to
provide limited information about prespecified images in the user's
environment, with ways for the user agent to enforce restrictions such
as requiring a prominent view of the image before initiating tracking.

### Detailed specification of trackable image types and related algorithms

This API leaves it up to the user agent to define which types of
images are considered trackable and what algorithms are used to
recognize and track them. This is a rapidly evolving field and the
state of the art is advancing steadily. Having a detailed
specification and associated detailed requirements for trackable
images would likely be a significant burden on implementors, while
also making it more difficult to benefit from advances in device
capabilities. The expectation is that the user agent can use platform
features to perform image tracking and that this is also likely to be
more efficient and scalable than custom image analysis by the user agent.

### Raw Camera Access

The proposed WebXR Raw Camera Access API
([explainer](https://github.com/immersive-web/raw-camera-access/blob/main/explainer.md),
[spec draft](https://immersive-web.github.io/raw-camera-access/))
would allow applications to use their own image processing code to do
roughly equivalent image tracking using computer vision
algorithms.

Raw camera access has some potential advantages. For example,
application-side image processing could potentially produce more
consistent results across platforms. There would still be a dependency
on camera characteristics and lighting conditions that affect the
ability to detect and track images, but it avoids the issue of
platform-dependent image recognition logic that may affect which
classes of images are detectable.

On the other hand, raw camera access offers less privacy protection
since it requires granting direct image access to the application and
its origin site. If that access is granted, the user agent has no way
to protect the user from potentially hostile sites that might upload
image streams or search for objects of interest in the user's
environment.

Also, a platform-native image tracking implementation can take
advantage of optimizations such as specialized processors which can be
faster and more power efficient than an equivalent JavaScript or WASM
implementation.

### Barcode Detection API

The Barcode Detection API ([spec
draft](https://wicg.github.io/shape-detection-api/#barcode-detection-api)
is intended to support detecting linear and two-dimensional barcodes
in images or live image feeds, for example finding and decoding QR
codes.

The Barcode Detection API's [Security and Privacy
Considerations](https://wicg.github.io/shape-detection-api/#security-and-privacy-considerations)
say that _It is critical for implementations to ensure that it cannot
be used to bypass protections that would otherwise protect an image
source from inspection_. This implies that the site should have raw
camera access permission or similar access rights before being granted
use of this API.

It would potentially be useful to support the Barcode Detection API in
conjunction with the WebXR Raw Camera Access API as a developer
convenience, but that is a separate feature, and it would not seem
appropriate to allow generic barcode identification and tracking
without raw camera access.

Applications can supply specific instances of QR codes or marker
images to the image tracking API as part of the `trackedImages` list,
and in that case it is up to the UA and device to determine if those
images are trackable. An implementation designed for naturalistic
images may be unable to find sufficient features for tracking in such
a marker.

### Using the pose `emulatedPosition` flag

The image tracking result uses a separate enum to distinguish
`"tracked"` and `"emulated"` results. An alternative would have been
to use the `emulatedPosition` boolean attribute for XRPose. However,
that attribute is intended to indicate a pose with a known orientation
combined with an unknown position, for example a 3DoF headset, and
that doesn't fit this use case where both orientation and pose are
emulated.


## Appendix: Proposed Web IDL

```webidl

partial dictionary XRSessionInit {
  sequence<XRTrackedImageInit> trackedImages;
};

dictionary XRTrackedImageInit {
  ImageBitmap image;
  float widthInMeters;
};

enum XRImageTrackingScore {
  "untrackable",
  "trackable",
};

partial interface XRSession {
  Promise<FrozenArray<XRImageTrackingScore>> getTrackedImageScores();
};

partial interface XRFrame {
  FrozenArray<XRImageTrackingResult> getImageTrackingResults();
};

enum XRImageTrackingState {
  "tracked",
  "emulated",
};

interface XRImageTrackingResult {
  [SameObject] readonly attribute XRSpace imageSpace;
  readonly attribute unsigned long index;
  readonly attribute XRImageTrackingState trackingState;
  readonly attribute float measuredWidthInMeters;
};
```
