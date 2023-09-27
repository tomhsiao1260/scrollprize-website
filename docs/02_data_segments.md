---
title: "Segments"
hide_table_of_contents: true
---

<head>
  <html data-theme="dark" />

  <meta
    name="description"
    content="A $1,000,000+ machine learning and computer vision competition"
  />

  <meta property="og:type" content="website" />
  <meta property="og:url" content="https://scrollprize.org" />
  <meta property="og:title" content="Vesuvius Challenge" />
  <meta
    property="og:description"
    content="A $1,000,000+ machine learning and computer vision competition"
  />
  <meta
    property="og:image"
    content="https://scrollprize.org/img/social/opengraph.jpg"
  />

  <meta property="twitter:card" content="summary_large_image" />
  <meta property="twitter:url" content="https://scrollprize.org" />
  <meta property="twitter:title" content="Vesuvius Challenge" />
  <meta
    property="twitter:description"
    content="A $1,000,000+ machine learning and computer vision competition"
  />
  <meta
    property="twitter:image"
    content="https://scrollprize.org/img/social/opengraph.jpg"
  />
</head>

Segmentation is the mapping of sheets of papyrus in a 3D X-ray volume. The resulting surface volumes can be used directly to look for ink. The community has made this a significantly more automated process, but it still involves human input.

We have a small group of contractors and volunteers who have been mapping the scrolls, mostly Scroll 1.

<figure>
<iframe width="484" height="293" seamless frameborder="0" scrolling="no" src="https://docs.google.com/spreadsheets/d/e/2PACX-1vRQxQefw-7rl3Hnt1Q7MFpI27FtzsvFo2x9q6egW8vN5am8QlQLE20BAjOSPZ2teztjdgMUOGc6FV7Y/pubchart?oid=1982586813&amp;format=interactive"></iframe>
<figcaption className="mt-0">Progress of mapping the scrolls, in area (cm²), from the <a href="https://docs.google.com/spreadsheets/d/1zC_5vkqWgb_5z4Q9BYsETF7_3r1BYPccdAnS_GRYOaQ/edit#gid=2051117465">Segment Directory spreadsheet</a> (dates are when a segment was started, not finished, so it lags behind a bit)</figcaption>
</figure>

The main tool which our segmenting team currently uses is Volume Cartographer, which you can learn more about in [Tutorial 3](tutorial3). The basic idea is to manually annotate a piece of papyrus in a single slice, after which an algorithm can extrapolate it into 3D, essentially by doing really fancy line-following. This algorithm still needs a lot of correction and supervision, but that’s the rough idea.

<figure className="max-w-[500px]">
  <video playsInline muted controls className="w-[100%] rounded-xl" poster="/img/tutorials/vc-extrapolation2.jpg">
    <source src="/img/tutorials/vc-extrapolation2.webm" type="video/webm"/>
    <source src="/img/tutorials/vc-extrapolation2.mp4" type="video/mp4"/>
  </video>
  <figcaption className="mt-0">Illustration of Volume Cartographer with an algorithm extrapolating in 3D.</figcaption>
</figure>

The resulting 3D shape is called a “mesh”. You can view the meshes of all our segments in a tool called [Volume Viewer](http://37.19.207.113:5173/) (select “segments” in the top right):

<figure className="max-w-[500px]">
  <div className="w-[100%]"><div className="overflow-hidden mb-2"><img loading="eager" src="/img/data/segmentation-animation.webp" className="w-[100%] mt-[-30px] mb-[-50px]"/></div>
  <figcaption className="mt-0">Some segments from Scroll 1</figcaption></div>
</figure>

For more technical details about how the segmentation team operates, check out this doc: [The Segmenter’s Guide to Volume Cartographer (for contractors)](https://docs.google.com/document/d/11B9Gy1gJRye_NQHphwbIxINvactUchJJsJOJi1FKrgI/edit?usp=sharing).

## Data format

The `.volpkg` format used by Volume Cartographer (learn more in [Tutorial 3](tutorial3)) has a `paths` folder, in which each segment has its own subfolder. Each subfolder contains two files:

* `Scroll1.volpkg/paths/<id>/meta.json`: Metadata of a segment.
* `Scroll1.volpkg/paths/<id>/pointset.vcps`: Pointset of a segment. This is a custom data format specific to Volume Cartographer.

There are two main places in which you can find segments on our [data server](http://dl.ash2txt.org/full-scrolls/):
* [`/full-scrolls/Scroll{1,2}.volpkg/paths`](http://dl.ash2txt.org/full-scrolls/Scroll1.volpkg/paths/): This is where we put finished paths long term.
* [`/hari-seldon-uploads/team-finished-paths/scroll{1,2}/`](http://dl.ash2txt.org/hari-seldon-uploads/team-finished-paths/scroll1/): This is the staging area for segments made by the segmentation team (led by @Hari_Seldon on Discord).
  * Every few weeks we move files from here to the main `/paths` folder.
  * We also update [Volume Viewer](http://37.19.207.113:5173/) when we do this.

## Surface volumes

In addition to the `meta.json` and `pointset.vcps` required by Volume Cartographer, we generate a bunch of other files:

* `/layers/{00-64}.tif`: Surface volumes of 65 layers.
  * This is most useful for detecting ink on.
* `/<id>_points.obj`: Pointcloud.
* `/<id>.obj`: Mesh of the segment.
* `/<id>.tif`: Texture of the surrounding voxels.
* `/<id>.ppm`: Custom data format mapping points between the surface volume and the original 3D volume of the scroll.
* `/author.txt`: Name of the author of the segment.
* `/area_cm2.txt`: Total surface area, in cm<sup>2</sup>.

The surface volume is the most useful dataset for ink detection. The middle layer (32) is sampled right from the segment mesh, and the other layers are sampled “above” and “below” the surface. This results in a new 3D volume “around” the surface of the segment.

<figure className="max-w-[600px]">
  <img src="/img/data/surface_volume.gif"/>
  <figcaption className="mt-0">Scrubbing through layers of the surface volume of <a href="http://dl.ash2txt.org/full-scrolls/Scroll1.volpkg/paths/20230827161846/layers/">segment 20230827161846</a></figcaption>
</figure>

All these extra files were generated using the following script: `export SLICE=20230827161846 && cd /Scroll1.volpkg/paths/${SLICE} && nice vc_convert_pointset -i pointset.vcps -o "${SLICE}_points.obj" && nice vc_render -v ../../ -s "${SLICE}" -o "${SLICE}.obj" --output-ppm "${SLICE}.ppm" && mkdir -p layers && nice vc_layers_from_ppm -v ../../ -p "${SLICE}.ppm" --output-dir layers/ -r 32 -f tif --cache-memory-limit 50G && vc_area ../.. ${SLICE} | grep cm | awk '{print $2}' > area_cm2.txt`

## Monster segment

The EduceLab team created special tooling ([“Quick Segment”](https://github.com/educelab/quick-segment/)) to create the “Monster Segment”: a massive segment in Scroll 1. At the time of writing, it is larger than all our other segments combined, so we recommend trying this one first. More information [here](https://discord.com/channels/1079907749569237093/1079907750265499772/1104116512396161144) and [here](https://discord.com/channels/1079907749569237093/1079907750265499772/1102710816656064663).

The data for this can be found on the data server at [`/stephen-parsons-uploads/`](http://dl.ash2txt.org/stephen-parsons-uploads/). There are two segments, which were created by projecting in different directions from a large air gap. The `recto` segment is the one where we expect written text.

<figure className="max-w-[700px]">
  <img src="/img/first-letters/Scroll1_wrap_recto_smaller.png" className="w-[100%]"/>
  <figcaption className="mt-0">Monster Segment texture</figcaption>
</figure>

<figure className="max-w-[700px]">
  <img src="/img/first-letters/monster_slices.png" className="w-[100%]"/>
  <figcaption className="mt-0">Location of the Monster Segment in the scroll</figcaption>
</figure>