---
title: "get"
_old_id: "1170"
_old_uri: "2.x/class-reference/xpdoobject/field-accessors/get"
---

## xPDOObject::get()

Get a field value (or a set of values) by the field key(s) or name(s).

## Syntax

API Docs: <http://api.modxcms.com/xpdo/om/xPDOObject.html#get>

``` php
mixed get(
   string|array $k,
   [string|array $format = null],
   [mixed $formatTemplate = null]
)
```

Do not use the $format parameter if retrieving multiple values of different types, as the format string will be applied to all types, most likely with unpredicatable results. Optionally, you can supply an associate array of format strings with the field key as the key for the format array.

## Examples

Get the name field value of the object.

``` php
$object->set('name','Charles');
$name = $object->get('name');
echo $name; // produces "Charles"
```

Get an array of values for multiple fields:

``` php
$object->set('name_first','Charles');
$object->set('name_last','Longbottom');
$names = $object->get(array('name_first','name_last'));
echo $names['name_first'].' '.$names['name_last'];
// produces "Charles Longbottom"
```