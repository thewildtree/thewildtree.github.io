---
layout: post
title: "GSoC 2021: Beat detection testing"
---

## Algorithmic beat detection

The first thing I need to implement music-syncing functionality in Pitivi is a
piece of software capable of analysing a given audio track and returning some
kind of information about its beat timing.

What exactly are we looking for? There's two possible ways that came to my mind:

- getting the BPM of the whole song plus the offset of the first beat, if the
  song has constant tempo - or an array of all detected sections with their BPMs
  and offsets, if the tempo changes throughout the song
- detecting each 'beat' (usually where a full note in 4/4 signature begins)
  separately and getting a list of where they are placed throughout the song

The first option would, in theory, be the better one - we could then calculate
the timing of each beat very precisely, making it possible to account for
different time signatures and beat divisors needed to properly time our edits to
the song.

In practice, none of the libraries I tested worked like this. The reason is
simple - an algorithm precise and confident enough doesn't really exist. While
constant-tempo songs usually aren't a problem, the moment the BPM changes for even a few
beats, everything gets *way* harder. And, sometimes, it doesn't have to really
change at all.

From now on I'm going to focus on [librosa](https://librosa.org/), as it
yielded the best results in my initial testing.

## The constant tempo inconsistency

One thing, which I've discovered early on, is that `librosa` tends to be a bit
overaggresive when detecting possible tempo changes - even if, in reality, there
aren't any.

It makes sense, since this behaviour allows the algorithm to detect real BPM
changes if they actually happen, but has unpleasant side effects if the tempo
just stays constant - see this example of 'Marry You' by Bruno Mars:

<video controls>
    <source src="https://tree.s-ul.eu/rKNYqF1C.mp4" type="video/mp4">
</video>

The audible clicks you hear, represented by circles on the timeline, indicate
where `librosa` detected a beat in this song - I'll talk more about that later
on.

As you can notice, between beats 46 and 52 - when he sings 'hey baby!' - the
algorithm can't seem to decide whether it's the vocals or the drums that are
leading the beat. It tries to stay kinda inbetween, introducing a weird sounding
beat skip.

Even though it quickly recovers (and stays consistent for the rest of the song),
those very short moments of uncertainty happen often enough to throw possible
section-wide BPM detection off the table, since we aren't able to detect tempo
precisely enough, even when it's constant.

## Visualizing the results

After analysing the song, `librosa` gives us the exact timestamps of detected
beats (in seconds). I decided to use the editor of a popular rhythm game -
[osu!](https://osu.ppy.sh/) - to visualize the results on a timeline
containing a waveform of the song, lines indicating where beats / half-beats
should land (precisely timed by a human against a metronome),
and circles representing where beats were detected by the algorithm, which make
an audible click when passed on the timeline.

<figure>
  <img src="https://tree.s-ul.eu/dkqz6r0u.png" alt="my alt text"/>
  <figcaption>an example visualisation of detected beats.
  the BPM shown in the upper part is the result of manual timing mentioned above,
  and the labels on the bottom can be ignored (leftovers of the original beatmap)</figcaption>
</figure>

It was as simple as converting a list of the timestamps given by `librosa` into
the [format osu! expects](https://osu.ppy.sh/wiki/en/osu%21_File_Formats/Osu_%28file_format%29#hit-objects)
and pasting it into a beatmap made and timed by a community
member.

It is worth noting that the algorithm placed the beats ever-so-slightly late
compared to where they should be, which
coupled with the OS-added audio delay made them sound *way* too
off-sync. The delay was constant, though, and applying an approx. -45ms offset
placed them almost perfectly on time in most cases.

<figure>
  <img src="https://tree.s-ul.eu/ymzSew5F.png" alt="my alt text"/>
  <figcaption>original beat timestamps</figcaption>
</figure>

<figure>
  <img src="https://tree.s-ul.eu/CZeI2uoO.png" alt="my alt text"/>
  <figcaption>timestamps after applying offset</figcaption>
</figure>

## Detecting beats in songs of various genres and rhythm complexity

With all of that in mind, it's time to test how beat detection works in various
kinds of music.

Overall, in most cases it's gonna be used for (constant-tempo
songs with simple rhythm) it performs quite well, detecting beats on time - save
for some really calm sections, where it just doesn't have enough information to
properly judge the rhythm. It also handles more advanced cases with... varying
results. Here's a couple examples:

### Taylor Swift - Stay Stay Stay
A rather calm country pop song with constant BPM and a few calmer sections.

<video controls>
    <source src="https://tree.s-ul.eu/2O4hjSsS.mp4" type="video/mp4">
</video>

The rhythm is detected correctly during the intro, but the algorithm loses track
right before the main beat begins and then ends up detecting beats on half-notes
(red ticks), instead of full notes. As you can hear it makes sense rhythmically
though, so no big deal.

<video controls>
    <source src="https://tree.s-ul.eu/VFNJBgiF.mp4" type="video/mp4">
</video>

In a later, calmer section of the song, it lets the vocals steer the detection
for a few seconds, resulting in a few off-beat detections. Quickly recovers,
this time correctly ending up on white lines (full notes). Otherwise, it stays
consistent throughout the song, giving satisfying results.

### One Republic - Counting Stars

A bit less calm pop song with a catchy beat. The main tricky section there is
the intro, where there's only vocals and a calm guitar background before the
main beat starts. Let's take a closer look at the first 30 seconds:

<video controls>
    <source src="https://tree.s-ul.eu/vRfKgsCQ.mp4" type="video/mp4">
</video>

The algorithm starts with a slight delay and seems to mainly focus on the
vocals, which yields mixed results - sometimes it lands on the beat, sometimes
not really.

If we were to sync a video to the detected
points in time, it would still be kind of in rhythm. In some cases, the vocal focus
might even be preferable in calmer sections like this one, so overall - not a
bad result.

<video controls>
    <source src="https://tree.s-ul.eu/2pHd1MYO.mp4" type="video/mp4">
</video>

During the rest of the song, beats are detected well and only slightly
drift off-sync during the slow section, again letting the vocals
take the lead for a short while. More than good enough for our use-case.

### Bruno Mars - Marry You

Aside from the short hiccup mentioned earlier in the post, beat detection works
perfectly fine throughout the whole song - here's an example:

<video controls>
    <source src="https://tree.s-ul.eu/YnoUtNEn.mp4" type="video/mp4">
</video>

### Stephen - Crossfire

This is an interesting one - it has short (one / two measures) pauses in the
background rhythm throughout almost the whole song, as well as a few
mostly-vocal sections and a sudden, energetic ending. Tempo remains constant,
though.

<video controls>
    <source src="https://tree.s-ul.eu/KgECkWWu.mp4" type="video/mp4">
</video>

The beginning is tricky - it doesn't have much audio information to work with.
The algorithm does quite well there, only drifting seriously off-beat once and
quickly recovering as soon as a more solid beat appears.

<video controls>
    <source src="https://tree.s-ul.eu/86BG9Z1g.mp4" type="video/mp4">
</video>

Chorus looks much easier to handle, but still, the short pauses could throw
`librosa` off. It handles those quite nicely, though, only sometimes offsetting
a beat or two by a small, acceptable amount of time.

<video controls>
    <source src="https://tree.s-ul.eu/gJuUfQxM.mp4" type="video/mp4">
</video>

This slow section is probably the hardest one - there's barely any audible
'beat' there, so the algorithm understandably has a hard time detecting it.
However, all it needs to recover is the vocal line - as soon as it appears,
beats start being detected correctly throughout the rest of the slow section and
into the final, energetic chorus. Looks pretty impressive to me.

### Rita - dorchadas

A relaxing song with an energetic chorus - the first one here with a variable
tempo as well as a time signature different than 4/4.

<video controls>
    <source src="https://tree.s-ul.eu/Q27GSCng.mp4" type="video/mp4">
</video>

Starting with the beginning of the song, the beat detection works fine, but
might sound a little weird. As you might have noticed, beats alternate between
full (white) and half (red) notes - that's because this section has a time
signature of 6/4, which the algorithm doesn't take into account - detecting it
would be a whole another story.

<video controls>
    <source src="https://tree.s-ul.eu/7eUO2Ybj.mp4" type="video/mp4">
</video>

The first tempo change - 188BPM 6/4 into 120BPM 4/4 - is handled pretty much
perfectly here.

<video controls>
    <source src="https://tree.s-ul.eu/VfuXD5X4.mp4" type="video/mp4">
</video>

Second tempo change - back into 188BPM 6/4 - also goes without issues. Overall,
`librosa` handles this song really well.

### MiddleIsland - Rose

Time for a real challenge - a **non-percussive piano song** that's not timed to a metronome.

<video controls>
    <source src="https://tree.s-ul.eu/kr0ozrhc.mp4" type="video/mp4">
</video>

The BPM values visible above the timeline are a result of timing the song by a
human, close to the actual tempo enough to be playable in a rhythm game.

For me, this is a really good result considering the constantly changing BPM.

<video controls>
    <source src="https://tree.s-ul.eu/Dnv8WL0I.mp4" type="video/mp4">
</video>

A more vivid part of the song is also handled really well. Overall, the results
are quite impressive to me - I honestly thought `librosa` was going to fail
entirely before tested it on this song. Of course, it's not perfect, but is
easily good enough for us and our use-case.

### Lady Gaga & Bradley Cooper - Shallow

Another non-constant tempo song, with piano + guitar background for the most of
the song - only the final chorus has drums in it.

<video controls>
    <source src="https://tree.s-ul.eu/SLKA6kHK.mp4" type="video/mp4">
</video>

The intro is only vocals + guitar, but that looks like enough to detect rhythm
in this case. Goes visibly off-beat during the short pauses, but that's
entirely understandable.

<video controls>
    <source src="https://tree.s-ul.eu/ajzXl4Ho.mp4" type="video/mp4">
</video>

This part, where the piano starts playing, is definitely not ideal - the
detection has issues with quieter moments and when vocal audibly overwhelms the
underlying piano. Still, the result is acceptable for a non-metronome song.

<video controls>
    <source src="https://tree.s-ul.eu/43M7vP0n.mp4" type="video/mp4">
</video>

The final buildup, on the other hand, isn't an issue at all - it's not a
surprise, though, considering that drums make an appearance there.

### Demetori - Mukau no Sato ~ Deep Mountain

A relatively calm, instrumental song performed by a Japanese metal band.

<video controls>
    <source src="https://tree.s-ul.eu/28X1A134.mp4" type="video/mp4">
</video>

This song is 168BPM 4/4, but the algorithm detected beats as if they've had a
different time signature - they don't all land on the same kind of notes. It
stays in sync with the song though, so that shouldn't be a big deal, but sounds a
little weird when directly overlayed onto the song.

### Demetori - Seijouki no Pierrot ~ The MadPiero Laughs

A much more energetic, instrumental metal song. Strong drumline throughout the
entire song, but the tempo changes a little in certain moments.

<video controls>
    <source src="https://tree.s-ul.eu/ENgbkCD6.mp4" type="video/mp4">
</video>

<video controls>
    <source src="https://tree.s-ul.eu/R1dV5SPA.mp4" type="video/mp4">
</video>

No issues whatsoever here, the slight BPM changes were handled perfectly.

### Kobaryo - Bookmaker

Finally, time for some fast and loud tech / speedcore tracks.

<video controls>
    <source src="https://tree.s-ul.eu/Mirss9t8.mp4" type="video/mp4">
</video>

Despite the constant BPM, the 'noisiness' of the song seems to confuse the
algorithm in certain moments. It snaps to 1/2 notes during the slow section,
correctly goes back to 1/1 when the buildup starts, only to be thrown onto 1/2
ticks by the same sound seconds after. Similar thing happens in the chorus later
on.

### sabi - true DJ MAG top ranker's song Zenpen (katagiri Remix)

This is definitely the most noisy / rhythmically complex song here, despite
having a constant 170 BPM. Volume warning I guess!

<video controls>
    <source src="https://tree.s-ul.eu/3x4xXvhv.mp4" type="video/mp4">
</video>

Once again, we see `librosa` doing pretty well, aside from the short moment of
confusion, resulting in a jump onto 1/2 notes. It unfortunately stays that way
until the chorus around 30 seconds later:

<video controls>
    <source src="https://tree.s-ul.eu/ZFJUxAyp.mp4" type="video/mp4">
</video>

As you can hear, while 1/2 notes are perfectly where they are supposed to,
they're not exactly the beats we want to track - 1/1 notes, indicated by white
lines, are much better moments to potentially sync an edit to.

Afterwards, it properly stays on 1/1 notes for a longer while, until this
section:

<video controls>
    <source src="https://tree.s-ul.eu/sRkvfih5.mp4" type="video/mp4">
</video>

Here the algorithm also jumps between white and red lines, this time
understandably so, because of the irregularity of the beat and vocal samples.
Seems to go back to 1/1 notes after this section and stays on them until the
end of the song.

## Conclusion

From what we're seeing in the results above, I think `librosa` will be a good
choice to begin working on the beat-synced video editing functionality in
Pitivi.

It's not ideal, but in most cases it'll be well within the 'good enough'
range of results for people looking to quickly synchronise their edits to a
catchy pop song, for example.

Thank you for reading this (rather lengthy) post and I hope it was useful in
showcasing the possibilites of a modern beat-tracking library like `librosa`.
See you next time!
