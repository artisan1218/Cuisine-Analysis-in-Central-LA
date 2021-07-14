# Cuisine Analysis in Central L.A.

## Introduction 
Los Angeles, officially the City of Los Angeles and often abbreviated as L.A., is the largest city in California. It has an estimated population of nearly 4 million and is the second-largest city in the United States, after New York City, and the third-largest city in North America, after Mexico City and New York City. Los Angeles is known for its Mediterranean climate, ethnic and cultural diversity, Hollywood entertainment industry, and its sprawling metropolitan area. According to the 2010 Census, the racial makeup of Los Angeles included: 49.8% Whites, 9.6% African Americans, 0.7% Native Americans, 11.3% Asians, 0.1% Pacific Islanders, 23.8% from other races, and 4.6% from two or more races. Los Angeles is home to people from more than 140 countries speaking 224 different identified languages. Ethnic enclaves like Chinatown, Historic Filipinotown, Koreatown, Little Armenia, Little Ethiopia, Tehrangeles, Little Tokyo, Little Bangladesh, and Thai Town provide examples of the polyglot character of Los Angeles.

This project explores the most popular restaurants and its type of cuisine in different neighborhoods in central Los Angeles. Central L.A. has 26 neighborhoods and diverse racial makeup: 46.1% Latino, 26.4% White, 16.2% Asian, 8.2% Black and 3.1% Other. This results in a diverse makeup of cuisines in central L.A. and the analysis data can be used to decide what type of restaurant should you open if you plan to open a restaurant in this area. 

This project answers the questions:
1. "Which type of cuisine is the most popular one in each neighborhood in central L.A.?"
2. "What are the most similar neighborhoods in terms of types of cuisine?"

<br />

## Data
To answer the above questions:
1. First use data from Los Angeles Times to identify different neighborhoods in central LA. LA Times also provides neighborhoods information in other areas in LA.
2. `geopy` will be used to convert an address into latitude and longitude values so that we can further use `FourSquare API`.
3. All data related to locations and quality of different restaurants will be obtained via the `FourSquare API` utilized via the `request` library in Python.
4. Then perform a `K-means clustering` on these neighborhoods and pick the cluster the most similar neighborhoods together. 

<br />

## Methodology
I first get the neighborhoods data from LA Times website. They’ve embedded the neighborhoods name in an interactive map, and I formatted these data in `.json` file and use json package in python to read them. Then I utilized the `geopy` API to obtain the geographical coordinates of the corresponding neighborhoods.

```
neighborhoods_file = open("CentralLA_Neighborhoods.json")
neighborhoods = json.load(neighborhoods_file)

content = list()

for neighborhood in neighborhoods['features']:
    
    neighborhood_info = {}
    
    # get the neighborhood name
    name = neighborhood['properties']['name']
    
    # get the coordinates of this neighborhood
    address = '{}, Los Angeles, CA'.format(name)
    geolocator = Nominatim(user_agent="la_explorer")
    location = geolocator.geocode(address)
    latitude = location.latitude
    longitude = location.longitude
    print('The geograpical coordinate of {} are {}, {}.'.format(name, latitude, longitude))
    
    neighborhood_info['Name'] = name
    neighborhood_info['Latitude'] = latitude
    neighborhood_info['Longitude'] = longitude
    
    content.append(neighborhood_info)

# make sure there are 26 neighborhoods
print('There are {} neighborhoods in the list'.format(len(content)))
```

Next step is to integrate venues data for each neighborhood by using `Foursquare API`. I choose to obtain at most `100 restaurants` in each neighborhood. 
```
url = 'https://api.foursquare.com/v2/venues/explore?categoryId=4d4b7105d754a06374d81259&client_id={}&client_secret={}&v={}&ll={},{}&limit={}'.format(
        CLIENT_ID, 
        CLIENT_SECRET, 
        VERSION, 
        lat, 
        lng, 
        LIMIT)
            
# make the GET request
results = requests.get(url).json()["response"]['groups'][0]['items']
```

Next step is to do the `one-hot encoding` on the categorical cuisine type in order to cluster these neighborhoods based on popular cuisine types. 
```
# one hot encoding
la_onehot = pd.get_dummies(la_venues[['Venue Category']], prefix="", prefix_sep="")

# add neighborhood column back to dataframe
la_onehot.insert(0, 'Neighborhood Name', la_venues['Neighborhood'])

la_onehot.head()
```

Since we don’t know the best `k` yet, I’ve plotted the elbow plot to do the visual check for picking best k. 
![image](https://user-images.githubusercontent.com/25105806/125700267-8a2b31be-d186-4d07-82c6-e50e7325c54f.png)

Then group each neighborhood and calculate the mean one-hot encoding value to the clustering using K-means. I’ve also tried using the sum value instead of mean value when quantifying each neighborhood, but the result is pretty similar to mean value, so I decided to stick with mean value. 
```
la_grouped = la_onehot.groupby('Neighborhood Name').mean().reset_index()
clustering_df = la_grouped.drop('Neighborhood Name', axis=1)

# set number of clusters
kclusters = 5

# run k-means clustering
clutering_result = KMeans(n_clusters=kclusters, random_state=0).fit(clustering_df)
```

<br />

## Results
Using the data I have after processing, I can finally answer the questions at the beginning. 
1. "Which type of cuisine is the most popular one in each neighborhood in central L.A.?"
    According to the dataframe variable “la_restaurants_sorted”, each neighborhood has 5 most popular types of cuisine. 
    The dataframe looks like this:
    ![image](https://user-images.githubusercontent.com/25105806/125700433-35001544-e0aa-4de5-bcdb-657955d2386b.png)

    <br />

    According to the bar plot shown below, Mexican restaurant is the most popular cuisine in central LA area. So if someone decides to open a restaurant in central LA, my recommendation will be a Mexican restaurant unless he/she decides to open in specific neighborhood where certain type of cuisine is much more popular than Mexican food such as Chinatown.
    <img src="https://user-images.githubusercontent.com/25105806/125700496-0414c547-a313-4f13-b2d3-c7fd9e8f522e.png" height="80%" width="80%">


2. "What are the most similar neighborhoods in terms of types of cuisine?"
    Most neighborhoods with Mexican foods as their most popular cuisine are clustered together\
    ![image](https://user-images.githubusercontent.com/25105806/125700694-ab9bbc98-9bfe-484e-9338-917464035919.png)
    
    Some other neighborhoods such as Koreantown and Harved Heights are clustered together because they share Korean restaurants as the popular cuisine.\
    ![image](https://user-images.githubusercontent.com/25105806/125700738-1baf315d-c20b-41b5-a66c-cbf960457bdb.png)
    
<br />

## Discussion
In some neighborhood, a certain type of cuisine is the dominating cuisine and surpass other types of cuisine by a lot. For example, neighborhood Chinatown has Chinese restaurants as the dominating cuisine and Koreantown has the Korean restaurants as the dominating cuisine. The observation makes sense because people lived in Chinatown or Koreantown are most likely to be Chinese or Korean who, reasonably, likes Chinese cuisine or Korean cuisine more. While some other neighborhoods such as Downtown, although likes Mexican foods more, does not have a dominating cuisine that surpass other types of cuisine by a lot. I think this is because the demographical makeup in downtown is more diverse than in Chinatown or Koreantown, so the cuisine types are more mixed together. 

Ethnicity makeup for Chinatown:\
![image](https://user-images.githubusercontent.com/25105806/125700897-afbedd08-9086-4f4c-a182-94328d640ac4.png)


<br />

## Conclusion
This project explores the types of cuisine in central LA area and analyze the popularity of cuisine in each neighborhood. The food preference in each neighborhood can somehow reflect the demographical makeup of races of people in each neighborhood.

