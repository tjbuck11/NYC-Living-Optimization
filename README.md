# Where to Live in NYC
### Optimization Project by Thomas Buck


## Project Description

I accepted an internship based in New York City for this summer, so I must find a place to live for  about 3 months. I will either sublet an apartment or reside in dorm-style living at a nearby university or private company. However, New York is a massive city with many different neighborhoods, and my housing location could either be the biggest convenience or inconvenience. My goal is to find the most convenient location to live that is within my rent budget and also determine my method of travel to each location within my transportation budget. 

Originally, I believed a facility location problem would be the best model choice to address this situation. However, finding all the possible locations I could live in New York would take countless hours to accomplish as there are hundreds if not thousands of different apartment buildings throughout Manhattan. Since this option isn't feasible, I decided to take a hierarchical goal programming approach. Another benefit to using this is I can easily account for the cost of my rent, which I originally believed was too difficult to include. This method would address each situation efficiently in a three stage process.

The first stage determines which neighborhood to live in based on my rent budget and total distance to all the locations I must travel to. Splitting this into the first stage allows me to create a model that determines the optimal neighborhood to live in without too much complexity. Each NYC neighborhood will be represented by it's centroid, and distances will be computed from each neighborhood to each location. This is arguably the most important goal since rent will account for a large majority of my incurred costs while living in New York. The model will be a mixed integer program with a binary variable representing which neighborhood I should live in.

The second stage determines exactly where I should live within the optimal neighborhood. To do so, I will formulate the problem similar to a Sylvester Optimization Model, where my living quarters will minimize my total distance to all locations (and account for the locations priority as well). This will allow me to minimize my distance traveled, while keeping my rent within my budget. This is a quadratically constrained optimization problem due to the distance formulation and will be solved using the qcp solver CPLEX.  

Finally, the third stage will take the optimal location to live and convert distances into costs and times based on my method of transportation. The model will then minimize the total transportation time while keeping my transportation costs within my budget. This will also be a mixed integer program since the formulations for cost and time are linear and I will have a binary variable determining which transportation method I take to each location.

Each stage will be labeled clearly and all data utilized will be displayed. Additionally, each model contains a formulation with the objective function and the constraints the problem is subjected to. 

\**Also, all the maps within this workbook are interactive, so feel free to zoom in to get a closer look at the locations. You can also hover over elements of the map such as points, regions, or lines and text will pop up denoting what this object is referring to (but regions must be clicked on). If you get too off center or the zoom is odd, rerun the cell and it should recenter your location and the zoom level (however, make sure you rerun the cells before it so it displays the proper map from the previous model).*


## General Data

The data for this project was generated through google map searches to find locations I want to be close to in Manhattan. I recorded the geographical coordinates for each location from google maps. A description of each location can be found below:

- **West Monroe NYC Office**: Work location for next summer
- **CitiGroup Headquarters**: My girlfriends work location for next summer
- **NYU Gym**: New York University Athletic Facility
- **Targets**: 3 Target locations for groceries and convenience products
- **Whole Foods**: 3 Whole Foods locations for organic and specialty groceries
- **Starbucks**: 3 Starbucks locations for coffee and convenient breakfast foods
- **Bars**: 2 bars for weekends and socializing
- **Burger Restaurant**: Restaurant specializing in hamburgers (my favorite food)
- **Italian Restaurant**: Upscale restaurant for date nights
- **Pizza**: Restaurant serving cheap pizza for convenience
- **Chinese Restaurant**: For Chinese food cravings
- **Subway Stops**: 5 all-train subway stops

Each location's geographic coordinates are converted to UTM coordinates (which closely model cartesian coordinates) with miles as the units. The Euclidean distance formulation between two UTM coordinates is an accurate approximation for the true distance over small areas (distances less than 100 km) and since the data is clustered around Manhattan, using Euclidean distance to approximate the true distance between two points is supported. 

Locations are represented by a set denoted by $ \ell $ and the coordinates are denoted as a set $x, y$ where $x$ and $y$ are the corresponding Easting and Northing UTM coordinates, respectively. Additionally, let the parameter $data_{\ell xy}$ refer to the coordinates for each location.

Additionally, each location has a relative weight associated with it. This value is assigned entirely by the modeler who must determine which locations are most important to them. I constructed these weights by thinking about my frequency of travel to the specific location in a given week and my own priority of the location (I also partially considered my girlfriends priorities). Not all locations will be traveled to in a given week or at that frequency, so the final distances (in miles) given by the models should not be taken at face value. However, I want to include all of these locations to ensure I am as close as possible to anything I may need while in Manhattan. The weights are denoted as a parameter referred to as $ w_\ell $. Traveling to a location entails traveling back to where I live, so the weights take this into account.

Therefore, all locations were assigned a **weight of 2** except for the ones noted below:

- Work 5 days in person each week : $w_{west\:monroe}$ **= 10** 
- Partial priority to my girlfriends workplace : $w_{citi}$ **= 6**
- Travel to a Target an average of 2 times per week: $w_{targets_{123}}$ = **4**
- Workout 5 days per week : $w_{gym}$ **= 10**
- Priority on restaurant with my favorite food : $w_{burger}$ **= 4**
- Subways for transportation convenience : $w_{subway_{12345}}$ **= 6**

## Stage 1: Determining a Neighborhood

There are a vast array of neighborhoods in New York, all of which have different costs of living. The data for these regions and their respective boundaries were collected from a geojson file found on [nyc.gov](https://www.nyc.gov/site/planning/data-maps/open-data/census-download-metadata.page) about neighborhood tabulation areas used for the 2020 census tracking. I cleaned the geojson file to only include regions in Lower Manhattan since most of the locations determined above are in the Lower Manhattan region. These neighborhoods include: 

- Financial District-Battery Park City
- Tribeca-Civic Center
- The Battery-Governors Island-Ellis Island-Liberty Island
- SoHo-Little Italy-Hudson Square
- Greenwich Village
- West Village
- Chinatown-Two Bridges
- Lower East Side
- East Village
- Chelsea-Hudson Yards
- Hell's Kitchen
- Midtown South-Flatiron-Union Square
- Midtown-Times Square
- Stuyvesant Town-Peter Cooper Village
- Gramercy
- Murray Hill-Kips Bay
- East Midtown-Turtle Bay
- Upper West Side-Lincoln Square
- Upper West Side (Central)
- Upper West Side-Manhattan Valley
- Upper East Side-Lenox Hill-Roosevelt Island
- Upper East Side-Carnegie Hill
- Upper East Side-Yorkville

The set of neighborhoods will be denoted by $n$. A visualization of these neighborhoods and the coordinates of each location in $\ell$ can be found below.

INSERT VISUALIZATION

For each neighborhood, I computed the centroid (center) of the region. This represents the neighborhood while I determine which one is most convenient to live in based on my rent budget. The median per month rent price (in USD) of studio apartments in each neighborhood was obtained from [renthop.com](https://www.renthop.com/average-rent-in/new-york-ny). There were a couple of pecularities with the data which include:

- **SoHo-Little Italy-Hudson Square**: The source had different rent prices for both SoHo and Little Italy, so I took the average between these two pricepoints.
- **Chinatown-Two Bridges**: There wasn't any data on Chinatown, however, Renthop.com had a listing for a studio apartment that was around \$3,000 per month and a simple google search showed this to be a good approximation of the true pricepoint.
- **Murray Hill-Kips Bay**: The source had different rent prices for both Murray Hill and Kips Bay, so I took the average between these two pricepoints.
- **All Upper West side locations**: The source did not break up these regions and instead had a price for the entirety of the Upper West Side region. Therefore, we will assume rent prices are approximately equal across all subdivisions of the Upper West Side region.
- **Upper East Side-Carnegie Hill and Upper East Side-Yorkville**: The source only had data for the Upper East Side region (but it did have data for the Roosevelt Island neighborhood) so we will take the same approach noted above and assume rent prices are approximately equal across these two subdivisions of the Upper East Side.

The rent price of a specific location will be denoted as $p_n$. 

### Formulation

For reference, we have the following:
- $\ell$ : set of 22 locations, for formulation purposes let $\ell = 1,2,...,L$
- $n$ : set of 22 neighborhoods, for for formulation purposes let $n = 1,2,...,N$
- $data_{\ell xy}$ : the coordinates for each location
- $w_\ell$ : the weight of each location (assigned by the modeler)
- $p_n$ : The median studio apartment monthly rent (US Dollars)
- $c_{nxy}$ : The centroid UTM coordinate for each neighborhood

First, we will compute the distance between each neighborhood (represented by the regions centroid) and all locations using the Euclidean Distance formulation. Let the distance between a neighborhood and a location be denoted as $d_{n\ell}$. The formula used to calculate the distance (in miles) for a neighborhood $n$ to all locations $\ell$ using the representations above is:

$$dist_{n\ell} = \sqrt{(c_{n x} - data_{\ell x})^2 + (c_{ny} - data_{\ell y})^2}\:\:\:\:\:for\:\ell=1,\ldots,L$$

We will define the following variables to include in the formulation:
- $live_n$ : a binary variable, determines if I live in the neighborhood
- $lbudget$ : a scalar value, contains my per month rent budget in US dollars
- $dist_{n \ell}$ : a variable to calculate the total distance for a neighborhood to each location (in miles)

The model to determine the optimal neighborhood to live in that minimizes the total distance can be solved by:

\begin{align*}
\min_{w_\ell, d_{n\ell}, live_n } & \sum_{i=1}^{N} \sum_{j=1}^{L} w_j dist_{ij} live_i \\ 
\text{subject to } & \sum_{i=1}^{N} live_i p_i \le lbudget\\
& \sum_{i=1}^{N} live_i = 1\\
\text{and distances from a neighborhood $n$ to each locations (without accounting weights) is:}\\
& distance_{n \ell} = dist_{n\ell} live_n \text{for $n=1,\ldots,N$}
\end{align*}

INSERT DATAFRAME

Here, I set my budget to be my absolute upper limit of **\\$4,000** per month and find the optimal neighborhood to be **Midtown South-Flatiron-Union Square** with a total travel distance of about **100 miles**.

However, it could be interesting to see how my budget impacts my total travel distance. If decreasing my budget only slightly increases my travel distance, I would consider decreasing my budget to save money. To observe this, I ran the model numerous times with different values for my rent budget ($lbudget$) and generated a Pareto Optimal Curve. The values of $lbudget$ range from **\\$2500-\\$5000**, since these are the approximate minimum and maximum median studio rental prices.

INSERT TABLE

INSERT GRAPH

### Results

When fixing my budget to be the absolute maxiumum I am willing to spend on rent in a given month, my total distance traveled was approximately **100 miles**. The graph above shows after about \\$3,800 (the high end of my budget), there is no benefit to increasing my budget, indicating this is the general solution to minimize total distance optimal. However, there is only a slight difference in distance of approximately 25 miles between a budget of \\$3,800 and a budget of about **\\$3,200**.

Given that over the course of 3 months I would be saving about **\\$1,800** on rent, I believe traveling an extra 25 miles (which is an overestimate of distance since I may not travel to all locations in a given week) is worth this trade off. Therefore, I solved the model again with the optimal budget value from the graph above. 


INSERT FINAL MODEL SOLVE

INSERT FINAL DISTANCE TABLE

INSERT MAP

The neighborhood with a median studio rental price within the optimal tradeoff budget that minimizes my total travel distance is **Hell's Kitchen**. Above is a visualization of the neighborhood in relation to the locations. 

However, Hell's Kitchen is a large neighborhood, and living in one location within the region could be drastically different from living in another when calculating total traveling distance and time spent traveling. Therefore, determining the ideal location to live within this neighborhood could prove to be very useful.

## Stage 2: Determining *Where* in the Neighborhood

To determine this, I used a formulation very similar to a sylvester optimization model, where instead of minimizing the radius of the smallest enclosing sphere I minimized the sum of the product of the distance to each location and its relative weight ($w_{\ell}$). The objective of this model is to determine an optimal location within Hell's Kitchen that minimizes the total distance needed to travel. Inherently, this will also minimize my total time and cost of traveling since both time and cost are strictly increasing functions of distance. 

There will be an additional dataset utilized that contains the $x$ and $y$ ranges for each neighborhood. I denote a set $coordbound$ which contains $xmin, xmax, ymin, ymax$. The parameter $bound_{n,coordbound}$ will hold this data. It must be noted that many regions are oddly shaped and for modeling purposes I took the minimum and maximum $x$ and $y$ coordinates to be the neighborhoods respective bounds. These coordinates are used to bound my optimal location to the neighborhood determined in Stage 1 to ensure the rent price is within my budget.

\**Note: Estimating the regions as a rectangle could cause the final location to be slightly outside of the Hell's Kitchen, but close enough that we will assume pricing for apartments still holds.*

### Formulation

The model takes in a variable which represents the optimal location to live within Hell's Kitchen. It contains a distance calculation to each location in $\ell$. The model then determines the optimal location by minimizing the total distance traveled. A detailed formulation is below.

For reference, the formulation involves these aspects of the formulation for the model in Stage 1:

- $\ell$ : set of 22 locations, for formulation purposes let $\ell = 1,2,...,L$
- $n$ : set of 22 neighborhoods, for for formulation purposes let $n = 1,2,...,N$
- $data_{\ell xy}$ : the coordinates for each location
- $w_\ell$ : the weight of each location (assigned by the modeler)

I also define the following variable:

- $home_{xy}$ : Optimal coordinates for my place of residence

The sylvester like model to find an optimal place to live that minimizes total distance traveled can the be solved by:
\begin{align*}
\min_{w_{\ell}, d_\ell} & \left( \sum_{i=1}^{L} w_i d_i\right) \\ 
\text{where  } & d_\ell = \sqrt{(home_{x} - data_{\ell x})^2 + (home_{y} - data_{\ell y})^2}\\
\text{subject to}\\
& bound_{n, xmin} \le home_x \le bound_{n,xmax}\\
& bound_{n, ymin} \le home_y \le bound_{n,ymax}\\ 
\text{ where $n$ = Hell's Kitchen}
\end{align*}

However, this must be reformulated due to the square root within the distance calculation, which will cause GAMS to throw an error. Therefore, the problem can be reformulated as

\begin{align*}
\min_{w_{\ell},d_\ell} & \left( \sum_{i=1}^{L} w_i d_i\right) \\ 
\text{where  } & d_\ell^2 = (home_{x} - data_{\ell x})^2 + (home_{y} - data_{\ell y})^2\\
\text{subject to}\\
& bound_{n, xmin} \le home_x \le bound_{n,xmax}\\
& bound_{n, ymin} \le home_y \le bound_{n,ymax}\\ 
\text{ where $n$ = Hell's Kitchen}
\end{align*}

INSERT SOLVE LINE
INSERT FINAL LOCATION
INSERT MAP

### Results

The model found an optimal solution with geographical coordinates of approximately **40.7546&deg;N, -73.9897&deg;E** (denoted by the red icon above) and my total distance traveled would be approximately **96.35 miles**. The location does appear to be slightly outside of the NYC's government definition of Hell's Kitchen, but this is most likely to due to a combination of estimating the region to be a rectangle as well as some error from converting UTM coordinates to geographical coordinates. Even so, the location is still only half a block outside of the Hell's Kitchen neighborhood. One could simply alter the bounds manually until the point gets is barely within the region, but this is not best practice in general.

It would also be interesting to see how the distance to each location has improved from the model in Stage 1 (using the centroid) to the model in Stage 2 (using the optimal location). 

INSERT GRAPH

The optimal location decreases the total distance by about **28 miles as well as decreasing the distance to almost every location** (the exceptions will always be present if we deviate any marginal amount from the centroid location). Therefore, solving the 'sylvester-like' model was incredibly useful and will also result in minimizing time and help me get more for my money (since both are strictly increasing functions of distance). 

Now for the fun part. Let's see what is actually at this location! Google shows this to be a commercial building with various stores and some apartments are nearby!

[Google Maps Link](https://www.google.com/maps/search/apartments/@40.7545696,-73.990761,18z/data=!4m7!2m6!3m5!1sapartments!2s40.7545925,-73.9896566!4m2!1d-73.9896566!2d40.7545925)

| ![image-2.png](attachment:image-2.png)| ![image-5.png](attachment:image-5.png)  |
|-|-|
| ![image-3.png](attachment:image-3.png) |


## Stage 3: Determining *how* to get to each Location

As noted in the problem description, I want to get a good idea for how I should travel to each location. An optimization model proves to be useful since there are a **many** different combinations of methods to choose from and doing so by hand is nearly impossible. The model takes in a distance and computes a time and cost to each $\ell$ for every transportation method. It chooses the optimal methods, which is done by implementing a binary variable. The objective of the model is to minimize my total travel time while keeping the cost within my transportation budget. The formulations for cost and time from a given distance in miles are described below.

### Data

I considered 3 methods of transportation that I am able to take at any given time in NYC. Those 3 methods are:
- **Walking**
- **The Subway**
- **Ubering**

As noted above, the only information I can calculate given the UTM coordinates is distance. Therefore, I must have a method for determining a time and cost given a distance. The time and cost to for each method are referred to as $time_{method}$ and $cost_{method}$. The formulas and justification for these formulas for each transportation method are denoted below:

#### <u>Walking</u>

**Cost**: There is no need to determine a cost for walking since it is always free (often just inconvenient or tiresome). Therefore, the cost for walking is **0**. $$cost_{walk} = 0$$\
**Time**: At the gym, I frequently walk at a pace of 3.5 miles per hour on a treadmill, but this speed feels slightly faster than my casual walking speed. Based on this, I assume my walking pace to be 3 miles per hour. Denoting my walking speed as $s_{walk}$ and distance to walk as $d_{walk}$, calculating time in minutes can be determined by solving $$time_{walk} = d_{walk}*(60 / s_{walk} )$$

#### <u>The Subway</u>
**Cost**: Based on the information from the Metropolitan Transportation Authority found at [mta.info](https://new.mta.info/fares), a subway ticket for a one way ride is **\$2.75**. The cost to ride the subway does not depend on distance and can be denoted as: $$cost_{subway} = 2.75$$

**Time**: According to [NYTimes](https://www.nytimes.com/2018/12/10/nyregion/new-york-subway-delay.html), a transportation report conducted in 2010 found the average NYC subway travels at a speed of **17 miles per hour**. Additionally, I assume walking to and waiting for the subway takes approximately 5 minutes (since there are smaller subway stations scattered all throughout New York). Denoting subway waiting time as $wait_{subway}$, subway speed as $s_{subway}$, and distance to ride the subway as $d_{subway}$, calculating time in minutes can be determined by solving $$time_{subway} = wait_{subway} + d_{subway}*(60 / s_{subway} )$$

#### <u>Ubering</u>

**Cost**: According to [Investopedia](https://www.investopedia.com/articles/personal-finance/021015/uber-versus-yellow-cabs-new-york-city.asp), Uber's pricing system for NYC consists of a **\$2.55 base fare** and **\$0.35 per minute** and **\$1.75 per mile** fees. It should be noted that time is one of the aspects I am estimating and even though it utilizes adequate sources and logic, it may deviate from the true value. Therefore, I formulated my Uber cost as only a function of distance, since this is a known value from Stage 2. The \\$0.35 per minute fee will be converted to an additional **\\$1.40 per mile** (since speed is about 16 mph) to give a true per mile cost of **\\$3.15**. I denoted these values as $base$ and $pmile$ respectively. Letting $d_{uber}$ represent the distance traveled in the Uber, the cost for an Uber can be determined by $$cost_{uber} = base + pmile*d_{uber}$$

**Time**: From a dataset about Real-Time Traffic Speed Data found on [data.cityofnewyork.us](https://data.cityofnewyork.us/Transportation/DOT-Traffic-Speeds-NBE/i4gi-tjb9), the median traffic speed for Manhattan is approximately **16.15 miles per hour**. Additionally, due to past experience and New York's reputation, I assume the time between ordering the Uber and its arrival to be about **2 minutes**. Denoting uber speed as $s_{uber}$ and uber wait time as $wait_{uber}$, calculating time in minutes can be determined by solving $$time_{uber} = d_{uber}*(60 /s_{uber} ) + wait_{uber}$$


To set this up for the formulation, I denote a set $tran$ that contains all three transportation methods in the order of $walk$, $subway$, $uber$. I represent the parameter used in the formulation below as the initial line and it's corresponding values as the bullets below:

- $s_{tran}$ : speed
    * $s_{walk} = 3$
    * $s_{subway} = 17$
    * $s_{uber} = 16.15$
- $wait_{tran}$ : waiting time
    * $wait_{walk} = 0$
    * $wait_{subway} = 5$
    * $wait_{uber} = 2$
- $base_{tran}$ : base fare
    * $base_{walk} = 0$
    * $base_{subway} = 2.75$
    * $base_{uber} = 2.55$
- $dfare_{tran}$ : fare for each mile
    * $dfare_{walk} = 0$
    * $dfare_{subway} = 0$
    * $dfare_{uber} = 3.15$


### Formulation

The model takes in the predefined distance to each location ($d_\ell$) from Stage 2. I utilize the following aspects of previous models:

- $\ell$ : set of 22 locations, for formulation purposes let $\ell = 1,2,...,L$
- $w_\ell$ : the weight of each location (assigned by the modeler)


I also define the new set $tran$:

- $tran$ : Set of 3 transportation methods, for formulation purposes let $trans = 1,2,...,T$

The values for the cost and time formulation for each transportation method are denoted using the representations defined above:

- $s_{tran}$ : speed
- $wait_{tran}$ : waiting time
- $base_{tran}$ : base fare
- $dfare_{tran}$ : fare for each mile

Addtionally, there are the following scalar values:

- $tbudget = 250$ : my original budget for transportation costs (weekly basis)

I also define the following variables:

- $t_\ell$ : time to travel to location $\ell$ (only including one trip, not multiplied by weights)
- $c_\ell$ : cost to travel to location $\ell$ (total cost to location, weights included)
- $method_{\ell, tran}$ : a binary variable to determine which transportation method I take to location $\ell$

The model to determine the optimal location that minimizes my total transportation costs and time can be solved by:

\begin{align*}
\min_{t_\ell} & \left( \sum_{i=1}^{L} w_i t_i\right) \\ 
& \\
\text{subject to}\\
& \sum_{j=1}^{T} method_{\ell j} = 1  \text{  for $\ell=1,\ldots,L$}\\
& \sum_{i=1}^{L} c_i \le tbudget\\
\text{where $d_{\ell}$ are the distance values determined in Stage 2 and }\\
& c_\ell = w_{\ell} \sum_{j=1}^{T} method_{\ell j}*(base_j + d_\ell*dfare_j) \text{  for $\ell=1,\ldots,L$}\\
& t_\ell = \sum_{j=1}^{T} method_{\ell j} * (wait_j + d_{\ell}*(60/s_j)) \text{  for $\ell=1,\ldots,L$}\\
\end{align*}

Additionally, I fix some values to say I must take a certain transportation method due to feasibility. I indicate to the model that I must either walk or subway to both the my work and the gym, since I would never take an Uber to these locations every single day.

\begin{align*}
method_{WestMonroe, uber} = 0 \\
method_{Gym, uber} = 0
\end{align*}

INSERT SOLVER LINE
INSERT MAP

It would also be interesting to do a similar analysis to that done in Stage 1 to see how altering my transportation budget impacts my total time traveled. To do so, I solve the model above numerous times with transportation budgets ranging from \$50-$450 (I would never spend \$450 on transportation but let's see how small my travel time can get).

INSERT MODEL OUTPUT

\**Originally, some of the model statuses were 8 meaning an integer solution was found. To get a global optimal solution, I set the model optcr to be $10^{-8}$ and this tolerance allowed an optimal global solution to be found for all budgets.*

INSERT PLOT

The graph above shows that varying budget does show some diminishing returns as the transportation budget gets larger and larger. It isn't your typical Pareto Optimal curve, but it will still help me decide on an optimal budget by finding the budget value where the marginal returns to time begin to become insignificant. 

\**Marginal returns in this case refers to how much time is saved by increasing my budget. Diminishing returns refers to how after a specific threshold (or budget value) my returns become significantly smaller and smaller.*

INSERT PLOT

The graph is noisy which I assume is due to the model changing methods (which changes total time) each time the budget increases. However, the marginal returns to time start to become significantly smaller at a budget of around **\$160**. One could also argue that \$200 or \$240 could be an optimal budget value due to larger marginal returns relative to local values, but I want to save money so a budget value of **\$160** is used.

INSERT MODEL OUTPUT

INSERT MAP

The plot above shows my transportation route to each location. 

\**Walk = **Red**, Subway = **Purple**, Uber = **Blue**. You can also hover over the line and it will display the method.* 

### Results

With a transportation budget of **\\$160**, my total traveling time would be approximately **846 minutes**. The model decided my primary mode of transportation to most locations would be the subway, which is exactly how I expected to get around Manhattan in the majority of cases. The maps generally show walking is optimal for short distances, Ubering is optimal for medium distances, and the subway is optimal for long distances. This is rather surprising to me as I typically trend towards Ubering long distances due to convenience, so this finding will hopefully save me a lot of time (and money) while in Manhattan.

## Conclusion

To decide the optimal location to live and transportation methods within Manhattan, I approached the problem with a hierachical goal programming approach.

The first stage determined the optimal neighborhood for me to live in, where the model originally chose Midtown South-Flatiron-Union Square with a total distance of about 100 miles. However, conducting sensitivity analysis on the total distance by changing my budget offered a different solution of **Hell's Kitchen** as the optimal neighborhood (in my situation). In comparison to Midtown South, Hell's Kitchen only increases total distance by about 25 miles but saves me about **\$2,000** over the course of the summer. 

The second stage determined my the optimal location to live within the Hell's Kitchen neighborhood. I formulated a quadratically constrained optimization problem similar to a sylvester model, where the objective was to minimize the sum of the product of the weights and distance to each location ($\sum_{i=1}^{L} w_i d_i$). The model determined the optimal location to be **40.7546&deg;N, -73.9897&deg;E** with a total distance of about **96.35 miles**. This turned out to be a corporate building, but there were numerous apartment buildings within half a block of these coordinates. I then examined the difference between this distance value and the objective value and distances from Stage 1 and found the optimal location decreased total travel distance by about 28 miles and decreased distance to the vast majority of locations (including my girlfriend's work which was a bonus).

The final stage determined the optimal way to get to each location while minimizing time and staying within my budget. With the original budget of about \\$250, the model determined an total travel time of about 717 minutes, or 11 hours and 57 minutes. Doing a similar sensitivity analysis to Stage 1, I determined an budget of \\$160 would be best since I want to minimize my cost and the marginal returns to time became increasingly stagnant after this value. Due to feasibility regions, I used introduced my tolerance level to find an optimal global solution of about **846 minutes**, or **14 hours and 6 minutes** (this is on the extreme high end because as referenced earlier, I won't travel to all locations in a given week at the corresponding weight). By conducting this analysis, I am able to save approximately **\\$900** over the summer while only traveling (at most) an additional 2 hours per week. 

Overall, using optimization models solve this problem helped me determine a good monthly budget estimate for rent (\\$3,160 per month for Hell's Kitchen) and transportation costs (\\$160 x 4 weeks) of **\\$3,800** while saving countless hours of calculations. This is a great start to planning my budget and gave insights into how I should start saving. Solving the problem in a three stage setup ensured all of my own constraints and objectives were met while still finding useful optimal solutions at each stage. By collecting the right data, this process could be done for a variety of different locations but Stage 3 worked particularly well for Manhattan due to the massive subway system and my own choice to not bring a car.