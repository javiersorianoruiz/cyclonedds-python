Working with idl
================

At the time of writing there is no official mapping from OMG IDL to Python. The solutions we came up with here are therefore not standardized and are thus not compatible with other DDS implementations. However, they are based purely on the standard library type-hinting functionality as introduced in Python 3.5, meaning that any Python tooling available that works with type hints is also compatible with our implementation. To generate initializers and nice string representations we use :py:mod:`dataclasses` standard library module. This is applied outside of the other IDL machinery, so if you want you can control immutability, equality checking or even use a different ``dataclasses`` representation, for example `runtype`_.

All idl type support is contained within the subpackage `cyclonedds.idl`, allowing you to use it even in contexts where you do not need the full |var-project| ecosystem.


Working with the IDL compiler
-----------------------------

You use the IDL compiler if you already have an idl file to define your types or if you require interoperability with non-Python projects. The ``idlpy`` library is built as part of the python package leveraging the scikit-build cmake integration. We will soon integrate a locator mechanism into ``idlc`` to retrieve the library location, since the ``idlpy`` library will be part of the python package installation and not on the `LD_LIBRARY_PATH` or equivalent. For now you can employ the same tactic as ``idlc`` will do eventually and use a hidden method:


.. code-block:: shell

   idlc -l py your_file.idl

If you wish to nest the resulting Python module inside an existing package you can specify the path from the intended root. So if you have a package 'wubble' with a submodule 'fruzzy' and want the generated modules and types under there you can pass ``py-root-prefix``:

.. code-block:: shell

   idlc -l py -p py-root-prefix=wubble.fruzzy your_file.idl


.. _datatypes:

IDL Datatypes in Python
-----------------------

The ``cyclonedds.idl`` package implements defining IDL unions and structs and their OMG XCDR-V1 encoding in pure python. There should almost never be a need to delve into the details of this package when using DDS. In most cases the IDL compiler will write the code that uses this package. However, it is possible to write the objects manually. This is of course only useful if you do not plan to interact with other language clients, but makes perfect sense in a python-only project. If you are manually writing IDL objects your most important tool is :class:`IdlStruct<cyclonedds.idl.IdlStruct>`.

The following basic example will be very familiar if you have used dataclasses before. We will go over it here again briefly, for more detail go to the standard library documentation of :mod:`dataclasses<python:dataclasses>`.

.. code-block:: python
   :linenos:

   from dataclasses import dataclass
   from cyclonedds.idl import IdlStruct

   @dataclass
   class Point2D(IdlStruct):
      x: int
      y: int

   p1 = Point2D(20, -12)
   p2 = Point2D(x=12, y=-20)
   p1.x += 5


As you can see the :func:`dataclass<python:dataclasses.dataclass>` decorator turns a class with just names and types into a dataclass. The :class:`IdlStruct<cyclonedds.idl.IdlStruct>` parent class will make use of the type information defined in the dataclass to :ref:`(de)serialize messages<serialization>`. All normal dataclasses functionality is preserved, so you can still use :func:`field<python:dataclasses.field>` from the dataclasses module to define default factories or add a `__post_init__` method for more complicated construction scenarios.

Types
-----

Not all types that are possible to write in Python are encodable with OMG XCDR-V1. This means that you are slightly limited in what you can put in an :class:`IdlStruct<cyclonedds.idl.IdlStruct>` class. An exhaustive list follows:

Integers
^^^^^^^^

The default python :class:`int<python:int>` type maps to a OMG XCDR-V1 64 bit integer. For most applications that should suffice, but the :mod:`types<cyclonedds.idl.types>` module has all the other integers types supported in python.

.. code-block:: python
   :linenos:

   from dataclasses import dataclass
   from cyclonedds.idl import IdlStruct
   from cyclonedds.idl.types import int8, uint8, int16, uint16, int32, uint32, int64, uint64

   @dataclass
   class SmallPoint2D(IdlStruct):
      x: int8
      y: int8

Note that these special types are just normal :class:`int<python:int>` s at runtime. They are only used to indicate the serialization functionality what type to use on the network. If you store a number that is not supported by that integer type you will get an error during encoding. The int128 and uint128 are not supported.

Floats
^^^^^^

The python :class:`float<python:float>` type maps to a 64 bit float, which would be a `double` in C-style languages. The :mod:`types<cyclonedds.idl.types>` module has a float32 and float64 type, float128 is not supported.

Strings
^^^^^^^

The python :class:`str<python:str>` type maps directly to the XCDR string. Under the hood it is encoded with utf-8. Inside :mod:`types<cyclonedds.idl.types>` there is the :class:`bounded_str<cyclonedds.idl.types.bounded_str>` type for a string with maximum length.


.. code-block:: python
   :linenos:

   from dataclasses import dataclass
   from cyclonedds.idl import IdlStruct
   from cyclonedds.idl.types import bounded_str

   @dataclass
   class Textual(IdlStruct):
      x: str
      y: bounded_str[20]


Lists
^^^^^

The python :func:`list<python:list>` is a versatile type. In normal python a list would be able to contain any other types, but to be able to encode it all of the contents must be the same type, and this type must be known beforehand. This can be achieved by using the :class:`sequence<cyclonedds.idl.types.sequence>` type.


.. code-block:: python
   :linenos:

   from dataclasses import dataclass
   from cyclonedds.idl import IdlStruct
   from cyclonedds.idl.types import sequence

   @dataclass
   class Names(IdlStruct):
      names: sequence[str]

   n = Names(names=["foo", "bar", "baz"])


In XCDR this will result in an 'unbounded sequence', which should be fine in most cases. However, you can switch over to a 'bounded sequence' or 'array' using annotations. This can be useful to either limit the maximum allowed number of items (bounded sequence) or if the length of the list is always the same (array).

.. code-block:: python
   :linenos:

   from dataclasses import dataclass
   from cyclonedds.idl import IdlStruct
   from cyclonedds.idl.types import sequence, array

   @dataclass
   class Numbers(IdlStruct):
      ThreeNumbers: array[int, 3]
      MaxFourNumbers: sequence[int, 4]


Dictionaries
^^^^^^^^^^^^

Currently dictionaries are not supported by the IDL compiler. However, if your project is pure python there is no problem in using them. Unlike a raw python :class:`dict<python:dict>` both the key and the value need to have a constant type. This is expressed using the :class:`Dict<python:typing.Dict>` from the :mod:`typing<python:typing>` module.

.. code-block:: python
   :linenos:

   from typing import Dict
   from dataclasses import dataclass
   from cyclonedds.idl import IdlStruct

   @dataclasses
   class ColourMap(IdlStruct):
      mapping: Dict[str, str]

   c = ColourMap({"red": "#ff0000", "blue": "#0000ff"})


Unions
^^^^^^

Unions in IDL are not like the Unions defined in the :mod:`typing<python:typing>` module. IDL unions are *discriminated*, meaning they have a value that indicates which of the possibilities is active. 

You can write discriminated unions using the :func:`@union<cyclonedds.idl.types.union>` decorator and the :func:`case<cyclonedds.idl.types.case>` and :func:`default<cyclonedds.idl.types.default>` helper types. You again write a class in a dataclass style, except only one of the values can be active at a time. The :func:`@union<cyclonedds.idl.types.union>` decorator takes one type as argument, which determines the type of what is differentiating the cases.

.. code-block:: python
   :linenos:

   from enum import Enum, auto
   from dataclasses import dataclass
   from cyclonedds.idl import IdlUnion, IdlStruct
   from cyclonedds.idl.types import uint8, union, case, default, MaxLen


   class Direction(Enum):
      North = auto()
      East = auto()
      South = auto()
      West = auto()


   class WalkInstruction(IdlUnion, discriminator=Direction):
      steps_n: case[Direction.North, int]
      steps_e: case[Direction.East, int]
      steps_s: case[Direction.South, int]
      steps_w: case[Direction.West, int]
      jumps: default[int]

   @dataclass
   class TreasureMap(IdlStruct):
      description: str
      steps: sequence[WalkInstruction, 20]


   map = TreasureMap(
      description="Find my Coins, Diamonds and other Riches!\nSigned\nCaptain Corsaro",
      steps=[
         WalkInstruction(steps_n=5),
         WalkInstruction(steps_e=3),
         WalkInstruction(jumps=1),
         WalkInstruction(steps_s=9)
      ]
   )

   print (map.steps[0].discriminator)  # You can always access the discriminator, which in this case would print 'Direction.North'


Objects
^^^^^^^

You can also reference other classes as member type. These other classes should be :class:`IdlStruct<cyclonedds.idl.IdlStruct>` or :class:`IdlUnion<cyclonedds.idl.IdlUnion>` classes and again only contain serializable members. 

.. code-block:: python
   :linenos:

   from dataclasses import dataclass
   from cyclonedds.idl import IdlStruct
   from cyclonedds.idl.types import sequence

   @dataclass
   class Point2D(IdlStruct):
      x: int
      y: int

   @dataclass
   class Cloud(IdlStruct):
      points: sequence[Point]

.. _Serialization:

Serialization
^^^^^^^^^^^^^

If you are using a DDS system you should not need this, serialization and deserialization happens automatically within the backend. However, for debug purposes or outside a DDS context it might be useful to look at the serialized data or create python objects from raw bytes. By inheriting from :class:`IdlStruct<cyclonedds.idl.IdlStruct>` or :class:`IdlUnion<cyclonedds.idl.IdlUnion>` the classes you define automatically gain ``instance.serialize() -> bytes`` and a ``cls.deserialize(data: bytes) -> cls``  functions. Serialize is a member function that will return :class:`bytes<python:bytes>` with the serialized object. Deserialize is a :func:`classmethod<python:classmethod>` that takes the :class:`bytes<python:bytes>` and returns the resultant object. You can also inspect the python builtin ``cls.__annotations__`` for the member types and the ``cls.__idl_annotations__`` and ``cls.__idl_field_annotations__`` for idl information.

.. code-block:: python
   :linenos:

   from dataclasses import dataclass
   from cyclonedds.idl import IdlStruct

   @dataclass
   class Point2D(IdlStruct):
      x: int
      y: int

   p = Point2D(10, 10)
   data = p.serialize()
   q = Point2D.deserialize(data)

   assert p == q


Idl Annotations
^^^^^^^^^^^^^^^

In IDL you can annotate structs and members with several different annotations, for example ``@key``. In python we have decorators, but they only apply to classes not to fields. This is the reason why the syntax in python for a class or field annotation differ slightly. As an aside, the IDL ``#pragma keylist`` is a class annotation in python, but functions in the exact same way.

.. code-block:: python
   :linenos:

   from dataclasses import dataclass
   from cyclonedds.idl import IdlStruct
   from cyclonedds.idl.annotations import key, keylist

   @dataclass
   class Type1(IdlStruct):
      id: int
      key(id)
      value: str

   @dataclass
   @keylist(["id"])
   class Type2(IdlStruct):
      id: int
      value: str


.. _runtype: https://pypi.org/project/runtype/