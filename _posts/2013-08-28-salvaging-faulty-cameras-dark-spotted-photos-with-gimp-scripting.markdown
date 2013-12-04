---
title: Salvaging spotty photos from a faulty camera with Gimp scripting
link: salvaging-spotty-photos-with-gimp
excerpt: Photos and videos of a family event turned out with numerous 4-6 pixel-wide dark spots due to sensor malfunctioning. Gimp's scripting capability came in quite handy in salvaging them.
tags: linux, gimp, scripts, scheme
layout: post
---

At a family get-together recently, some of the photos and videos were taken with a Sony Xperia Z. Unfortuantely, apart from the usual lack of quality from smartphone cameras and poor indoor lighting, these turned out to have quite prominent dark blotches throughout [<sup>\*</sup>](#note1). Manually healing the handful of choice photos one-by-one with Gimp's [heal tool](http://docs.gimp.org/en/gimp-tool-heal.html) wouldn't have been too difficult but it was out of the question for the 15 min video footage. This was a good opportunity to learn a little bit of scheme and Gimp scripting.

After dumping the frames of video clips into individual images with mplayer, I picked up the coordinates of the dark spots with the idea of healing a small area (20 sq. px) around each of the spots with the 'heal selection' script that comes with [resynthesizer plugin](https://github.com/bootchk/resynthesizer). Frustratingly the spots weren't at the same position in all the frames - focus changes, camera pans and in the case of photos, resolution differences all presumably caused the spots to move around. I tried to simply call the heal script with larger area selections but instead of dark spots, I now had distortions that were even worse when the frames were stitched back together. 

So instead I went with a two stage spot detection approach. First pick approximate locations of the dark spots and choose a 150x150 px wide selection around each. Then use edge detection filters to isolate the spots, pick up a small selection 20-30 sq. px around them on which heal script could be run. With a bit of trial and error with parameters, I got the edge detect plugin to mostly ignore other edges that may be included in the initial crude selection (like face outlines, furniture, prominent details etc) And even when image details were picked up, since the selections were very small (20 sq. px in a 1920x1080 frame), the distortion was not very noticeable.

On our 2011 Core-i5 mac, I got about 8000 odd frames (around 5 min video) processed per hour with 4 instances of gimp running the script. It could probably be better with some optimizations but it was more than enough for my needs.


#### Image Enhancement

Probably because the photos were indoor ones and mostly without flash, the camera chose a high ISO resulting in lot of [CCD noise](http://gimpguru.org/Tutorials/ReducingCCDNoise/). Blur techniques outlined in most of the guides about cleaning CCD noise didn't quite seem satisfying to me with these photos. But [channel based wavelet denoising](http://wiki.meetthegimp.org/doku.php?id=noise) tip on _meetthegimp_ worked wonderfully well. High values in the wavelet denoise settings for Cb and Cr channels, without touching the luma(Y) channel too much, removed chroma noise drastically. [G'Mic](http://gmic.sourceforge.net/gimp.shtml) anisotropic blur smoothened out the graininess quite well.

#### mp4 moov corruption

Another hurdle in all this was that one of the mp4 videos from the camera was corrupt. Hexdump showed data all over instead of zeros but every player I tried failed to play it. A bit of googling and use of ffmpeg debug flags showed that the 'moov' atom - which contains all the mp4 timing metadata, frame start and size information etc - wasn't present. I did remember android framework restarting while recording one of the videos. Apparently the camera app had crashed while capturing and hadn't written anything apart from the raw AV data. Search turned up [untrunc](http://vcg.isti.cnr.it/~ponchio/untrunc.php) which did ferret out audio but failed with video. [A maemo thread](http://talk.maemo.org/showthread.php?t=84800) linked to [Grau Video Repair Utility](http://grauonline.de/cmsimple2_6/en/?Solutions:HD_Video_Repair_Utility) which extracted video as well (only half of it in demo version, but that's more than enough again)

-----

#### Commands and Code
##### Dump frame jpegs

    $ mplayer -nosound -vf harddup\
    -vo jpeg:progressive:quality=99 spotty-video-file 

OR

    $ ffmpeg -i spotty-video-file -vsync 1 -f image2 'frame-%05d.jpg'
-----

##### Re-encode corrected jpegs into video
To avoid AV sync issues, fps value below needs to match the original video's frame rate.

    $ mencoder "mf://\*fixed.jpg" -mf fps=29.7085 -vf harddup -of rawvideo -o \
    output.h264 -ovc x264 -x264encopts \
    subq=6:partitions=all:8x8dct:me=umh:frameref=5:bframes=3:b_pyramid=normal:weight_b:threads=auto
-----

##### Rip audio stream and mux in the fixed video to a new mp4

    $ MP4Box -raw 2 _original-video_
    $ MP4Box -new output.mp4  -add output.h264:fps=29.97 -add \*aac
-----

##### Gimp script to heal black spots in video frames/photos

    ; function name and args
    (define (fixdots pattern)
    (let* ((filelist (cadr (file-glob pattern 1))))
        (while (not (null? filelist)) ; loop over files matching the glob
            (let* (
                    (filename (car filelist))
                    (newfilename (string-append 
                                    (substring filename 0 (- (string-length filename) 4)) 
                                    "-fixed.jpg"))
                    (image (car (gimp-file-load RUN-NONINTERACTIVE filename filename)))
                    )

                ; approximate co-ords of the spots ; make selections around each
                (let pickSpots ((spots '((20 120) (300 35) (1035 145) (1730 8) (1050 45))))
                ; block below is like a while loop,
                ; pops a list and runs till it still has items
                (cond ((not (null? spots))
                        (apply (lambda (x y)
                                (gimp-image-select-rectangle image 0 x y 175 175)
                                )
                                (car spots))
                        (pickSpots (cdr spots))
                        )
                        )
                )
                ; new layer from selection so we work only on spots, not whole image
                (define layer (car (gimp-image-get-active-layer image)))
                (gimp-edit-cut layer)
                (define newlayer (car (gimp-layer-new-from-visible image image "dots")))
                (gimp-image-insert-layer image newlayer 0 -1)
                (gimp-drawable-fill newlayer 3)
                (define sel (gimp-edit-paste newlayer FALSE))
                (gimp-floating-sel-anchor (car sel))

                ; duplicate the spots layer to run edge-detection on
                (define duplayer (car (gimp-layer-new-from-drawable newlayer image)))
                (gimp-image-insert-layer image duplayer 0 0)
                (plug-in-edge RUN-NONINTERACTIVE image duplayer 1.5 2 5)

                ; within the edges, further improve spot detection by
                ; selecting based on nearness to black
                (gimp-context-set-sample-threshold-int 20)
                (gimp-image-select-color image 0 duplayer '(255 255 255))
                (gimp-image-remove-layer image duplayer)
                (gimp-image-merge-visible-layers image 0)

                (gimp-selection-grow image 7)
                ; back to the layer with actual photo
                (define layer (car (gimp-image-get-active-layer image)))
                (python-fu-heal-selection RUN-NONINTERACTIVE image layer 25 2 0)
                (gimp-selection-none image)
                (file-jpeg-save RUN-NONINTERACTIVE image layer newfilename newfilename 0.95 0 1 1 "" 1 1 0 1)
                (gimp-image-delete image)
                (set! filelist (cdr filelist)) ; like a while loop
                )
            )
        )
    )

* Save the scheme script in Gimp's script directory (~/Library/Application Support/Gimp/Scripts for Mac)
* Call gimp executable like this in the directory containing frame jpgs

        $ gimp-2.8 -i -s -d -b '(fixdots "\*.jpg")' -b '(gimp-quit 0)'
------

<a name="note1"><sup>*</sup></a> 
Apparently this is a somewhat [widely seen](https://encrypted.google.com/search?hl=en&q=xperia%20z%20camera%20dark%20spots) problem with Xperia Zs. The particular phone in question was replaced without any hassle by Sony India.

 
