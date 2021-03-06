# glif.js

glif.js is a fork of [omggif](https://github.com/deanm/omggif) that speeds up gif decoding using WebGL.

# install

For those using [bower](http://bower.io/)
```bash
$ bower install glif.js
```

# usage

## raw decoding

```javascript
var gr = new GifReader(byteArray);
var glif = new GLIF(canvas);

gr.decodeAndGLIF(frame_num, glif);
```

* `byteArray` is a `Uint8Array` containing the data to be decoded.
* `canvas` is a `HTMLCanvasElement` where the gif will be displayed.
* `frame_num` is the frame to display

## animating

Use `viewer.js` to help animate the gif.

```javascript
var view = viewer(canvas, data, callback);
var timeoutID = view(false);
```

* `canvas` is a `HTMLCanvasElement` where the gif will be displayed.
* `data` is one of `ArrayBuffer`, `Uint8Array`, or `GifReader` (from `omggif`).
* `callback` is a function that is called after each frame is drawn. One parameter is passed - the `timeoutID` of the `setTimeout()` call. `clearTimeout(timeoutId)` will stop the animation.

`viewer` returns a function (`view`) to draw the next frame of the gif. The function takes one argument, `once`. If `once` is true, then only one frame is drawn. The function returns the `timeoutID` of the _first_ frame.

# how

GIF decoding is slow because LZW decompression must be performed serially. But the next step, mapping palette indices to RGB colors, is easily parallelizable by the GPU.

More specifically, it turns this slow `for` loop:

```javascript
for (var i = 0, il = index_stream.length; i < il; ++i) {
	var index = index_stream[i];

	if (xleft === 0) {  // Beginning of new scan line
		op += scanstride;
		xleft = framewidth;
	}

	if (index === trans) {
		op += 4;
	} else {
		var r = buf[palette_offset + index * 3];
		var g = buf[palette_offset + index * 3 + 1];
		var b = buf[palette_offset + index * 3 + 2];
		pixels[op++] = r;
		pixels[op++] = g;
		pixels[op++] = b;
		pixels[op++] = 255;
	}
	--xleft;
}
```

into this fragment shader:

```glsl
varying highp vec2 vTextureCoord;

uniform sampler2D uIndexStream;
uniform sampler2D uPalette;

uniform bool uTransparency;
uniform mediump float uTransparent;

void main(void) {
	mediump float uIndex = texture2D(uIndexStream, vTextureCoord).r;

	if (uTransparency && uIndex == uTransparent) {
		gl_FragColor = vec4(0.0, 0.0, 0.0, 0.0);
	} else {
		gl_FragColor = texture2D(uPalette, vec2(uIndex, 0.5));
	}
}
```

# License

```
(c) Dean McNamee <dean@gmail.com>, 2013.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to
deal in the Software without restriction, including without limitation the
rights to use, copy, modify, merge, publish, distribute, sublicense, and/or
sell copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS
IN THE SOFTWARE.
```