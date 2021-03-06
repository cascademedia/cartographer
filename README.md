Cartographer
============
Cartographer is a php library that provides the ability to transform data from one type into another type.

[![Build Status](https://scrutinizer-ci.com/g/cascademedia/cartographer/badges/build.png?b=master)](https://scrutinizer-ci.com/g/cascademedia/cartographer/build-status/master)
[![Code Coverage](https://scrutinizer-ci.com/g/cascademedia/cartographer/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/cascademedia/cartographer/?branch=master)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/cascademedia/cartographer/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/cascademedia/cartographer/?branch=master)

Installation
============
To install Cartographer, simply require the library by executing the following composer command.

```
$ composer require cascademedia/cartographer @stable
```

Alternatively, you can clone/download this repository and install the package manually.

Basic Usage
===========
There are three basic parts to the library: the mapper, mappings, and contexts.

- A context object exists to contain rules for an individual mapping use-case, for example: mapping user input to a
data object or mapping an entity to a view model.
- A mapping object is effectively a rule that determines how data is mapped from one field to another within a given
context.
- Finally, the mapper ties it all together and ensures that the proper mappings are executed from the
source data to the destination.

Without further introduction, here is a simple example that maps from an input array to a User class. For more advanced
usage, feel free to browse around the documentation.

```php
class User
{
    public $firstName;

    public $lastName;
}

class MyContext implements ContextInterface
{
    public function getMap()
    {
        return (new MapBuilder())
            ->from(MapBuilder::REF_ARRAY)
            ->to(MapBuilder::REF_CLASS_PROPERTIES)
            ->add('firstName', 'first_name')
            ->add('lastName', 'last_name')
            ->getMap()
        ;
    }
}

$source = [
    'first_name' => 'Test First',
    'last_name' => 'Test Last'
];

$destination = new User();

$mapper = new Mapper();
$context = new MyContext();

$destination = $mapper->map($destination, $source, $context);
var_dump($destination);
/*
class User#2 (2) {
  public $firstName =>
  string(10) "Test First"
  public $lastName =>
  string(9) "Test Last"
}
*/
```

Contexts
========
Contexts are classes that contain all of the rules used for an individual ```Mapper::map()``` operation. When using the
mapper, it is required to contain mapping rules within a context.

To create a context, create a class that implements ```ContextInterface``` and use ```getMap()``` to return a map
containing all mapping rules desired.

```php
class MyContext implements ContextInterface
{
    public function getMap()
    {
        return (new MapBuilder())
            ->from(MapBuilder::REF_ARRAY)
            ->to(MapBuilder::REF_ARRAY)
            ->add('first_name', 'fname')
            ->add('last_name', 'lname')
            ->getMap()
        ;
    }
}

$source = [
    'fname' => 'Test First',
    'lname' => 'Test Last'
];

$mapper = new Mapper();

$result = $mapper->map([], $source, new MyContext());
var_dump($result);
/*
array(2) {
  'first_name' =>
  string(10) "Test First"
  'last_name' =>
  string(9) "Test Last"
}
*/
```

References
==========
References are classes used by the mapper to to retrieve or store field data in an object or array. The mapper currently
supports array, object mutator, and object property references.

Reference classes are not typically used directly unless you are manually creating field mappings.

Array References
----------------
The ```ArrayReference``` class tells the mapper that you wish to access data contained within the top level of an
array.

The ```getValue()``` method allows users to retrieve data from any given array.
```php
$reference = new ArrayReference('first_name');

$data = [
    'first_name' => 'Test First',
    'last_name' => 'Test Last'
];

var_dump($reference->getValue($data));
// string(10) "Test First"
```

The ```setValue()``` method allows users to put data into any given array. Note that this method returns a copy of the
modified array and does not modify the original array passed into it.
```php
var_dump($reference->setValue($data, 'Another Test First'));
/*
array(2) {
  'first_name' =>
  string(18) "Another Test First"
  'last_name' =>
  string(9) "Test Last"
}
*/
```

Mutator References
------------------
The ```MutatorReference``` class tells the mapper that you wish to access data returned from a class' method call. By
default, this reference will attempt to use getters and setters of the named field. For example, referencing a
field named ```test``` will call ```getTest()``` and ```setTest()``` respectively, but can be configured to call other
methods if necessary.

Note that only public methods can be accessed. Accessing private and protected methods will result in a
```ReflectionException``` being thrown.

The ```getValue()``` method will call the configured getter method for the given object and return its result.
```php
class User
{
    private $firstName;

    private $lastName;

    public function getFirstName()
    {
        return $this->firstName;
    }

    public function setFirstName($firstName)
    {
        $this->firstName = $firstName;
        return $this;
    }

    public function getLastName()
    {
        return $this->lastName;
    }

    public function setLastName($lastName)
    {
        $this->lastName = $lastName;
        return $this;
    }
}

$reference = new MutatorReference('first_name');

$user = (new User())
    ->setFirstName('Test First')
    ->setLastName('Test Last')
;

var_dump($reference->getValue($user));
// string(10) "Test First"
```

The ```setValue()``` method will call the configured setter method for the given object.
```php
var_dump($reference->setValue($user, 'Another Test First'));
/*
class User#3 (2) {
  private $firstName =>
  string(18) "Another Test First"
  private $lastName =>
  string(9) "Test Last"
}
*/
```

If the default getters and setters are not satisfactory, you can change the methods that are called via the reference's
constructor.
```php
class User
{
    private $firstName;

    public function retrieveFirstName()
    {
        return $this->firstName;
    }

    public function addFirstName($firstName)
    {
        $this->firstName = $firstName;
        return $this;
    }
}

$reference = new MutatorReference('first_name', 'retrieveFirstName', 'addFirstName');

$user = (new User())
    ->addFirstName('Test First')
;

// calls $user->addFirstName('Changed First')
$reference->setValue($user, 'Changed First');

// calls $user->retrieveFirstName()
$value = $reference->getValue($user);

var_dump($value);
// string(13) "Changed First"
```

Property References
-------------------
The ```PropertyReference``` class tells the mapper that you wish to access data contained within a class' public
property. Note that only public properties can be accessed. Accessing private and protected properties will result in
a ```ReflectionException``` being thrown.

The ```getValue()``` method will return the data contained in the referenced object property.
```php
class User
{
    public $firstName;

    public $lastName;
}

$reference = new PropertyReference('firstName');

$user = new User();
$user->firstName = 'Test First';
$user->lastName = 'Test Last';

var_dump($reference->getValue($user));
// string(10) "Test First"
```

The ```setValue()``` method will put data into the referenced object property.
```php
var_dump($reference->setValue($user, 'Another Test First'));
/*
class User#3 (2) {
  public $firstName =>
  string(18) "Another Test First"
  public $lastName =>
  string(9) "Test Last"
}
*/
```

Mappings
========
Mapping classes are the workhorse of the mapper library. They are the classes that map individual pieces of data using
references to determine where the data comes from and where it goes.

Mapping
-------
The ```Mapping``` class is the most straightforward of all mappings. It simply takes the source and maps it directly to
the configured destination. When constructing the ```Mapping``` class, the destination reference comes first followed
by the source reference.

```php
$source = [
    'fname' => 'First',
    'lname' => 'Last'
];

$destination = [];

$mapping = new Mapping(new ArrayReference('first_name'), new ArrayReference('fname'));

$destination = $mapping->map($destination, $source);
var_dump($destination);
/*
array(1) {
  'first_name' =>
  string(5) "First"
}
*/
```

Embedded Mapping
----------------
The ```EmbeddedMapping``` class allows for mapping embedded data structures to the destination. When constructing the
```EmbeddedMapping``` class, the source field comes first followed by a map describing how the embedded structure will
be mapped to the destination.

```php
$source = [
    'name' => [
        'fname' => 'First',
        'lname' => 'Last'
    ]
];

$destination = [];

$mapping = new EmbeddedMapping(new ArrayReference('name'), (new MapBuilder())
    ->from(MapBuilder::REF_ARRAY)
    ->to(MapBuilder::REF_ARRAY)
    ->add('first_name', 'fname')
    ->add('last_name', 'lname')
    ->getMap()
);

$destination = $mapping->map($destination, $source);
var_dump($destination);
/*
array(2) {
  'first_name' =>
  string(5) "First"
  'last_name' =>
  string(4) "Last"
}
*/
```

Resolver Mapping
----------------
The ```ResolverMapping``` class allows for the use of [Value Resolvers](#user-content-value-resolvers) to map data to a
destination.

```php
class FullNameResolver implements ValueResolverInterface
{
    public function resolve($source, $destination)
    {
        return $source['fname'] . ' ' . $source['lname'];
    }
}

$source = [
    'fname' => 'First',
    'lname' => 'Last'
];

$destination = [];

$mapping = new ResolverMapping(new ArrayReference('name'), new FullNameResolver());

$destination = $mapping->map($destination, $source);
var_dump($destination);
/*
array(1) {
  'name' =>
  string(10) "First Last"
}
*/
```

Value Resolvers
===============
A value resolver is an object that will take the entire source object and return an individual value from it. Using a
value resolver allows for arbitrary logic in order to map a value.

```php
class FullNameResolver implements ValueResolverInterface
{
    public function resolve($source, $destination)
    {
        $firstName = $destination['fname'];
    
        if (array_key_exists('fname', $source)) {
            $firstName = $source['fname'];
        }
    
        return $firstName . ' ' . $source['lname'];
    }
}

$destination = [
    'fname' => '1First'
];

$source = [
    'lname' => 'Last'
];

$resolver = new FullNameResolver();
$result = $resolver->resolve($source, $destination);
var_dump($result);
// string(11) "1First Last"
```

Callable Value Resolver
-----------------------
The library comes with a flexible ```CallableValueResolver``` class out-of-the-box, allowing value resolvers to be
defined without the need to define resolver classes.

```php
$destination = [
    'fname' => '1First'
];

$source = [
    'lname' => 'Last'
];

$resolver = new CallableResolver(function ($source, $destination) {
    $firstName = $destination['fname'];

    if (array_key_exists('fname', $source)) {
        $firstName = $source['fname'];
    }

    return $firstName . ' ' . $source['lname'];
});

$result = $resolver->resolve($source, $destination);
var_dump($result);
// string(11) "1First Last"
```

The Map Builder
===============
To ease the creation of maps within a context, the library comes with a ```MapBuilder``` class that provides a simple,
fluent interface to build a map. The ```MapBuilder``` is built to solve most use cases so that the user should rarely
have to manually create mappings.

Defining default reference types
--------------------------------
The ```MapBuilder``` class allows you to designate the default [reference types](#user-content-references) using the
```from()``` and ```to()``` methods. These methods accept the ```MapBuilder``` constants ```REF_ARRAY```,
```REF_CLASS_PROPERTIES```, and ```REF_CLASS_MUTATORS``` respectively. If any values other than the previously mentioned
constants are used, an ```InvalidReferenceTypeException``` is thrown.

If the ```from()``` and ```to()``` methods are not called, then the ```REF_ARRAY``` reference type is assumed for both.

```php
(new MapBuilder())
    ->from(MapBuilder::REF_ARRAY)
    ->to(Mapbuilder::REF_CLASS_MUTATORS)
;
```

Adding Mappings
---------------
There are methods on the ```MapBuilder``` class that allows for the addition of pre-built mapping types, as well
as the ability to add custom mappings.

The ```add()``` method simply creates a new [Mapping](#user-content-mapping) using the determined reference types.
```php
(new MapBuilder())
    ->add('first_name', 'fname')
;

// add() creates a mapping equivalent to:

new Mapping(new ArrayReference('first_name'), new ArrayReference('fname'))
```

The ```addEmbedded()``` method creates an [Embedded Mapping](#user-content-embedded-mapping) using the determined
reference types.
```php
(new MapBuilder())
    ->addEmbedded('name', (new MapBuilder())
        ->add('first_name', 'fname')
        ->add('last_name', 'lname')
        ->getMap()
    )
;

// addEmbedded() creates an embedded mapping equivalent to:

new EmbeddedMapping(new ArrayReference('name'), (new MapBuilder())
    ->add('first_name', 'fname')
    ->add('last_name', 'lname')
    ->getMap()
)
```

The ```addResolver()``` method creates a [Resolver Mapping](#user-content-resolver-mapping) using the determined
reference types.
```php
(new MapBuilder())
    ->addResolver('full_name', new FullNameResolver())
;

// addResolver() creates a mapping equivalent to:

new ResolverMapping(new ArrayReference('full_name'), new FullNameResolver())
```

Finally, the ```addMapping``` method simply adds a custom user-defined mapping.
```php
(new MapBuilder())
    ->addMapping(new Mapping(new ArrayReference('first_name'), new ArrayReference('fname')))
;
```

Building the Map
----------------
Once all mappings have been added into the ```MapBuilder```, it's time to generate a ```Map``` that can be used within
a context. The ```getMap()``` method generates a new ```Map``` for our uses.

```php
(new MapBuilder())
    ->add('first_name', 'fname')
    ->add('last_name', 'lname')
    ->getMap()
;

// is equivalent to:

new Map([
    new Mapping(new ArrayReference('first_name'), new ArrayReference('fname')),
    new Mapping(new ArrayReference('last_name'), new ArrayReference('lname'))
]);

```

Want to contribute?
===================
If you would like to contribute to this library, you can do so in a couple of ways:

- Submit issues for bugs or functionality that would improve the project.
- Submit a pull request for new functionality.
- Update any documentation that appears to be lacking.

When submitting pull requests, please make sure functionality is covered by unit tests, and that all of the tests pass.

License
=======
This library is MIT licensed, meaning it is free for anyone to use and modify.

```
The MIT License (MIT)

Copyright (c) 2015 Cascade Media, LLC.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
