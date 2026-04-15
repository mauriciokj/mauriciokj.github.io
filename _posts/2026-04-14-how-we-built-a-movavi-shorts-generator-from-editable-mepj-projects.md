---
layout: default
title: "How we built a Movavi Shorts generator from editable .mepj project files"
date: 2026-04-14 21:15:00 -0300
categories: [movavi, automation, shorts, jekyll]
tags: [movavi, shorts, automation, mepj, json, python, video-editing]
---

![Movavi Shorts generator blog hero](/assets/images/posts/2026-04-14-movavi-shorts-generator/movavi-shorts-generator-blog-hero.png)

A few days ago I started with a simple idea: I wanted a repeatable way to generate Movavi projects for short-form videos without rebuilding everything manually every time.

The result turned into a small but very useful tool: a local **Movavi Shorts generator** that reads a template `.mepj`, swaps the audio and images, keeps the visual style intact, and produces a new project ready to open in Movavi.

If you want to check the project or contribute, the repository is here:

<https://github.com/mauriciokj/shorts-project-generator>

## What I wanted to solve

The manual workflow was always the same:

- open a template project
- replace the audio
- replace the images
- keep the visual style
- avoid breaking the project structure
- test it in Movavi again

That sounds simple until you try to automate it.

Movavi project files are editable, but they are also picky. If you remove the wrong clip, alter the wrong field, or rebuild too much structure, the app can refuse to open the project or silently discard media.

So the real challenge was not just “generate a file”. The challenge was:

> generate a file that Movavi actually accepts.

## What we built

The generator now does the following:

1. opens a `.mepj` template
2. extracts `config.json` and `meta.json`
3. finds the audio clip and updates it with the new MP3
4. finds the image clips and replaces their paths
5. recalculates timing based on the audio length
6. syncs image metadata such as width, height, format, and size
7. keeps the original template look and feel as much as possible
8. writes a new `.mepj` file

In practice, that means I can now create a new short project from:

- one MP3
- four images
- a template base

## The breakthrough

The biggest breakthrough was realizing that **structure matters more than cleanup**.

At first I tried aggressively removing leftover clips, subtitles, and inherited items from the template. That seemed logical, but Movavi did not always like it. In some cases, removing extra visual clips directly from the JSON made the project unstable.

What worked much better was:

- preserving the structure of the project
- replacing media conservatively
- keeping the template base stable
- embedding a fallback template inside the project itself

That fallback is important because it means the generator does not depend on a fragile external file to keep working.

## The new default template

I also created a new base project:

- `padrao.mepj`

That file became the better default template because it already starts closer to the final shape I want:

- 4 images
- no unwanted subtitle clutter
- a cleaner starting structure

So the generator now has a more reliable foundation.

## Why this matters

This is useful for me because I produce a lot of short-form boxing and sports content. Instead of opening Movavi and rebuilding the same layout over and over, I can now:

- pick a story
- generate the audio
- choose the images
- create the project automatically
- review and export faster

That saves time and makes the workflow repeatable.

## Repository

If you want to look at the code, test it, or improve it, here is the repository again:

<https://github.com/mauriciokj/shorts-project-generator>

## What’s next

The next steps are:

- make the generator even more template-agnostic
- improve cleanup rules safely
- support more workflow presets
- keep reducing manual editing inside Movavi

In short: the goal is not to replace Movavi. The goal is to make Movavi behave like part of a repeatable production pipeline.
