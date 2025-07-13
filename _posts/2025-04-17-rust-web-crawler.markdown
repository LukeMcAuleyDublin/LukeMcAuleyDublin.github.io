---
layout: post
title: "Building a web crawler with Rust: Part 1"
date: 2025-04-17 19:21:54 +0100
---
A while ago I became curious about how search engines work. After a cursory search, I learned that the initial phase of building a search engine involves creating a [web crawler][web-crawler-url] that is responsible for grabbing as many links as possible, as the basis for what the search engine will actually *search*.

This intrigued me, and I wanted to see if I could build my own. It had also been a while since I had used my favourite programming language [Rust][rust-url], so I took it as an opportunity to sharpen my Rust skills.
There a number of constaints that come with developing a web crawler:
- Web crawlers are IO bound; meaning the program will only be as fast as the latency of it's non-CPU work. This includes database writes and HTTP requests.
Keeping these constraints in mind, I mapped out the following requirements for a web crawler.
- The program should be asynchronous. It should not wait for one link to be crawled before proceeding to another one.
- It should save any link it comes across to a database, which an indexer will read from at a later date.
- It should proceed with crawling while performing IO actions like making HTTP requests or writing to a database.
First things first, I'll need a development database set up locally to make sure my program is saving what I expect. I will use a docker container and PostgreSQL for this, which makes it easy to tear down and boot up the database whenever I like without needing to install `pgsql` on my machine.
Here is the `docker-compose.yml` file that I run with `docker compose up -d` which gives me all I need in a development database.
{% highlight yml %}
services:
  postgres:
    image: postgres:latest
    container_name: postgres-db
    environment:
      POSTGRES_DB: crawler
      POSTGRES_USER: username
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
volumes:
  postgres_data:
{% endhighlight %}

## Architecture and Design

With the database ready, I designed the crawler around several key components that work together to efficiently crawl the web. The architecture follows a modular approach where each component has a specific responsibility.

The main entry point is straightforward - I use the `clap` crate to handle command line arguments, allowing users to specify a starting URL, whether to restrict crawling to a single domain, and a timeout value. The `tokio` runtime handles all the asynchronous operations that make this crawler performant.

{% highlight rust %}
#[derive(Parser, Debug)]
#[command(version, about, long_about = None)]
struct Args {
    #[arg(short, long)]
    url: String,

    #[arg(short, long, action = ArgAction::SetTrue)]
    restrict_domain: bool,

    #[arg(short, long, default_value = "30")]
    seconds: u64,
}
{% endhighlight %}

The core of the application is built around three main components:

**AppServices**: This struct manages shared resources like the HTTP client and database connection pool. By using `Arc` (Atomically Reference Counted) smart pointers, multiple parts of the application can safely share these resources across async tasks without expensive cloning.

**LinkCollection**: This is where the crawling logic lives. It maintains two vectors - one for visited links and another for unvisited ones. The crawler pops links from the unvisited queue, processes them, and adds any newly discovered links back to the queue. This breadth-first approach ensures we don't get stuck in deep link chains.

**Link**: Each individual link is represented as a struct that can fetch its own HTML content, extract links from that content, and save itself to the database. This encapsulation makes the code more maintainable and testable.

## Link Processing and Extraction

The heart of the crawler is the link extraction logic. When processing a link, the crawler first fetches the HTML content using the shared HTTP client. I chose to use the `scraper` crate for HTML parsing because it provides a jQuery-like interface that makes selecting elements intuitive.

{% highlight rust %}
let document = Html::parse_document(&html_content);
let link_selector = Selector::parse("a[href]").unwrap();
let mut links: Vec<String> = Vec::new();
let base_url = Url::parse(&self.address)?;

for element in document.select(&link_selector) {
    if let Some(href) = element.value().attr("href") {
        match base_url.join(href) {
            Ok(absolute_url) => {
                let url_string = absolute_url.to_string();
                if !links.contains(&url_string)
                    && !visited_links.iter().any(|link| link.address == url_string)
                {
                    if self.valid_https(&url_string) {
                        links.push(url_string);
                    }
                }
            }
            Err(_) =>
        }
    }
}
{% endhighlight %}

The extraction process handles several edge cases that are common in web crawling. Relative URLs are converted to absolute URLs using the current page as the base. The crawler also filters out non-HTTPS links and duplicates to keep the crawl focused and efficient.

## Database Integration and Persistence

For persistence, I used `sqlx` which provides compile-time checked SQL queries and excellent async support. The database connection is established once and shared across the application using connection pooling, which is crucial for performance when making frequent database writes.

{% highlight rust %}
pub async fn save(&self, db_conn: &PgPool) -> Result<(), sqlx::Error> {
    match sqlx::query("INSERT INTO urls (address) VALUES ($1)")
        .bind(&self.address)
        .execute(db_conn)
        .await
    {
        Ok(_) =>
        Err(e) =>
    }
    Ok(())
}
{% endhighlight %}

The database schema is simple but effective - just a table with URL addresses that an indexer can later process. This separation of concerns allows the crawler to focus on discovery while other components handle indexing and search functionality.

## Logging and Observability

Throughout the development process, I found that having good logging was essential for debugging and monitoring the crawler's progress. I built a custom logger that uses the `crossterm` crate to provide colored output, making it easy to distinguish between different types of log messages.

The logger supports different prefixes for different components (like "crawler.link_collection" or "crawler.link"), which helps trace the flow of execution. Each log message can have its own color, making it easy to spot errors or successful operations at a glance.

This first implementation provides a solid foundation for a web crawler. In the next part, I'll explore how to make it truly asynchronous by processing multiple links concurrently, which will dramatically improve performance for large-scale crawling operations.

If you don't feel like waiting, [here is the source code][project-url] (although I do plan on iteratively improving it!)

[web-crawler-url]: https://moz.com/beginners-guide-to-seo/how-search-engines-operate
[rust-url]: https://www.rust-lang.org/
[project-url]: https://github.com/LukeMcAuleyDublin/crawler-rs
