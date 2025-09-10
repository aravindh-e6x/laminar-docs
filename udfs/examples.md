---
title: UDF Examples
sidebar_position: 4
---


Real-world examples of UDFs for common streaming data processing tasks.

## Data Parsing

### Parse User Agent

```rust
/*
[dependencies]
woothee = "0.13"
*/

#[udf]
fn parse_user_agent(ua: String) -> Option<String> {
    let parser = woothee::parser::Parser::new();
    parser.parse(&ua)
        .map(|result| result.name.to_string())
}
```

### Extract URL Components

```rust
/*
[dependencies]
url = "2.4"
*/

use url::Url;

#[udf]
fn extract_domain(url_str: String) -> Option<String> {
    Url::parse(&url_str)
        .ok()
        .and_then(|url| url.host_str().map(|s| s.to_string()))
}

#[udf]
fn extract_path(url_str: String) -> Option<String> {
    Url::parse(&url_str)
        .ok()
        .map(|url| url.path().to_string())
}

#[udf]
fn extract_query_param(url_str: String, param: String) -> Option<String> {
    Url::parse(&url_str)
        .ok()
        .and_then(|url| {
            url.query_pairs()
                .find(|(key, _)| key == &param)
                .map(|(_, value)| value.to_string())
        })
}
```

### Parse Log Lines

```rust
/*
[dependencies]
regex = "1.9"
*/

use regex::Regex;

#[udf]
fn parse_nginx_log(log: String) -> Option<String> {
    let re = Regex::new(
        r#"^(\S+) \S+ \S+ \[([\w:/]+\s[+\-]\d{4})\] "([^"]+)" (\d{3}) (\d+)"#
    ).ok()?;
    
    let caps = re.captures(&log)?;
    
    // Return JSON with parsed fields
    Some(format!(r#"{{
        "ip": "{}",
        "timestamp": "{}",
        "request": "{}",
        "status": {},
        "size": {}
    }}"#,
        caps.get(1)?.as_str(),
        caps.get(2)?.as_str(),
        caps.get(3)?.as_str(),
        caps.get(4)?.as_str(),
        caps.get(5)?.as_str()
    ))
}
```

## Data Transformation

### Hash PII Data

```rust
/*
[dependencies]
sha2 = "0.10"
*/

use sha2::{Sha256, Digest};

#[udf]
fn hash_email(email: String, salt: String) -> String {
    let mut hasher = Sha256::new();
    hasher.update(email.to_lowercase());
    hasher.update(salt);
    format!("{:x}", hasher.finalize())
}

#[udf]
fn mask_credit_card(cc: String) -> String {
    let digits: String = cc.chars()
        .filter(|c| c.is_ascii_digit())
        .collect();
    
    if digits.len() >= 8 {
        format!("****-****-****-{}", &digits[digits.len()-4..])
    } else {
        "****-****-****-****".to_string()
    }
}
```

### Normalize Data

```rust
#[udf]
fn normalize_country_code(country: String) -> String {
    match country.to_uppercase().as_str() {
        "US" | "USA" | "UNITED STATES" | "U.S." | "U.S.A." => "US",
        "UK" | "GB" | "GBR" | "UNITED KINGDOM" | "GREAT BRITAIN" => "GB",
        "DE" | "DEU" | "GERMANY" | "DEUTSCHLAND" => "DE",
        "FR" | "FRA" | "FRANCE" => "FR",
        other => other.to_string()
    }
}

#[udf]
fn normalize_currency(amount: f64, from_currency: String, to_currency: String) -> f64 {
    let rate = match (from_currency.as_str(), to_currency.as_str()) {
        ("USD", "EUR") => 0.85,
        ("EUR", "USD") => 1.18,
        ("GBP", "USD") => 1.27,
        ("USD", "GBP") => 0.79,
        _ => 1.0
    };
    amount * rate
}
```

## Enrichment

### Geolocation

```rust
/*
[dependencies]
maxminddb = "0.23"
*/

use maxminddb::geoip2;

static READER: Lazy<maxminddb::Reader<Vec<u8>>> = Lazy::new(|| {
    maxminddb::Reader::open_readfile("/data/GeoLite2-City.mmdb")
        .expect("Cannot open GeoIP database")
});

#[udf]
fn get_country_from_ip(ip: String) -> Option<String> {
    let ip_addr: IpAddr = ip.parse().ok()?;
    let city: geoip2::City = READER.lookup(ip_addr).ok()?;
    city.country?.iso_code.map(|s| s.to_string())
}

#[udf]
fn get_city_from_ip(ip: String) -> Option<String> {
    let ip_addr: IpAddr = ip.parse().ok()?;
    let city: geoip2::City = READER.lookup(ip_addr).ok()?;
    city.city?.names?.en
}
```

### External API Enrichment

```rust
#[udf]
async fn enrich_company_data(domain: String) -> Option<String> {
    let client = reqwest::Client::new();
    let response = client
        .get(format!("https://api.clearbit.com/v2/companies/find"))
        .query(&[("domain", domain)])
        .header("Authorization", format!("Bearer {}", API_KEY))
        .timeout(Duration::from_millis(500))
        .send()
        .await
        .ok()?;
    
    if response.status().is_success() {
        response.text().await.ok()
    } else {
        None
    }
}
```

## Analytics

### Calculate Metrics

```rust
#[udf]
fn calculate_engagement_score(
    views: i64,
    clicks: i64,
    shares: i64,
    time_spent_seconds: i64
) -> f64 {
    let view_weight = 1.0;
    let click_weight = 5.0;
    let share_weight = 10.0;
    let time_weight = 0.1;
    
    (views as f64 * view_weight) +
    (clicks as f64 * click_weight) +
    (shares as f64 * share_weight) +
    (time_spent_seconds as f64 * time_weight)
}

#[udf]
fn calculate_conversion_rate(conversions: i64, visitors: i64) -> Option<f64> {
    if visitors > 0 {
        Some((conversions as f64 / visitors as f64) * 100.0)
    } else {
        None
    }
}
```

### Time Series Analysis

```rust
#[udf]
fn detect_anomaly(value: f64, mean: f64, std_dev: f64, threshold: f64) -> bool {
    let z_score = (value - mean).abs() / std_dev;
    z_score > threshold
}

#[udf]
fn moving_average(values: Vec<f64>, window_size: usize) -> Vec<f64> {
    if values.len() < window_size {
        return vec![];
    }
    
    values.windows(window_size)
        .map(|window| window.iter().sum::<f64>() / window_size as f64)
        .collect()
}
```

## Data Quality

### Validation

```rust
/*
[dependencies]
validator = "0.16"
*/

use validator::validate_email;

#[udf]
fn is_valid_email(email: String) -> bool {
    validate_email(&email)
}

#[udf]
fn is_valid_phone(phone: String) -> bool {
    let digits: String = phone.chars()
        .filter(|c| c.is_ascii_digit())
        .collect();
    
    // Check for valid US phone number
    digits.len() == 10 || 
    (digits.len() == 11 && digits.starts_with('1'))
}

#[udf]
fn validate_json_schema(json: String, schema: String) -> bool {
    // Validate JSON against schema
    let value: serde_json::Value = match serde_json::from_str(&json) {
        Ok(v) => v,
        Err(_) => return false,
    };
    
    let schema: serde_json::Value = match serde_json::from_str(&schema) {
        Ok(s) => s,
        Err(_) => return false,
    };
    
    // Simplified validation - in practice use jsonschema crate
    true
}
```

### Data Cleaning

```rust
#[udf]
fn clean_text(text: String) -> String {
    text
        .trim()
        .replace("\n", " ")
        .replace("\r", " ")
        .replace("\t", " ")
        .split_whitespace()
        .collect::<Vec<_>>()
        .join(" ")
}

#[udf]
fn remove_html_tags(html: String) -> String {
    let re = Regex::new(r"<[^>]+>").unwrap();
    re.replace_all(&html, "").to_string()
}

#[udf]
fn fix_encoding(text: String) -> String {
    // Fix common encoding issues
    text
        .replace("â€™", "'")
        .replace("â€"", "–")
        .replace("â€"", "—")
        .replace("â€œ", """)
        .replace("â€", """)
}
```

## Complex Event Processing

### Pattern Detection

```rust
#[udf]
fn detect_fraud_pattern(
    transaction_amount: f64,
    avg_transaction: f64,
    location_change: bool,
    time_since_last: i64
) -> bool {
    // High amount relative to average
    let high_amount = transaction_amount > avg_transaction * 3.0;
    
    // Rapid transactions from different locations
    let suspicious_velocity = location_change && time_since_last < 60;
    
    high_amount || suspicious_velocity
}

#[udf]
fn detect_bot_behavior(
    requests_per_minute: i64,
    unique_pages: i64,
    user_agent: String
) -> bool {
    // Too many requests
    let high_rate = requests_per_minute > 100;
    
    // Suspicious patterns
    let low_diversity = unique_pages < 3 && requests_per_minute > 20;
    
    // Known bot user agents
    let bot_ua = user_agent.to_lowercase().contains("bot") ||
                 user_agent.to_lowercase().contains("crawler");
    
    high_rate || low_diversity || bot_ua
}
```

## Using UDFs in Queries

```sql
-- Data enrichment pipeline
SELECT 
    user_id,
    email,
    hash_email(email, 'salt123') as hashed_email,
    extract_domain(website_url) as company_domain,
    get_country_from_ip(ip_address) as country,
    calculate_engagement_score(
        page_views, 
        clicks, 
        shares, 
        time_on_site
    ) as engagement_score
FROM user_events
WHERE is_valid_email(email);

-- Fraud detection
SELECT 
    transaction_id,
    user_id,
    amount,
    detect_fraud_pattern(
        amount,
        avg_amount_last_30d,
        location != last_location,
        unix_timestamp() - last_transaction_time
    ) as is_suspicious
FROM transactions
WHERE amount > 100;

-- Log parsing and analysis
SELECT 
    parse_nginx_log(raw_log) as parsed,
    json_extract_nested(parse_nginx_log(raw_log), 'status') as status_code,
    json_extract_nested(parse_nginx_log(raw_log), 'ip') as client_ip,
    get_city_from_ip(
        json_extract_nested(parse_nginx_log(raw_log), 'ip')
    ) as city
FROM nginx_logs;
```

## Next Steps

- [Async UDFs](./async-udfs) - External service integration
- [Best Practices](./best-practices) - Performance optimization
- [Testing UDFs](./testing) - Unit and integration testing