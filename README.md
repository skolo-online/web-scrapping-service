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


Let us run the spider to scrap just the first page and inspect the results:
```
scrapy crawl eskom_tenders -o output.json -a page=1
```
We will see nothing, the problem with most website scrapping jobs in JS rendering. This means, by the time the scrapper tries to get the HTML from the link, the website has not been rendered yet. The HTML only creates a <div> that is later populated with JS, this is what happens with most JS web frameworks. 

We need to wait for the JS to render, then read the HTML content to see the appropriate elements that we need to scrap, then scrap that. 


## JS Web Scrapping
For this we are going to use [Scrapy Playwright](https://github.com/scrapy-plugins/scrapy-playwright). 

first install scrapy-playwright
```sh
pip install scrapy-playwright
```
Then
```sh
playwright install
```
And then install the browsers
```sh
playwright install firefox chromium
```

Update the settings.py file to work with scrapy-playwright
```python
# settings.py

# Scrapy settings for tenders project
BOT_NAME = 'tenders'

SPIDER_MODULES = ['tenders.spiders']
NEWSPIDER_MODULE = 'tenders.spiders'

# Obey robots.txt rules
ROBOTSTXT_OBEY = True

# Configure maximum concurrent requests performed by Scrapy (default: 16)
CONCURRENT_REQUESTS = 16

# Configure a delay for requests for the same website (default: 0)
DOWNLOAD_DELAY = 0

# Override the default request headers:
DEFAULT_REQUEST_HEADERS = {
    'User-Agent': (
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
        "(KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
    ),
}


DOWNLOAD_HANDLERS = {
    "http": "scrapy_playwright.handler.ScrapyPlaywrightDownloadHandler",
    "https": "scrapy_playwright.handler.ScrapyPlaywrightDownloadHandler",
}

# Enable or disable spider middlewares
# SPIDER_MIDDLEWARES = {
#     'scrapy_playwright.middleware.PlaywrightMiddleware': 800,
# }

# Configure item pipelines (if any)
# ITEM_PIPELINES = {
#     'tenders.pipelines.PostgresPipeline': 300,
# }

# # PostgreSQL Database Connection String
# POSTGRES_DSN = "postgresql://username:password@host:port/database"

# Enable Playwright
TWISTED_REACTOR = 'twisted.internet.asyncioreactor.AsyncioSelectorReactor'

# Playwright settings
PLAYWRIGHT_BROWSER_TYPE = "chromium"  

# Optional: Set Playwright launch options
PLAYWRIGHT_LAUNCH_OPTIONS = {
    "headless": True,
    "args": [
        "--no-sandbox",
        "--disable-setuid-sandbox",
        "--disable-dev-shm-usage",
        "--disable-gpu",
        "--window-size=1920,1080",
    ],
}

# Feed Export Settings
FEED_EXPORT_ENCODING = 'utf-8'

# Timeout settings
DOWNLOAD_TIMEOUT = 30  # Timeout for requests in seconds

# Logging configuration
LOG_LEVEL = "INFO"

# Screenshot settings (optional)
PLAYWRIGHT_SCREENSHOT_PATH = 'screenshots/'  # Directory to save screenshots
```

then update the eskom_spider.py file to the following:
```python
# tenders/spiders/eskom_tenders.py

import os
import scrapy
from scrapy_playwright.page import PageMethod
from tenders.items import TenderItem

class EskomTenderSpider(scrapy.Spider):
    name = "eskom_tenders"

    def __init__(self, *args, **kwargs):
        """
        Initializes the spider with dynamic pagination limits and starting page.
        """
        super().__init__(*args, **kwargs)
        self.start_page = int(getattr(self, "page", 1))  # Start page (default: 1)
        self.current_page = self.start_page
        self.max_pages = self.start_page if "page" in kwargs else 5  # Scrape up to 5 pages if not specified

    def start_requests(self):
        """
        Initiates requests for the starting page using Playwright.
        """
        url = f"https://tenderbulletin.eskom.co.za/?pageSize=20&pageNumber={self.current_page}"
        
        # Define directories for screenshots and HTML files
        screenshot_dir = os.path.join(os.getcwd(), "screenshots")
        html_dir = os.path.join(os.getcwd(), "html_pages")
        os.makedirs(screenshot_dir, exist_ok=True)
        os.makedirs(html_dir, exist_ok=True)

        yield scrapy.Request(
            url=url,
            callback=self.parse,
            meta={
                "playwright": True,
                "playwright_page_methods": [
                    # Wait for 20 seconds (20000 milliseconds)
                    PageMethod("wait_for_timeout", 20000),
                    # Take a screenshot of the page
                    PageMethod("screenshot", path=os.path.join(screenshot_dir, f"page_{self.current_page}.png")),
                ],
                "playwright_include_page": False,  # We don't need to keep the page open
            },
        )

    def parse(self, response):
        """
        Parses the main page to save the entire HTML content and handles pagination.
        """
        # Define directory for saving HTML files
        html_dir = os.path.join(os.getcwd(), "html_pages")
        os.makedirs(html_dir, exist_ok=True)
        
        # Create a filename based on the current page number
        filename = f"page_{self.current_page}.html.txt"
        filepath = os.path.join(html_dir, filename)

        # Save the HTML content to the file
        with open(filepath, 'wb') as f:
            f.write(response.body)
        self.logger.info(f"Saved HTML page to {filepath}")

        # Yield an item with the HTML content for output.json
        yield {
            'page_number': self.current_page,
            'html_content': response.text,  # Alternatively, use response.body.decode('utf-8')
        }

        # Handle pagination
        if self.current_page < self.max_pages:
            self.current_page += 1
            next_page_url = f"https://tenderbulletin.eskom.co.za/?pageSize=20&pageNumber={self.current_page}"
            yield scrapy.Request(
                url=next_page_url,
                callback=self.parse,
                meta={
                    "playwright": True,
                    "playwright_page_methods": [
                        # Wait for 20 seconds
                        PageMethod("wait_for_timeout", 20000),
                        # Take a screenshot of the next page
                        PageMethod("screenshot", path=os.path.join("screenshots", f"page_{self.current_page}.png")),
                    ],
                    "playwright_include_page": False,
                },
            )
        
        # If you still want to process "Read More" links and save their HTML:
        # Uncomment the following block and adjust as needed

        # for tender in response.css("div.tender-item"):
        #     read_more_link = tender.css("a.read-more::attr(href)").get()
        #     if read_more_link:
        #         full_url = response.urljoin(read_more_link)
        #         yield scrapy.Request(
        #             url=full_url,
        #             callback=self.parse_tender_details,
        #             meta={
        #                 "playwright": True,
        #                 "playwright_page_methods": [
        #                     # Wait for 20 seconds
        #                     PageMethod("wait_for_timeout", 20000),
        #                     # Take a screenshot of the tender details page
        #                     PageMethod("screenshot", path=os.path.join("screenshots", f"tender_{self.current_page}.png")),
        #                 ],
        #                 "playwright_include_page": False,
        #             },
        #         )

    def parse_tender_details(self, response):
        """
        Parses the detailed tender page to save the entire HTML content.
        """
        # Define directory for saving HTML files
        html_dir = os.path.join(os.getcwd(), "html_pages")
        os.makedirs(html_dir, exist_ok=True)
        
        # Extract tender number for naming the file, if available
        tender_number = response.css("div.tender-number::text").get(default="unknown").strip()
        filename = f"tender_{tender_number}.html.txt"
        filepath = os.path.join(html_dir, filename)

        # Save the HTML content to the file
        with open(filepath, 'wb') as f:
            f.write(response.body)
        self.logger.info(f"Saved tender details HTML to {filepath}")

        # Yield an item with the HTML content for output.json
        yield {
            'tender_number': tender_number,
            'html_content': response.text,  # Alternatively, use response.body.decode('utf-8')
        }

```
We are doing the following:
1. Waiting 20 seconds for the page to load, you can reduce this --- just check how long it takes the page to load
2. Taking a screenshot of the page to see what is there
3. Saving the actual HTML that was rendered -- so we can see the actual CSS elements we need to use

Run the command:
```sh
scrapy crawl eskom_tenders -o output.json -a page=1
```
This command creates a file called output.json and saves the HTML there also. You can now inspect the HTML that was rendered.

After inspecting the HTML. I have (pretty much fed it to the AI :) and adjusted my code to the following:
```python
import os
import scrapy
from scrapy_playwright.page import PageMethod
from tenders.items import TenderItem


class EskomTenderSpider(scrapy.Spider):
    name = "eskom_tenders"

    def __init__(self, *args, **kwargs):
        """
        Initializes the spider with dynamic pagination limits and starting page.
        """
        super().__init__(*args, **kwargs)
        self.start_page = int(getattr(self, "page", 1))  # Start page (default: 1)
        self.current_page = self.start_page
        self.max_pages = self.start_page if "page" in kwargs else 5  # Scrape up to 5 pages if not specified

    def start_requests(self):
        """
        Initiates requests for the starting page using Playwright.
        """
        url = f"https://tenderbulletin.eskom.co.za/?pageSize=20&pageNumber={self.current_page}"

        yield scrapy.Request(
            url=url,
            callback=self.parse,
            meta={
                "playwright": True,
                "playwright_page_methods": [
                    PageMethod("wait_for_timeout", 20000),
                ],
                "playwright_include_page": False,
            },
        )

    def parse(self, response):
        """
        Parses the main page and extracts tender information into TenderItem objects.
        """
        for tender in response.css("li.relative"):
            item = TenderItem()

            # Extracting fields based on CSS selectors
            item["title"] = tender.css("h3::text").get().strip()
            item["tender_number"] = tender.css("h3::text").get().strip()
            item["short_description"] = tender.css("p.text-sm::text").get().strip()
            item["location"] = tender.css("dd span.font-medium::text").get()
            item["closing_date"] = tender.css("dd span.font-medium").xpath("following-sibling::text()").get()
            item["download_link"] = response.urljoin(
                tender.css("a[href*='DownloadAll']::attr(href)").get()
            )
            item["contract_type"] = tender.css("dt:contains('Contract Type') + dd::text").get()
            item["target_audience"] = tender.css("dt:contains('Target Audience') + dd::text").get()
            item["tender_id"] = tender.css("a::attr(href)").re_first(r"/tender/(\d+)")
            item["company"] = tender.css("dt::text").re_first(r"[a-zA-Z.-]+")
            yield item

        # Handle pagination
        if self.current_page < self.max_pages:
            self.current_page += 1
            next_page_url = f"https://tenderbulletin.eskom.co.za/?pageSize=20&pageNumber={self.current_page}"
            yield scrapy.Request(
                url=next_page_url,
                callback=self.parse,
                meta={
                    "playwright": True,
                    "playwright_page_methods": [
                        PageMethod("wait_for_timeout", 20000),
                    ],
                    "playwright_include_page": False,
                },
            )

```

This code above you can run again
```sh
scrapy crawl eskom_tenders -o output_1.json -a page=1
```

and you should get the JSON of the actual tenders in a list. You can add pipelines to process the data further, but we will leave it here for now.

## Bonus Scrapper
If you want a scrapper to just return the HTML for further processing, so you can actually build with the actual scrapper using the correct css codes, this is the minimal code below:

This will work on any URL and takes in  a URL as an input. Create a new file in `tenders/spiders/fetch_html_spider.py`
```python
import scrapy
from scrapy_playwright.page import PageMethod

class FetchHTMLSpider(scrapy.Spider):
    name = "fetch_html_spider"
    allowed_domains = [] 

    def __init__(self, url=None, *args, **kwargs):
        """
        Initializes the spider with a URL provided as an argument.

        Usage:
            scrapy crawl fetch_html_spider -a url="https://example.com" -o output_html.json
        """
        super().__init__(*args, **kwargs)
        if not url:
            raise ValueError("A 'url' argument must be provided. Usage: -a url='https://example.com'")
        self.start_urls = [url]

    def start_requests(self):
        """
        Initiates a request to the provided URL using Playwright and waits for 20 seconds.
        """
        for url in self.start_urls:
            yield scrapy.Request(
                url=url,
                callback=self.parse,
                meta={
                    "playwright": True,
                    "playwright_page_methods": [
                        # Wait for 20 seconds (12000 milliseconds)
                        PageMethod("wait_for_timeout", 12000),
                    ],
                    "playwright_include_page": False,
                },
            )

    def parse(self, response):
        """
        Parses the response and yields the HTML content.
        """
        yield {
            'url': response.url,
            'html_content': response.text, 
        }

```

Run the code witht the command:
```py
scrapy crawl fetch_html_spider -a url="http://www.publicworks.gov.za/tenders.html#gsc.tab=0" -o output_html.json
```

