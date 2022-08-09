# `avar`—Axis Variations Table Version 2

The axis variations table (`avar`) is an optional table used in variable fonts. Version 1 of `avar` modifies aspects of how a design varies for different instances along a particular design-variation axis. It does this by piecewise linear remapping on a per-axis basis, with certain restrictions. See the [OpenType specification](https://docs.microsoft.com/en-us/typography/opentype/spec/avar) for details.

Version 2, as proposed here, allows changes in one design-variation axis to affect many other design-variation axes, and conversely for one design-variation axis to be affected by many other design-variation axes.

In many cases, `avar` version 2 data does not offer brand new functionality but allows desirable functionality to be implemented much more efficiently by:

* reducing data footprint;
* easing source maintenance by avoiding redundant data;
* making fonts easier for users to control.

Use cases include:

* warped variable font designspaces to reflect a designer’s intention accurately;
* parametric fonts with intuitive control methods and a much reduced data footprint;
* offering simpler methods of control for specialized variable fonts.

# Table format

Axis variation table:

| Type	| Name	| Description |
| ------|-------|-------------|
| `uint16`	| `majorVersion`	| Major version number of the axis variations table — set to 2. |
| `uint16`	| `minorVersion`	| Minor version number of the axis variations table — set to 0. |
| `uint16`	| `<reserved>`	| Permanently reserved; set to zero. |
| `uint16`	| `axisCount`	| The number of variation axes for this font. This must be the same number as axisCount in the 'fvar' table. |
| `SegmentMaps`	| `axisSegmentMaps[axisCount]`	| The segment maps array — one segment map for each axis, in the order of axes specified in the 'fvar' table. |
| `Offset32To<DeltaSetIndexMap>` | **`axisIdxMap`** | Offset from beginning of the table, to optional DeltaSetIndexMap storing variation index mapping. |
| `Offset32To<ItemVariationStore>` | **`varStore`** | Offset from beginning of the table, to optional ItemVariationStore storing variations. |

 The table format for `avar` table version 2 is the same as `avar` table version 1 followed by two extra members `axisIdxMap` and `varStore`. It is expected that implementations that handle only version 1 with ignore the entire table by checking the `majorVersion` value.

# Processing

Processing of axis values in a `avar` version 2 table starts off the same as a `avar` version 1 table. That is, axis coordinates are normalized and then mapped according to `axisSegmentMaps`.

After that, the following algorithm is applied to produce the final normalized axis coordinates:

```c++
    // let coords be the vector of current normalized coordinates.

    std::vector<int> out;
    for (unsigned i = 0; i < coords.size(); i++)
    {
      int v = coords[i];
      uint32_t varidx = i;
      
      if (axisIdxMap != 0)
        varidx = (this+axisIdxMap).map(varidx);
      
      float delta = 0;
      
      if (varStore != 0)
        delta = (this+varStore).get_delta (varidx, coords);

      v += std::roundf (delta);
      v = std::clamp (v, -(1<<14), +(1<<14));

      out.push_back (v);
    }
    for (unsigned i = 0; i < coords.size(); i++)
      coords[i] = out[i];
```

# Discussion

This table uses the same variation mechanism used in multiple other places in OpenType 1.8, to vary normalized axis values themselves. In particular, variation deltas are stored in an `ItemVariationStore`, and the variation indices are stored in a `DeltaSetIndexMap`.

The processing is broken into two steps: let's call them the v1 and v2 steps. The v1 step is the same as an `avar` version 1 table. The v2 step is the algorithm listed above. The `ItemVariationStore` is driven with the normalized axis values generated by the v1 step, to produce deltas to add to those same normalized axis values, to produce the final normalized axis values. It is this step that allows arbitrary multi-axis transformation, because unlike the v1 step, the delta for, and hence adjustment to, each axis can be affected by the input value of every axis.

## Hidden axes

Since many `avar` version 2 fonts have axes not not intended for manual adjustment, it is recommended that such axes set the “hidden” flag in `VariationAxisRecord` of the [`fvar`](https://docs.microsoft.com/en-us/typography/opentype/spec/fvar) table.


# Construction

The way `ItemVariationStore`s are built is typically by using a variation modeler, that takes a series of _master_ values at certain locations in the design-space, and produces delta-sets to be stored in an `ItemVariationStore`. This usage is no different. To build the `avar` version 2 mapping tables, the designer will need to produce a mapping of input axis locations and their respective output axis locations. This data then will constitute the set of _masters_ to be fed to the variation modeler and populate the `ItemVariationStore` that will go into the `avar` version 2 table. The variation index for each axis will be stored in `axisIdxMap`.

# Use cases

## 1. Designspace warping

In the OpenType variations specification, [registered axes](https://docs.microsoft.com/en-us/typography/opentype/spec/dvaraxisreg) offer a standard way to offer users control of a variable font on reasonably well defined scales. For example, users learn that the Regular version of a font has Weight=400, Width=100, while the Bold has wght=700, wdth=100. A typical 2-axis variable font with `wght` and `wdth` axes might have the following nine Named Instances:


|  | **75** | **100** | **125** |
| --- |---|---|---|
| **300**   | Light Condensed | Light | Light Extended |
| **400** | Condensed | Regular | Extended |
| **700**    | Bold Condensed | Bold | Bold Extended |

However, in fonts with more than one design axis, this approach lacks the flexibility of older methods of interpolation, where the type designer used a font design application to specify freely the axis values for, say, the Bold Condensed. Notably, the `wght` coordinate of Bold Condensed did not need to match that of the Bold, nor did the `wdth` coordinate need to match that of the Condensed. A Bold Condensed with `wght`,`wdth` of (677,81) rather than (700,75) was perfectly reasonable. Our grid of instances can then look something like this:

|  | **Condensed** | **Regular** | **Expanded** |
| --- |---|---|---|
| **Light**   | 300,75 | 300,100 | 300,115 |
| **Regular** | 400,75 | 400,100 | 400,120 |
| **Bold**    | 677,81 | 695,100 | 700,125 |

While with OpenType 1.8 it is possible to set up a designspace like this, it has the significant drawback that many applications and systems expect all Bold weights to be at `wght`=700 and all Condensed weights to be at `wdth`=75, whatever the values of other axes. Thus, in practice, many OpenType 1.8 fonts encode numerous additional masters that ensure the design is as intended at all designspace locations. This not only impacts the data footprint significantly, but also adds to the maintenance burden of the font.

Using `avar` version 2 we can apply deltas to axis values anywhere in the designspace, thus “warping” it to resolve the issue described. As with variation deltas in general, we also benefit from interpolation when axis values are influenced by a delta but are not exactly at the location of the delta. Each delta value is encoded as F2DOT14. Its interpolated value is determined by the standard OpenType variations algorithm, then added to the value obtained from the standard normalization process. Thus, assuming the Bold Condensed instance at (700,75) has normalized coordinates (1,-1) and assuming the designspace location (677,81) has normalized coordinates ((677-400)/(700-400), -1 + (81-75)/(100-75)) = (0.9233,-0.76), then `avar2` needs a delta set with peaks at (-1,1), thus with region ((-1,-1,0),(0,1,1)) and with delta values:

* `wght` -0.0767
* `wdth` +0.24

In practice there may be other delta sets required at the partial default locations (-1,0) and (0,1), which then influence the delta values required at (-1,1), in line with normal variations math.

The result of implementing designspace warping this way is that we preserve the original axis values where applications and systems expect them, namely all Bold instances at `wght`=700 and all Condensed weights at `wdth`=75, while at the same time minimizing the data footprint.


## 2. Duplication of axis values for non-linear interpolation

TKTK


## 3. Simplified controls for parametric fonts without redundant data

TKTK