# Dream

[![Latest Version](https://img.shields.io/badge/release-v1.0.2-blue.svg?style=flat-square)](https://github.com/bulwark1374/dream/releases)
[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE)
[![Build Status](https://travis-ci.org/bulwark1374/dream.svg?branch=master&style=flat-square)](https://travis-ci.org/bulwark1374/dream)
[![Coverage Status](https://coveralls.io/repos/github/bulwark1374/dream/badge.svg?branch=master)](https://coveralls.io/github/bulwark1374/dream)
[![Total Downloads](https://img.shields.io/badge/downloads-1-red.svg?style=flat-square)](https://packagist.org/packages/bulwark/dream)

## Introduction

A Laravel base controller class and a trait that will enable to add filtering, sorting, eager loading and pagination to your
resource URLs.

## Functionality

* Parse GET parameters for dynamic eager loading of related resources, sorting and pagination
* Advanced filtering of resources using filter groups
* Use [Bulwark\Architect](https://github.com/bulwark1374/architect) for sideloading, id loading or embedded loading of related resources
* ... [Ideas for new functionality is welcome here](https://github.com/bulwark1374/dream/issues/new)


For Laravel 5.4 and above
```bash
composer require bulwark/dream ~1.0
```

## Usage

The examples will be of a hypothetical resource endpoint `/books` which will return a collection of `Book`,
each belonging to a `Author`.

```
Book n ----- 1 Author
```

### Available query parameters

Key | Type | Description
--- | ---- | -----------
Includes | array | Array of related resources to load, e.g. ['author', 'publisher', 'publisher.books']
Sort | array | Property to sort by, e.g. 'title'
Limit | integer | Limit of resources to return
Page | integer | For use with limit
Filter_groups | array | Array of filter groups. See below for syntax.

### Implementation

```php
<?php

namespace App\Http\Controllers;

use Bulwark\Api\Controller\EloquentBuilderTrait;
use Bulwark\Api\Controller\LaravelController;
use App\Models\Book;

class BookController extends LaravelController
{
    use EloquentBuilderTrait;

    public function getBooks()
    {
        // Parse the resource options given by GET parameters
        $resourceOptions = $this->parseResourceOptions();

        // Start a new query for books using Eloquent query builder
        // (This would normally live somewhere else, e.g. in a Repository)
        $query = Book::query();
        $this->applyResourceOptions($query, $resourceOptions);
        $books = $query->get();

        // Parse the data using Optimus\Architect
        $parsedData = $this->parseData($books, $resourceOptions, 'books');

        // Create JSON response of parsed data
        return $this->response($parsedData);
    }
}
```

## Syntax documentation

### Eager loading

**Simple eager load**

`/books?includes[]=author`

Will return a collection of 5 `Book`s eager loaded with `Author`.

**IDs mode**

`/books?includes[]=author:ids`

Will return a collection of `Book`s eager loaded with the ID of their `Author`

**Sideload mode**

`/books?includes[]=author:sideload`

Will return a collection of `Book`s and a eager loaded collection of their
`Author`s in the root scope.

[See mere about eager loading types in Optimus\Architect's README](https://github.com/bulwark1374/architect)

### Pagination

Two parameters are available: `limit` and `page`. `limit` will determine the number of
records per page and `page` will determine the current page.

`/books?limit=10&page=3`

Will return books number 30-40.

### Sorting

Should be defined as an array of sorting rules. They will be applied in the
order of which they are defined.

**Sorting rules**

Property | Value type | Description
-------- | ---------- | -----------
key | string | The property of the model to sort by
direction | ASC or DESC | Which direction to sort the property by

**Example**

```json
[
    {
        "key": "title",
        "direction": "ASC"
    }, {
        "key": "year",
        "direction": "DESC"
    }
]
```

Will result in the books being sorted by title in ascending order and then year
in descending order.

### Filtering

Should be defined as an array of filter groups.

**Filter groups**

Property | Value type | Description
-------- | ---------- | -----------
or | boolean | Should the filters in this group be grouped by logical OR or AND operator
filters | array | Array of filters (see syntax below)

**Filters**

Property | Value type | Description
-------- | ---------- | -----------
key | string | The property of the model to filter by (can also be custom filter)
value | mixed | The value to search for
operator | string | The filter operator to use (see different types below)
not | boolean | Negate the filter

**Operators**

Type | Description | Example
---- | ----------- | -------
ct | String contains | `dm` matches `Bulwark dream` and `dream`
sw | Starts with | `bl` matches `blah` but not `mombo blah`
ew | Ends with | `sh` matches `linux bash` but not `bash linux`
eq | Equals | `bash bash` matches `bash bash` but not `bash`
gt | Greater than | `1548` matches `1600` but not `1400`
gte| Greater than or equalTo | `1548` matches `1548` and above (ony for Laravel 5.4 and above)
lte | Lesser than or equalTo | `1600` matches `1600` and below (ony for Laravel 5.4 and above)
lt | Lesser than | `1600` matches `1548` but not `1700`
in | In array | `['me', 'you']` matches `you` and `me` but not `other`
bt | Between | `[1, 10]` matches `5` and `7` but not `11`

**Special values**

Value | Description
----- | -----------
null (string) | The property will be checked for NULL value
(empty string) | The property will be checked for NULL value

#### Custom filters

Remember our relationship `Books n ----- 1 Author`. Imagine your want to
filter books by `Author` name.

```json
[
    {
        "filters": [
            {
                "key": "author",
                "value": "Bulwark",
                "operator": "sw"
            }
        ]
    }
]
```

Now that is all good, however there is no `author` property on our
model since it is a relationship. This would cause an error since
Eloquent would try to use a where clause on the non-existant `author`
property. We can fix this by implementing a custom filter. Where
ever you are using the `EloquentBuilderTrait` implement a function named
`filterAuthor`

```php
public function filterAuthor(Builder $query, $method, $clauseOperator, $value)
{
    // if clauseOperator is idential to false,
    //     we are using a specific SQL method in its place (e.g. `in`, `between`)
    if ($clauseOperator === false) {
        call_user_func([$query, $method], 'authors.name', $value);
    } else {
        call_user_func([$query, $method], 'authors.name', $clauseOperator, $value);
    }
}
```

*Note:* It is important to note that a custom filter will look for a relationship with
the same name of the property. E.g. if trying to use a custom filter for a property
named `author` then `Dream` will try to eagerload the `author` relationship from the
`Book` model.

**Custom filter function**

Argument | Description
-------- | -----------
$query | The Eloquent query builder
$method | The where method to use (`where`, `orWhere`, `whereIn`, `orWhereIn` etc.)
$clauseOperator | Can operator to use for non-in wheres (`!=`, `=`, `>` etc.)
$value | The filter value
$in | Boolean indicating whether or not this is an in-array filter

#### Examples

```json
[
    {
        "or": true,
        "filters": [
            {
                "key": "author",
                "value": "Bulwark",
                "operator": "sw"
            },
            {
                "key": "author",
                "value": "Prime",
                "operator": "ew"
            }
        ]
    }
]
```

Books with authors whoose name start with `Bulwark` or ends with `Prime`.

```json
[
    {
        "filters": [
            {
                "key": "author",
                "value": "Brian",
                "operator": "sw"
            }
        ]
    },
    {
        "filters": [
            {
                "key": "year",
                "value": 1990,
                "operator": "gt"
            },
            {
                "key": "year",
                "value": 2000,
                "operator": "lt"
            }
        ]
    }
]
```

Books with authors whoose name start with Brian and which were published between years 1990 and 2000.

### Optional Shorthand Filtering Syntax (Shorthand)
Filters may be optionally expressed in Shorthand, which takes the a given filter
array[key, operator, value, not(optional)] and builds a verbose filter array upon evaluation.

For example, this group of filters (Verbose)
```json
[
    {
        "or": false,
        "filters": [
            {
                "key": "author",
                "value": "Bulwark",
                "operator": "sw"
            },
            {
                "key": "author",
                "value": "Prime",
                "operator": "ew"
            }
            {
                "key": "deleted_at",
                "value": null,
                "operator": "eq",
                "not": true
            }
        ]
    }
]
```
May be expressed in this manner (Shorthand)
```json
[
    {
        "or": false,
        "filters": [
            ["author", "sw", "Bulwark"],
            ["author", "ew", "Prime"],
            ["deleted_at", "eq", null, true]
        ]
    }
]
```


## Standards

This package is compliant with [PSR-1], [PSR-2] and [PSR-4]. If you notice compliance oversights,
please send a patch via pull request.

[PSR-1]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-1-basic-coding-standard.md
[PSR-2]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md
[PSR-4]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md

## Testing

``` bash
$ phpunit
```

## Contributing

Please see [CONTRIBUTING](https://github.com/bulwark1374/dream/blob/master/CONTRIBUTING.md) for details.

## License

The MIT License (MIT). Please see [License File](https://github.com/bulwark1374/dream/blob/master/LICENSE) for more information.
