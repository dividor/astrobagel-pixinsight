# PJSR snippets by stage

Join the statements of a stage into one line before entering them. Replace `<...>` placeholders. `img` is the view id used in these examples; choose a short id per target.

Row formats worth knowing:

- `ScreenTransferFunction.STF` rows are `[c0, c1, m, r0, r1]`.
- `HistogramTransformation.H` rows are `[c0, m, c1, r0, r1]`; the fourth row is the combined RGB/K channel.
- `MultiscaleLinearTransform.layers` rows are `[enabled, biasEnabled, bias, noiseReductionEnabled, threshold, amount, iterations]`.

## Helper: auto screen stretch with channel statistics

Changes only the display, not the pixels.

```
function stf(v){var c0=0,m=0;for(var c=0;c<3;c++){v.image.selectedChannel=c;var med=v.image.median();var mad=v.image.MAD()*1.4826;console.writeln("CH"+c+" median="+med+" mad="+mad);c0+=med-2.8*mad;m+=med;}v.image.resetSelections();c0=Math.max(0,c0/3);m=Math.mtf(0.25,m/3-c0);var S=new ScreenTransferFunction;S.STF=[[c0,1,m,0,1],[c0,1,m,0,1],[c0,1,m,0,1],[0,1,0.5,0,1]];S.executeOn(v);}
```

## 1. Open and measure

```
var w=ImageWindow.open("<full path to master>")[0]; w.mainView.id="img"; w.show(); var v=w.mainView;
console.writeln("OPEN "+v.image.width+"x"+v.image.height+" astrometry="+w.hasAstrometricSolution);
stf(v); w.zoomFactor=-8; w.bringToFront();
```

A plain WBPP master also opens its rejection maps as extra windows. Close them:

```
ImageWindow.windowById("<id>_rejection_low").forceClose();
```

## 2. Background removal

Object with sky around it:

```
var G=new GradientCorrection; console.writeln("GC_RESULT "+G.executeOn(v));
```

Object filling the frame (plane fit):

```
var A=new AutomaticBackgroundExtractor; A.polyDegree=1; A.targetCorrection=1; A.normalize=false; A.discardModel=true; A.replaceTarget=true; console.writeln("ABE_RESULT "+A.executeOn(v));
```

`targetCorrection` must be the integer 1 (subtraction). The prototype constant throws an error.

## 3. Color calibration

First choice:

```
var ok=false; try{var S=new SpectrophotometricColorCalibration; ok=S.executeOn(v);}catch(e){} console.writeln("SPCC_RESULT "+ok);
```

Fallback. Pick a 300 x 300 pixel sky patch in the corner farthest from the object and substitute its corners for `X0, Y0, X1, Y1`. Run BackgroundNeutralization before ColorCalibration.

```
var B=new BackgroundNeutralization; B.backgroundLow=0; B.backgroundHigh=0.6; B.useROI=true; B.roiX0=X0; B.roiY0=Y0; B.roiX1=X1; B.roiY1=Y1; console.writeln("BN_RESULT "+B.executeOn(v));
var C=new ColorCalibration; C.whiteLow=0; C.whiteHigh=0.9; C.whiteUseROI=false; C.structureDetection=false; C.backgroundLow=0; C.backgroundHigh=0.1; C.backgroundUseROI=true; C.backgroundROIX0=X0; C.backgroundROIY0=Y0; C.backgroundROIX1=X1; C.backgroundROIY1=Y1; console.writeln("CC_RESULT "+C.executeOn(v));
```

With structure detection off and no white region, the white reference is the whole frame, which suits a frame dominated by a galaxy. For a nebula field set `C.structureDetection=true` so stars set the white reference.

If the console prints the background reference as `0.00000e+00`, no pixels were inside the limits: raise `backgroundHigh` above the sky level and rerun.

## 4. Permanent stretch

```
var c0=0,m=0; for(var c=0;c<3;c++){v.image.selectedChannel=c;var med=v.image.median();var mad=v.image.MAD()*1.4826;c0+=med-2.8*mad;m+=med;} v.image.resetSelections(); c0=Math.max(0,c0/3); m=Math.mtf(0.20,m/3-c0);
var H=new HistogramTransformation; H.H=[[0,0.5,1,0,1],[0,0.5,1,0,1],[0,0.5,1,0,1],[c0,m,1,0,1],[0,0.5,1,0,1]]; console.writeln("HT_RESULT "+H.executeOn(v));
var S=new ScreenTransferFunction; S.executeOn(v);
```

## 5. Star separation

```
var P=new PixelMath; P.expression="$T"; P.createNewImage=true; P.newImageId="img_orig"; P.executeOn(v);
var N=new StarNet2; N.linear=false; console.writeln("SN_RESULT "+N.executeOn(v));
var Q=new PixelMath; Q.expression="~(~$T/max(~img,0.0001))"; Q.rescale=false; Q.truncate=true; console.writeln("STARS_RESULT "+Q.executeOn(View.viewById("img_orig"))); ImageWindow.windowById("img_orig").iconize();
```

After this, `img` is starless and `img_orig` holds only the stars.

## 6. Starless work

Noise reduction (raise the thresholds a little for noisy data):

```
var M=new MultiscaleLinearTransform; M.layers=[[true,true,0,true,3.0,0.6,3],[true,true,0,true,2.0,0.6,2],[true,true,0,true,1.0,0.5,2],[true,true,0,true,0.5,0.5,1],[true,true,0,false,3.0,1.0,1]]; console.writeln("MLT_RESULT "+M.executeOn(v));
```

Curves. `K` is the combined RGB curve and `S` is saturation. These are starting points.

Nebula:

```
var C=new CurvesTransformation; C.K=[[0,0],[0.15,0.07],[0.30,0.30],[0.60,0.70],[1,1]]; C.S=[[0,0],[0.40,0.60],[1,1]]; console.writeln("CURVES_RESULT "+C.executeOn(v));
```

If the nebula is still dim, a second lift: `C.K=[[0,0],[0.07,0.06],[0.25,0.38],[0.60,0.80],[1,1]]`.

Galaxy (gentler, keeps the faint outer disk):

```
var C=new CurvesTransformation; C.K=[[0,0],[0.12,0.05],[0.25,0.22],[0.50,0.58],[0.80,0.90],[1,1]]; C.S=[[0,0],[0.35,0.60],[1,1]]; console.writeln("CURVES_RESULT "+C.executeOn(v));
```

Clipped galaxy core to neutral white (blends toward the brightest channel only where the image is already very bright):

```
var F=new PixelMath; F.expression="$T+min(max((max($T[0],$T[1],$T[2])-0.75)/0.15,0),1)*(max($T[0],$T[1],$T[2])-$T)"; F.rescale=false; F.truncate=true; console.writeln("COREWHITE_RESULT "+F.executeOn(v));
```

Mild green reduction (galaxies; avoid on OIII-rich nebulae):

```
var N=new SCNR; N.amount=0.6; console.writeln("SCNR_RESULT "+N.executeOn(v));
```

Optional core compression, only if the user asks for it:

```
var D=new HDRMultiscaleTransform; D.numberOfLayers=6; D.toLightness=true; D.luminanceMask=true; console.writeln("HDR_RESULT "+D.executeOn(v));
```

Starless copy for export:

```
var P=new PixelMath; P.expression="$T"; P.createNewImage=true; P.newImageId="img_starless"; P.executeOn(v); ImageWindow.windowById("img_starless").iconize();
```

## 7. Stars back

`k` is 0.85 for dense star fields, 0.9 otherwise. Remember the value used.

```
var Q=new PixelMath; Q.expression="~(~$T*~(img_orig*0.9))"; Q.rescale=false; Q.truncate=true; console.writeln("COMBINE_RESULT "+Q.executeOn(v));
```

To rebuild a starless image later from a combined one: `~(~$T/max(~(img_orig*k),0.0001))` with the same `k`.

## 8. Save

`saveAs(path, queryOptions, allowMessages, strict, verifyOverwrite)`.

```
var d="<folder>/<Name>_processed"; var w=ImageWindow.windowById("img");
try{console.writeln("SAVE_XISF "+w.saveAs(d+".xisf",false,false,false,false));
var P=new PixelMath; P.expression="$T"; P.createNewImage=true; P.newImageId="img_out"; P.executeOn(w.mainView); var j=ImageWindow.windowById("img_out");
j.setSampleFormat(16,false); console.writeln("SAVE_TIF "+j.saveAs(d+".tif",false,false,false,false)+" bits="+j.bitsPerSample);
j.setSampleFormat(8,false); console.writeln("SAVE_JPG "+j.saveAs(d+".jpg",false,false,false,false)); j.forceClose();
var s=ImageWindow.windowById("img_starless"); s.setSampleFormat(16,false); console.writeln("SAVE_SL_TIF "+s.saveAs(d+"_starless.tif",false,false,false,false));
s.setSampleFormat(8,false); console.writeln("SAVE_SL_JPG "+s.saveAs(d+"_starless.jpg",false,false,false,false));}catch(e){console.writeln("SAVE_FAIL "+e);}
```

Check that the target names do not already exist before saving; `saveAs` here overwrites without asking. Do not rerun this script after the starless window has been converted to 8-bit, or the 16-bit starless TIFF is rewritten from 8-bit data.
