---
title: "What Have I Been Listening To?"
layout: post
categories: [music, data]
tags: [python, ggplot2, R]
---

Spotify doesn't (easily) let users track their listening history, which is a bummer for someone like me who listens to a lot of music and would love to play with the data. However, buried in the Spotify API is a functionality that **does** let users access their own "top artists".<sup>1</sup>

I grabbed my top 50 'long term' favorite artists and ran a clustering algorithm on them to try and find similarities (all the code is [here]({{site.baseurl}}{% post_url 2017-07-25-spotify-code %})). The results are ... that I'm pretty lame. I honestly didn't expect my tastes to be so out-of-date; the majority of my top artists are from the window from 1990-2010. What's more, most of what I listen to that was released after 2010 is either incredibly popular hip-hop ("have you heard of these indie rappers Run The Jewels?" /s) or local NYC bands that I go and see live.

Anyway, data don't lie, see for yourself below.


<iframe src='{{site.baseurl}}/spotify_cluster_output.html' height = "600px" width="100%">
</iframe>

*A note on the Spotify API:*
[1] Look, I just admitted I'm lame, but I'm not *that* lame. I cannot figure out why Spotify thinks I listen to Harry Belafonte a lot. I don't think I've ever done that knowingly; the same goes for Imagine Dragons. And I certainly don't listen to Bob Marley often enough to justify him being a top artist. My best guess is there's some idiosyncrasy in Spotify's own 'top artist' algorithm that weights very popular artists more heavily in the rankings. It just seems impossible for these to be based purely on my own play counts.
