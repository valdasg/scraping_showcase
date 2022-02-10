# Scraping of Lithuanian job search websites 

The idea is to scrape most popular Lithuanian job search websites using Scrapy / Selenium or Beutiful Soup:
www.cvbankas.lt
www.cvmarket.lt
www.cv.online.lt

## Extract part

### CV Bankas scraping
Page has a quite simple HTML structure (no JavaScript workarounds), sp I have decided to use Scrapy as the fastest asynchronous engine.
I have imported Scrapy and creted a spider class with two methods:
- main, parse divs and extract list of job ads and handles pagination:
~~~
    def parse(self, response):
        last_page_num = int(response.css(".pages_ul_inner li a::text").extract()[-1])
        for page_num, _ in enumerate(range(last_page_num)):
            next_page_url = f'https://www.cvbankas.lt/?page={page_num}'
            all_divs = response.css('a.list_a.can_visited.list_a_has_logo')
            for job in all_divs:
                link = job.css('.list_a_has_logo').attrib['href']
                yield response.follow(link, callback=self.parse_details)
            yield response.follow(next_page_url, callback=self.parse)
~~~

- parse details method to parse details of the job add and to generator for further transformation:
~~~
    def parse_details(self, response):
        position = response.css('#jobad_heading1::text').extract()
        company = response.css('#jobad_company_title::text').extract()
        payment_way = response.css('.salary_calculation::text').extract()
        city = response.css("span[itemprop*='addressLocality']::text").extract()[0]
        if not response.css('.salary_amount::text').extract():
            salary = ['0']
        else:
            salary = response.css('.salary_amount::text').extract()[0]
        description = response.css('.jobad_txt::text').extract()
        link = response.request.url

        yield {
            'position': position,
            'company': company,
            'city': city,
            'description': description,
            'salary': salary,
            'payment_way': payment_way,
            'link': link,
            'source': 'CV Bankas'
        }
~~~

Scrapeed data is raw. Many of the inconsistencies, like:
* Salaries indicated are before/after taxes/no salary indicated
* Description is scrapeed from one div with no distinct selector, needs to be handled. At this stage, I just asign zero value to salaries field where spider did not find any value:
~~~
if not response.css('.salary_amount::text').extract():
            salary = ['0']
        else:
            salary = response.css('.salary_amount::text').extract()[0]
~~~
* Description field contains raw job description data. Data is does not have specific structure, and vary from add to add, so the idea is to use this column for keyword search feature only.

### CV Market scraping
Page has a quite simple HTML structure (no JavaScript workarounds), very similar to cvbankas.lt. I have a strong suspition that it was developed by the same developper.
The first difference I see is the way of pagination is managed: final number of pages is not indicated on landing page, thus looping with range will not be possible. The other one is that data is nested in table instead of divs (as was in cvbankas.lt). These parts of code need to be adopted.

- Parse divs and extract list of job ads and deal with pagination. In cvmarket case pagination is based on following 'next' class selector, istead of loop as it was in cvbankas case:
~~~
    def parse(self, response):
        table_elements = response.css('#f_jobs_main table')
        for element in table_elements:
            row = element.css('.f_job_row2')
            print(row)
            for td in row.css('.column.d-none.d-md-table-cell.main-column'):

                link = td.css('.f_job_title.main_job_link.limited-lines').attrib['href']
                print(link)
                yield response.follow(link, callback=self.parse_details)

        next_page = response.css('.next').attrib['href']
        if next_page is not None:
            yield response.follow(next_page, callback=self.parse)
~~~
- Parse details of the job add and to generator for further transformation:
~~~
    def parse_details(self, response):
        position = response.css('#main-job-title::text').extract()
        company = response.css('.job-main-info a::text').extract()
        payment_way = response.css('.salary-type::text').extract()
        city = response.css(".jobdetails_value span.align-middle::text").extract()
        if not response.css('.salary b::text').extract():
            salary = ['0']
        else:
            salary = response.css('.salary b::text').extract()[0]
        description = response.css('.main-lang-block li::text').extract()
        link = response.request.url

        yield {
            'position': position,
            'company': company,
            'city': city,
            'description': description,
            'salary': salary,
            'payment_way': payment_way,
            'link': link,
            'source': 'CV Market'
        }
~~~
At this point I have noticed that cvmarket publishes over 30 000 job postings, lots of it reposting from other countries. Point to consider if running production version as this many rows takes time to scrape and probably nit much use for end user.
Next on www.cvonline.lt

### CV Online scraping
CV Online in comparisson to other two main job posting sites is heavily JavaScrript loaded, thus Scrapy will encounter issues. I will use Selenium in thi scase. Selenium is slower as it runs synchroniuosly and is driver based, however it can easiry handle popups and JS.
Also, CV Online give users much more control over job ad contents (ex. no particular structure, jpegs, pdfs), thus scraping of job description is very complex, I will skip it as empty value.

For CV online scraping I have created two classes: Parser (it is responsible for scraping of a site data and composing it to dictionary):
~~~
from selenium.webdriver.common.by import By
from selenium.webdriver.remote.webelement import WebElement
from selenium.common.exceptions import NoSuchElementException


class Parser:
    def __init__(self, boxes_section_element: WebElement):
        self.boxes_section_element = boxes_section_element
        self.ad_boxes = self.pull_all_adds()  # variable, that runs a method pull all ads

    def pull_all_adds(self):
        return self.boxes_section_element.find_elements(
            By.CLASS_NAME, 'vacancies-list__item'
        )

    def pull_attributes(self):

        jobs = []
        # Payment way
        payment_way = 'Neatskaičius mokesčių'

        # Source
        source = 'CVOnline'

        # Description
        description = ''

        for box in self.ad_boxes:
            # Position
            position = box.find_element(
                By.CLASS_NAME, 'vacancy-item__title'
            ).get_attribute('innerHTML').strip()

            # Company
            company = box.find_element(
                By.CLASS_NAME, 'vacancy-item__info-main a'
            ).get_attribute('innerHTML').strip()

            # Salary, if salary is not indicated, except 0
            try:
                salary = box.find_element(
                    By.CLASS_NAME, 'vacancy-item__salary-label'
                ).get_attribute('innerHTML')
            except NoSuchElementException:
                salary = '0'

            # City
            city = box.find_element(
                By.CLASS_NAME, 'vacancy-item__locations'
            ).get_attribute('innerText').replace('\xa0—\xa0', '').strip()

            # Link
            link = box.find_element(
                By.CLASS_NAME, 'vacancy-item__content a'
            ).get_attribute("href")

            # Compose output
            details = {
                'position': position,
                'company': company,
                'city': city.strip().split(',')[0],
                'description': description,
                'salary': salary,
                'payment_way': payment_way,
                'link': link,
                'source': 'CV Online'
            }

            jobs.append(details)

        return jobs

~~~

and Engine (responsible for all the site interaction methods):

~~~
import constants
from parser import Parser
import os
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import NoSuchElementException
import time


class Engine(webdriver.Chrome):
    def __init__(self, driver_path=r";C:\\SeleniumDrivers",
                 teardown=True):  # teardown is disabled
        self.driver_path = driver_path
        self.teardown = teardown
        os.environ['PATH'] += self.driver_path
        super(Engine, self).__init__()

        self.implicitly_wait(15)  # Give every method time to execute
        # self.maximize_window()  # Cleaner look for testing

    # Close windows upon end
    def __exit__(self, exc_type, exc_val, exc_tb):
        if self.teardown:
            self.quit()

    # Land first page
    def land_first_page(self):
        self.get(constants.URL)

    # Get rid of pop up banner
    def get_rid_of_banner(self):
        try:
            element = WebDriverWait(self, 10).until(
                EC.presence_of_element_located((By.CLASS_NAME, "modal-wrapper"))
            )
            banner_close = self.find_element(
                By.CLASS_NAME, "close-modal-button__content"
            )
            banner_close.click()
        except NoSuchElementException:
            pass

    # Get rid of cookies banner
    def get_rid_of_cookies(self):
        try:
            # Get rid of cookies
            element = WebDriverWait(self, 10).until(
                EC.presence_of_element_located((By.CLASS_NAME, "cookie-consent-container"))
            )
            cookie_close = self.find_element(
                By.ID, "rcc-confirm-button"
            )
            cookie_close.click()
        except NoSuchElementException:
            'No cookie banner'
        finally:
            pass

    # Select job ad search link to show all
    def search_ads(self):
        search_ads_button = self.find_element(
            By.CSS_SELECTOR, 'a[data-gtm-id="header-menu-link.job.ad.search"]'
        )
        search_ads_button.click()

    # Wait for the page to load and pull results
    def report_results(self=webdriver):
        try:
            element = WebDriverWait(self, 10).until(
                EC.presence_of_all_elements_located((By.CLASS_NAME, "vacancy-item__content"))
            )
        finally:
            ad_boxes = self.find_element(
                By.CLASS_NAME, 'search__content'
            )
            report = Parser(ad_boxes)
            return report.pull_attributes()

    # Go to next page
    def next_page(self):
        next_page = self.find_element(
            By.CSS_SELECTOR, 'button[aria-label="Next"]'
        )
        if next_page is not None:
            next_page.click()
        else:
            return
        time.sleep(1)
~~~
Spider is started from run.py, lopping through available pages:
~~~
from engine import Engine
from selenium.webdriver.common.by import By
import json

# Initialise class with 'with' to execute teardown actions as soon as program goes out of indentation.
# Teardown actions are specified in class as magic __exit__ method.
with Engine(teardown=True) as spider:  # set to true to enable teardown under __exit__ in class CvOnline
    spider.land_first_page()
    spider.get_rid_of_banner()
    spider.get_rid_of_cookies()
    spider.search_ads()
    spider.report_results()
    result = []

    for _ in range(267):
        spider.next_page()
        result += spider.report_results()
        data = json.dumps(result, indent=4)
    with open('jobs.json', 'w') as f:
        f.write(data)
~~~
Final result for CV Online counts to raw 5321 records:

#### Prints out:
![print result](https://github.com/valdasg/scraping_showcase/blob/master/cvonline.png?raw=true)

