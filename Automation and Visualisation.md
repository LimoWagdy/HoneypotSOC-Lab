This page contains all the configurations and important code for Automation and Visualisation components, namely the Function App, Logic App, and Sentinel Workbook.

# 1. Automation

## 1.1 Azure Function App

I set up a Function App in Azure, installed the correct extensions in my VScode, and created a function project. I then wrote the following function in C# (with the generous help of ChatGPT) to read the IP from the HTTP query parameter, look up the corresponding location in the GeoLite2-City.mmdb file, and return JSON to my Logic App (next sub-section):

```
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;
using MaxMind.GeoIP2;
using System.IO;

namespace GeoIPFunction;

public class GeoIPLookup
{
    private readonly ILogger<GeoIPLookup> _logger;
    
    private static readonly string DbPath = Path.Combine(
        AppContext.BaseDirectory,
        "GeoLite2-City.mmdb"
    );

    private static readonly DatabaseReader Reader =
        new DatabaseReader(DbPath);

    public GeoIPLookup(ILogger<GeoIPLookup> logger)
    {
        _logger = logger;
    }

    [Function("GeoIPLookup")]

    public IActionResult Run(
        [HttpTrigger(
            AuthorizationLevel.Function,
            "get"
        )] HttpRequest req)
    {
        string? ip = req.Query["ip"];

        if (string.IsNullOrEmpty(ip))
        {
            return new BadRequestObjectResult(
                "Add ?ip=x.x.x.x"
            );
        }
        
        try
        {
            var result = Reader.City(ip);

            return new OkObjectResult(new
            {
                IP = ip,
                Country = result.Country.Name,
                City = result.City.Name,
                Latitude = result.Location.Latitude,
                Longitude = result.Location.Longitude
            });
        }

        catch
        {
            return new OkObjectResult(new
            {
                IP = ip,
                Error = "No location found"
            });
        }
    }
}
```
 
And added the 'GeoLite2-City.mmdb' path to 'GeoIPFunctionAzure.csproj':
```
<ItemGroup>
	<None Update="GeoLite2-City.mmdb">
		<CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
	</None>
</ItemGroup>
```

## 1.2 Azure Logic App

<img src="./images/Logicapp.png" width="1000" height="350">

The Logic App is triggered every 15 mins using a reoccurring trigger. Then, a 'Run Query and list results' action queries the SecurityEvent table to retrieve all distinct IPs that tried to connect to the VM in the last 20 minutes (EventID=4625 is for failed logins). 

After that, a 'For each' loop takes each IP and runs it through the GeoIPLookup function to retrieve the corresponding location. The JSON data from that response is then sent to a custom table in the Log Analytics Workspace called 'GeoIPAttacks' using the 'Send Data' action.

# 2. Visualisation
## 2.1 Sentinel Workbook
I set up a new Workbook in Sentinel and used the following KQL queries and JSON to generate data I thought would be interesting to look at. I frequently combined the SecurityEvents and GeoIPAttacks_CL tables, and used map, tiles, and basic grid to visualise the results.

### HeatMap
KQL:
```
GeoIPAttacks_CL
| extend
	Latitude = todouble(Latitude_s),
	Longitude = todouble(Longitude_s)
| where isnotempty(Latitude) and isnotempty(Longitude)
| summarize DistinctIPs = dcount(IP_s) by 
	City = City_s, 
	Country = Country_s, Latitude, Longitude
| project 
	DistinctIPs, 
	City,
	Country,
	Latitude,
	Longitude
```

JSON:
```
{
    "type": 3,
    "name": "GeoIP Attack Heat Map",
    "content": {
        "version": "KqlItem/1.0",
        "query": "GeoIPAttacks_CL\n| extend\n    Latitude = todouble(Latitude_s),\n    Longitude = todouble(Longitude_s)\n| where isnotempty(Latitude) and isnotempty(Longitude)\n| summarize\n    DistinctIPs = dcount(IP_s)\n    by City = City_s,\n       Country = Country_s,\n       Latitude,\n       Longitude\n| project\n    DistinctIPs,\n    City,\n    Country,\n    Latitude,\n    Longitude",
        "size": 2,
        "timeContext": {
            "durationMs": 604800000
        },
        "queryType": 0,
        "resourceType": "microsoft.operationalinsights/workspaces",
        "visualization": "map",
        "mapSettings": {
            "locInfo": "LatLong",
            "latitude": "Latitude",
            "longitude": "Longitude",
            "sizeSettings": "DistinctIPs",
            "sizeAggregation": "Sum",
            "opacity": 0.8,
            "labelSettings": "Country",
            "legendMetric": "DistinctIPs",
            "legendAggregation": "Sum",
            "itemColorSettings": {
                "nodeColorField": "DistinctIPs",
                "colorAggregation": "Sum",
                "type": "heatmap",
                "heatmapPalette": "greenRed"
            },
            "numberOfMetrics": 0
        },
        "crossComponentResources": [
            "/subscriptions/aca226f5-9e42-4d1b-b66a-33ac59ee8e76/resourceGroups/rg-soc-lab/providers/Microsoft.OperationalInsights/workspaces/law-soc-lab"
        ],
        "title": "Unique Attacker IPs",
        "headingLevel": 2
    },
    "customWidth": "100",
    "styleSettings": {
        "maxWidth": "100",
        "showBorder": false
    }
}
```

### Attacks per Country
```
SecurityEvent
| where EventID == 4625
| where isnotempty(IpAddress)
| summarize Attempts = count() by IP = IpAddress
| join kind=inner (
	GeoIPAttacks_CL
	| summarize 
	Country = any(Country_s),
	City = any(City_s) by IP = IP_s
) on IP
| summarize Attempts = sum(Attempts) by Country
| order by Attempts desc
```
Visualisation = Tiles
### Unique IPs per Country
```
GeoIPAttacks_CL
| where isnotempty(IP_s)
| where isnotempty(Country_s)
| summarize DistinctIPs = dcount(IP_s) by Country = Country_s
| order by DistinctIPs desc
```
Visualisation = Tiles
### Most Attempts per Attacker
```
SecurityEvent
| where EventID == 4625
| summarize Attempts = count() by IP = IpAddress
| join kind=inner(
	GeoIPAttacks_CL
	| summarize Country = any(Country_s), 
	City = any(City_s) by IP = IP_s
) on IP
| project IP, 
	Attempts, 
	Country, 
	City
| order by Attempts desc
```
Visualisation = Tiles

### Average Time between Attacks per IP
```
let StartTime = datetime(2026-08-12 13:00:00);
let EndTime = datetime(2026-08-12 17:00:00);

SecurityEvent
| where TimeGenerated between (StartTime .. EndTime)
| where EventID == 4625
| where isnotempty(IpAddress)
| sort by IpAddress asc, TimeGenerated asc
| serialize
| extend PreviousTime = prev(TimeGenerated)
| extend PreviousIP = prev(IpAddress)
| where IpAddress == PreviousIP
| extend SecondsBetweenAttempts = datetime_diff('second', TimeGenerated, PreviousTime)
| summarize
Attempts = count() + 1,
AvgSecondsBetweenAttempts = avg(SecondsBetweenAttempts) by IpAddress
| where Attempts > 1
| extend AttackPattern = case(
AvgSecondsBetweenAttempts < 60, "Highly automated (<1 min)",
AvgSecondsBetweenAttempts <= 300, "Likely automated (1–5 min)",
AvgSecondsBetweenAttempts <= 1800, "Repeated (5–30 min)",
"Infrequent (>30 min)"
)
| summarize IPs = count() by AttackPattern
| order by IPs desc
```
Visualisation = Pie chart