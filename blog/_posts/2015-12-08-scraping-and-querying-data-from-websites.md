---
layout: blog-post
title:  "Scraping and querying data from websites"
date:   2015-12-08 00:00:00
categories: python scraping mongodb scrapy
image-base: /assets/images/posts/2015-12-08-scraping-and-querying-data-from-websites
---

The 21st century has brought with it many technological advances, such as the mass surveillance of swathes of the world's population and light bulbs that can be controlled via a phone. What it still hasn't got around to though - possibly due to more pressing matters like world peace - is allowing data that is displayed on websites to be easily queried; instead we find ourselves at the mercy of website owners and their (hopefully tabulated) data embedded in pages of convoluted HTML.

In the interim a solution to this problem can be achieved with the help of a web scraping framework ([scrapy](http://scrapy.org/)), database ([MongoDB](https://www.mongodb.org/)) and a cup of tea ([Miles](http://www.djmiles.co.uk/)) - which we'll see in action scraping, storing and querying popular baby names from the website [BabyNameWizard](http://www.babynamewizard.com/).


The baby name scraper that we're about to create is [hosted on GitHub](https://github.com/SketchingDev/Babyname-Scraper).

![Illustration of website being scraped then searched]({{ page.image-base }}/process.png)

## Scraping data from HTML

Before we can begin we must install Scrapy and its dependencies by following the [installation guide](http://doc.scrapy.org/en/latest/intro/install.html). With that out the way and having had a quick peruse over [Scrapy at a glance](http://doc.scrapy.org/en/latest/intro/overview.html) we can create a default Scrapy project with the `startproject` command:

```console
$ scrapy startproject babynamescraper
$ cd babynamescraper
```

We then define the data that we're going to extract as items in our new project under './babynamescraper/items.py'

```python
from scrapy.item import Item, Field

class NameList(Item):
  country = Field()
  region = Field()
  names = Field()
  gender = Field()
```

Once that's done then create the following spider under the directory './babynamescraper/spiders', which will populate a NameList item for each of the lists of baby names. It does this by:

1. Scrapy requests the URL in `start_url` and passes the response to the `parse` method
2. `parse` makes a request for each link contained in the response that points to a page containing a list of baby names
3. `parse_item` receives each response and populates an item with the list of baby names, country and region

```python
import scrapy
from scrapy.spiders import Spider
from scrapy.selector import Selector
from babynamescraper.items import NameList

class WizardSpider(Spider):
  name = "Wizard"
  start_urls = [
    "http://www.babynamewizard.com/international-names-lists-popular-names-from-around-the-world"
  ]

  def parse(self, response):
    item_urls = response.xpath("//a[text()='Boys' or text()='Girls']/@href").extract()

    for item_url in item_urls:
      yield scrapy.Request(item_url, self.parse_item)

  def parse_item(self, response):
    title = response.xpath(".//*[@id='page-title']/text()")
    gender = title.re_first(r'(Boy|Girl)')

    item = NameList()
    item['gender'] = 'male' if gender == 'Boy' else 'female'
    item['country'] = title.re_first(r'in ([A-Z a-z]+)(?: \([A-Z a-z]+\))?').strip()
    item['region'] = title.re_first(r'in [A-Z a-z]+(?:\(([A-Z a-z]+)\))?').strip()

    item['names'] = response.xpath(".//*[@class='item-list']/ol/li//text()").extract()

    return item
```

The methods in the spider can be tested to ensure that they're working as expected by using the [parse command](http://doc.scrapy.org/en/latest/topics/debug.html#parse-command). *Contracts can also be defined in the spider, which make testing much easier - you can see them in action in the [project on GitHub](https://github.com/SketchingDev/Babyname-Scraper)*.

```bash
$ scrapy parse "http://www.babynamewizard.com/international-names-lists-popular-names-from-around-the-world" --spider=Wizard -c parse
```

Or to test just the `parse_item` method:

```bash
$ scrapy parse "http://www.babynamewizard.com/name-list/scottish-boys-names-most-popular-names-for-boys-in-scotland" --spider=Wizard -c parse_item
```


## Storing scraped items in a database

We're now going to want to store all the items that our spider is scraping into a database. I found that the easiest database to integrate into the scraping process is MongoDB, which has an [installation guide](https://docs.mongodb.org/manual/installation) for most popular operating systems.

Once installed and accessible from the same computer as our spider, we need to configure an [item pipeline component](http://doc.scrapy.org/en/latest/topics/item-pipeline.html) in our spider's project which will take each parsed item and insert it into our instance of MongoDB. Fortunately for us there's already a marvellous pipeline for this called [scrapy-mongodb](http://sebdah.github.io/scrapy-mongodb/).

The easiest way to install scrapy-mongodb is to use [pip](https://pypi.python.org/pypi/pip/), like so:

```bash
$ pip install scrapy-mongodb
```

After pip has installed scrapy-mongodb we can add it to our project's pipelines in the './babynamescraper/settings.py' file:

```python
ITEM_PIPELINES = [
  'scrapy_mongodb.MongoDBPipeline',
]

MONGODB_URI = 'mongodb://localhost:27017'
MONGODB_DATABASE = 'scrapy'
MONGODB_COLLECTION = 'baby_names'
```

*You can also create your own pipeline items for manipulating each item etc.*

## Querying scraped items

Now let's run our spider! If all goes well the spider will scrape all the popular baby names per country from the target website and store the items in MongoDB as part of the pipeline

```console
$ scrapy crawl Wizard
```

After the spider has finished we can connect to MongoDB and query the data to our heart's content!

```console
$ mongo

> use scrapy
> db.baby_names.find({},{"_id":0, "country":1, "gender":1}).pretty()
{ "gender" : "female", "country" : "Mexico" }
{ "gender" : "male", "country" : "Mexico" }
{ "gender" : "male", "country" : "Paraguay" }
{ "gender" : "female", "country" : "Paraguay" }
{ "gender" : "female", "country" : "Chile" }
{ "gender" : "male", "country" : "Chile" }
{ "gender" : "male", "country" : "Bolivia" }
{ "gender" : "female", "country" : "Bolivia" }
{ "gender" : "female", "country" : "Australia" }
{ "gender" : "male", "country" : "Australia" }
{ "gender" : "female", "country" : "Argentina" }
{ "gender" : "male", "country" : "Argentina" }
{ "gender" : "male", "country" : "Taiwan" }
{ "gender" : "male", "country" : "South Korea" }
{ "gender" : "female", "country" : "South Korea" }
{ "gender" : "male", "country" : "Israel" }
{ "gender" : "female", "country" : "Australia" }
{ "gender" : "male", "country" : "Australia" }
{ "gender" : "female", "country" : "Japan" }
{ "gender" : "male", "country" : "Japan" }
```
