---
title: Calendar
---
# Calendar Extra

Add pagination filtering by calendar time unit (Year, Month, Week, Day).

This extra is a sort of wrapper around another backend extra. It adds single or multiple chained calendar navs that act as calendar filters on the collection records, placing each record in its time unit. 

![calendar_app](../assets/images/calendar-app.png)
_Screenshot from the single-file self-contained [pagy_calendar_app.ru](https://github.com/ddnexus/pagy/blob/master/apps/pagy_calendar_app.ru) demo_

## Use cases

This extra makes sense when the result to paginate have some _time continuity_ and you want to provide a simple chronological browsing. For example: a movie catalog could allow the user to browse all the movies by year, or you may want to browse a long list of events by jumping and narrowing between years, months, days.

On the other hand it does not make much sense for the result of a search that hits just a few sparse records scattered over a possibly long period of time. In that case the calendar extra has an `:active` switch that can inactivate the calendar and fallback to the regular pagination. No need to maintain different UIs for wide browsing and narrow searching. 

## Synopsis

See [extras](../extras.md) for general usage info.

Require and configure it in the `pagy.rb` initializer:

```ruby
require 'pagy/extras/calendar'
# Optional: customize the default
```

Implement and use it in your controllers:

```ruby
# Define the pagy_calendar_period method in your application
def pagy_calendar_period(collection)
  return_period_array_using(collection)
end

# Define the pagy_calendar_filter method in your application  
def pagy_calendar_filter(collection, from, to)
  return_filtered_collection_using(collection, from, to)
end

# Use it in your actions:
@calendar, @pagy, @records = pagy_calendar(collection, year:  { size:  [1, 1, 1, 1], ... },
                                                       month: { size:  [0, 12, 12, 0], ... },
                                                       pagy:  { items: 10, ...})
```

Use the calendar and pagy objects in your views:

```erb
<!-- calendar pagination -->
<%== pagy_nav(@calendar[:year]) %>
<%== pagy_nav(@calendar[:month]) %>

<!-- pagy info extended for the calendar unit -->
<%== pagy_info(@pagy) %> for <%== @calendar[:month].label(format: '%B %Y') %>

... display @records ...

<!-- standard pagination of the selected month -->
<%== pagy_nav(@pagy) %>
```

**Notice** For a more complete and detailed example, see the [pagy_calendar_app.ru](https://github.com/ddnexus/pagy/blob/master/apps/pagy_calendar_app.ru).

## Usage

Since the time can be stored or calculated in many different ways in different collections, this extra requires you to implement a couple of simple methods in your app and configure the objects that it creates and coordinates for you.

The whole usage boils down to 3 steps:

1. Define the [pagy_calendar_period](#pagy_calendar_periodcollection) method in your controller
2. Define the [pagy_calendar_filter](#pagy_calendar_filtercollection-from-to) method in your controller
3. Configure the UI using the [pagy_calendar](#pagy_calendarcollection-conf) method in your action

You can play with a quick demo app, working without any additional configuration with:

```shell
git clone --depth 1 https://github.com/ddnexus/pagy
cd pagy
rackup -o 0.0.0.0 -p 8080 apps/pagy_calendar_app.ru
```

Then point your browser to `http://0.0.0.0:8080`.

## Variables and Accessors

See [Pagy::Calendar](../api/calendar.md#variables)

## Files

- [calendar.rb](https://github.com/ddnexus/pagy/blob/master/lib/pagy/extras/calendar.rb)

## Methods

All the methods in this module are prefixed with the `"pagy_calendar"` string in order to avoid any possible conflict with your own methods when you include the module in your controller. They are also all private, so they will not be available as actions.

### pagy_calendar(collection, conf)

This method is somehow similar to the generic `pagy` method (see the [pagy doc](../api/backend.md#pagycollection-varsnil)), but it handles multiple levels of pagy objects through the `conf` argument. Besides it returns an array with one more item than the usual two. E.g.: `@calendar, @pagy, @results = pagy_calendar(...)`.

### `collection`

The `pagy_calendar` method can handle any type of collection (e.g. `ActiveRecord`, `ElasticSearchRails`, `Searchkick`, `Meilisearch`, ...) with any of the pagy backend extras (e.g. builtin backend, `arel`, `array`, `countless`, `elasticsearch_rails`, `searchkick`, `meilisearch`, ...).

You can configure the backend extra method that will handle the `collection` by setting an optional `:backend` variable in the [Pagy conf](#pagy-conf).

### `conf` argument

The `conf` argument must be a Hash structure with the keys representing the type of configuration and the values being the Hash of vars that you want to pass to the creation of the specific object (or a `boolean` for the [Active conf](#active-conf)). 

The `conf` hash can be composed by the following type of configurations:

#### Calendar conf

The calendar configuration determines the time objects generated for the calendar. These are used for filtering the collection to the selected time units.

You can add one or more levels with keys like `:year`, `:month`, `:week`, `:day`. Each key must be set to the hash of the variables that will be used to initialize the relative `Pagy::Calendar::*` object. Use an empty hash for default values. E.g.: `year: {}, month: {}`.

**Restrictions**: The `:page`, `:page_param`, `:params` and `:period` variables for the calendar objects are managed automatically by the extra. Setting them explicitly has no effect. (See also [Calendar params](#calendar-params) for solutions in case of conflicts)

#### Pagy conf

This is the optional configuration of the final pagination object which is always used regardless the value of the [Active conf](#active-conf).

You can pass one optional `:pagy` key, set to the hash of variables to initialize a `Pagy` object. It has none of the restriction mentioned in the [Calendar conf](#calendar-conf).

Besides the usual pagy variables, you can add a `:backend` variable, set to the name of the backend extra method that you want to use for managing the collection. E.g.: `pagy: { backend: :pagy_searchkick, items: 10, ... }`. Notice that the `collection` argument must be exactly what you would pass to the backend method.

If the `:pagy` key/value is omitted, a default `Pagy` instance will be created by the default `:pagy` backend method.

#### Active conf

The calendar is active by default, however you can add an optional `:active` boolean flag to the `conf` hash in order to switch it ON or OFF, depending on its usefulness in different conditions (see the [Use cases](#use-cases)).

Take a look at the [pagy_calendar_app.ru](https://github.com/ddnexus/pagy/blob/master/apps/pagy_calendar_app.ru) for a simple example of a manual toggle in the UI.

### pagy_calendar_period(collection)

**This method must be implemented by the application.**

It receives a `collection` argument that must not be changed by this method, but can be used to return the starting and ending local Time objects array defining the calendar `:period`. See the [Pagy::Calendar Variables](../api/calendar.md#variables) for details.

Depending on the type of storage, the `collection` argument can contain different kind of objects:

#### ActiveRecord managed storage

If you use `ActiveRecord` the `collection` is going to be an `ActiveRecord::Relation` object. You can chain it to whatever other scope you need in order to get the starting and ending local `Time` objects array to return. Here are a few examples with the `created_at` field (but you can pull the time from anywhere):

```ruby
# Simpler version (2 queries)
def pagy_calendar_period(collection)
  starting = collection.minimum('created_at')
  ending   = collection.maximum('created_at') 
  [starting.in_time_zone.to_time, ending.in_time_zone.to_time]
end

# Faster version with manual pluck (1 query)
def pagy_calendar_period(collection)
  minmax = collection.pluck('MIN(created_at)', 'MAX(created_at)').first
  minmax.map { |time| Time.parse(time).in_time_zone.to_time }
end

# Fastest version (no queries)
# If you have the time range in the request (from UI selectors), 
# filter the collection by the param before passing it to `pagy_calendar`. 
# In this example you just use the :starting and :ending params to return the period
def pagy_calendar_period(collection)
  params.fetch_values(:starting, :ending).map { |time| Time.parse(time).in_time_zone.to_time }
end
```

See also [Time conversion](../api/calendar.md#time-conversions) for details.

#### Search frameworks storage

_If you use `ElasticSearchRails`, `Searchkick`, `Meilisearch` the `collection` argument is just the Array of the captured search arguments that you passed to the `Model.pagy_search` method. That array is what pagy uses internally to setup its variables before passing it to the standard `Model.search` method to do the actual search._

So you should use what you need from the `collection` array and do your own `Model.search(...)` in order to get the starting and ending local `Time` objects array to return.

### pagy_calendar_filter(collection, from, to)

**This method must be implemented by the application.**

It receives the main `collection` and must return a filtered version of it using the `from` and `to` **local Time** objects.

**IMPORTANT**: The logic for querying your collection shown in the following pseudo code is extremely important to get the right records. You should adapt it to your storage time zone and syntax:
 
```
storage_time >= from && storage_time < to
```

Depending on the type of storage, the `collection` argument can contain a different kind of object:

#### ActiveRecord managed storage

If you use `ActiveRecord` the `collection` is going to be an `ActiveRecord::Relation` object. You can chain it to whatever other scope you need in order to return a new scope, filtered with the logic indicated above. Here is an example with the `created_at` field again (but you can use anything, of course):

```ruby
def pagy_calendar_filter(collection, from, to)
  collection.where(created_at: from...to)  # 3-dots range excluding the end value
end
```

See also [Time conversion](../api/calendar.md#time-conversions) for details.

#### Search frameworks storage

_If you use `ElasticSearchRails`, `Searchkick`, `Meilisearch` the `collection` argument is just the Array of the captured search arguments that you passed to the `Model.pagy_search` method. That array is what pagy uses internally to setup its variables before passing it to the standard `Model.search` method to do the actual search._

So in order to filter the actual search with the `from` and `to` local `Time` objects, you should simply return the same array with the filtering added to its relevant item. Pagy will use it to do the actual (filtered) search. 

## Customization

### Order

If you set `:order` to `:desc`, you will get the page units in reversed order (e.g. May, then April, then March, ...), but keep in mind that you still have to reverse the records in the collection since pagy has no control over that (indeed it's your own collection scope).

### Offset

If you use the `:week` time unit, consider that the first day of the week is different for different locales, so you should adjust it by setting the `:offset` to the day offset from Sunday (0: Sunday; 1: Monday;... 6: Saturday).

### Calendar params

This extra handles the request params of its objects automatically, and you should not need to customize them unless they clash with other params in your requests. In that case you have a couple of alternatives:

- Renaming the clashing param of your app
- Passing a custom `:page_param` to the [pagy conf](#pagy-conf). That will internally rename the `:page_param` vars and update the `:params` procs of all the calendar objects accordingly.

## View

You can use the calendar objects with any `pagy_*nav` and `pagy_*nav_js` helpers or any templates in your views.

The `pagy_*combo_nav_js` keeps into account only page numbers and not labels, so it is not very useful (if at all) with `Pagy::Calendar::*` objects.

### Label format

Each page link in the calendar navs is conveniently labeled with the specific `Time` period it refers to. You can change the time format to your needs by just setting any of the `:format` variables with a standard `strftime` format. (See the [Pagy::Calendar variables](../api/calendar.md#variables))

You can also get the [current page label](../api/calendar.md#labelopts) with e.g.: `@calendar[:month].label`, which might be useful to use in your UI.

### I18n localization

Pagy implements its own faster version of the i18n `translate` (i.e. `pagy_t`) method, but does not provide any built-in support for the i18n `localize` method. If you need localization of calendar labels in other locales you should delegate it to the `I18n` gem, so that a change in the global `I18n.locale` will automatically localize all the time labels accordingly. 

You have a couple of options:

- Use the [i18n extra](i18n.md), which delegates the translation and localization to the `I18n` gem. Notice however that you would lose the performance gain offered by the built-in `pagy_t` translation.
- Uncomment the block in the calendar section in the [pagy.rb](https://github.com/ddnexus/pagy/blob/master/lib/config/pagy.rb) initializer, which will add the localization from the `I18n` gem without using the [i18n extra](../extras/i18n.md), so preserving the builtin `pagy_t` translation.

## Caveats

- Calendar pages with no records are currently shown as empty: you may want to display some message when `@records.empty?`.
- Skipping the empty pages in the nav is not possible, besides it would not look as a calendar anymore. However there is an experimental branch with a partial implementation of automatic disabling of the nav links of empty pages, which provides a better feedback. That, and other related improvements, may make it to master in the future.
- Currently you can use the [Active conf](#active-conf) to programmatically skip the calendar when it's not convenient, or add a link to the UI to allow manual toggling (see the [pagy_calendar_app.ru](https://github.com/ddnexus/pagy/blob/master/apps/pagy_calendar_app.ru) for an example of toggle link).