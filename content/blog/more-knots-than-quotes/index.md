---
title: "More knots than quotes: an underdetermined curve fit against the bootstrap"
date: 2026-10-13
draft: false
description: "Out of sample, an underdetermined curve fit's median and tail errors are at or below both bootstraps' in every window and currency I tested, and its lead is largest on the front end when the market is pricing scheduled central-bank decisions. Seven windows of OIS curves, 2019 to 2025."
summary: "Out of sample, an underdetermined curve fit's median and tail errors are at or below both bootstraps' in every window and currency I tested, and its lead is largest on the front end when the market is pricing scheduled central-bank decisions. Seven windows of OIS curves, 2019 to 2025."
tags:
  - Yield Curves
  - Curve Construction
  - OIS
  - Interest Rates
  - Quantara
categories:
  - Quant Finance
  - Quantara
series:
  - Quantara Internals
math: true
mermaid: false
---

## A weekend in March

On Tuesday 7 March 2023 the Fed chair told the Senate that rates were likely to peak higher than
previously expected. On Friday Silicon Valley Bank was closed. On Sunday evening the authorities
guaranteed all of its deposits and the Fed opened a new lending facility. Between Wednesday and
Monday the two-year Treasury yield fell from 5.05 to 4.03 per cent. On 22 March the Fed raised by
a quarter point.

Every curve built that Monday had to absorb a front end that had repriced a policy path in three
sessions. A yield curve is never observed; it is fitted. On a day like that, how much of a curve's
error comes from the market, and how much from the way we fit it?

I measured it on USD, EUR and GBP overnight-index swap curves, comparing the standard bootstrap
with an underdetermined fit built on the search Tom Hyer describes in *Derivatives Algorithms,
Volume 1: Bones* (World Scientific). Asked to price quotes it was not given, the underdetermined
fit's median and tail errors are at or below both bootstraps' in every window and currency. Where
its lead is largest surprised me. On the front end it is not largest when markets are stressed;
it is largest when the front end is pricing what central banks will do at their next scheduled
meetings.

## Two ways to fit a curve

**The bootstrap** puts one knot per instrument and solves them in maturity order, each quote fixing
the curve out to its own maturity. It reprices every input exactly and it is fast, but the shape
between quotes is whatever the interpolation says. I test two versions: flat forwards between
pillars, the common default, and a smooth monotone Hermite interpolation, the stronger of the two.

**The underdetermined fit** uses many more knots than instruments, on purpose, so infinitely many
curves reprice the market. Among them it picks the smoothest. Nothing in it comes from yesterday:
each date is fitted from that day's quotes and the meeting calendar alone. Each step is the
minimum-norm Gauss-Newton step

$$
s = W^{-1} J^{\top} \left( J W^{-1} J^{\top} \right)^{-1} r
$$

where $J$ is the Jacobian of the instruments with respect to the curve parameters, $r$ the
mispricing, and $W$ the metric that measures roughness: it penalises the step between neighbouring
forwards and, beyond the scheduled meetings, how sharply the forward curve bends. Because the
metric lives in forward space, refining the grid does not change the answer.

Instruments no longer own knots, so the knots can sit where the overnight rate actually moves: on
the dates the upcoming scheduled decisions take effect, then on a grid that fills the rest of the
curve. A bootstrap cannot do this from pillars alone: it can step on a meeting date only if an
instrument ends there. Desks that step the front end either fit meeting-dated swaps, the
comparison in Exhibit 3, or spread each move over the meetings by a fixed rule, which is already a
hand-made choice among the curves that reprice the market. I did not test the second. The
front-end results below are a joint effect of the calendar and the freedom to use it.

## The test

Every method sees the same twenty OIS pillars, one week to thirty years, on the same dates, with
library defaults and no tuning per window. Leave one pillar out, refit, and price the missing
pillar off the refitted curve against its market quote: how well does a curve know the rates it
was not told?

The seven windows are a quiet first half of 2019 as the control, the September 2019 repo spike,
COVID in 2020, the 2022 hiking cycle, SVB and Credit Suisse in 2023, the April 2025 tariff
announcements, and the weak US payrolls of August 2025. EUR starts in October 2019, with the euro
short-term rate. Every window I ran is reported: 36,062 refits scored by all three methods.

## What I found

{{< chart src="charts/exhibit1-p90-by-window.svg" data="data/exhibit1_loo_by_window.csv" datalabel="p50, p90 and max by window, currency and method (CSV)" >}}
**Exhibit 1.** Leave-one-out par error, 90th percentile, by window and currency. The smooth bootstrap's tail error is 1.4 to 4.3 times the underdetermined fit's.
{{< /chart >}}

The narrowest margin is the 2019 repo window in USD, 2.33 against 3.36 bp at the 90th percentile.
In 2022 USD the fit's median and 90th percentile are 0.54 and 1.58 bp,
against 1.79 and 6.00 for the smooth bootstrap and 2.64 and 8.15 for the flat one.

{{< chart src="charts/exhibit2-p50-by-tenor.svg" data="data/exhibit2_loo_by_tenor.csv" datalabel="p50 and p90 by tenor (CSV)" >}}
**Exhibit 2.** Leave-one-out par error by tenor, median, pooled over windows. The underdetermined fit is ahead of both bootstraps at every tenor from one year to 25 years in every currency, and far ahead at most tenors from one month to two years. The exceptions, all shorter than a year, are under "Where it loses".
{{< /chart >}}

The sharpest gap is at three and six months, where the smooth bootstrap's median error is four to
eleven times the fit's. That is where the policy path lives: a meeting or two sits between
neighbouring pillars, and the shape of the curve between them is the information. A bootstrap
interpolates across it; a fit with knots where the decisions take effect represents it. At 25
years the fit's median error is 0.3 to 0.5 bp against 0.5 to 1.0 for the smooth bootstrap,
narrowly so in GBP.

**It isn't the crisis, it's the policy path.** On the front end, two weeks to two years, the
smooth bootstrap's tail error is about 2.5 times the fit's in the quiet control, 1.3 to 2.7 times
in the repo and COVID windows, and 3.2 to 4.8 times in 2022, 2023 and 2025
([CSV](data/frontend_ratio.csv)). The gap is largest where the front end repriced around
scheduled decisions and smallest in COVID, which brought the fastest cuts in the sample: the Fed
and the Bank of England made them between meetings and the ECB did not move. No calendar knows an
emergency cut in advance, so the method says nothing about unscheduled decisions. And this is a
tendency, not a law: the repo window held two scheduled Fed cuts and shows only a moderate gap.

**Exhibit 3: meeting-dated swaps.** Swaps on single meeting-to-meeting periods price each decision
directly, and their trades are public swap-repository prints. In the two 2025 windows I fitted
them alongside the pillars, held out in turn each traded period the bootstrap chain can reach, and
priced it against its traded rate. A period is scored on a date only when at least three
independent prints survive the data provider's off-market screen. A one-knot-per-instrument
bootstrap can only take these periods as a chain, dropping on average 5.3 to 6.4 of the twenty
pillars per date; the fit takes every period and every pillar together. The held-out period and
the scored set are the same for all three methods.

{{< metric-table src="data/exhibit3_meeting_swaps.csv" group="window" label="ccy" count="periods"
    metrics="p90=90th percentile, absolute; mean=Mean, signed; median=Median, signed"
    methods="flat=Flat; smooth=Smooth; ud=Fit" signed="mean median"
    datalabel="meeting-dated swap errors by window, currency and method (CSV)" >}}
**Exhibit 3.** Held-out meeting-dated swaps against their traded rate, bp; scored periods in brackets. Negative means the curve sits below the traded rate. Bold marks the value closest to zero in each group.
{{< /metric-table >}}

The clearest result is bias. Both bootstraps sit below the traded rate on average in every cell,
the flat version more than the smooth, and just as strongly when the refit contains no other
traded period: the bias comes from how the bootstrap builds the curve, not from the targets. The
fit's average error is within a basis point of zero in every cell, with no consistent sign. Its
90th percentile is the lowest in all six cells: significantly so against the flat bootstrap
everywhere and against the smooth one in the EUR and GBP tariff cells, a consistent pattern in the
other four. At the median the smooth bootstrap and the fit are within 0.4 bp in every cell. Here
the bootstraps know the calendar too: the median gap to the smooth bootstrap closes, while the
tail and the bias remain.

Two qualifications. Both windows priced easing; if the bias comes from how the bootstrap builds
the curve, a window that priced hikes should reverse its sign, and that is the next test. And the
USD sample is narrow, about one period per date.

## Where it loses

At two weeks the fit's median error is larger than either bootstrap's in every currency, by at
most 0.15 bp, and at three weeks it trails the flat bootstrap in EUR and GBP, although at both
tenors its 90th percentile is two to four times smaller. At nine months it trails the smooth
bootstrap marginally at the median in EUR and clearly at the 90th percentile in GBP (2.01 against
1.22 bp). On 24 USD dates in early 2019 without a 25-year quote it misses the 20-year by about
1.8 bp, the bootstraps by 1.4 to 1.7. Its single worst error, 17.7 bp on a 20-year quote on
17 March 2020, is larger than either bootstrap's worst in that window; in the other nineteen
window and currency cells its worst is the smallest of the three.

Nor does it reprice everything exactly. On the front end it treats the short quotes as targets
rather than hard constraints: by window and currency the median in-sample miss is 0.04 to 0.4 bp
and the 90th percentile 0.1 to 2.1 bp, the large misses mostly on one-week quotes. The bootstraps
reprice their inputs to 0.005 bp. Whether that trade is worth it is the point of Exhibit 1.

## What it costs

More knots should mean slower. On the same twenty instruments the fit solves for about 160 knots
against the bootstrap's 20, an eight times larger problem, yet in the two windows I timed (2022
and the 2025 tariffs) it costs 0.81 to 1.10 times the flat bootstrap per date, Python binding
overhead included. The smooth bootstrap costs two to nearly five times as much as the fit: its
slope at each knot depends on the next knot, so it solves the chain several times over before
it settles. Of 36,067 refits per method the fit failed none; the smooth bootstrap failed five.

## An old idea

The idea is not new: Hyer's book describes the search. What this adds is evidence, scored against
quotes the curve was not given: the underdetermined fit is at least as close to the market as the
bootstrap, at the median and in the tail, in every window I tested, and its front-end lead is
largest when the market is pricing a path of scheduled central-bank decisions.

The fit described here is part of Quantara, a cross-asset analytics library; the
[Quantara Internals](/series/quantara-internals/) series covers more of it.

*Data: OIS quotes from a commercial vendor capture (only aggregate error statistics are shown);
meeting-period swaps from public swap data repository prints, after the provider's screen for
fee-bearing, unwound and outlier prints (88 per cent of excluded period-days had fewer than three
prints to begin with); central-bank calendars as known on each date, cancelled and moved meetings
included.*
