# WebXR Marker Tracking API - Explainer [DRAFT]

## DISCLAIMER

**WARNING:** This is an early draft of a proposed API that's still under active discussion. **DO NOT** quote this draft as a specification, and expect incompatible changes in the future.

## Introduction

The Marker Tracking proposal aims to extend the [WebXR Device API](https://www.w3.org/TR/webxr/) with a WebXR way to detect, decode and track physical machine-readable markers in the user’s environment. The initial version focuses on QR codes, while keeping marker tracking separate from the existing [image tracking proposal](https://github.com/immersive-web/image-tracking/tree/main) for developer supplied image targets.

Marker detection and tracking is expected to be performed by the XR device and user agent in a privacy-perserving manner. This allows WebXR applications to receive useful marker information and spatial tracking results without being given access to raw camera images. Marker detection, pose and decoded data may still reveal sensitive information about the user’s environment, so privacy and permission considerations are an important part of the proposal.

## User-Facing Problem

People increasingly encounter machine-readable markers on physical objects and in shared spaces, such as product packaging, equipment, exhibits, signs and printed material. Native XR applications can use these markers to identify an object, retrieve relevant information and place virtual content at the marker’s physical location.

WebXR does not currently provide a standard way to support the same interaction in the browser. As a result, browser based XR experiences may be unable to offer marker linked content, or may offer it only through device specific solutions with varying availability and behaviour. A standard WebXR capability could allow users to access more consistent marker-based XR experiences across supported devices and browsers.

WebXR authors can attempt to build marker-based experiences using platform-specific native APIs, third-party SDKs or application-side computer vision. These approaches reduce portability, may require broader access to camera imagery than the experience needs and do not provide a consistent WebXR model that combines marker data with spatial tracking information.

The [WebXR Image Tracking proposal](https://github.com/immersive-web/image-tracking/tree/main) addresses a related problem, but is designed around images supplied by the application when requesting an XR session. QR-style marker use cases often require runtime discovery: detecting supported physical markers encountered after an XR session begins, where the application may not know the marker’s location or decoded data in advance.


## Goals

- Enable immersive WebXR experiences to detect supported physical markers in the user’s environment, initially focusing on QR codes.

- Support use cases where markers are discovered during an active XR session, rather than requiring every individual marker target to be supplied beforehand.

- Allow experiences to place or update virtual content relative to a detected marker’s spatial pose.

- Allow experiences to use data encoded within supported markers, subject to appropriate privacy and permission protections.

- Provide a WebXR-native capability that does not require applications to access raw camera images.

- Provide a portable WebXR abstraction that can map to platform-native marker tracking capabilities where available, while allowing for differences between devices. 

## Non-goals

- Detect and track arbitrary natural images or developer-supplied image targets. These remain the concern of the separate WebXR Image Tracking proposal.

- Expose raw camera frames or provide a general purpose computer vision API.

- Define or change QR code, barcode, GS1 or other marker format standards.

- Interpret the application-specific meaning of data encoded within a marker.

- Guarantee support for every QR code variant, stylised or branded marker or other barcode and fiducial marker family, such as AprilTag or ArUco markers, in the initial version.

- Guarantee frame-perfect or high-frequency tracking for moving markers.

- Replace general purpose barcode scanning APIs for non-XR use cases.

## User research

This explainer is informed by early design and ecosystem research; it does not yet report formal end-user research. The work so far has focused on three areas:

- Existing native platform support for QR code and barcode tracking.

- Issue discussions and lessons from the previous WebXR Image Tracking proposal.

- Related Web Platform APIs, especially general purpose barcode detection APIs.

### Native platform support

[A comparison of native platform support was carried out](https://github.com/immersive-web/marker-tracking/issues/3) to understand where existing QR code and barcode tracking APIs already overlap and where they differ. The clearest common pattern is that native platforms expose marker tracking as a runtime capability and is treated as sensitive functionality commonly gated by scene, spatial or world-sensing permissions or capabilities. Applications can detect physical QR codes or barcodes in the user’s environment, obtain spatial information about the detected marker and use decoded marker data where available.

The research also highlighted important differences to be further discussed within the Immersive Web group:

- Some platforms focus specifically on QR codes, while others support a broader set of barcode symbologies.

- Some platforms expose marker size, extents or boundary information, while others expose a smaller set of marker result data.

- Some platforms expose capability information such as size-estimation support or maximum tracked marker count, while others only expose a more general support check.

- Some platforms support updates for moving markers, while others document lower-frequency updates or recommend using marker tracking with stationary markers.

These findings support starting with QR codes as the first marker type for this proposal, while keeping the proposal’s terminology and structure broad enough to allow future discussion of supporting other marker families.

### Lessons from image tracking

The WebXR Image Tracking proposal addresses a related problem, but uses a different model. It requires developers to provide the images they want to track upfront when requesting the XR session. That approach works for known image targets, but is less suitable for QR-style marker use cases where markers may be encountered during a session and may carry decoded data that the application did not know in advance.

This proposal can build on the parts of image tracking that fit marker tracking, while taking a different approach where marker tracking has different requirements and use cases. In particular, marker tracking should not assume that every individual marker target needs to be supplied before the session starts.

Some components of image tracking remain useful precedent however, namely:

- Using WebXR session feature negotiation.

- Exposing tracking results during XR frames.

- Representing marker pose through WebXR spatial concepts.

- Distinguishing actively tracked results from results based on previous observations.

- Avoiding raw camera access where a narrower tracking capability is sufficient.

### Related web platform APIs

The [Barcode Detection API](https://wicg.github.io/shape-detection-api/#barcode-detection-api) is useful prior work in this area because it exposes decoded barcode data and supports multiple barcode formats. However, it operates on image sources and does not provide XR-specific spatial tracking. Marker tracking for WebXR is different because the useful result is not only *“this marker was decoded”*, but *“this physical marker was detected at this pose in the user’s environment and is being tracked”*. The proposal is therefore focused on the combination of decoded marker data *and* XR spatial tracking data, without exposing raw camera frames to the application.

This distinction is important for both ergonomics and privacy. A WebXR-native marker tracking API can provide the application with constrained marker results, while keeping camera imagery and platform-specific tracking details under the control of the user agent and XR device.

## Proposed Approach
 
This proposal adds a marker tracking capability to the WebXR Device API, initially supporting QR code markers. A WebXR experience opts into the capability when requesting an immersive AR session. If granted, the user agent can use the XR device’s marker-tracking capabilities to detect and track supported physical markers in the user’s environment.

Marker detection, decoding and tracking are performed by the XR device and user agent. Applications receive constrained marker tracking results rather than raw camera frames.

At a high level, each marker result could expose:

- A pose-bearing space for placing virtual content on top of, or relative to, the marker.
  
- A tracking state indicating whether the marker is currently tracked or based on a previous observation.

- Decoded marker data, where available.

- Marker geometry, such as size or extents, where available.

- Marker type or symbology, if the API later supports more than one marker family.

> [!NOTE]
> The exact shape of marker tracking results is still under discussion in the [Immersive Web marker result issue](https://github.com/immersive-web/marker-tracking/issues/5). In particular, the group is considering how pose, decoded data, marker geometry, marker type and session scoped identity should be represented.

The initial design should be informed by the existing WebXR Image Tracking proposal where the same concepts apply, especially around pose spaces, per-frame results and tracking state. It should differ where marker tracking has different requirements, particularly around runtime discovery and decoded marker data.

### Runtime marker discovery

The application requests marker tracking as a capability, and the user agent surfaces supported markers detected after the XR session begins. Unlike image tracking, the application does not need to provide every individual marker target before the session starts.

This supports QR code use cases where the application may not know a marker’s physical location or decoded data in advance. For example, a user may encounter a QR code on an exhibit, product, sign or piece of equipment during an experience, and the application can respond once that supported marker is detected.

This does not prevent future versions of this proposal from considering optional filters, supported marker type selection or size hints. Those remain open design questions.


### Marker results during XR frames

The marker tracking API should fit naturally into the WebXR frame loop. During an XR animation frame, the experience can obtain the current set of marker tracking results and use each result to update virtual content.

For example, an application might:

- Request an immersive AR session with marker tracking enabled.
- Receive marker results during XR frames.
- Read the decoded content of a detected marker.
- Obtain the marker’s pose relative to a reference space.
- Place or update virtual content at the marker’s physical location.

This follows the general WebXR pattern of working with frame specific tracking data and existing pose esimtation mechanisms to place content in the scene. It is also similar to the WebXR Image Tracking proposal, where tracking results are obtained during XR frames and each result provides a pose-bearing space.

### Example: starting a session with marker tracking

> [!NOTE]
> TODO: add primer example once the API shape is agreed.

### Example: using marker results in an XR frame (decoded content AND pose)

> [!NOTE]
> TODO: add primer example once the API shape is agreed.

### Marker identity and session scope

Marker results should be treated as session-scoped unless the spec explicitly defines otherwise. A user agent should not imply that it remembers markers across XR sessions, and marker identity should not be assumed to be stable beyond the current XR session. Applications that need to remember decoded marker data across sessions can store that data using existing web storage mechanisms, subject to the usual Web Platform privacy expectations.

### Marker size and geometry

Some platforms can estimate a marker’s physical size, while others may benefit from an application provided size hint. The initial proposal should avoid requiring a size hint unless it is necessary for interoperability. Where available, marker results should expose enough geometry for common AR use cases, such as drawing a highlight over the marker or aligning virtual content to the marker’s surface or bounds. The spec may provide developers with general guidance for optimal marker tracking conditions such as physical marker size, environment lighting conditions and distance away from the viewer.

### Moving markers

Marker tracking should not be presented as high-frequency object tracking. Some platforms can update a marker’s pose after it moves, but this may happen at a lower rate than head or controller tracking. The initial proposal should be clear that marker tracking is primarily intended and optimised for markers that are stationary, or at least not moving in ways that require frame-perfect updates.

### Relationship to other marker types

The initial proposal focuses on QR code markers because they are a practical starting point that is well supported across platforms. However, the term “marker tracking” is intentionally broader than “QR code tracking”. Future versions of this proposal could consider other marker families, such as additional barcode symbologies or fiducial markers, if there is sufficient platform support and agreement on use cases. Natural-image targets should remain part of the separate image tracking proposal.

### Privacy-preserving shape

This proposal should avoid exposing raw camera images to applications. The user agent and XR device perform marker detection and tracking, and only expose constrained marker results to the WebXR application. This is still sensitive functionality as marker presence, marker pose and decoded payloads can still reveal information about the user’s environment. The proposal should therefore define appropriate permission and feature gating behaviour, and should clearly explain how marker tracking differs from broader camera access or general purpose computer vision APIs.

## Alternatives considered

This proposal builds on earlier efforts to support tracking physical content in the user’s environment, rather than starting from a completely new problem space. The main alternatives and related previous approaches considered are outlined below.

### WebXR Image Tracking

The WebXR Image Tracking proposal addresses a closely related use case: recognising and tracking known image targets in the user’s environment. It provides useful precedent for WebXR feature negotiation, frame-based tracking results, pose-bearing spaces, tracking state and avoiding raw camera access where a narrower capability is sufficient. However, image tracking is based on images supplied by the application when requesting the XR session. That model is suitable when the application already knows the visual targets it wants to recognise. It is less suitable for marker-based use cases where supported physical markers may be encountered after the session begins, and where the application may not know their location or decoded data in advance.

### Raw Camera Access

Raw camera access or application side computer vision could allow an experience to implement its own marker or barcode detection and tracking. However, this would expose a broader privacy surface than is needed for the intended use cases. For marker tracking, the application needs constrained information about detected markers, such as a spatial pose, tracking state, geometry and decoded data. It should not need direct access to raw camera frames to achieve those use cases.

A user agent implementation may also be able to use platform-native marker tracking capabilities where available, rather than requiring each application to process camera imagery itself.

### General-purpose barcode detection

General-purpose barcode detection APIs are useful for decoding barcode data from image sources. They can answer questions such as “what barcode was decoded from this image?”, and may remain appropriate for non-XR scanning flows. They do not, however, provide the XR-specific spatial relationship needed by this proposal: a pose bearing representation of a physical marker that can be used to place or update virtual content in the user’s environment. 

Marker tracking therefore complements, rather than replaces, general-purpose barcode detection. It is focused on combining decoded marker data with spatial tracking in an immersive XR session.

## Accessibility

> [!NOTE]
> Accessibility review will be sought from the W3C's Accessible Platform Architectures Working Group and other relevant accessibility stakeholders as the proposal develops.

Marker tracking can support accessible immersive experiences. For example, a detected marker may help an application identify a physical object, retrieve accessible information about it or place virtual guidance relative to it.

This proposal does not define accessibility requirements for QR codes, barcodes, GS1 identifiers or other marker formats themselves. Those formats and ecosystems have their own guidance for reliable symbol presentation and scanning. Marker tracking for WebXR builds on supported machine-readable formats by adding spatial detection and tracking within an XR session.

Reliable detection of a marker does not by itself make an interaction accessible. Marker scanning can create barriers where it is the only way to access information or complete an action. Applications should provide an equivalent non-marker path where appropriate, particularly for critical actions and for users who may have difficulty locating, seeing, aiming at or physically approaching a marker.
Where an application or content author controls the presentation of a physical marker, it should:

- Provide readable text or other accessible instructions explaining the marker’s purpose and how to use it.

- Follow the relevant marker symbology’s guidance on contrast, quiet zones, size and placement to support reliable detection.

- Avoid relying on colour alone in the instructions or surrounding visual cues used to identify or explain a marker.

- Place the marker where it can reasonably be seen, reached or approached by the intended users.

- Avoid making marker scanning the only available path for important information, authentication or task completion.

- Provide an alternative way to access equivalent information or functionality.

Applications should communicate marker-detection status, tracking loss and resulting actions through accessible user-interface mechanisms, not only through visual overlays or changes in the XR scene. Where possible, decoded marker information and related actions should also be available through accessible Web UI.

## Internationalization

> [!NOTE]
> Internationalization review will be sought from the W3C's Internationalization Working Group as the proposal develops.

QR codes and other barcode symbologies are used across languages, regions and industries. Some marker payloads may be plain text, while others may be URLs, product identifiers, serial numbers or application-specific data. This proposal does not define the meaning, language, directionality or application-specific interpretation of marker payloads. Standards and ecosystems such as [GS1 Digital Link](https://www.gs1uk.org/standards-services/get-market-ready/qr-codes-powered-by-gs1/gs1-digital-link) can define how identifiers are represented in barcode payloads and connected to online information, but this proposal does not define or reinterpret those payload semantics.

Applications remain responsible for interpreting decoded marker data and presenting any resulting content in an internationalized way. This includes handling language, script, writing direction, locale-sensitive formatting and user-facing text according to normal Web Platform expectations.

Decoded marker data must not be assumed to be natural language text. Many payloads are non-linguistic identifiers, URLs or structured data. If this proposal exposes decoded payload data, it should define whether the data is exposed as text, bytes, or both; how text decoding is performed; and what happens when a payload cannot be represented as text.

The proposal should avoid locale-dependent transformations, inferred language or inferred text direction when handling decoded payloads. Where a payload is presented as user-facing text, applications should determine its language and direction from relevant metadata or application context rather than relying on heuristics.

QR codes are the initial focus of this proposal because they provide a practical cross-platform starting point. This does not imply that QR codes are the only globally relevant marker format. Future versions may consider other barcode symbologies or marker families where platform support and use cases justify them.

## Privacy and Security Considerations

> [!NOTE]
> Privacy and security review will be sought from the W3C Privacy Interest Group and other relevant horizontal-review groups as the proposal develops.

Marker tracking is sensitive functionality. Even without exposing raw camera frames, the API may reveal information about the user’s physical environment and surroundings. The main information exposed by this proposal may include:

- Presence of supported physical markers near the user.

- Spatial pose or approximate location of those markers in physical space.

- Marker size, extents or geometry where available.

- Decoded marker contents, such as URLs, identifiers or application specific data.

- Changes in marker tracking state over time during a session.

Decoded marker data can be especially sensitive. QR codes and related barcode formats may encode product identifiers, location identifiers, serial numbers, URLs, authentication-related data or other information that the user did not explicitly type into the application. Some standards, such as GS1 Digital Link, are specifically designed to connect physical products or identifiers to online information. This makes marker contents useful, but also means they should be treated as environment derived data. The proposal should minimise exposure by avoiding raw camera access and exposing only constrained marker results. It should also define clear feature gating and permission expectations, aligned with the WebXR Device API’s existing approach to sensitive immersive capabilities.

Marker tracking is sensitive functionality. Even without exposing raw camera frames, the API may reveal information about the user’s physical environment and surroundings. The main information exposed by this proposal may include:
- Presence of supported physical markers near the user;
- Spatial pose or approximate location of those markers in physical space;
- Marker size, extents or geometry, where available;
- Decoded marker contents, such as URLs, identifiers or application-specific data; and
- Changes in marker tracking state over time during a session.

Decoded marker data can be especially sensitive. QR codes and related barcode formats may encode product identifiers, location identifiers, serial numbers, URLs, authentication related data or other information that the user did not explicitly provide to the application. Some standards, such as GS1 Digital Link, are designed to connect physical products or identifiers to online information. Decoded payloads should therefore be treated as data observed from the user’s physical environment and as untrusted input. The proposal should not require user agents to automatically navigate to, fetch, resolve or otherwise act on decoded payload data as part of marker detection or tracking.

The proposal should minimise exposure by avoiding raw camera access and exposing only constrained marker results. It should define clear feature gating and user awareness expectations, aligned with the WebXR Device API’s approach to sensitive immersive capabilities, and consider whether decoded marker payloads require different treatment from pose-only marker tracking.

## Stakeholder Feedback / Opposition

> [!NOTE] 
> This proposal is at an early exploratory stage. It will be shared for wider community, developer and browser-engine review as the explainer, design questions and initial specification direction mature.

Early issue discussions indicate developer interest in a portable WebXR solution for runtime detection and spatial tracking of physical markers. The proposal is also informed by existing platform capabilities and by lessons from the WebXR Image Tracking effort.

No public browser-engine implementation commitments or formal positions have been recorded yet. Public feedback, implementation interest and concerns will be recorded here as they emerge.

- [Implementor A] : Positive
- [Stakeholder B] : No signals
- [Implementor C] : Negative

[If appropriate, explain the reasons given by other implementors for their concerns.]

## References & acknowledgements

[TBD]
