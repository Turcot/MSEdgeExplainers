# Extending the PointerEvent with Unique DeviceId Attribute

Consider all sections required unless otherwise noted.

Authors: [Ben Mathwig](https://github.com/bmathwig), [Sahir Vellani](https://github.com/sahirv)

## Status of this Document

This document is a starting point for engaging the community and standards bodies in developing collaborative solutions fit for standardization. As the solutions to problems described in this document progress along the standards-track, we will retain this document as an archive and use this section to keep the community up-to-date with the most current standards venue and content location of future work and discussions.

* This document status: **Active**
* Expected venue: [W3C Web Incubator Community Group](https://wicg.io/)
* Current version: [Pointer Event Device Id](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/PointerEventDeviceId/explainer.md)

## Introduction
As devices with advanced pen input capabilities are becoming increasingly prevalent, it is important that the web platform continues to evolve to fully support these advanced features in order to unlock rich experiences for both end users and developers. One such advancement is the ability for a device's digitizer to recognize more than one pen device interacting with it simultaneously. In this Explainer, we propose an extension to the `PointerEvent` interface to include a new attribute, `deviceId`, that represents a session-persistent, document isolated, unique identifier that a developer can reliably use to identify individual pens interacting with the page.

## Goals
1. Provide web developers with access to a reliable unique identifier for each pen interacting with a page.
1. Preserve the privacy and security of the end-user by preventing fingerprinting using this exension.

## Alternative Solutions
### PointerEvent.pointerId 
Changes to the PointerEvent specification outlined [here](https://github.com/w3c/pointerevents/commit/d5e6171c04d5fbb336220db1bfe39ee8d1321635) allow `PointerEvent.pointerId` to be used as a unique device ID. However, there are certain pen devices that do not yield a device ID from the hardware itself, making it impossible to distinguish between a pen that supports persistent ID and a pen that does not. If we were to use `pointerId`, devices that do not support hardware ID will appear as a new `pointerId` for each event, while pens that do support hardware ID will reuse a stable `pointerId`. By introducing `deviceId`, we can make this capability distinction clear to the developer.

## Featured Use Case
Microsoft's Surface Hub device supports detection of multiple pen devices interacting with the digitizer simultaneously. Microsoft Whiteboard takes advantage of this capability in their native Win32 application to provide a rich multi-user inking experience. Microsoft Whiteboard also ships as a web application, but browser support for multiple pen detection does not exist and creates a feature gap between the native application and the web application. Augmenting `PointerEvent` with `deviceId`,  will enable the web application to reach parity with the native desktop application. In addition to Whiteboard, other applications such as Google Jamboard can also utilize this API.

## Proposed Solution
The proposed solution is to add a new attribute `deviceId` to `PointerEvent` that has the following characteristics:

1. The attribute will be populated if both the digitizer and the pen support getting a unique hardware ID for the pen.
1. The attribute will be `null` if the digitizer and pen support getting a unique hardware ID, but the ID is not available during the current event due to limitations (see [Limitations of Current Hardware](#limitations-of-current-hardware)).
1. The attribute will be `undefined` if either the digitizer or the pen do not support getting a unique hardware ID.

### Web IDL
The proposed WebIDL for this feature is as follows:

```webidl
partial interface PointerEvent {
    readonly attribute unsigned long? deviceId;
}
```

### Usage

This ID allows the developer to assign a different inking color to each unique pen interacting with a canvas:
```html
<html>
    <body>
        <canvas id="inking-surface" width="1280" height="720"></canvas>
        <script>
            const COLOR_BLUE = 0;
            const COLOR_GREEN = 1;
            const COLOR_YELLOW = 2;
            const COLORS = [COLOR_BLUE, COLOR_GREEN, COLOR_YELLOW];

            const pen_to_color_map = new Map();
            const color_assignment_index = 0;

            const canvas = document.querySelector('#inking-surface');

            // Listen for a `pointerdown` event and map the deviceId to a color if it exists
            // and has not been mapped yet.
            canvas.addEventListener('pointerdown', function(e) {
                if (e.deviceId && !pen_to_color_map.has(e.deviceId)) {
                    pen_to_color_map.set(e.deviceId, COLORS[color_assignment_index]);

                    // Bump the color assignment index and loop back over if needed
                    color_assignment_index = (color_assignment_index + 1) % COLORS.length;
                }
            });

            // Listen for a `pointermove` and get the color assigned to this pen if deviceId exists
            // and the pen has been color mapped.
            canvas.addEventListener('pointermove', function(e) {
                if (e.deviceId && pen_to_color_map.has(e.deviceId)) {
                    const pen_color = pen_to_color_map.get(e.deviceId);

                    // ... Do some inking on the <canvas> ...
                }
            })
        </script>
    </body>
</html>
```

### Limitations of Current Hardware
The ability to gather a unique hardware ID from the pen is a fairly new capability. For example, on the Surface Hub 2S and later, the hardware ID is accessible for all events. This means that all events (`pointerdown`, `pointermove`, `pointerenter` etc.) will have access to the unique hardware ID and thus will have the `deviceId` populated. The same cannot be said for Surface Laptop and Surface Devices with older generations of Surface Pen. Because of this, the following table will be used to determine the value of `deviceId`:

| Scenario | Value of `deviceId` |
| :- | :- |
| Digitizer and Pen support Hardware ID on OS | `unsigned long` |
| Digitizer and Pen support Hardware ID, but not for initial contact | `null` |
| No support for Hardware ID | `undefined` |

## Privacy and Security Considerations
### Privacy
The following section enumerates the potential privacy concerns identified during the development of this proposal and summarizes proposed solutions for each.

| Potential privacy concern | Description | Proposed solution |
| :- | :- | :- |
| Fingerprinting of pen device | Exposing a device ID could lead to fingerprinting: users could be tracked across the web based on the id of the pen device they use. | A stable and unique hardware identifier (e.g. the hardware serial number) is not exposed to the web developer. Instead, a randomly-generated ID for the pen device is used, and regenerated per browser session and document instance. Therefore, the same pen in the same browser session across two different websites will expose two uncorrelatable device IDs.

## References
[PointerEvent interface](https://w3c.github.io/pointerevents/#pointerevent-interface)

[WinRT PenDevice.PenId Property](https://docs.microsoft.com/en-us/uwp/api/windows.devices.input.pendevice.penid?view=winrt-22000)
