---
title: "US Universities Map using Leaflet"
author: "Manan Rajvir"
date: "13 July 2020"
output: 
        html_document:
        keep_md: true
---
```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
library(knitr)
```


# Synopsis:  
The aim of this project is to help students looking for Universities in USA for Undergraduate or Master's degrees. The leaflet package has been used to create the map and visualize different factors. Two important factors that currently affect foreign students are - Racial Crimes and Covid Cases. These have been mapped, along with the different Universities in the United States.

# Datasets Used:  
1. [University Dataset](https://nces.ed.gov/ipeds/datacenter/DataFiles.aspx?goToReportId=7):  
The list of universities and their details used, is from the Integrated Postsecondary Education Data System. The dataset includes several details about location, highest degree and financial aid for different universities and colleges across the United States.  
2. [Hate Crime Dataset](https://ucr.fbi.gov/hate-crime/2018/topic-pages/jurisdiction):  
The hate crime statistics provided by FBI's Uniform Crime Reporting (UCR). It lists the number of hate crime incidents reported across different states in the United States in 2018.  
3. [Covid Dataset](https://www.cdc.gov/covid-data-tracker/index.html#cases):  
The data related to Covid-19 cases has been obtained from the official Centers for Disease Control and Prevention (CDC) website. The number of cases are as of Jun 26, 2020 5:45PM.

# Creating the Map:
One of the advantages of using the leaflet package, is that the map can be built layer by layer. Here, I will be explaining how each layer of the map has been created. The final step will then combining these different layers to form the final interactive map.

### Loading the Libraries
```{r, results = "hide",message=FALSE,warning=FALSE}
library(dplyr)
library(leaflet)
library(leaflet.extras)
library(rgdal)
library(htmltools)
library(httr)
```

### Extracting Data from the Datasets:
```{r}
universities <- read.csv("names.csv")[,c("UNITID","INSTNM","CONTROL","LONGITUD","LATITUDE","ICLEVEL","HLOFFER")]
universities <- filter(universities,CONTROL > 0)  ## Remove universities whose Sector is not known
universities <- filter(universities,ICLEVEL <= 2 & ICLEVEL >= 1)  ## Remove universities whose programs are less than 2 years
universities <- filter(universities,HLOFFER >= 7)  ## Remove universities who do not offer degrees up to the Master's Degree.
```
From the Universities database, the columns we are interested in are:  
1. **UNITID** - University ID  
2. **INSTNM** - Institute Name  
3. **CONTROL** - The sector to which the university belongs (1 = Public, 2 = Private and 3 = For-Profit)  
4. **LONGITUD** and **LATITUDE** - The longitude and latitude of the University Campus  
5. **ICLEVEL** - A classification of how long the institutes programs are (1 = 4 or more years, 2 = 2 to 4 years and 3 = less than 2 years)  
6. **HLOFFER** - Represents the highest level of offering (6 = Postbaccalaureate Certificate, 7 = Master's Degree and 8 = Post Master's Degree ) 


The dataset is then filtered so that we only have universities whose sector is known, who offer atleast 2 year long programs, and whose highest degree offering is atleast Master's.  

```{r}
hate_crime <- read.csv("table-12.csv")
hate_crime <- hate_crime[,c("Participating.state.Federal","Total.number.of.incidents.reported")]
```
From the hate crime dataset, we are only interested in the State and the Number of Incidents reported columns.  

```{r, results = 'hide'}
covid <- read.csv("covid_data.csv")[,c("jurisdiction","Total.Cases")]
states <- readOGR("cb_2018_us_state_500k.shp")
```
From the covid dataset, we are only interested in the States and Total number of cases in the state columns. Additionally, we will also be using a .shp file for marking the boundaries of states.

### Creating the Base Map:
```{r}
base_map <- universities %>% leaflet() %>% setView(lat = 39.8282, lng = -98.5795, 3) %>% addProviderTiles(providers$Stamen.Toner)
base_map
```
The base map have been created using the leaflet function, and we have used setView to centre the view of the map over the United States.

### Adding Polygons:
The data for Covid Cases and Hate Crime across the states will be mapped using the addPolygon function.
```{r,warning=FALSE}
states <- subset(states, is.element (states$NAME, hate_crime$Participating.state.Federal))
hate_crime <- hate_crime[order(match(hate_crime$Participating.state.Federal,states$NAME)),]
labels <- paste0("<b>",hate_crime$Participating.state.Federal, "</b>", "</br>", "Hate Crime Incidents (2018): ",hate_crime$Total.number.of.incidents.reported)
bins <- c(0,100,200,300,400,500,600)
pal2 <- colorBin("Blues",domain = hate_crime$Total.number.of.incidents.reported,bins = bins)
map_crime <- base_map %>% addProviderTiles(providers$Stamen.Toner) %>% addPolygons(data = states, weight = 1, smoothFactor = 0.5, color = "white", fillOpacity = 0.8, fillColor = pal2(hate_crime$Total.number.of.incidents.reported), label = lapply(labels,HTML),group = "Crime") %>% addLegend(pal = pal2, values = hate_crime$Total.number.of.incidents.reported, opacity = 0.7, position = "bottomleft",title = "Number of Hate Crimes (2018)")

map_crime
```
Initially we subset the states names from shp file and reorder our dataset, so that the order of the states in the shp file, matches the order of the states in our dataset. The color-palette for the polygon has been created using the colorBin function, that creates a color palette based on the values of the hate crime incidents. The addPolygons function has been used to highlight the number of hatecrimes in different states and color code them based on this data. Additionally, we have also addded a label to show the exact figures for each state.  
The same process has been used to map the Covid data.
```{r}
states <- subset(states, is.element(states$NAME,covid$jurisdiction))
covid <- covid[order(match(covid$jurisdiction,states$NAME)),]
labels2 <- paste0("<b>",covid$jurisdiction, "</b>", "</br>", "Total Covid Cases: ", covid$Total.Cases)
bins2 <- round(seq(0,200000,length.out = 10))
pal3 <- colorBin("Oranges",domain = covid$Total.Cases,bins = bins2)
map_covid <- base_map %>% addPolygons(data = states, weight = 1, smoothFactor = 0.5, color = "white", fillOpacity = 0.8, fillColor = pal3(covid$Total.Cases),label = lapply(labels2,HTML), group = "Covid") %>% addLegend(pal = pal3, values = covid$Total.Cases, opacity = 0.7, position = "bottomright",title = "Number of Covid Cases")

map_covid
```


### Adding Universities:
Now that we have added the polygons to represent the Crime Rate and Covid Data, we can add the markers for different universities. I prefer using Circle Markers for this, over the normal markers, as the normal markers take up a lot of space, and can get congested. The markers can be placed based on the longitude and latitude of the university, using the addCircleMarkers() function. For this map, we have divided the universities in 3 groups - Private, Public and For-Profit and each of these universities will be represented with a different colour. Additionally, we will also add labels that show the names of the university.
```{r}
us_public <- universities %>% filter(CONTROL == 1)
us_private <- universities %>% filter(CONTROL == 2)
us_forprofit <- universities %>% filter(CONTROL == 3)
pal <- colorFactor(palette = c("red", "blue", "green"), levels = c(1, 2, 3))
map_universities <- base_map %>% addCircleMarkers(data = us_public, ~LONGITUD, ~LATITUDE, color = ~pal(CONTROL), label = ~htmlEscape(INSTNM), group = "Public",radius = 1) %>% addCircleMarkers(data = us_private, ~LONGITUD, ~LATITUDE, color = ~pal(CONTROL), label = ~htmlEscape(INSTNM), group = "Private",radius=1) %>% addCircleMarkers(data = us_forprofit, ~LONGITUD, ~LATITUDE, color = ~pal(CONTROL), label = ~htmlEscape(INSTNM), group = "For-Profit",radius=1)

map_universities

```


### Combining the layers:
So far we have created 4 layers for the map:    
1. **Base Layer**  
2. **Crime Layer** - With poygons to represent Hate Crime data  
3. **Covid Layer** - With polygons to represent Covid Cases data    
4. **University Layer** - With markers to show university location    
We can now add these layers to a single map object. To make the map more interactive and easier to read, we will add the use the addLayersControl() function, that allows the user to customize the layers he wants to view.
```{r,warning=FALSE}
final_map <- base_map %>% addProviderTiles(providers$Stamen.Toner) %>% addPolygons(data = states, weight = 1, smoothFactor = 0.5, color = "white", fillOpacity = 0.8, fillColor = pal2(hate_crime$Total.number.of.incidents.reported), label = lapply(labels,HTML),group = "Crime") %>% addLegend(pal = pal2, values = hate_crime$Total.number.of.incidents.reported, opacity = 0.7, position = "bottomleft",title = "Number of Hate Crimes (2018)") %>% addPolygons(data = states, weight = 1, smoothFactor = 0.5, color = "white", fillOpacity = 0.8, fillColor = pal3(covid$Total.Cases),label = lapply(labels2,HTML), group = "Covid") %>% addLegend(pal = pal3, values = covid$Total.Cases, opacity = 0.7, position = "bottomright",title = "Number of Covid Cases") %>% addCircleMarkers(data = us_public, ~LONGITUD, ~LATITUDE, color = ~pal(CONTROL), label = ~htmlEscape(INSTNM), group = "Public",radius = 1) %>% addCircleMarkers(data = us_private, ~LONGITUD, ~LATITUDE, color = ~pal(CONTROL), label = ~htmlEscape(INSTNM), group = "Private",radius=1) %>% addCircleMarkers(data = us_forprofit, ~LONGITUD, ~LATITUDE, color = ~pal(CONTROL), label = ~htmlEscape(INSTNM), group = "For-Profit",radius=1) %>% addLayersControl(overlayGroups = c("Crime","Covid","Public","Private","For-Profit"))   

final_map
```

**Note: The Control Option in the topright of the map can be used to select the required layers / data.**

### Extras:
We can also add some extra features, using the leaflet.extras package that allow us to reset the zoom, and search universities based on their names.
```{r}
final_map <- final_map %>% addSearchFeatures(targetGroups = c("Public","Private","For-Profit"), options = searchFeaturesOptions(zoom = 10)) %>% addResetMapButton()

final_map
```

