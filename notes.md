# notes

learning log for mandelbrot forever. first person, messy on purpose, for future me. this one was built by two sessions working at the same time (one on the look, one on the engine) plus a bunch of my own debugging, so the story is a bit tangled.

### design direction (one line, as the guide asks)
a dark instrument. the fractal is the only thing on the page allowed to have color, everything else is near-black and gray with one muted violet accent used maybe three times. no gradients on chrome, no glass, no glow, no entrance animation.

### measured numbers (real runs, screenshots, not estimates)

| view | engine | time |
|---|---|---|
| default | gpu | 9 to 14 ms |
| default | cpu, 16 workers | ~148 ms |
| seahorse valley, 1000 iter | gpu | ~15 ms |
| double spiral | gpu | 21 ms |
| deep bookmark, 3000 iter | gpu | 48 ms |
| deep bookmark, 3000 iter | cpu, 15 workers | 1674 ms |
| island at 6.4e10x | gpu | 81 ms |
| needle tip | gpu | renders at 3.2e20x |

main-thread cost per frame during a glide: about 0.4 ms at 8550 iterations. so glides are gpu-bound, not js-bound.

double-double vs plain doubles, checked against a 60-digit bigint reference orbit: double-double matched to the last bit after 400 iterations, plain doubles drifted to 1e-13.

### the bug that cost the most time: the torn stripes
symptom: after any zoom the image had alternating horizontal bands, some rows fully rendered, some rows still the blocky preview. it looked interlaced. i chased it for way too long in the wrong places.

what i thought it was, in order, and why each was wrong:
1. stale strips from an old render job being painted. added a job id guard. stripes stayed.
2. a worker index bug (`workers[y/STRIP % NW]` giving a fractional index). switched to a counter. stripes stayed.
3. the glide animation flooding workers with 60 renders a second. added a one-render-in-flight gate. stripes stayed.
4. web workers not returning at all in headless chrome. this one was REAL but a different problem (see below), and it hid the actual bug because i could never see a finished frame.

the actual cause, found by both sessions independently once someone could look at a real browser: `ctx.putImageData(img, 0, y0, 0, y0, W, h)`. the third arg is where the image ORIGIN lands, and the fifth is the dirty rect's y inside the image. i was passing y0 for both, so every strip landed at canvas row 2*y0. rows 30..45 painted at 60..75. half the rows landed in the wrong place, half never got painted. fix was `putImageData(img, 0, 0, 0, y0, W, h)`. one argument. proof: row 45 of the computed values matched canvas row 90 at 872 of 1062 pixels and canvas row 45 at only 323.

lesson: when a bug survives three "fixes", stop theorizing and go LOOK at the output. i was debugging blind.

### assumption that was wrong: headless chrome is a fine substitute for a browser
it is not, for anything with web workers. under `--virtual-time-budget` the workers never returned for any real-sized strip (64px wide or more), but the identical code returned all 45 strips in 130 ms under node. so headless said "stuck" when the page was fine, and at the same time it could not show me the stripe bug that was actually there. i burned a lot of time on a testing artifact. now in the building guide: for workers, animation, or anything time-based, take a real screenshot or use the browser tool, do not trust dump-dom.

### assumption that was wrong: 64-bit floats would be enough
i assumed doubles would carry a deep zoom fine and the limit would be the gpu's 32-bit floats. turns out doubles die at ~1e13x too, just later, and in a sneakier way: the VIEW freezes because one pixel of pan is smaller than the gap between two adjacent doubles. the zoom keeps "working" but you can't move. fix was double-double for the view center and the reference orbit.

### why perturbation instead of just more precision
the gpu only does 32-bit. you cannot upload doubles to a shader. perturbation sidesteps it: compute ONE orbit at the center in high precision on the cpu, and each pixel only tracks its small delta from that orbit, which stays small enough for 32-bit even at 1e20x. the rebasing (zhuoran) part is what stops the delta from blowing up when a pixel's orbit drifts far from the reference. without rebasing you get glitches, with it you get the needle at 3.2e20x. this was the single biggest idea in the project.

### why oklab for the palettes
blending colors in plain rgb goes gray and muddy in the middle of a gradient. oklab is a perceptual space, so a blend from orange to purple actually passes through nice colors. the difference is obvious side by side. cheap to do, big visual win.

### why the two-session thing bit me once
one session committed a restrained "strip it back" design. the other session's first engine commit overwrote it by accident. the second engine commit restored the restrained direction with the new engine underneath. lesson: when two things touch the same file, one owns the css and one owns the js, and say so up front.

### to try later
- arbitrary-precision reference orbit so it goes past 1e26x
- confirm whether the island bookmark is real interior (run way more iterations offline)
- series approximation to skip the first thousands of iterations on deep zooms
- a proper "record a zoom video" button
- interior coloring (distance estimation) so the black isn't flat
