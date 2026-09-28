# How to plot in gangaplot

`gangaplot.mplstyle` holds every rcParam. This file holds what does not fit in
a style file: figure size, layout, what each kind of mark means, and the order
in which to finish a figure. The reference is the figures of *When energy isn't
enough* (2026): a 2×2 grid of panels at revtex's `\columnwidth`.

## Get the style

Install it once: `cp gangaplot.mplstyle ~/.matplotlib/stylelib/` (or
`~/.config/matplotlib/stylelib/`), then

```python
plt.style.use("default")      # start clean, so nothing from an earlier style leaks in
plt.style.use("gangaplot")
if shutil.which("latex") is None:            # no TeX: Computer Modern via mathtext
    plt.rcParams.update({"text.usetex": False, "mathtext.fontset": "cm",
                         "font.serif": ["DejaVu Serif"]})
```

On macOS put `/Library/TeX/texbin` on `PATH` first, or an IDE will not find
`latex`. The TeX preamble loads `amsmath`, `amssymb` and `braket`, so
`$\braket{S^2}$` works in labels.

## Size: draw at the width it is printed

Draw at `\columnwidth` = 246 pt = 3.404 in and include it unscaled,
`\includegraphics{fig.pdf}`, never `[width=...]`. Any rescaling in LaTeX
multiplies every stroke and every font, and the style's numbers stop meaning
anything. For a `\textwidth` figure, build it at 495.08 pt = 6.850 in.

Text is absolute: 10 pt axis labels (the body size), 6 pt tick labels, 8 pt
legend. Strokes are the lost original's, scaled to a ~1 in panel but floored so
they stay black rather than antialiased grey: 0.7 pt spines, 0.45/0.30 pt
major/minor ticks. Do not shrink them further; at a smaller figure, cut text
before strokes.

## Layout

A grid of panels sharing x down each column, with a small gap and no titles:

```python
COLUMN = 246.0 / 72.27                     # in
ASPECT = 1080 / 1217                       # panel width / height
tick, lab = 6 / 72.27, 10 / 72.27          # tick-label and label sizes, in
side, bottom, top, gap = 4.2*tick + 1.8*lab, 2.2*tick + 1.8*lab, 0.34, 0.055
w = (COLUMN - 2*side) / (2 + gap)
h = 2 * w / ASPECT * (1 + gap/2) + bottom + top
fig, axes = plt.subplots(2, 2, figsize=(COLUMN, h), sharex="col", squeeze=False,
    gridspec_kw=dict(wspace=gap, hspace=gap, left=side/COLUMN, right=1 - side/COLUMN,
                     bottom=bottom/h, top=1 - top/h))
for ax in axes[:, -1]:                     # right column: labels on the right
    ax.yaxis.set_label_position("right")
    ax.tick_params(axis="y", which="both", left=True, right=True,
                   labelleft=False, labelright=True)
```

A single panel takes the same shape, centred, at 72% of the width between the
same outer margins.

- **Y labels are horizontal and outside the grid**: left column on the left,
  right column on the right. `ax.set_ylabel(s, rotation=0, ha="right",
  va="center", labelpad=6)` (`ha="left"` on the right). Align the left
  column's with `fig.align_ylabels`, then widen the margins if a label
  overflows the canvas.
- **One legend, above the grid**, gathered from every panel's labelled handles
  with duplicates dropped: `fig.legend(handles, labels, loc="upper center",
  bbox_to_anchor=(0.5, 1.0), ncol=3)`. If it is wider than the figure, shrink
  its font by 0.88 until it fits. Name guide lines in it rather than writing
  text beside them.
- **Panel letters** `(a)`–`(d)` go 3 pt in from a top corner, at 0.62 × label
  size + 0.38 × tick size (≈ 8.5 pt). Bottom-row panels take the top left;
  top-row panels take whichever top corner covers fewer data points.
- **Shared edges**: blank a tick label within 10% of an edge shared with the
  next panel (x on all but the last column, y on all but the top row), so
  labels from adjacent panels never touch.
- **Log axes** carry all nine minor ticks per decade and labelled decades no
  closer than 26 pt: `LogLocator(base=10**stride)` for the majors,
  `LogLocator(subs=range(1, 10), numticks=500)` with a `NullFormatter` for the
  minors.

Finish in this order, since each step needs the previous one drawn: legend →
log ticks → y-label alignment → panel letters → edge-label trimming.

## What marks mean

Theory is shading and lines; runs are markers, never lines.

| mark | how |
|---|---|
| exact / theory | `ax.plot(x, y, "k--", lw=1.25*0.55)`, zorder 1 |
| a run (good) | hollow `#0000a8` squares, `ls="none"`, zorder 3 |
| a run (poor) | hollow `#c92ad5` diamonds, `ls="none"`, zorder 3 |
| a region a run is allowed | `fill_between` in the run's colour, `alpha=0.22`, `lw=0`, zorder 0 |
| a reference level (F = 1, 10^0) | `axhline(y, color="k", ls=":", lw=0.7)`, zorder 0, labelled |

The prop cycle already gives good then poor, with markers, hollow, at the right
size (2.4 pt, 0.68 pt edges), so `ax.plot(x, y, ls="none")` draws a run
correctly. A format string clears the cycle's marker, so `"k--"` is a bare line;
set the colour by keyword and you must pass `marker=""` as well.

Two colours per figure is the norm: a poor example and a good one, the same
two in every panel. The cycle's third and fourth entries (pink `#ff5ca3`, gold
`#ffc23d`) are for the rare figure that needs them; gold is weak as a thin line
and is better kept for bands.

## Checking a figure

Look at the rendered figure at 100% before calling it done: a legend or panel
letter over data, a clipped label, or two tick labels touching across a shared
edge are all invisible in the code. When changing the style itself, render a
figure before and after and diff the pixels; checking the rcParams alone passes
exactly when a whole figure has been scaled wrong.
