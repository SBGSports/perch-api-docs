# Perch API Guide

## Introduction

Welcome to the Perch API Guide. This document is intended to walk you through the API beyond just endpoint documentation. It isn’t very long, but it should contain everything you need to know in order to successfully integrate. Please read it carefully and don’t hesitate to ask questions!

### Basics

Our API is an RPC-style, JSON based HTTP API. The URI defines the action, and almost every request is a `POST` request with a JSON body representing either filters by which you intend to query results or a payload to be used for creating or updating a record.

Almost all integrations only read data, and the API tokens that customers generate are read-only. If your integration needs to write data, contact Perch and we can grant write access.

### Endpoint Documentation

The endpoints are documented in an OpenAPI spec:

[https://github.com/SBGSports/perch-api-docs/blob/main/openapi.yaml](https://github.com/SBGSports/perch-api-docs/blob/main/openapi.yaml)

To browse the spec, open it in any OpenAPI viewer, for example [Swagger Editor](https://editor.swagger.io/). You can also generate a client from it.

### Changes

Our change management for the API is quite strict, as we ourselves use our API internally on mobile devices which are not easily updated. As such, we never make breaking changes without appropriate versioning and support old API functionality for a relatively long time.

Whenever improvements are made to our API, or if ever we need to make a breaking change, we will notify integrators via email and allow ample time for migration.

### Terms

* *User* - an individual user in Perch
  * *Admin* - a *User* with full read and write permissions over an organization
  * *Coach* - a *User* with full read permissions over an organization or part of it
  * *Athlete* - a *User* with only permission over themselves and their own data
* *Group* - a grouping of users, examples: team, lifting group, organization
  * *Organization* - a specific type of *Group* at the top of a customer’s hierarchy
* *Set* - an individual workout set tracked on Perch
* *Exercise* - an “activity” or “movement” definition, e.g. “Back Squat”

## Getting Started

Firstly, you need an API token. Perch customers can generate API tokens using the Perch web app [as shown here](https://perch.catapultsports.com/hc/en-us/articles/13220068158735-Generating-an-API-Token). Ask your customer to generate a token and share it with you securely. If for some reason the Perch customer cannot generate a token for you, have them connect you with Perch’s support team.

The next thing you need to determine is the `id` of the Perch *Organization* you’re using. To get that `id`, you can make a simple `GET` request to the `/v2/user` endpoint and extract the `id` from the `org_id` field in the response payload. The response contains one of the *Admin* level users from the *Organization* in Perch, and they will each have the `org_id` field which corresponds to the `id` of the *Organization*.

The `id` of the *Organization* will never change (this is true of all IDs in Perch), so this can be stored on your end and reused. As you move forward, you can supply this `id` to other endpoints in order to fetch data for the entire organization all at once.

## Core Concepts

### Authentication / Identity

As mentioned above, authentication for the API is handled with API tokens. Refer to [the above section](#getting-started) to find out how to acquire one. **Protect this token as if it’s a password or SSH key. Your API token provides access to client data.**

API tokens generated in Perch’s system are either associated with a specific *User* or a specific *Organization*. This is so that the API token can inherit permissions, with a caveat: API tokens that customers generate are read-only. Thus, an API token has the read permissions of the associated entity, and not the write permissions. If your integration needs write access, contact Perch.

As of writing, all customer generated tokens are associated with their *Organization*. When a token is associated with an *Organization*, the token can read *Users* and data from the entire *Organization*. This simplifies integration, as one API token can be used to access all data within the *Organization* and the token is not tied to any single *User* account.

To authenticate with the API, add an HTTP Header to each request of this form:

```
Authorization: Bearer <your token here>
```

The API tokens never expire, **so certainly be careful with their storage**. If ever you are worried that a token has been leaked, you can ask the customer for which the token is issued to delete the token, which will revoke its access, and create a new one. Do not hesitate to reach out to Perch for assistance.

### Time Base Filtering

Our core endpoints return data in bulk with paging, but it can be helpful to filter for specific time ranges. We use a consistent pattern across endpoints to accomplish this.

The two fields in the request you will want to use are `begin_time` and `end_time`. Both are expected to be floating point [Unix timestamps](https://en.wikipedia.org/wiki/Unix_time). You can supply either one on its own or both parameters. The effect of supplying these parameters is to filter objects which were created in the half-open interval `[begin_time, end_time)`. If you omit `begin_time`, all data whose timestamp falls before `end_time` are returned. If you omit `end_time`, all data with timestamps at or after `begin_time` are returned.

### Paging

Most of Perch’s endpoints that return bulk data perform cursor based paging. The response will always container a structure as follows:

```
{
    "data": [...],
    "next_token": <string or null>,
    "truncated": <true or false>
}
```

The `data` key will hold an array of the individual objects returned, and the value of the other fields depends on the number of records that the API found. If there were more than one page’s worth of records, `truncated` will be `true` and `next_token` will contain a string. Otherwise, `truncated` will be `false` and `next_token` will be `null`.

To get the next page of results, you simply need to pass the `next_token` received on the previous request into the JSON body of a new request (with the rest of the parameters the same). Repeat this until either the `truncated` boolean changes to `false` or the `next_token` is `null`.

### Related Objects

In many circumstances it is useful to have related objects attached to the ones you’re working with. The primary example within the Perch API is having the *Exercise* object associated with a *Set*.

Where possible, an additional `refs` field will be provided in the response payload. Everywhere this pattern is used is documented in the API endpoint documentation. This field will hold useful objects, organized by type, with relations to the core data being returned.

The structure of `refs` is common amongst all endpoints using it. `refs` is a JSON Object, holding string keys with logical names which map to Arrays of Objects. The keys denoting the type of data contained in the Arrays nested thereunder. The Arrays contain **only the unique referenced objects**. For instance, if you request *Sets* from the *Set* API Endpoint, many *Set* objects will refer to the same *Exercise*. As such, only one copy of each referenced *Exercise* will be present in the list under `refs -> exercises`. Below is an example using the *Set* response to help explain, which not only contains *Exercise* objects in the `refs` field, but also *User* objects:

```
{
    "data": [
        {
            "id": 1,
            "user_id": 2,
            "exercise_id": 3,
            ... omitted fields ...
        }
    ],
    "refs": {
        "exercises": [
            {
                "id": 3,
                "name": "Bench Press"
                ... omitted fields ...
            }
        ],
        "users": [
            {
                "id": 2,
                "first_name": "John"
                ... omitted fields ...
            }
        ],
    }
}
```

The pattern being followed here in this example (and in other endpoints) is that if the core data type (*Set* in this case) has the fields `exercise_id` and `user_id`, there would be both an `exercises` and a `users` key inside of the `refs` structure holding lists of the respective, unique objects referenced by the core data type.

### Rate Limits

The API returns HTTP Status Code `429` when you are getting rate limited. We have different rate limits on different endpoints, and if you’re not doing anything strange we don’t expect you to butt up against them.

In all likelihood, our infrastructure can handle what you’re going to throw at it. Fetching data daily, or even every 15 minutes should not be a problem. The only situation we want to avoid is tens or hundreds of extra requests per second, which is why the rate limits exist.

With all that said, please be kind.

## Mapping Your Users

*Users* in Perch have an `id` field which will never change and can be used forever as reference to the *User*.

To fetch all the users in the organization, use the `/v2/users` endpoint with the `group_id` parameter set to the *Organization’s* `id` we retrieved earlier. The response will contain one page of users, with potentially more to follow.

These *User* objects can be mapped to users in your system however you like. Common choices are email address or name. Once you’ve mapped the users using one of these other fields, we recommend you store the Perch *User’s* `id` as a “foreign id” since that will never change.

## Exercises

Perch’s database of *Exercises* is a mixture of “default” *Exercises* controlled by Perch and updated infrequently and customer generated *Exercises* which can be modified at any time. As a result, you will want to regularly synchronize *Exercises* from Perch with your system.

### Global vs Customer Specific

The *Exercises* Perch maintains are those with `org_id = null`, and are accessible to any customer organization in Perch. No matter which API token is used to request exercises, these “global” *Exercises* will be included in the response.

Those *Exercises* with `org_id = <some id>` are only accessible by the *Organization* in question.

When you request *Exercises* from `/v4/exercises`, you will receive all the *Exercises* that fall into both categories:

- Exercises that are global (`org_id = null`)
- Exercises that are specific to the Customer you’re requesting on behalf of (`org_id = <customer org_id>`)

It is worth noting that this means **you will receive different results when requesting using different API tokens*.*

### Organization Overrides

An *Organization* can customize some fields of a global *Exercise* for its own users, for example the `name` or whether the *Exercise* is visible. The API applies these customizations to every *Exercise* it returns, including those in the `refs` of a *Set* response. As a result, **the same global *Exercise* `id` can have different field values for different *Organizations***.

If you integrate with more than one Perch customer, store *Exercises* separately for each *Organization*, keyed by the *Organization's* `id` and the *Exercise's* `id`. Do not keep one shared copy of the global *Exercises*.

`/v4/exercises` also returns an `original` field on each customized *Exercise*. It holds the global value of each field the *Organization* changed.

### Hidden Exercises

By default, `/v4/exercises` does not return *Exercises* that the *Organization* has hidden. Older *Sets* can still refer to a hidden *Exercise*. To also receive hidden *Exercises*, set `invisible = INCLUDE`, or use the `refs` field of the *Set* response, which always contains the referenced *Exercises*.

## Sets

*Set* objects are the main data you will want to pull into your system. Each time a user performs an activity on Perch, a new *Set* is created as a record of that. Set’s can be fetched much the same as other types: `POST` to `/v3/sets`, and add the parameter `group_id = <Organization’s id>` to get all sets for the *Organization*.

The *Set* object has numerous fields holding statistics and rich data about the activity performed, but the most important fields are the following:

* `id` - the unique identifier for the *Set* (never changes)
* `user_id` - the `id` of the *User* who performed the *Set*
* `exercise_id` - the `id` of the *Exercise* which was performed
* `start_time` - the time when the *Set* began
* `error` - details about any errors that occurred during the recording of this *Set* or `null` if there were none.

### Types of Sets: Tracked vs Untracked

There are two distinct types of *Sets*, those that were tracked using a Perch camera and those that were entered manually.

**The default behavior of our old `/v2/sets` was to EXCLUDE untracked sets. The default behavior of the `/v3/sets` is to INCLUDE them.** You can opt out of receiving untracked sets by specifying `untracked = EXCLUDE`.

*Tracked Sets* will have the vast majority of the fields documented on the *Set* type filled in with non-null values. These sets were captured using a Perch camera, and therefore have rich statistics.

*Untracked Sets* will be very sparse compared to *Tracked Sets*. The same core fields exist on *Untracked Sets* (`user_id`, `exercise_id`, etc.), but none of the statistics available for a set captured using the Perch camera system will be available. The two primary fields available are:

* `num_reps`
* `weight`

### Unit Conversions

Every attribute of the *Set* object is documented under `components.schemas.Set` in the OpenAPI spec. The units for many of the attributes will not be the units in which you will want to work with the statistics.

In the documentation, each attribute is labeled with its unit, and more importantly an equation for converting it to the commonly desired unit. For example:

```yaml
avg_mean_velocity:
  type: number
  format: in / s
  description: >-
    The mean of `Rep.concentric_mean_velocity_z` across all reps.
    Convert to m/s: `m/in * avg_mean_velocity`
  example: 31.32
```

At the end of the description is the equation for converting `avg_mean_velocity` from inches per second to meters per second. The term `m/in` represents the number of meters per inch, which is a constant: `0.0254`. As such, using the example value, if you have `avg_mean_velocity` equal to `31.32 in/s`, converting it to `m/s` is as simple as `31.32 * 0.0254 = ~0.796 m/s`.

Some of the conversion equations are more complex than this example, but at the end of the day they will involve the same process of inserting the correct constants and multiplying them with the statistic from the *Set*.

### Data Modification

*Sets* are the most critical data type to synchronize between systems, and making sure the data is in sync is important.

Manual data entry on the weight room floor is error prone. Saving a *Set* with the wrong weight, user, or exercise is relatively common. As such, we allow users to modify sets after they have been saved to correct errors of this nature.

In order to make sure your system always has correct data, we strongly recommend implementing something to handle situations like the following:

1. User saves a *Set*, `id = 1`
2. Integrator pulls *Set* `1` into 3rd party system
3. User notices an error, and changes the `weight`
4. *Set* `1` is now out of sync in the two systems

To mitigate this problem, we recommend your choose one of the following approaches:

1. If you’re pulling data regularly throughout hours when data will be generated, you should implement a “lookback” time. For instance, if you’re pulling data every hour, you might consider pulling the last 1.5 hours of data every hour so you can update any *Set* objects that were changed after you pulled them.
2. Only ever pull data during off hours when users have had ample time to correct errors.

### Set vs Rep Level Statistics

Perch collects a variety of statistics for each repetition of a *Set*. In your integration, you may or may not want such granular data. As a result, we have parameters in the API requests that allow you to opt into more granular data if you so desire.

When using our latest endpoint to retrieve *Sets*, **the default behavior is to exclude the `reps` array from the *Set* objects** returned as this information is quite granular and often unnecessary. Most of the time, your needs might be met with the synthesized data present on the *Set*. For example if you want only the *Set’s* average and maximum value for “Mean Velocity”, you can refer directly to `avg_mean_velocity` and `max_mean_velocity` respectively. Many statistics have such synthesized values, see the *Set* type in our endpoint documentation.

#### Accessing Rep Statistics

To have the raw repetition data returned from the *Set* endpoint, be sure to set the `include_reps = true`. This will give you access to the `reps` field, which contains the most granular data Perch provides.

## Migration from `/v3/exercises` to `/v4/exercises`

`/v4/exercises` returns the same *Exercises* and fields as `/v3/exercises`, plus some new fields. No existing field changed, so you can switch by changing the URL.

## Migration from `/v2/exercises` to `/v3/exercises`

The only change between these endpoints is that more types of exercises will be returned to you. These exercises are not substantially different from the others, so this should be a very minor migration.

## Migration from `/v2/sets` to `/v3/sets`

The core changes between the `/v2/sets` and `/v3/sets` endpoints revolve around default behavior and the inclusion of referenced objects.

### Breaking Changes

- Omission of `reps` and the removal of the `no_reps` parameter
  - By default, the `reps` field on *Set* is no longer included. To continue receiving `reps` on the sets returned to you, use the `include_reps` field to opt in by setting it to `true`
  - The endpoint no longer allows you to opt out of receiving the `reps` field on sets using the `no_reps` parameter. **The default behavior is to omit the `reps` field in the response.**
- Untracked sets
  - The `v3` endpoint includes *Untracked Sets* **by default in the response*.*
  - See [Types of Sets](#types-of-sets-tracked-vs-untracked) for more information
  - To opt out of receiving any *Untracked Sets*, set the `untracked` parameter equal to `EXCLUDE` to have them removed from the response
- `clean_reps`
  - This parameter tells the endpoint whether or not you would like any “Ghost reps” removed from the list of `reps` on a *Set*. **The default value for `clean_reps` has changed to `true`**
- Nested data removed
  - In responses from the `/v2/sets` endpoint, there were two nested objects present on each *Set*: `user` and `exercise`. **These nested objects will no longer be present in `/v3/sets` and instead will be present in the `refs` field of the response (as described above)**

### The `reps` field and Error handling

#### Do you still need to use `reps`?

The `reps` field of *Set* holds an array of *Rep* objects. The `reps` hold granular and very raw data. **Today, most integrations can find all the data they need using the “summary” statistics preset on the *Set* itself** and avoid parsing `reps` entirely (e.g. `avg_mean_velocity` and `max_mean_velocity`). See the documentation for these fields to understand more about how they are computed.

#### If you do need `reps`, errors removed by default

Perch’s camera system is very good, but not perfect. Occasionally, *Sets* aren’t tracked correctly and contain extra spurious reps.

The default behavior of `/v2/sets` was to leave the extra reps in the `reps` list on a *Set* unless you specified `clean_reps=true` in your request. **This behavior has now changed in `/v3/sets`: you no longer need to specify `clean_reps=true`, as that is the default behavior.**

If you were not ever utilizing the `reps` previously and had written code to parse the `ghost_rep_indices` from the *Set*’s `error` field, you can safely remove that code and let the `/v3/sets` endpoint handle it for you (automatically).

## Final Notes

Use of the Perch API is subject to the [Catapult Standard Terms](https://www.catapult.com/standard-terms).

If you notice any bugs or potential security issues with our API or documentation, please let us know promptly. Email us at [support@perch.fit](mailto:support@perch.fit).
