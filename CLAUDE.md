# CLAUDE.md

Guidance for Claude (and contributors) working on this exporter. The conventions below were verified empirically against AE preview — please don't change them based on Adobe / forum / community sources without first re-running the probe in `Verifying against AE` below.

## Coordinate convention (TresSims)

```
position:  (x,  -y, -z)     # AE world → USD world
rotation:  (rx, -ry, -rz)
```

At the matrix level: bilateral conjugation by `S = diag(1, -1, -1)`:

```
M_usd_col = S · R_ae · S         (column-vector form)
M_usd_row = S · R_ae^T · S       (row-vector form for USD storage)
```

Identity AE → identity USD. Applied uniformly to cameras, lights, nulls, AVLayers — no per-prim-type branching. If you find yourself wanting to add a sign-flip for one prim type only, you've probably misdiagnosed something else.

## AE Orientation Euler order is `X*Y*Z`

In `aeRotMatrix`:

```javascript
var Mo = m3mul(rotX(ori[0]), m3mul(rotY(ori[1]), rotZ(ori[2])));   // ✓ correct
// NOT: m3mul(rotY(ori[1]), m3mul(rotX(ori[0]), rotZ(ori[2])));     // ✗ Y*X*Z is wrong
```

**Verified empirically** with the probe-null script (`Verifying against AE` below). The `Y*X*Z` order — which various Adobe docs, ProVideoCoalition articles, and community plugins assume — produces a ~2° drift on animated 2-node cameras. `X*Y*Z` matches AE's rendered camera matrix to ~1e-4 across all tested frames.

The X/Y/Z Rotation order (`Mi`) is currently `Z*Y*X`. Untested empirically (test cameras have all-zero individual rotations). If a similar drift appears on cameras/nulls with non-zero X/Y/Z Rotation values, suspect that order next.

## 2-node camera composition

For an AE 2-node camera with `autoOrient = CAMERA_OR_POINT_OF_INTEREST`:

```javascript
Rae = m3mul(lookAtMatrix(rawPos, poi),
            aeRotMatrix(rp.ori, rp.xr, rp.yr, rp.zr));
```

(column-vector form — `aeRotMatrix` is applied first, then `lookAt`)

For 1-node cameras (`NO_AUTO_ORIENT` or `ALONG_PATH`) and nulls:

```javascript
Rae = aeRotMatrix(rp.ori, rp.xr, rp.yr, rp.zr);
```

Other compositions tested and rejected (all worse than the above):

| Composition | Result |
|---|---|
| `lookAt × aeRotMatrix` (X\*Y\*Z order) | ✅ matches AE to ~1e-4 |
| `aeRotMatrix × lookAt` | wrong — R rotates in world, not camera-local |
| `lookAt` only | wrong — ignores user's keyframed orientation |
| `aeRotMatrix` only | wrong — ignores POI auto-orient |

So Adobe DOES apply orientation values on top of POI lookAt for 2-node cameras, despite some doc text suggesting they're "ignored". Don't tear out the composition.

## Verifying against AE (probe-null trick)

When the script's camera matrix doesn't match AE preview, **don't iterate on math fixes by guessing**. Run this to get AE's ground-truth camera matrix at sample frames, then compare and solve.

`Layer.toWorld()` is not directly callable on a `CameraLayer` from ExtendScript (errors `Function layer.toWorld is undefined`), but it IS available in expressions. We use that.

Run with the target camera selected. Cleans up after itself:

```javascript
(function () {
    var comp = app.project.activeItem;
    var cam = comp.selectedLayers[0];
    if (!cam || !(cam instanceof CameraLayer)) {
        alert("Select a camera layer first"); return;
    }
    var fps = comp.frameRate;
    var frames = [0, 30, 60];   // edit as needed
    var probes = [[0,0,0], [100,0,0], [0,100,0], [0,0,100]];
    var nulls = [];
    app.beginUndoGroup("camera matrix probe");
    try {
        for (var i = 0; i < probes.length; i++) {
            var n = comp.layers.addNull();
            n.threeDLayer = true;
            n.name = "_PROBE_" + i;
            n.transform.position.expression =
                'thisComp.layer("' + cam.name + '").toWorld([' + probes[i].join(",") + '])';
            nulls.push(n);
        }
        var out = "Camera: " + cam.name + "  autoOrient=" + cam.autoOrient + "\n\n";
        for (var f = 0; f < frames.length; f++) {
            var t = frames[f] / fps;
            var o = nulls[0].transform.position.valueAtTime(t, false);
            out += "Frame " + frames[f] + ":\n  origin: (" + o.join(", ") + ")\n";
            var labels = ["+X", "+Y", "+Z"];
            for (var ax = 1; ax < 4; ax++) {
                var p = nulls[ax].transform.position.valueAtTime(t, false);
                var d = [(p[0]-o[0])/100, (p[1]-o[1])/100, (p[2]-o[2])/100];
                out += "  " + labels[ax-1] + ":     (" +
                       d[0].toFixed(4) + ", " + d[1].toFixed(4) + ", " + d[2].toFixed(4) + ")\n";
            }
            out += "\n";
        }
        alert(out);
    } finally {
        for (var i = nulls.length - 1; i >= 0; i--) { try { nulls[i].remove(); } catch (e) {} }
        app.endUndoGroup();
    }
})();
```

The `+X / +Y / +Z` lines are the camera's local axes expressed in AE world coordinates — i.e. the columns of the camera's rotation matrix. That's the ground truth to compare against the script's output.

## Camera focal length

Read per-frame from `cameraOption.zoom` (pixels) and converted via:

```javascript
flS.push([frame, FILM_WIDTH_MM * zoom / comp.width * MM_TO_USD]);
```

`FILM_WIDTH_MM = 36` is hardcoded — AE's `filmSize` property isn't exposed via ExtendScript. If a project uses a non-default film back (e.g. APS-C 24 mm), edit the constant at the top of the script or expose it in the dialog.

USD `focalLength` units are "tenths of scene unit" with `metersPerUnit = 1`. A 50 mm AE camera lands as `focalLength = 0.5`, which Houdini reads correctly.

## Build version

Bump `BUILD_DATE` (YYMMDD format with optional letter suffix) at the top of the script on each meaningful change. Shown in the dialog title.

## What's verified vs not

End-to-end against AE preview:
- ✅ Translation, hierarchy, comp-centre offset
- ✅ 1-node camera (static + animated)
- ✅ 2-node camera (static + animated, with keyframed Orientation)
- ✅ Camera focal length, aperture, focus distance
- ✅ Static-value optimisation, dedup of held timeSamples
- ✅ Visibility filter (eyeball off, solo)

Not yet visually verified:
- ⚠️ All four light types (data is exported, framing/falloff in Houdini not confirmed)
- ⚠️ 3D AVLayer anchor point (currently ignored — will mis-pivot for non-default anchors)
- ⚠️ Parented 2-node camera with animated parent (POI is in world space, position in parent space — mixed-space lookAt)
- ⚠️ Negative scale on AVLayers
- ⚠️ Layer time-stretch / time-remap (script ignores)

When chasing one of these, run the probe trick first if rotation/orientation is involved.
