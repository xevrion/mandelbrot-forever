# mandelbrot forever

a shape with an infinitely detailed edge. zoom in anywhere on the boundary and there's always more, and all of it comes from one line of math repeated over and over. this page lets you fly into it at hundreds of billions of times magnification, on your gpu, in one html file.

live: <https://xevrion.github.io/mandelbrot-forever/>

## how it works

the whole thing is the formula **z = z^2 + c**. pick a point c on the screen, start z at zero, keep squaring and adding. if z stays small forever the point is in the set (black). if it flies off to infinity it's outside, and the color is how many steps that took. every pixel is that one loop:

```js
function escape(cx, cy, maxIter){
  let x=0, y=0, i=0;
  while(x*x + y*y <= 4 && i < maxIter){
    const xt = x*x - y*y + cx;   // real part of z^2 + c
    y = 2*x*y + cy;              // imaginary part
    x = xt; i++;
  }
  return i;
}
```

the `<= 4` test is the only trick in the basic version. once z gets more than 2 from the origin it's mathematically guaranteed to escape, so you stop early. that's why this runs in a browser at all.

### why it's smooth

coloring by the raw integer step count gives hard stripes. after a point escapes you look at how far past 2 it overshot and turn that into a fractional step count (37.62 instead of 37). the palettes are blended in oklab so the gradients don't go muddy in the middle, and the mapping is log plus sqrt so the outside glows gradually instead of one flat band.

### why it's fast (the gpu part)

every pixel is independent, which is exactly what a gpu is for. with webgl2 the entire image is one fragment shader and a frame takes single-digit milliseconds, so every frame of a zoom glide is a real render.

the catch: gpus only do 32-bit floats, which die at around 100,000x zoom. the fix is **perturbation**. the cpu computes one high-precision orbit for the center of the screen and uploads it as a texture. each pixel then only tracks its tiny offset from that reference, and offsets stay small enough for 32-bit math even at absurd zooms. when an offset drifts too far it rebases onto a new point of the reference orbit (the zhuoran trick) so it never blows up.

### why panning works deep

past about 1e13x, one pixel of pan is smaller than the gap between two representable doubles, so the view would freeze. the view center and the reference orbit use **double-double** arithmetic (two doubles glued together, about 32 digits). verified against a 60-digit bigint orbit: double-double matched to the last bit after 400 iterations, plain doubles had drifted to 1e-13. the needle tip renders at 3.2e20x.

### when there's no gpu

it falls back to plain javascript across web workers, one per core, same math (perturbation once deep). the image is cut into small strips, workers pull them off a shared queue so fast cores just take more, a low-res pass shows first, and during motion it stretches the last finished frame so nothing tears. there's a live gpu / cpu switch in the toolbar so you can feel the difference.

## usage

open `index.html`. no server, no build, nothing to install.

```sh
xdg-open index.html
# or, if your browser is fussy about file://
python3 -m http.server 8000
```

click to glide into a spot. scroll to zoom. drag to pan. pinch on touch. hover for the julia set in the corner. the url updates as you move, so copy it to share any place you find. keys: `+` `-` zoom, arrows pan, `r` reset, `j` toggle julia.

## limits

| thing | reality |
|---|---|
| past ~1e26x | double-double runs out of digits, you'll see blocks or noise. real deep-zoom tools use arbitrary precision |
| gpu availability | needs webgl2. if missing or lost mid-session it drops to cpu automatically, which is slower |
| the "island" bookmark | the black shape there is an iteration-limited halo, not confirmed interior. looks right, not proven |
| glide smoothness | main-thread cost is ~0.4 ms per frame so it should be gpu-bound, but only you can feel it on your machine |
| the conjecture itself | this page draws it, it proves nothing about anything |
