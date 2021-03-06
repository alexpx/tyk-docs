---
date: 2017-03-23T17:08:35Z
title: Rate Limiting
menu:
  main:
    parent: "Control & Limit Traffic"
weight: 1 
---

## Rate Limiting Overview

Also known as throttling, Tyk API will actively only allow a key to make `x` requests per `y` time period. This is very useful if you want to ensure your API does not get flooded with requests.

### How do Rate Limits Work?

There are two rate limiters in Tyk as of v2.3: The hard-synchronised rate limiter (the rate limiter used in v2.2) and the distributed rate limiter (also referred to as the DRL in future). Both rate limiters come with different benefits and trade-offs, and in v2.3 we have opted to make the DRL the default rate limiter.

### Hard-synchronised Rate Limiter

Here the limit is enforced using a pseudo "leaky bucket" mechanism: Tyk will record each request in a timestamped list in Redis, at the same time it will count the number of requests that fall between the current time, and the maximum time in the past that encompasses the rate limit (and remove these from the list). If this count exceeds the number of requests over the period, the request is blocked.

This approach means that rate limits are applied across all Gateway instances equally and near-instantaneously and also that the actual limit is a "moving window" so that there is no fixed point in time to flood the limiter or execute more requests than is permitted by any one client.

The downside of this rate limiter is that it puts a high amount of traffic to and from Redis, which can cause Redis itself to become a bottleneck in high-traffic situations.

### Distributed Rate Limiter (DRL)

The distributed rate limiter (DRL) operates on an eventual-consistency basis, it too is a "leaky bucket" algorithm, but the rate limiter is not synchronised explicitly via Redis across all instances. Instead, the rate limiter is entirely in-memory to the instance servicing the request, and the "size" of the token value is determined by a quorum established between Tyk instances that share a common zone or tag group.

This approach means that Tyk will continually measure load on each instance that is running, and then use the load across all instances to calculate a value by which to normalise the rate limiter leaky buckets across all instances - this happens eventually (within a second or so) and is entirely dynamic. If a new instance joins the cluster, or an instance leaves the cluster, the token bucket value is recalculated and rate limits rebalance.

The benefit of this approach is scalability and speed - the DRL is much more performant and puts much less pressure on Redis, meaning smaller deployments and higher availability.

The DRL can work accurately only if your rate limit is a few times higher then number of your servers. DRL has a built in mechanism to automatically switch to a hard-synchronised rate limiter (a bit slower but more accurate) on a per user basis, if the rate limit is too low. You can control this behaviour using the `drl_threshold` option, which specifies the minimum number of requests PER server, in order to enable the DRL algorithm. By default `drl_threshold`  is set to 5, which means that if you have 3 servers, users with rate limit of > 15 (5 * 3) will use the DRL algorithm, and other users will fallback to the hard-synchronised rate limiter.

### Global Rate Limiter

Using `global_rate_limit` API definition field you can specifies a global API rate limit in the following format: `{"rate": 10, "per": 1}` similar to policies or keys. 
 
The API rate limit is an aggregate value across all users, which works in parallel with user rate limits, but has higher priority.

#### Global Rate Limiter Algorithm

The algorithm used by the global rate limiter will be the same algorithm configured for the key rate limiter.

#### Setting Global Rate Limits from the Dashboard 
 
From the Dashboard, you can specify the Global Rate Limits from the API Designer: 

![global-limits](/docs/img/2.10/rate_limits_quotas.png)

### Can I disable the rate limiter?

Yes, the rate limiter can be disabled for an API Definition by checking **Disable Rate Limits** in your API Designer, or by setting the value of `disable_rate_limit` to `true` in your API definition.

Alternatively, you could also set the values of `Rate` and `Per (Seconds)` to be 0 in the API Designer.

{{< note success >}}
**Note**  

Disabling the rate limiter at the global level does not disable the rate limiting at the key level.  Tyk will enforce the rate limit at the key level regardless of this setting.
{{< /note >}}

### Can I rate limit by IP address?

Not yet, though IP-based rate limiting is possible using custom pre-processor middleware JavaScript that generates tokens based on IP addresses.

## Setting a rate limit from the Dashboard

1.  From **System Management** > **Keys** > **Add Key**.

2.  Ensure the new key has access to the APIs you wish it work with by selecting the API from **Access Rights** > **Choose API**. Turn the **Set per API Limits and Quota** options on.

3.  From the **Rate Limit** section, select the **rate** (number of requests) and the **per seconds** period. If the period is not available in the drop down, you can set it to a custom value using the [Tyk Dashboard API](/docs/tyk-apis/tyk-dashboard-api/api-definitions/).
    
![Tyk API Gateway Rate Limits](/docs/img/2.10/api_rate_limits_keys.png)

4.  Save the token, it will be created instantly.

## Set a rate limit on the session object (API)

All actions on the session object must be done via the Dashboard or Gateway REST API.

1. Ensure that `allowance` and `rate` are set to the same value, this should be number of requests to be allowed in a time period, so if you wanted 100 requests every second, set this value to 100.

2. Ensure that `per` is set to the time limit. Again, as in the above example, if you wanted 100 requests per second, set this value to 1. If you wanted 100 per 5 seconds, set this value to 5 etc.