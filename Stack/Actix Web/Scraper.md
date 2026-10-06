dependancy

```rust
scraper = "0.20"
```

  

```rust
use scraper::{Html, Selector};

async fn scrape_events(client: &reqwest::Client) -> Result<Vec<String>, reqwest::Error> {
    // 1. Fetch the page — same reqwest, .text() instead of .json()
    let html = client
        .get("https://example-sports-site.com/schedule")
        .header("User-Agent", "Mozilla/5.0")   // many sites reject requests with no UA
        .send()
        .await?
        .error_for_status()?
        .text()
        .await?;

    // 2. Parse the HTML into a queryable document
    let document = Html::parse_document(&html);

    // 3. Build selectors — literally CSS selector syntax
    let card_sel = Selector::parse("div.event-card").unwrap();
    let title_sel = Selector::parse("h3.event-title").unwrap();

    // 4. Query and extract
    let mut titles = Vec::new();
    for card in document.select(&card_sel) {
        if let Some(title_el) = card.select(&title_sel).next() {
            let text = title_el.text().collect::<String>();
            titles.push(text.trim().to_string());
        }
    }

    Ok(titles)
}
```