title: Exploring Firefox crash data, Part 2
lightbox: yes
mathjax: yes

[A couple of weeks ago][prev], I spent some time exploring Firefox crash data
so that I can come up with a model to catch "explosive crashes" in a timely
fashion. Over the last couple of weeks, I have evaluated and identified a
model that is "good enough". Currently, the code is sitting in a
[pull request][pr], waiting to be landed in Socorro.

[prev]: /blog/exploring_firefox_crash_data_1.html
[pr]: https://github.com/mozilla/socorro/pull/1394

This post will mainly focus on many of the models I have tried and did not
include in my final implementation.

-------------------------------------------------------------------------------

At the end of the last post, I identified some trends in the data. We looked at
crash counts and the possibility of using a predictive model and measure the
deviation to figure out if a crash is explosive.

The first thing I tried is a [sinusoidal model][sm] as we intuitively knew that
our data has a seasonal pattern and the ACF plot shows that this is the case. 

[sm]: http://en.wikipedia.org/wiki/Sinusoidal_model

<div class="center">
  <a href="/static/img/moz-crash-analytics2/acf_sine.png" rel="lightbox[as1]"><img src="/static/img/moz-crash-analytics2/thumbs/acf_sine.png" /></a>
  <p>ACF plot if we correlated using 20 days of data from May 27 2013 to
    June 15th 2013</p>
</div>

The sinusoidal model uses the following formula for prediction:

$$\hat{y}_t = \mu + \alpha \sin(\frac{2\pi}{\omega} t + \phi)$$

We need to fit for \\( \alpha \\), \\( \omega \\), \\( \mu \\) and \\( \phi \\)
with this formula. For each signature, we take the last 2 weeks (or any other
periods) of data find the values. Then, we simply plug in tomorrow's \\( t \\)
to get a predicted value.

We can lock \\( \omega \\) to be 7 as we know that our period is 7 due to
weekly data fluctuations.

This formula is not exactly ideal as it does not account for an overall upward
or downward trend in the crashes, which we see in crash data.

<div class="center">
  <a href="/static/img/moz-crash-analytics2/downward_trend.png" rel="lightbox[b]"><img src="/static/img/moz-crash-analytics2/thumbs/downward_trend.png" /></a>
  <p>General downward trend as Firefox matures.</p>
</div>

This is easily accounted for if we replace \\( \mu \\) with a linear function
so our model becomes:

$$\hat{y}_t = \beta_0 + \beta_1 t + \alpha \sin(\frac{2\pi}{\omega} t + \phi)$$

This model worked reasonably well. In fact, it is one of the better models from
my arsenal that I've tried (average error is only 5%, with a catch). Here is an
attempt at predicting tomorrow's global crash volume:

<div class="center">
  <a href="/static/img/moz-crash-analytics2/sine_global_calm_predicted.png" rel="lightbox[a2]"><img src="/static/img/moz-crash-analytics2/thumbs/sine_global_calm_predicted.png" /></a>
  <p>Predicted global crash counts vs actual global crash counts via a
    sinusoidal model</p>
</div>

The residuals graphs also look pretty good as it seems fairly random.

<div class="center">
  <a href="/static/img/moz-crash-analytics2/sine_global_calm_residual_hist.png" rel="lightbox[r1]"><img src="/static/img/moz-crash-analytics2/thumbs/sine_global_calm_residual_hist.png" /></a>
  <a href="/static/img/moz-crash-analytics2/sine_global_calm_residual_lagged_scatter.png" rel="lightbox[r1]"><img src="/static/img/moz-crash-analytics2/thumbs/sine_global_calm_residual_lagged_scatter.png" /></a>
  <br />
  <a href="/static/img/moz-crash-analytics2/sine_global_calm_residual_vs_fit.png" rel="lightbox[r1]"><img src="/static/img/moz-crash-analytics2/thumbs/sine_global_calm_residual_vs_fit.png" /></a>
  <p>Residual histogram looks fairly normal. Lagged scatter plot does not show
    significant correlations. Residual vs fits has an average of approximately zero.</p>
</div>

However, The model required the time series to be a sinusoidal curve. This is
to say that the number of crashes must show a pattern over the week. Otherwise
it performs poorly as shown below:


<div class="center">
  <a href="/static/img/moz-crash-analytics2/sine_sig1_calm_predicted.png" rel="lightbox[a2]"><img src="/static/img/moz-crash-analytics2/thumbs/sine_sig1_calm_predicted.png" /></a>
  <p>Predicted crash counts vs actual crash counts via a
    sinusoidal model for a specific crash signature</p>
</div>

<div class="center">
  <a href="/static/img/moz-crash-analytics2/sine_sig1_calm_residual_hist.png" rel="lightbox[r1]"><img src="/static/img/moz-crash-analytics2/thumbs/sine_sig1_calm_residual_hist.png" /></a>
  <a href="/static/img/moz-crash-analytics2/sine_sig1_calm_residual_lagged_scatter.png" rel="lightbox[r1]"><img src="/static/img/moz-crash-analytics2/thumbs/sine_sig1_calm_residual_lagged_scatter.png" /></a>
  <br />
  <a href="/static/img/moz-crash-analytics2/sine_sig1_calm_residual_vs_fit.png" rel="lightbox[r1]"><img src="/static/img/moz-crash-analytics2/thumbs/sine_sig1_calm_residual_vs_fit.png" /></a>
  <p>These are bad in the sense that the residuals do not seem that random.</p>
</div>

A problem with the sinusoidal model is that there are hundreds of thousands of
time series and I cannot feasibly look through all of them. While the
sinusoidal model is good for certain types of time series, I cannot say that
all crash signatures at all times exhibits the same patterns, which they don't.

It turns out, this is also the problem with many other models as well.
Specifically, [exponential weighted moving average][ewma] and
[decompositions models][dm] worked terribly as they just do not fit the data
like they do in textbooks. With these models, 20% to 30% errors are not uncommon.

[ewma]: http://www.jstor.org/discover/10.2307/2984031?uid=3739560&uid=2129&uid=2&uid=70&uid=4&uid=3739256&sid=21102568940463
[dm]: https://onlinecourses.science.psu.edu/stat510/?q=node/69

So at this point I realized that I'm generalizing too much. Although many crash
signatures exhibits roughly the same behaviour, I cannot just come up with a
model by looking at a small subset of the signatures and say that this model
fits for all signatures. It just doesn't. In order for me to get good prediction
results, I (or an algorithm) would have to look at every single crash signatures
and decide on a prediction model for each signature. It is not feasible for me
to look through the entire data set and an algorithm would perhaps be even more 
challenging.

Another important thing that I realized at this point is that I do not actually
know how well these prediction models will perform when it comes to classifying
explosive crashes. There are no training examples other than the Olympic doodle
crash. Without a proper test set, all I can do is cross my fingers and hope
that the model works.

In summary, there are several problems that I faced with the time series based
approach:

 1. Technical limitations which would mean that I have to implement algorithms
    that are difficult to implement and usually not implemented as we rely on
    tools such as scipy.
 2. Time series approach assumes that all crash signatures can be modeled by
    the same type of time series.
 3. We lacked test set examples so we cannot evaluate how well the model
    performed after it is developed.

Since my internship is coming rapidly to a close, it was decided, after
some discussions, that we should first try a much simpler model than the ones 
proposed thus far. I can use this model to first test for additional explosive
examples so that I can use it to verify whatever models I can come up with. 
It so turned out that the method used to find these examples, despite being
very simple, is also very good. I'll detail this in a follow up blog post.

