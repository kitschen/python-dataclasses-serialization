# dataclasses_serialization

`dataclasses_serialization` provides serializers/deserializers for transforming between Python dataclasses, and JSON and BSON objects.

## Basic Usage

Suppose we have the following dataclass:

```python
from dataclasses import dataclass


@dataclass
class InventoryItem:
    name: str
    unit_price: float
    quantity_on_hand: int
```

Then we may serialize/deserialize it to/from JSON by using `JSONSerializer`

```pycon
>>> from dataclasses_serialization.json import JSONSerializer
>>> JSONSerializer.serialize(InventoryItem("Apple", 0.2, 20))
{'name': 'Apple', 'unit_price': 0.2, 'quantity_on_hand': 20}

>>> JSONSerializer.deserialize(InventoryItem, {'name': 'Apple', 'unit_price': 0.2, 'quantity_on_hand': 20})
InventoryItem(name='Apple', unit_price=0.2, quantity_on_hand=20)
```

### Mongo

As Mongo collections store objects as BSON, you can use `BSONSerializer` to dump dataclasses directly into Mongo.

```python
from dataclasses_serialization.bson import BSONSerializer

    
collection.insert_one(BSONSerializer.serialize(item))

item = BSONSerializer.deserialize(InventoryItem, collection.find_one())
```

## Custom Serializers

To create a custom serializer, create an instance of `dataclasses_serialization.serializer_base.Serializer`:

```python
from dataclasses import dataclass, asdict
from dataclasses_serialization.serializer_base import Serializer, noop_serialization, noop_deserialization, dict_to_dataclass

from toolz import valmap


JSONSerializer = Serializer(
    serialization_functions={
        dataclass: lambda datacls: JSONSerializer.serialize(asdict(datacls)),
        dict: lambda dct: valmap(JSONSerializer.serialize, dct),
        list: lambda lst: list(map(JSONSerializer.serialize, lst)),
        (str, int, float, bool, type(None)): noop_serialization
    },
    deserialization_functions={
        dataclass: lambda cls, serialized_obj: dict_to_dataclass(cls, serialized_obj, JSONSerializer.deserialize),
        (dict, list, str, int, float, bool, type(None)): noop_deserialization
    }
)
```

## Reference

### `dataclasses_serialization.serializer_base`

A collection of utilities to make it easier to create serializers.

- `isinstance(o, t)`, `issubclass(cls, clsinfo)`

  Extended versions of the builtin `isinstance` and `issubclass`, to treat `dataclass` as a metaclass for dataclasses, and to be usable with supported `typing` types.

- `noop_serialization(obj)`, `noop_deserialization(cls, obj)`

  The trivial serialization/deserialization functions, which serialize by doing nothing.

- `dict_to_dataclass(cls, dct, deserialization_func=noop_deserialization)`

  The inverse of `dataclasses.asdict`, which deserializes a dictionary `dct` to a dataclass `cls`, using `deserialization_func` to deserialize the fields of `cls`.

- `union_deserialization(type_, obj, deserialization_func=noop_deserialization)`

  Deserialize a `Union` `type_`, by trying each type in turn, and returning the first that does not raise a `DeserializationError`.

  As `Optional`s are implemented as `Union`s, this function also works for them.

- `Serializer(serialization_functions, deserialization_functions)`

  The general serialization class.

  Takes two dictionaries of serialization and deserialization functions, and defers to them appropriately when serializing/deserializing and object by the `serialize` and `deserialize` methods.
  Serializer functions take a single parameter, the object to be serialized, and returns a serialized version of it.
  Deserializer functions take two parameters, the desired type of the deserialized object, and the object to be deserialized.

  By default, `Union`s are deserialized using `union_deserialization`.

  Serialize a Python object with `serializer.serialize(obj)`, and deserialize with `serializer.deserialize(cls, serialized_obj)`.

- `SerializationError`, `DeserializationError`

  Errors to be raised when serialization/deserialization fails, respectively.

### `dataclasses_serialization.json`

- `JSONSerializer`

  Serializer/deserializer between Python dataclasses and JSON objects.

  ```pycon
  >>> JSONSerializer.serialize(InventoryItem("Apple", 0.2, 20))
  {'name': 'Apple', 'unit_price': 0.2, 'quantity_on_hand': 20}

  >>> JSONSerializer.deserialize(InventoryItem, {'name': 'Apple', 'unit_price': 0.2, 'quantity_on_hand': 20})
  InventoryItem(name='Apple', unit_price=0.2, quantity_on_hand=20)
  ```

- `JSONSerializerMixin`

  Adds `as_json` and `from_json` methods to dataclasses when used as a mixin.

  ```python
  @dataclass
  class InventoryItem(JSONSerializerMixin):
      ...
  ```

  ```pycon
  >>> InventoryItem("Apple", 0.2, 20).as_json()
  {'name': 'Apple', 'unit_price': 0.2, 'quantity_on_hand': 20}

  >>> InventoryItem.from_json({'name': 'Apple', 'unit_price': 0.2, 'quantity_on_hand': 20})
  InventoryItem(name='Apple', unit_price=0.2, quantity_on_hand=20)
  ```

- `JSONStrSerializer`

  Serializer/deserializer between Python dataclasses and JSON strings.

  ```pycon
  >>> JSONStrSerializer.serialize(InventoryItem("Apple", 0.2, 20))
  '{"name": "Apple", "unit_price": 0.2, "quantity_on_hand": 20}'

  >>> JSONStrSerializer.deserialize(InventoryItem, '{"name": "Apple", "unit_price": 0.2, "quantity_on_hand": 20}')
  InventoryItem(name='Apple', unit_price=0.2, quantity_on_hand=20)
  ```

- `JSONStrSerializerMixin`

  Adds `as_json_str` and `from_json_str` methods to dataclasses when used as a mixin.

  ```python
  @dataclass
  class InventoryItem(JSONStrSerializerMixin):
      ...
  ```

  ```pycon
  >>> InventoryItem("Apple", 0.2, 20).as_json_str()
  '{"name": "Apple", "unit_price": 0.2, "quantity_on_hand": 20}'

  >>> InventoryItem.from_json_str('{"name": "Apple", "unit_price": 0.2, "quantity_on_hand": 20}')
  InventoryItem(name='Apple', unit_price=0.2, quantity_on_hand=20)
  ```

### `dataclasses_serialization.bson`

- `BSONSerializer`

  Serializer/deserializer between Python dataclasses and BSON objects.

  ```pycon
  >>> BSONSerializer.serialize(InventoryItem("Apple", 0.2, 20))
  {'name': 'Apple', 'unit_price': 0.2, 'quantity_on_hand': 20}

  >>> BSONSerializer.deserialize(InventoryItem, {'name': 'Apple', 'unit_price': 0.2, 'quantity_on_hand': 20})
  InventoryItem(name='Apple', unit_price=0.2, quantity_on_hand=20)
  ```

- `BSONSerializerMixin`

  Adds `as_bson` and `from_bson` methods to dataclasses when used as a mixin.

  ```python
  @dataclass
  class InventoryItem(BSONSerializerMixin):
      ...
  ```

  ```pycon
  >>> InventoryItem("Apple", 0.2, 20).as_bson()
  {'name': 'Apple', 'unit_price': 0.2, 'quantity_on_hand': 20}

  >>> InventoryItem.from_bson({'name': 'Apple', 'unit_price': 0.2, 'quantity_on_hand': 20})
  InventoryItem(name='Apple', unit_price=0.2, quantity_on_hand=20)
  ```

- `BSONStrSerializer`

  Serializer/deserializer between Python dataclasses and binary BSON strings.

- `BSONStrSerializerMixin`

  Adds `as_bson_str` and `from_bson_str` methods to dataclasses when used as a mixin.

  ```python
  @dataclass
  class InventoryItem(BSONStrSerializerMixin):
      ...
  ```

## Installation

Install and update using the standard Python package manager [pip](https://pip.pypa.io/en/stable/):

```bash
pip install dataclasses_serialization
```
