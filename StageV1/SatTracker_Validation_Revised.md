# How SatTracker Calculates and Predicts Satellite Positions

## Scientific methods, numerical validation and their limitations

**Revised editorial draft — 11 November 2026**  
**Intended readers:** satellite observers, radio amateurs, engineers and spaceflight professionals.  
**Status:** narrative and figure specifications for review; not yet a typeset publication. All quoted validation numbers refer to the archived November 2026 experimental scientific engine, **not** to a completed production release.

> **Production note (not for publication):** This manuscript is the editorial authority for a subsequent illustrated report. Bracketed **FIGURE BRIEF** passages describe illustrations to be prepared from archived data. The existing 28-page V1 engineering validation report and the B1–B3b reports remain the numerical evidence, not prose to be copied wholesale. Codex should check every numerical claim and refer any substantive discrepancy back for editorial review rather than silently change the narrative.

---

## Executive Summary

SatTracker predicts where an Earth-orbiting satellite will appear in the sky, when it will rise above and fall below a selected elevation, how far away it is, and how its motion changes the frequency of a received radio signal. These apparently straightforward answers require several coordinated calculations: orbit propagation, accurate treatment of time, the rotation and orientation of the Earth, the position of the observer, and the detection of events along a satellite pass.

The scientific foundation examined here uses the established SGP4 orbital model, together with independently implemented calculations for time, Earth orientation, observer geometry, range rate, one-way Doppler shift and geometric pass prediction. The investigations covered the 33-input propagation regression corpus, 2,799 observer-geometry requests and a final suite of 196 pass-search configurations. In the practical satellite-pass investigation, the algorithm detected all **310 complete and 14 clipped visibility intervals** identified by an independent reference across **153 searches**, including deliberately constructed grazing intervals lasting only a few seconds. The 196-search regression preserved **359 intervals and 672 crossings** after corrections to event reporting. These are overlapping and sometimes deliberately repeated test populations, not independent satellite observations.

The numerical comparisons show very close agreement in many parts of the calculation chain. For example, the largest matched classical terrestrial-frame position difference was **0.162 mm** across 248 comparisons, and the largest integrated slant-range arithmetic difference was about **30 nanometres** across 1,260 requests. These figures describe agreement between calculations using specified inputs. They are not estimates of the uncertainty in a satellite's actual orbit.

The tests also revealed limitations worth understanding. Different terrestrial-frame conventions can produce metre-scale differences; SGP4's returned velocity is not always the exact derivative of its position curve; and the timing of an extremely shallow satellite pass can be numerically sensitive. One deliberately artificial, millisecond-scale visibility interval escaped the search, although no genuine SGP4 interval was missed in the bounded practical test set. Several strict numerical comparison criteria were exceeded and remain documented rather than concealed.

For an observer, the most important external limitation is often the orbital data themselves. Precise orbits for specialist altimetry missions can be determined to centimetre or sub-centimetre radial quality by established tracking-data and orbit-comparison assessments, whereas predictions for lower satellites subject to atmospheric drag and manoeuvres can differ by kilometres after days. The detailed report explains the calculations, the evidence behind the validation, and the circumstances in which their results need careful interpretation.

---

## Introduction — Where is that satellite?

The thousands, or tens of thousands, of active satellites in Earth-orbit have a multitude of purposes, but one thing they have in common is that they make contact with installations on the ground for the exchange of data, via radio signals or sometimes laser communications. There are a few exceptions, which route thier communications via a relay satellite of some sort. There are also non-communicating objects in orbit: defunct satellites, old rocket bodies, debris from collisions, and so on.

Whether the objective is to downlink data from the satellite's payload, perform command and control with its computer, use a telescope or imaging system to make visual observations of it, or as a radio amateur, use the satellite's transponders as a relay to talk to others, the challenge remains the same: to align the radio or optical system with the direction to the satellite at the right time and to follow its movement across the sky.

Predicting that apparently simple event involves several different calculations. We must first predict where the spacecraft will be in its orbit. We must then account for the rotation and slight changes in orientation of the Earth, locate the observer on its curved surface, and finally determine the direction of the line joining observer and satellite. If we want to predict a pass rather than a single position, we must repeat these calculations and identify precisely when the satellite crosses a chosen elevation threshold. For a radio contact there is an extra consideeration: the rate at which the range changes determines the first-order Doppler shift. 

But the demands of looking for a satellite visually, pointing a telescope and acquiring a two-way telemetry, tracking and command (TTC) link are not identical. A satellite being geometrically above the horizon does not mean that it is illuminated, visible to the eye or available for a useful radio contact.

SatTracker combines a well-established orbit-propagation method with new calculations for time, Earth orientation, observer geometry and satellite-pass prediction. These calculations have been examined in a series of independent numerical investigations. The purpose of this document is to explain how they work, what the tests demonstrate, and—equally importantly—what they do not demonstrate.

A distinction is needed between how closely two calculations agree and how accurately the resulting trajectory represents a satellite. A range difference of thirty nanometres between independently calculated results is excellent evidence for the numerical geometry; the orbital elements used in both calculations may nevertheless be uncertain by kilometres.

The contrast with precise orbit determination (POD) is instructive. For a radar-altimetry satellite, radial orbit errors feed directly into the inferred sea-surface height, so specialist missions carry dedicated tracking systems and use elaborate orbit-estimation procedures. There is no perfectly known orbit available as a universal reference. POD quality is assessed through tracking-observation residuals, independent techniques such as satellite laser ranging, comparisons of independently generated orbit solutions and their overlapping arcs, and related consistency checks.

Preliminary Sentinel-6B results presented in 2026 illustrate the achievable standard: a JPL GPS-based solution reported **6.5 mm RMS satellite-laser-ranging residuals**, while a NASA GSFC solution reported **0.88 cm SLR fit** and **6.75 mm RMS radial differences** against POEG orbits [R1, R2]. Its onboard DORIS/DIODE real-time solution was reported at approximately **2.5 cm radial difference** relative to CNES precise orbits [R3]. These are different assessments and products, not interchangeable measurements of absolute three-dimensional position error. The source material consulted does not establish a universal six-week delivery latency for these figures.

Prediction is another matter. A 2025 study of **Sentinel-6A**, not Sentinel-6B, found best-case growth rates of **2.6, 5.0 and 4.0 cm/hour** in orbital user-range error for three different prediction methods [R4]. By contrast, the Space Telescope Science Institute warns that Hubble's position predicted two days ahead can be wrong by as much as **30 km**, depending in part on atmospheric drag and solar activity [R5]. The ISS is similarly affected by drag and planned or unplanned manoeuvres; NASA publishes updated trajectory data for operational planning [R6]. These examples are not a controlled comparison using the same error metric, data latency or forecast horizon. They illustrate why prediction quality depends so strongly on the satellite and the available orbit information.

**[FIGURE BRIEF 1 — From orbit to observation]** Draw an intelligible three-dimensional Earth–satellite–observer scene: satellite orbit, rotating Earth, a marked ground station, the station's local horizon, line of sight, azimuth and elevation. Add a compact sequence beneath it: orbital elements → SGP4 position → Earth orientation → observer-relative direction/range → pass events. Use labels large enough to read on an A4 page. This is an explanatory diagram, not a measured-data graph. Caption should emphasise that each step answers a different physical question.

## 1. Predicting the orbit: what SGP4 actually provides

### 1.1 Orbital elements are not a position

Satellite orbital data are stored and interchanged in a variety of formats In general terms they are referred to as a state vector with an associated epoch. One approach is to provide the 3 components of the vector defining its geocentric position, 3  components for the velocity and the corresponding time. After propagation with a suitable model, combining the laws of motion with modelling of external forces (both gravitational and non-gravitational), a sequential set of such state vectors form an orbit representation. The six pieces of information may be expressed in other ways, such as the classical keplerian elements.

Many widely available software aplications, including SatTracker, use a specific form of orbital element representation called the "two-line element" (TLE) or in more recent general perturbations (GP) formats. They contain an epoch and parameters describing an orbit in the particular mathematical framework of the SGP4 propagator. They are not a set of instantaneous Cartesian coordinates that can simply be advanced using the elementary two-body equations.

A purely Keplerian orbit would be an ellipse fixed in space, with the satellite moving according to the gravitational attraction of a spherical Earth; but real satellite motion is more complicated. The Earth is not spherical, its gravitational field is not uniform, solar and lunar gravity afeect the motion too. Non-gravitational forces include atmospheric drag and solar radiation pressure. The propagation tool SGP4 uses a particular collection of approximations and corrections designed to work with the mean orbital elements distributed in the GP/TLE system.

The crucial practical rule is therefore to use orbital elements with the propagator for which they were generated. Substituting a different orbit model (even a better one), or interpreting SGP4 mean elements as ordinary osculating Keplerian elements in such a model, will generally produce worse results.

SatTracker uses the complete original Vallado-derived C++ SGP4 implementation. It has been retained unchanged as the propagation baseline, with its gravity constants, operation mode and time conventions explicitly recorded. Other implementations were useful during development, but they are not automatically interchangeable. The early comparisons identified differences caused by such details as element parsing and resonance calculations. This is one reason the validation programme eventually selected a single, precisely identified propagation baseline rather than assembling a hybrid from several libraries.

### 1.2 The TEME reference frame

SGP4 produces a satellite position and velocity in a frame conventionally called **TEME**: true equator, mean equinox. For present purposes the important fact is that TEME is not a frame attached to the rotating Earth. A satellite vector in TEME cannot simply be subtracted from the latitude and longitude of an observer.

We must first transform the satellite's coordinates into a terrestrial frame, or transform the observer position in the opposite direction. Either approach can work, provided that the conventions and epochs are consistent. This apparently mundane requirement can be the source of satellite-tracking errors: the arithmetic might be flawless but the vectors being combined belong to different coordinate systems.

**[FIGURE BRIEF 2 — An orbit is not fixed to the ground]** Two related panels: (a) orbit plane and TEME-oriented axes; (b) Earth rotating beneath the orbit, with the same satellite position projected to different terrestrial longitudes as time advances. Avoid implying that TEME is a perfectly inertial modern celestial frame. A small side annotation should identify the time-dependent coordinate transformation.

### 1.3 How was the propagation checked?

The programme used the original Vallado test cases (established as a reference set to validate SGP4 implementations), retained raw position and velocity outputs, and comparisons with independently maintained implementations, including Orekit and other SGP4 ports. The experiments deliberately included ordinary near-circular low-Earth orbits, resonant and geosynchronous cases, and highly eccentric stress cases. Independent reviewers examined the calculations and reproduced the archived outputs.

An important qualification is that many SGP4 implementations share the same Vallado/NORAD mathematical ancestry. Agreement between them is evidence of implementation consistency, not agreement between wholly independent physical theories. Nor were the tests a comparison with satellite laser ranging, precise orbit determination or other observations of actual spacecraft. The tests tell us a great deal about whether the selected algorithm is being used correctly; they do not establish a universal prediction error for every satellite or every age of orbital elements.

This is also why the age and quality of the orbital data matter. An impeccably implemented propagator cannot recover information that is absent from an outdated or poorly determined orbit.

**[FIGURE BRIEF 3 — Representative orbit and validation cases]** Main panel: a physically intelligible set of schematic orbital geometries—near-circular LEO, inclined/resonant orbit, GEO and highly eccentric orbit—drawn to illustrate *regimes*, not to scale. Accompany with a small table identifying the archived Vallado test-case numbers and why each was chosen. Avoid making the extreme stress orbit look typical. If a numerical propagation-comparison figure is included, use an actual retained data series with a clearly named comparison and units; do not revive the early historical SatelliteKit outlier distribution as the main story.

## 2. Time: the hidden coordinate

A satellite position is meaningless without its time. Unfortunately, there is more than one useful definition of time in astronomy and geodesy.

**UTC** is the civil time scale used for dates and clock displays. It incorporates leap seconds. **TAI** is an atomic time scale whose seconds progress uniformly. **TT** is related to TAI by the fixed offset of 32.184 seconds and is useful in astronomical calculations. **UT1** follows the actual rotation of the Earth; it is not identical to UTC, and the difference must be obtained from Earth-orientation measurements.

The distinctions are not academic. A positive leap second gives the UTC sequence 23:59:59, 23:59:60, 00:00:00. Those labels are separated by one SI second each. A software system that treats every UTC day as precisely 86,400 elapsed seconds will mishandle such a boundary.

SatTracker's native time foundation therefore represents instants with a split, uniform-time representation and treats UTC as a calendar label. It also preserves the distinction between the time used for Earth rotation and the elapsed minutes supplied to SGP4. In the retained validation, independently calculated TT and UT1 conversions agreed within picoseconds and femtoseconds respectively. These are numerical checks of time conversion, not claims about the precision of the Earth-orientation observations.

There remains a narrowly defined policy question about how SGP4 elapsed minutes should be interpreted across an inserted leap second. The experimental implementation records those searches as conditional rather than silently asserting that one convention is uniquely correct. For ordinary search intervals not crossing a leap second, this ambiguity does not arise.

**[FIGURE BRIEF 4 — Four clocks for one event]** A horizontal timeline through the 2016 leap second with aligned UTC, TAI and elapsed SI-second labels, plus a separate annotation explaining UT1 as Earth-rotation time and TT=TAI+32.184 s. Use a real step diagram, not a cumulative residual plot. If showing a numerical comparison, add a small annotation: “tested time-conversion arithmetic: picosecond-scale or better”, with the exact scope and figures verified from B1.

## 3. From the orbit to the rotating Earth

### 3.1 The Earth does not rotate like a perfect clockwork sphere

If the Earth rotated uniformly around an absolutely fixed axis, transforming a satellite's coordinates would be straightforward: rotate the vector by an angle proportional to time. In reality the rotation rate varies slightly, and the rotation axis moves relative to the solid Earth. The orientation must be obtained from measurements and predictions published by the International Earth Rotation and Reference Systems Service (IERS), including products distributed by the United States Naval Observatory (USNO).

Two quantities are particularly important here. The difference UT1−UTC (or its leap-continuous counterpart UT1−TAI) tells us the Earth's rotation angle at the required instant. The polar-motion coordinates, conventionally denoted \(x_p\) and \(y_p\), describe the displacement of the terrestrial pole. These are small angles, but a small angular error becomes a displacement when applied to a position vector thousands of kilometres long.

The experimental native transformation follows a classical, explicit route:

\[
\mathrm{TEME}\;\xrightarrow{\mathrm{GMST82\ using\ UT1}}\;\mathrm{PEF}
\;\xrightarrow{\mathrm{polar\ motion}}\;\mathrm{ITRF\!\text{-}oriented}.
\]

PEF is a pseudo-Earth-fixed intermediate frame. The last label is deliberately **ITRF-oriented**, not a claim that all the modern corrections and station-coordinate realisation transformations of a full ITRF2014 or ITRF2020 chain have been implemented. An ITRF realisation also concerns the coordinates, epochs and possible motion of actual stations; simply rotating a vector does not confer survey-quality coordinates on an observer.

### 3.2 How well does the transformation agree with its reference?

This is one of the strongest numerical results in the programme. Against an independently evaluated version of the **same classical transformation**, the largest satellite-position difference was **0.162 millimetres** across the principal matched comparison set. That is less than the thickness of a typical sheet of card. The largest corresponding velocity difference was about **46 nanometres per second**.

Those figures are remarkable as evidence that the mathematical transformation has been implemented consistently. They must not be read as a claim that we know a satellite's true position to a fraction of a millimetre.

A comparison against Orekit's **modern ITRF2014 transformation** answers a different question. There the largest position difference in the retained set was about **9.16 metres**, with a maximum velocity difference around **1.5 millimetres per second**. This is not a nine-metre numerical blunder in the classical implementation: the two transformations do not embody identical conventions or corrections. The precise attribution of every term has not been completed, and the effect on a particular observer's azimuth/elevation depends on the geometry.

This distinction between *implementing a model correctly* and *choosing the best model for a purpose* is central to scientific software validation.

**[FIGURE BRIEF 5 — Two different questions about accuracy]** A deliberately simple, two-panel visual with physical units. Left: maximum matched-classical position disagreement, **0.162 mm**, drawn on a millimetre scale; right: maximum direct-versus-modern-model difference, **9.16 m**, drawn on a metre scale. Label both panels conspicuously: “same equations, different implementations” versus “different reference-frame models”. Do not put them on a common scale or call both “errors”. A small secondary table may give N and velocity comparisons. The illustration must not imply physical orbit accuracy.

### 3.3 Why velocity requires extra care

A rotation changes not only a position vector but also the relationship between velocities in the two frames. If \(\mathbf r_A\) and \(\mathbf v_A\) are position and velocity in one frame, and \(M(t)\) is the time-dependent rotation into another, then

\[
\mathbf r_B=M\mathbf r_A,
\qquad
\mathbf v_B=M\mathbf v_A+\dot M\mathbf r_A.
\]

The second term is essential. It describes the contribution from the rotating frame itself. Merely applying the same static rotation to position and velocity would omit it.

Four definitions of the rotation rate were examined, including nominal Earth-spin rates, rates adjusted by length-of-day data and a derivative calculated consistently with the selected interpolated transformation. The derivative-based version agreed best with controlled finite differences of the same mathematical model. That does **not** prove it is physically superior in every application; the choice of operational rate convention remains a separate modelling decision.

### 3.4 Earth-orientation data must remain valid

The retained USNO file contained 20,038 populated Earth-orientation records and 50 trailing rows without the required values. The software explicitly distinguishes observed, rapid and predicted records, and refuses to replace missing values with zero or extrapolate outside supported interpolation intervals. This is more important than it may sound: a plausible-looking number generated from unavailable data is often more dangerous than an explicit “unavailable” result.

The validated interpolation uses four consecutive support nodes and treats UT1−TAI as continuous across leap seconds. Its matched-polynomial arithmetic residuals are extraordinarily small—polar-coordinate comparisons of order \(10^{-22}\) radians in the retained tests—but that precision describes the numerical interpolation calculation, **not** the accuracy of predicted polar motion.

**[FIGURE BRIEF 6 — Earth-orientation data in practice]** Upper panel: a simple time axis showing final/rapid/predicted EOP data, an interpolation window and an unsupported gap or blank tail. Lower panel: a magnified, schematic example of four-node interpolation of UT1−TAI, showing why the leap discontinuity belongs in UTC labels rather than the interpolated quantity. Include a visible distinction between data *quality* and numerical *calculation precision*. Use actual archived date coverage for any dated elements, verified against the EOP file.

## 4. From terrestrial coordinates to what the observer sees

### 4.1 Where exactly is the observer?

A ground station is usually specified by latitude, longitude and height. Internally, SatTracker uses geodetic coordinates referred to the WGS84 ellipsoid (locations in the Ground Station database use geographic latitude and ellipsoidal height, rather than height above sea-level). Geodetic latitude is defined relative to the normal to the reference ellipsoid, not to a line drawn from the Earth's centre. That difference matters when constructing the observer's position and the local directions of east, north and up.

The coordinate values themselves may be much less accurate than the mathematics used to process them. A station surveyed with space-geodetic techniques may have a documented terrestrial reference-frame realisation, coordinate epoch and velocity. An antenna position estimated from a map or satellite image may be uncertain by metres. Both can be represented mathematically, but it would be misleading to assign them the same geodetic pedigree.

### 4.2 Azimuth, elevation and range

Once satellite and observer are expressed in the same frame at the same instant, their relative position is

\[
\boldsymbol\rho=\mathbf r_{\rm sat}-\mathbf r_{\rm obs},
\qquad R=\lVert\boldsymbol\rho\rVert.
\]

Here \(R\) is the slant range, not the satellite's altitude above the Earth's surface. Projecting \(\boldsymbol\rho\) onto the observer's local east, north and up directions gives the familiar azimuth and elevation angles.

This simple geometry has some instructive singularities. A satellite directly overhead has a perfectly meaningful range and elevation, but no uniquely defined azimuth: every horizontal direction is equally valid. At the geographic poles, the choice of local east and north depends on a conventional meridian. If satellite and observer positions coincide, range-direction quantities are undefined. A reliable implementation must report these conditions rather than manufacture an apparently precise angle.

### 4.3 How precise is the geometry calculation?

The independent observer-geometry calculations agreed exceptionally closely. In the integrated tests, the maximum difference in computed slant range was about **30 nanometres**—roughly one-thirtieth of a micron. The maximum difference in range rate was approximately **2.7 picometres per second**. These results compare two implementations working from the supplied states. They show that the geometry is internally consistent to a level vastly finer than the accuracy of ordinary satellite orbit data or station coordinates.

Eight comparisons disagreed on the exact classification of azimuth validity at a one-micrometre horizontal-separation boundary. That is a numerical edge condition near a direction that is already poorly defined, not an ordinary pointing-angle discrepancy.

**[FIGURE BRIEF 7 — The view from a ground station]** Main explanatory drawing: ellipsoidal Earth section, geodetic latitude and local vertical, observer, satellite, slant range and local horizon; a small plan-view inset for azimuth. Alongside, a simple physical-unit comparison: “largest independently calculated range disagreement: 30 nm” and “example station-coordinate uncertainty: metres (illustrative, not measured in this study)”. The graphic must visually distinguish numerical precision from input-coordinate uncertainty, without asserting a specific accuracy for all stations.

### 4.4 Range rate and Doppler

The rate of change of range follows from the relative velocity:

\[
\dot R=\frac{\boldsymbol\rho\cdot(\mathbf v_{\rm sat}-\mathbf v_{\rm obs})}{R}.
\]

Taking positive \(\dot R\) to mean increasing range, the first-order one-way Doppler shift at transmitted frequency \(f\) is

\[
\Delta f\simeq-\frac{f\dot R}{c}.
\]

A satellite approaching the observer has negative range rate and a positive received-frequency shift under this convention. The validated geometry and Doppler arithmetic agree with independent calculations to better than \(10^{-10}\) hertz in the tested integrated cases. But a perfect Doppler calculation from an imperfect satellite velocity still inherits that velocity's limitations. The next section explains an unexpected and important aspect of the SGP4 velocity itself.

## 5. An interesting subtlety: SGP4 position and velocity

A reader might reasonably expect the velocity returned by an orbit propagator to be exactly the time derivative of the position it returns. The SGP4 investigations showed that this is not, in general, true for the complete evolving SGP4 position expression.

The reported velocity is reconstructed using approximate analytical expressions. The position, meanwhile, contains a sequence of secular, long-period, short-period and—where applicable—deep-space corrections. Differentiating the *whole implemented position expression* introduces additional terms. The difference is a property of this mathematical reconstruction, not evidence that the Swift observer geometry is wrong.

To test this, the investigators evaluated the original C++ propagator at closely spaced times and formed a five-point numerical derivative of the position. They also independently differentiated the executed mathematical branches. Agreement between those two approaches established that the discrepancy was not simply an artefact of choosing a poor finite-difference interval.

For an ordinary near-circular LEO test case, the largest difference was about **0.040 metres per second**—four centimetres per second. For a deliberately extreme, highly eccentric historical stress case, a supplementary valid state reached **424 metres per second**. These figures should not be lumped together as a “typical SGP4 velocity error”: they describe different orbital regimes, and they are differences between two quantities derived from the same model, not measured deviations from a spacecraft's true velocity.

What might four centimetres per second mean to a radio observer? If its entire vector difference happened to project along the line of sight, it would correspond to no more than approximately **0.019 Hz at 145 MHz**, **0.058 Hz at 435 MHz**, or **1.11 Hz at 8.4 GHz**. These are sensitivity envelopes for the particular LEO test result, not demonstrated Doppler prediction errors. Actual line-of-sight projections can be smaller, and the much larger high-eccentricity stress-case differences require separate interpretation.

The practical consequence for pass prediction is straightforward. We determine rise, set and maximum elevation by evaluating the **position** and the resulting elevation curve, rather than assuming the returned velocity is an exact tangent to that curve. We do not silently replace the original SGP4 velocity with a numerical derivative: which quantity best represents observed motion is a different scientific question.

**[FIGURE BRIEF 8 — Why SGP4's two velocities differ]** Left: a conceptual curve r(t) with the tangent inferred from neighbouring positions, contrasted with the returned approximate analytical velocity; do not imply a specific 2D orbit-plane direction without supporting data. Right: a genuine-data plot with ordinary LEO residuals on a centimetres-per-second scale and the extreme high-e case on a separate metres-per-second scale. Show orbital-phase/radius dependence only where archived values support it. Caption: “Internal mathematical consistency, not observed velocity accuracy.” Add a small, clearly labelled Doppler-sensitivity strip for 145/435 MHz.

## 6. Predicting a pass rather than a position

### 6.1 What do AOS and LOS mean?

For a ground observer, a geometrical satellite pass is most naturally defined by its elevation above a chosen threshold. The threshold may be the ideal local horizon (0°) or an elevated angle such as 5° or 10° to exclude the lowest part of the sky. Define

\[
 g(t)=e(t)-e_{\rm threshold}.
\]

When \(g(t)\) changes from negative to positive, the satellite has an acquisition-of-signal (AOS) *geometric crossing*; the reverse is loss of signal (LOS). The historical names are familiar in satellite tracking, but a geometric AOS is not proof that a radio receiver has acquired a signal. Nor does it guarantee that an optical satellite is illuminated or visible.

A pass-search algorithm must discover crossings and then refine their times. It must also find maximum elevation, including cases where the maximum is not halfway between AOS and LOS. A search interval may start with the satellite already visible or end before it sets; those are clipped intervals, not opportunities to invent an AOS or LOS at the search boundary. A geosynchronous satellite can remain above the threshold for the whole search window. That does not mean it will remain visible forever.

**[FIGURE BRIEF 9 — Anatomy of a pass]** A readable actual elevation-versus-time curve for the archived near-circular LEO reference pass at 44° N, 7° E. Show 0° threshold, AOS, peak elevation and LOS, and, optionally, a 5° line explicitly labelled as **not reached** (the peak is below 5°). Do not show 5° crossings for this pass. For the 0° test, the archived example gives AOS 20:19:18.3696 UTC, peak 20:21:56.6421, LOS 20:24:34.5574 on 25 June 2006; maximum elevation about 2.965°. Make clear this is a historical test case, not a forecast.

### 6.2 Finding short and grazing passes

A naive search samples elevation every few minutes and looks for a sign change. It can miss a short pass entirely if the satellite rises and sets between two samples. The experimental SatTracker algorithm therefore refines its sampling, searches for local extrema and brackets crossings before applying numerical root refinement.

No finite set of samples can guarantee discovery of an arbitrarily narrow feature in an arbitrary smooth function. To demonstrate this, the validation included a deliberately artificial Gaussian elevation curve with a positive interval only **1.665 milliseconds** long. Both the main search and an independent dense sampler missed it. The analytic solution proves that a mathematical completeness guarantee was not established.

But that artificial interval is not an SGP4 satellite pass, and it tells us little about whether the algorithm would miss an opportunity that matters to an observer. A further investigation therefore constructed short visibility intervals from **actual SGP4 trajectories and physically specified ground stations**.

Across 153 orbital searches, the independent reference identified **310 complete visibility intervals and 14 clipped intervals**. The SatTracker algorithm found every one. The tests included **45 trials lasting 5–10 seconds**, all detected. Those 45 trials are 15 distinct orbital/station/threshold configurations each repeated at three different sampling alignments—not 45 independent satellites or a statistical population survey. The evidence is strong within this deliberately bounded test domain, but it is not a universal theorem.

The short-pass geometry itself is illuminating. One constructed 0° horizon-to-horizon pass lasted about **7.50 seconds**, but its peak rose only **0.00143°** above the horizon. That is about **5.1 arcseconds**. Another 7.5-second interval above a 10° threshold was part of a horizon-to-horizon pass lasting approximately **489 seconds**. A short interval above a selected threshold is therefore not necessarily a short overall satellite pass.

For visual observing, a near-horizon event may be obscured by terrain, atmosphere or low brightness. For passive beacon reception, a short interval might be useful under suitable circumstances. A two-way TTC contact introduces acquisition, synchronisation, command/response and equipment-dependent timing. These applications cannot be assigned one universal minimum usable duration.

**[FIGURE BRIEF 10 — Duration is not the whole story]** Two side-by-side actual-data pass curves: (a) the constructed 7.496 s horizon-to-horizon grazing LEO pass with peak only 0.001426° above 0°, shown with a carefully expanded vertical scale and a prominent arcsecond annotation; (b) the constructed 7.501 s interval above 10° embedded within its approximately 488.6 s horizon-to-horizon parent pass. Use two time scales or an inset so the difference is immediately obvious. The figure's purpose is to explain *elevation margin* and threshold selection, not merely plot 310 points on log axes.

### 6.3 How accurately are crossing times calculated?

For the ordinary fixed-threshold cases in the supplementary investigation, the largest difference between SatTracker and the independent position-based reference was about **0.28 microseconds** for AOS or LOS. This is the agreement of two numerical event searches using the same declared orbit and terrestrial geometry; it is not a claim that an actual satellite rises within a third of a microsecond of the prediction.

The specially constructed shallow cases are harder. One geosynchronous AOS comparison differed by **2.121 milliseconds**, slightly exceeding the preselected 2 ms numerical criterion. Independent root searches showed that tiny floating-point differences in elevation can shift a crossing time by milliseconds when the elevation curve is almost tangent to the threshold. The discrepancy remains recorded; it was not removed by changing the criterion.

The independent reviewers also checked that no previously found AOS, LOS or interval disappeared when the event-reporting corrections were introduced. Across the complete 196-search regression set—including mathematical controls and overlapping earlier cases—**672 crossings and 359 intervals** were preserved exactly.

### 6.4 When is maximum elevation reached?

For an ordinary LEO pass, the elevation maximum is usually a well-defined peak. For a satellite that moves very slowly in the observer's sky, especially near a geosynchronous maximum, the curve may be extremely flat. A minute angular difference can then move the calculated peak time appreciably while barely changing the maximum elevation itself.

The corrected reporting method separates the *value* of the maximum elevation from the confidence we can place in its *time*. It also distinguishes a boundary maximum from a truly flat or non-unique one. Twelve spurious isolated-contact labels in shallow GEO test cases were corrected without altering any of their actual passes.

Before that correction, 18 of 81 targeted maximum-time comparisons exceeded the 20 ms numerical criterion; afterwards, none did. Six range-at-maximum comparisons previously exceeded 1 metre; afterwards, none did. However, **18 target maximum times remain explicitly marked as poorly conditioned**, even though their selected times now agree with the reference within the numerical criterion. Better agreement does not magically make a nearly flat maximum physically sharp.

**[FIGURE BRIEF 11 — The elusive GEO maximum]** A two-level illustration based on the retained B3b shallow GEO samples. Top: the very shallow elevation curve and positive-duration visibility interval. Bottom: a greatly magnified peak, with original selected time, corrected selected time, independent reference and an *empirical numerical conditioning span*. The caption must explain that the interval is neither a certified bound nor an orbital prediction error bar. Prefer annotated actual angles and milliseconds over scientific-notation-only axes.

### 6.5 The scale of the numerical validation

The investigations tested much more than a handful of representative passes. The propagation corpus contained **33 distinct inputs** evaluated at **2,348 requested epochs** in its principal schedule; valid and invalid propagation results were retained separately. The observer-geometry programme contained **2,799 requests**, including **1,440 valid pure-geometry cases**, **1,260 integrated cases**, three analytical anchors and 96 deliberately undefined zero-range cases. The native terrestrial transformation was tested at **263 requests**, of which 248 were supported by the available data; alternative rotation-rate models were evaluated on overlapping inputs.

The position–velocity investigation examined **318 labelled derivative centres** in its principal set, with seven numerical step sizes, and kept a separate extreme-eccentricity supplement. The practical pass-search study comprised **153 searches** yielding **310 complete intervals, 14 clipped intervals and 626 crossings**. In the subsequent **196-configuration** regression, **359 intervals and 672 crossings** remained unchanged after event-reporting corrections. Some searches are repeated at different sampling alignments, and the final regression includes the earlier searches: these numbers must not be added as if they were independent observations.

An audit-selected ledger contains **12,707 individual numerical criterion outcomes**, of which **413 exceeded their respective criteria**. This is useful as an indication of the volume of recorded comparisons, but it is *not* a complete count of all calculations or a meaningful overall pass rate. For example, 327 of those exceedances deliberately test the known difference between SGP4's returned velocity and the derivative of its position curve; others concern strict representation-level diagnostics. Cross-model comparisons and many additional checks are outside this selected ledger. The accompanying plots will therefore show the distributions, test populations and exceptional cases separately, rather than combine them into a single score.

**[FIGURE BRIEF — Validation populations and numerical distributions]** Add a compact table of the populations above and physical-unit plots from the retained evidence: propagation state differences by orbital regime and elapsed time; observer range, range-rate and Doppler residual distributions; ordinary and targeted AOS/LOS timing differences; and complete-pass duration against elevation margin. Identify valid N, comparator, repeated families, exclusions and remaining exceedances. Use the approved `UserReportFigurePlan.md` as the plotting specification; no new propagation or root solving is authorised by this editorial text.

## 7. What confidence should we place in the results?

The validation programme provides several complementary forms of evidence. Matched implementation comparisons test whether the time, terrestrial-frame and observer calculations execute their declared mathematical conventions consistently. Cross-model comparisons show the consequences of using different conventions or approximations. The practical pass searches test whether geometrically visible intervals and events are found within a defined set of real SGP4 trajectories and constructed edge cases. None of these populations is an observation campaign establishing a single physical pointing or timing accuracy for every satellite.

The strongest results are accompanied by recorded exceptions. The strict B1 GMST comparison exceeded its original position criterion in **64 of 72** cases and its velocity criterion in **21 of 72**; high-precision checks strongly support a time-representation explanation, but the original comparison limits remain exceeded. A separate reference orientation/rate finite-difference diagnostic also exceeded its criterion in **72 of 72** cases, with its full attribution still incomplete. These diagnostics are distinct from the successful matched B2a classical transformation comparisons.

For pass prediction, an original Orekit cross-model shallow-GEO maximum-time comparison differed by **6.825 seconds**. The corrected matched-curve reporting tests subsequently removed all 18 targeted maximum-time criterion exceedances, but this does not retrospectively resolve that separate cross-model result. One shallow-GEO AOS comparison still differs by **2.121 milliseconds**, slightly above its **2 ms** numerical criterion. Of the final 359 intervals, **28 maxima** remain classified as poorly conditioned, three as non-unique and three as boundary maxima. These classifications are useful information, not simply failures to print enough decimal places.

A deliberately artificial Gaussian elevation interval lasting **1.665 milliseconds** was missed by both search methods. This shows that unrestricted mathematical completeness was not proved. In contrast, every independently confirmed interval in the bounded practical SGP4 test set was found. The difference matters: the practical result is substantial, while the mathematical limitation is real.

For actual use, the suitability of a prediction also depends on the orbit-data age and provenance, the ground-station coordinates, Earth-orientation data, atmospheric and optical conditions, and the characteristics of the antenna, receiver or telescope. A geometric horizon crossing does not by itself guarantee optical visibility or radio acquisition.

**[FIGURE BRIEF — Numerical evidence and remaining qualifications]** Show the principal comparison populations with a concise limitations table; retain the GMST and reference-rate exceedances, the 6.825 s cross-model GEO diagnostic, the 2.121 ms AOS exceedance, the Gaussian missed interval and the shallow-maximum classifications. A separate synthesis diagram may distinguish implementation agreement, model choice and actual observing performance, without adding incompatible quantities into a single error budget.

## 8. Conclusions

Predicting the passage of a satellite across the sky requires considerably more than calculating its position in orbit. It involves a chain of calculations linking orbital motion, time, the orientation of the Earth and the observer's location, ultimately producing the directions, distances, velocities and event times needed for satellite tracking.

SatTracker's scientific foundation has been subjected to an extensive programme of numerical validation. The investigations covered orbital conditions ranging from ordinary low-Earth orbits to geosynchronous and highly eccentric trajectories, together with deliberately constructed cases designed to expose weaknesses in the algorithms.

The results demonstrate very close agreement with independent calculations across the principal time, coordinate-transformation and observer-geometry tests. The practical pass-prediction investigations identified every independently confirmed visibility interval in the tested satellite trajectories, including grazing opportunities lasting only a few seconds. The report also documents specific numerical exceedances and situations in which an event time is inherently sensitive to very small changes in the calculated elevation.

The investigations revealed several less obvious characteristics of satellite prediction. In particular, the velocity returned by SGP4 does not always correspond exactly to the derivative of its calculated position, and the time of maximum elevation can become poorly defined when a satellite's apparent motion is extremely slow. Understanding such behaviour helps users interpret calculated results appropriately.

The resulting scientific foundation has a documented chain of calculations, reproducible numerical comparisons and explicitly characterised limitations. These are essential qualities in a tool used to predict and follow objects moving across the sky.

---

## References

**Real-world orbit determination and prediction**

- **[R1]** Conrad, A., Desai, S. & Haines, B. (2026). *GPS-Based Precise Orbit Determination of the Sentinel-6B Mission*. Ocean Surface Topography Science Team (OSTST) Meeting. [Meeting abstract](https://ostst.aviso.altimetry.fr/programs/abstracts-details.html?cHash=cceeb540f7c27ff871deee738f0f2f82&tx_ausyclsseminar_pi2%5Baction%5D=show&tx_ausyclsseminar_pi2%5Bcontroller%5D=Abstracte&tx_ausyclsseminar_pi2%5BobjAbstracte%5D=4494).
- **[R2]** Lemoine, F. et al. (2026). *First POD results for Sentinel-6B & Update on the std2400 orbits*. OSTST Meeting. [Meeting abstract](https://ostst.aviso.altimetry.fr/programs/abstracts-details.html?cHash=de21016157efc6677fc991091d857691&tx_ausyclsseminar_pi2%5Baction%5D=show&tx_ausyclsseminar_pi2%5Bcontroller%5D=Abstracte&tx_ausyclsseminar_pi2%5BobjAbstracte%5D=4675).
- **[R3]** Chauveau, J.-P., Didelot, F. & Dumond, O. (2026). *DORIS/DIODE Real-Time orbit determination onboard Sentinel-6B*. International DORIS Service Workshop. DOI: [10.24400/312072/i03-2026.4593](https://doi.org/10.24400/312072/i03-2026.4593).
- **[R4]** Liu, J. et al. (2025). *Accuracy Degradation Rate of LEO Satellite Predicted Orbits Using Different POD Methods*. Proceedings of the Institute of Navigation International Technical Meeting, pp. 646–657. DOI: [10.33012/2025.20020](https://doi.org/10.33012/2025.20020).
- **[R5]** Space Telescope Science Institute. *Hubble Space Telescope Primer, Orbital Constraints: Predicted HST Position*. [STScI documentation](https://hst-docs.stsci.edu/hsp/past-hst-proposal-opportunities/the-hubble-space-telescope-primer-for-cycle-26/hst-cycle-26-primer-orbital-constraints).
- **[R6]** NASA. *ISS Trajectory Data*. Operational ephemeris and trajectory-data service (reference to be checked against the current NASA page before publication).
- **[R7]** Calliess, D., Montenbruck, O., Wermuth, M. & Reichinger, H. (2024). *Long-term analysis of Sentinel-6A orbit determination: Insights from three years of flight data*. *Advances in Space Research*, 74, 3011–3027. DOI: [10.1016/j.asr.2024.06.043](https://doi.org/10.1016/j.asr.2024.06.043). Demonstrates POD assessment with independent receiver solutions, orbit comparisons and SLR residuals.

**Scientific algorithms and reference systems**

- Hoots, F. R. & Roehrich, R. L. (1980). *Spacetrack Report No. 3*.
- Vallado, D. A., Crawford, P., Hujsak, R. & Kelso, T. S. (2006). *Revisiting Spacetrack Report No. 3*. AIAA 2006-6753.
- Petit, G. & Luzum, B. (eds.) (2010). *IERS Conventions (2010)*. IERS Technical Note 36.
- Orekit and Hipparchus: pinned scientific reference implementations used in the archived numerical investigations, not required SatTracker runtime dependencies.

## Editorial and scientific notes for the subsequent production stage

*This section is not part of the published reader-facing report.*

### Numerical evidence to verify before typesetting

| Claim in manuscript | Retained source | Check required |
|---|---|---|
| Classical frame maximum 0.162 mm; velocity ~46 nm/s | B2a results; V1 pp. 13–15 | Exact comparison model, N=248, units and maxima |
| Modern frame difference up to 9.16 m and ~1.5 mm/s | B2a results; V1 p. 15 | Different model, not error against physical truth |
| Observer range arithmetic ~30 nm and range-rate ~2.7 pm/s | B2 results; V1 p. 17 | N=1260 integrated and signed conventions |
| B1 TT/UT1 time residuals | B1 results; V1 p. 10 | Do not confuse computed time-scale residuals with measured UT1 uncertainty |
| SGP4 velocity ~0.040 m/s ordinary LEO, 424 m/s extreme stress | B2b results; V1 pp. 6–9 | Distinguish principal and supplementary samples |
| Doppler sensitivity at 145/435 MHz/8.4 GHz | B2b results; V1 p. 9 | Vector envelope, not line-of-sight measurement |
| 310 complete, 14 clipped, 45 5–10 s trials, 626 crossings | B3a results; V1 pp. 20, 23 | Repeated alignments, 153 searches |
| 7.496 s horizon pass with 0.001426° peak; 10° case with ~489 s parent | B3a results; V1 p. 20 | Use actual threshold/station fixtures |
| Ordinary AOS/LOS max ~0.28 µs; shallow GEO 2.121 ms | B3a/B3b results; V1 pp. 19, 22 | Different populations; retained failure |
| 196 searches, 359 intervals, 672 crossings preserved | B3b results; V1 p. 22 | Includes scalar/negative controls; overlapping stages |
| 18 maximum-time failures corrected; 18 still poorly conditioned | B3b results; V1 pp. 21–22 | Criteria agreement versus timing uncertainty |
| EOP 20,038 populated rows and 50 blank tail | B2a results; V1 p. 11 | Archived snapshot, not live availability |

### Figure-production rules

All numbered figure briefs in the manuscript are proposals. For every data-driven plot, use the retained raw data and preserve its population and units. Do not invent smooth curves or exact sample positions. Conceptual diagrams may be drawn schematically but must be labelled as such. Where a proposed illustration requires data not already archived (for example a second elevation threshold on a representative pass), either calculate it with the unchanged validated scientific core as a separately identified illustration-only result, or omit that feature pending approval. No new result should silently become part of the frozen scientific validation population.

Use real units on primary axes and explanatory annotations. Scientific notation is acceptable where necessary, but the caption should interpret the scale in words. Avoid defaulting to cumulative distributions of residual/criterion ratios. Figures should complement the prose, not force readers to decipher a legend before understanding the point.

Keep references, stage identifiers, detailed frozen acceptance criteria, source hashes and reproduction procedures in a separate technical companion or appendix. In the public-facing report, cite scientific sources and refer to a concise validation-methods note. Do not call the reader “owner”, “developer” or “reviewer”. Do not state that the production application has passed tests performed only on the scientific prototype.

### Suggested scientific references

- Hoots, F. R. & Roehrich, R. L. (1980), *Spacetrack Report No. 3*.
- Vallado, D. A., Crawford, P., Hujsak, R. & Kelso, T. S. (2006), *Revisiting Spacetrack Report No. 3*, AIAA 2006-6753 (revised publication).
- IERS Conventions (2010), Technical Note 36, and the applicable IERS/USNO EOP documentation.
- Orekit scientific reference implementation, retained pinned validation version (not a shipped SatTracker dependency).
- The archived B1–B3b numerical results, conventions and independent QA reports (technical validation companion).
