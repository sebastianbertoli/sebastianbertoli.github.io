---
title: "New test"
excerpt_separator: "<!--more-->"
categories:
  - Uncategorized
tags:
  - Post Formats
---

```python
# ATTENTION: Run this cell first to appreciate the wonders of good typography!
from IPython.core.display import HTML
HTML(open("custom_styles.css", "r").read())
```

<a id='beginning'></a>
# Where people stay - extracting destinations from GPS data
<p> Author: [Sebastian Bertoli](https://www.sebastianbertoli.net) <br> Date: 01.01.2018</p>

## Introduction

This notebook showcases some of the work completed during my summer internship at the [Bruno Kessler Foundation](https://www.fbk.eu/en/). My responsibility was to find and implement an efficient algorithm that could, using GPS data, tell us where people had stayed for a pre-determined amount of time. So called stop-locations.

To extract stop-locations from GPS data I implemented and tested two algorithms: [ST-DBSCAN](https://www.sciencedirect.com/science/article/pii/S0169023X06000218) and an algorithm proposed by Kentaro and Toyama in [[1]](#hariharan2004). The focus of this notebook will be the latter.

### Outline
This notebook is structured as follows. First, we load and explore dataset I prepared for this experiment. Second, we process it and extract its stop locations. Third, we proceed with clustering the stop locations into so-called destinations (more on this later). Finally, we plot the results. (TODO: Compute statistics?) In short:

1. [Exploring the dataset](#eda)
1. [Extracting the users' stop locations](#extract_stops)
1. [Clustering the locations into destinations](#cluster_stops)
1. [Plotting the results](#calculate_rgyration)

**Note**: Most functions and plots are loaded from the accompanying `lachesis.py` and `plotly_helpers.py` files to avoid overloading this notebook with code. [Final thoughts](#finalthoughts), [acknowledgments](#acknowledgments) and [references](#references) can be found at the bottom.

<a id='eda'></a>
## Exploring the Data 
The proliferation of smartphones with GPS sensors has allowed to capture peoples movements in regular time intervals and at large scale. Once the data is collected, it can be further processed for research purposes. The unprocessed data typically consists of a timestamp, some type of identifier and longitude-latitude coordinates.

Foe the experiments we will use a random sample of the [T-Drive trajectory data sample](https://www.microsoft.com/en-us/research/publication/t-drive-trajectory-data-sample/). It contains one-week trajectories of 100 taxis in Bejing.

*Note:  If you prefer to use your own data you can do so, just make sure it is structured in the same way as the sample dataset.*

Without further adue let us load and explore the data!


```python
import pandas as pd
import matplotlib as mpl
import matplotlib.pyplot as plt
import plotly
from plotly_helpers import *  # Plot specifications
from lachesis import *  # Stop detection implementations
```


```python
# Various notebook settings
%load_ext autoreload
%autoreload 2
%matplotlib inline
mpl.rcParams['figure.dpi'] = 120
plt.style.use('ggplot')
```


```python
df = (pd.read_csv("data/df_sample.csv", parse_dates=["timestamp"])
      .sort_values(['user_id', 'timestamp']))
print(df.iloc[:3,:], 
      "\n\nNumber records:", df.shape[0],
      "\nNumber users:", len(df["user_id"].unique()))
```

           user_id           timestamp  longitude  latitude
    48013      165 2008-02-02 13:46:16  116.49889  39.92208
    48014      165 2008-02-02 13:56:16  116.48131  39.92130
    48015      165 2008-02-02 14:06:15  116.42837  39.90675 
    
    Number records: 111712 
    Number users: 100


As we can see our sample dataset contains of 111,712 records coming from 100 different users. Each record has a `user_id` identifying the user, a `timestamp` and a `longitude` as well as `latitude` value. 

One important property of GPS data is the time elapsed between subsequent observations denoted as $\delta t$. Let's quickly calculate it and plot its distribution.


```python
df_stats = pd.DataFrame()
df_stats['delta_t'] = (df.groupby('user_id')['timestamp']
                       .transform(lambda x: x.diff()) / np.timedelta64(1,'s'))
delta_t_plot = (df_stats[df_stats['delta_t'] < df_stats['delta_t']
                .quantile(.96)]
                .hist(bins=50, figsize=(6,2), alpha=.9, color="black"))
```


![png](output_9_0.png)


We observe that for most records (96th percentile) the time lapsed between subsequent recordings is within 310 seconds. This gives us sufficient granularity to run our stop-detection algorithms on this dataset. 

Now let us get a better idea of the data by plotting a random sample on a map. We use the popular [Plotly library](https://plot.ly) for this. The plot is fully interactive so feel free explore the data on the map. 


```python
fig_datsample = plot_datasample(df.sample(df.shape[0]//20, random_state=10))
py.iplot(fig_datsample, filename='fig_datsample')
```




<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~public.sebastian/11.embed" height="340px" width="680px"></iframe>



Despite having only plotted 5 % of the data we still get a good idea of the area covered by the dataset. If we zoom in we can can also discern some of the structural street patterns and denser areas. 

Next let us look at only a handful of datapoints of user 4813. 


```python
fig_oneuser = plot_one_user(df[df['user_id'] == 4813].iloc[109:115,:])
py.iplot(fig_oneuser, filename='fig_oneuser')
```




<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~public.sebastian/17.embed" height="340px" width="680px"></iframe>



In the figure above we have plotted six datapoints. Let us first focus on the three points in the upper right corner east of the *China Foreign Affairs University*. If you zooms-in on those points you can see from the labels that this user has stayed in close proximity of that location from `12:41` to `12:57`. Subsequently the users moves South (`12:58`) to *Outer Fuchengmen Street* (`13:00`) and moves west (`13:02`) until he disappears from the frame.

The three points in the upper right corner should be a stop location. In other words, a location where a user (in this case a taxi) has stayed for some time. Unfortunately, the dataset does not yet have this information but we will add this information in the next setion.

## Extract the users' stop-locations <a id='extract_stops'></a>
In this section we briefly explain the stop location extraction algorithm from [[2]](#hariharan2004). After that we extract all the stop locations from the sample dataset. Let us begin with some definitions. 

### Definitions
There are three essential concepts: the position, the stop location and the destination.

<span style="color:blue">&#9679;</span> **Position**: Tells us where a user was at a certain point in time. It is just an observation from the dataset consisting of a *timestamp*, a *user identifier* as well as *latitude* and *longitude* coordinates. 


<span style="color:orange">&#9679;</span> **Stop location**: Tells us where and when a user has stopped (stayed). In addition to the position it has two additional variables: *t_start* and *t_end* indicating the start- and end-time of the stop. 

<a id='destination_definition'></a><span style="color:green">&#9679;</span> **Destination**: Aggregates stop locations in close proximity of each other and tells us where and how often a user has stopped there.  Like the position the destination consists of a *timestamp*, a *user identifier* as well as *latitude* and *longitude* coordinates. In addition, the *visitation count* variable denotes the number of stops at that particular destination and *cluster_assignment* identifies the destination. 

<figure class="left">
    <img src="/assets/images/2017-01-07-post-test/definitions.jpg"/> 
    <figcaption>**Figure 1**: *Defintions of the key concepts.*</figcaption>
</figure>

### The stop location algorithm explained
Below we have pictured a fictional user's positions with associated timestamps. The user is moving from left to right starting at 8:24. Between 8:26 and 8:46 the user moves less than 50 meters in 20 minutes. Thus, the algorithm detects that the user has stopped. It picks the [medoid](https://en.wikipedia.org/wiki/Medoid) (in orange) of those three points as the stop location (8:36). The start and end-times of the stop location are 8:26 and 8:46 respectively.

<figure class="left">
    <img src="/assets/images/2017-01-07-post-test/stoplocation_explanation3.jpg"/> 
    <figcaption>**Figure 2**: The stop detection algorithm explained.</figcaption>
</figure>

The algorithm has two parameters which are dependent on the application:

**Roaming distance**: defines how far a user's position is allowed to roam to still be conisdered part of the stop location (dashed orange line). 

**Stay duration**: specifies the minimum time a user has to stay within the roaming distance for his position to be considered as being part of a stop location.

### Extracting stop locations from the sample data
In the next cell we extract the stop locations from the sample dataset. We set the parameters `roaming_distance` to 50 meters and the `minimum_stay` parameter to 10 minutes.

I implemented the algorithm with parallel processing in mind for further speed gains. Therefore, there is a third parameter called `number_jobs` which specifies how many processors to use for parallel processing. For this example we set it to 2. If necessary change it in accordance to your speed requirements and hardware. See the [joblib documentation](https://pythonhosted.org/joblib/parallel.html#parallel-reference-documentation) for details.

Now let us run the cell below to extract the stop locations for each user! It should take less than 30 seconds to complete.


```python
# Parameters
roaming_distance = meters2degrees(50) # 50 meters converted to degrees
minimum_stay = 10  # minutes
number_jobs = 2 # number of parallel jobs

# Set index 
df = df.reset_index().set_index(["user_id", "timestamp"])

# Call helper-function to process entire df in one go
df_stops = process_data(df=df, 
                        roam_dist=roaming_distance, 
                        min_stay=minimum_stay, 
                        n_jobs=number_jobs,
                        print_output="notebook")
df_stops = pd.concat(df_stops)

# Only keep users with more than one stop
df_stops = (df_stops
    .groupby("user_id").filter(lambda x: len(x) > 1)
    .set_index(["user_id"]))

# Preview data
df_stops.iloc[:3,:]
print("Number of stop locations: {}".format(df_stops.shape[0]))
```

    Processing user 100 of 100.
    Number of stop locations: 2557


Congratulations! We have just successfully extracted 2557 stop locations from the data set. As you can see from the output above each stop location has now a start time and end time (`t_start`, `t_end`). Now, let us plot the stop locations on a map!


```python
fig_stops = plot_stops(df_stops.reset_index())
py.iplot(fig_stops, filename='fig_stops')
```




<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~public.sebastian/19.embed" height="340px" width="680px"></iframe>



Next, we will aggregate the stop locations into destinations.

## Aggregate stop locations into destinations <a id='cluster_stops'></a>

### Why do we need the destinations?
Recall that we defined the [destination](#destination_definition) as the aggregation of one or several stop locations that are in close proximity to each other. Now we want to aggregate the stop locations into destinations (figur 3).

<figure class="left">
    <img src="/assets/images/2017-01-07-post-test/locationdestination.jpg"/> 
    <figcaption>**Figure 3**: From stop locations to destinations.</figcaption>
</figure>

<figure class="right">
    <img src="/assets/images/2017-01-07-post-test/noclustering.jpg"/> 
    <figcaption>Figure 4.</figcaption>
</figure>
To appreciate why this is useful, let us have a look at the figure on the right where we have plotted some stop locations in orange (from a different dataset). The stop locations appear to form small clusters: one bigger cluster in proximity of *Building 7* on the left and two other clusters in proximity of *Building 1* and *Building 2* respectively. Finally, there are also two stop locations on *Malcolm Boulevard* at the bottom.

What we are actually interested in is the *destination* where a user has stopped and not neccessarily the exact *GPS position* recorded. For instance, the group of stop locations surrounding *Building 7* should be regarded as one destination. Similarly, those surrounding *Building 1* and *Building 2* respectively. In other words, we need to cluster or aggregate the stop locations into destinations.

To aggregate the stop locations into destinations we use [Scipy's hierarchical clustering functions](https://docs.scipy.org/doc/scipy/reference/cluster.hierarchy.html). For our application there are two important parameters: The [linkage parameter](https://docs.scipy.org/doc/scipy/reference/generated/scipy.cluster.hierarchy.linkage.html#scipy.cluster.hierarchy.linkage) defines how the clusters are formed. The [distance parameter](https://docs.scipy.org/doc/scipy/reference/generated/scipy.cluster.hierarchy.fcluster.html#scipy.cluster.hierarchy.fcluster) specifies within what distance stop locations are part of the same cluster. Let us briefly look at two linkage methods in practice.

The figure below shows the difference between using a complete linkage method and a centroid linkage method to form the clusters. For our purpose complete linkage forms too many small clusters. In contrast, centroid linkage seems to be just right forming approximately one cluster for each building. Of course, these parameters will have to be set based on the application at hand. Further information on linkage methods can be found [here](https://www.youtube.com/watch?v=VMyXc3SiEqs).
<figure class="left">
    <img src="/assets/images/2017-01-07-post-test/destinations_composite2.jpg"/> 
    <figcaption>**Figure 5:** A comparison of linkage methods for clustering GPS positions.</figcaption>
</figure>

### Running the clustering algorithm on the stop locations data
In the next cell we finally cluster the stop locations into destinatinos. We set the parameter `linkage method` to 'centroid' and the `distance` threshold to 100 meters.


```python
# Clustering parameters
linkage_method = 'centroid'
distance = meters2degrees(100)

# Cluster stoplocations on a per user basis
df_clusters = (df_stops.groupby('user_id')
              .apply(lambda x: 
                     cluster_stoplocations(x, 'centroid', distance))
              .reset_index())
# Export data
df_clusters.iloc[:3,:]
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>user_id</th>
      <th>timestamp</th>
      <th>latitude</th>
      <th>longitude</th>
      <th>t_start</th>
      <th>t_end</th>
      <th>cluster_assignment</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>165</td>
      <td>2008-02-05 13:22:00</td>
      <td>39.92888</td>
      <td>116.46690</td>
      <td>2008-02-05 13:22:00</td>
      <td>2008-02-05 13:32:00</td>
      <td>8</td>
    </tr>
    <tr>
      <th>1</th>
      <td>165</td>
      <td>2008-02-05 22:11:56</td>
      <td>40.16936</td>
      <td>116.73150</td>
      <td>2008-02-05 22:11:56</td>
      <td>2008-02-05 22:21:56</td>
      <td>3</td>
    </tr>
    <tr>
      <th>2</th>
      <td>165</td>
      <td>2008-02-03 05:06:10</td>
      <td>40.17049</td>
      <td>116.73527</td>
      <td>2008-02-03 01:36:12</td>
      <td>2008-02-03 09:06:09</td>
      <td>2</td>
    </tr>
  </tbody>
</table>
</div>



Now each stop locations has been assigned to a destination denoted by the new column `cluster_assignment`.

#### Get medoid for each destination

Now that we have assigned each stop location to a destination we want to have only one point representing each destination. To do this we calculate the [medoid](https://en.wikipedia.org/wiki/Medoid) of the stop locations within each destination. We also calculate the number of stop locations at each destination which is useful for ranking destinations based on visitation counts. 


```python
# Get medoid of each destination (cluster)
df_clustermedoids = (df_clusters.groupby('user_id')
    .apply(lambda x: get_clustermedoids(x))
    .reset_index(drop=True))

# Compute stop counts at each destination
df_clustersizes = (df_clusters
                   .groupby(['user_id', 'cluster_assignment'])
                   .apply(lambda x: len(x))
                   .reset_index(name='count'))

# Merge medoids and counts
df_destinations = pd.merge(df_clustermedoids.loc[:,['user_id', 'timestamp',
                                                 'latitude', 'longitude',
                                                 'cluster_assignment']], 
                             df_clustersizes, 
                             on=['user_id', 'cluster_assignment'], 
                             how='left')
# Export and preview data
df_destinations.iloc[:3,:]
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>user_id</th>
      <th>timestamp</th>
      <th>latitude</th>
      <th>longitude</th>
      <th>cluster_assignment</th>
      <th>count</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>165</td>
      <td>2008-02-08 12:07:13</td>
      <td>40.23100</td>
      <td>116.69343</td>
      <td>1</td>
      <td>5</td>
    </tr>
    <tr>
      <th>1</th>
      <td>165</td>
      <td>2008-02-06 00:41:55</td>
      <td>40.17044</td>
      <td>116.73519</td>
      <td>2</td>
      <td>7</td>
    </tr>
    <tr>
      <th>2</th>
      <td>165</td>
      <td>2008-02-05 22:11:56</td>
      <td>40.16936</td>
      <td>116.73150</td>
      <td>3</td>
      <td>1</td>
    </tr>
  </tbody>
</table>
</div>



Congratulations, we now have a new dataset `df_destinations` containing the destinations of each user and the visitation frequency. Let us plot our final result on a map.


```python
fig_destinations = plot_destinations(df_destinations)
py.iplot(fig_destinations, filename='fig_destinations')
```




<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~public.sebastian/21.embed" height="340px" width="680px"></iframe>



On the map you can appreciate the main destinations on a per-user basis the color intensity and the size indicate the visitation frequency (number of stop locations) at each destination.

### Bonus - destinations across all users

Instead of clustering the stop locations into destinations for each user we can also do that across all 100 users in the sample data. This allows us to see the top destinations across the city. Therefore, let us redo the clustering on all stops and then plot the results on a map. The next cell will take approximately 15 seconds to run.


```python
# Clustering parameters
linkage_method = 'centroid'
distance = meters2degrees(100)

# Cluster stoplocations on a per user basis
df_clusters_all = (cluster_stoplocations(df_stops, 'centroid', distance)
                   .reset_index())

# Get medoid of each destination (cluster)
df_clustermedoids_all = (get_clustermedoids(df_clusters_all)
                         .reset_index(drop=True))

# Compute stop counts at each destination
df_clustersizes_all = (df_clusters_all
                   .groupby(['cluster_assignment'])
                   .apply(lambda x: len(x))
                   .reset_index(name='count'))

# Merge medoids and counts
temp_cols = ['user_id', 'timestamp','latitude', 'longitude',
             'cluster_assignment']
df_destinations_all = pd.merge(df_clustermedoids_all.loc[:, temp_cols], 
                             df_clustersizes_all, 
                             on=['cluster_assignment'], 
                             how='left')
```


```python
fig_destinations_all = plot_destinations(
    df_destinations_all[df_destinations_all['count'] > 1])
py.iplot(fig_destinations_all, filename='fig_destinations')
```




<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~public.sebastian/21.embed" height="340px" width="680px"></iframe>



Well done! We now have plotted the most popular taxi destinations in Bejing based on visitation count. Next we will briefly look at some statistics and conclude this demonstration. 

<a id='finalthoughts'></a>
## Final thoughts

To sum up, we have loaded a GPS dataset, extracted the stop locations and clustered them into destinations. Now that we have this information we can do so much more. For instance we could calculate the [radius of gyration](https://en.wikipedia.org/wiki/Individual_mobility#Characteristics) of each user.


```python
# Cluster stoplocations on a per user basis
df_rgyration = (df_destinations
                .loc[:,['user_id', 'longitude', 'latitude', 'count']]
                .groupby('user_id')
                .apply(lambda x: rgiration_at_k(x, k=None))
                .reset_index())
df_rgyration.columns = ['user_id', 'radius_gyration']
df_rgyration.iloc[:3,:]
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>user_id</th>
      <th>radius_gyration</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>165</td>
      <td>14371.129878</td>
    </tr>
    <tr>
      <th>1</th>
      <td>185</td>
      <td>23379.968301</td>
    </tr>
    <tr>
      <th>2</th>
      <td>685</td>
      <td>35287.911599</td>
    </tr>
  </tbody>
</table>
</div>



Using the radius of gyration measure we could now start characterising the mobility patterns of our users more precisely... but let that be the topic for a new notebook. :-)

**Thanks for reading this far**. If you found this notebook useful feel free to [follow me on the web](http://www.sebastianbertoli.net/). Comments and feedback are always appreciated!

<a id='acknowledgments'></a>
## Acknowledgments
My thanks go to Marco de Nadai and Lorenzo Lucchini for their insights, guidance and code-fixes of the stop locaion algorithm and to the staff of the Bruno Kessler foundation for arranging the internship.

<a id='references'></a>
## References
<a id='hariharan2004'></a> [1] Hariharan R., Toyama K. (2004) [Project Lachesis: Parsing and Modeling Location Histories.](https://link.springer.com/chapter/10.1007/978-3-540-30231-5_8#citeas) In: Egenhofer M.J., Freksa C., Miller H.J. (eds) Geographic Information Science. GIScience 2004. Lecture Notes in Computer Science, vol 3234. Springer, Berlin, Heidelberg

[Back to top](#beginning)