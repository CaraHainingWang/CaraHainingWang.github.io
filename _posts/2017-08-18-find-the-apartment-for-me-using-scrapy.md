---
layout: post
title: How I Found the Apartment for Me Using Scrapy
published: true
keywords: ["lifehack", "web scraping", "python"]
---

Apartment hunting is a painful experience. If you were like me who is moving
with my newborn baby to the San Francisco Bay Area, you would have to spend even more time browsing through websites to make sure everything is OK before making a commitment on the lease. Besides the general requirement on neighborhood, distance to work, bedroom/bathroom numbers, rents, etc. I also need to make sure it is close to some shuttle stations to facilitate my every day commute. Checking the address of each potential apartment to all nearby shuttle stations becomes extremely tedious since no known website supports the function of taking a list of address and returning only  apartments that are close to at least one of the addresses.

<img src="{{ site.baseurl }}images/apartments_map.png" alt="apartment map" style="width: 500" />

After spending a few hours looking on the Internet. I became really distraught. In fact, I might need to go through this ordeal again and again before we can actually save up enough money to buy a place in the bay area with this soaring housing price. So why not try to build some simple command line tool to generate a list of promising apartments automatically? I am new to web crawlers and I have only one week before I have to find a good place to live, but I believe I can do it!

To make things a little easier, I probably don't need to make complex searches across multiple websites (at least for this time) since I won't consider sub-lease or condos because of my baby. After some searching and comparing, `apartments.com` seems to be a great choice. So let's get started!

## Step 1: Build the Web Crawler

I chose to use python and [Scrapy]("https://scrapy.org/") to create the Web Crawler since they are relatively straight-forward to learn and use.

### Install Scrapy
Scrapy is an application framework for crawling websites and extracting structured data. Even though it was originally designed for web scraping, it can also be used to extract data using APIs or as a general purpose web crawler. To install Scrapy, just run
```bash
pip install scrapy
```
### Create a project
Before starting scraping, we need to set up a new Scrapy project by running the following command in a directory where you'd like to to store your code.
```bash
scrapy startproject findApartments
```
Alternatively, you can use `scrapy startproject findApartments [project_dir]` to
specify project_dir in the command line directly.

`startproject` creates a `findApartments` directory with the following contents:
```python
findApartments/
    scrapy.cfg            # deploy configuration file
    findApartments/             # project's Python module, you'll import your code from here
        __init__.py
        items.py          # project items definition file
        pipelines.py      # project pipelines file
        settings.py       # project settings file
        spiders/          # a directory where you'll later put your spiders
            __init__.py
```
Inside the project folder and run `scrapy`, you can see a list of available scrapy commands.


### Write the `apartments` Spider

Here is what the `ApartmentsSpider` look like.
```python
class ApartmentsSpider(CrawlSpider):
    name = 'apartments'
    allowed_domains = ['www.apartments.com']

    def start_requests(self):
        url = "https://www.apartments.com/"
        input_file_dir = getattr(self, 'input', None)
        with open(input_file_dir) as data_file:
            data = json.load(data_file)
        if data:
            for query in data['queries']:
                area_url = "".join([url, query['area']])
                yield Request(area_url, self.parse)
        else:
            yield Request(url, self.parse)

    def parse(self, response):
        # omit code for extracting apartment_urls
        for url in apartment_urls:
            yield(Request(url, callback=self.parse_apartment))

    def parse_apartment(self, response):
        # omit code for extracting apartment level info ...

        for entry in table_entries:
            # omit code for extracting room level info ...
            yield {
                'name': apt_name,
                'address': apt_address,
                'bathroom_num': bathroom_num,
                'bedroom_num': bedroom_num,
                'min_rent': min_rent,
                'max_rent': max_rent,
                'unit': unit,
                'sqrt_foot': sqrt_foot,
                'avail_date': avail_date,
                'phone': phone,
                'url': response.url,
                'feature_list': feature_list,
            }
        # omit code for extracting next_page info ...
        if next_page:
            yield Request(next_page, callback=self.parse_apartment)
```
I have used `apartments.com` in my case, but the code can be easily modified to extract
info from other website as well. I will talk about how to extract specific info
later using `XPath`. The code looks much neater without those extraction code.

The crawl started by making requests to the URLs defined in the start_urls.

In my case, I would like to make the start_urls to be configurable so that I can specify
which cities / areas I would like to extend my apartment search to.
I use a `json` file as an input.
```json
{
  "queries": [
    {
      "area": "palo-alto-ca"
    },
    {
      "area": "mountain-view-ca"
    }
  ]
}

```
You can add more query entries to fine-tune the initial search, such as mininum
number of bed/bathroom, which are all supported by apartments.com.

The initial urls are determined by `start_requests` function. If you have a pre-determined
initial urls instead of some configurable ones, you can use a short-cut form by directly specify
```python
start_urls = [urls_list]
```
as a class variable instead of using the class method.

After specifying the starting urls, the code further elaborates how each url should be processed using the `parse()` method. In particular, we first extract the urls that link
to the list of apartments for each particular area (the urls are extracted using `XPath` and will be introduced briefly later), and then delegate the extraction of each apartment info to a method called
`parse_apartment`.

To make it easier to see the logic, I have omitted the relatively tedious extraction process. We can see a recursive call at the end that will keep parsing for apartment unless it is the last page available.

### About XPath
XPath expressions provide a very powerful tool for extracting web contents. The primary purpose
of XPath is to address parts of an XML document. In my attempt, since `apartments.com` does not provide an API, I solely rely on `XPath` to get access to every piece of information that I need. I will illustrate how to use `XPath` using some examples from my code, but if you really want to get a better grasp of it, here are a list of resources that can be of help.
* [XPath 1.0 Tutorial](http://zvon.org/comp/r/tut-XPath_1.html)
* [Concise XPath](http://plasmasturm.org/log/xpath101/)
* [Using XPath in Scrapy](https://docs.scrapy.org/en/latest/topics/selectors.html#topics-selectors)

`Scrapy shell` is an important command line tool that can be used to check whether
you have got the right XPath for the contents you need. To access Scrapy shell, simply type
```bash
scrapy shell [url]
```
and use the `response` to test your selectors. For example, you can use
```bash
>>> response.xpath('//title').extract_first()
```  
to get the page title.

Now let's see some example for our parser.

_Example 1_: get the url for all apartments.
The HTML element for urls to some apartment page looks like the following
```html
<article class="diamond placard" data-listingid="some id" data-url="some url">
```
To get all the `data-urls`, we can use the following `XPath` expression.
```python
apartment_urls = response.xpath('//article[contains(@data-url, "www.apartments.com")]/@data-url').extract()
```
Instead of selecting all `article` node, we filtered out some articles by specifying
only articles with `data-url` that contains `www.apartments.com`. We have to do this because there exists some article node that is not linked to any apartment page and we don't want to crawl those urls. The `extract()` method is used to extract the text from `data-url` instead of the actual node. Note that `apartment_urls` is a list and an empty one if there is no element satisfying the `XPath` expression.

_Example 2_: Parsing data from the table
For table data passing, we need to first locate the right table in the page since there
could exist many tables. For our case, the table is under some navigation tab and we
only need to select the one that is active.
```python
table_entries = response.xpath('//div[contains(@class, "active")]/*/*/*/tr')
```
Note that we don't care about the parent, grand-parent and grand-grand-parent node of `tr`, so we can use `*` to replace an actual node. Again, `table_entries` is a list, we can then extract the info we need, for example, the rent for each entry:
```python
for entry in table_entries:
    rent_string = entry.xpath( './td[contains(@class, "rent")]/text()').extract_first()
    avail_date = entry.xpath('./td[contains(@class, "avail")]/text()').re(r'(\w+)')
    ...
```
For each entry, we use relative addressing `./` instead of `//`.
The difference is that '//' will select every node from the page by searching from the root node, while `./` will only search in the child node from the current one. `extract_first()` is like `extract()`, but returns only the first element from the list and returns `None` if there is nothing to extract. For `avail_date`, I use a regular expression method `re()` to get the right format, which gets rid of some annoying white spaces.

### Some Data Cleaning
After we get all the information we need, we may still need to clean up the data a little more before storing it. For example, we need the number of bathrooms to be a float number since it is possible to have 1\u00bd bedrooms.

Another important step in data cleaning is to give the right default value when the corresponding data is missing. For example, no avail_date is likely to mean the unit is not available and we should set the default value to be `date.max`. In another case, some apartment may not have a bedroom number because it is a studio, so it makes more sense to set the default bedroom number to be 0.

### Initiate the Crawling
`scrapy crawl` can be used to initiate the crawling, and you can also add an output filename and some arguments to the spider in the following form
```
scrapy crawl apartments -o outputFile.json -a input=input_data
```

## Data Processing
After getting the data, the next step is to get the list of apartments that fit our requirements.
Recall that one important factor I care is whether the apartment is close to some shuttle station. Therefore, we will first find a way to get the distance data and then build a command line tool that can be used to filter the data we need.

### Pre-processing Using Google Map APIs
The `googlemaps` api is readily available for us to get the direction data. In order to find the closest station to some apartment, we can simply iterate through all available stations and initiate an API call for the walking time from the apartment to each station to find the closest one. However, since we can have hundreds of apartment candidate and roughly 30 stations to consider. This is definitely too much. In fact, Google Direction API only supports 2500 free api calls a day. I don't want to spend money for any unnecessary API calls.

I mainly use the following strategies to save API calls.
1. Caching the best stations. For each address, I will save the best station offline, so that the next time I run the command line tool, it will not make API calls at all.
2. Get rid of unnecessary API calls for unlikely stations. Although the accurate walking time should only be calculated using directions instead of geometric distances, the use case here is very special. In fact, I don't believe there are more than three stations that we need to care about for finding the best station. Therefore, instead of making 30 API calls for all stations, I can confine my search to only three closest stations in using geo codes. The extra cost is that we now need to make some API calls to find the geo code for each apartment. But since we can also cache the result for geo code, it won't cause any trouble.
3. A good exception handling. Once I've reached the free API call limit number, the API call will raise an exception and to avoid previous work being wasted, I will save the data to cache immediately once the exception is caught.

Here is some code excerpt for `distanceCalculator`:
```python
class DistanceCalculator:
    gmaps = googlemaps.Client(key=google_api_key)

    cache = {}
    cache_smallest_station = {}
    cache_geo_code = {}
    direction_api_calls = 0
    geocode_api_calls = 0
    def __init__(self):
      ...

    def save_cache(self):
      ...

    def calculate_distance_and_duration(self, start, dest, mode, departure_hour=9):
        departure_time = datetime(
            self.departure_time.year,
            self.departure_time.month,
            self.departure_time.day,
            departure_hour)
        try:
            self.direction_api_calls = self.direction_api_calls + 1
            if (self.direction_api_calls % 30 == 0):
                print('save cache...')
                self.save_cache()
            print("number of direction api cals: {}".format(self.direction_api_calls))
            directions_result = self.gmaps.directions(
                start, dest, mode, departure_time=departure_time)
        except:
            print('google map direction api error')
            self.save_cache()
            raise
        if (not directions_result) or (not directions_result[0]['legs']):
            return math.inf, math.inf
        else:
            return self.second_to_minute(directions_result[0]['legs'][0]['duration']['value']), \
                self.meter_to_mile(directions_result[0]['legs'][0]['distance']['value'])

    def get_geocode(self, address):
        if address in self.cache_geo_code :
            return self.cache_geo_code [address]
        else:
            try:
                self.geocode_api_calls = self.geocode_api_calls + 1
                print("number of geocode api cals: {}".format(self.geocode_api_calls))
                geocode = self.gmaps.geocode(address)
                if geocode and 'geometry' in geocode[0] and  'location' in geocode[0]['geometry']:
                    geocode = (geocode[0]['geometry']['location']['lat'], geocode[0]['geometry']['location']['lng'])
                self.cache_geo_code[address] = geocode
                return geocode
            except:
                print('google map api error')
                self.save_cache()
                raise

    def find_approx_station_with_shortest_time(self, start, max_station_considered=3):
        if start in self.cache_smallest_station:
            return {'time_to_shuttle': self.cache_smallest_station[start]['min_duration'], \
                'best_station': self.cache_smallest_station[start]['best_station']}
        start_geocode = self.get_geocode(start)
        print('Calcuating approximate shortest path for {}...'.format(start))
        print(start_geocode)
        print(self.stations_geocode[0])
        stations_considered = []
        for index, station_geocode in enumerate(self.stations_geocode):
            dist = (start_geocode[0] - station_geocode[0])**2 + (start_geocode[1] - station_geocode[1])**2
            if len(stations_considered) < max_station_considered:
                stations_considered.append((dist, self.stations[index]))
            else:
                stations_considered.sort()
                stations_considered[-1] = (dist, self.stations[index])

        stations = [x[1] for x in stations_considered]
        return self.find_station_with_shortest_time(start, stations)
```

### The Apartment Finding Command Line Tool
After getting all the data we need, writing a command line tool is almost trivial. Here is a list of functions that I essentially support:
```
usage: rank_apts [--bed=min_bedroom] [--bath=min_bathroom] [--walk=min_walking_time] [--avail_before=date] [--avail_after=date] [--price=max_price] [--dist=distance][--topk=k] [-w] [-a] [--clean] output_file
```
While most options are very obvious, some specials ones are
* -w: only consider apartments with in-room washer/dryer
* -a: only consider apartments with A/C (Yes, a lot of apartments in the bay area does not have an A/C...)
* --clean: re-crawl the data for ef

The final output is a `.csv` file containing all apartments that are of my interest sorted by price in ascending order.

## Conclusions
Finding an apartment is definitely not an easy job, but at least I got some ease with a search that is exhaustive enough so that I would be confident to say that I get the best I can find. The project is mostly tailored for my personal interests, but hopefully it can inspire you for solving some problems you have in your life as well. :)
