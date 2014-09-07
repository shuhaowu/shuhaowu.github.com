title: Exploring Firefox crash data, Part I
published: yes
date: 2013-07-25 23:04
mathjax: yes
lightbox: yes

Recently I've taken on a new challenge at Mozilla: analyze Firefox crash stats
to find "explosive" crashes. That is, finding events that have a high upward
trend and should be a cause for concern for those in the engineering
department.

So what are "explosive crashes"? Before we can get to that, we need to take a
look at what each crash report consists of. When Firefox crashes, it sends back
a core dump with some information to [Socorro][socorro]. Socorro takes the core
dump and generates a signature. An example signature could be
`gfxContext::PushClipsToDT(mozilla::gfx::DrawTarget*)` or
`CContext::RestorePipelineStateImpl<int>(SAPIPipelineState*)` and they group
similar/identical crashes together. Explosive crashes are signatures that
experience a sudden increased in crash volume. We need to catch these signatures
as they are usually related to an emerging bug. Since we get so many crashes per
day (around 3000 crashes per minute), it is simply infeasible for human beings
to go in and identify these explosive events. This is why we need an algorithm
to detect these automatically.

[socorro]: https://github.com/mozilla/socorro

------------------------------------------------------------------------------

For the past week or so, I have been looking at historic Firefox crash data
available from [crash-analytics][ca] to explore the data and get a good idea
of what I am dealing with. The data set available on the site represents about
10% of the crash reports that are randomly selected. 

[ca]: https://crash-analysis.mozilla.com/crash_analysis/

First, I generated some summarizing graphs to see what the data is all about.

*Note: the data here are from April 15th 2013 to June 15th 2013. The x axis
probably does not make any sense. It simply represents the bins.*

<div class="center">
  <a href="/static/img/moz-crash-analytics/global-crash-6.png" rel="lightbox[gcc]" title="Crash counts binned by 6 hours"><img src="/static/img/moz-crash-analytics/thumbs/global-crash-6.png" /></a>
  <a href="/static/img/moz-crash-analytics/global-crash-12.png" rel="lightbox[gcc]" title="Crash counts binned by 12 hours"><img src="/static/img/moz-crash-analytics/thumbs/global-crash-12.png" /></a>
  <a href="/static/img/moz-crash-analytics/global-crash-24.png" rel="lightbox[gcc]" title="Crash counts binned by 24 hours"><img src="/static/img/moz-crash-analytics/thumbs/global-crash-24.png" /></a>
  <p>All crash counts per 6, 12, and 24 hours</p>
</div>

These here are all crashes binned against time. Each point represents the number
of crashes that happened in that bin (6, 12, 24 hour period). Visually, we can
already tell that the data is highly periodic, especially in the 6 and 12 hour
bins. We can see that there are more crashes during the weekdays than the
weekends. We can also see that there is a day night cycle. 

<div class="center">
  <a href="/static/img/moz-crash-analytics/global-crash-6-annotated.png" rel="lightbox[a1]" title="The blue boxes are the weekdays and the red boxes are the  weekends."><img src="/static/img/moz-crash-analytics/thumbs/global-crash-6-annotated.png" /></a>
  <p>The blue boxes are the weekdays and the red boxes are the weekends.</p>
</div>

Similar cycles exist for crash volumes for individual signatures. Most of them
follow one global trend or another global trend. We see that the signatures all
are gradually decreasing like the global crash volume. You can see that the
second and third signature do not appear to follow the weekday/weekend cycle.
This is an interesting phenomenon that I do not have an explanation to.

*Note: the SHA1 on top is simply a SHA1 of the crash signature so it is easier
to work with when I'm calling the tool I wrote in command line. It is
insignificant.*

<div class="center">
  <a href="/static/img/moz-crash-analytics/sig-1-crash-6.png" rel="lightbox[scc]" title="Crashes for a signature binned by 6 hours"><img src="/static/img/moz-crash-analytics/thumbs/sig-1-crash-6.png" /></a>
  <a href="/static/img/moz-crash-analytics/sig-2-crash-6.png" rel="lightbox[scc]" title="Crashes for a signature binned by 6 hours"><img src="/static/img/moz-crash-analytics/thumbs/sig-2-crash-6.png" /></a>
  <a href="/static/img/moz-crash-analytics/sig-3-crash-6.png" rel="lightbox[scc]" title="Crashes for a signature binned by 6 hours"><img src="/static/img/moz-crash-analytics/thumbs/sig-3-crash-6.png" /></a>
  <p>Crashes for the top 3 crash signatures binned by 6 hours.</p>
</div>
<div class="center">
  <a href="/static/img/moz-crash-analytics/sig-1-crash-24.png" rel="lightbox[scc]" title="Crashes for a signature binned by 24 hours"><img src="/static/img/moz-crash-analytics/thumbs/sig-1-crash-24.png" /></a>
  <a href="/static/img/moz-crash-analytics/sig-2-crash-24.png" rel="lightbox[scc]" title="Crashes for a signature binned by 24 hours"><img src="/static/img/moz-crash-analytics/thumbs/sig-2-crash-24.png" /></a>
  <a href="/static/img/moz-crash-analytics/sig-3-crash-24.png" rel="lightbox[scc]" title="Crashes for a signature binned by 24 hours"><img src="/static/img/moz-crash-analytics/thumbs/sig-3-crash-24.png" /></a>
  <p>Crashes for the top 3 crash signatures binned by 24 hours.</p>
</div>

At this point, it is pretty clear that it is probably a good idea to work with
the 24 hour bin as oppose to anything smaller as it smooths out the day night
cycle. Also, since the plan is to run the explosive detection script as a cron
job that happens every night, looking at smaller intervals do not really
give us any significant advantages.

However, it still seems obvious that the global crash rates influences crash
rates for particular signatures such that when global crash rate rises, crash
rates for individual signatures also rise and vice versa. In our detection
algorithm, we need to establish a baseline so that we won't flag events such as
people turning on their computers after a weekend as an explosive crash. So the
first thing I tried is to use a ratio between crashes for a particular
signature and the global crashes. This is computed using the following:

$$\mbox{ratio} = \frac{\mbox{crash count for signature}}{\mbox{global crash count}}$$

*Note: this ratio is displayed on the y axis.*

<div class="center">
  <a href="/static/img/moz-crash-analytics/sig-1-crash-24-norm.png" rel="lightbox[sccn]" title="Crash ratios for a signature binned by 24 hours"><img src="/static/img/moz-crash-analytics/thumbs/sig-1-crash-24-norm.png" /></a>
  <a href="/static/img/moz-crash-analytics/sig-2-crash-24-norm.png" rel="lightbox[sccn]" title="Crash ratios for a signature binned by 24 hours"><img src="/static/img/moz-crash-analytics/thumbs/sig-2-crash-24-norm.png" /></a>
  <a href="/static/img/moz-crash-analytics/sig-3-crash-24-norm.png" rel="lightbox[sccn]" title="Crash ratios for a signature binned by 24 hours"><img src="/static/img/moz-crash-analytics/thumbs/sig-3-crash-24-norm.png" /></a>
  <p>Crash ratios for the top 3 crash signatures binned by 24 hours.</p>
</div>

We can see it smoothed out a the original data by a little. While the graphs
may be deceiving, we can see from the y axis that the drops and rises are not
as steep as the original graphs. This remained to be a pretty tempting method
until I started experimented on a different set of data: during the 2012
Olympics, Google released a doodle that crashed Firefox and doubled our total
crash volume. It looks something like this:

<div class="center">
  <a href="/static/img/moz-crash-analytics/global-crash-olympics.png" rel="lightbox[a2]" title="Crash volume doubled due to a Google Doodle"><img src="/static/img/moz-crash-analytics/thumbs/global-crash-olympics.png" /></a>
  <p>Crash volume doubled due to a <a href="http://www.google.com/doodles/hurdles-2012" target="_blank" rel="nofollow">Google Doodle</a></p>
</div>

Since most signatures did not see an "explosive" rise in volume, the normalized
graphs showed a dip during the day Google activated the problematic doodle
followed by a drastic rise as Google turned off the doodle. This dip is
problematic as it follows with a rise, which is not really the case when you
look at the absolute crash volume.

<div class="center">
  <a href="/static/img/moz-crash-analytics/sig-1-crash-olympics.png" rel="lightbox[sccn]" title="Crashes for a signature binned by 24 hours during the around the Google Doodle event"><img src="/static/img/moz-crash-analytics/thumbs/sig-1-crash-olympics.png" /></a>
  <a href="/static/img/moz-crash-analytics/sig-2-crash-olympics.png" rel="lightbox[sccn]" title="Crashes for a signature binned by 24 hours during the around the Google Doodle event"><img src="/static/img/moz-crash-analytics/thumbs/sig-2-crash-olympics.png" /></a>
  <a href="/static/img/moz-crash-analytics/sig-3-crash-olympics.png" rel="lightbox[sccn]" title="Crashes for a signature binned by 24 hours during the around the Google Doodle event"><img src="/static/img/moz-crash-analytics/thumbs/sig-3-crash-olympics.png" /></a>
  <p>Crash <strong>counts</strong> for 3 crash signatures during the 2 months
    surrounding the Google Doodle event. The last one is <em>probably</em> the
    signature for the crash caused by the Doodle, although I cannot be sure.</p>
</div>

<div class="center">
  <a href="/static/img/moz-crash-analytics/sig-1-crash-olympics-norm.png" rel="lightbox[sccn]" title="Crash ratios for a signature binned by 24 hours during the around the Google Doodle event"><img src="/static/img/moz-crash-analytics/thumbs/sig-1-crash-olympics-norm.png" /></a>
  <a href="/static/img/moz-crash-analytics/sig-2-crash-olympics-norm.png" rel="lightbox[sccn]" title="Crash ratios for a signature binned by 24 hours during the around the Google Doodle event"><img src="/static/img/moz-crash-analytics/thumbs/sig-2-crash-olympics-norm.png" /></a>
  <a href="/static/img/moz-crash-analytics/sig-3-crash-olympics-norm.png" rel="lightbox[sccn]" title="Crash ratios for a signature binned by 24 hours during the around the Google Doodle event"><img src="/static/img/moz-crash-analytics/thumbs/sig-3-crash-olympics-norm.png" /></a>
  <p>Crash <strong>ratios</strong> for 3 crash signatures during the 2 months
    surrounding the Google Doodle event</p>
</div>

At this point in time, I felt like I was not getting anywhere with these graphs.
I did feel fairly comfortable with the data set presented to me, so I moved on
to formulating a more rigorous definition of explosive crashes. (although there
is still a lot to see, even though I have seen much more graphs than just the
ones I've displayed here).

Intuitively, we tend to think of these crashes as crashes that happens over
night and represents a significant portion of the total crash volume. This seem
relatively simple as all we really need to determine is a high upward slope for
any given crash rates. However, as we talked about previously, it is difficult
to establish a baseline to filter out the seasonality and global trends.

This is further complicated by the fact that how do you turn "high upward slope"
into an actually mathematical formula? One could simply establish a threshold
for the slope or counts (or other measure of "crashiness"). However, since slope
is calculated via the current value minus the last value, an increase from 500
crashes to 10500 crashes would have a slope of 10000 as well as an increase from
100000 to 110000 crashes. The increase from 500 to 10500 seems rather unexpected
while the 100000 to 110000 increase could just be noise.

If all we care is unexpected events, then can't we just detect any outliers?
Pretend that we can predict the amount of crashes that we are expected to see
tomorrow. When tomorrow becomes today, we can then check if the actual rate is
much much higher than the predicted rate for each signature. If this is true, we
mark the signature as being explosive. Mathematically, it is defined as:

$$\widehat{y_t} - y_t > \sigma_\hat{y}$$

Where \\( \widehat{y_t} \\) is the predicted crash counts at time \\( t \\) and
\\( \sigma_\hat{y} \\) is the uncertainty or error of the prediction 
\\( \widehat{y_t} \\).

A neat thing about this formula is that \\( y \\) does not have to be based on
the crash count alone. We could use a normalized version of the crash count,
the slope of the crash counts over time, or any other properties that we can
derive from the crash volume.

The first thing we want to find out is if we could in fact infer tomorrow's
crash counts based on history. To do this, we check out the lagged scatter plot
and [ACF][acf] of our dataset.

[acf]: https://en.wikipedia.org/wiki/Autocorrelation_function

<div class="center">
  <a href="/static/img/moz-crash-analytics/lagged-scatter-plot-1.png" rel="lightbox[acf]" title="Lagged scatter plot of 1 for crash data binned by 24 hours. r=0.70"><img src="/static/img/moz-crash-analytics/thumbs/lagged-scatter-plot-1.png" /></a>
  <a href="/static/img/moz-crash-analytics/sample-acf-1.png" rel="lightbox[acf]" title="ACF Plot for a week period."><img src="/static/img/moz-crash-analytics/thumbs/sample-acf-1.png" /></a>
  <p>Lagged scatter plot and sample ACF plot</p>
</div>

This is all I have done so far and I think the next step for me would be figure
out a prediction algorithm for \\( \widehat{y_t} \\) and 
\\( \sigma_\hat{y} \\). I think it is a fairly safe bet to use [ARMA][arma] as
it seem to satisfy all the requirements for it to be used. However, I am still
working on confirming that (this is the first time I have done time series
analysis so I am learning as I go along).

[arma]: http://en.wikipedia.org/wiki/Autoregressive%E2%80%93moving-average_model

[Subscribe][sub] for part 2 (hopefully by the middle of next week)!

[sub]: /blog/rss.xml
