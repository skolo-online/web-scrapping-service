# web-scrapping-service
Follow along coding with us on Skolo Online, the vide is available on YouTube here
[Web Scrapping Video]()

## Get Started
Get started by creating a virtual environment and installing scrapy 

```sh
virtualenv scrapenv
```

```sh
source scrapenv/bin/activate
pip install scrapy
```

Let us start our scrapy project, we will call it tenders, because we are going to use this service on a daily basis to scrap the internet for tenders for our tender project:
```sh
scrapy startproject tenders
```

A new folder called tendes will be created, the tenders directory will look like so
```
tenders/
    scrapy.cfg
    tenders/
        __init__.py
        items.py
        middlewares.py
        pipelines.py
        settings.py
        spiders/
```

## Data Structure
We are going to be using this tender scrapper on multiple websites, but we will start with the Eskom Website and looking at their tender page here:
```
https://tenderbulletin.eskom.co.za/?pageSize=20&pageNumber=1
```
We decided to go with the following structure . . . we can update it as we go along: save it inside the `tenders/items.py` file

```python
import scrapy

class TenderItem(scrapy.Item):
    """
    Defines the fields for the tender item.
    """
    title = scrapy.Field()
    tender_number = scrapy.Field()
    short_description = scrapy.Field()
    location = scrapy.Field()
    closing_date = scrapy.Field()
    download_link = scrapy.Field()
    contract_type = scrapy.Field()
    target_audience = scrapy.Field()
    tender_id = scrapy.Field()
    company = scrapy.Field()

```
If you look at a single tender item from the page, these are the fields available:
![Tender Item To Be Scraped](https://bertha.ams3.digitaloceanspaces.com/example_tender.png)

Then there is a button to see more information, which has pretty much the same info but with extra items and there is a button to download tender documents.

For now, we will just save this information to a database, including the download link for the tender documents. We will deal with them later. We are just creating a service to save tender information to a database.

## Web Crawlers 
Create a file called: `tenders/spiders/eskom_spider.py` and save the following:

```python
import scrapy
from tenders.items import TenderItem

class EskomTenderSpider(scrapy.Spider):
    name = 'eskom_tenders'

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.start_page = int(getattr(self, 'page', 1))  # Start page (default to 1)
        self.current_page = self.start_page
        self.max_pages = self.start_page if 'page' in kwargs else 5  # Scrape only the page provided, or default to 5 pages.

    def start_requests(self):
        # Start from the specified page or page 1
        url = f'https://tenderbulletin.eskom.co.za/?pageSize=20&pageNumber={self.current_page}'
        yield scrapy.Request(url, self.parse)

    def parse(self, response):
        """
        Parses the main tender bulletin page to extract tender details and navigates to 'Read More' pages for additional information.
        """
        for tender in response.css('div.tender-item'):
            # Create an instance of TenderItem
            item = TenderItem()

            # Populate the fields
            item['title'] = tender.css('h2.tender-title::text').get()
            item['tender_number'] = tender.css('div.tender-number::text').get()
            item['short_description'] = tender.css('div.tender-description::text').get()
            item['location'] = tender.css('div.tender-location::text').get()
            item['closing_date'] = tender.css('div.tender-closing-date::text').get()

            # Extract the download link for tender documents
            download_link = tender.css('a.download-tender-documents::attr(href)').get()
            if download_link:
                item['download_link'] = response.urljoin(download_link)

            # Extract the 'Read More' link to navigate to the detailed tender page
            read_more_link = tender.css('a.read-more::attr(href)').get()
            if read_more_link:
                yield response.follow(
                    read_more_link,
                    self.parse_tender_details,
                    meta={'item': item}  # Pass the partially populated item to the next method
                )
            else:
                # Yield the item directly if there's no "Read More" link
                item['company'] = 'Eskom'
                yield item

        # Pagination: Stop after the specified or maximum number of pages
        if self.current_page < self.max_pages:
            self.current_page += 1
            next_page_url = f'https://tenderbulletin.eskom.co.za/?pageSize=20&pageNumber={self.current_page}'
            yield scrapy.Request(next_page_url, self.parse)

    def parse_tender_details(self, response):
        """
        Parses the detailed tender page to extract additional information such as contract type, target audience, and tender ID.
        """
        # Retrieve the partially populated item from the meta data
        item = response.meta['item']

        # Extract additional details from the detailed tender page
        item['contract_type'] = response.css('div.contract-type::text').get()
        item['target_audience'] = response.css('div.target-audience::text').get()
        item['tender_id'] = response.css('div.tender-id::text').get()

        # Set the company field (static for this example)
        item['company'] = 'Eskom'

        # Yield the fully populated item
        yield item
```

## Pipelines
After the data has been scrawled, it will be saved in a database - so create the following file to add it to a postgresl database:
`tenders/pipelines.py`

First install the binary:
```sh
pip install psycopg2-binary
```

```python
import psycopg2
from psycopg2 import sql
from scrapy.exceptions import DropItem

class PostgresPipeline:
    """
    Pipeline to handle the storage of scraped items into a PostgreSQL database.
    """

    def open_spider(self, spider):
        """
        Initializes database connection and creates table if it doesn't exist.
        """
        self.connection = psycopg2.connect(
            dsn="database_dsn_goes_here"
        )
        self.cursor = self.connection.cursor()
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS tenders (
                id SERIAL PRIMARY KEY,
                title TEXT,
                tender_number TEXT UNIQUE,
                short_description TEXT,
                location TEXT,
                closing_date DATE,
                download_link TEXT,
                contract_type TEXT,
                target_audience TEXT,
                tender_id TEXT,
                company TEXT
            );
        """)
        self.connection.commit()

    def close_spider(self, spider):
        """
        Closes the database connection when the spider is closed.
        """
        self.cursor.close()
        self.connection.close()

    def process_item(self, item, spider):
        """
        Processes each item and inserts it into the database if it doesn't already exist.
        """
        try:
            self.cursor.execute("""
                INSERT INTO tenders (
                    title, tender_number, short_description, location, closing_date,
                    download_link, contract_type, target_audience, tender_id, company
                ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON CONFLICT (tender_number) DO NOTHING;
            """, (
                item['title'],
                item['tender_number'],
                item['short_description'],
                item['location'],
                item['closing_date'],
                item['download_link'],
                item['contract_type'],
                item['target_audience'],
                item['tender_id'],
                item['company']
            ))
            self.connection.commit()
        except psycopg2.Error as e:
            spider.logger.error(f"Error saving item: {e}")
            raise DropItem(f"Error saving item: {e}")
        return item

```

## Finally 
Let us run the spider to scrap just the first page and inspect the results:
```
scrapy crawl eskom_tenders -o output.json -a page=1
```

Then let us save to the Database -> first add this to your settings file:
```python
ITEM_PIPELINES = {
   'tenders.pipelines.PostgresPipeline': 300,
}
DOWNLOAD_DELAY = 1
```

and run the command
```
scrapy crawl eskom_tenders
```
