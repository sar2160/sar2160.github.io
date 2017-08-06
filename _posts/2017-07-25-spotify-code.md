---
title: "Spotify Artist Clustering Code (Python & R)"
layout: post
categories: [music, data]
tags: [python , k-means clustering, R, ggplot2]
---



This is the Python and R code I used to make a [visualization of my listening tastes]({{site.baseurl}}{% post_url 2017-07-25-spotify-top-artists %}) on Spotify.

## Getting API data and munging (in Python)

I used the SpotiPy API wrapper to download my top 50 artists, as well as some data for their album release dates. I then munged the genres to be more machine-friendly in two steps:

1. Matched the artists on any self-reported genres they shared. I found most genres were too specific to match against other artists.
2. To counter this, I pulled out the top 20 single words appearing in any genre e.g, rock, pop, etc.. These became more broadly categorized *derived genres*.




{% highlight python %}
# In[1]:

import pandas as pd
import json
from numpy import log , nan , ravel

import spotipy
sp = spotipy.Spotify()

# In[2]:

# I saved results from spotipy API to a json
with open('examples/long_term.json') as data_file:    
    data = json.load(data_file)
top_artists = pd.io.json.json_normalize(data['items']).set_index('name', verify_integrity = True)

top_artists['log_followers'] = log(top_artists['followers.total'])

# In[5]: Finding every unique reported genre and making dummy variable columns for them
genres = top_artists.genres.apply(pd.Series)
genre_cat  =  pd.DataFrame(columns= pd.Series(genres.values.ravel()).unique() , index = top_artists.index )
genre_cat.fillna(False, inplace=True)

## finding most common words within self-reported genres in order to find "derived genres".
genre_words = ravel ( zip(pd.Series(genre_cat.columns).dropna().str.split(' ')))

slist =[]
for x in genre_words:
    slist.extend(x)

genre_words = pd.Series(slist).value_counts()[:20]  # only save 20 most frequently occuring words

print genre_words

genre_list = top_artists.genres.apply(lambda x: ' '.join(x))

## Find if artists report themselves in one of my derived genres
for g in genre_words.index:
    column_str = 'derived_'+ g
    top_artists[column_str] = genre_list.str.contains(g)


# In[8]: Get first album release year from Spotipy for each artist. Note: Some of these dates seem suspect, may be re-releases.

top_artists['artist_id'] = top_artists.uri.apply(lambda x: x[15:])

# This function look at all the albums returned for an artist and finds the earliest release date.
def get_earliest_album_date(artist_uri):
    all_albums = sp.artist_albums(artist_id = artist_uri)['items']

    earliest_date = pd.datetime.today() # initialize to today
    for a in all_albums:
        album_date = pd.to_datetime(sp.album(a['uri'][14:])['release_date'])
        #print album_date
        earliest_date = min(album_date, earliest_date)

    return earliest_date

# In[9]:

top_artists['first_album_release'] = top_artists.apply(lambda x: get_earliest_album_date(x.artist_id), axis = 1 )


# scary nested loop, not great practice but this is a tiny computation
for a in genre_cat.index:
    a_genres =  genres.ix[a].unique()
    for g in a_genres:
        genre_cat.ix[a][g] = True

genre_cat.drop(nan,inplace=True, axis = 1 )

## drop unneeded columns and export
top_artists.drop(['external_urls.spotify','href','images','type','followers.href','genres','id','uri','artist_id']                    , axis=1, inplace=True)
top_artists = top_artists                .merge(genre_cat, left_index= True, right_index = True, how = 'inner').reset_index()

top_artists.to_csv('top_artists_v2.csv', encoding='utf-8')

{% endhighlight %}


## Data Prep (in R)

I wanted to use some clustering algorithms where I am familiar with the R implementation. I pulled the munged file into R.


{% highlight r %}
top_artists <- read.csv('top_artists_v2.csv')
top_artists$X <- NULL
rownames(top_artists) <- top_artists$name
#top_artists <- top_artists[rownames(top_artists) != 'Harry Belafonte',] # this is an error, i swear. I have never listened to Harry Belafonte (other than after I found this).
top_artists$name <- NULL

top_artists$first_album_release <- as.Date(top_artists$first_album_release, origin = '1900-01-01')
top_artists$album_release_numeric <- as.numeric(top_artists$first_album_release)
top_artists$popularity <- as.numeric(top_artists$popularity)

date_vector <- top_artists$first_album_release

top_artists$first_album_release <- NULL
{% endhighlight %}

## Finding Similarities Using Gower Clustering

I wanted to find similarities in the artists I listen to based on any readily available information. In this case, I had access to **genre, time of being active, and popularity on Spotify** through the Spotify API.

Since music genre comparisons are binary in my implementation, I needed a clustering algorithm that can handle categorical comparisons as well as numerical distance. K-means clustering uses Euclidian distance and is not suited for this. Gower clustering can handle both types, even though categorical "distance" doesn't contain a lot of useful similarity information between two artists (two artists either share a genre or don't, there's no 'distance').

I used trial-and-error to weight my variables-of-interest:  

* Log( followers ) was weighted twice as heavy as the default. I used the log of followers because I wanted to control for gigantic variation in popularity among artists. Incidentally, the Spotify internal popularity index also resembles the log of total followers.   
* *derived genres* got less weight (0.7) than self-reported genres (1). Since self-reported genres are more specific I gave a match on derived genres less weight. For example, two self-reported "witch punk" artists are weighted more similarly than two derived "punk" artists.  
* Year of release for the artist's first album was also weighted twice as heavy (2).  


{% highlight r %}
library(cluster)

weight_vector = rep(1, length(colnames(top_artists)))
weight_vector[1] = 0 # ignore total followers (use log instead)
weight_vector[grepl('derived',colnames(top_artists))] = .7 # weight derived metagenres lower
weight_vector[2:3] = 2 #weight popularity heavily

weight_vector[length(weight_vector)] = 2 # weight release year heavily

dist <- daisy(top_artists , metric="gower", weights = weight_vector)
{% endhighlight %}

### Silhouette Width

Silhouette width measures the average similarity within a cluster vs. the similarity to the next-closest artist outside the cluster. A higher silhouette width [-1,+1] means that artists within a cluster are more similar to each other than artists nearby but outside the cluster. I tested silhouette width for 1-10 clusters (k), and decided to go with k = 6. As the graph beloew shows, K = 6 is not the maximum width, but I found it acceptable in order to find more variation in my tastes than just having k = 3.


{% highlight r %}
# Calculate silhouette width

sil_width <- c(NA)

for(i in 2:10){

  pam_fit <- pam(dist,
                 diss = TRUE,
                 k = i)

  sil_width[i] <- pam_fit$silinfo$avg.width

}

# Plot sihouette width (higher is better)

plot(1:10, sil_width,
     xlab = "Number of clusters",
     ylab = "Silhouette Width")
lines(1:10, sil_width)
{% endhighlight %}

![center]({{site.baseurl}}/images/2017-07-25-spotify-genres/unnamed-chunk-4-1.png)


### The results

I've collapsed the clusters onto only 2 dimensions - time and popularity -  in order to plot. Gower clustering uses medoids to cluster around, so each group is defined by a real artist, listed below.


{% highlight r %}
pam_fit <- pam(dist, diss = TRUE, k = 6)

pam_fit$medoids
{% endhighlight %}



{% highlight text %}
## [1] "Interpol"        "Gøggs"           "Otis Taylor"     "Mikal Cronin"   
## [5] "U2"              "Harry Belafonte"
{% endhighlight %}

I then defined the clusters using my own interpretation.Some were very naturally interpretable, such as the *Early 2000s indie rock* cluster. The *Harry Belafonte* cluster was definitely an outlier and I swear I have never listened to him - on Spotify or otherwise - in my life; I insist this is an error with the API. I also hope that with more data it would be possible to do a better job than putting so many modern artists in the *Modern Pop/Rock/HipHop Grab Bag* cluster.


{% highlight r %}
 library(ggplot2)
 library(plotly)

## These are my own interpretations of the clusters.
Clusters <- factor(pam_fit$clustering, labels= c('Early 2000s Indie','Modern Pop/Rock/HipHop Grab Bag','Blues','West Coast Indie','Classic Rock','Harry Belafonte ¯\\_(ツ)_/¯'))


gg <- ggplot(top_artists) + geom_point(mapping = aes(x = date_vector, y = log_followers, col = Clusters,
                                                     text = paste("Artist: ", rownames(top_artists),
                                                                  "</br> 1st Album: ", date_vector,
                                                                  "</br> Followers:", followers.total))) + ggthemes::theme_solarized(light = FALSE) + ggtitle('Simon\'s Spotify') + labs (x= '1st Album Release', y =' log (Followers on Spotify)') + theme(text = element_text(size = 9))

#save data for use elsewhere
save(pam_fit,file="pam_fit_r")
save(top_artists,file="top_artists_r")
save(date_vector,file="date_vector_r")

## vis in plotly to make things interactive
ggplotly(tooltip = c('text'), filename = 'spotify-plotly.html')
{% endhighlight %}

[See the final scatter plot here.]({{site.baseurl}}{% post_url 2017-07-25-spotify-top-artists %})
