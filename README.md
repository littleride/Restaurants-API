# Restaurants-API


## Menu Update
This API allows you to update various entities in the Little Africa menu, including items, modifier items, modifier groups, and categories.

### Base URL

https://api.little.africa/deliveries/menu/v2/store/

## Update Context Types

The `context_type` parameter is used to specify the type of entity being updated. The possible values are:

- `items`
- `modifier_items`
- `modifier_groups`
- `categories`

### Context Type: `items`

To modify the items with the specified data, set `context_type` to `items`.

### Context Type: `modifier_items`

To modify the modifier items (add-ons) with the specified data, set `context_type` to `modifier_items`.

#### Sample Request for Modifier Items

```
{
    "context_type": "modifier_items",
    "id": "Q1R2S3T9",
    "details": {
        "modifier_name": "Mocha",
        "modifier_description": "Mocha"
    },
    "price_info": {
        "modifier_price": 10,
        "modifier_special_price": 0
    }
}
```
Context Type: modifier_groups
To update modifier groups, set context_type to modifier_groups.

Sample Request for Modifier Groups
```
{
    "context_type": "modifier_groups",
    "id": "M1N2O3P9",
    "details": {
        "title": "New group title 3"
    },
    "quantity_info": {
        "type_of_selection": "ONE",
        "max_items_permitted_per_order": 1,
        "required": 1
    }
}
```

Note: required value 1 means true, 0 means false.
Context Type: categories
To update categories, set context_type to categories.

Sample Request for Categories


```
{
    "context_type": "categories",
    "id": "18203d77",
    "details": {
        "title": "Cat_po"
    },
    "quantity_info": {
        "max_items_permitted_per_order": 50
    }
}

```


Additional Information

Ensure that the id provided corresponds to the entity you wish to update.
The details object contains metadata specific to the entity type.
The quantity_info object, when present, specifies selection rules for the entity.



