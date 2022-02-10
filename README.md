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

- parse details method to parse details of the job add


~~~



