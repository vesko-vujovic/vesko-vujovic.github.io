---
title: "🛡️ Data Quality Checks vs Unit Tests: The Line You Need to Draw"
draft: false
date: 2025-10-28T15:06:41+02:00
tags:
  - Data Quality
  - Testing
  - Data Engineering
cover:
  image: "/posts/agentic-ai-for-business-leaders/agentic_ai_for_business_leaders.png"
  alt: "Data Quality Checks vs Unit Tests"
  caption: "Data Quality Checks vs Unit Tests"
---

Your data quality dashboard shows all green. Your pipeline just merged duplicate records and nobody noticed for a week.

Or maybe it's the opposite. Your unit tests all pass. You deploy with confidence. Then your pipeline breaks in production because the upstream API changed a field name.
Does this brings vivid memories? 😊

Here's the fact: **most data engineering teams either over-rely on data quality checks or confuse them with unit tests.** 

Both are critical, but they catch completely different problems. If you don't mix them up, you'll have blind spots in your testing strategy.

In this post, I'll show you real examples where each approach fails on its own. By the end, you'll know exactly when you need a unit test versus when you need a data quality check.

## 🔍 Quick Definitions (So We're on the Same Page)

Let's get clear on what we're talking about.

**Data quality checks validate your data.** They answer questions like: Is this field null when it shouldn't be? Are these values in the expected range? Does this schema match what we expect?

**Unit tests validate your code logic.** They answer: Does this function do what I think it does? Does my transformation handle edge cases correctly? Does my join logic work as intended?

_Both prevent problems. But they prevent different problems._

### ⚠️ Real Example #1: Where Data Quality Checks Miss the Bug
**The Scenario: Revenue Calculation with Returns**

Let's say you're building a pipeline to calculate daily revenue for an e-commerce platform. Simple enough, right?

Here's your transformation logic:

```python

def calculate_revenue(df):
    df['revenue'] = df['price'] * df['quantity']
    return df.groupby('date')['revenue'].sum()

```

You set up your data quality checks:

- ✅ All prices are positive
- ✅ No null quantities
- ✅ Revenue values are reasonable (between $0 and $1M per day)
- ✅ No missing dates

Everything passes. Your dashboard is green. Ship it.

But here's what you missed: **some quantities are negative. These are returns. A quantity of -2 means the customer returned 2 items.**

Your calculation multiplies price by quantity. So a $50 item with quantity -2 gives you -$100 in revenue. You're subtracting revenue when you should be tracking returns separately.

The data looks totally fine to your quality checks. Prices are positive. Quantities exist (they're just negative). Revenue totals fall within expected ranges because returns are a small percentage of orders.

But your business logic is wrong. You're treating returns as negative revenue instead of handling them properly.

**Here's what you should have written inside of your logic:**

```python
    def calculate_revenue(df):
    #Calculate total revenue, handling returns correctly
    # Separate orders from returns
    orders = df[df['quantity'] > 0]
    returns = df[df['quantity'] < 0]
    
    # Calculate separately
    order_revenue = (orders['price'] * orders['quantity']).sum()
    return_amount = (returns['price'] * returns['quantity'].abs()).sum()
    
    return {
        'gross_revenue': order_revenue,
        'returns': return_amount,
        'net_revenue': order_revenue - return_amount
    }
```

**A unit test would have caught this:**
```python
    def test_calculate_revenue_handles_returns():
    test_data = pd.DataFrame({
        'date': ['2024-01-01', '2024-01-01', '2024-01-01'],
        'price': [50, 50, 100],
        'quantity': [2, -1, 1]  # Mix of orders and returns
    })
    
    result = calculate_revenue(test_data)
    
    # Should calculate net revenue correctly
    expected = (50 * 2) + (100 * 1) - (50 * 1)  # 100 + 100 - 50 = 150
    assert result['net_revenue'] == 150
```

The data passes all quality checks. But your logic is broken. That's what unit tests catch.

### ⚠️ Real Example #2: Where Data Quality Checks Miss the Bug

**The Scenario: Conversion Rate Calculation**

You're building a marketing dashboard that tracks conversion rates. The metric is simple: what percentage of users who view a product page actually make a purchase?

Here's your code:

```python
    def calculate_conversion_rate(events_df):
    """Calculate conversion rate from page views to purchases"""
    # Count page views
    page_views = events_df[events_df['event_type'] == 'page_view'].shape[0]
    
    # Count purchases
    purchases = events_df[events_df['event_type'] == 'purchase'].shape[0]
    
    # Calculate conversion rate
    conversion_rate = (purchases / page_views) * 100
    
    return conversion_rate
```



Your data quality checks:

- ✅ Conversion rate is between 0% and 100%
- ✅ No null values
- ✅ Page view count is positive
- ✅ Purchase count is positive
- ✅ Numbers are within expected ranges (2-5% conversion is normal)

Everything passes. The metric shows 3.2% conversion rate. Looks reasonable.

But there's a critical flaw in your logic. You're counting ALL page views and ALL purchases, regardless of whether they're from the same user session.

A user might view a product 10 times over a week before finally purchasing. Your calculation counts 10 page views and 1 purchase. But that's not how conversion works. You need to track unique user journeys.

Even worse, if a user purchases without viewing the product page first (maybe they clicked directly from an email), you're counting a purchase with no corresponding page view. This inflates your conversion rate.

The data passes all checks. The percentage is within a reasonable range. But your business logic doesn't match what conversion rate actually means.


**What you should have written:**

```python
def calculate_conversion_rate(events_df):
    """Calculate conversion rate: users who purchased / users who viewed"""
    # Get unique users who viewed product pages
    viewers = events_df[events_df['event_type'] == 'page_view']['user_id'].unique()
    
    # Get unique users who made purchases AND also viewed
    purchasers = events_df[
        (events_df['event_type'] == 'purchase') & 
        (events_df['user_id'].isin(viewers))
    ]['user_id'].unique()
    
    # Calculate conversion rate
    if len(viewers) == 0:
        return 0.0
    
    conversion_rate = (len(purchasers) / len(viewers)) * 100
    
    return conversion_rate
```

**A unit test with realistic user behavior would catch this:**

```python
def test_conversion_rate_counts_unique_user_journeys():
    test_data = pd.DataFrame({
        'user_id': [1, 1, 1, 2, 2, 3, 4],
        'event_type': ['page_view', 'page_view', 'purchase',  # User 1: viewed 2x, purchased
                      'page_view', 'page_view',              # User 2: viewed 2x, no purchase
                      'purchase',                             # User 3: purchased without viewing
                      'page_view']                            # User 4: viewed only
    })
    
    result = calculate_conversion_rate(test_data)
    
    # 3 users viewed (1, 2, 4)
    # 1 user purchased after viewing (1)
    # User 3's purchase doesn't count (no view)
    # Expected: 1/3 = 33.33%
    assert abs(result - 33.33) < 0.01
```
**Your data quality checks can't tell you that your aggregation logic is conceptually wrong. The numbers look fine. But you're measuring the wrong thing.**


### ⚠️ Real Example #3: Where Data Quality Checks Miss the Bug

**The Scenario: Join Logic Error**

You're creating a report showing all customers and their order counts. Simple aggregation.
Here's your Spark code:
```python
def create_customer_order_report(customers_df, orders_df):
    """Create report of customers with order counts"""
    # Join customers with orders
    report = customers_df.join(
        orders_df,
        customers_df.customer_id == orders_df.customer_id,
        'inner'  # This is the bug
    )
    
    # Count orders per customer
    result = report.groupBy('customer_id', 'customer_name') \
                   .agg(count('order_id').alias('order_count'))
    
    return result
```

Your data quality checks on the output:

- ✅ All customer_ids are valid
- ✅ No null order counts
- ✅ Order counts are positive integers
- ✅ No duplicate customer records

Everything passes. The data looks perfect.

But you used an INNER JOIN. Customers with no orders don't appear in your report at all. They vanished.

Your stakeholder asks, "Where are all the new sign-ups from last week?" You don't have an answer. They're not in the report because they haven't placed an order yet.
The data quality checks can't catch this. There are no nulls. No invalid IDs. No anomalies in the data that does exist. The problem is the data that doesn't exist.

**A unit test with the right test data would catch this:**
```python
    def test_report_includes_customers_with_no_orders():
    # Test data with customers who have no orders
    customers = spark.createDataFrame([
        (1, 'Alice'),
        (2, 'Bob'),
        (3, 'Charlie')  # This customer has no orders
    ], ['customer_id', 'customer_name'])
    
    orders = spark.createDataFrame([
        (1, 1, 'order_1'),
        (2, 2, 'order_2')
        # No orders for customer 3
    ], ['order_id', 'customer_id', 'order_ref'])
    
    result = create_customer_order_report(customers, orders)
    
    # Should include all 3 customers
    assert result.count() == 3
    
    # Charlie should have 0 orders
    charlie = result.filter(result.customer_name == 'Charlie').first()
    assert charlie.order_count == 0
```

### 🔄 Real Example #4: Where Unit Tests Miss the Problem
**The Scenario: Upstream Schema Change**

Now let's flip the coin. Here's where unit tests fail you.
You're building a pipeline that ingests user data from an external API. You parse the JSON response and extract key fields.
```python
    def parse_user_data(json_response):
    """Parse user data from API response"""
    users = []
    for item in json_response['data']:
        user = {
            'user_id': item['user_id'],
            'email': item['email'],
            'name': item['full_name'],
            'signup_date': item['created_at']
        }
        users.append(user)
    return users
```

Your faithful servant unit test:

```python
    def test_parse_user_data():
    mock_response = {
        'data': [
            {
                'user_id': 123,
                'email': 'alice@example.com',
                'full_name': 'Alice Smith',
                'created_at': '2024-01-15'
            }
        ]
    }
    
    result = parse_user_data(mock_response)
    
    assert len(result) == 1
    assert result[0]['user_id'] == 123
    assert result[0]['name'] == 'Alice Smith'
```

Unit test passes. ✅ You deploy.

Two weeks later, the upstream team refactors their API. They change user_id to `userId` (camelCase). They don't tell you because "it's a minor change."
Your pipeline breaks. Every single run fails with `KeyError: 'user_id'`.

Your unit test is perfect. Your code logic is correct. But the data structure changed, and your unit test uses mock data that doesn't reflect reality.

**What would catch this:**
```python
# Data quality check on the raw input
def validate_api_schema(json_response):
    """Validate API response has expected schema"""
    required_fields = ['user_id', 'email', 'full_name', 'created_at']
    
    if 'data' not in json_response:
        raise ValueError("Missing 'data' field in API response")
    
    if len(json_response['data']) > 0:
        first_item = json_response['data'][0]
        missing_fields = [f for f in required_fields if f not in first_item]
        
        if missing_fields:
            raise ValueError(f"API schema changed. Missing fields: {missing_fields}")
    
    return True
```

This data quality check runs on the actual data coming from the API. It catches schema changes immediately.
Your code logic was fine. Your unit test was fine. But the real-world data didn't match your assumptions.

### 🔄 Real Example #5: Where Unit Tests Miss the Problem

**The Scenario: Duplicate Records from Source System**

You're aggregating daily active users from your application logs.
Here's your code:

```python
def count_daily_active_users(events_df):
    """Count unique active users per day"""
    return events_df.groupby('date')['user_id'].nunique()
```

Your unit test:

```python
    def test_count_daily_active_users():
        test_data = pd.DataFrame({
            'date': ['2024-01-01', '2024-01-01', '2024-01-02'],
            'user_id': [1, 1, 2]  # User 1 appears twice on same day
        })
        
        result = count_daily_active_users(test_data)
        
        # Should count unique users
        assert result['2024-01-01'] == 1  # User 1, counted once
        assert result['2024-01-02'] == 1  # User 2
```

Test passes. ✅ Logic is correct.

Then your upstream logging service has a bug. It starts double-logging events. Every user action gets written to the database twice.

Your aggregation logic still works perfectly. It counts unique users. But your counts are now wrong because you're deduplicating events that shouldn't exist in the first place.

Your dashboard shows 50,000 daily active users. In reality, half of those are duplicates from the source system. Your business makes decisions based on inflated numbers.

Your code is doing exactly what it's supposed to do. Your unit test validates that the logic works. But the input data is garbage.

```python
    # Data quality check on input data
def check_for_duplicate_events(events_df):
    """Check if we're receiving duplicate events"""
    # Count how many events exist per user per timestamp
    duplicates = events_df.groupby(['user_id', 'event_timestamp', 'event_type']).size()
    
    duplicate_count = (duplicates > 1).sum()
    
    if duplicate_count > 0:
        # Alert if more than 1% of events are duplicates
        duplicate_rate = duplicate_count / len(events_df)
        if duplicate_rate > 0.01:
            raise ValueError(
                f"High duplicate rate detected: {duplicate_rate:.2%}. "
                f"Possible issue with upstream logging system."
            )
```

This check runs on the actual data and catches anomalies that your unit test can't see because your test uses clean, hand-crafted data.

### 🔄 Real Example #6: Where Unit Tests Miss the Problem
**The Scenario: Business Logic Drift**

You're building a discount validation pipeline. Company policy says discounts can't exceed 50%.
Here's your code:

```python
def validate_discount(discount_percent):
    """Validate discount is within company policy"""
    max_discount = 50.0
    
    if discount_percent > max_discount:
        raise ValueError(f"Discount {discount_percent}% exceeds maximum {max_discount}%")
    
    return discount_percent
```

Test passes. ✅ Your logic matches the business rules.
You deploy. Everything works great.

Three months later, marketing launches a massive promotional campaign. Black Friday. They need to offer 70% off to compete.

Your pipeline rejects every single transaction. Orders are getting dropped. Marketing is furious. Sales are lost.
Your code is working perfectly according to the rules you coded. Your unit test validates those rules. But the business rules changed, and nobody updated your code.

**What would catch this:**

```python
# Data quality check that alerts on unexpected patterns
def check_discount_rejection_rate(transactions_df):
    """Alert if we're rejecting an unusual amount of transactions"""
    total = len(transactions_df)
    rejected = len(transactions_df[transactions_df['status'] == 'rejected'])
    
    rejection_rate = rejected / total
    
    # Normal rejection rate is < 5%
    if rejection_rate > 0.05:
        raise ValueError(
            f"High rejection rate detected: {rejection_rate:.2%}. "
            f"Possible issue with validation rules or new business requirements."
        )
```

This data quality check monitors the behavior in production. It catches when your "correct" code is actually rejecting valid business transactions.
Your logic is correct for the old rules. But rules changed.


## 📏 Drawing the Line: What Each Should Test

Now that you've seen both sides fail, let's draw the line clearly.

### Unit Tests Should Verify:

- **Your transformation logic is correct**: Does my calculation work as intended?
- **Edge cases are handled**: What happens with nulls, negative values, empty lists, returns?
- **Your joins work as intended**: Am I using the right join type? Are keys matching correctly?
- **Your calculations produce expected results**: Does this aggregation logic give me the right answer?
- **Conditional logic handles all branches**: Do all my if/else paths work correctly?

### Data Quality Checks Should Verify:

- **Data conforms to expected schema**: Are the fields I need actually present?
- **Values are within expected ranges**: Is this revenue number suspiciously high or low?
- **No unexpected nulls or missing data**: Are required fields populated?
- **Data freshness and completeness**: Is this data recent? Are we missing records?
- **Relationships between tables are maintained**: Do foreign keys match? Are there orphaned records?
- **Volume anomalies**: Did we suddenly get 10x more records than usual?

## Conclusion

**Here's the thing you need to know: Don't pick one over the other. You need both.**


`Unit tests don't directly check your data quality, but they're essential for preventing bad code from producing bad data in the first place.`

`Think of it this way:`
- `Unit tests prevent YOU from creating bad data`
- `Data quality checks prevent OTHERS from sending you bad data`

Both protect your data quality, just at different stages


The question isn't "which one should I use?" The question is "am I testing my logic or validating my data?"

If you're testing logic → write unit tests.
If you're validating data → add quality checks.

__*Without unit tests, your buggy code will pollute your data warehouse.*__
__*Without data quality checks, upstream issues will pollute your data warehouse.*__

*You need both layers of defense.*

Both are essential. Both catch different problems. Together, they keep your pipelines reliable.
