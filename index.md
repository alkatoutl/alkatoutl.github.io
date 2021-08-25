# Building TransitHealth

Would you bet that most taxis drop off their riders inside or outside their same pick up location?

You’re probably thinking people who take taxis are trying to get somewhere further away than where they currently are, but let's take a look at the data:

![Image](https://i.ibb.co/1JphCYR/taxisnip.jpg)

Contrary to popular belief, the majority of taxi rides pickup and drop off their riders within the same neighborhood area! 

Utilizing raw data given from the city, I was able to answer more questions about taxi rides in the city and contribute to a website that helped to inform chicagoans about transit and health patterns in Chicago.

# About Transithealth

The website I’m talking about is called [TransitHealth](https://scarletstudio.github.io/transithealth ) and it was developed by a team of interns, including myself, with the help of professional software engineers. What we each did was we took raw data on different statistics provided by the city of Chicago and used Python to ingest and transform those large datasets in an offline pipeline. Each of us then used Flask and SQL to implement features in a RESTful API and write unit tests as well as JavaScript (React) to implement features in the frontend- a full-stack development project!

# My Contributions

## Extracting, Transforming, & Loading
I had the responsibility of looking over the [Taxi Trip Data](https://data.cityofchicago.org/Transportation/Taxi-Trips/wrvz-psew) provided by the [Chicago Data Portal](https://data.cityofchicago.org/). I had to decide what information would be most beneficial to users visiting our site and- using the same methodology as described above- I extracted, transformed, and loaded that data to our offline pipeline.

To extract the data that I thought would help provide the most useful insights to our users, I used the following SQL query:
```markdown
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
```markdown
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
```markdown
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

## Taxi Trip Metrics

My first task was to find metrics that could fit nicely on our website's [Explorer Scatter View](https://scarletstudio.github.io/transithealth/scatter).

![Image](https://i.ibb.co/zSCTX7h/scatterv.jpg)

Based on the data I extracted I thought it would be useful for our users to know the average speed of taxis per pickup and dropoff area. 
To do this I wrote the appropriate queries in the following code:
```markdown
from api.utils.database import rows_to_dicts

#multiple metrics for the taxi trip dataset
class TaxiTripMetrics:
    def __init__(self, connection):
        self.connection = connection
        
    #returns avergae trip speed per pickup area    
    def get_avg_speed_per_pickup(self):
        query = """
        SELECT pickup_community_area as area_number, AVG(trip_miles/(trip_minutes/60)) as value
        FROM taxitrips
        GROUP BY pickup_community_area
        """
        cur = self.connection.cursor()
        cur.execute(query)
        rows = rows_to_dicts(cur, cur.fetchall())
        return rows
        
    #returns average trip speed per dropoff area
    def get_avg_speed_per_dropoff(self):
        query = """
        SELECT dropoff_community_area as area_number, AVG(trip_miles/(trip_minutes/60)) as value
        FROM taxitrips
        GROUP BY dropoff_community_area
        """
        cur = self.connection.cursor()
        cur.execute(query)
        rows = rows_to_dicts(cur, cur.fetchall())
        return rows
```
I then wrote unit tests for my code to make sure it worked given a smaller set of data which you can see [here.](https://github.com/scarletstudio/transithealth/blob/main/api/metrics/taxitrips_test.py)

When running the tests, I ran into an interesting problem. The tests I wrote were failing but when I ran my code on the actual data it worked- a common issue any software engineer runs into. 

After some help from one of the mentors, Vinesh Kannan, we realized what the problem was. We were using SQLite and the AVG() function in that language has undefined behavior for integers meaning we need to make sure we use float values. After changing the integers in my unit tests to floats, both tests finally passed.

However, I looked back at the [table](https://alkatoutl.github.io/#extracting-transforming--loading) I created for the extracted data and confirmed that I did take trip_miles in as integers, so why was my code working on the actual data the whole time? Shouldn't it have failed the same way my unit tests did?

Turns out, SQLite has flexible "type affinity" meaning that columns with a certain type can store values that are of a different type. In the case of the table I created, I had the column trip_minutes as integers but the data values from the [Data Portal](https://data.cityofchicago.org/Transportation/Taxi-Trips/wrvz-psew) for trip_minutes were floats so my table held the data as floats and not integers, allowing it to work with SQLite's AVG() function.

After figuring all of that out, my data was ready to go live on our [Explorer Scatter View](https://scarletstudio.github.io/transithealth/scatter).
I added my metric functions to the Scatter View's endpoint created using Flask by including these two lines of code in its supported metrics:
```markdown
supported_metrics = {
...
"avg_speed_per_dropoff": lambda: metric_tt.get_avg_speed_per_dropoff(),
"avg_speed_per_pickup": lambda: metric_tt.get_avg_speed_per_pickup(),
... }
```
and then used JavaScript to finally add my metric to the website, giving the final results!

![Image](https://i.ibb.co/YtrxW9q/scatter1.jpg)   ![Image](https://i.ibb.co/WvRSX3D/scatter2.jpg)

## Taxi Trip Data Questions

After adding to [TransitHealth's Scatter View](https://scarletstudio.github.io/transithealth/scatter), I then worked on what questions our users would have that couldn't be answered by a scatter plot. Again, based on the data I extracted, I thought it would be most useful to know, for taxis, the most common drop off locations per pickup area and the most common form of payment for every pickup and drop off area. 

To do this I wrote the following queries in the same code format as I did for the taxi metrics before:
```markdown
from api.utils.database import rows_to_dicts

#multiple taxi trip questions
class TaxiTripQuestions:
    def __init__(self, connection):
        self.connection = connection
   
    #takes in pickup_community_area and returns most common dropoff_community_area
    def most_common_dropoff(self):
        query = """
        SELECT
            pickup_community_area,
            dropoff_community_area,
            max(count) as max_count,
            CAST(max(count) as FLOAT) / sum(count)  as percentage
        FROM (
            SELECT
                pickup_community_area,
                dropoff_community_area,
                CAST(count(1) as FLOAT) as count
            FROM taxitrips
            GROUP BY
                pickup_community_area,
                dropoff_community_area
            ) 
        GROUP BY pickup_community_area
        """
        cur = self.connection.cursor()
        cur.execute(query)
        rows = rows_to_dicts(cur, cur.fetchall())
        return rows
       
**The same queries were used for the most used payment types per pickup and drop off location 
except pickup_community_area or dropoff_community_area (depending on the function) and payment_type 
were selected instead of both pickup_community_area and dropoff_community_area**       
```
Originally, though, I had the following query in the code:
```markdown
SELECT q.pickup_community_area, q.dropoff_community_area
        FROM(
            SELECT p.pickup_community_area, p.dropoff_community_area, p.count
            FROM (
                SELECT pickup_community_area, dropoff_community_area, count(1) as count
                FROM  taxitrips
                GROUP BY pickup_community_area, dropofff_communtiy_area
            ) as p
            GROUP BY p.pickup_community_area, p.count
        ) as q
        GROUP BY q.pickup_community_area
```
Opposite to what happened with the [taxi metrics](https://alkatoutl.github.io/#taxi-trip-metrics), this code passed the [unit tests](https://github.com/scarletstudio/transithealth/blob/main/api/questions/taxitrips_test.py) that I wrote but failed to pass the tests on GitHub when committed. 

After some brainstorming with one of our mentors, we figured out what went wrong. The way I was using the GROUP BY function was having the query return random drop off areas for every pickup area instead of the most common one. So, while I was lucky to pass my own unit tests, GitHub put a halting stop to that! 

To fix this, as you see in the first codeblock in this section, I added the count function to count the number of unique pickup area and drop off area combinations and then used the MAX function to pick the combination with the highest count for each pickup area. I also decided to add a percentage attribute to help the user visualize and understand the actual commonness of the dropoff areas so I needed to cast the counts as floats.

For the most common payment method for pickup and drop off areas I used the same exact methodology and you can view it [here.](https://github.com/scarletstudio/transithealth/blob/main/api/questions/taxitrips.py)

I then created my own endpoint using Flask:
```markdown
@app.route("/question/taxitrips")
    def taxitrips():
        most_common_dropoff = metric_tt.most_common_dropoff()
        payment_per_pickup = metric_tt.get_payment_type_by_pickup()
        payment_per_dropoff = metric_tt.get_payment_type_by_dropoff()
        return jsonify({
            "most_common_dropoff": most_common_dropoff,
            "payment_per_pickup": payment_per_pickup,
            "payment_per_dropoff": payment_per_dropoff
        })
```
and then it was time to use JavaScript to add it to our website and implement the frontend design of the data presented to our users.

For the frontend design I used the following structure (for the full code you can look [here](https://github.com/scarletstudio/transithealth/blob/main/app/components/questions/TaxiMostCommonDropoff.js) for the most common dropoff locations and [here](https://github.com/scarletstudio/transithealth/blob/main/app/components/questions/TaxiPaymentMethod.js) for the most common payment method per pickup and drop off locations):
```markdown
- Create constant pointing to corresponding endpoint
- Create constant representing columns
- Function that accesses endpoint and returns appropriate set of data
- Create Table component from React
```
After that the data questions were ready to be added to the site using JavaScript:
```markdown
...},
"taxi-most-common-dropoff": {
    title: "Where are people usually getting dropped off by taxis?",
    author: "Leilah Alkatout",
    component: "taxi-most-common-dropoff",
    description: "People get dropped off in different community areas by [..]",
  },
  "taxi-popular-payment-method":{
    title: "What is the most popular payment method for taxis?",
    author: "Leilah Alkatout",
    component: "taxi-payment-method",
    description: "People in different areas of Chicago prefer using [...],
  }, ...
```
where the `component` comes from the imported frontend component desccribed previously.

After that, the data questions were live! You can view all of them [here](https://scarletstudio.github.io/transithealth/questions).

![Image](https://i.ibb.co/FHk3dtT/dq2.jpg)
