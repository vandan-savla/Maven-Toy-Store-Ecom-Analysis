# Fuzzy Factory Store \| DAX Measure Reference

> Verify actual column names and relationships in your Power BI model
> before using these starter measures.

## Core measures

``` dax
Total Sessions =
DISTINCTCOUNT(website_sessions[website_session_id])
```

``` dax
Total Orders =
DISTINCTCOUNT(orders[order_id])
```

``` dax
Total Revenue =
SUM(orders[price_usd])
```

``` dax
Conversion Rate =
DIVIDE([Total Orders], [Total Sessions], 0)
```

This is orders per session. For the percentage of sessions that placed
at least one order:

``` dax
Converting Sessions =
DISTINCTCOUNT(orders[website_session_id])
```

``` dax
Session Conversion Rate =
DIVIDE([Converting Sessions], [Total Sessions], 0)
```

``` dax
Average Order Value =
DIVIDE([Total Revenue], [Total Orders], 0)
```

``` dax
Revenue Per Session =
DIVIDE([Total Revenue], [Total Sessions], 0)
```

## Pageview and engagement measures

``` dax
Pageview Count =
COUNTROWS(website_pageviews)
```

Create the following as a **calculated column** on `website_sessions`:

``` dax
Session Pageview Count =
COUNTROWS(RELATEDTABLE(website_pageviews))
```

Then create the bounce-rate measure:

``` dax
Bounce Rate =
DIVIDE(
    CALCULATE(
        [Total Sessions],
        website_sessions[Session Pageview Count] = 1
    ),
    [Total Sessions],
    0
)
```

``` dax
Average Pageviews Per Session =
DIVIDE([Pageview Count], [Total Sessions], 0)
```

## Checkout funnel

Create a disconnected helper table:

``` dax
Checkout Stages =
DATATABLE(
    "Stage", STRING,
    "Stage Order", INTEGER,
    {
        {"Cart", 1},
        {"Shipping", 2},
        {"Billing", 3},
        {"Order Confirmation", 4}
    }
)
```

Sort `Checkout Stages[Stage]` by `Checkout Stages[Stage Order]`. Do not
create a relationship from this table to the fact tables.

``` dax
Checkout Funnel Sessions =
VAR SelectedStage =
    SELECTEDVALUE('Checkout Stages'[Stage])
RETURN
    SWITCH(
        SelectedStage,
        "Cart",
            CALCULATE(
                DISTINCTCOUNT(website_pageviews[website_session_id]),
                website_pageviews[pageview_url] = "/cart"
            ),
        "Shipping",
            CALCULATE(
                DISTINCTCOUNT(website_pageviews[website_session_id]),
                website_pageviews[pageview_url] = "/shipping"
            ),
        "Billing",
            CALCULATE(
                DISTINCTCOUNT(website_pageviews[website_session_id]),
                website_pageviews[pageview_url] IN {"/billing", "/billing-2"}
            ),
        "Order Confirmation",
            CALCULATE(
                DISTINCTCOUNT(website_pageviews[website_session_id]),
                website_pageviews[pageview_url] = "/thank-you-for-your-order"
            )
    )
```

Use `Checkout Stages[Stage]` as the funnel category and
`[Checkout Funnel Sessions]` as values.

This counts distinct sessions reaching each URL. It does not prove that
sessions visited stages in that order. A strict sequential funnel
requires original event timestamps.

## Refund measures

Assuming `order_item_refunds` contains `refund_amount_usd` and
`order_id`:

``` dax
Total Refund Amount =
SUM(order_item_refunds[refund_amount_usd])
```

``` dax
Refunded Orders =
DISTINCTCOUNT(order_item_refunds[order_id])
```

``` dax
Order Refund Rate =
DIVIDE([Refunded Orders], [Total Orders], 0)
```

If the refunds table does not contain `order_id`, but does contain
`order_item_id`, derive refunded orders through `order_items`:

``` dax
Refunded Orders =
CALCULATE(
    DISTINCTCOUNT(order_items[order_id]),
    TREATAS(
        VALUES(order_item_refunds[order_item_id]),
        order_items[order_item_id]
    )
)
```

## Filter-context notes

-   Single-direction relationships may not propagate a pageview URL
    selection back to sessions or orders.
-   Use `TREATAS` when a measure must apply selected IDs to another
    table.
-   A disconnected funnel stage table controls funnel categories only.
    It does not automatically filter unrelated KPI cards.
-   Check Edit interactions for each visual and card.
-   Validate measures with slicers cleared and with individual date,
    channel and device selections.
-   Avoid duplicating order-level revenue by joining it to multiple
    order-item rows.
