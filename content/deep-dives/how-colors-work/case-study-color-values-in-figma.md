# Case Study: Exact Color Values in Figma

Figma stores color channels and opacity as 32-bit floats, so a plugin that wants to show the exact value someone
entered reads back numbers such as 0.8999999761581421. This case study continues from
[How Colors Work](./how-colors-work-from-light-to-implementation.md), which covers the general ideas.

## How Figma holds color

**Figma:** a [solid fill](https://developers.figma.com/docs/plugins/api/Paint/) holds channels `{r, g, b}`
from 0 to 1, with alpha held separately as the paint's `opacity`. All are stored in 32 bits, so 0.9 reads
back as 0.8999999761581421. Writing a channel outside 0 to 1 is an error, and channels are not limited to
multiples of 1/255 ([measurements](#figma-storage)).

**Figma:** paint opacity belongs to the fill. Layer opacity belongs to a node and affects everything
beneath it in the tree, so it changes what a viewer sees without changing the fill.

## Rounding stored values

![Six cases with the exact value, the common mistake, and the correct result.](assets/fig-07-rounding.png)

### Exact halves are common

Exact halves are ordinary results of these formulas. Across all 16,777,216 8-bit colors there are
250,752 hues, 92,160 HSL saturations, and 135,936 HSB saturations that lie exactly on a half at integer
precision. For example, `rgb(0, 24, 1)` has a hue of exactly 122.5, while 64-bit arithmetic produces
122.49999999999999, and plain half up gives 122 rather than 123.

Alpha has its own halves. `a * 255` lies on a half only at a = 0.1, 0.3, 0.5, 0.7, and 0.9.

### Noise and how it grows

A stored 32-bit value carries an error of up to about 3e-8. Formulas that divide by a small difference
amplify it, as hue divides by `max - min` and HSL saturation by `1 - |2L - 1|`. Measured over every
8-bit color, the error in a computed H, S, L, or B reaches:

| Input | Largest error in a computed value |
|---|---|
| 32-bit channels | 1.14e-4 |
| 64-bit channels (k/255 exactly) | 7.1e-13 |

A method that absorbs error has to treat a number as a half when it is within the noise, but not
when it is merely close. At integer precision, the closest any 8-bit color's value comes to a half
without being one is 0.00198. A tolerance of 3 or 4 decimals on stored numbers does give correct integers
for every 8-bit color, but 5 decimals and 2 decimals both fail, every tolerance tried fails at 2-decimal
precision, and full-precision output still prints the noise ([measurements](#rounding-methods-h-s-l-and-b)).

### Recovering the stored value

Storage is a many-to-one function, but the values people enter are a small set. Inverting it for that
set removes the noise instead of tolerating it:

```
k = round(255 * stored)
d = round(10000 * stored) / 10000
if float32(k / 255) == stored                      use k / 255
else if stored is alpha, and float32(d) == stored  use d
else                                               use stored
```

Then compute in 64-bit, and apply half up with a tolerance of 1e-9, wide enough for the 7.1e-13 that
remains, and far narrower than the 0.00198 gap. Full-precision output then gives the entered value, `0.9`,
rather than the stored `0.899999976`.

For a color that is not 8-bit, nothing is recovered and the stored number is used. Its noise then
changes a displayed digit only at a half, which such a color has no particular reason to land on.

## Color profiles

![Four colors with luminance and ratio against white in each space. #B400FF passes AA against white in sRGB and fails it in Display P3.](assets/fig-11-display-p3.png)

**Figma:** a file names [its color profile](https://help.figma.com/hc/en-us/articles/360039825114): sRGB,
Display P3, or Legacy for files created before color management. Figma documents that a Legacy file renders
in the user's preferred profile, or in sRGB where none is set. Converting a file's profile changes the stored
values and keeps the appearance, while assigning one keeps the values and changes the appearance. A plugin
can [read the profile](https://developers.figma.com/docs/plugins/api/DocumentNode/) but not change it.

## culori

**culori:** `lrgb` is its linear sRGB mode, and it uses 0.04045. Because the transfer function is
shared, its output is also the linear form of Display P3 numbers.

**culori:** `hsl` and `hsv` modes, with values from 0 to 1 for S, L, and V. For an achromatic color
it omits `h` rather than setting it, and sets `s` to 0.

**culori:** `wcagLuminance` and `wcagContrast` read through the `lrgb` mode and ignore alpha. A color's
mode says which space its channels are in, so tagging them `p3` gives Display P3 luminance. That route
passes through XYZ and linear sRGB, which has two effects:

- Y stays within 4.9e-5 of the published weights above. Of 140,608 sampled colors, 5 get a different AA
  result against white than those weights give.
- White comes out at Y = 0.9999999999999999, so white on black is 20.999999999999996, which floors to 20.99.
  A gray is the same color in both spaces, so tagging grays `rgb` gives exactly 21.

**culori:** supports Lab, LCH, OKLab, and OKLCH, each by registering its mode. Its
[list of color spaces](https://culorijs.org/color-spaces/) has no HCT or CMYK.

**Registering modes in culori.** culori holds a table of the color modes it knows, and a mode's
definition contains its conversions. Registering, by calling `useMode(modeHsl)`, adds an entry to
that table. The main `culori` entry registers every mode when imported, so a bundler has to keep them
all. The `culori/fn` entry registers none, so only the modes actually imported reach the bundle, and
calling a conversion whose mode is missing throws. `wcagLuminance` and `wcagContrast` need `lrgb` to be
registered, which is easy to miss because they depend on it internally, and a color tagged `p3` needs `p3`
as well.

## 12. Measurements

Taken on 2026-11-03, with Figma's plugin API. The sweeps cover all 16,777,216 8-bit colors, against
exact integer arithmetic on the channels. The rounding sweeps use five outputs per color (H, HSL S, L,
HSB S, B), and they use `Math.fround` of k/255 as the model of Figma's storage, which the first table
confirms. The integer and 1-decimal columns used culori's conversions, and the 2-decimal column the
formulas in section 7.

### Figma storage

Written to a temporary rectangle and read back.

| Written | Read | Result |
|---|---|---|
| Paint opacity 0.9 | 0.8999999761581421 | The 32-bit form |
| Paint opacity i/1000, all 1001 values | | All the 32-bit form |
| Paint opacity k/255, all 256 values | | All the 32-bit form |
| Layer opacity 0.9 | 0.8999999761581421 | The 32-bit form |
| k/255 in each single channel, all 256 values | | All the 32-bit form |
| Gray, k/255 in all channels, all 256 values | | All the 32-bit form, except k = 234 |
| Gray, 234/255 in all channels | 0.9188234806060791, about 234.3/255 | Repeatable in two files; cause unknown |
| `figma.util.solidPaint("#1A2B3CE6")` | Opacity 0.9019607901573181 | Alpha from HEX is k/255, not 0.9 |
| Channel 1.0000001, 1.5, or -0.0001 | | Validation error |
| `#C04040` exported as PNG from an sRGB file, tagged Display P3 | Pixels 192, 64, 64 | The same numbers with a different tag, so no conversion |

The gray at 234 still rounds to `EA`, and its integer HSL and HSB match those of 234/255. Its full-precision
channel is 0.918823481 rather than 0.917647059. Separately, 5 of the 21 channels in one file's color variables
were not multiples of 1/255, such as about 243.48/255.

### Rounding methods, H, S, L, and B

Count of wrong numbers among 83,886,080 per precision. The tolerance means rounding to that many
decimal places, at the output's scale, before applying half up.

| Input | Tolerance | Integer | 1 decimal | 2 decimals |
|---|---|---|---|---|
| 64-bit | None | 72,136 | Not run | 36,608 |
| 32-bit | None | 195,957 | Not run | 108,177 |
| 32-bit | 2 decimals | 123,150 | 125,406 | 129,129 |
| 32-bit | 3 decimals | 0 | 0 | 1,050 |
| 32-bit | 4 decimals | 0 | 1,740 | 59,523 |
| 32-bit | 5 decimals | 2,964 | 82,683 | Not run |
| Recovered | 9 decimals | 0 | 0 | 0 |

### Alpha

Entered as i/10000, all 10,001 values, stored in 32 bits, against exact integer arithmetic.

| Output | No handling | Recovered |
|---|---|---|
| Byte (`a * 255`) | 2 wrong (0.7 and 0.9) | 0 |
| Percent, integer | 48 wrong | 0 |

All 256 k/255 entries gave the right byte under each method.

### Display P3

Computed from the formulas in sections 5 and 8. Of colors sampled at every fifth channel value, 1,968 of
140,608 change their AA result against white between sRGB and Display P3 weights.

Measured in a Figma file changed to Display P3 by assigning, with an instance pasted beforehand from an sRGB
file:

| Checked | Result |
|---|---|
| Document color profile | Reads `DISPLAY_P3` |
| A fill on the instance | Keeps its numbers: 42, 140, 252 |
| Remote variables on the instance | Keep their numbers, such as 255, 96, 99 |
| Storage of channels and opacity | As in the sRGB table above, including gray at 234 |
| Default PNG export of `#C04040` | Pixels 192, 64, 64, tagged Display P3 |
| PNG export of `#C04040`, chosen as sRGB | Pixels 192, 64, 64, tagged sRGB. A conversion would give 208, 49, 58, so this changes the tag only |

### Tone at a target contrast

At a target of 3, with channels sampled at every third value, neither direction failed to reach the
target for any color.

### Not measured

- A value entered through Figma's interface rather than the API. Surveyed variables were all valid 32-bit
  values, but opacity typed as a percentage in the color picker was not directly observed.
- An instance pasted into a file already in Display P3, and a file changed by converting rather than assigning.
- Whether Figma reports a change of profile to a running plugin.
- That Figma's color picker shows a Display P3 file's stored numbers, which its help page implies.
- How Figma's interface rounds an exact half in general. One color was checked: `#001801`, whose hue is
  exactly 122.5, shows as H 123, S 100, L 5, A 100% in the color picker.
- Other Figma clients.

## References

- [Manage color profiles in design files](https://help.figma.com/hc/en-us/articles/360039825114), Figma
  Help. Profiles, Legacy files, and converting against assigning.
- [DocumentNode](https://developers.figma.com/docs/plugins/api/DocumentNode/) and
  [Paint](https://developers.figma.com/docs/plugins/api/Paint/), Figma Plugin API. The document color
  profile, and how a solid fill holds color and opacity.
