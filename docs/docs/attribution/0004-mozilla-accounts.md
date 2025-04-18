---
render_macros: true
---

# Mozilla accounts attribution

For products such as Mozilla VPN, Relay, and Monitor, we use Mozilla account as an authentication and subscription service. In addition to Google Analytics for basic conversion tracking, we attribute web page visits and clicks and through to actual subscriptions and installs by passing a specific allow-list of known query parameters through to the subscription platform. This is accomplished by adding referral data as parameters to sign up links on product landing pages.

## How does attribution work?

When using any of the [Mozilla accounts helpers](../mozilla-accounts.md) in bedrock, a default set of attribution parameters are added to each account sign-in / subscription link on a product landing page. Here's what we set for Mozilla VPN, as an example:

| Name              | Description                                                                              | Example value                      |
| ----------------- | ---------------------------------------------------------------------------------------- | ---------------------------------- |
| `utm_source`      | Query param identifying the referring site which sent the visitor.                       | `www.mozilla.org-vpn-product-page` |
| `utm_medium`      | Query param identifying the type of link, such as referral, cost per click, or email.    | `referral`                         |
| `utm_campaign`    | Query param identifying the specific marketing campaign that was seen.                   | `vpn-product-page`                 |
| `entrypoint`      | ID for which page of the website the request originates from (used for funnel analysis). | `www.mozilla.org-vpn-product-page` |
| `device_id`       | ID that correlates to the active device being used (used for funnel analysis).           | Alpha numeric string               |
| `flow_id`         | The flow identifier. A randomly-generated opaque ID (used for funnel analysis).          | Alpha numeric string               |
| `flow_begin_time` | The time at which a flow event occurred (used for funnel analysis).                      | Timestamp                          |
| `service`         | Product ID used for data analysis in BigQuery (optional).                                | Alpha numeric string               |

When performing data analysis, the default `UTM (Urchin Tracking Module)`{.interpreted-text role="abbr"} values above are what we equate to "direct" traffic (i.e. someone came to the landing page directly then subscribed. They did not arrive from a specific marketing campaign or other channel).

If we do detect that someone came from a marketing campaign or other form of referral, then we have logic in place that will replace the default UTM parameters on each link with more specific referral data, so that we can attribute subscriptions to individual campaigns.

We also support passing several other optional referral parameters:

| Name                    | Description                                                                   | Example value        |
| ----------------------- | ----------------------------------------------------------------------------- | -------------------- |
| `coupon`                | A coupon code that can be automatically applied at checkout (case sensitive). | `VPN20`              |
| `entrypoint_experiment` | Experiment name ID.                                                           | Alpha numeric string |
| `entrypoint_variation`  | Experiment variation ID                                                       | Alpha numeric string |

### Attribution logic

See the [Application Logic Flow Chart](https://www.figma.com/file/etj3w6Sv2QLXIPH5rdTW4U/Firefox-Account-Referrals---Attribution-Flow?node-id=0%3A1&t=OGAxLbRzT99Op8op-1) for a visual representation of the steps below (Mozilla access only).

1.  A website visitor loads a product landing page in their web browser.
2.  A [JavaScript function](https://github.com/mozilla/bedrock/blob/main/media/js/base/fxa-attribution.es6.js) then checks for and validates attribution data via a list of known URL parameters (see tables above).
3.  If there are UTM parameters in the referral data, then those are used to replace the default values in each sign up link. Additionally if `coupon` or `entrypoint_experiment` params found, those are also appended.
4.  If no UTM params exist, but there is a referrer cookie set, then the cookie value is used for `utm_campaign` and `utm_source` is set to `www.mozilla.org`. This cookie is often set when we display a "Get Mozilla VPN" promo on another mozorg page, such as `/whatsnew`.
5.  If there's no referrer cookie, we next look at `document.referrer` to see if the visitor came from a search engine. If found, we set `utm_medium` as `organic` and `utm_source` as the search engine name.
6.  Next, an [metrics function](https://github.com/mozilla/bedrock/blob/main/media/js/base/fxa-product-button.es6.js) makes a flow API request to the Mozilla accounts authentication server. The request returns a series of metrics parameters that are used to track progress through the sign-up process. These "flow" parameters are also appended to each sign up link in addition to the existing attribution data.
7.  When someone clicks through and completes the sign up process, attribution data we passed through is emitted as event logs. This data is then joined to a person's Mozilla account data during the Data Science team's ETL process (Extract, Transform, Load), where data is then brought together in Big Query.

!!! note
    UTM parameters on sign up links will only be replaced if the page URL contains both a valid `utm_source` and `utm_campaign` parameter. All other UTM parameters are considered optional, but will still be passed through, as long as the required parameters exist. This is to avoid mixing referral data from different campaigns.


## Attribution referrer cookie

In situations where we want to try and track a visitor's first entry point, say if someone lands on a `/whatsnew` page and then clicks on a "Get Mozilla VPN" promo link, then we can set a referral cookie in someone's browser when they click a same-site link (step 4 in the list above).

The cookie can be set simply by adding the class name `js-fxa-product-referral-link` to a same-site link, along with a `data-referral-id` attribute. When clicked, our attribution logic will use the value of `data-referral-id` to augment `utm_campaign` when someone click through to the product page.

For example, a referral with `data-referral-id="navigation"` would result in the following utm parameters being set on sign up links in the product landing page:

> -   `utm_source=www.mozilla.org`.
> -   `utm_campaign=navigation`.
> -   `utm_medium=referral`.

### Mozilla VPN referral link helper

For Mozilla VPN, there's a `vpn_product_referral_link` helper built specifically to help implement account referral links to the VPN landing page:

``` jinja
{{ vpn_product_referral_link(
    referral_id='navigation',
    link_to_pricing_page=True,
    link_text='Get Mozilla VPN',
    class_name='mzp-t-secondary mzp-t-md',
    page_anchor='#pricing',
    optional_attributes= {
        'data-cta-text' : 'Get Mozilla VPN',
        'data-cta-type' : 'button',
        'data-cta-position' : 'navigation',
    }
) }}
```

The helper supports the following parameters:

| Parameter name         | Definition                                                                                       | Format                            | Example                                                                                                     |
| ---------------------- | ------------------------------------------------------------------------------------------------ | --------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `referral_id`          | The ID for the referring page / component. This serves as a value for 'utm_campaign'.          | String                            | 'navigation'                                                                                              |
| `link_to_pricing_page` | Link to the pricing page instead of the landing page (defaults to `False`).                      | Boolean                           | True                                                                                                        |
| `link_text`            | The link copy to be used in the call to action.                                                  | Localized string                  | 'Get Mozilla VPN'                                                                                         |
| `class_name`           | A class name to be applied to the link (typically for styling with CSS).                         | String of one or more class names | 'mzp-t-secondary mzp-t-md'                                                                                |
| `page_anchor`          | An optional page anchor for the link destination.                                                | String                            | '#pricing'                                                                                                |
| `optional_attributes`  | An dictionary of key value pairs containing additional data attributes to include in the button. | Dictionary                        | {'data-cta-text': 'Get Mozilla VPN', 'data-cta-type': 'vpn', 'data-cta-position': 'navigation'} |

The cookie has the following configuration:

| Cookie name               | Value               | Domain            | Expiry |
| ------------------------- | ------------------- | ----------------- | ------ |
| `fxa-product-referral-id` | Campaign identifier | `www.mozilla.org` | 1 hour |

## Flow metrics

Whilst UTM parameters are passed through to sign up links automatically for any page of the website, in order for flow metrics to be added to links, a specific JavaScript bundle needs to be manually run in the page that requires it. The reason why it's separate is that depending on the situation, flow metrics need to get queried and added at specific times and conditions (more on that below).

To add flow metrics to links, a page's respective JavaScript bundle should import and initialize the `FxaProductButton` script.

``` javascript
import FxaProductButton from './path/to/fxa-product-button.es6.js';

FxaProductButton.init();
```

The above JS is also available as a pre-compiled bundle, which can be included directly in a template:

``` jinja
{{ js_bundle('fxa_product_button') }}
```

When ``init()`` is called, flow metrics will automatically be added to add account sign up links on a page.

!!! important
    Requests to metrics API endpoints should only be made when an associated `CTA (Call To Action)`{.interpreted-text role="abbr"} is visibly displayed on a page. For example, if a page contains both a Mozilla accounts sign-up form and a Mozilla Monitor button, but only one CTA is displayed at any one time, then only the metrics request associated with the visible CTA should occur.


!!! note
    For links generated using the `fxa_link_fragment` helper, you will also need to manually add a CSS class of `js-fxa-product-button` to trigger the script.


## Google Analytics guidelines

For `GTM (Google Tag Manager)`{.interpreted-text role="abbr"} `datalayer` attribute values in Mozilla account links, please use the `analytics<analytics>`{.interpreted-text role="ref"} documentation.
