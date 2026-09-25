## PCB Layer Stackup

The board uses a 4 layer stackup, arranged to keep every signal close to a solid reference plane and every return path as short as possible.

<table>
<tr><th>Layer</th><th>Function</th></tr>
<tr><td>L1 (Top)</td><td>Components and signal routing, with GND copper pour in free areas</td></tr>
<tr><td>L2</td><td>Continuous solid GND plane</td></tr>
<tr><td>L3</td><td>3.3V power plane with a few routed traces</td></tr>
<tr><td>L4 (Bottom)</td><td>Components and signal routing, with GND copper pour in free areas</td></tr>
</table>

**Why this arrangement:**

1. The solid GND plane on L2 sits directly under the top layer, giving the high speed flash lines, crystal and RF feed an unbroken reference and a low impedance return path.
2. The dedicated 3.3V plane on L3 distributes power with minimal resistive loss. Components connect to it through a via at the pad edge, or a short trace and then a via.
3. The GND pours on L1 and L4, stitched to L2 with vias, give an even shorter return path for the surface components and add shielding around sensitive signals.
4. All layers are kept clear of copper beneath the antenna, so the planes and pours stop at the antenna feed boundary.
