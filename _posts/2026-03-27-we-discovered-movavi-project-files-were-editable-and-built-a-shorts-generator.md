---
layout: default
title: "We discovered Movavi project files were editable — and built a Shorts generator"
date: 2026-03-27 00:53:00 -0300
categories: [automation, video, productivity]
tags: [movavi, automation, python, shorts, video-editing, reverse-engineering]
---

Editing short-form videos in batches sounds easy — until you repeat the exact same workflow over and over again.

In our case, the process always looked roughly the same:

- write the script
- generate the voice-over
- gather the images
- open Movavi
- import everything
- adjust the timeline
- review transitions
- export the final video

None of that was especially difficult on its own. The real problem was repetition.

The question that changed everything was very simple:

**what if the Movavi project file was editable?**

That question kicked off an experiment that ended up becoming something much more interesting: a functional proof of concept for a short-form video project generator.

---

## The problem: too much repetitive work for short videos

When you work with repeatable short-form formats, it does not take long to notice that part of the editing process is not really creative work.

A lot of the time, the workflow is basically this:

- swap the main audio
- swap the images
- redistribute the clips on the timeline
- keep an approved visual template
- open the editor just for review and export

In other words: a lot of the work was operational, not creative.

It felt like this should be automatable somehow.

---

## The hypothesis: maybe `.mepj` was not a black box

Movavi stores projects as `.mepj` files.

The first question was whether that file was a completely proprietary black box — or whether there was something readable inside it.

The answer was better than expected.

When we inspected the file, we discovered that a `.mepj` file was, in practice, a **compressed archive** containing at least:

- `meta.json`
- `config.json`

That meant the project was not fully opaque. There was editable structure inside.

That was the first major turning point.

---

## What we found inside the project

The most important file was `config.json`.

That was where the project structure became useful for automation.

It contained things like:

- the timeline
- clips
- absolute media paths
- timing information
- transitions
- audio references
- image references
- effects applied to clips
- even subtitle elements inherited from the template

In other words, `config.json` worked like a map of the edit.

If the `.mepj` file is the package, `config.json` is the editing blueprint.

---

## The first proof of concept

Once we confirmed that, the first test was straightforward:

1. take a real Movavi project we had already built
2. use it as a template
3. programmatically replace:
   - the voice-over audio file
   - the image paths
4. repack everything as a new `.mepj`
5. try opening it in Movavi

The goal at that stage was not perfection.

It was just to answer the main question:

**would Movavi accept a project generated like this?**

The answer was: **yes**.

The file opened.

That validated the core idea.

---

## What was still wrong at first

The first generated project opened successfully, but it was far from perfect.

Some of the issues were:

- not all images were replaced correctly
- the audio duration was still inheriting the wrong timing from the template
- subtitles from the original project were still present
- parts of the timeline were still carrying leftovers from the previous edit

So the foundation was viable, but we still needed to understand the internal project structure much better.

---

## Understanding how audio and images were represented

As we kept digging, it became clear that the project stored a lot of very useful information.

### For audio
We were able to locate:

- the MP3 path
- file size
- media length (`length`)
- clip duration on the timeline (`timing.duration`)
- source duration (`timing.sourceDuration`)

That allowed us to make one major improvement:

instead of inheriting the wrong duration from the template, we started measuring the real MP3 length with `ffprobe` and synchronizing those fields with the actual audio duration.

### For images
We were also able to identify:

- image paths
- the main visual clips on the correct track
- timestamps
- per-image duration
- transition settings

That made it possible to distribute the visual timeline based on the audio duration.

The logic was simple:

- if the audio is 55.4 seconds long
- and the video uses 4 images
- each image gets roughly one quarter of the total duration

That turned timeline generation into something predictable and automatic.

---

## The template subtitle could also be removed

The base project contained a subtitle clip inherited from the original template.

That clip showed up with this name:

```txt
#Subtitle_template#
```

Once we detected that in the JSON, we could simply filter it out before generating the new project.

Result: the generated `.mepj` opened without the subtitle from the previous edit contaminating the new one.

---

## The effects were there too

Another important discovery was that the template did not just store media references.

The visual clips also contained information such as:

- `effects`
- `cropEnvelope`
- `moveEnvelope`

In practice, that meant the template already carried part of the visual language of the edit:

- zoom behavior
- image movement
- crop animation
- visual treatment applied to the clip

That was great news because it meant we did not need to rebuild everything from scratch. In many cases, we only had to preserve those envelopes and effects from the original template.

That made the automation far more useful. The generated project was not just functional — it also inherited part of the visual finish.

---

## The result: a short-form project generator

At that point, the proof of concept stopped being just a one-off hack and started looking like an actual internal tool.

That is how the:

## `shorts-project-generator`

started taking shape.

The idea is simple.

You provide:

- a template `.mepj`
- an MP3 voice-over
- a list of images
- an output path

And it generates a new `.mepj` with:

- updated audio
- updated images
- timing distributed across images
- inherited subtitle removed
- basic transitions preserved
- a project ready to open in Movavi

The goal is not to replace the editor.

The goal is to eliminate the mechanical work and leave only the final review/export step inside Movavi.

---

## What has already been validated

At this point, we were able to validate that:

- `.mepj` files are editable
- Movavi accepts projects generated from a template
- audio and images can be replaced programmatically
- the real MP3 duration can be measured and applied to the timeline
- inherited subtitle clips can be removed
- a good part of the template’s visual structure can be preserved
- a usable project can be generated in practice

And the best part is: **none of this required AI**.

This part is just automation.

---

## What can still be improved

Even with the proof of concept working well, there are still obvious improvements to make:

- map every auxiliary clip in the template more precisely
- make transition rules configurable
- preserve zoom/effects more intelligently across generated clips
- handle variable numbers of images more elegantly
- build a simple interface instead of relying only on a CLI

But the important part already happened:

**the idea moved from hypothesis to functional beta tool.**

---

## Conclusion

What makes this experiment interesting is that it started from a very simple operational question:

**do we really need to repeat this manually every single time?**

The answer turned out to be no.

By investigating the Movavi project format, we found a clear path to automation. Starting from a real template, we were able to generate new projects with fresh audio, fresh images, updated timing, and basic structure already solved.

It does not replace the editor — at least not yet.

But it already works as a very useful shortcut for a kind of work that used to consume too much time for too little actual creative decision-making.

And that is often the best type of automation: less glamour, more leverage.

---

## Want to test `shorts-project-generator`?

If you want to test `shorts-project-generator`, leave a comment.
If there is enough interest, I can put it on GitHub for people to try.
