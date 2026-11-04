# How Colors Work: From Light to Implementation

How color values are represented, converted, and rounded, stated mathematically and independently
of any library. Implementation notes on culori and Figma are in the
[case study](./case-study-color-values-in-figma.md). Sources are linked where they are used, and collected under [References](#12-references).

## Contents

1. [Vocabulary](#1-vocabulary)
2. [Numeric representations](#2-numeric-representations)
3. [Channels and alpha](#3-channels-and-alpha)
4. [Discretization and rounding](#4-discretization-and-rounding)
5. [Linearization](#5-linearization)
6. [Color spaces and gamut](#6-color-spaces-and-gamut)
7. [RGB, HSL, and HSB](#7-rgb-hsl-and-hsb)
8. [Luminance and contrast](#8-luminance-and-contrast)
9. [Finding a tone at a target contrast](#9-finding-a-tone-at-a-target-contrast)
10. [Writing values for reuse](#10-writing-values-for-reuse)
11. [Other models](#11-other-models)
12. [References](#12-references)

## Part I: Light and vision

### 1. What color is before there are numbers: light and spectra

Start without any numbers at all, with a lamp in a room.

Light is electromagnetic radiation, and the part we can see has wavelengths from roughly 380 to 780 nanometers. A single wavelength is a pure spectral light, and moving through the range gives the familiar rainbow:

![The visible spectrum as a horizontal band, running from violet near 380 nanometers through blue, green, yellow, and orange to red near 750 nanometers.](assets/ref-visible-spectrum.svg)

*Image: Gringer, public domain, via Wikimedia Commons.*

Almost no light you encounter is a single wavelength. A light source emits some power at every wavelength in a range, and the complete description of a light is its **spectral power distribution**, or spectrum: a function that gives power for each wavelength, S(λ). That function has hundreds of meaningful degrees of freedom, and two white lights can have very different ones:

![A chart of relative power from 380 to 780 nanometers. The D65 curve is jagged and roughly level around 100, while the illuminant A curve rises smoothly from about 10 at 380 nanometers to about 240 at 780 nanometers. They cross at 560 nanometers.](assets/fig-12-light-sources.svg)

Both curves are standards of the International Commission on Illumination (CIE). [Illuminant D65](https://cie.co.at/datatable/cie-standard-illuminant-d65) represents average daylight, and it is a measured, tabulated shape. Illuminant A represents a tungsten filament, and it follows directly from Planck's law for a hot body at 2848 K. Comparing 700 nm with 400 nm, the filament has more than ten times as much long-wave power, relative to short-wave power, as daylight. Yet in a room lit by either, once your eyes settle, a sheet of paper looks white.

Most objects do not emit light. They reflect part of what falls on them, with a reflectance R(λ) from 0 to 1 at each wavelength, so what reaches your eye is the product at every wavelength:

```
light reaching the eye at λ = S(λ) * R(λ)
```

So an object does not "have" a spectrum of its own in the sense of emitting one, and the light that leaves it changes with the illumination. A screen is simpler: its pixels emit light directly, and for now we can ignore the room.

Nothing in that description mentions color. The spectrum is a physical quantity that an instrument can measure, and "red" is not one of its properties. Color is what a visual system makes of a spectrum. To store color in a few numbers, we need to know what that system keeps and what it throws away, which brings up the question this deep dive really rests on: **how can a function with hundreds of degrees of freedom be represented by three numbers?**

### 2. Why three numbers are enough: cones, metamerism, and CIE XYZ

Three numbers are enough because of how the retina samples light, and that fact shapes everything else in this document.

#### Three kinds of sensor

Human color vision begins with three classes of cone cell, named for the part of the spectrum they favor: long (L), medium (M), and short (S). Rod cells serve vision at very low light levels, and color perception in ordinary lighting rests on the cones. [Measurements of pigments in cells from a human retina](https://pubmed.ncbi.nlm.nih.gov/7359434/) put the absorbance peaks of the three cone types near 563, 534, and 420 nm, with rods near 498 nm.

![Three normalized curves over wavelength, labeled S, M, and L, drawn over the color of the spectrum. S peaks near 440 nanometers. M and L overlap heavily, peaking near 545 and 570 nanometers.](assets/ref-cone-response.svg)

*Image: BenRG, public domain, via Wikimedia Commons.*

These response curves are measured at the front of the eye, so the lens and other pigments have already absorbed some short-wave light, which is why the peaks sit higher here than in the pigment measurements. The curves overlap heavily, especially M and L. Modern estimates of them, the [Stockman and Sharpe cone fundamentals](http://www.cvrl.org/cones.htm), are published as tables, with [their derivation](https://doi.org/10.1016/S0042-6989(00)00021-3) in the literature.

A single cone reports only one thing: how many photons it absorbed. Once absorbed, a photon's wavelength is forgotten. A dim light at the cone's peak and a brighter light off its peak can produce the identical signal. Rushton called this the [principle of univariance](https://doi.org/10.1113/jphysiol.1972.sp009719), and it means that one cone class cannot tell wavelength from intensity. Vision compares the classes to recover something about wavelength.

Each cone's response is therefore a single weighted sum across the spectrum:

```
L = sum over λ of  S(λ) * l(λ)
M = sum over λ of  S(λ) * m(λ)
S = sum over λ of  S(λ) * s(λ)
```

Here is the reduction. Whatever the spectrum, the retina's first step keeps three numbers, and everything else is gone before the signal has left the eye.

#### Different light, same color

If the visual system keeps only three numbers, then any two spectra that give the same three numbers must look the same. Such a pair is a **metamer**, and metamers are not rare curiosities. Every color screen depends on them:

![A chart of relative power from 380 to 780 nanometers. The D65 daylight curve is broad and near 100, while the other spectrum is three narrow peaks near 465, 535, and 615 nanometers, each over 230. Below the chart, both have X 0.9504, Y 1.0000, and Z 1.0886.](assets/fig-13-metamer.svg)

The three peaks in this figure are illustrative Gaussian curves, scaled by solving a 3 by 3 linear system so that their tristimulus values equal those of D65. Their spectra barely resemble daylight, and between the peaks they emit almost nothing, yet for the standard observer the two lights are the same color. There is no need to reproduce a spectrum to reproduce a color, which is why a display needs only three emitters rather than hundreds.

Metamerism has limits, and they are worth keeping in mind:

- It depends on the observer. People's cones differ a little, so a pair that matches for the standard observer may not match exactly for you.
- For surfaces, it depends on the light. Two paints that match under daylight have different R(λ), so under a filament the products with S(λ) differ and the match can fail. That is why a shirt and trousers can match in the shop and clash outdoors.

#### From a private response to a public standard

To use this in engineering we need the three numbers in a standard form, for one agreed observer. In the late 1920s, Wright and Guild measured this without knowing the cone curves at all. Their observers adjusted three lights to match a test color, wavelength by wavelength. The method works because matches obey simple linear rules, [Grassmann's laws](https://en.wikipedia.org/wiki/Grassmann%27s_laws_%28color_science%29): if A matches B, then A plus C matches B plus C. Light mixing in matches behaves like vector addition.

There was one awkward result. Some test lights could not be matched by any positive mixture of the three primaries. Observers had to add a primary to the test light instead, which amounts to a negative amount of that primary. We will meet the reason shortly.

The CIE [standardized this data in 1931](https://doi.org/10.1002/(SICI)1520-6378(199702)22:1%3C11::AID-COL4%3E3.0.CO;2-7), then applied a linear transform to get three new functions, x̄, ȳ, and z̄, chosen for practical properties of the time: none ever goes negative, and ȳ equals the standard curve of perceived brightness across wavelength. Weighted sums with them give the **CIE XYZ tristimulus values**:

```
X = sum over λ of  S(λ) * xbar(λ)
Y = sum over λ of  S(λ) * ybar(λ)
Z = sum over λ of  S(λ) * zbar(λ)
```

XYZ is not a set of cone responses, and X, Y, and Z are not lights that exist. But the transform is linear and the original data came from color matching, so two spectra with the same XYZ are a metamer for the standard observer. The [CVRL database](http://www.cvrl.org/cmfs.htm) publishes these tables, and the metamer above was computed from them. XYZ is the common ground that every color space below is defined against, and **Y** has a physical meaning we will use repeatedly: it is **luminance**, a measure of how intense a light appears, weighted by wavelength.

#### Separating color from intensity

Doubling the power of a light doubles X, Y, and Z, but leaves what we would call its color nearly the same. To plot color without intensity, divide out the scale:

```
x = X / (X + Y + Z)
y = Y / (X + Y + Z)
```

These are **chromaticity coordinates**. Our running color, with XYZ of 0.021054, 0.022735, and 0.046030, has x = 0.2344 and y = 0.2531. Plotting every spectral wavelength this way traces a curve known as the spectral locus:

![The CIE 1931 xy chromaticity diagram, a horseshoe with wavelengths marked along its curved edge from 460 to 620 nanometers. Green fills the top, red the lower right, and blue the lower left, fading toward white in the middle.](assets/ref-cie-chromaticity.svg)

*Image: BenRG, public domain, via Wikimedia Commons. White background added. Colors are approximate, because no screen can show this entire region.*

Two properties of this diagram matter for the rest of the document:

- **Mixing draws straight lines.** Because mixing is linear in XYZ, the chromaticity of a mix of two lights lies on the straight line between them. A mix of three lies inside their triangle.
- **The locus curves outward.** All real lights lie on or inside the horseshoe, and no triangle with corners inside it can cover it all.

That second point explains the negative amounts Wright and Guild needed: a pure spectral color lying outside the triangle of three primaries cannot be reached by adding positive light. We have three numbers for any color we can see, but we can't build a device that reaches them all. **Given that limit, how does a screen actually make color?**

### 3. How a screen makes color: subpixels, primaries, and white point

A screen is the metamer trick built in hardware. It never reproduces the spectrum of the thing being shown; it emits three kinds of light, in amounts that give your cones the right three responses.

Look at a screen through a magnifying lens and each pixel resolves into separate emitters, called subpixels:

![Four close-up photographs of display surfaces. Two cathode ray tubes show red, green, and blue phosphors as bars and dots. Two liquid crystal displays show diamond-shaped and vertical striped subpixels in red, green, and blue.](assets/ref-pixel-geometry.jpg)

*Image: Pengo, CC BY-SA 3.0, via Wikimedia Commons.*

At normal viewing distance the subpixels are too small to see individually, and [the retina spatially integrates their light](https://learnvisualcomputing.github.io/display-optics.html). The mix that results is **additive**: red and green light together give yellow, and all three at full power give white. This is not the subtractive mixing of paint, where each pigment removes light.

The source of that light differs by technology, as [Zhu's *Foundations of Visual Computing*](https://learnvisualcomputing.github.io/display-optics.html) describes:

- An **LCD** has a white backlight. Each subpixel has a color filter, and a liquid crystal cell between two polarizers controls how much light gets through.
- An **OLED** or **MicroLED** display has subpixels that emit light directly, so a subpixel set to zero is simply off.

Both reduce to the same model for our purposes. Each subpixel adds a fixed spectrum, scaled by how hard it is driven.

#### Primaries and their triangle

The chromaticity of one subpixel at full power is a **primary**. Three primaries are three points on the chromaticity diagram, and since mixes stay inside the triangle they form, that triangle is everything the display can show at its given brightness. This is its **gamut**.

Two standard sets of primaries matter for the web, from the [predefined color spaces of CSS Color 4](https://www.w3.org/TR/css-color-4/#predefined):

| Primary | sRGB x, y | Display P3 x, y |
|---|---|---|
| Red | 0.640, 0.330 | 0.680, 0.320 |
| Green | 0.300, 0.600 | 0.265, 0.690 |
| Blue | 0.150, 0.060 | 0.150, 0.060 |
| White | 0.3127, 0.3290 (D65) | 0.3127, 0.3290 (D65) |

#### What "white" means

Three primaries fix the corners of the triangle, but not how much of each makes "full on" look neutral. For that a display needs a **white point**, the chromaticity reached when all three channels are at maximum. Both spaces above use D65, the daylight curve from section 1, whose chromaticity is x = 0.3127, y = 0.3290.

There is nothing physically special about this choice. As we saw, D65 and a filament both look white to an adapted eye, so white is a convention, and the two spaces simply agree on one that suits displays viewed in ordinary conditions.

We now have the physical picture: three emitters, each driven at some level, whose sum makes a color. The numbers `26, 43, 60` must say how hard to drive them. The tempting assumption is that 60 means about 24% of full power. It does not, and **the reason encoded values are not amounts of light is the next layer**.

## 1. Vocabulary

These terms are often used interchangeably, but they answer different questions.

| Term | Question it answers | Examples |
|---|---|---|
| Color space | Which physical color do these numbers mean? Fixed by primaries, a white point, and a transfer function. | sRGB, Display P3 |
| Color model | How is a color in that space parameterized? | RGB, HSL, HSB, CMYK |
| Encoding | How are the numbers written down? | `#1A2B3C`, 0 to 1 floats, 0 to 255 integers |

HSL and HSB are not color spaces. They rearrange the RGB numbers of some space, so H 210, S 40, L 17
means different colors in sRGB and in Display P3. Moving between models is exact arithmetic. Moving
between spaces changes the numbers to preserve the physical color.
[culori's list of color spaces](https://culorijs.org/color-spaces/) is a quick survey of both kinds, and
[CSS Color 4](https://www.w3.org/TR/css-color-4/) defines most of them precisely.

## 2. Numeric representations

| Representation | Values | Resolution | Typical use |
|---|---|---|---|
| 8-bit integer | 0 to 255 | 1/255, about 0.39% of the range | HEX, RGB |
| 32-bit float | 24-bit significand | Near 1, a spacing of 2^-24 and an error of at most 2^-25, about 3e-8 | GPUs, and Figma's storage |
| 64-bit float | 53-bit significand | Near 1, an error of at most 2^-53, about 1e-16 | JavaScript arithmetic |
| Decimal text | Fixed digits | Set by the format | Displayed and copied values |

A 32-bit float distinguishes every 8-bit value, and its error on the 0 to 255 scale is at most about
8e-6, so `round(x * 255)` recovers the byte. Most decimals, and most values k/255, have no exact
binary form in either width. In JavaScript,
[`Math.fround`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/fround)
gives the 32-bit form of a number, and
[Goldberg's paper](https://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html) is the standard
account of this arithmetic.

## 3. Channels and alpha

Alpha is coverage, not a color channel. It says how much of the color is laid over what is behind
it, so a translucent color has no single appearance:

```
C_out = alpha * C_source + (1 - alpha) * C_backdrop
```

![The color #1A2B3C at 90, 50, and 10 percent paint opacity over white, gray, and black. The fill's value stays the same while the seen color changes.](assets/fig-02-alpha.png)

With **straight** alpha, channels hold the color independently of alpha. **Premultiplied** alpha stores
`alpha * C` instead, which suits compositing but loses channel precision at low alpha.

Contrast formulas take opaque colors. For a translucent color the result depends on the backdrop, so a
ratio computed from the channels alone describes the color at full opacity.

## 4. Discretization and rounding

### Quantization

Writing a continuous value in finite digits is quantization. For a channel x from 0 to 1:

```
k = round(255 * x)        quantize, x to an 8-bit value
x' = k / 255              reconstruct
|x - x'| <= 1 / 510       worst-case error
```

![A black to white ramp at 4, 16, 64, and 256 levels. Bands are obvious at 4 and 16 levels, faint at 64, and absent at 256.](assets/fig-01-quantization.png)

Quantization is lossy, so each number should be computed from the exact value rather than from another
rounded number.

### Display precision

The integer HSL grid has 360 x 101 x 101 = 3,672,360 points, against 16,777,216 8-bit RGB colors, so
integers cannot round-trip as a matter of counting. Measured over every 8-bit color, converted to the
display precision and back:

| Decimals | HSL, colors changed | Largest channel error | HSB, colors changed | Largest channel error |
|---|---|---|---|---|
| 0 | 15,001,678 | 5 | 14,732,503 | 3 |
| 1 | 0 | 0 | 0 | 0 |
| 2 | 0 | 0 | 0 | 0 |

Integer HSL and HSB are therefore for reading, while HEX and RGB integers hold an 8-bit color exactly.
Color pickers commonly show integers, as Figma's does.

![Three colors converted back from integer HSL and from HSL to 9 significant digits. Integers change each one, and 9 digits return the same color.](assets/fig-06-precision-round-trip.png)

### Rounding modes

| Mode | 2.5 | 3.5 | -2.5 | Suits |
|---|---|---|---|---|
| Half up | 3 | 4 | -2 | Displaying a value |
| Half even | 2 | 4 | -2 | Summing many rounded values, where it avoids bias |
| Floor | 2 | 3 | -3 | Reporting against a threshold, so that a failure never rounds into a pass |

Half up is the rule in [CSS Color 4](https://www.w3.org/TR/css-color-4/#the-hsl-notation), as "rounded
towards +∞", and Figma's color picker rounds a hue of exactly 122.5 up to 123.
[`toFixed`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/toFixed)
follows neither mode, because it rounds the stored binary value, so a number that looks like a half can go
either way.

For non-negative values, half up is `floor(x + 0.5)` in exact arithmetic. In floating-point arithmetic
that expression is not enough, as the case study shows.

**Contrast takes no tolerance.** [WCAG's guidance](https://www.w3.org/WAI/WCAG22/Understanding/contrast-minimum.html)
is that computed values should not be rounded, and that 4.499:1 does not meet 4.5:1. Flooring a ratio
already errs low, and a tolerance could lift 4.49999 to 4.5 and turn a failure into a pass. A ratio that
floating-point error puts just under a two-decimal boundary reads one hundredth low, which is the safe
direction.

## 5. Linearization

RGB values in sRGB are encoded nonlinearly, to spend precision where vision is most sensitive. Light
adds and mixes in linear values, so luminance and any conversion between spaces need decoding first.

```
decode:  c_lin = c / 12.92                       if c <= 0.04045
         c_lin = ((c + 0.055) / 1.055) ^ 2.4     otherwise

encode:  c = 12.92 * c_lin                       if c_lin <= 0.0031308
         c = 1.055 * c_lin ^ (1 / 2.4) - 0.055   otherwise
```

The straight segment near black avoids an infinite slope. The curve is close to an overall gamma of
2.2, but is not one. It is also why mid-gray is not half the light: an encoded 0.5 decodes to 0.214.

![Two ramps, equal steps in encoded value and equal steps in linear light, followed by three pairs of colors mixed half and half, as encoded values and as light.](assets/fig-03-linearization.png)

[WCAG](https://www.w3.org/TR/WCAG22/#dfn-relative-luminance) gave a threshold of 0.03928 until May 2021,
and now gives 0.04045 ([issue 360](https://github.com/w3c/wcag/issues/360)). They give identical results
for every 8-bit value, because no k/255 falls between them. For other values they differ by at most 7.6e-7.

[Display P3](https://www.w3.org/TR/css-color-4/#predefined-display-p3) uses this same transfer function,
so the same decoding applies.

## 6. Color spaces and gamut

A linear RGB space maps to the device-independent CIE XYZ by a matrix fixed by its primaries and white
point. Both of these use the D65 white point:

```
sRGB to XYZ                              Display P3 to XYZ
| 0.4124564  0.3575761  0.1804375 |      | 0.4865709  0.2656677  0.1982173 |
| 0.2126729  0.7151522  0.0721750 |      | 0.2289746  0.6917385  0.0792869 |
| 0.0193339  0.1191920  0.9503041 |      | 0.0000000  0.0451134  1.0439444 |
```

The middle row gives Y, the luminance. CSS Color 4 publishes
[sample code](https://www.w3.org/TR/css-color-4/#color-conversion-code) for these conversions. To convert
between spaces:

```
encoded A -> decode -> linear A -> matrix A -> XYZ -> inverse matrix B -> linear B -> encode -> encoded B
```

Display P3 has a wider gamut than sRGB, so converting a saturated P3 color into sRGB gives channels
outside 0 to 1, which must be clipped or gamut-mapped, and either one changes the color.

The space decides what the numbers mean, so `#FF0000` is a different red in sRGB and in Display P3. It
bears on each kind of value differently:

| Value | Depends on the space? | Why |
|---|---|---|
| HEX, RGB | No | They are the numbers themselves |
| HSL, HSB | No | They rearrange the numbers |
| Luminance, contrast | Yes | They measure light, so they need the primaries |
| White and black | No | Y is 1 and 0 in both spaces |

## 7. RGB, HSL, and HSB

HSL and HSB (also called HSV) describe the RGB cube by hue and two other axes. CSS Color 4
[defines HSL](https://www.w3.org/TR/css-color-4/#the-hsl-notation), with
[sample code](https://www.w3.org/TR/css-color-4/#hsl-to-rgb), and CSS has no HSB. Both start from these,
with channels from 0 to 1:

```
M = max(R, G, B)     m = min(R, G, B)     C = M - m
```

![Hue 210 laid out by saturation and lightness, and by saturation and brightness, with a hue strip from 0 to 360.](assets/fig-04-hsl-hsb.png)

### Hue

The position around the cube's edge, in degrees, and shared by both models:

```
H = undefined                            if C = 0
H = 60 * (((G - B) / C) mod 6)           if M = R
H = 60 * ((B - R) / C + 2)               if M = G
H = 60 * ((R - G) / C + 4)               if M = B
```

Where C = 0 the color is achromatic and has no hue, so a displayed value needs a convention, usually 0.
Since 360 and 0 are the same hue, a hue of 359.6 rounds to 360 and should display as 0.

### Saturation and lightness or brightness

```
HSL:  L = (M + m) / 2
      S = 0                        if C = 0
      S = C / (1 - |2L - 1|)       otherwise

HSB:  B = M
      S = 0                        if M = 0
      S = C / M                    otherwise
```

The two saturations are different quantities. Pure red is S = 100 in both, while a pale pink can be
S = 100 in HSL and S = 10 in HSB.

### Back to RGB

```
HSL:  a = S * min(L, 1 - L)
      f(n) = L - a * max(-1, min(k - 3, 9 - k, 1))     where k = (n + H / 30) mod 12
      (R, G, B) = (f(0), f(8), f(4))

HSB:  f(n) = B - B * S * max(0, min(k, 4 - k, 1))      where k = (n + H / 60) mod 6
      (R, G, B) = (f(5), f(3), f(1))
```

### Directly between HSL and HSB

```
B = L + S_l * min(L, 1 - L)
S_b = 0 if B = 0, else 2 * (1 - L / B)

L = B * (1 - S_b / 2)
S_l = 0 if L is 0 or 1, else (B - L) / min(L, 1 - L)
```

### Worked example

`#1A2B3C` at 90% opacity:

| Step | Exact | Integer | 9 significant digits |
|---|---|---|---|
| Channels | 26/255, 43/255, 60/255 | `26, 43, 60` | `0.101960784, 0.168627451, 0.235294118` |
| M, m, C | 60/255, 26/255, 34/255 | | |
| H (M = B) | 60 * ((26 - 43) / 34 + 4) = 210 | `210` | `210` |
| L | 86/510 = 16.8627...% | `17` | `16.8627451` |
| S, HSL | 34/86 = 39.5348...% | `40` | `39.5348837` |
| S, HSB | 34/60 = 56.6666...% | `57` | `56.6666667` |
| B | 60/255 = 23.5294...% | `24` | `23.5294118` |
| Alpha byte | 0.9 * 255 = 229.5, half up | `E6` | |

### HSL lightness is not perceived lightness

HSL and HSB are geometric rearrangements with no model of vision. Yellow and blue both have L = 50%,
but their luminances are 0.93 and 0.07. Decisions about legibility therefore use luminance rather
than L. CSS Color 4 lists this among [HSL's disadvantages](https://www.w3.org/TR/css-color-4/#disadvantage-hsl),
and [Ottosson's Oklab post](https://bottosson.github.io/posts/oklab/) shows the same unevenness in HSV.

![Six fully saturated colors at L = 50, each above the gray with the same luminance, with ratios against white and black.](assets/fig-05-lightness-luminance.png)

## 8. Luminance and contrast

```
Y = w_r * R_lin + w_g * G_lin + w_b * B_lin

ratio = (Y_lighter + 0.05) / (Y_darker + 0.05)
```

The weights are the middle row of the space's matrix, reflecting how bright each primary is:

| Space | w_r | w_g | w_b |
|---|---|---|---|
| sRGB | 0.2126 | 0.7152 | 0.0722 |
| Display P3 | 0.2289746 | 0.6917385 | 0.0792869 |

[WCAG defines relative luminance](https://www.w3.org/TR/WCAG22/#dfn-relative-luminance) for "any point
in a colorspace" and gives the formula for sRGB as a note. A comment in
[issue 360](https://github.com/w3c/wcag/issues/360) says that a space such as Display P3 should use its own
chromaticities and white point. Checkers such as [WebAIM's](https://webaim.org/resources/contrastchecker/)
and [Siege Media's](https://www.siegemedia.com/contrast-ratio) take hex values, which CSS defines as sRGB,
so their figures describe the sRGB color with those numbers.

The [contrast ratio](https://www.w3.org/TR/WCAG22/#dfn-contrast-ratio) adds 0.05 to each side, which models
ambient flare and keeps the ratio finite at black. It runs from 1 to 21, since
(1 + 0.05) / (0 + 0.05) = 21. Thresholds are 4.5 (AA), 3 (AA large text), and 7 (AAA), compared against the
unrounded ratio. In sRGB, `#1A2B3C` has Y = 0.02274, which gives 14.43 against white and 1.45 against black,
each floored to 2 decimals.

![White text on seven grays from 1.51 to 21 to 1, with large text, body text, and labels, and each threshold's result.](assets/fig-08-contrast-ranges.png)

**The best any color can do.** A color with luminance Y reaches its highest ratio against white or
against black. The lower of those two is smallest where they meet:

```
1.05 / (Y + 0.05) = (Y + 0.05) / 0.05    gives    Y + 0.05 = sqrt(1.05 * 0.05)    so    Y = 0.1791
ratio there = sqrt(21) = 4.583
```

So every color reaches at least 4.583 against white or black, and no higher guarantee is possible.

**Choosing between two text colors.** Compare the two ratios, because a rule based on L gets colors near
the crossing wrong. With `#F2F2F2` and `#181818` rather than pure white and black, the crossing point
moves: they tie at Y = 0.1855, near `#777777`, at 3.98 : 1.

![Twelve grays with two text colors and their ratios. The higher ratio wins, and a rule based on L below 50 picks the worse color at #7C7C7C.](assets/fig-09-text-color-choice.png)

## 9. Finding a tone at a target contrast

A tone of a color's own hue, at the least contrast that stays readable, can be found by holding H and S
fixed and searching L for the smallest change whose ratio against the color reaches a target T.

![Seven colors, each with same-hue text at a ratio of 3 and with the higher-contrast of two text colors.](assets/fig-10-same-hue-text.png)

**A search can stop at the first passing value.** With H and S fixed, every channel from the
formula in [Back to RGB](#back-to-rgb) is non-decreasing in L, and luminance is strictly increasing,
so the ratio grows steadily in each direction away from the color. Each direction therefore has one
boundary, which bisection finds. All of this holds in any RGB space, since the weights are positive.

**When a direction reaches the target.** L = 0 is black and L = 1 is white, for every H and S, so the
extremes of the search are exactly the extremes of section 8. For a color of luminance Y:

```
darker reaches T    if (Y + 0.05) / 0.05 >= T
lighter reaches T   if 1.05 / (Y + 0.05) >= T
```

At T = 3, darker reaches for Y >= 0.1, and lighter reaches for Y <= 0.3. These ranges overlap, so one
direction always reaches and both do for Y between 0.1 and 0.3. Neither direction can reach only if
T is above sqrt(21), about 4.583. At T = 5, `#E53935` is such a color.

The rule can land near an extreme. `#0B6E4F` has a luminance just above 0.1, so its smallest change is
down to L = 4.59, which is almost black, though lightening would have kept more of the hue.

## 10. Writing values for reuse

The same color can be written for different consumers, shown here for `#1A2B3C` at 90%:

| Form | HEX | RGBA | HSL |
|---|---|---|---|
| Rounded, for reading | `#1A2B3CE6` | `26, 43, 60, 0.9` | `210, 40, 17, 90` |
| CSS | `#1A2B3CE6` | `rgba(26, 43, 60, 0.9)` | `hsla(210, 40%, 17%, 0.9)` |
| Full precision | `#1A2B3CE6` | `0.101960784, 0.168627451, 0.235294118, 0.9` | `210, 39.5348837, 16.8627451, 90` |

The CSS form of HSL above is rounded, so CSS that has to reproduce a color should use HEX or `rgba()`.

Nine significant digits are [enough to recover any 32-bit value](https://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html),
and stop short of the digits that are only 64-bit noise. Consumers with 0 to 1 channels, such as SwiftUI,
Unity, and shaders, take this form. Every 8-bit value comes back as the same byte. 249 of the 256 come back
as the same 32-bit number as `float32(k / 255)`, and the other 7 are one 32-bit step away.

| Reused from | Acceptable? | Why |
|---|---|---|
| HEX, RGBA, for an 8-bit color | Yes | Exactly the color |
| HEX, RGBA, for a color that is not 8-bit | With care | Rounded to the nearest 8-bit color |
| Integer HSL, HSB | No | Up to 5 units per channel off when converted back |

A value also leaves its color space behind. Display P3 numbers mean a different color when read as sRGB, and
CSS `#RRGGBB`, `rgba()`, and `hsl()` all [resolve to sRGB](https://www.w3.org/TR/css-color-4/#the-hsl-notation).
CSS's way to say Display P3 is
[`color(display-p3 r g b / a)`](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/color), widely
available in browsers since May 2023. It keeps the color on wide-gamut screens, while the same numbers in sRGB
syntax can look less saturated. HEX and HSL have no Display P3 form in CSS.

## 11. Other models

For Oklab, [Ottosson's post](https://bottosson.github.io/posts/oklab/) gives the derivation and the matrices.
On building palettes programmatically, see
[Coloring With Code](https://tympanus.net/codrops/2021/12/07/coloring-with-code-a-programmatic-approach-to-design/)
and [HSB interpolation for Figma color variables](https://uxdesign.cc/simplified-figma-color-variables-based-on-hsb-interpolation-c3e404585176).

| Model | Built for | Conversion path |
|---|---|---|
| CIELAB, LCH | Approximate perceptual uniformity | Linear RGB, XYZ, adapt to D50, then a cube-root function of X/Xn, Y/Yn, Z/Zn; LCH is polar Lab |
| OKLab, OKLCH | Better hue linearity than Lab | Linear RGB, a matrix to cone-like responses, a cube root, a second matrix; OKLCH is polar OKLab |
| HCT | Hue and chroma from CAM16, tone from L* | Requires a color appearance model, and has an iterative inverse |
| CMYK | Ink on a specific press | Device-dependent, so an accurate conversion needs an ICC profile. The formula `K = 1 - M` is only a rough approximation |

## 12. References

Light, vision, and displays:

- [CIE standard illuminant D65](https://cie.co.at/datatable/cie-standard-illuminant-d65), International Commission on Illumination (CIE), [DOI 10.25039/CIE.DS.hjfjmt59](https://doi.org/10.25039/CIE.DS.hjfjmt59). The tabulated spectrum used for the D65 curves.
- [Colour & Vision Research Laboratory database](http://www.cvrl.org/), UCL. Tables of the [CIE 1931 color-matching functions](http://www.cvrl.org/cmfs.htm) and [cone fundamentals](http://www.cvrl.org/cones.htm).
- [Visual pigments of rods and cones in a human retina](https://pubmed.ncbi.nlm.nih.gov/7359434/), Bowmaker and Dartnall, *Journal of Physiology*, 1980. Peak absorbances of the three cone types and rods.
- [Pigments and signals in colour vision](https://doi.org/10.1113/jphysiol.1972.sp009719), Rushton, *Journal of Physiology*, 1972. The principle of univariance.
- [The spectral sensitivities of the middle- and long-wavelength-sensitive cones](https://doi.org/10.1016/S0042-6989(00)00021-3), Stockman and Sharpe, *Vision Research*, 2000. The derivation of modern cone fundamentals.
- [How the CIE 1931 color-matching functions were derived from Wright-Guild data](https://doi.org/10.1002/(SICI)1520-6378(199702)22:1%3C11::AID-COL4%3E3.0.CO;2-7), Fairman, Brill, and Hemmendinger, *Color Research and Application*, 1997.
- [Grassmann's laws](https://en.wikipedia.org/wiki/Grassmann%27s_laws_%28color_science%29), Wikipedia. A starting point on the linearity of color matching.
- [A perceptual color space for image processing](https://bottosson.github.io/posts/oklab/), Björn Ottosson. The derivation of Oklab by its creator, with its math and a comparison with HSV.
- [RGB/XYZ Matrices](http://www.brucelindbloom.com/index.html?Eqn_RGB_XYZ_Matrix.html), Bruce Lindbloom. The derivation of RGB to XYZ matrices, with tables.

- [Optical Mechanisms](https://learnvisualcomputing.github.io/display-optics.html), Yuhao Zhu, *Foundations of Visual Computing*. How LCD and emissive displays make light.

Standards:

- [CSS Color Module Level 4](https://www.w3.org/TR/css-color/), W3C. Definitions of HSL and Display P3,
  rounding, and [sample conversion code](https://www.w3.org/TR/css-color-4/#color-conversion-code).
- [Web Content Accessibility Guidelines 2.2](https://www.w3.org/TR/WCAG22/), W3C. Definitions of
  [relative luminance](https://www.w3.org/TR/WCAG22/#dfn-relative-luminance) and
  [contrast ratio](https://www.w3.org/TR/WCAG22/#dfn-contrast-ratio).
- [Understanding Success Criterion 1.4.3: Contrast (Minimum)](https://www.w3.org/WAI/WCAG22/Understanding/contrast-minimum.html),
  W3C. The thresholds, and why ratios are not rounded.
- [WCAG issue 360](https://github.com/w3c/wcag/issues/360), W3C. The sRGB threshold, and luminance in
  other color spaces.

Color science and libraries:

- [A perceptual color space for image processing](https://bottosson.github.io/posts/oklab/), Björn
  Ottosson. The derivation of Oklab by its creator, with its math and a comparison with HSV.
- [Color spaces](https://culorijs.org/color-spaces/) and [API](https://culorijs.org/api/), culori.
- [`color()`](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/color), MDN. Naming a color space
  in CSS, and browser availability.

Numerics:

- [What Every Computer Scientist Should Know About Floating-Point Arithmetic](https://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html),
  David Goldberg. Includes why 9 decimal digits recover a 32-bit number.
- [`Math.fround`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/fround)
  and [`Number.prototype.toFixed`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/toFixed),
  MDN.

Tools and practice:

- [Contrast Checker](https://webaim.org/resources/contrastchecker/), WebAIM.
- [Contrast Ratio](https://www.siegemedia.com/contrast-ratio), Siege Media.
- [Coloring With Code: A Programmatic Approach to Design](https://tympanus.net/codrops/2021/12/07/coloring-with-code-a-programmatic-approach-to-design/),
  Codrops.
- [Simplified Figma color variables based on HSB interpolation](https://uxdesign.cc/simplified-figma-color-variables-based-on-hsb-interpolation-c3e404585176),
  UX Collective.

## Image credits

These images keep their own licenses, as linked, rather than the repository's.

| File | Source | Author | License | Change |
|---|---|---|---|---|
| `ref-visible-spectrum.svg` | [Linear visible spectrum](https://commons.wikimedia.org/wiki/File:Linear_visible_spectrum.svg) | Gringer | Public domain | White background added |
| `ref-cone-response.svg` | [Cone fundamentals with sRGB spectrum](https://commons.wikimedia.org/wiki/File:Cone-fundamentals-with-srgb-spectrum.svg) | BenRG | Public domain | None |
| `ref-cie-chromaticity.svg` | [CIE1931xy blank](https://commons.wikimedia.org/wiki/File:CIE1931xy_blank.svg) | BenRG | Public domain | White background added |
| `ref-pixel-geometry.jpg` | [Pixel geometry 01 Pengo](https://commons.wikimedia.org/wiki/File:Pixel_geometry_01_Pengo.jpg) | Pengo | [CC BY-SA 3.0](https://creativecommons.org/licenses/by-sa/3.0/) | None |
