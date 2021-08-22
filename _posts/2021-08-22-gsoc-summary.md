---
layout: post 
title: "GSoC 2021: Summary"
---

### About my first GSoC

The past few months have been very exciting for me. Starting out with some
oldschool C coding, getting to know the GStreamer Editing Services codebase,
then proceeding with working in Python on the Pitivi side of things. Overall,
I'm pretty satisfied with what I've been able to accomplish, even if a few
things are still not exactly 100% finished.

Here's a short summary of everything that I've done in the past couple months.
As a reminder - my project was to implement a beat-synced editing functionality
in Pitivi, which would automatically detect beats in audio tracks and let users
easily sync their video edits to the rhythm.

### Summary of all the work done

#### GES - GStreamer Editing Services

First thing to be done was to work on GStreamer Editing Services, which Pitivi
uses as its backend. 

Initially, I added support for "snapping" elements on the timeline to markers
present on other elements. This was a bit challenging because I didn't have much
experience with C beforehand, but with the help of my mentor and other people
working on GES I was able to get my first MR there merged. Then I had to add a
few more details such as serialization support for those markers, as well as
some QoL improvements like copying markers when a clip is split.

Merge requests:
- **MR #1** - initial marker snapping implementation:
  [HERE](https://gitlab.freedesktop.org/gstreamer/gst-editing-services/-/merge_requests/259)
- **MR #2** - serialization support + QoL improvements:
  [HERE](https://gitlab.freedesktop.org/gstreamer/gst-editing-services/-/merge_requests/260)
- **MR #3** - fix for a bug caught during Pitivi-side development:
  [HERE](https://gitlab.freedesktop.org/gstreamer/gst-editing-services/-/merge_requests/263)
  

#### Pitivi

I then proceeded to work on the Pitivi side of things. This was split into two
parts: adding support for clip markers (displaying and managing them on the
timeline + some settings) and implementing the beat detection functionality,
which uses the first part as its base.

Each part has its respective MR created - the first one is in the final stages
of review, while the second one is awaiting it. Overall though, the whole
functionality works as expected and can be easily used to create beat-synced
edits with Pitivi.

Merge requests:
- **MR #1** - clip markers display, management and settings:
  [HERE](https://gitlab.gnome.org/GNOME/pitivi/-/merge_requests/402)
- **MR #2** - beat detection implementation and settings:
  [HERE](https://gitlab.gnome.org/GNOME/pitivi/-/merge_requests/404)



