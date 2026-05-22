---
name: data-visualization
description: Produce publication-quality charts for chemical, biological, and toxicological data using matplotlib, seaborn, plotly, and ggplot2; follows accessibility and scientific reporting standards
license: MIT
compatibility: opencode
metadata:
  domain: data-visualization
  languages: Python, R
---

## What I do

Guide the creation of clear, accurate, and accessible visualizations for cheminformatics, bioactivity, and toxicology datasets — covering dose-response curves, SAR heatmaps, chemical space plots, property distributions, and regulatory summary figures.

## When to use me

Load this skill when:
- Generating any chart, plot, or figure from chemical or biological data
- Deciding which plot type best represents a dataset
- Ensuring figures meet publication or regulatory submission standards
- Applying accessible color palettes and accessible design principles

---

## General principles

- **Data integrity first**: never alter the visual encoding to make data look better. Truncated y-axes, cherry-picked ranges, and misleading scales are scientific misconduct.
- **One message per figure**: each plot should communicate one clear finding. If you need to show two unrelated things, make two plots.
- **Label everything**: axis labels with units, figure title or caption, legend entries. A figure must be interpretable without reading surrounding text.
- **Accessibility**: use colorblind-safe palettes by default (see below). Never rely on color alone to encode categorical information — pair color with shape, linetype, or pattern.
- **Resolution**: export at ≥ 300 DPI for print/regulatory use (`dpi=300` in `savefig`). For screen/web use SVG or interactive HTML.

---

## Library choice

### Python
| Use case | Library |
|----------|---------|
| Static publication figures | `matplotlib` + `seaborn` |
| Interactive / exploratory | `plotly` (Dash for apps) |
| Chemical structure grids | `rdkit.Chem.Draw` + `PIL` |
| Large datasets (>10k points) | `datashader` or `plotly` WebGL |

### R
| Use case | Library |
|----------|---------|
| All static figures | `ggplot2` (primary) |
| Interactive | `plotly::ggplotly()` or `highcharter` |
| Chemical structures | `ChemmineR` + `ggplot2` |
| Dose-response | `drc` package |

---

## Colorblind-safe palettes

Always use one of these by default:

```python
# Python / matplotlib / seaborn
import seaborn as sns

# Categorical (≤ 8 groups)
palette = sns.color_palette("colorblind")   # Okabe-Ito-like

# Sequential (continuous / heatmap)
cmap = "viridis"     # perceptually uniform, colorblind-safe
cmap = "cividis"     # optimized for deuteranopia

# Diverging (centered data, e.g. log fold-change)
cmap = "RdBu_r"      # acceptable; or "coolwarm"
```

```r
# R / ggplot2
library(ggplot2)
scale_color_manual(values = c("#E69F00","#56B4E9","#009E73","#F0E442",
                               "#0072B2","#D55E00","#CC79A7","#000000"))
# or
scale_color_viridis_d()       # discrete
scale_fill_viridis_c()        # continuous
```

Never use rainbow/jet colormaps — they are perceptually non-uniform and inaccessible.

---

## Chart types by data situation

### Dose-response curves
**When**: plotting biological response vs. log concentration (IC50, EC50, LD50).

```python
import numpy as np
import matplotlib.pyplot as plt
from scipy.optimize import curve_fit

def hill(x, bottom, top, ec50, n):
    return bottom + (top - bottom) / (1 + (ec50 / x) ** n)

# Fit
popt, _ = curve_fit(hill, conc, response, p0=[0, 100, 1e-6, 1])

# Plot on log x-axis
x_fit = np.logspace(np.log10(conc.min()), np.log10(conc.max()), 200)
plt.semilogx(x_fit, hill(x_fit, *popt), '-', label='4PL fit')
plt.scatter(conc, response, zorder=5)
plt.axhline(50, linestyle='--', color='gray', label='50% response')
plt.xlabel('Concentration (M)')
plt.ylabel('Response (%)')
plt.legend()
```

In R with `drc`:
```r
library(drc)
fit <- drm(response ~ conc, data = df, fct = LL.4())
plot(fit, log = "x", xlab = "Concentration (M)", ylab = "Response (%)")
ED(fit, 50, interval = "delta")   # EC50 with 95% CI
```

Always: plot raw data points on top of the fitted curve; show 95% CI band; label EC50/IC50 with confidence interval in figure legend.

### SAR heatmap (compound × assay)
**When**: showing potency (pIC50, % inhibition) across a compound panel and multiple assays.

```python
import seaborn as sns
import pandas as pd

# df: rows = compounds, columns = assays, values = pIC50
g = sns.clustermap(
    df,
    cmap="viridis",
    linewidths=0.5,
    figsize=(12, 10),
    cbar_kws={"label": "pIC50"},
    yticklabels=True,
    xticklabels=True,
)
g.fig.suptitle("SAR Heatmap: pIC50 across assay panel", y=1.02)
```

- Use hierarchical clustering on both axes to reveal structure-activity patterns.
- Mask cells where data are missing rather than filling with zeros.
- Include a compound identifier column (CAS or ChEMBL ID) as a row annotation.

### Chemical space scatter plot
**When**: visualizing a compound library in 2D descriptor space (PCA, UMAP of fingerprints).

```python
import matplotlib.pyplot as plt
from matplotlib.lines import Line2D

fig, ax = plt.subplots(figsize=(8, 7))
scatter = ax.scatter(
    pca[:, 0], pca[:, 1],
    c=pic50,           # continuous color = activity
    cmap="viridis",
    alpha=0.7,
    s=30,
    edgecolors="none",
)
cbar = fig.colorbar(scatter, ax=ax)
cbar.set_label("pIC50")
ax.set_xlabel("PC1 (24.3% variance)")
ax.set_ylabel("PC2 (12.1% variance)")
ax.set_title("Chemical Space — Morgan ECFP4, PCA")
```

- Annotate outliers by compound ID (use `ax.annotate` for top-N active compounds).
- When coloring by category (scaffold class), pair color with marker shape.
- For UMAP: report n_neighbors and min_dist parameters in the caption.

### Property distribution plots
**When**: comparing physicochemical properties across a dataset or between approved drugs and a test set.

```python
import seaborn as sns

# Violin + strip (shows distribution shape and individual points)
ax = sns.violinplot(x="dataset", y="logP", data=df, inner=None, palette="colorblind")
sns.stripplot(x="dataset", y="logP", data=df, size=3, alpha=0.4, ax=ax, color=".3")
ax.axhline(5, linestyle="--", color="red", label="Lipinski LogP ≤ 5")
ax.set_xlabel("")
ax.set_ylabel("LogP (AlogP)")
ax.legend()
```

- Show the distribution, not just the mean/median. Violin + strip is preferred over bar + error bar for small-to-medium N.
- Annotate Lipinski or regulatory thresholds as horizontal reference lines.

### Volcano plot (bioactivity / toxicogenomics)
**When**: showing statistical significance vs. magnitude of effect (differential gene expression, assay hits).

```python
import matplotlib.pyplot as plt
import numpy as np

fig, ax = plt.subplots(figsize=(8, 6))
ax.scatter(df["log2fc"], -np.log10(df["pvalue"]),
           c=df["color_label"].map({"up": "#D55E00", "down": "#0072B2", "ns": "#999999"}),
           alpha=0.6, s=20, edgecolors="none")
ax.axvline(1, linestyle="--", linewidth=0.8, color="gray")
ax.axvline(-1, linestyle="--", linewidth=0.8, color="gray")
ax.axhline(-np.log10(0.05), linestyle="--", linewidth=0.8, color="gray")
ax.set_xlabel("log₂ Fold Change")
ax.set_ylabel("–log₁₀(p-value)")
ax.set_title("Differential Activity: Treatment vs. Control")
```

### Regulatory bar chart / summary table figure
**When**: presenting hazard data (LD50, NOAEL, RfD) for a chemical or a group of chemicals in a report.

- Use horizontal bar charts when compound names are long.
- Sort bars by value (descending hazard) for immediate readability.
- Show error bars only when reporting mean ± SD/SE from multiple studies; do not invent uncertainty.
- Use log scale for LD50 comparisons spanning orders of magnitude.

---

## Subplots and multi-panel figures

For regulatory reports and publications, multi-panel figures must follow consistent layout rules:

```python
fig, axes = plt.subplots(2, 2, figsize=(12, 10))
fig.tight_layout(pad=3.0)

# Panel labels (A, B, C, D) at top-left of each panel
for ax, label in zip(axes.flat, "ABCD"):
    ax.text(-0.12, 1.05, label, transform=ax.transAxes,
            fontsize=14, fontweight="bold", va="top")
```

- Panel labels: bold uppercase letters, placed outside the axes at top-left.
- Shared axis: use `sharex`/`sharey` to align axes; suppress redundant tick labels.
- Consistent font size across all panels in a figure (recommend: title 12pt, axis labels 10pt, tick labels 9pt).

---

## Exporting figures

```python
# High-resolution raster (journals, regulatory submissions)
fig.savefig("figure1.png", dpi=300, bbox_inches="tight")
fig.savefig("figure1.tiff", dpi=300, bbox_inches="tight")

# Vector (editable, lossless — preferred for line art / text)
fig.savefig("figure1.svg", bbox_inches="tight")
fig.savefig("figure1.pdf", bbox_inches="tight")
```

```r
# R
ggsave("figure1.png", plot = p, width = 8, height = 6, units = "in", dpi = 300)
ggsave("figure1.svg", plot = p, width = 8, height = 6)
```

---

## Accessibility checklist for figures

- [ ] Colorblind-safe palette (Okabe-Ito or viridis family)
- [ ] Color is not the sole encoding for any categorical variable (paired with shape/linetype/pattern)
- [ ] Minimum font size 9pt for tick labels, 10pt for axis labels
- [ ] Sufficient contrast between data elements and background (≥ 3:1 for large elements)
- [ ] Alt-text or caption describes what the figure shows for screen readers
- [ ] No information conveyed only by figure position relative to other figures

---

## Common pitfalls

- **Connecting non-sequential data points with a line**: only connect points if the x-axis is truly continuous (time series, concentration series). Never draw lines between discrete categories.
- **Dual y-axes**: almost always misleading. Use two separate panels instead.
- **3D bar / pie charts**: distort perception of magnitude. Use 2D bars or a simple table instead.
- **Overplotting**: use `alpha`, jitter, or hexbin/density when more than a few hundred points overlap.
- **Saturated color scales**: when using a diverging colormap, ensure zero / midpoint is centered on the neutral color, not offset by outliers. Use `vmin`/`vmax` symmetrically: `vmin=-v, vmax=v` where `v = abs(data).max()`.
- **Missing units in axis labels**: always include units. "LogP" is acceptable (unitless); "IC50" without units is not.
