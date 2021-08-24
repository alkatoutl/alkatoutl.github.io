# Building TransitHealth

Would you bet that most taxis drop off their riders inside or outside their same pick up location?

You’re probably thinking people who take taxis are trying to get somewhere further away than where they currently are, but let's take a look at the data:

![Image](https://i.ibb.co/1JphCYR/taxisnip.jpg)

Contrary to popular belief, the majority of taxi rides pickup and drop off their riders within the same neighborhood area! 

Utilizing raw data given from the city, I was able to answer more questions about taxi rides in the city and contribute to a website that helped to inform chicagoans about transit and health patterns in Chicago.

# About Transithealth

The website I’m talking about is called TransitHealth and it was developed by a team of interns, including myself, with the help of professional software engineers. What we each did was we took raw data on different statistics provided by the city of Chicago and used Python to ingest and transform those large datasets in an offline pipeline. Each of us then used Flask and SQL to implement features in a RESTful API and write unit tests as well as JavaScript (React) to implement features in the frontend- a full-stack development project!

# My Contributions

## Extracting, Transforming, $ Loading
I had the responsibility of looking over the [Taxi Trip Data](https://data.cityofchicago.org/Transportation/Taxi-Trips/wrvz-psew) provided by the [Chicago Data Portal](https://data.cityofchicago.org/). I had to decide what information would be most beneficial to users visiting our site and- using the same methodology as described above- I extracted, transformed, and loaded that data to our offline pipeline.

To extract the data that I thought would help provide the most useful insights to our users, I used the following SQL query:
```
SELECT date_trunc_ymd(trip_start_timestamp) as ymd, 
    trip_id , 
    taxi_id, 
    trip_miles, 
    trip_seconds/60 as trip_minutes,
    pickup_community_area,
    dropoff_community_area,
    payment_type
    
WHERE trip_end_timestamp > "2021-01-01"


LIMIT 100000
```

and proceeded to transform it with the following code:
```
import sys
sys.path.append("./")

import argparse
import numpy as np
import pandas as pd
from timeit import default_timer as timer
from utils.data import extract_data_portal_dates


cli = argparse.ArgumentParser(description="Transform raw taxi trip data.")
cli.add_argument("--input_file", help="File path to read raw data from.")
cli.add_argument("--output_file", help="File path to write results to.")
args = cli.parse_args()

start = timer()

# handling NULL values in ymd column (dropping the row)
raw_df = pd.read_csv(args.input_file)
df = pd.DataFrame(raw_df.dropna(subset=[
    "ymd"
]))
print(f"Dropped {(len(raw_df) - len(df)):,d} rows with nulls.")

# Parse dates and add week column
df = extract_data_portal_dates(df, col="ymd", prefix="")

#replace null values in pickup/dropoff areas with 0
df["pickup_community_area"] = df["pickup_community_area"].fillna(0)
df["dropoff_community_area"] = df["dropoff_community_area"].fillna(0)

# Write output
df.to_csv(args.output_file, index=False)

# Show summary
end = timer()
secs = end - start
print(f"Transformed and wrote {len(df)} records in {secs:.1f} secs.")
```
Basically, I created an input and output file pathway for the data that could go into our Makefile and setup a few conditions. I removed any rows with an unknown date to reinforce the 2021 constraint and then replaced null values in pickup and drop off areas with the value 0. Looking back, I think it would have been more beneficial to drop those rows as well due to the fact that they don't provide valuable insight other than the amount of reports with unknown pickup/drop off areas. 

Finally, I needed to load my newly extracted and transformed data into our database so I created a table for it:
```
CREATE TABLE taxitrips (
    ymd TEXT,
    trip_id TEXT,
    taxi_id TEXT,
    trip_miles INTEGER,
    trip_minutes INTEGER,
    pickup_community_area INTEGER,
    dropoff_community_area INTEGER,
    payment_type TEXT
);
```
After adding some finishing touches to our Makefile, my data was now ready for me to start using it!


