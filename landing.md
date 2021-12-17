---
title: Code Samples
layout: landing
description: 'Explore some sample data analytics work'
image: assets/images/pic07.jpg
nav-menu: true
---

<!-- Main -->
<div id="main">

<!-- One -->
<section id="one">
	<div class="inner">
		<header class="major">
			<h2>Looking at Vacant Lots in Chicago</h2>
		</header>
		<p>Along with my classmates, Kashif Ahmed and Jason Winick, we conducted an analysis of vacant lots in Chicago to identify candidates for green space conversion. Vacant lots are a persistent problem in the city of Chicago. The City holds thousands of pieces of property throughout its jurisdiction, many the result of foreclosures and systemic disinvestment. This represents a huge burden to the city in maintaining these lots and attempting to transfer them to new owners. However, current management and disposition of vacant lots is hindered by information problems, including:
<ul>
	<li>Lack of transparency for residents regarding decision-making process</li>
   	<li>Relevant information about the lots is spread through multiple data sources that do not cross-populate</li>
	<li>Lack of clear standards for lots to meet before transfer to private ownership</li>
		</ul>
Our project attempts to address some of the information concerns plaguing vacant lot disposition in Chicago, particularly the lack of clear standards for transfer eligibility. Using the existing City-Owned Land Inventory, we created an index by which lots are "scored" on their candidacy for disposition and transformation into green space.</p>
	<ul class="actions">
					<li><a href="https://github.com/iph-97/vacant_lots" class="button">View the Project</a></li>
				</ul>
<details><summary markdown="span">View a Code Sample</summary>
<pre class="line-numbers">
   <code class="language-python">
import pandas as pd
from shapely.geometry import Point, MultiPolygon
from shapely.ops import transform
import geopandas as gpd
import matplotlib.pyplot as plt
from pygeoif import geometry
import os
from shapely.geometry.multipoint import MultiPointAdapter
from shapely.geometry.multipolygon import MultiPolygon
from sodapy import Socrata
from shapely.geometry import shape
import pyproj

PATH =  os.path.abspath(os.getcwd())

def api_get(fname):
    client = Socrata("data.cityofchicago.org", None)
    results = client.get(fname, limit = 20005)
    df = pd.DataFrame.from_records(results)
    return df

def read_lots(api_key="aksk-kvfp"):
    df = api_get(api_key)
    gdf = gpd.GeoDataFrame(df, geometry=gpd.points_from_xy(df["x_coordinate"], df["y_coordinate"]))
    gdf = gdf.dropna(subset=["address", "location", ":@computed_region_rpca_8um6", "latitude", "longitude"])
    gdf = gdf.set_crs("ESRI: 102671")
    return gdf

def fix_geom(df, geom='the_geom'):
    df['geometry'] = df[str(geom)].apply(lambda row:geometry.as_shape(row))
    return df

def re_proj(row):
    proj = pyproj.Proj('+proj=tmerc +lat_0=36.66666666666666 +lon_0=-88.33333333333333 +k=0.9999749999999999 +x_0=300000 +y_0=0 +ellps=GRS80 +datum=NAD83 +to_meter=0.3048006096012192 +no_defs')
    x, y = proj(row["longitude"], row["latitude"])
    return x, y

def gdf_proj(df):
    df["projected_coord"] = df.apply(re_proj, axis=1)
    x = []
    y = []
    for tuple in df["projected_coord"]:
        a, b = tuple
        x.append(a)
        y.append(b)
    df["x"] = x
    df["y"] = y
    gdf = gpd.GeoDataFrame(df, geometry=gpd.points_from_xy(df["x"], df["y"]))
    return gdf

def csv_to_gdf(csv_name):
    df = pd.read_csv(os.path.join(PATH,csv_name))
    df['Coordinates'] = df['Map Site CSV'].str.split('GEOSEARCH:').str[1]
    df[['latitude', 'longitude']] = df.pop('Coordinates').str.split(' ', 1, expand=True)
    df = df[df.longitude != ""]
    df = df[df.latitude != ""]
    gdf = gdf_proj(df)
    return gdf

def read_bf(csv=r"CIMC Basic Search Result.csv"):
    gdf = csv_to_gdf(csv)
    gdf = gdf.set_crs("ESRI: 102671")
    return gdf

def check_tuple(gdf):
    #col_list = []
    for col in gdf.columns:
        print(col, 'is a tuple : ', all(isinstance(x,tuple) for x in gdf[col]))

def find_centroid(df):
    centroids = []
    for multipolygon in df["geometry"]:
        center = multipolygon.centroid
        centroids.append(center)
    df["centroid"] = centroids
    return df

def proj_transform(df, to_wgs=True, geom_col="geometry", epsg="EPSG: 4326"):
    reprojections = []
    if to_wgs == True:
        for point in df[geom_col]:
            inproj = pyproj.CRS('+proj=tmerc +lat_0=36.66666666666666 +lon_0=-88.33333333333333 +k=0.9999749999999999 +x_0=300000 +y_0=0 +ellps=GRS80 +datum=NAD83 +to_meter=0.3048006096012192 +no_defs')
            project = pyproj.Transformer.from_crs(inproj, epsg, always_xy=True).transform
            new_point = transform(project, point)
            reprojections.append(new_point)
    else:
        for point in df[geom_col]:
            outproj = pyproj.CRS('+proj=tmerc +lat_0=36.66666666666666 +lon_0=-88.33333333333333 +k=0.9999749999999999 +x_0=300000 +y_0=0 +ellps=GRS80 +datum=NAD83 +to_meter=0.3048006096012192 +no_defs')
            project = pyproj.Transformer.from_crs(epsg, outproj, always_xy=True).transform
            new_point = transform(project, point)
            reprojections.append(new_point)
    df["geometry_reproj"] = reprojections
    return df

def read_park_nbh(api_key, park=False):
    df = api_get(api_key)
    df = fix_geom(df)
    gdf = gpd.GeoDataFrame(df, geometry=df["geometry"])
    gdf = gdf.set_crs("EPSG:4326")
    if park == True:
        gdf = find_centroid(gdf)
        gdf = proj_transform(gdf, to_wgs=False, geom_col="centroid")
    return gdf

def read_bus(api_key="qs84-j7wh"):
    df = api_get(api_key)
    df = df.rename(columns={"point_x":"longitude", "point_y":"latitude"})
    gdf = gdf_proj(df)
    streets = "55th|Garfield|63rd|79th|Ashland|Chicago|Lake Shore|Western"
    gdf = gdf[gdf["street"].str.contains(streets,case=False)]
    return gdf

def read_el(api_key="8pix-ypme"):
    df = api_get(api_key)
    for idx, row in df.iterrows():
        df.loc[idx, 'latitude'] = df['location'][idx]['latitude']
        df.loc[idx, 'longitude'] = df['location'][idx]['longitude']
    gdf = gdf_proj(df)
    return gdf

def is_near(coordinate, df, distance, geom_column="geometry"):
    counter = 0
    circle_buffer = coordinate.buffer(distance)
    for station in df[str(geom_column)]:
        if station.within(circle_buffer):
            counter += 1
    return counter

def near_counter(gdf, comp_gdf, near_col, distance=2500, geom_column="geometry"):
    near = []
    for point in gdf["geometry"]:
        counter = is_near(point, comp_gdf, distance, geom_column)
        near.append(counter)
    gdf[near_col] = near
    return gdf

def find_eligibility(df, elig_col, elig_list, new_col_name):
    eligible = []
    for lot in df[elig_col]:
        if lot in elig_list:
            eligible.append(1)
        else:
            eligible.append(0)
    df[new_col_name] = eligible
    return df
    
def find_candidates(row):
    if row["ANLAP Eligible"] + row["Large Lots Eligible"] >= 1:
        score = row["Near El"] + (row["Near Bus"]/10) + row["Invest SW Eligible"] - row['Near Park'] - row['Near Brownfield']
    else:
        score = 0
    return score
    
gdf = read_lots()
gdf_park = read_park_nbh("ejsh-fztr", park=True)
gdf_nbh = read_park_nbh("y6yq-dbs2")
gdf_bus = read_bus()
gdf_el = read_el()
gdf_bf = read_bf()

gdf = near_counter(gdf, gdf_park, "Near Park", geom_column="geometry_reproj")
gdf = near_counter(gdf, gdf_bus, "Near Bus")
gdf = near_counter(gdf, gdf_el, "Near El")
gdf = near_counter(gdf, gdf_bf, "Near Brownfield")

invest_sw = ["AUBURN GRESHAM", "AUSTIN", "BRONZEVILLE", "ENGLEWOOD", "NEW CITY", "NORTH LAWNDALE", "GREATER ROSELAND", "SOUTH CHICAGO"]
anlap = ["RM-5", "RT-4", "RS-1", "RS-2", "RS-3"]
large_lots = ['RS-2', 'RS-3', 'RT-4', 'RM-5', 'RT-4A', 'RM-4.5', 'RM-6', 'RS-1','RM-5.5', 'RT-3.5', 'RM-6.5']

gdf = find_eligibility(gdf, "community_area_name", invest_sw, "Invest SW Eligible")
gdf = find_eligibility(gdf, "zoning_classification", anlap, "ANLAP Eligible")
gdf = find_eligibility(gdf, "zoning_classification", large_lots, "Large Lots Eligible")

gdf["score"] = gdf.apply(find_candidates, axis=1)
</code>
</pre>
</details>

	</div>	
</section>

<!-- Two -->
<section id="two" class="spotlights">
	<section>
		<a href="generic.html" class="image">
			<img src="{% link assets/images/pic08.jpg %}" alt="" data-position="center center" />
		</a>
		<div class="content">
			<div class="inner">
				<header class="major">
					<h3>Visualizing Local Violence Prevention and Response Budgets</h3>
				</header>
				<p>I wrote a script in R to create a series of graphs for city government budgets to compare allocations for violence prevention and response. Theses graphs were used in presentations internally to help staff prepare budget proposals. </p>
				<ul class="actions">
					<li><a href="https://github.com/iph-97/vpr_budgets" class="button">View the Project</a></li>
				</ul>
			</div>
	<details><summary markdown="span">View a Code Sample</summary>
		<pre class="line-numbers">
   		<code class="language-r">
		ibrary(tidyverse)
library(treemap)
setwd("~/Documents")

vpr <- read_csv("VPR_Budgets.csv")

vpr_grouped <- vpr %>%
  select(-Dept, -Delegates, - `Community Area`, -`Subdelegate Amount`, - Subdelegates) %>%
  group_by(Unit) %>%
  filter(Unit !="LA", 
         Unit != "Federal",
         Unit != "Illinois", 
         Unit != "Maryland",
         Unit != "State") %>%
  mutate(Program = `Funding Categories†`) %>%
  select(-`Funding Categories†`)

vpr_unit_total_spend <- vpr %>% group_by(Unit) %>%
  filter(Unit !="LA", 
         Unit != "Federal",
         Unit != "Illinois", 
         Unit != "Maryland",
         Unit != "State") %>%
  summarise(total_spend = sum(`Total Amount`, na.rm = T))

vpr_unit_total_spend %>% 
  ggplot(aes(x = Unit, y = total_spend)) +
  geom_bar(stat = "identity") +
  theme(axis.text.x = element_text(angle=65, vjust=0.6)) +
  labs(title = "Total Spending on Violence Reduction and Prevention (self-defined)")+
  theme_minimal()

chart <- function(category) {
  png(str_c("chart", category, ".png", sep = ""))
  
  chart_data <- vpr_grouped %>%
    mutate(category_spend = ifelse(Category == category, `Total Amount`, 0)) %>%
    summarise(total_category_spend = sum(category_spend)) %>%
    mutate(Chi = ifelse(Unit == "Chicago", "yes", "no"))
  
  chart <- chart_data %>% 
    ggplot(aes(x = Unit, y = total_category_spend/1000, fill = Chi)) +
             geom_bar(stat = "identity") +
             theme(axis.text.x = element_text(angle=65, vjust=0.6)) +
             theme_minimal() +
             labs(title = str_c("Total Spending on", category, sep = " "),
                  y = "Total Spending (000s)") +
             scale_fill_manual(values = c(yes = "#41B6E6", no = "#4D4D4D"), guide = F)
  
  print(chart)
  print(str_c("The", category, "Chart is Done", sep = " "))
  dev.off()
}

categories <- unique(vpr_grouped$Category)

categories %>% walk(chart)
		</code>
		</pre>
		</details>
		</div>
	</section>
	
	<section>
		<a href="generic.html" class="image">
			<img src="{% link assets/images/pic09.jpg %}" alt="" data-position="top center" />
		</a>
		<div class="content">
			<div class="inner">
				<header class="major">
					<h3>Rhoncus magna</h3>
				</header>
				<p>Nullam et orci eu lorem consequat tincidunt vivamus et sagittis magna sed nunc rhoncus condimentum sem. In efficitur ligula tate urna. Maecenas massa sed magna lacinia magna pellentesque lorem ipsum dolor. Nullam et orci eu lorem consequat tincidunt. Vivamus et sagittis tempus.</p>
				<ul class="actions">
					<li><a href="generic.html" class="button">Learn more</a></li>
				</ul>
			</div>
		</div>
	</section>
	<section>
		<a href="generic.html" class="image">
			<img src="{% link assets/images/pic10.jpg %}" alt="" data-position="25% 25%" />
		</a>
		<div class="content">
			<div class="inner">
				<header class="major">
					<h3>Sed nunc ligula</h3>
				</header>
				<p>Nullam et orci eu lorem consequat tincidunt vivamus et sagittis magna sed nunc rhoncus condimentum sem. In efficitur ligula tate urna. Maecenas massa sed magna lacinia magna pellentesque lorem ipsum dolor. Nullam et orci eu lorem consequat tincidunt. Vivamus et sagittis tempus.</p>
				<ul class="actions">
					<li><a href="generic.html" class="button">Learn more</a></li>
				</ul>
			</div>
		</div>
	</section>
</section>

<!-- Three -->
<section id="three">
	<div class="inner">
		<header class="major">
			<h2>Massa libero</h2>
		</header>
		<p>Nullam et orci eu lorem consequat tincidunt vivamus et sagittis libero. Mauris aliquet magna magna sed nunc rhoncus pharetra. Pellentesque condimentum sem. In efficitur ligula tate urna. Maecenas laoreet massa vel lacinia pellentesque lorem ipsum dolor. Nullam et orci eu lorem consequat tincidunt. Vivamus et sagittis libero. Mauris aliquet magna magna sed nunc rhoncus amet pharetra et feugiat tempus.</p>
		<ul class="actions">
			<li><a href="generic.html" class="button next">Get Started</a></li>
		</ul>
	</div>
</section>

</div>
