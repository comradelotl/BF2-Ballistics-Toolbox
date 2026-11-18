# BF2 Ballistics Toolbox: maths reference

Every equation the HTML uses in sections ① to ⑦, as implemented. Angles in radians unless marked °.
Not covered: DXT5/DDS export encoding, UI handling, input parsing.

## Conventions (shared)

| Quantity | Equation | Note |
|---|---|---|
| Focal length (px) | $f = \dfrac{H/2}{\tan(\mathrm{FOV}_v/2)}$ | $H$ = image height, $\mathrm{FOV}_v$ = vertical FOV |
| Pixels → angle | $\theta = \arctan(\mathrm{px}/f)$ | used by all px readouts |
| Pixels → MOA | $M(\mathrm{px}) = \theta \cdot \dfrac{180}{\pi}\cdot 60$ | |
| Mils per radian | mrad $1000$, NATO $\dfrac{6400}{2\pi}$, Warsaw $\dfrac{6000}{2\pi}$, MOA $\dfrac{21600}{2\pi}$ | $m_r$ below |
| px per mil | $p_m = f / m_r$ | |
| px per mrad | $p = f/1000$ | ladder, rangefinder |
| px per MOA | $f / 3437.747$ | $3437.747 = 60\cdot180/\pi$ |
| Overlay scale | $s = 600/H$ | all px values × $s$ on the 800×600 overlay |
| HUD frame width | $w = \tfrac{4}{3}H$ | dashed 4:3 box |

## ① Camera, Scope & Zoom

| Quantity | Equation | Note |
|---|---|---|
| TFOV from m/1000 m | $2\arctan(v/2000)$ | also ft/1000 yd: $v/6000$, m/100 m: $v/200$, ft/100 yd: $v/600$ |
| TFOV from size at distance | $2\arctan\!\big(v/(2d)\big)$ | ft @ yd uses $d\to 3d$ |
| Magnification | $M = \dfrac{\tan(\mathrm{AFOV}/2)}{\tan(\mathrm{TFOV}/2)}$ | from TFOV + AFOV |
| Magnification (optics) | $M = \mathrm{objective}/\mathrm{exit\ pupil}$ | |
| AFOV | $2\arctan\!\big(M\tan(\mathrm{TFOV}/2)\big)$ | |
| TFOV | $2\arctan\!\big(\tan(\mathrm{AFOV}/2)/M\big)$ | |
| Game magnification | $M_g = M^{\alpha}$ | $\alpha=1$ screen, $0$ AFOV |
| addZoomFactor, camera mode | $\mathrm{fr} = 2\arctan\!\big(\tan(b/2)/M_g\big)$ | $b$ = soldier camera FOV (1.1 rad) |
| addZoomFactor, scope mode | $\mathrm{fr} = 2\arctan\!\big(\tan(\mathrm{AFOV}/2)/M_g\big)$ | |
| Cap | $\mathrm{fr} \le b - 0.001$ | |
| Vertical FOV | $\mathrm{FOV}_v = \mathrm{fr}\cdot 180/\pi$ | feeds $f$ |
| In-game scope circle (px) | $R_{opt} = \dfrac{H}{2}\cdot\dfrac{\tan(\mathrm{TFOV}_r/2)}{\tan(\mathrm{fr}/2)}$ | $\mathrm{TFOV}_r$ = TFOV at real $M$; scope mode, $\alpha \le 1$ |

## ② Mildot Reticle

| Quantity | Equation | Note |
|---|---|---|
| Dot spacing (px) | $\mathrm{step} = \max(ms, 0.01)\cdot p_m$ | $ms$ = spacing in chosen mil |
| Dot $k$ position | $c \pm k\cdot\mathrm{step}$ | while $k\cdot\mathrm{step} \le r - d_r$ |
| Dot radius (px) | $d_r = \max(0.1\,p_m,\ 2)$ | |
| Line mark | half length $0.5\,p_m$, width $\max(0.03\,p_m,\ 1)$ | |
| Scope FOV | $a = 2\arctan(r/f)$, mil $= a\cdot m_r$ | $r$ = scope radius px |

## ③ Bullet Drop Compensation

**Setup**

| Quantity | Equation | Note |
|---|---|---|
| Effective gravity | $g = 9.81\cdot b_f\cdot b_g$ | $b_f = 1.5$ base, $b_g$ = gravityModifier |
| Drag constant | $k = \pi\cdot 0.01\cdot \mathrm{drag}/m$ | $m=1$ kg if unset |
| Launch angle | $\alpha = (\mathrm{be} + \mathrm{hrx})\cdot\pi/180$ | barrel elevation + bone rotation |
| Launch velocity | $v_x = v\cos\alpha,\quad v_y = v\sin\alpha$ | |
| Bone offset height | $h = -hc\cdot hk$ | $hk = 1.0217$ |
| Range limit (≈) | $v\cos\alpha / k$ | horizontal |

**Tick step** ($\Delta t = 1/30$ s)

| Order | Update | Note |
|---|---|---|
| Velocity first | $v_x \mathrel{-}= k v_x\Delta t,\ v_y \mathrel{-}= (k v_y + g)\Delta t,\ x \mathrel{+}= v_x\Delta t,\ y \mathrel{+}= v_y\Delta t$ | engine default |
| Move first | $x \mathrel{+}= v_x\Delta t,\ y \mathrel{+}= v_y\Delta t$, then the same velocity update | |
| Smooth | $e = e^{-k\Delta t},\ w = v_y + g/k$; $x \mathrel{+}= v_x(1-e)/k$; $y \mathrel{+}= w(1-e)/k - g\Delta t/k$; $v_x \mathrel{*}= e$; $v_y = we - g/k$ | exact drag integral |
| Smooth, $k\Delta t<10^{-9}$ | $y \mathrel{+}= v_y\Delta t - g\Delta t^2/2,\ v_y \mathrel{-}= g\Delta t$ | no drag |

**Outputs**

| Quantity | Equation | Note |
|---|---|---|
| Trajectory height | $y(x)$ by linear interpolation between ticks | |
| Hold (mrad) | $\mathrm{hold}(x) = 1000\,\dfrac{h - y(x)}{x}$ | positive = below crosshair |
| Hold at 0 m | $-1000\,Y_1/X_1$ | first tick |
| Mark position (px) | $y_{px} = c_y + \mathrm{hold}\cdot p$ | |
| Zero elevation | solve $y(R_z) = h$ for $\alpha$ | ternary search for the peak, then bisection, 50 iterations each, range −45° to 89° |
| Barrel elevation | $\mathrm{be} = \alpha_z - \mathrm{hrx}$ | |
| Far zero | linear interpolation at the downward crossing $y = h$ | $x = X_{i-1} + (X_i - X_{i-1})\frac{pv}{pv - v}$ |
| Texture px per mil | $p\cdot 600/H$ | |

**Drag estimation**

| Quantity | Equation | Note |
|---|---|---|
| Fit | $\min_k\ (p_1-d_1)^2 + (p_2-d_2)^2$ | $p_i$ = predicted hold at range $i$ |
| Search | grid $k\leftarrow 1.5k$ from $0.001$, then golden section ($\varphi = (\sqrt5-1)/2$) | 80 iterations |
| Drag value | $\mathrm{drag} = k\,m/(\pi\cdot 0.01),\ m = \mathrm{mass}_g/1000$ | |
| Drop → mrad | NATO $\times\,2000\pi/6400$; MOA $\times\,2000\pi/21600$; inch $\times\,0.0254/R\cdot 1000$ | inverse for display |
| Yards → m | $\times\,0.9144$ | |
| APPLY gravity | $b_g = 0.6667$ | |

**Bone offset from `_zoom_stand.baf`**

| Quantity | Equation | Note |
|---|---|---|
| Quaternion → matrix | $s = 2/\lVert q\rVert^2$; $R = \begin{pmatrix}1-s(y^2+z^2) & s(xy-wz) & s(xz+wy)\\ s(xy+wz) & 1-s(x^2+z^2) & s(yz-wx)\\ s(xz-wy) & s(yz+wx) & 1-s(x^2+y^2)\end{pmatrix}$ | |
| Local transform | $L = T(p)\,R(q)$ | |
| World pose | $W = \prod L_i$ along the chain | Camerabone; torus → mesh1 |
| Relative pose | $\mathrm{rel} = \mathrm{ref}^{-1}\cdot\mathrm{tgt}$, with $\begin{pmatrix}R & t\end{pmatrix}^{-1} = \begin{pmatrix}R^T & -R^T t\end{pmatrix}$ | rigid inverse |
| Offset | $hc = \mathrm{rel}_{1,3}$ | metres |
| Rotation | $\mathrm{hrx} = -\mathrm{atan2}(\mathrm{rel}_{2,1},\ \mathrm{rel}_{2,2})\cdot 180/\pi$ | |
| Fixed16 decode | $v = \mathrm{word}\big/\big(32767/2^{15-\mathrm{prec}}\big)$ | prec 15 for rotation channels; rotation x, y, z negated |

## ④ Stadiametric Rangefinder

| Quantity | Equation | Note |
|---|---|---|
| Target height (mrad) | $\mu = 1000\,h/x$ | small-angle form |
| Bracket height (px) | $\mu\cdot p$ | drawn from the crosshair upwards |
| Bracket top | $y = c_y - \mu\cdot p$ | |
| Ranges | $x = 100\cdot n$ | $n$ = entered hundreds of m |

## ⑤ Geometric Estimation

| Quantity | Equation | Note |
|---|---|---|
| Scope diameter | $d = \lVert b - a\rVert$ | px |
| Focal length from FOV | $f = \dfrac{d/2}{\tan(\mathrm{FOV}/2)}$ | op 1, op 3 |
| Angle between two points | $u = (a - c,\ f),\ w = (b - c,\ f)$; $\theta = \mathrm{atan2}\!\big(\lVert u\times w\rVert,\ u\cdot w\big)$ | $c$ = scope centre |
| Object size (op 1) | $S = 2R\tan(\theta/2)$ | |
| Range (op 3) | $R = \dfrac{S}{2\tan(\theta/2)}$ | |
| Mil size | $\theta\cdot m_r$ | |
| FOV from reference (op 2) | $f = \dfrac{\lVert O_1\rVert/2}{\tan\!\big(\arctan\frac{S}{2R}\big)},\quad \mathrm{FOV} = 2\arctan\dfrac{d/2}{f}$ | calibrated on $O_1$ |

## ⑥ Shot Group & MOA

| Quantity | Equation | Note |
|---|---|---|
| Radius about crosshair | $r_i = \lVert (x_i - c_x,\ y_i - c_y)\rVert$ | $c = (W/2,\ H/2)$ |
| MPI offset | $\bar{x} = \mathrm{mean}(x_i - c_x),\ \bar{y} = \mathrm{mean}(y_i - c_y)$ | |
| Mean / median | $\mathrm{mean}(r),\ \mathrm{pct}(r,\,0.5)$ | |
| RMS | $\sqrt{\mathrm{mean}(r^2)}$ | yellow circle |
| Percentile | $k = (n-1)q$, interpolate between neighbours | P90, P95 |
| Group centre | $\bar{x}_g,\ \bar{y}_g$ per group | |
| Pooled sums | $s_x = \sum (x - \bar{x}_g)^2,\ s_y = \sum (y - \bar{y}_g)^2$ | |
| Degrees of freedom | $\mathrm{dof} = \sum (n_g - 1)$ | |
| RMS radius (bias-corrected) | $\sqrt{(s_x + s_y)/\mathrm{dof}}$ | |
| Per-axis σ | $\sigma_x = \sqrt{s_x/\mathrm{dof}},\ \sigma_y = \sqrt{s_y/\mathrm{dof}}$ | |
| Extreme spread | $\max_{i<j}\lVert p_i - p_j\rVert$ per group | mean and max shown |
| All MOA values | $M(\mathrm{px})$ | see Conventions |

## ⑦ Deviation

**Constants:** $K = 46$ MOA RMS per minDev; $E = 2.172$ (mean 5-shot ES / RMS); uncertainty $10\,\%$ on $K$; floor $0.010000001$.

| Quantity | Equation | Note |
|---|---|---|
| Per-axis σ (MOA) | $\sigma = d\,K/\sqrt2$ | $d$ = minDev |
| RMS (MOA) | $\mathrm{RMS} = \sigma\sqrt2 = dK$ | |
| Mean ES, 5 shots | $E\cdot\mathrm{RMS}$ | |
| Target half-size (MOA) | $h = \arctan\!\dfrac{s}{2R}\cdot\dfrac{180}{\pi}\cdot 60$ | $s$ = size, $R$ = range |
| Hit chance | $P = \mathrm{erf}\!\dfrac{h_w}{\sigma\sqrt2}\cdot\mathrm{erf}\!\dfrac{h_h}{\sigma\sqrt2}$ | independent x, y; square: $w=h$ |
| erf | $1 - (a_1t + a_2t^2 + a_3t^3 + a_4t^4 + a_5t^5)e^{-x^2},\ t = \dfrac{1}{1 + 0.3275911x}$ | Abramowitz–Stegun 7.1.26 |
| Required minDev | solve $P(\sigma) = P_{target}$ by bisection (90 steps), $d = \sigma\sqrt2/K$ | |
| Plausible range | $P$ at $K(1+0.1)$ and $K(1-0.1)$ | |
| Zero deviation | $d < 0.010000001 \Rightarrow P = 100\,\%$ | |
| MOA → metres | $2R\tan\!\big(\mathrm{MOA}\cdot\frac{\pi}{180\cdot 60}\big/2\big)$ | |

**Heatmap**

| Quantity | Equation | Note |
|---|---|---|
| Radius holding fraction $p$ | $r_p = \sigma\sqrt{-2\ln(1-p)}$ | Rayleigh |
| Zone radii | $p = 0.25,\ 0.50,\ 0.75,\ 0.99 \to 0.759\sigma,\ 1.177\sigma,\ 1.665\sigma,\ 3.035\sigma$ | red, yellow, green, blue |
| Average radius ring | $r_{avg} = \sigma\sqrt{\pi/2} \approx 1.253\sigma$ | white ring |
| σ in pixels | $\sigma_{px} = \sigma\cdot f/3437.747$ | |
| Blue fade | alpha $1 \to 0$, linear in radius from $r_{0.75}$ to $r_{0.99}$ | radial gradient |
| Target square side (px) | $f\cdot s/R$ | centred, 1 px stroke, difference blend (inverts) |
