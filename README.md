## To convert string to array type for genres
```
db.artists.find().forEach(function(doc) {
  var genresArray = doc.artist_genres.replace('[', '').replace(']', '').split(', ').map(function(item) {
    return item.replace(/'/g, '').trim();
  });

  db.artists.update(
    { "_id": doc._id },
    { "$set": { "artist_genres": genresArray } }
  );
}); 
```



## Query 1
#### Query: Style the top 10 artists with the highest average album popularity
```
db.albums.aggregate([ 
  {
    $match: {
      album_type: "album"
    }
  },
  {
    $group: {
      _id: "$artist_id",
      averageAlbumPopularity: { $avg: "$album_popularity" }
    }
  },
  {
    $lookup: {
      from: "artists",
      localField: "_id",
      foreignField: "id",
      as: "artistInfo"
    }
  },
  {
    $unwind: "$artistInfo"
  },
  {
    $project: {
      _id: 0,
      artist_id: "$_id",
      artist_name: "$artistInfo.name",
      averageAlbumPopularity: 1
    }
  },
  {
    $sort: {
      averageAlbumPopularity: -1
    }
  },
  {
    $limit: 10
  }
])
```
#### Result: 
```
{
  averageAlbumPopularity: 90,
  artist_name: 'Mitski'
}
{
  averageAlbumPopularity: 89.33962264150944,
  artist_name: 'Bad Bunny'
}
{
  averageAlbumPopularity: 88.11428571428571,
  artist_name: 'Harry Styles'
}
{
  averageAlbumPopularity: 86,
  artist_name: 'Troye Sivan'
}
{
  averageAlbumPopularity: 85,
  artist_name: 'Metro Boomin'
}
{
  averageAlbumPopularity: 84.72340425531915,
  artist_name: 'Imagine Dragons'
}
{
  averageAlbumPopularity: 84,
  artist_name: 'Olivia Rodrigo'
}
{
  averageAlbumPopularity: 84,
  artist_name: 'Zé Neto & Cristiano'
}
{
  averageAlbumPopularity: 82,
  artist_name: 'Paulo Londra'
}
{
  averageAlbumPopularity: 82,
  artist_name: 'Offset'
}
```
#### Interpreation: 







## Query 2
#### Query: Evolution of song's explicitness over the years (from 1960 on)
Indexes were created to speed up a such heavy query
```
db.tracks.createIndex({ "id": 1 })
db.features.createIndex({ "id": 1 })
db.albums.createIndex({ "track_id": 1 })
db.albums.createIndex({ "release_date": 1 })
```

To check that indexes were actually used, after the aggregate operation:
```
.explain("executionStats");
```
And the debug was successful. Without indexes the result couldn't be observed due to the long execution time needed.
```
db.tracks.aggregate([
  {
    $lookup: {
      from: "features",
      localField: "id",
      foreignField: "id",
      as: "track_features"
    }
  },
  {
    $unwind: "$track_features"
  },
  {
    $lookup: {
      from: "albums",
      localField: "id", 
      foreignField: "track_id",
      as: "album_info"
    }
  },
  {
    $unwind: "$album_info"
  },
  {
    $match: {
      "album_info.release_date": { $gte: ISODate("1960-01-01T00:00:00.000Z") }
    }
  },
  {
    $group: {
      _id: {
        year: { $year: "$album_info.release_date" },
        explicit: "$explicit"
      },
      totalTracks: { $sum: 1 }
    }
  },
  {
    $group: {
      _id: "$_id.year",
      explicitCount: {
        $sum: { $cond: [{ $eq: ["$_id.explicit", true] }, "$totalTracks", 0] }
      },
      implicitCount: {
        $sum: { $cond: [{ $eq: ["$_id.explicit", false] }, "$totalTracks", 0] }
      },
      totalTracks: { $sum: "$totalTracks" }
    }
  },
  {
    $project: {
      _id: 0,
      year: "$_id",
      explicitPercentage: {
        $multiply: [
          { $divide: ["$explicitCount", "$totalTracks"] },
          100
        ]
      },
      implicitPercentage: {
        $multiply: [
          { $divide: ["$implicitCount", "$totalTracks"] },
          100
        ]
      }
    }
  },
  {
    $sort: { year: 1 } 
  }
])

```
#### Result: 
A (very long) result is shown to prove the correctness of the query. A better view can be seen with the plot showing the trend of explicitness increase in the last decades.
```
[{
    "year": 1960,
  "explicitPercentage": 0,
  "implicitPercentage": 100
},
{
    "year": 1961,
  "explicitPercentage": 0,
  "implicitPercentage": 100
},
{
 
  "year": 1962,
  "explicitPercentage": 0,
  "implicitPercentage": 100
},
{
    "year": 1963,
  "explicitPercentage": 0,
  "implicitPercentage": 100
},
{
    "year": 1964,
  "explicitPercentage": 0,
  "implicitPercentage": 100
},
{
    "year": 1965,
  "explicitPercentage": 0,
  "implicitPercentage": 100
},
{
    "year": 1966,
  "explicitPercentage": 0,
  "implicitPercentage": 100
},
{
    "year": 1967,
  "explicitPercentage": 0,
  "implicitPercentage": 100
},
{
    "year": 1968,
  "explicitPercentage": 0,
  "implicitPercentage": 100
},
{
 
  "year": 1969,
  "explicitPercentage": 0,
  "implicitPercentage": 100
},
{
    "year": 1970,
  "explicitPercentage": 0,
  "implicitPercentage": 100
},
{
 
  "year": 1971,
  "explicitPercentage": 0.16260162601626016,
  "implicitPercentage": 99.83739837398375
},
{
 
  "year": 1972,
  "explicitPercentage": 0,
  "implicitPercentage": 100
},
{
    "year": 1973,
  "explicitPercentage": 0.6201550387596899,
  "implicitPercentage": 99.37984496124031
},
{
    "year": 1974,
  "explicitPercentage": 0,
  "implicitPercentage": 100
},
{
    "year": 1975,
  "explicitPercentage": 0.2958579881656805,
  "implicitPercentage": 99.70414201183432
},
{
 
  "year": 1976,
  "explicitPercentage": 0,
  "implicitPercentage": 100
},
{
    "year": 1977,
  "explicitPercentage": 0,
  "implicitPercentage": 100
},
{
    "year": 1978,
  "explicitPercentage": 0.19880715705765406,
  "implicitPercentage": 99.80119284294234
},
{
    "year": 1979,
  "explicitPercentage": 0.42194092827004215,
  "implicitPercentage": 99.57805907172997
},
{
    "year": 1980,
  "explicitPercentage": 0.7334963325183375,
  "implicitPercentage": 99.26650366748166
},
{
 
  "year": 1981,
  "explicitPercentage": 0.33167495854063017,
  "implicitPercentage": 99.66832504145937
},
{
,
  "year": 1982,
  "explicitPercentage": 0.1303780964797914,
  "implicitPercentage": 99.86962190352021
},
{

  "year": 1983,
  "explicitPercentage": 0,
  "implicitPercentage": 100
},
{

  "year": 1984,
  "explicitPercentage": 2.635431918008785,
  "implicitPercentage": 97.36456808199122
},
{

  "year": 1985,
  "explicitPercentage": 2.2530329289428077,
  "implicitPercentage": 97.74696707105718
},
{

  "year": 1986,
  "explicitPercentage": 0.2638522427440633,
  "implicitPercentage": 99.73614775725594
},
{

  "year": 1987,
  "explicitPercentage": 0.7035175879396985,
  "implicitPercentage": 99.2964824120603
},
{

  "year": 1988,
  "explicitPercentage": 3.329369797859691,
  "implicitPercentage": 96.6706302021403
},
{

  "year": 1989,
  "explicitPercentage": 3.423236514522822,
  "implicitPercentage": 96.57676348547717
},
{

  "year": 1990,
  "explicitPercentage": 1.5527950310559007,
  "implicitPercentage": 98.4472049689441
},
{

  "year": 1991,
  "explicitPercentage": 4.265997490589712,
  "implicitPercentage": 95.73400250941029
},
{

  "year": 1992,
  "explicitPercentage": 3.720405862457723,
  "implicitPercentage": 96.27959413754228
},
{

  "year": 1993,
  "explicitPercentage": 4.915992532669571,
  "implicitPercentage": 95.08400746733044
},
{

  "year": 1994,
  "explicitPercentage": 4.438860971524288,
  "implicitPercentage": 95.56113902847572
},
{

  "year": 1995,
  "explicitPercentage": 2.535211267605634,
  "implicitPercentage": 97.46478873239437
},
{

  "year": 1996,
  "explicitPercentage": 8.294693456980937,
  "implicitPercentage": 91.70530654301906
},
{

  "year": 1997,
  "explicitPercentage": 8.265342319971621,
  "implicitPercentage": 91.73465768002838
},
{

  "year": 1998,
  "explicitPercentage": 10.131826741996234,
  "implicitPercentage": 89.86817325800376
},
{

  "year": 1999,
  "explicitPercentage": 5.842575060860157,
  "implicitPercentage": 94.15742493913984
},
{

  "year": 2000,
  "explicitPercentage": 8.64406779661017,
  "implicitPercentage": 91.35593220338983
},
{

  "year": 2001,
  "explicitPercentage": 5.629669156883671,
  "implicitPercentage": 94.37033084311632
},
{

  "year": 2002,
  "explicitPercentage": 7.041587901701324,
  "implicitPercentage": 92.95841209829868
},
{

  "year": 2003,
  "explicitPercentage": 6.454047933572372,
  "implicitPercentage": 93.54595206642763
},
{

  "year": 2004,
  "explicitPercentage": 7.8275134755097255,
  "implicitPercentage": 92.17248652449027
},
{

  "year": 2005,
  "explicitPercentage": 8.721965112139552,
  "implicitPercentage": 91.27803488786044
},
{

  "year": 2006,
  "explicitPercentage": 7.350402761795167,
  "implicitPercentage": 92.64959723820483
},
{

  "year": 2007,
  "explicitPercentage": 5.678690424453137,
  "implicitPercentage": 94.32130957554686
},
{

  "year": 2008,
  "explicitPercentage": 6.11358574610245,
  "implicitPercentage": 93.88641425389756
},
{

  "year": 2009,
  "explicitPercentage": 6.075922142165824,
  "implicitPercentage": 93.92407785783418
},
{
  "year": 2010,
  "explicitPercentage": 5.746697514739906,
  "implicitPercentage": 94.2533024852601
},
{
  "year": 2011,
  "explicitPercentage": 6.576110392410523,
  "implicitPercentage": 93.42388960758949
},
{
  "year": 2012,
  "explicitPercentage": 7.42041333256577,
  "implicitPercentage": 92.57958666743423
},
{
  "year": 2013,
  "explicitPercentage": 9.112736612876207,
  "implicitPercentage": 90.8872633871238
},
{
  "year": 2014,
  "explicitPercentage": 9.137893116340443,
  "implicitPercentage": 90.86210688365955
},
{
  "year": 2015,
  "explicitPercentage": 9.559696178495134,
  "implicitPercentage": 90.44030382150487
},
{
  "year": 2016,
  "explicitPercentage": 12.438013721893892,
  "implicitPercentage": 87.5619862781061
},
{
  "year": 2017,
  "explicitPercentage": 14.558927695744565,
  "implicitPercentage": 85.44107230425544
},
{
  "year": 2018,
  "explicitPercentage": 16.986899563318776,
  "implicitPercentage": 83.01310043668121
},
{
  "year": 2019,
  "explicitPercentage": 19.069621388839046,
  "implicitPercentage": 80.93037861116096
},
{
  "year": 2020,
  "explicitPercentage": 22.754130842444184,
  "implicitPercentage": 77.24586915755582
},
{
  "year": 2021,
  "explicitPercentage": 22.987689393939394,
  "implicitPercentage": 77.01231060606061
},
{
  "year": 2022,
  "explicitPercentage": 23.10514041395394,
  "implicitPercentage": 76.89485958604607
},
{
  "year": 2023,
  "explicitPercentage": 24.6222538083614,
  "implicitPercentage": 75.3777461916386
}]
```

![alt text](plt.png "Title")
#### Interpreation: 




## Query 3
#### Query: based on the time signature, get the average instrumentalness of the tracks
```
db.features.aggregate([
  {
    $group: {
      _id: "$time_signature",
      average_instrumentalness: { $avg: "$instrumentalness" }
    }
  },
  {
    $sort: {
      _id: 1
    }
  },
  {
    $project: {
      _id: 0,
      time_signature: "$_id",
      average_instrumentalness: 1
    }
  }
])
```
#### Result: 
```
{
  average_instrumentalness: 0.3384449754709141,
  time_signature: 0
}
{
  average_instrumentalness: 0.3533327869221757,
  time_signature: 1
}
{
  average_instrumentalness: 0.3154482332345381,
  time_signature: 3
}
{
  average_instrumentalness: 0.25649063421458285,
  time_signature: 4
}
{
  average_instrumentalness: 0.3072569320669578,
  time_signature: 5
}

## Query 4: find artists popular only in few albums
Useful for instance if you want to find underground artists that collaborate with big stars.Very likely in those album the one with low popularity will have a big album popularity in that album. Useful also to spot artists that, for instance, wrote a summer-hit but for the rest of the year no one listen to him. The query works by finding the stardard deviation of the artist's albums popularity. Artists that only wrote 2 albums were discarted, as well as artist with low popularity (<50) to spot artists that we may know!
```
db.artists.aggregate([
  {
    $lookup: {
      from: "albums",
      localField: "id",
      foreignField: "artist_id",
      as: "albums"
    }
  },
  {
    $unwind: "$albums"
  },
  {
    $group: {
      _id: "$id",
      name: { $first: "$name" },
      popularityArray: { $push: "$albums.album_popularity" },
      overall_popularity: { $first: "$artist_popularity" }
    }
  },
  {
    $project: {
      _id: 1,
      name: 1,
      overall_popularity: 1,
      coefficientOfVariation: {
        $cond: {
          if: { $eq: [{ $avg: "$popularityArray" }, 0] },
          then: 0,
          else: {
            $divide: [
              { $stdDevSamp: "$popularityArray" },
              { $avg: "$popularityArray" }
            ]
          }
        }
      },
      total_albums: { $size: "$popularityArray" }
    }
  },
  {
    $match: {
      total_albums: { $gt: 2 }, 
      overall_popularity: { $gte: 50 }
    }
  },
  {
    $sort: { coefficientOfVariation: -1 }
  },
  {
    $limit: 200
  }
])
```
#### Result: 
```
{'_id': '0GQkTwFb7D3ePIpnxwYavf', 'name': 'Done Again', 'overall_popularity': 40, 'coefficientOfVariation': 6.085199811164736, 'total_albums': 710}
{'_id': '28Y5nsvbE8IdoUAGNgCk0Y', 'name': 'Marsha', 'overall_popularity': 42, 'coefficientOfVariation': 4.123105625617661, 'total_albums': 17}
{'_id': '4CDMSd6tmH66svpunz3aWP', 'name': 'Gerald Finzi', 'overall_popularity': 42, 'coefficientOfVariation': 3.8729833462074166, 'total_albums': 15}
{'_id': '00wclaNli3JTTLho4LPjBF', 'name': 'breezy brooks', 'overall_popularity': 51, 'coefficientOfVariation': 3.283041725144403, 'total_albums': 314}
{'_id': '0GM7qgcRCORpGnfcN2tCiB', 'name': 'Tainy', 'overall_popularity': 77, 'coefficientOfVariation': 2.775648839312855, 'total_albums': 23}
{'_id': '2QFXAOEj2ow8a3xVkD8Ntg', 'name': 'Ian Carey', 'overall_popularity': 41, 'coefficientOfVariation': 2.145911990647552, 'total_albums': 19}
{'_id': '2d09AaNvj1TRW0GociCEDY', 'name': 'Demi Femme', 'overall_popularity': 40, 'coefficientOfVariation': 2.006283636074083, 'total_albums': 10}
{'_id': '1SAugjIcuwNPKS4urSB7A6', 'name': 'Joe Budden', 'overall_popularity': 46, 'coefficientOfVariation': 2.0, 'total_albums': 4}
{'_id': '1qSJaftSab2kTTsj7fLxvM', 'name': 'DJ SpinKing', 'overall_popularity': 47, 'coefficientOfVariation': 1.8780179725759258, 'total_albums': 4}
{'_id': '0f1IECbrVV952unZkzrsg2', 'name': 'Mc Gw', 'overall_popularity': 76, 'coefficientOfVariation': 1.8715688828099524, 'total_albums': 9}
{'_id': '5MAp6rMiUJjRLXMWtArXRS', 'name': 'Dorrough Music', 'overall_popularity': 44, 'coefficientOfVariation': 1.7659154284040566, 'total_albums': 4}
{'_id': '6ugw7JCu0AG7txRcRAxU8d', 'name': 'Mc Rd', 'overall_popularity': 61, 'coefficientOfVariation': 1.7320508075688776, 'total_albums': 3}
{'_id': '2JSjCHK79gdaiPWdKiNUNp', 'name': 'Dionne Warwick', 'overall_popularity': 59, 'coefficientOfVariation': 1.732050807568877, 'total_albums': 3}
{'_id': '3IZrrNonYELubLPJmqOci2', 'name': 'Nancy Sinatra', 'overall_popularity': 62, 'coefficientOfVariation': 1.5696710443592952, 'total_albums': 3}
{'_id': '6diqrwyTlcmngrGatZc08b', 'name': 'Ray Mak', 'overall_popularity': 41, 'coefficientOfVariation': 1.5434342355060342, 'total_albums': 12}
{'_id': '2pywmTkxO0H1CY8ZXSJTSC', 'name': 'Neelkamal Singh', 'overall_popularity': 65, 'coefficientOfVariation': 1.5143755588800727, 'total_albums': 4}
{'_id': '0U8dIwzBn17JkhYxmznp6T', 'name': 'Jayrick', 'overall_popularity': 55, 'coefficientOfVariation': 1.499536106375175, 'total_albums': 12}
{'_id': '4oRyLnxDdIzd2POQfX9Drd', 'name': 'Piano Dreamers', 'overall_popularity': 45, 'coefficientOfVariation': 1.4967228844190756, 'total_albums': 1247}
{'_id': '5sK8BsvyDl4TFA6KaBf8or', 'name': 'Charly Black', 'overall_popularity': 58, 'coefficientOfVariation': 1.4351278119856414, 'total_albums': 3}
{'_id': '6DVipHzYsPlIoA0DW8Gmns', 'name': 'Royce Da 5\'9"', 'overall_popularity': 55, 'coefficientOfVariation': 1.428782740077439, 'total_albums': 5}
{'_id': '0Cs47vvRsPgEfliBU9KDiB', 'name': 'D.O.D', 'overall_popularity': 66, 'coefficientOfVariation': 1.4072912811497131, 'total_albums': 3}
{'_id': '7aBzpmFXB4WWpPl2F7RjBe', 'name': 'Wyclef Jean', 'overall_popularity': 68, 'coefficientOfVariation': 1.405502855249306, 'total_albums': 5}
{'_id': '0sLb0ouettR8lDLnEgCSVK', 'name': 'Living Room', 'overall_popularity': 54, 'coefficientOfVariation': 1.3732131246511905, 'total_albums': 15}
{'_id': '6gbGGM0E8Q1hE511psqxL0', 'name': 'Ray J', 'overall_popularity': 51, 'coefficientOfVariation': 1.3693063937629153, 'total_albums': 15}
{'_id': '1JJzFmhpsWbTRNEApLEzgW', 'name': 'RavilZ', 'overall_popularity': 40, 'coefficientOfVariation': 1.360897063089832, 'total_albums': 3}
{'_id': '4GMgdB3vwbBOc42hbXEi9p', 'name': 'N.O.R.E.', 'overall_popularity': 52, 'coefficientOfVariation': 1.35400640077266, 'total_albums': 4}
{'_id': '1zrFFDzoE9XXyjEqqgDpMm', 'name': 'Dr Zeus', 'overall_popularity': 56, 'coefficientOfVariation': 1.3428790339129553, 'total_albums': 3}
{'_id': '6pAwHPeExeUbMd5w7Iny6D', 'name': 'Richard Strauss', 'overall_popularity': 51, 'coefficientOfVariation': 1.3381695237968192, 'total_albums': 44}
{'_id': '5oNgAs7j5XcBMzWv3HAnHG', 'name': 'DJ Drama', 'overall_popularity': 58, 'coefficientOfVariation': 1.3293185425231797, 'total_albums': 15}
{'_id': '2ThUOWiiWP3YdZqs4WYNOi', 'name': 'Smooth Jazz All Stars', 'overall_popularity': 41, 'coefficientOfVariation': 1.3291052646628143, 'total_albums': 823}
{'_id': '6If57j6e3TXXk0HiLcIZca', 'name': 'Sevyn Streeter', 'overall_popularity': 54, 'coefficientOfVariation': 1.2933068447274447, 'total_albums': 13}
{'_id': '5Berubt6ysOy2LCMyqhmXP', 'name': 'YFN Lucci', 'overall_popularity': 54, 'coefficientOfVariation': 1.2836665240232603, 'total_albums': 5}
{'_id': '3nwYsifpwrKmCIpw4i0HDW', 'name': 'Konshens', 'overall_popularity': 62, 'coefficientOfVariation': 1.2727272727272727, 'total_albums': 4}
{'_id': '6i1qu6ITcSL2Ss6qr7Nzkn', 'name': 'Kidzone', 'overall_popularity': 43, 'coefficientOfVariation': 1.2624896226537887, 'total_albums': 149}
{'_id': '2mxe0TnaNL039ysAj51xPQ', 'name': 'R. Kelly', 'overall_popularity': 64, 'coefficientOfVariation': 1.2264400208137942, 'total_albums': 3}
{'_id': '76PJKS3IQsf4sSayx2taE0', 'name': 'The Hit Crew', 'overall_popularity': 45, 'coefficientOfVariation': 1.206249727960333, 'total_albums': 188}
{'_id': '5TVRuxsMCXb9bBuUhe484o', 'name': 'Inder Arya', 'overall_popularity': 50, 'coefficientOfVariation': 1.190408320573755, 'total_albums': 57}
{'_id': '6Xgp2XMz1fhVYe7i6yNAax', 'name': 'Trippie Redd', 'overall_popularity': 79, 'coefficientOfVariation': 1.1740668394254867, 'total_albums': 6}
{'_id': '0NbfKEOTQCcwd6o7wSDOHI', 'name': 'The Game', 'overall_popularity': 70, 'coefficientOfVariation': 1.1628654833100662, 'total_albums': 13}
{'_id': '3ddw8VOjGZrR2G6dFCjamb', 'name': 'Cardi Mist', 'overall_popularity': 47, 'coefficientOfVariation': 1.154140347897691, 'total_albums': 23}
{'_id': '2YzXsQoI3rqYNEVd4nac7g', 'name': 'Ultra Beats', 'overall_popularity': 53, 'coefficientOfVariation': 1.1456439237389602, 'total_albums': 3}
{'_id': '34R6DQd8ErBy1xyOyMHFrq', 'name': 'Sachin Gupta', 'overall_popularity': 48, 'coefficientOfVariation': 1.1425551151779059, 'total_albums': 48}
{'_id': '2CgVSpL4tfbUuHmTGS7wF3', 'name': 'Ocean Waves For Sleep', 'overall_popularity': 63, 'coefficientOfVariation': 1.124297533493792, 'total_albums': 90}
{'_id': '3jksrX4oBklxR78ft8gv3j', 'name': 'Plies', 'overall_popularity': 60, 'coefficientOfVariation': 1.1182926774314021, 'total_albums': 3}
{'_id': '5Bw9kFNhy019e4IBCJZlzw', 'name': 'Jordana', 'overall_popularity': 55, 'coefficientOfVariation': 1.1090536506409416, 'total_albums': 6}
{'_id': '2Di8r9df6xjyj6CVOqbGVz', 'name': 'Marshall Jefferson', 'overall_popularity': 48, 'coefficientOfVariation': 1.10798218182113, 'total_albums': 10}
{'_id': '6RM2eUH2grJfd6gkRgiRXG', 'name': 'Piano Mage', 'overall_popularity': 45, 'coefficientOfVariation': 1.0979691854418916, 'total_albums': 6}
{'_id': '7sBx432MZn1MzHeYHAA5qr', 'name': 'Scott Hamilton', 'overall_popularity': 44, 'coefficientOfVariation': 1.0815397404103666, 'total_albums': 35}
{'_id': '0eVyjRhzZKke2KFYTcDkeu', 'name': 'The Alchemist', 'overall_popularity': 69, 'coefficientOfVariation': 1.0776416767445998, 'total_albums': 8}
{'_id': '7K78lVZ8XzkjfRSI7570FF', 'name': 'José Feliciano', 'overall_popularity': 72, 'coefficientOfVariation': 1.067362151551925, 'total_albums': 47}
{'_id': '5yCd7bxcAc3MdQ1h54ESsD', 'name': 'Houston', 'overall_popularity': 40, 'coefficientOfVariation': 1.066932012315654, 'total_albums': 36}
{'_id': '0tJCNteqwm7LmRZ6KWr8GT', 'name': 'Chip', 'overall_popularity': 52, 'coefficientOfVariation': 1.0537322062554604, 'total_albums': 3}
{'_id': '2iE18Oxc8YSumAU232n4rW', 'name': 'The Jackson 5', 'overall_popularity': 71, 'coefficientOfVariation': 1.0523676206400019, 'total_albums': 24}
{'_id': '3p2pMpzDerhMR4w2xZyHWg', 'name': 'Rosenfeld', 'overall_popularity': 54, 'coefficientOfVariation': 1.0513149660756935, 'total_albums': 3}
{'_id': '7uUBTiZ2u5b40vymlFmXrn', 'name': 'George Shearing', 'overall_popularity': 41, 'coefficientOfVariation': 1.0452034510055521, 'total_albums': 476}
{'_id': '430byzy0c5bPn5opiu0SRd', 'name': 'Edward Elgar', 'overall_popularity': 56, 'coefficientOfVariation': 1.0415994745386914, 'total_albums': 55}
{'_id': '087ZBrJyDhTGPwPsOFXXqj', 'name': 'ONY9RMX', 'overall_popularity': 40, 'coefficientOfVariation': 1.0338619460140595, 'total_albums': 4}
{'_id': '5aIqB5nVVvmFsvSdExz408', 'name': 'Johann Sebastian Bach', 'overall_popularity': 76, 'coefficientOfVariation': 1.032957075492959, 'total_albums': 2883}
{'_id': '7vUJCRmF1if4uhMp2V3tRP', 'name': 'Arman Cekin', 'overall_popularity': 45, 'coefficientOfVariation': 1.0298680477436568, 'total_albums': 3}
{'_id': '5ZP65l78WPZimUs9LVvMW0', 'name': 'Pozle', 'overall_popularity': 41, 'coefficientOfVariation': 1.023532631438318, 'total_albums': 22}
{'_id': '3AoiBjr0pSGswX96XxEgH7', 'name': 'Guitar Tribute Players', 'overall_popularity': 48, 'coefficientOfVariation': 1.0213535138607068, 'total_albums': 618}
{'_id': '0YOr5sV4zMMyj5xviWiFjW', 'name': 'Chris Beats Zn', 'overall_popularity': 54, 'coefficientOfVariation': 1.0183501544346312, 'total_albums': 3}
{'_id': '2gClsBep1tt1rv1CN210SO', 'name': 'Gabriel Fauré', 'overall_popularity': 59, 'coefficientOfVariation': 1.0181760124203896, 'total_albums': 79}
{'_id': '7iUy9ctasMRtp5jwWs95Fm', 'name': 'LO-FI BEATS', 'overall_popularity': 46, 'coefficientOfVariation': 1.0148106034274749, 'total_albums': 200}
{'_id': '1DptMAbvFajxK4ZObg1OdA', 'name': 'Éric Serra', 'overall_popularity': 45, 'coefficientOfVariation': 1.0051414220648331, 'total_albums': 98}
{'_id': '3nRCmlti96JKj7IwKFA93r', 'name': 'White Noise Relaxation', 'overall_popularity': 41, 'coefficientOfVariation': 1.0050378152592119, 'total_albums': 100}
{'_id': '2RCefexnr0KGbN7ddvuwNE', 'name': '(((())))', 'overall_popularity': 51, 'coefficientOfVariation': 0.9948090120290434, 'total_albums': 3}
{'_id': '2KsP6tYLJlTBvSUxnwlVWa', 'name': 'Mike Posner', 'overall_popularity': 67, 'coefficientOfVariation': 0.9930506454796424, 'total_albums': 4}
{'_id': '58xmt13Xf7RsThzGOM1aKh', 'name': 'Travis Atreo', 'overall_popularity': 40, 'coefficientOfVariation': 0.9751494098207033, 'total_albums': 76}
{'_id': '3Fl1V19tmjt57oBdxXKAjJ', 'name': 'Blueface', 'overall_popularity': 65, 'coefficientOfVariation': 0.9662827003234844, 'total_albums': 5}
{'_id': '3XcCT5MPlQPWFTJyzXbfuX', 'name': 'Maejor', 'overall_popularity': 56, 'coefficientOfVariation': 0.9662404468922987, 'total_albums': 4}
{'_id': '6pMR8Zgot664613rAiLC2Z', 'name': 'AGNEZ MO', 'overall_popularity': 41, 'coefficientOfVariation': 0.9643650760992954, 'total_albums': 3}
{'_id': '4TkCMPznXOjlsYLfzIU1rw', 'name': 'Geek Music', 'overall_popularity': 59, 'coefficientOfVariation': 0.9602875265152125, 'total_albums': 12}
{'_id': '4JAIvx8vd1sMssmNTcwnPX', 'name': 'The Remix Station', 'overall_popularity': 44, 'coefficientOfVariation': 0.9587031225711359, 'total_albums': 34}
{'_id': '13y7CgLHjMVRMDqxdx0Xdo', 'name': 'Gucci Mane', 'overall_popularity': 75, 'coefficientOfVariation': 0.9515464417240825, 'total_albums': 93}
{'_id': '2w9zwq3AktTeYYMuhMjju8', 'name': 'INNA', 'overall_popularity': 67, 'coefficientOfVariation': 0.9481105157866451, 'total_albums': 21}
{'_id': '4Xx6QMLTWppMwdABkN0Afj', 'name': 'Piano Tribute Players', 'overall_popularity': 50, 'coefficientOfVariation': 0.9462297051502558, 'total_albums': 616}
{'_id': '7KkUirCiJZhgRN3NbgG98L', 'name': 'Ottorino Respighi', 'overall_popularity': 50, 'coefficientOfVariation': 0.945984335830337, 'total_albums': 45}
{'_id': '6gssIbF04dCX3COZvyr0JF', 'name': 'Bruno Furlan', 'overall_popularity': 43, 'coefficientOfVariation': 0.9447549859466604, 'total_albums': 4}
{'_id': '0Kekt6CKSo0m5mivKcoH51', 'name': 'Sergei Rachmaninoff', 'overall_popularity': 62, 'coefficientOfVariation': 0.9431688001817823, 'total_albums': 177}
{'_id': '09ao6AC5gW8AxRBUVkqWIB', 'name': 'Alexia', 'overall_popularity': 41, 'coefficientOfVariation': 0.9406586964690676, 'total_albums': 57}
{'_id': '1cNDP5yjU5vjeR8qMf4grg', 'name': 'YNW Melly', 'overall_popularity': 71, 'coefficientOfVariation': 0.9327261582020215, 'total_albums': 3}
{'_id': '5KWOCu1saEHAhPiLKlOLIy', 'name': 'Leo', 'overall_popularity': 52, 'coefficientOfVariation': 0.9326427425370879, 'total_albums': 3}
{'_id': '72nLe76yBFSlP6VBzME358', 'name': 'SHAUN', 'overall_popularity': 56, 'coefficientOfVariation': 0.9326427425370879, 'total_albums': 3}
{'_id': '22PU1TSXSiqGGk4mSVMxYj', 'name': 'Sam James', 'overall_popularity': 42, 'coefficientOfVariation': 0.9326427425370879, 'total_albums': 3}
{'_id': '5TfnQ0Ai1cEbKY5katFK14', 'name': 'Ladyhawke', 'overall_popularity': 41, 'coefficientOfVariation': 0.9320374863587596, 'total_albums': 28}
{'_id': '6f4XkbvYlXMH0QgVRzW0sM', 'name': 'Waka Flocka Flame', 'overall_popularity': 63, 'coefficientOfVariation': 0.9290690397878223, 'total_albums': 3}
{'_id': '6F3vLfyutkUhpM50G84eMt', 'name': 'Endor', 'overall_popularity': 54, 'coefficientOfVariation': 0.9207338730881488, 'total_albums': 5}
{'_id': '656RXuyw7CE0dtjdPgjJV6', 'name': 'Joseph Haydn', 'overall_popularity': 54, 'coefficientOfVariation': 0.9150122490548046, 'total_albums': 254}
{'_id': '47SxJ59gTjhzswBGMkwWfz', 'name': 'ROSE BEAT', 'overall_popularity': 50, 'coefficientOfVariation': 0.9145291883606759, 'total_albums': 6}
{'_id': '77ehFS1P2bU6Bfcs1qu6Jd', 'name': 'Regi', 'overall_popularity': 44, 'coefficientOfVariation': 0.9128709291752769, 'total_albums': 4}
{'_id': '2LmyJyCF5V1eQyvHgJNbTn', 'name': 'Leonard Bernstein', 'overall_popularity': 60, 'coefficientOfVariation': 0.907836620036756, 'total_albums': 255}
{'_id': '7JIRtB4hn75Md3nPGzeRIL', 'name': 'Sam Smyers', 'overall_popularity': 44, 'coefficientOfVariation': 0.8977116357836274, 'total_albums': 12}
{'_id': '23XJNT1Hb35h3ZCDl7lpWY', 'name': 'Miguel Aceves Mejia', 'overall_popularity': 48, 'coefficientOfVariation': 0.8890755610656017, 'total_albums': 754}
{'_id': '6L1XC7NrmgWRlwAeLJvVtA', 'name': 'Sam Fischer', 'overall_popularity': 61, 'coefficientOfVariation': 0.8849315029548596, 'total_albums': 8}
{'_id': '7HBA3bLuJTLRvjK8NX9ZSy', 'name': 'Sweet Little Band', 'overall_popularity': 52, 'coefficientOfVariation': 0.879277446679218, 'total_albums': 806}
{'_id': '3EYY5FwDkHEYLw5V86SAtl', 'name': 'SICK LEGEND', 'overall_popularity': 66, 'coefficientOfVariation': 0.8784076502399099, 'total_albums': 3}
{'_id': '6jEiUoyyJNPHzSR0Nib6HX', 'name': 'Walk off the Earth', 'overall_popularity': 60, 'coefficientOfVariation': 0.8717797887081348, 'total_albums': 3}
{'_id': '4kCZ5nyurc9eIqLJfUcW0Y', 'name': 'Anonymous', 'overall_popularity': 58, 'coefficientOfVariation': 0.869965461231079, 'total_albums': 221}
{'_id': '4mwXUEKaW4ftbncf9Hi58l', 'name': 'Tank', 'overall_popularity': 54, 'coefficientOfVariation': 0.8676603408708855, 'total_albums': 4}
{'_id': '1Zatb2YN4erBOoSivOXc0o', 'name': 'O.T. Genasis', 'overall_popularity': 50, 'coefficientOfVariation': 0.8620799372342556, 'total_albums': 6}
{'_id': '62TD7509VQIxUe4WpwO0s3', 'name': 'Johann Pachelbel', 'overall_popularity': 56, 'coefficientOfVariation': 0.8618793784500668, 'total_albums': 194}
{'_id': '5HONdRTLNvBjlD2LirKp0q', 'name': 'Maxwell Young', 'overall_popularity': 42, 'coefficientOfVariation': 0.8591794631192843, 'total_albums': 58}
{'_id': '7M1FPw29m5FbicYzS2xdpi', 'name': 'King Crimson', 'overall_popularity': 52, 'coefficientOfVariation': 0.856713302668477, 'total_albums': 4}
{'_id': '75pnTZQozf5CtkbWxmUtCf', 'name': 'Joyce Trio', 'overall_popularity': 41, 'coefficientOfVariation': 0.8531692254604166, 'total_albums': 23}
{'_id': '2qAwMsiIjTzlmfAkXKvhVA', 'name': 'Armani White', 'overall_popularity': 60, 'coefficientOfVariation': 0.8531038622665893, 'total_albums': 5}
{'_id': '1zo2ucFhzu58hKcniMpSQs', 'name': 'Soley', 'overall_popularity': 50, 'coefficientOfVariation': 0.8523439991716134, 'total_albums': 8}
{'_id': '436sYg6CZhNefQJogaXeK0', 'name': 'Camille Saint-Saëns', 'overall_popularity': 63, 'coefficientOfVariation': 0.8481740950443286, 'total_albums': 293}
{'_id': '7kFfY4UjNdNyaeUgLIEbIF', 'name': 'DeJ Loaf', 'overall_popularity': 62, 'coefficientOfVariation': 0.8476195931019601, 'total_albums': 3}
{'_id': '6STjC3QJTieuM5WHHtkGuh', 'name': 'Raven & Kreyn', 'overall_popularity': 41, 'coefficientOfVariation': 0.8410698320255681, 'total_albums': 31}
{'_id': '0NIIxcxNHmOoyBx03SfTCD', 'name': 'Tinashe', 'overall_popularity': 65, 'coefficientOfVariation': 0.8406019094957695, 'total_albums': 4}
{'_id': '4gB2Nnsapxi4chX9f5zgku', 'name': 'Skooly', 'overall_popularity': 41, 'coefficientOfVariation': 0.8398647280065034, 'total_albums': 36}
{'_id': '2cADQgiLMjNhbsfeN52Bf3', 'name': 'Swizz Beatz', 'overall_popularity': 60, 'coefficientOfVariation': 0.8386222017284267, 'total_albums': 3}
{'_id': '5EehXjjMktLuJmbRsM7YfB', 'name': 'Disciples', 'overall_popularity': 60, 'coefficientOfVariation': 0.8385254915624212, 'total_albums': 5}
{'_id': '21WS9wngs9AqFckK7yYJPM', 'name': 'PnB Rock', 'overall_popularity': 66, 'coefficientOfVariation': 0.8349065522782805, 'total_albums': 4}
{'_id': '2p0UyoPfYfI76PCStuXfOP', 'name': 'Franz Schubert', 'overall_popularity': 62, 'coefficientOfVariation': 0.821437463188524, 'total_albums': 273}
{'_id': '4wP1kxjUsc9IR4Iy2smL7o', 'name': 'Trinidad Cardona', 'overall_popularity': 57, 'coefficientOfVariation': 0.8201060195726961, 'total_albums': 6}
{'_id': '5Wg2b4Mp42gicxEeDNawf7', 'name': 'A7S', 'overall_popularity': 66, 'coefficientOfVariation': 0.8185556300278125, 'total_albums': 9}
{'_id': '6KZDXtSj0SzGOV705nNeh3', 'name': 'Kid Ink', 'overall_popularity': 64, 'coefficientOfVariation': 0.8179488251610985, 'total_albums': 5}
{'_id': '7FNnA9vBm6EKceENgCGRMb', 'name': 'Anitta', 'overall_popularity': 76, 'coefficientOfVariation': 0.8163895999588751, 'total_albums': 4}
{'_id': '1QL7yTHrdahRMpvNtn6rI2', 'name': 'George Frideric Handel', 'overall_popularity': 65, 'coefficientOfVariation': 0.8155750598232241, 'total_albums': 513}
{'_id': '66Q9dkZ7EXdwU2h6tEkUdC', 'name': 'Isabèl Usher', 'overall_popularity': 43, 'coefficientOfVariation': 0.8128616486770043, 'total_albums': 10}
{'_id': '2HPaUgqeutzr3jx5a9WyDV', 'name': 'PARTYNEXTDOOR', 'overall_popularity': 77, 'coefficientOfVariation': 0.8124038404635959, 'total_albums': 7}
{'_id': '660YptcR0hNHJ8iEr1qcse', 'name': 'Anthony Ramos', 'overall_popularity': 64, 'coefficientOfVariation': 0.8112383168212872, 'total_albums': 12}
{'_id': '3KlU8ZNbRdGSe2hGY5aUF9', 'name': 'Joyce Cândido', 'overall_popularity': 42, 'coefficientOfVariation': 0.8110828001417311, 'total_albums': 48}
{'_id': '0WFFfBGhY0aC6MQiQ1UQi8', 'name': 'Max Milner', 'overall_popularity': 42, 'coefficientOfVariation': 0.8074908807481045, 'total_albums': 7}
{'_id': '3rJPS8fYBokXpYw1mS9wr0', 'name': 'Raphaella', 'overall_popularity': 51, 'coefficientOfVariation': 0.8074904434892902, 'total_albums': 9}
{'_id': '6PZYFmF3PH6cOREAzfXiAL', 'name': 'The Countdown Kids', 'overall_popularity': 54, 'coefficientOfVariation': 0.8065820771913076, 'total_albums': 105}
{'_id': '14mErTJ0ubFVjx2zBAwjkE', 'name': 'Brandyn Burnette', 'overall_popularity': 40, 'coefficientOfVariation': 0.803208790063097, 'total_albums': 89}
{'_id': '5Nf5yishRW9Ye174sJISkg', 'name': 'London On Da Track', 'overall_popularity': 50, 'coefficientOfVariation': 0.8024904545882247, 'total_albums': 4}
{'_id': '2ZQifdPOptKHxTaYTLh0BC', 'name': 'Kim Larsen', 'overall_popularity': 54, 'coefficientOfVariation': 0.7991866964711807, 'total_albums': 55}
{'_id': '5GXruybcLmXPjR9rKKFyS6', 'name': 'Jimmy Smith', 'overall_popularity': 43, 'coefficientOfVariation': 0.7989373663471653, 'total_albums': 126}
{'_id': '085pc2PYOi8bGKj0PNjekA', 'name': 'will.i.am', 'overall_popularity': 71, 'coefficientOfVariation': 0.798784474403755, 'total_albums': 8}
{'_id': '2r8r62VGJKGi463aH1HJUZ', 'name': 'Kirko Bangz', 'overall_popularity': 47, 'coefficientOfVariation': 0.7937253933193772, 'total_albums': 3}
{'_id': '6k8y1np6q05Us6dNn3vmF8', 'name': 'Arianna', 'overall_popularity': 51, 'coefficientOfVariation': 0.7932995709256178, 'total_albums': 79}
{'_id': '3VzBBEtT4vel6khrAiFlwA', 'name': 'Robin White', 'overall_popularity': 45, 'coefficientOfVariation': 0.790255525890168, 'total_albums': 4}
{'_id': '4qwGe91Bz9K2T8jXTZ815W', 'name': 'Janet Jackson', 'overall_popularity': 62, 'coefficientOfVariation': 0.7889543583705186, 'total_albums': 5}
{'_id': '0st5vgzw9XkH5ALJiUM1lE', 'name': 'Slim Thug', 'overall_popularity': 55, 'coefficientOfVariation': 0.7870264675263836, 'total_albums': 99}
{'_id': '0oBsnAC3fzYkTHF3bkfNx6', 'name': 'Phony Ppl', 'overall_popularity': 45, 'coefficientOfVariation': 0.7839467665315086, 'total_albums': 5}
{'_id': '5ZS223C6JyBfXasXxrRqOk', 'name': 'Jhené Aiko', 'overall_popularity': 76, 'coefficientOfVariation': 0.7815342908349353, 'total_albums': 3}
{'_id': '0SfsnGyD8FpIN4U4WCkBZ5', 'name': 'Armin van Buuren', 'overall_popularity': 72, 'coefficientOfVariation': 0.7809907531541453, 'total_albums': 133}
{'_id': '7qPISKHhhKDLZTmYcX7bWd', 'name': 'Stylo G', 'overall_popularity': 49, 'coefficientOfVariation': 0.7806524723258592, 'total_albums': 6}
{'_id': '3cjxUpsyr916tEX0DBBSGH', 'name': 'Lesfm', 'overall_popularity': 46, 'coefficientOfVariation': 0.7806247497997998, 'total_albums': 3}
{'_id': '39mHYiNmLR7p8PXNG8Wll6', 'name': 'Remy Ma', 'overall_popularity': 56, 'coefficientOfVariation': 0.7788762414183549, 'total_albums': 3}
{'_id': '3XVpDdKav6C6zwlDXPhMEO', 'name': 'Kamaiyah', 'overall_popularity': 50, 'coefficientOfVariation': 0.7766853498088809, 'total_albums': 4}
{'_id': '3v3clM1KQgVfpPjKJFPAmx', 'name': 'SHOP BOYZ', 'overall_popularity': 44, 'coefficientOfVariation': 0.774805096154691, 'total_albums': 8}
{'_id': '0KYOBAf6Zky4CFQne2JPTX', 'name': 'Hiko', 'overall_popularity': 63, 'coefficientOfVariation': 0.7721554247688739, 'total_albums': 4}
{'_id': '3oGVQWQy7lgTMuTnKUZZNZ', 'name': 'Yuki Hayashi', 'overall_popularity': 55, 'coefficientOfVariation': 0.7705517503711221, 'total_albums': 57}
{'_id': '65RQbLHJIWPfWwxYJ5a5BZ', 'name': 'Rosalie.', 'overall_popularity': 43, 'coefficientOfVariation': 0.767095413980334, 'total_albums': 7}
{'_id': '2J3qGaj5UzHvu0fjlLgb8k', 'name': 'Steven Halpern', 'overall_popularity': 53, 'coefficientOfVariation': 0.7628214415734117, 'total_albums': 30}
{'_id': '6i392l38cR3uBPF0DbNs7S', 'name': 'Quality Control', 'overall_popularity': 63, 'coefficientOfVariation': 0.7620394746672081, 'total_albums': 10}
{'_id': '3DELNHPLdJgXkDHOTt3ok8', 'name': 'MOUNT WESTMORE', 'overall_popularity': 45, 'coefficientOfVariation': 0.7597249950293522, 'total_albums': 73}
{'_id': '2QOIawHpSlOwXDvSqQ9YJR', 'name': 'Antonio Vivaldi', 'overall_popularity': 69, 'coefficientOfVariation': 0.7593182391784142, 'total_albums': 189}
{'_id': '5Dl3HXZjG6ZOWT5cV375lk', 'name': 'Yo-Yo Ma', 'overall_popularity': 65, 'coefficientOfVariation': 0.7584717670640533, 'total_albums': 21}
{'_id': '6GMYJwaziB4ekv1Y6wCDWS', 'name': 'Soulja Boy', 'overall_popularity': 66, 'coefficientOfVariation': 0.7551137953212006, 'total_albums': 100}
{'_id': '7CCjtD0hCK005Bvg2WG1a7', 'name': 'Carnage', 'overall_popularity': 50, 'coefficientOfVariation': 0.7542542066730711, 'total_albums': 6}
{'_id': '4qBgvVog0wzW75IQ48mU7v', 'name': 'KYLE', 'overall_popularity': 60, 'coefficientOfVariation': 0.7459091566069128, 'total_albums': 5}
{'_id': '09zTbBto6vE4LzVjQluIz2', 'name': 'Relaxing Piano Crew', 'overall_popularity': 51, 'coefficientOfVariation': 0.7434460458807259, 'total_albums': 102}
{'_id': '3tMLo1k3iUo82coMLWXzxq', 'name': 'Henry Purcell', 'overall_popularity': 54, 'coefficientOfVariation': 0.7423892076049486, 'total_albums': 81}
{'_id': '5B7uXBeLc2TkR5Jk23qKIZ', 'name': 'Gustav Holst', 'overall_popularity': 54, 'coefficientOfVariation': 0.7413570343263033, 'total_albums': 40}
{'_id': '4O49GHbECmNppFvzK0WZXf', 'name': 'maeshima soshi', 'overall_popularity': 47, 'coefficientOfVariation': 0.7409327454859405, 'total_albums': 13}
{'_id': '3p8LxkUdPRd5hPtdTSrCoS', 'name': 'Niken Salindry', 'overall_popularity': 57, 'coefficientOfVariation': 0.7407072582712926, 'total_albums': 3}
{'_id': '49GY4uPAwdlk5lSGtfKWYl', 'name': 'MJ Cole', 'overall_popularity': 45, 'coefficientOfVariation': 0.739887481730935, 'total_albums': 3}
{'_id': '3DiSC0nSNNWpPy5ZK3mcrz', 'name': 'Nef The Pharaoh', 'overall_popularity': 43, 'coefficientOfVariation': 0.7347056468703997, 'total_albums': 3}
{'_id': '3bUwxJgNakzYKkqAVgZLlh', 'name': 'Travis', 'overall_popularity': 56, 'coefficientOfVariation': 0.7333365574568518, 'total_albums': 356}
{'_id': '62DmErcU7dqZbJaDqwsqzR', 'name': 'Popcaan', 'overall_popularity': 63, 'coefficientOfVariation': 0.7326432102545871, 'total_albums': 5}
{'_id': '568ZhdwyaiCyOGJRtNYhWf', 'name': 'Deep Purple', 'overall_popularity': 63, 'coefficientOfVariation': 0.7299421235712272, 'total_albums': 862}
{'_id': '41LqrhKD3Hs6MOOFPhb59G', 'name': 'Anikdote', 'overall_popularity': 44, 'coefficientOfVariation': 0.7284313590846835, 'total_albums': 4}
{'_id': '1JOQXgYdQV2yfrhewqx96o', 'name': 'Giuseppe Verdi', 'overall_popularity': 60, 'coefficientOfVariation': 0.7283624416700294, 'total_albums': 184}
{'_id': '4Mr0dXg1KM8bmg2tBe2xEe', 'name': 'Kim Bo', 'overall_popularity': 47, 'coefficientOfVariation': 0.7273794970884431, 'total_albums': 19}
{'_id': '4hGjngc0tPOBwTgTPci3IK', 'name': 'Santo & Johnny', 'overall_popularity': 50, 'coefficientOfVariation': 0.724566058792278, 'total_albums': 52}
{'_id': '3ScY9CQxNLQei8Umvpx5g6', 'name': 'Fat Joe', 'overall_popularity': 65, 'coefficientOfVariation': 0.7244670247229275, 'total_albums': 16}
{'_id': '5SLOTZhruJRRGgIRtTSPc5', 'name': 'Priscilla Chan', 'overall_popularity': 46, 'coefficientOfVariation': 0.7231766321593037, 'total_albums': 102}
{'_id': '5ZsFI1h6hIdQRw2ti0hz81', 'name': 'ZAYN', 'overall_popularity': 74, 'coefficientOfVariation': 0.7224402252523985, 'total_albums': 4}
{'_id': '7sfl4Xt5KmfyDs2T3SVSMK', 'name': 'Lil Jon', 'overall_popularity': 70, 'coefficientOfVariation': 0.7191546256554823, 'total_albums': 5}
{'_id': '5U7ed4eqjReC376kSJKfs8', 'name': 'India Shan', 'overall_popularity': 44, 'coefficientOfVariation': 0.7166524403234853, 'total_albums': 17}
{'_id': '5dHt1vcEm9qb8fCyLcB3HL', 'name': 'A$AP Ferg', 'overall_popularity': 68, 'coefficientOfVariation': 0.7100256916838257, 'total_albums': 20}
{'_id': '0sOfFTIMocxq5mPB8SFLHn', 'name': 'Piano Project', 'overall_popularity': 42, 'coefficientOfVariation': 0.7087898707807363, 'total_albums': 570}
{'_id': '3MKCzCnpzw3TjUYs2v7vDA', 'name': 'Pyotr Ilyich Tchaikovsky', 'overall_popularity': 70, 'coefficientOfVariation': 0.7067374837752999, 'total_albums': 158}
{'_id': '7HCqGPJcQTyGJ2yqntbuyr', 'name': 'Amit Trivedi', 'overall_popularity': 66, 'coefficientOfVariation': 0.7049785392452693, 'total_albums': 12}
{'_id': '1Yss4ClgrS9sIprNlq5O3l', 'name': 'Yhung T.O.', 'overall_popularity': 40, 'coefficientOfVariation': 0.7042034027664834, 'total_albums': 211}
{'_id': '5pbBGJlVCUzwmdfd1Q1tEX', 'name': 'En?gma', 'overall_popularity': 43, 'coefficientOfVariation': 0.704088321054413, 'total_albums': 57}
{'_id': '3UVRliakQfa1pMWIsNuiZ8', 'name': 'Musiq Soulchild', 'overall_popularity': 56, 'coefficientOfVariation': 0.701812721957872, 'total_albums': 355}
{'_id': '5pU9lKGn9IUnVvOCONrcIS', 'name': 'Rich Gang', 'overall_popularity': 51, 'coefficientOfVariation': 0.7014430244225438, 'total_albums': 10}
{'_id': '1mYsTxnqsietFxj1OgoGbG', 'name': 'A.R. Rahman', 'overall_popularity': 77, 'coefficientOfVariation': 0.7000501067736135, 'total_albums': 29}
{'_id': '7kwCkEJ384PWm0UQW3hxjS', 'name': 'Yella Beezy', 'overall_popularity': 48, 'coefficientOfVariation': 0.6997023176561111, 'total_albums': 4}
{'_id': '0N0d3kjwdY2h7UVuTdJGfp', 'name': 'Cascada', 'overall_popularity': 63, 'coefficientOfVariation': 0.6982367979695578, 'total_albums': 11}
{'_id': '1Uff91EOsvd99rtAupatMP', 'name': 'Claude Debussy', 'overall_popularity': 67, 'coefficientOfVariation': 0.6973154529156471, 'total_albums': 90}
{'_id': '3Qubu5zXcOh0EIb2bDwMdB', 'name': 'WYR GEMI', 'overall_popularity': 43, 'coefficientOfVariation': 0.6967804235050535, 'total_albums': 6}
{'_id': '2rTo8KIkBTFjQS7VvaKYQ4', 'name': 'W&W', 'overall_popularity': 66, 'coefficientOfVariation': 0.6928203230275509, 'total_albums': 4}
{'_id': '20sxb77xiYeusSH8cVdatc', 'name': 'Meek Mill', 'overall_popularity': 73, 'coefficientOfVariation': 0.6920647560994262, 'total_albums': 6}
{'_id': '0NWbwDZY1VkRqFafuQm6wk', 'name': 'Mike WiLL Made-It', 'overall_popularity': 66, 'coefficientOfVariation': 0.6867896330027122, 'total_albums': 6}
{'_id': '3BG0nwVh3Gc7cuT4XdsLtt', 'name': 'Joe Henderson', 'overall_popularity': 40, 'coefficientOfVariation': 0.6860434083140444, 'total_albums': 56}
{'_id': '2g1ULtHvT0HfONgiTgVE6U', 'name': 'One Piano', 'overall_popularity': 40, 'coefficientOfVariation': 0.6858249965431165, 'total_albums': 52}
{'_id': '2HcwFjNelS49kFbfvMxQYw', 'name': 'Robbie Williams', 'overall_popularity': 72, 'coefficientOfVariation': 0.6852201304828507, 'total_albums': 6}
{'_id': '2retT7MFwHDVTeGKDdybEx', 'name': 'K. Michelle', 'overall_popularity': 46, 'coefficientOfVariation': 0.6848725175126659, 'total_albums': 4}
{'_id': '4cSobXLXhJMHYUZvBMuQFG', 'name': '3Breezy', 'overall_popularity': 47, 'coefficientOfVariation': 0.6828224723018068, 'total_albums': 39}
{'_id': '36HuUPCABTgaY4e8rgzSNG', 'name': 'Sukh Lotey', 'overall_popularity': 52, 'coefficientOfVariation': 0.6819943394704735, 'total_albums': 4}
{'_id': '3hv9jJF3adDNsBSIQDqcjp', 'name': 'Mark Ronson', 'overall_popularity': 71, 'coefficientOfVariation': 0.6802424501489441, 'total_albums': 12}
{'_id': '4iEphwHHSj7O2OvhB8RXFY', 'name': 'HYPERAVE', 'overall_popularity': 49, 'coefficientOfVariation': 0.6802120510200563, 'total_albums': 13}
```
#### Interpretation: Maroon 5, Jhonny Cash, Robbie Williams, Will.I.am, jackson 5, royce da 5'9'', Gucci mane
Unconsistent artists in term of popularity