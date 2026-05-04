# Notes

## Transformation and cleaning

- new column: Flight Time 
```m
= Table.AddColumn(#"Changed Type", "Flight Time", each [Arrival Time] - [Departure Time])
```

- new column: Fuel Cost/Seat
*Searched the average number of seats for a commercial planes and average cost of fuel per flight. Naturally, these values vary based on the aircraft type, but I used the average values to calculate the fuel cost per seat, which is more relevant for our analysis since we only have one customer per plane in this dataset.
average number of seats - 180
average fuel cost per flight - 13000*
```m
= Table.AddColumn(#"Changed Type1", "Fuel Cost/Seat", each Value.Divide([Fuel Cost], 180))
```

- new column: Age Bracket
*Created custom age brackets to categorise passengers into different age groups to allow for further analysis based on age demographics. The brackets are defined as follows: Under 18, 18-34, 35-49, and 50+.
```m
 = Table.AddColumn(#"Changed Type2", "Age Bracket", each if [Passenger Age] < 18 then "Under 18"
				else if [Passenger Age] < 29 then "18-29"
				else if [Passenger Age] < 39 then "30-39"
				else if [Passenger Age] < 49 then "40-49"
				else if [Passenger Age] < 59 then "50-59"
				else if [Passenger Age] < 69 then "60-69"
				else if [Passenger Age] < 79 then "70-79"
				else "80+"),
```

- new column: DateKey
*Created a DateKey column in the format DD/MM/YYYY to facilitate easier join for dim_Date table in the data model.
```m
= Table.AddColumn(#"Added Custom2", "DateKey", each Date.From([Departure Time]))
```


While ispecting the dataset I've noticed few duplicated values in the [Flight Number] column. Upon closer investigation I've decided that this is fine as these duplicates are all linked to different airports of origin. This is because the same flight number can be used for different routes and destinations, which is common in the airline industry. Therefore, I have decided to keep these duplicates in the dataset as they do not indicate any data quality issues and are relevant for the analysis.


--- 
Created dim_Date table in PQ using the following code:
```m
let
    StartDate = #date(2023, 1, 1),
    Today = Date.From( DateTimeZone.SwitchZone( DateTimeZone.UtcNow(), 10) ),
    EndDate = Date.EndOfYear( Date.AddYears( Today, 0) ),
    Length = Duration.Days( EndDate - StartDate ) + 1,
    Source = List.Dates(StartDate, Length, #duration(1,0,0,0)),
    #"Converted to Table" = Table.FromList(Source, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    #"Renamed Columns" = Table.RenameColumns(#"Converted to Table",{{"Column1", "Date"}}),
    #"Added DateKey" = Table.AddColumn(#"Renamed Columns", "DateKey", each Int64.From(Date.ToText([Date], "yyyyMMdd")), Int64.Type),
    #"Inserted Year" = Table.AddColumn(#"Added DateKey", "Year", each Date.Year([Date]), Int64.Type),
    #"Inserted Month" = Table.AddColumn(#"Inserted Year", "Month", each Date.Month([Date]), Int64.Type),
    #"Inserted Month Name" = Table.AddColumn(#"Inserted Month", "Month Name", each Date.MonthName([Date]), type text),
    #"Inserted MMM" = Table.AddColumn(#"Inserted Month Name", "MMM", each Text.Start([Month Name], 3), type text),
    
    #"Inserted M Month" = Table.AddColumn(#"Inserted MMM", "M", each Text.Start([MMM], 1) & Text.Repeat(Character.FromNumber(8203), [Month]), type text),
    
    #"Inserted Day of Week" = Table.AddColumn(#"Inserted M Month", "Day of Week", each Date.DayOfWeek([Date], Day.Monday)+1, Int64.Type),
    #"Inserted Day Name" = Table.AddColumn(#"Inserted Day of Week", "Day Name", each Date.DayOfWeekName([Date]), type text),
    #"Inserted DDD" = Table.AddColumn(#"Inserted Day Name", "DDD", each Text.Start([Day Name], 3), type text),
    #"Inserted D Day" = Table.AddColumn(#"Inserted DDD", "D", each Text.Start([DDD], 1) & Text.Repeat(Character.FromNumber(8203), [Day of Week]), type text),
    #"Inserted YYMM" = Table.AddColumn(#"Inserted D Day", "YYMM", each (Date.Year([Date]) * 100) + Date.Month([Date]), Int64.Type),
    #"Inserted MMMYY" = Table.AddColumn(#"Inserted YYMM", "MMMYY", each [MMM] & " " & Text.End(Text.From(Date.Year([Date])), 2), type text),
    #"Added MonthID" = Table.AddColumn(#"Inserted MMMYY", "MonthID", each (Date.Year([Date]) - Date.Year(StartDate))*12 + Date.Month([Date]), Int64.Type),
    #"Added Quarter" = Table.AddColumn(#"Added MonthID", "Quarter", each Date.QuarterOfYear([Date]), Int64.Type),

    #"Changed Type" = Table.TransformColumnTypes(#"Added Quarter",{
        {"Date", type date}, 
        {"DateKey", Int64.Type},
        {"Year", Int64.Type}, 
        {"Month", Int64.Type}, 
        {"Month Name", type text}, 
        {"MMM", type text}, 
        {"M", type text}, 
        {"Day of Week", Int64.Type}, 
        {"Day Name", type text}, 
        {"DDD", type text}, 
        {"D", type text}, 
        {"YYMM", Int64.Type}, 
        {"MMMYY", type text}, 
        {"MonthID", Int64.Type}, 
        {"Quarter", Int64.Type}
    })
in
    #"Changed Type"
```
---
added dim_geoData with longitude and latitude for each airport to allow for geospatial analysis and visualisation in Power BI. This will enable us to create maps and perform spatial analysis based on the location of the airports.

---
Colour schema for airports:

|Airport|City|Hex|
|--------|----|---|
|SEA|Seattle|#1A6FBF|
|BOS|Boston|#C93B3B|
|ATL|Atlanta|#E07B1A|
|DEN|Denver|#2EA06E|
|JFK|New York|#7B52C7|
|MIA|Miami|#C4447E|
|ORD|Chicago|#3A9BB8|
|DFW|Dallas|#B87D14|
|LAX|Los Angeles|#5B8C28|
|SFO|San Francisco|#5A6472



