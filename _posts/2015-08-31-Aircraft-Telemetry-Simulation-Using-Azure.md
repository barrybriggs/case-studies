---
layout: post
title:  "Aircraft Telemetry Simulation Using Azure"
author: "Barry Briggs"
author-link: "http://blogs.msdn.com/b/msarchitecture/"
#author-image: "{{ site.baseurl }}/images/BarryBriggs/photo.jpg" //should be square dimensions
date:   2015-07-21 23:34:28
categories: Aircraft, telemetry, Azure, Event Hubs, Stream Analytics, Storage
color: "blue"
#image: "{{ site.baseurl }}/images/imagename.png" #should be ~350px tall
---

A vendor of aircraft avionics wanted to understand how the Microsoft cloud could be used to capture weather events (such as [PIREPS](http://aviationweather.gov/airep), “Pilot Reports”) generated by aircraft in flight, and to provide information of interest from those aircraft to others in the vicinity. In real life these events would be visualized on an in-cockpit display called the Electronic Flight Bag (EFB), a very simple emulation of which is included in the solution (for more information on the EFB, see [here](https://en.wikipedia.org/wiki/Electronic_flight_bag)). The objective, ultimately, is to give the pilot near-real-time information to enable decisions that will make the flight faster, safer, and more comfortable. 

In this Case Study you will learn how to easily connect an application to Azure services such as Event Hubs, and (as well) how to quickly build an application using the powerful Bing Maps API.

##Overview of the Solution##

The solution consists of a WPF client application which emulates the EFB, as discussed above. The application displays a map retrieved using the Bing Maps API and the WPF Bing Maps control, available from [here](http://www.microsoft.com/maps/choose-your-bing-maps-API.aspx). Note that to run the application you will also need to procure a Bing Maps key, also available at that link.
 
This simple application emulates an aircraft in flight. The user enters an airline, a flight number (neither of which is validated) and an origin and destination (three-character airport codes). Pushing a button starts the “flight,” and along the way the user can record various weather events, such as icing, turbulence and wind shear, using buttons at the bottom of the display. Every thirty miles (configurable) the simulated aircraft sends a weather report to Azure. 

The weather data structure is a simple JSON document serialized from this C# class:

    public class AirTelemetry : TableEntity
    {
        public DateTimeOffset dto { get; set; }
        public string inflight { get; set; }
        public string airline { get; set; }
        public string flight { get; set; }
        public double lat { get; set; }
        public double lng { get; set; }
        public double heading { get; set; }
        public double airspeed { get; set; }
        public double altitude { get; set; }
        public DateTime arrival { get; set; }
        public double temp { get; set; }
        public double windshear { get; set; }
        public double ice { get; set; }
        public double windspeed { get; set; }
        public double lightning { get; set; }
        public AirTelemetry() { }

    }

Using the Event Hub API this is sent to Azure Event Hubs.
 
        await eventHubClient.SendAsync(new EventData(Encoding.UTF8.GetBytes(json)));

In Azure, three components handle these events: the Event Hub, a Stream Analytics instance, and an Azure Table. The function of Stream Analytics is simply to take events from Event Hub and store them in the table using a wild card query (SELECT * FROM air) where “air” is the Event Hub.

Here is an example of the user interface (note the weather buttons at the bottom):
![Photo]({{site.baseurl}}/images/2015-08-31-Aircraft-Telemetry-images/Fig1.png)


Every thirty miles (configurable) the application looks for new weather events within the immediate vicinity by querying the Azure Table. Here is an example showing the presence of a thunderstorm in the immediate flight path:
![Photo]({{site.baseurl}}/images/2015-08-31-Aircraft-Telemetry-images/Fig2.png)

###Haversine Algorithm and the Great Circle###

This project has an implementation of the Haversine algorithm ([code](https://github.com/barrybriggs/aircraft/blob/master/Plane/Haversine.cs)) for determining the distance and bearing between two points given by latitude and longitude, taking into account the radius of the Earth. More information can be found in this [Wikipedia article](https://en.wikipedia.org/wiki/Haversine_formula) and in this excellent [article](http://www.movable-type.co.uk/scripts/latlong.html) which provides an implementation in JavaScript. As can be seen from the screenshots all routes are calculated using the [Great Circle](https://en.wikipedia.org/wiki/Great_circle) method. 

This code is kept in a static class and is very reusable. 

###Bing Maps Use###

The map display uses the [Bing Maps WPF control](http://msdn.microsoft.com/en-us/library/hh750210.aspx) which makes application development extremely fast and straightforward. Many other (including pure Web-based) approaches are possible but since real EFB’s are intelligent devices, a thick client seemed reasonable for this proof of concept. 

The application internally keeps a dictionary of three-letter airport codes to place names. It then uses the Bing Map API to resolve those place names to latitude and longitude (in a production version it would probably be faster to hardcode the coordinates, but the code is useful for demonstration and learning purposes):

    static int GetLatLongFromName(string name, out double latitude, out double longitude)
       {
         string url = "http://dev.virtualearth.net/REST/v1/Locations/";
         string resp;
         latitude = 0.0;
         longitude = 0.0;

         WebClient wc = new WebClient();
         url += name;
         url += '?' + "o=xml&key=" + bingKey;
         wc.Headers.Add("user-agent", "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.2;.NET CLR 1.0.3705;)");

         Stream data = wc.OpenRead(url);

This returns an XML document which is easily parsed to get the latitude and longitude.

###Azure Usage###
The application uses three Azure components connected together to form a pipeline.

###Azure Event Hubs###

[Azure Event Hubs](http://azure.microsoft.com/en-us/services/event-hubs/) creation and configuration is quickly accomplished within the Service Bus tab of the Azure portal. Note that to use any Service Bus components you must have a Service Bus connection string of the form:

    Endpoint=sb://[YOUR NAMESPACE].servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=”YOUR KEY GOES HERE”

An Event Hub may have many partitions to spread the load. It also enables multiple users, called Consumer Groups, to read from the event queue.

![Photo]({{site.baseurl}}/images/2015-08-31-Aircraft-Telemetry-images/Fig3.png)


###Azure Stream Analytics###

[Azure Stream Analytics](http://azure.microsoft.com/en-us/services/stream-analytics/) provides a way to read and filter events in real time. For our purposes, ASA requires input, query and output configuration. In the picture below we have configured ASA to read from the Event Hub named “air.”

![Photo]({{site.baseurl}}/images/2015-08-31-Aircraft-Telemetry-images/Fig4.png)
 
In the picture below, we have taken the default filter – SELECT * FROM AIR – to select all the events. Note the SQL syntax of the filter which could be much more sophisticated; for example, WHERE clauses could be used to only allow certain events to be stored.  
![Photo]({{site.baseurl}}/images/2015-08-31-Aircraft-Telemetry-images/Fig5.png)
 
Finally below, we have configured the events to be stored in an Azure Table. 

![Photo]({{site.baseurl}}/images/2015-08-31-Aircraft-Telemetry-images/Fig6.png)
###Azure Table Storage###
 
The data is stored in an [Azure Table](http://azure.microsoft.com/en-us/services/storage/). The output from the fictional “Frontline” airline looks like this (as visualized in Cerebrata’s Azure Management Studio):

![Photo]({{site.baseurl}}/images/2015-08-31-Aircraft-Telemetry-images/Fig7.png)
 
Setting this pipeline up took a matter of minutes, and is a good illustration of the simplicity of the Azure environment. 

(One minor glitch: Azure Stream Analytics at the time of coding this app type-checked the data and threw an exception for invalid timestamps, where a valid timestamp had to be greater than January 1, 1601 (and not January 0, 0000!). This happened because the arrival time was zeroed out until our simulated aircraft made a simulated landing. The error message, which will be fixed, was opaque.) 

###Code Artifacts###
Application code: [http://www.github.com/barrybriggs/Aircraft ](http://www.github.com/barrybriggs/Aircraft)

##Opportunities for Reuse##
There are many possibilities for map-related and telemetry-related applications; in particular as previously noted the Haversine static class is separable from the main application and easy to reuse (see code for more details). 
It should be noted that this is a proof-of-concept created to illustrate various aspects of the Microsoft platform: Bing, Azure, and WPF. There is much that could be done to optimize performance, make better use of the communications network and so on.