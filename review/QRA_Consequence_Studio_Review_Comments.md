# QRA Consequence Studio: Consolidated Review Comments

| Item | Detail |
|---|---|
| Document reviewed | `index.html` (single-file "Risk2Safe QRA Consequence Studio"; generates the "QRA Consequence Assessment" report), commit `1b07c16` |
| Review type | Independent technical review: lead adjudication of three specialist reviews plus lead comments |
| Lead reviewer | Principal process-safety engineer: flare system design (API 521 / ISO 23251), thermal radiation, QRA consequence analysis |
| Specialist inputs | Fire and thermal radiation (IDs `F-xx`); Source, dispersion, VCE and vulnerability (`D-`, `S-`, `E-`, `V-`, `W-`, `DB-xx`); Report integrity, QA and security (`QA-xx`) |
| Status | Comments issued for response. No changes were made to the code under review. |

---

## 1. Overall verdict

**NOT FIT FOR USE in its current state, whether for screening, training or software cross-checking. Reject for issue.**

The engine has good bones:
- The discharge equations, Briggs sigmas, the B&M continuous table, the API 521 point-source equation, the TNO fireball correlations and the probit constants were all verified correct.
- The code is a pure-function engine that can be tested in Node.

However, six Critical defects put silently wrong numbers into the deliverable on **mainstream** cases, and most of them are non-conservative:

1. The **default project** (propane, 10 mm G.3 leak, "Jet fire" selected) reports a **20 t BLEVE fireball** (12.5 kW/m² at 281 m) that the user never selected. With "Pool" selected, it prints pool-fire distances under the BLEVE heading.
2. The **pool-fire view factor** is not the Mudan/SFPE formula. The lead verified this with an independent quadrature: it overstates F by 2.4× at L/R = 3 and by 6× at L/R = 10.
3. **Dense-gas toxic distances** are computed at a clamped 2000 ppm (continuous) or 10 000 ppm (instantaneous). For chlorine, IDLH, ERPG-2 and ERPG-3 all print as **163 m**, against a passive-plume bound of about 2.8 km to 20 ppm.
4. **G.1 (instantaneous)** ignores the inventory. 20 t and 200 t give the same 28 m LFL, which is smaller than the 10 mm leak.
5. **G.2 / FBR / LEAK** are labelled but not implemented. G.2 is 22× under the required rate.
6. **Ammonia and CO crash** the Results tab and Print, through `fmt(x,0)` and `toExponential(-1)`.

### Comment count by severity (consolidated register, Section 4)

| Severity | Count | Definition used |
|---|---|---|
| **Critical** | 6 | A wrong or misattributed result reaches the deliverable silently on a mainstream case, typically order-of-magnitude and/or non-conservative, or the tool cannot run for common materials. |
| **Major** | 20 | A substantive method, implementation or traceability error that materially changes results or would cause a competent reviewer to reject the study. |
| **Minor** | 17 | A limited-impact inaccuracy, labelling, usability or robustness defect. |
| **Observation** | 8 | A method limitation or choice to document; no code change required. |
| **Total** | **51** | Merged from 79 specialist findings (fire 23, non-fire 24, QA 32), plus 3 new lead comments (C-42, C-43, C-50) and lead additions to 9 others. |

---

## 2. Fitness for stated purpose ("screening / training / software cross-checking")

| Purpose | Verdict now | Basis | After "must-fix" tier (Section 6) |
|---|---|---|---|
| **Screening / study scoping** | Not fit | The default case and the G.1, G.2 and dense-toxic cases give order-of-magnitude errors in both directions. A screening tool must at least be bounding or transparent, and this one is neither. | Fit for gas-jet (far-field) and neutral-gas scoping, with validity flags. Pool fire and VCE need the Major fixes. |
| **Training** | Not fit | It would teach a wrong view factor, wrong event-tree logic (BLEVE following a 10 mm leak), and that toxic endpoints of different severity have identical distances. | Good candidate: transparent equations and an inspectable engine, provided citations are corrected (C-24). |
| **Software cross-check** (vs PHAST, FLARESIM, Aspen Flare System Analyzer, EFFECTS) | Not fit | A cross-check tool needs a V&V record: golden cases against published worked examples and reference-software runs, with stated tolerances. None exists. The UI and AI prompt nevertheless call the engine "validated". Today, discrepancies against reference software would come mainly from this tool's own defects. | Only after a documented V&V suite (C-24) and the Major fire items (C-11 to C-17). |

The banner and disclaimer ("screening… not a substitute for a validated package") are appropriate. However, the contradictory "validated engine" statements (lines 1656, 1723, 1784) must be removed until a V&V record exists.

---

## 3. Adjudication of specialist Critical and Major findings

Method: the lead read every cited line in `index.html` at the current commit. Several reviewer line references differ by ±15 lines; the actual lines are used in Section 4. Key claims were re-run with the reviewers' harnesses (`fire/t1_vf.js`, `fire/t5_driver.js`, `fire/t6_jet_bleve.js`, `nonfire/driver_tests.js`). The `fmt` crash was reproduced in Node. The view factor was checked with the **lead's own independent quadrature** (`scratchpad/lead_vf.py`: differential target element to the visible lateral cylinder surface), which reproduced the SFPE values exactly: H/R = 2, L/R = 3 gives Fv/Fh/Fmax = 0.1322 / 0.0501 / 0.1414, against the code's 0.343.

| Source ID | Reviewer severity | Lead verdict | Final severity | Consolidated ID | Lead notes |
|---|---|---|---|---|---|
| F-01 | Critical | **Confirmed** | Critical | C-04 | Independently verified (above). The horizontal term contains no flame height, and the negative Fv is squared into Fmax. |
| F-02 | Critical | **Confirmed** | Critical | C-05 | Lines 1005-1031 confirm three things: BLEVE triggers on `phase==='liquid' && T>Tb`; `fluxFn = jet‖pool‖bleve` while the name uses the order jet→bleve→pool; "none" is ignored. The lead adds that a gas release with "Pool" selected produces a 10 m² pool fire. |
| F-03 | Major (Critical for LNG) | **Confirmed with modification** | Major | C-11 | Defect confirmed (SEP 30.9 kW/m² for a 20 m LNG pool). The upgrade to Critical is not accepted: the error is roughly 2× in the near field and partly masked by C-04. The quoted measured LNG SEP range (150-280 kW/m², Maplin/Montoir) should be **verified against the test reports** before it is used as a default. |
| F-04 | Major | **Confirmed** | Major | C-12 | Lines 1019 and 1021 use different Fs. `Psv` is never passed. Default 0.3 gives 37.5 kW/m² at 155 m, against 97-120 m with Roberts Fs. |
| F-05 | Major | **Confirmed**, expanded | Major | C-13 | The lead adds the API 521 point-source validity range, the multi-point/solid-flame requirement, and datum handling. |
| F-06 | Major | **Confirmed with modification** | **Minor** | C-27 | F is user-adjustable and shown in the results header. The error is conservative for CH₄/H₂ and about 20 % non-conservative for C₃/C₄. The citation error moves to C-24. The per-fuel F ranges (Kalghatgi/API 521) should be **verified against source**. |
| F-07 | Major | **Confirmed** | Major | C-15 | Only becomes visible once C-04 is fixed. |
| F-08 | Major | **Confirmed** | Major | C-14 | Line 1012: `s.phase==='gas'` only. |
| F-09 | Major | **Confirmed**, merged | Major | C-07 | The reviewers quote different "Purple Book minimum layer" values (1.5 cm vs 5 mm/10 mm). **Verify against CPR 18E** before coding. |
| F-10 | Major | **Confirmed**, expanded | Major | C-17 | The lead adds escape/reaction-time modelling. |
| F-11 | Major | **Confirmed with modification** | Major | C-16 | The tool's labels do not cite API 521, but 4.7/1.6 are evidently rounded API 521 values used without solar. Table number ("Table 9"), the solar footnote and the 0.79-1.04 kW/m² solar range should be **verified against the API 521 edition cited** (table numbering changed between editions). |
| F-12 | Major | **Confirmed with modification**, split | Major | C-23, C-24, C-18, C-37 | Each mis-citation claim (Burgess vs Babrauskas; YB Eq 6.12/6.19/6.66/6.21/6.22/6.59/6.90-6.96) must be **checked against the actual CPR 14E edition**, not corrected from memory. The internal inconsistency is confirmed in the code (lines 166-172 vs 538-545). |
| F-13 | Major | **Confirmed with modification** | **Minor** | C-28 | Calculator only. A separate, correctly labelled "Field radiation converter" with "Exposure (s)" exists (lines 1574-1578, 1609-1611). |
| D-1 | Critical | **Confirmed with modification** | Critical | C-03 | The code confirms `clampRatio` (1085) and the instantaneous re-clamp (439). The "17×" figure compares against a pure passive point-source plume, which is an *upper* bound. A correct B&M→passive hand-over will give less, but still kilometre-scale distances, so the error remains order-of-magnitude and non-conservative. QA-01's 157 m (vs 163 m) comes from different input T and is consistent. |
| D-2 | Critical | **Confirmed** | Critical | C-01 | Reproduced: V = 32.7 m³ used, against ≈7,300 m³ of flashed vapour. 20 t and 200 t give identical LFL. The VCE uses the inventory, so the two results in the same report are inconsistent. |
| D-3 | Major | **Confirmed with modification** | Major | C-08 | The reviewer's reference coefficients agree with the lead's recollection of the C&L/B&M instantaneous table, but must be **transcribed from the B&M (1988) workbook / C&L table**, not memory. Code segment 3 repeating segment 2 is confirmed. |
| D-4 | Major | **Confirmed** | Major | C-09 | Cliff edge reproduced: propane ρ = 1.803 vs threshold 1.806 kg/m³ at 25 °C. |
| S-1 | Major | **Confirmed** | Major | C-07 | Reproduced: refrigerated propane evaporates 8.39 kg/s from a 6.08 kg/s spill. |
| S-2 | Major | **Confirmed, upgraded** | **Critical** | C-02 | The UI note states "Constant rate = inventory / 600 s" for a mandated Purple Book LOC, yet the rate is 22× lower, silently. This is the same class of defect as D-2. |
| E-1 | Major | **Confirmed** | Major | C-19 | Reproduced: 5 mm and 10 mm holes give the same 60 kg (floor governs). Diesel gets a 3 t instantaneous VCE cloud. |
| E-2 | Major | **Confirmed** | Major | C-20 | Line 621 `return R0*lo`. |
| E-3 | Major | **Confirmed with modification** | Major | C-21 | Lead arithmetic: class 3 is 0.050 at R′ = 0.6⁻ and 6.05e-2·0.6^-0.99 = 0.100 at 0.6⁺. Class 9 is 5.0 → 4.10. The correct coefficients must be **taken from Alonso et al. (2006)**. |
| E-4 | Minor/Major | **Confirmed with modification** | Minor | C-29 | The inconsistency with the class-10 curve is confirmed. The "Eq 4.5/4.6" attribution and 1.5/3.0 coefficients could not be confirmed by the reviewer or the lead from memory. **Verify against CPR 18E §4.8.** |
| V-1 | Major | **Confirmed** | Major | C-22 | Lines 1634/1647: the dropdown filters `kind==='toxic'` (none exist) and saves `c.probit`, while the engine reads `toxKey`. |
| QA-01 | Critical | **Confirmed** | Critical | C-03 | Duplicate of D-1. |
| QA-02 | Critical | **Confirmed** | Critical | C-01 | Duplicate of D-2. |
| QA-03 | Critical | **Confirmed** | Critical | C-05 | Duplicate of F-02. |
| QA-04 | Critical | **Confirmed** | Critical | C-06 | Node: `fmt(150000,0)`, `fmt(1e5,0)` and `fmt(0.0005,0)` throw RangeError. Ammonia LFL is 150 000 ppm and CO is 125 000 ppm. |
| QA-05 | Critical | **Confirmed with modification** | (merged) Critical | C-06 | **The "silent" claim is partly rejected.** QA demonstrated it by calling `switchTab` programmatically. Through the real Run button (line 1253) the user does get an alert ("Calculation error: toExponential()…"). However, the failing result is persisted to localStorage, so the Results tab stays broken across reloads. Print fails silently (it throws at line 1792 before `window.open`). A stale DOM is displayed if render is triggered while results are on screen. |
| QA-06 | Major | **Confirmed** | Major | C-23 | |
| QA-07 | Major | **Confirmed** | Major | C-23 | Line 1794 prints TNO when present and falls back to TNT, so "both" drops TNT. |
| QA-08 | Major | **Confirmed** | Major | C-19 | Duplicate of E-1. The help text at line 1219 contradicts the code. |
| QA-09 | Major | **Confirmed** | Major | C-20 | Duplicate of E-2. |
| QA-10 | Major | **Confirmed**, folded in | Major (via C-25) | C-25 | Line 592 `table[cls]‖table[7]`. Treated as an input-validation defect. |
| QA-11 | Major | **Confirmed, upgraded** | Critical | C-02 | Duplicate of S-2. |
| QA-12 | Major | **Confirmed** | Major | C-07 | |
| QA-13 | Major | **Confirmed** | Major | C-25 | |
| QA-14 | Major | **Confirmed** | Major | C-12 | Duplicate of F-04. |
| QA-15 | Major | **Confirmed** | Major | C-09 | Duplicate of D-4. |
| QA-16 | Major | **Confirmed** | Critical (merged) | C-01 | Passive instantaneous release uses a continuous plume. |
| QA-17 | Major | **Confirmed with modification** | **Minor** | C-32 | Compare tab only; does not reach the printed report. |
| QA-18 | Major | **Confirmed** | Major | C-26 | Unescaped interpolations confirmed at lines 1155, 1616, 1642 and 1801. The entity-decoding bypass inside the `onclick` attribute is valid. |
| QA-19 | Major | **Confirmed** | Major | C-26 | Line 1822 exports `DB.settings.aiKey`. |
| QA-20 | Major | **Confirmed with modification** | Major (merged) | C-23 | **Sub-item rejected:** "Patm 101 325 vs TNO default 101 510 Pa" is a 0.2 % effect, and the attribution of 101 510 Pa to TNO is unverified. Remove it from the register. All other hidden defaults are confirmed. |
| QA-21 | Major | **Confirmed** | Major (merged) | C-18 | Merged into the lead's datum and receptor-geometry comment. |
| QA-22 | Major | **Confirmed** | Major | C-24 | |

**Rejected in substance:** none of the Critical or Major findings.

**Rejected or modified sub-claims:**
- QA-05 "no error shown" (Run-button path).
- QA-20 Patm sub-item.
- F-03 upgrade to Critical.
- The D-1 "17×" magnitude, restated as an order-of-magnitude bound.

**Downgraded:** F-06, F-13 and QA-17 to Minor.

**Upgraded:** S-2/QA-11 to Critical; D-5 from Minor to Major.

---

## 4. Consolidated comment register

Line numbers refer to `index.html` at commit `1b07c16`. Area order follows the calculation chain: Source → Dispersion → Fire → Explosion → Vulnerability → Report → Software.

### 4.1 Critical

| ID | Severity | Area | Location (lines) | Comment | Basis / Reference | Required action | Source reviewer IDs |
|---|---|---|---|---|---|---|---|
| C-01 | **Critical** | Source term / LOC G.1 | 986, 1001-1004, 433-441, 1037-1038 | Instantaneous (G.1) dispersion uses V = qo·min(dur, 60 s) from the **10 mm orifice rate**, so the inventory is ignored. A passive instantaneous release is modelled with a *continuous* Gaussian plume. The VCE in the same run uses the inventory, so the report is internally inconsistent. Propane G.1: 20 t and 200 t both give LFL 28.4 m (V = 32.7 m³ against ≈7,300 m³ of flashed vapour). That is smaller than the 10 mm G.3 leak (49.7 m). | Purple Book CPR 18E Ch.3 (G.1 = whole inventory instantaneously); B&M (1988) instantaneous V₀; Gaussian puff (YB CPR 14E Ch.4; C&L Ch.5) | For `instantaneous`: airborne mass = inventory × (flash + aerosol fraction), or the evaporated mass; V₀ = m/ρ_cloud; no orifice calculation and no 60 s cap. Add a Gaussian puff model for passive clouds. Report the basis. Add a regression test that LFL increases with inventory. | D-2, QA-02, QA-16 |
| C-02 | **Critical** | Source term / LOC G.2, FBR, LEAK | 781-787, 1255-1262, 922-958 | G.2 is labelled "Constant rate = inventory/600 s" but uses the orifice rate (propane 20 t: 1.50 kg/s against 33.3 kg/s, 22× under). FBR only sets Cd = 1: there is no pipe ID and no two-sided outflow. LEAK always sets 50 mm, not 10 % DN. G.1 keeps the previous hole. The LOC id is never printed. Severity upgraded: silent and non-conservative on mandated LOCs. | Purple Book CPR 18E Ch.3 LOC definitions; YB §2 pipe outflow | Implement each LOC in `runScenario`: G.2 ṁ = M/600; FBR with a pipe-ID input, 2 × single-sided, and a two-phase pipe model (HEM/Fauske or YB); LEAK = min(0.1·DN, 50 mm). Otherwise remove the presets. Print the LOC and its basis in the report. | S-2, QA-11 |
| C-03 | **Critical** | Dispersion (dense gas, toxic) | 1085, 986-987, 439, 393-416 | `clampRatio` forces Cm/C₀ ≥ 0.002 (continuous); the instantaneous path re-clamps to ≥ 0.01. Every ppm-level toxic endpoint is solved at 2000 or 10 000 ppm but labelled with its true ppm. Chlorine G.3 F1.5: IDLH = ERPG-2 = ERPG-3 = 163 m. A passive plume from the same source reaches 20 ppm at 2.76 km; that is an upper bound, and the correct hand-over result will be smaller but kilometre-scale. Error is order-of-magnitude and non-conservative on the land-use-planning driver chemical. | B&M (1988) workbook range (Cm/C₀ ≥ 0.002 continuous, ≥ 0.001 instantaneous); CCPS *Guidelines for Consequence Analysis* (1999) hand-over to passive dispersion | Never clamp. Run B&M to its lowest curve, then hand over to a passive Gaussian with a virtual-source offset (σ matched at the transition). Flag "outside correlation range". Print the computed concentration. Add a test that different ppm targets give different distances. | D-1, QA-01 |
| C-04 | **Critical** | Fire: pool view factor | 488-510, 516; Calculator 1593-1596 | `mudanViewFactor` is not the Mudan/SFPE closed form. The horizontal term has no flame-height dependence (F_h = 0.34 at L = 3R even for a 1 cm flame). The vertical term has the wrong structure, goes negative, and is then squared into Fmax. Far-field decay is ~1/L instead of ~1/L². The lead verified this independently: H/R = 2 gives code 0.343 vs 0.141 at L/R = 3, and 0.081 vs 0.0135 at L/R = 10. As a result, pool 1.6 kW/m² distances are 2-4× too long and 12.5 kW/m² about 70 % too long. This affects every pool result, the Calculator, and the report. | Mudan (1984); SFPE Handbook (Beyler), vertical cylinder to ground target; YB CPR 14E §6.5 view-factor tables | Replace with the SFPE expressions (reference implementation `scratchpad/fire/vf.js` `vfSFPE`). Guard S ≤ 1. Add regression tests against the SFPE table values and a numerical integral. Once fixed, re-check C-15 (tilt). | F-01 |
| C-05 | **Critical** | Fire: outcome selection and labelling | 1005-1006, 1012, 1017-1021, 1025, 1031, 1290, 1353-1355, 1807 | The user's fire type is not honoured. Any liquid with T > Tb triggers a **whole-inventory BLEVE**, even for a 10 mm continuous leak and even with "None". In the default project ("Jet fire" selected) the result is a Ø162 m fireball with 12.5 kW/m² at 281 m. `fluxFn` uses jet→pool→bleve but `fluxFnName` uses jet→bleve→pool, so with "Pool" selected, pool distances (5/13/36/103 m) are published under a BLEVE header and citation. Only one outcome is reported, and the others are discarded silently. Lead addition: a gas release with "Pool" selected produces a 10 m² pool fire (lines 1005-1006). | Purple Book CPR 18E Ch.4/6 event trees (BLEVE follows G.1 or fire engulfment, not a hole leak); CCPS CPQRA | Drive outcomes from an explicit event set; honour "None". Tie BLEVE to G.1 or an explicit engulfment flag. Compute and report **each** outcome in its own table with name, parameters and citation. Store `{name, fn}` together. Print "Fire event modelled: X (selected: Y)". Refuse physically inapplicable outcomes with a message. | F-02, QA-03 |
| C-06 | **Critical** | Software: formatting / state | 867-869; uses at 1271, 1331, 1792 etc.; 1253; 1790-1795 | `fmt(x,0)` calls `toExponential(-1)` for \|x\| ≥ 1e5 or < 1e-3 and throws a RangeError. **Ammonia (LFL 150 000 ppm) and CO (125 000 ppm) cannot be displayed or printed**, nor can any distance ≥ 100 km. Via the Run button the user gets a cryptic alert. The failing result is nevertheless persisted, so the Results tab stays broken across reloads. Print fails silently. A stale DOM from the previous scenario can remain displayed. | Software QA; IEC 61508-3 / good engineering practice for calculation tools | `toExponential(Math.max(0, d-1))` or a proper significant-figure formatter. Wrap render and print in try/catch with an explicit error panel. Do not persist a result that failed to render. Clear stale output at run start. Add unit tests over the full numeric range. | QA-04, QA-05 |

### 4.2 Major

| ID | Severity | Area | Location (lines) | Comment | Basis / Reference | Required action | Source reviewer IDs |
|---|---|---|---|---|---|---|---|
| C-07 | Major | Source term: pool size, evaporation, mass balance | 949-955, 280-298, 1005-1006 | The pool spreads to 1 cm over the whole duration, unbounded. There is no burning-equilibrium diameter: gasoline 10 mm gives D = 13.2 m against ≈6.3 m, and 100 mm gives 132 m against ≈62 m. Evaporation is not capped by the spill rate (refrigerated propane: 8.39 kg/s from 6.08 kg/s). The rain-out pool from flashing releases is dropped from both dispersion and fire (propane 31 %, NH₃ 67 %, Cl₂ 69 % of discharge). Flashing liquids get an arbitrary 10 m² pool fire. There is no inventory check (methanol 100 mm: 196 t released from 20 t, Ø177 m pool). Boiling pool uses fixed t = 60 s and Tg = 288 K. | YB CPR 14E Ch.3 (pool spreading/evaporation) and §6.5 (D_eq = √(4ṁ/πṁ″)); Purple Book minimum layer thickness (**verify value**; reviewers disagree); CCPS (1999) §2-3 | Release mass = min(ṁ·t, M); cap duration. D = min(bund, D_eq, minimum-layer D). Evaporation ≤ supply. Use `poolRate` for flashing liquids (evaporate and burn it). Tg = Tamb. Add a bund-area input. Print the D basis. | S-1, F-09, QA-12 |
| C-08 | Major | Dispersion: B&M instantaneous table | 386-391 | Only the 0.10 low-α branch is correct. The 0.05/0.02/0.01 constants and slopes differ. Segment 3 repeats segment 2 (no turnover). The 0.005/0.002/0.001 curves are missing. Distance errors are ×0.44 to ×1.4. | B&M (1988) workbook / C&L instantaneous table (**transcribe from source**; the reviewer's coefficients were quoted from memory) | Replace with the full 7-curve table. Add unit tests against tabulated β. Depends on C-01 for a meaningful V₀. | D-3 |
| C-09 | Major | Dispersion: dense/passive regime | 962-964, 428-429, 437-441 | The regime switch is `ρ_gas(Tamb) > 1.5·1.204`. The factor 1.5 has no basis, and ρ_air is fixed at 20 °C while ambient defaults to 9 °C. Cold or aerosol clouds (LNG, flashing NH₃) are never dense. The B&M criterion is computed but unused, and its continuous form is wrong. There is a cliff edge: propane D5 gives 49.9 m at 24 °C and 19.2 m at 25 °C. | B&M (1988) criteria: continuous (g₀q₀/u³D)^⅓ ≥ 0.15 with D = (q₀/u)^½; instantaneous (g₀V₀^⅓)^½/u ≥ 0.2; non-isothermal correction | Compute source density for the actual release (flash vapour, aerosol, entrained air; or at least the vapour at Tb for flashing/refrigerated cases). Select the model with the B&M criteria. Use ρ_air(Tamb). Disclose the regime. | D-4, QA-15 |
| C-10 | Major (upgraded) | Dispersion: averaging time, near-field validity, flash fire | 304-368, 788-789, 998 | Briggs/P-G sigmas (~10 min) are used for LFL, while the Purple Book flammable averaging (20 s; `avgFlammable` defined but unused) is ignored, under-predicting LFL by about 40 % (propane D5: 19.2 m against 27.1 m). Briggs is applied below 100 m, and jet-momentum entrainment is ignored. Lead addition: the flash-fire lethality zone (100 % inside LFL, per Purple Book) needs a **cloud footprint** (width and area), but only a centre-line distance is given. Upgraded because flash fire is a lethality endpoint and the error is non-conservative. | Purple Book CPR 18E §4 (averaging 20 s flammable / 600 s toxic); Briggs (1973) validity 100 m-10 km; YB Ch.4 free-jet | Apply σ_y ∝ t^0.2 per endpoint. Warn when x < 100 m. Consider a free-jet near-field model for high-momentum gas releases. Report LFL cloud width and footprint for flash fire. | D-5 (+ lead) |
| C-11 | Major | Fire: pool SEP | 465-467; DB 821-849 | The Mudan smoky-fuel SEP is applied to every fuel, and Hc is unused. A 20 m LNG pool gets 30.9 kW/m², against a measured order of 150-280 kW/m² (**verify against the Maplin Sands / Montoir reports**). The 37.5 kW/m² contour vanishes, and 12.5 kW/m² is about 2× short once C-04 is fixed. Methanol and H₂ also need non-sooty treatment. There is no SEP override in the UI (`o.sep` exists in the engine). | YB CPR 14E §6.5; CCPS (1999) §3.4; Mudan & Croce (sooty fuels only) | Add a per-material SEP model key (sooty / luminous / non-luminous, χ_r-based). Expose an SEP override. Print the SEP basis. | F-03 |
| C-12 | Major | Fire: BLEVE radiant fraction | 1019, 1021, 543-545, 1598 | The SEP shown uses `radFrac<0.25?0.3:radFrac`, but the distances hard-code Fs = 0.3. User values below 0.25 are silently replaced. The Psv-based Fs branch is unreachable because `Psv` is never passed, although `s.P` is available. For 50 t propane, 37.5 kW/m² is at 155 m against 97-120 m with Roberts Fs (0.23-0.26). | Roberts (1982) / CCPS: Fs = 0.27·P^0.32 (P in MPa); YB form 0.00325·Psv^0.32 (Pa), cap 0.4 | Use one Fs for display and distances. Default Psv = Psat(T) or P_abs. Allow an override without silent replacement. Show the Fs used and its source. Use separate radFrac inputs for jet and BLEVE. | F-04, QA-14 |
| C-13 | Major | Fire: jet geometry and API 521 point-source validity | 521-530, 1012-1014, 1353, 1418-1430 | The point source sits at the **release point**: there is no flame length, lift-off, orientation, source elevation or wind tilt. For 10 kg/s CH₄ (L ≈ 48 m), the 37.5 and 12.5 kW/m² endpoints lie inside the flame. A horizontal jet toward the receptor is under-predicted 1.8-2.8×; a vertical jet is over-predicted at grade. Lead: the API 521 point-source approach measures D from the **flame centre** and is only valid in the far field, beyond about one flame length (**verify the exact validity statement in API 521 Annex C**). The near field needs a multi-point source (5-10 points along the distorted flame axis with apportioned power) or a solid-flame model. | API 521 / ISO 23251 (point source D = √(τFQ/4πK); flame length vs Q; flame distortion by U∞/Uj); Chamberlain (1987) / YB §6.5 frustum; Johnson et al. (1994) horizontal jets | Compute flame length (expanded-jet basis for choked flow) and flame-centre location with orientation and wind distortion. Add release-height and orientation inputs. Offer a multi-point or solid-flame option. Flag receptors within ~1 flame length ("near field / impingement"). Report flame length and datum. | F-05 (+ lead) |
| C-14 | Major | Fire: two-phase jet fire | 1012 | There is no jet fire for liquid or flashing releases, which is the dominant LPG and NH₃ continuous fire outcome. It is silently replaced by BLEVE or pool (see C-05). | Purple Book CPR 18E Ch.4; YB §6.5 two-phase jet | Allow jet fire for `two-phase` on the total or airborne rate with a two-phase F or SEP basis. Warn when a requested outcome can't be computed. | F-08 |
| C-15 | Major | Fire: pool flame tilt and drag | 462-468, 503-516 | The flame is an upright cylinder in all winds. After the C-04 fix, the downwind 12.5 kW/m² distance is understated by 45-79 % (LNG 20 m at 5 m/s: 25.4 m vs 14.2 m). There is no downwind vs crosswind distinction. | YB CPR 14E §6.5 (tilt and drag); CCPS (1999) §3.4; Welker-Sliepcevich | Implement a sheared-cylinder view factor (tables or numerical, e.g. `scratchpad/fire/vf.js` `vfTilt`). Report downwind and crosswind distances separately. | F-07 |
| C-16 | Major | Fire: thermal criteria basis (API 521 / Purple Book) | 746-751, 1342, 1401-1406, 1423 | The endpoints 37.5/12.5/4.7/1.6 kW/m² have no stated source. The values 4.7 and 1.6 are evidently API 521's 4.73/1.58, but they are applied to **fire-only** flux, whereas API 521 personnel levels include solar radiation (typically about 0.8-1.0 kW/m²; **verify the table number and footnote in the API 521 edition cited**). The 6.31 and 9.46 kW/m² levels (escape / short emergency action) are missing. The Purple Book 35 kW/m² boundary is quoted in text but never calculated. For 10 kg/s CH₄, the API 521 1.58 kW/m² level (with 0.8-1.0 solar) lies at 101-117 m, against 71 m from the tool (τ = 1). | API 521 / ISO 23251 personnel radiation table; Purple Book CPR 18E §5; HSE TDU criteria | Provide selectable, labelled criteria sets: API 521 (with a solar-flux input, subtracted from the target in API mode only, never added in probit or Purple Book modes); Purple Book 35/10 kW/m²; HSE 1000/500 TDU; World Bank/IFC. Label each endpoint with its source and exposure basis. | F-11 (+ lead) |
| C-17 | Major | Fire: vulnerability, exposure time and escape | 655-675, 702, 746-751, 1025-1029, 1401-1406 | Thermal probits are never applied to scenario results. There is no exposure time, escape model, or Purple Book 20 s / ×0.14 clothing / 35 kW/m² rules. Labels mix exposure bases: "100 % lethality 1 min" is meaningless for an 11-14 s fireball. The BLEVE dose-based 1 %/50 % lethality distances (387/238 m for 50 t propane) are not reported. Lead: API 521-style personnel assessment needs a reaction time (typically 3-5 s) plus escape at about 2.5 m/s (**verify values in the API 521 annex**), with dose integrated as flux decreases along the escape path. | Green Book CPR 16E (Pr = −36.38 + 2.56 ln(t·q^4/3)); Purple Book CPR 18E §5.3; API 521 personnel exposure guidance | Add exposure-time and escape inputs (default per the selected criteria set). For BLEVE, use the fireball duration. Compute and report 1 %/50 %/99 % lethality distances. Apply the Purple Book rules in Purple Book mode. State the exposure basis on each endpoint. | F-10, QA-27 (+ lead) |
| C-18 | Major (lead) | Fire / plot: distance datum and receptor geometry | 512-516 (pool: from centre), 522-530 (jet: from release), 545-547 (BLEVE: ground zero), 1418-1430, 1487-1513 | Each model uses a different distance datum, and none is stated in the results, report or plot. Pool distances run from the pool centre (many regulators use the pool edge), jet distances from the release point (API 521 uses the flame centre), and BLEVE distances from the ground point below the fireball. Only ground-level receptors are supported: there is no receptor height for platforms, flare decks or elevated structures, and no source height (`releaseHeight` is unused). The plot draws downwind dispersion and directional jet contours as full circles around the source; the wind arrow is decorative. | API 521 (flame-centre datum); YB §6.5; CCPS CPQRA documentation | State the datum for every result row. Report distances from a common origin (the release point), with flame or pool geometry stated. Add receptor and source height inputs. Draw dispersion as downwind plumes or ellipses and jets as directional, or label "max distance, any direction". | F-12 (datum), F-20, QA-21, D-6 (+ lead) |
| C-19 | Major | Explosion: VCE cloud mass | 1036-1041, 1219 | Cloud mass = airborne × min(dur, 30 s) × fraction × 3, floored at 2 % × inventory × fraction. None of these factors has a basis. The floor governs the default case (60 kg), and 5 mm and 10 mm holes give the same cloud. Diesel at 25 °C gets a 3 t instantaneous VCE. In Purple Book mode, `vceFraction` and `f_obstr` are both applied (double reduction). The help text says "mass within LFL contour", which is false. | Purple Book CPR 18E §4.8 / YB Ch.5: E = flammable mass (LFL-UFL, from dispersion) in the obstructed region × ΔHc | Derive flammable mass from the dispersion result (or ṁ·x_LFL/u as a screening value). Remove the ×3, the 30 s cap and the floor. Apply only f_obstr in Purple Book mode. Suppress VCE for non-volatile releases. Disclose the basis in the report. | E-1, QA-08 |
| C-20 | Major | Explosion: TNO ME distance solver | 616-630 | When the target overpressure exceeds the class plateau, the solver returns R₀·0.23 instead of "not reached". Class 3 reports 20.7/13.8/6.9 kPa all at 6.6 m. `f(hi)>0` returns R₀·100 silently. | TNO Yellow Book Ch.5 ME curves | Return null with the reason "not reached for class N". Flag the capped upper bound. Render it explicitly. | E-2, QA-09 |
| C-21 | Major | Explosion: TNO ME curve-fit table | 566-577, 578-589 | The fitted curves are discontinuous at segment joins: class 3 is ×2.0 at R′ = 0.6 (0.050 → 0.100), class 4 is non-monotonic (0.100 → 0.128), and class 9 drops 5.0 → 4.10. The class-3 far-field coefficient equals the impulse-table entry, which suggests a transcription error. | Alonso et al. (2006) ME curve fits (**re-transcribe from source**); YB Ch.5 charts are continuous and monotonic | Re-transcribe. Add unit tests asserting continuity within 5 % and monotonicity. | E-3 |
| C-22 | Major | Vulnerability: toxic lethality | 1383-1396, 1634, 1647, 1604 | Toxic probits are evaluated only at fixed concentrations (IDLH, ERPG-3, and an arbitrary 3×ERPG-3), with exposure = release duration, also for instantaneous releases. No lethality distance is computed. In the chemical editor the probit dropdown filters `PROBITS` for `kind==='toxic'` (none exist) and saves `c.probit` while the engine reads `c.toxKey`, so **a new or edited material can never receive a toxic probit**. | Purple Book CPR 18E §5 (probit along the centre line, indoor/outdoor); Green Book CPR 16E | Fix the editor to write `toxKey` from `TOXPROBITS`. Compute the C(t) profile along the centre line and solve Pr = 2.67/3.72/5 for the 1 %/10 %/50 % lethality distances. Use cloud passage time for instantaneous releases. Add the Purple Book indoor/outdoor rules. | V-1, V-4 (part) |
| C-23 | Major | Report: content and assumptions register | 1790-1815 | The printed report lacks: a full input echo (Cd, duration, LOC, fire type selected vs modelled, F/Fs, VCE method, class, f_obstr, cloud mass and energy); model parameters (pool D, H, SEP; jet Q, F, L; fireball D, t, H, Fs; B&M regime); distance datum; exposure basis; threshold sources and versions; multi-weather results; plot; references; revision and approval block; and software version. In "TNO + TNT" mode only TNO is printed. Hidden defaults are not disclosed: H = 0; 10-min sigmas; 1 cm spreading; 60 s boiling pool; 1.5 ρ_air; 2× rain-out rule; rain-out loss; 3×ERPG-3; toxic exposure = duration. | CCPS CPQRA (documentation requirements); Purple Book reporting; typical client QRA deliverable checklists | Regenerate the report from the same objects as the Results tab. Sections: basis and scope; inputs; properties used; per-outcome parameters; results with datum; criteria; assumptions register (auto-generated); limitations; references; revision block; engine version and hash; plot. Print TNO and TNT side by side. | QA-06, QA-07, QA-20, F-12 (part) |
| C-24 | Major | Report: citations, claims and V&V | 158-178, 194-230, 449-460, 532-552, 1198, 1206, 1656, 1723, 1743-1786, 1813 | Citations are internally inconsistent. Equation numbering differs between the header and code comments (for example, BLEVE Fs as "Eq 6.21" vs Eq 6.90-6.96). The burn rate is attributed to "Burgess", but the coded form is Babrauskas. "Fs Eq 6.59" is cited but not implemented. The Methods tab says BLEVE uses "point-source" while the code uses a solid sphere, and it omits the Purple Book simplified ME (the default). The reference list omits the Purple Book, Green Book, B&M, Alonso, Bagster, Mudan, API 521 and Briggs. The UI banner (line 1656) and the AI prompt (line 1723) claim a "validated engine", but there is no V&V record and no regression suite. | CPR 14E/16E/18E as published; QA/V&V practice for calculation software | Build one reference list from `CITES` (author, year, edition) and print it in the report. **Verify every equation number against the actual edition**; don't correct from memory. Remove "validated" until a V&V dossier exists. Create a regression suite: the hand-check cases from all three reviews, published worked examples, and reference-software comparisons with tolerances. | F-12 (part), QA-22 (+ lead) |
| C-25 | Major | Software: input validation | 1243-1244, 592, 922-958 | There is no input validation. Wind = 0 gives NaN distances that silently drop out. An empty hole field gives NaN, but BLEVE and VCE are still reported. Negative P gives a negative mass flow. Cd = 5, hole 1e6 mm and inventory 0 are accepted (the last gives "12.5 kW/m² at 0.5 m"). TNO class 6.5 or 11 silently runs as class 7 while labelled 11. Phase plausibility is unchecked (diesel as "gas"), and so is stability/wind pairing (F at 9 m/s). | Good engineering practice for calculation tools | Add a central validator with physical ranges (wind ≥ 0.5-1 m/s, 0 < Cd ≤ 1, integer ME class 1-10, positive geometry and inventory, P vs Psat, phase vs Tb, P-G class vs wind). Block the run on errors. Echo warnings in the report. | QA-13, QA-10, QA-28 |
| C-26 | Major | Software: security and data handling | 1155, 1616, 1642, 1801, 1822, 1830, 1662, 1732-1736 | XSS was confirmed by execution: the chemical key in `onclick`; the name in `confirm()`, where `esc()` is defeated by attribute entity decoding; unescaped `c.react`; and unescaped report inputs. A shared study `.json` can therefore run script in the app origin, where localStorage holds the **plaintext API key**. Export writes `aiKey` into the shareable file. Import does no schema validation. Lead addition: the AI layer sends scenario data straight from the browser to a third-party API; this data flow is not disclosed to the user. | OWASP XSS prevention; data-protection practice | Escape every interpolation and replace inline handlers with `addEventListener`. Validate keys (`/^[a-z0-9_]+$/`) and the import schema. Build the report with the DOM or `textContent`. Add a CSP. Strip `settings` from export. Disclose the AI data flow. | QA-18, QA-19, QA-23 (part) (+ lead) |

### 4.3 Minor

| ID | Severity | Area | Location (lines) | Comment | Basis / Reference | Required action | Source reviewer IDs |
|---|---|---|---|---|---|---|---|
| C-27 | Minor (downgraded) | Fire: jet radiant fraction | 523-525, 881, 1204 | F is fixed at 0.20 by default regardless of fuel or exit velocity. The cited Chamberlain Fs(u_j) is not implemented. Effect: CH₄ +9-20 % (conservative), H₂ up to about 2× (conservative), C₃/C₄ gas about 20 % short (non-conservative). One shared input serves both jet and BLEVE. | Chamberlain (1987) Fs = 0.21e^(−0.00323u_j) + 0.11; API 521 / Kalghatgi fuel F (**verify ranges**) | Use Chamberlain as the default, with per-fuel defaults and an override. Show the F used. | F-06, F-22 |
| C-28 | Minor (downgraded) | Calculator: probit time unit | 1567-1571, 1603-1608 | The thermal probit takes its exposure from the field labelled "Time (min, toxic)" but uses it as seconds. The toxic branch is dead code. A correct, separate "Field radiation converter (Exposure (s))" exists. | n/a | Add a dedicated "Exposure (s)" field for thermal. Route toxic through `toxicProbit`. | F-13 |
| C-29 | Minor | Explosion: Purple Book simplified ME | 665-681 | R(0.3 bar) = 1.5 and R(0.1 bar) = 3.0 × (f·E/Pa)^⅓ disagree with the engine's own class-10 curve (1.33/2.82, +13 %/+6 %). The "Eq 4.5/4.6" attribution is unverified. | CPR 18E §4.8 (**verify**) | Use `tnoDistanceToOverpressure({cls:10})` with f_obstr·E, or cite the exact source. State the centre-of-obstructed-region datum. | E-4 |
| C-30 | Minor | Explosion: TNT basis | 633-664 | The Kinney-Graham free-air curve is labelled "surface burst". Ground-level distances are under-predicted by 2^⅓ = 1.26×. The near- and far-field bias of TNT for VCEs is not noted. | Kinney & Graham (1985); Kingery-Bulmash surface burst | Use W_eff = 2W (hemispherical) or KB surface fits. Note the TNT-for-VCE limitations. | E-5 |
| C-31 | Minor | Explosion: endpoint labels | 752-757 | "2.1 kPa: 90 % glass breakage" should be about 10 % (Clancey "safe distance"). "13.8 kPa: repairable damage" understates it: 2 psi is partial collapse of walls and roofs. | Clancey (1972) / CCPS CPQRA damage table | Relabel with source. | E-6 |
| C-32 | Minor (downgraded) | Compare tab | 1295-1309, 1437-1468 | The rows mix 30 kPa (Purple Book) with 20.7 kPa (TNO), and 10 with 6.9 kPa. The "0.1 barg / 6.9 kPa" label is wrong. TNT-only runs show "—". | n/a | Store the threshold with each snapshot, keep per-threshold rows, and include TNT. | QA-17 |
| C-33 | Minor | Source term: property and discharge consistency | 832-856, 937-947 | ρ_liq tabulated at Tb is used at storage T (propane +8.6 % ṁ). There is no P ≥ Psat(T) check; the default propane case at 8 barg / 25 °C is at or below saturation (**verify Psat**). The liquid model (ΔP = P₁ − P_a) is used regardless of hole length. | Rackett ρ(T); YB Ch.2 discharge model selection | Use ρ(T). Validate P vs Psat. Select the discharge model by L/D and add a two-phase pipe model. | S-3 |
| C-34 | Minor | Fire: characteristic wind velocity | 462-464 | u* uses ρ_air; YB/CCPS use the vapour density ρ_v. Flame length is +2-8 %, and tilt will be affected once implemented. | YB §6.5; CCPS (1999) | Use ρ_v at Tb. | F-14 |
| C-35 | Minor | Calculator: pool | 1547-1551, 1593-1596 | ṁ″∞ = 0.055 and kβ = 0.7 are hard-coded (the gasoline DB value is 2.1). The ΔHc input has no effect. The calculator inherits C-04. | n/a | Add a material selector. Use DB values. Use or drop ΔHc. | F-15 |
| C-36 | Minor | Fire: falsy defaults | 516, 527 | `RH \|\| 0.5` turns RH = 0 (desert) into 50 %: τ at 50 m becomes 0.775 instead of 1.0. Same pattern for `ambientT`. | n/a | Use `??` or explicit undefined checks. | F-16 |
| C-37 | Minor | Solvers and "—" semantics | 1086-1093, 348-368, 1331, 1356, 1369 | `solveDistance` returns the 20 km bound as a real distance, and `gaussianPlumeDistance` caps at 100 km (the comment says 50 km). Null (never reached) is not distinguished from "inside source". The bisection assumes monotonic decay, which will fail for elevated or tilted sources. "—" can mean any of five different conditions. | n/a | Return a status flag. Scan for the maximum before bisecting. Render explicit text ("not reached: max X kW/m²", "within flame", "> solver limit"). | F-17, QA-25, QA-26, F-12 (part) |
| C-38 | Minor | Data: chemical DB and probit sets | 831-857, 689-705 | Values to **verify against current sources** (the reviewers quoted them from memory): NH₃ ERPG-3 750 ppm (possibly revised by AIHA); VCM ERPG-2/3; H₂S LFL 4.3 % vs 4.0 %; every Purple Book probit constant against the CPR 18E table. The "gb" set copies the Purple Book values for 13 of 15 chemicals and is not an independent Green Book set. Several probit chemicals have no DB entry. There is no fallback for toxic chemicals without a probit. | AIHA ERPG/WEEL Handbook (current year); NIOSH IDLH; CPR 16E/18E | Verify and cite each value with its source and year. Label the "gb" set honestly. Add the missing DB entries. | DB-1, V-2 |
| C-39 | Minor | Vulnerability: Purple Book bands | 738-743, 1400-1407 | `overpressureLethality` is never called. It uses ">" where the Purple Book uses "≥". The band table omits flash fire indoors (0). | Purple Book CPR 18E §5 | Apply the bands to the computed contours and complete the table. | V-3 |
| C-40 | Minor | Report HTML and locale | 1790-1815, 868-869 | No doctype or charset (quirks mode). `window.open` is not null-checked. Section numbering skips. The date is in UTC. Numbers use the browser locale (de-DE prints 1.497 kg/s as "1,497"). | n/a | Add doctype and charset, a null check and fixed numbering. Use a fixed locale or state it. | QA-24, QA-29 |
| C-41 | Minor | State: live references and staleness | 926, 1790-1801 | `out.chem` is a live reference, so editing a material changes old results. There is no "results out of date" indicator. The report prints current metadata beside old results. | n/a | Deep-copy at run time. Add a stale flag. Snapshot the metadata. | QA-30, QA-31 |
| C-42 | Minor (lead) | Fire: duration and inventory-limited fires | 1012-1021, 746 | Jet and pool fire duration (inventory/rate, burn-out) is never computed, and jet blowdown decay is ignored (conservative for G.3 gas). The "37.5 kW/m² equipment damage" label presumes sustained exposure (**verify the duration basis**). Dose-based results (C-17) need a fire duration. | YB §6.5; API 521 (flare/jet duration context) | Compute and report fire duration. Cap dose time at fire duration. Optionally add a time-averaged or blowdown rate. | lead |
| C-43 | Minor (lead) | Scope: flare and vent radiation | 521-530, 155, 1743-1786 | The jet module is labelled "API 521 point-source", but the tool has no flare capability: no stack height, grade-level radiation profile, sterile-area radius, wind distortion (ΔX/ΔY vs U∞/Uj), or Brzustowski-Sommer / multi-point methods. Users may mistake it for a flare radiation check, or use it to cross-check FLARESIM or Aspen Flare System Analyzer. | API 521 / ISO 23251 flare radiation methods | State in the scope that flares and vents are excluded. If a flare cross-check is intended, implement the API 521 flare method with solar, receptor elevation and sterile-area output. | lead |

### 4.4 Observations

| ID | Severity | Area | Location (lines) | Comment | Basis / Reference | Required action | Source reviewer IDs |
|---|---|---|---|---|---|---|---|
| C-44 | Obs | Fire: transmissivity | 472-485 | Bagster is coded correctly. It is valid for 10⁴ < p_w·x < 10⁵ Pa·m (about 10-105 m at the Purple Book defaults), extrapolated beyond that, and clamped to [0.4, 1]. | Bagster (1989); YB | State the validity range. Optionally use the Wayne (1991) H₂O + CO₂ form for long BLEVE paths. | F-18 |
| C-45 | Obs | Fire: thermal probit basis | 666-669 | The Tsao & Perry / Green Book probit is correctly coded but about 2.3× more severe in dose than Eisenberg (1 % at 422 vs 958 TDU). UK-profile users may expect HSE dangerous-dose distances. | Green Book CPR 16E; Eisenberg (1975); HSE | Label the probit and offer an HSE TDU option. | F-19 |
| C-46 | Obs | Fire: fireball geometry / BLEVE scope | 535-552 | Centre height 2·r_fb (YB) vs 0.75 D (CCPS), and no lift-off or growth phase; both are acceptable screening choices. Lead: BLEVE blast overpressure and fragment hazards are not addressed. | YB; CCPS BLEVE guidelines | State the choices and the exclusions in the report. | F-21 (+ lead) |
| C-47 | Obs | Source term: flash and rain-out rule | 264-276 | The isentropic flash is valid. The 2× aerosol rule is a CCPS/Lees heuristic, and the "YB Eq 2.173" citation is to be verified. | CCPS; Lees | Document it as a heuristic. | S-4 |
| C-48 | Obs | Source term: gas discharge | 231-249 | The equations are exact. The rate is the constant initial rate (conservative for G.3 gas; no Purple Book time-averaging). The engine default Cd is 0.85 against 0.62 in the driver. | Purple Book time averaging | Document. Align the defaults. | S-5 |
| C-49 | Obs | Code hygiene | 326, 788-789, Source card cite | Dead `sy` expression; unused `releaseHeight`, `avgFlammable` and `avgToxic`; rural/urban only (no z₀); the source card always cites "flash". | n/a | Clean up or implement. | D-6, QA-32 |
| C-50 | Obs (lead) | Fire: view-factor orientation | 488-510 | The maximum (worst-orientation) view factor is used for every receptor. This is appropriate and conservative for personnel, but equipment and wall assessments need orientation-specific F. | SFPE; YB §6.5 | Disclose. Optionally offer a vertical/horizontal target selection. | lead |
| C-51 | Obs | Fire: still-air flame length | 462-466 | At u = 0 the code uses the wind form with u* = 1, giving −5 % vs Thomas still-air. This **conforms to the YB u* ≥ 1 clamp**. It is not a defect and needs no action. | YB §6.5 | None. | F-23 |

---

## 5. Lead reviewer's thermal-radiation and flare commentary (new points)

1. **API 521 treatment.** The jet module reproduces √(τFQ/4πK) exactly (verified to rounding), but that is the only part of API 521 implemented. API 521 practice also requires:
   - (a) a flame length from the heat-release correlation;
   - (b) flame distortion by the wind/jet velocity ratio;
   - (c) a point source at the flame centre, with the distance D measured from there;
   - (d) design levels that **include solar radiation**, so the solar flux is added to the flare/fire flux before comparing (or subtracted from the allowable);
   - (e) personnel exposure criteria tied to reaction time and escape.

   None of (a) to (e) is implemented (C-13, C-16, C-17). Until they are, the module must not be described as "API 521".
2. **Personnel exposure and escape.** Fixed-flux contours without an exposure time cannot support conclusions on personnel safety. Recommended approach:
   - reaction time, typically 3-5 s;
   - escape at about 2.5 m/s radially away from the source (both values to be verified in the API 521 annex for the edition cited);
   - dose ∫q^(4/3)dt along the escape path;
   - the result compared with the selected probit or dangerous dose.

   For fireballs, the exposure is the fireball duration with no escape credit, consistent with the Purple Book.
3. **Sterile area / grade-level profile.** There is no receptor-elevation input and no radiation-vs-distance profile output. A profile plot q(x) at grade and at user-defined elevations is the standard deliverable for flare and jet-fire siting, and it is easy to add from the existing flux functions.
4. **Point-source vs multi-point vs solid-flame.**
   - The single point source is acceptable only in the far field (beyond about one flame length from the flame centre) and for F estimated consistently with the fuel and velocity.
   - In the near field, which is where 37.5 and 12.5 kW/m² usually fall for jets, use a multi-point source (5-10 points along the distorted axis) or a solid-flame/frustum SEP model (Chamberlain/YB; Johnson et al. for horizontal jets).
   - The tool should compute the flame length, flag near-field receptors, and switch method or warn.
   - For pools, the solid-flame approach is the right choice, but it needs the correct view factor (C-04), a fuel-aware SEP (C-11) and tilt (C-15).
5. **Jet-fire distance datum.** Distances are measured from the release point, while API 521 measures from the flame centre. For a horizontal 10 kg/s CH₄ jet toward the receptor, this alone under-predicts by up to 2.8×. Every thermal result must state its datum, and a common origin should be used across models (C-18).
6. **Event logic is the larger risk.** The worst thermal error in practice is not a correlation coefficient but C-05: a BLEVE assigned to a 10 mm leak. Outcome selection must follow an explicit, documented event tree.

---

## 6. Prioritised fix plan

### Tier 1: must fix before **any** use (including training and internal screening)

| Order | IDs | Rationale |
|---|---|---|
| 1 | C-06 | Crash on ammonia and CO; a simple fix that unblocks testing. |
| 2 | C-05, C-14 | Explicit outcome selection; no unsolicited BLEVE; correct labelling; two-phase jet. |
| 3 | C-04 | Correct pool view factor (drop-in SFPE code available). |
| 4 | C-03 | Remove the dense-gas clamp; B&M → passive hand-over. |
| 5 | C-01, C-02 | Inventory-based G.1 with a puff model; implement G.2/FBR/LEAK or remove them. |
| 6 | C-12, C-20 | One-line class fixes (single Fs; "not reached" for ME). |
| 7 | C-25 (minimum set), C-26 | Block invalid inputs; fix XSS and key export before any study file is shared. |
| 8 | C-24 (claims part) | Remove "validated" statements now. |

### Tier 2: must fix before **issue** of any report or use as a software cross-check

- **Source and dispersion:** C-07, C-08, C-09, C-10
- **Fire:** C-11, C-13, C-15, C-16, C-17, C-18
- **Explosion and vulnerability:** C-19, C-21, C-22
- **Reporting:** C-23, C-24 (citations and reference list)
- **V&V regression suite:** all hand-check cases from the three reviews, published worked examples (YB, CCPS, C&L), and reference-software comparisons with stated tolerances. Re-review the results after the fixes.

### Tier 3: improvements

- C-27 to C-43 (Minor), including the lead's flare-scope (C-43) and fire-duration (C-42) items.
- C-44 to C-50 (Observations): documentation only.
- Verify against source every value marked **"verify"** in this register before coding it:
  - API 521 table, solar footnote, escape parameters and point-source validity statement;
  - CPR 14E and CPR 18E equation numbers and minimum pool-layer thickness;
  - B&M instantaneous table;
  - Alonso (2006) coefficients;
  - LNG SEP field data;
  - AIHA ERPG revisions;
  - Purple Book probit constants.

---

## 7. What is done well (retain)

- Choked, subsonic and liquid discharge equations are exact, and continuous at the critical ratio. Briggs rural and urban sigmas are correct. The Gaussian plume and its inversion (including the elevated peak search) are correct. The B&M continuous table is transcribed exactly.
- The API 521 point-source equation, the Bagster transmissivity, the TNO fireball correlations (r_fb, t, SEP and view factor) and the Babrauskas burning-rate form are coded correctly. Hc is the net value (LHV), which is correct for radiation.
- Probit constants, units (mg/m³ with minutes; W/m² with seconds) and ppm↔mg/m³ conversions are correct. The eardrum probit is correct.
- The pure-function engine with `module.exports` enabled independent verification. The citation registry and screening disclaimers are good practice and should be kept and made accurate.

---

*Evidence:*
- *Reviewer scripts and outputs:* `/tmp/claude-0/-home-user-QRA-Consequence-Studio/563caf17-2447-5d2c-b4c7-9b8c85dbd693/scratchpad/` (`fire/`, `nonfire/`, `qa/`).
- *Lead's independent view-factor check:* `/tmp/claude-0/-home-user-QRA-Consequence-Studio/563caf17-2447-5d2c-b4c7-9b8c85dbd693/scratchpad/lead_vf.py`.
