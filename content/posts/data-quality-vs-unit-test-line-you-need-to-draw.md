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

✅ All prices are positive

✅ No null quantities

✅ Revenue values are reasonable (between $0 and $1M per day)

✅ No missing dates

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