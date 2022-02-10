# Scraping of Lithuanian job search websites 

On a EXTRACT SIDE
The idea is to scrape most popular Lithuanian job search websites using Scrapy / Selenium or Beutiful Soup:
www.cvbankas.lt
www.cvmarket.lt
www.cv.online.lt

## CV Bankas scraping
Page has a quite simple HTML structure (no JavaScript workarounds), sp I have decided to use Scrapy as the fastest asynchronous engine.
I have imported Scrapy and creted a spider class with two methods:
- main, parse divs and extract list of job ads
~~~
    def parse(self, response):
        last_page_num = int(response.css(".pages_ul_inner li a::text").extract()[-1])
        for page_num, _ in enumerate(range(last_page_num)):
            next_page_url = f'https://www.cvbankas.lt/?page={page_num}'
            all_divs = response.css('a.list_a.can_visited.list_a_has_logo')
            for vacancy in all_divs:
                link = vacancy.css('.list_a_has_logo').attrib['href']
                yield response.follow(link, callback=self.parse_details)
            yield response.follow(next_page_url, callback=self.parse)
~~~

- parse details method to parse details of the job add and to generator for further transformation
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
* Description is scrapeed from one div with no distinct selector, needs to be handled. At this stage, I just asign zero value to salaries field where spider did not find abny value:
~~~
if not response.css('.salary_amount::text').extract():
            salary = ['0']
        else:
            salary = response.css('.salary_amount::text').extract()[0]
~~~

## CV Market scraping
Page has a quite simple HTML structure (no JavaScript workarounds), very similar to cvbankas.lt. I have a strong suspition that it was developed by the same developper.
I have imported Scrapy and creted a spider class with two methods:
- main, parse divs and extract list of job ads
