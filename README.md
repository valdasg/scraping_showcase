# Scraping of Lithuanian job search websites 

On a EXTRACT SIDE
The idea is to scrape most popular Lithuanian job search websites using Scrapy / Selenium or Beutiful Soup:
www.cvbankas.lt
www.cvmarket.lt
www.cv.online.lt

## CV Bankas scraping
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

## CV Market scraping
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

## CV Online scraping
CV Online in comparisson to other two main job posting sites is heavily JavaScrript loaded, thus Scrapy will encounter issues. I will use Selenium in thi scase. Selenium is slower as it runs synchroniuosly and is driver based, however it can easiry handle popups and JS.




