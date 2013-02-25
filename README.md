# Schema-Inspector

Schema-Inspector is powerful tools to validation or sanitize javascript object.
It is disigned to work both client-side and server-side. Although originally
designed for use with [node.js](http://nodejs.org), it can also be used directly
in the browser.

Schema-Inspector has to be very scalable, and allow asynchronous and synchronous
calls.

## Quick Examples

### Synchronous call

```javascript
	var si = require('schema-inspector');

	var schema = {
		type: 'object',
		properties: {
			lorem: { type: 'string', eq: 'ipsum' },
			dolor: {
				type: 'array',
				items: { type: 'number' }
			}
		}
	};

	var candidate = {
		lorem: 'not_ipsum',
		dolor: [ 12, 34, 'ERROR', 45, 'INVALID' ]
	};
	var result = SchemaInspector.validate(schema, candidate); // Candidate is not valid
	console.log(result.format());
	/*
		Property @.lorem: must be equal to "ipsum", but is equal to "not_ipsum"
		Property @.dolor[2]: must be number, but is string
		Property @.dolor[4]: must be number, but is string
	*/
```

### Asynchronous call

```javascript
	var si = require('schema-inspector');

	var schema = { ...	};

	var candidate = { ... };

	SchemaInspector.validate(schema, candidate, function (err, result) {
		console.log(result.format());
		/*
			Property @.lorem: must be equal to "ipsum", but is equal to "not_ipsum"
			Property @.dolor[2]: must be number, but is string
			Property @.dolor[4]: must be number, but is string
		*/
	});
```

### Custom fields

```javascript
	var si = require('schema-inspector');

	var schema = {
		type: 'array',
		items: { type: 'number', $divisibleBy: 5 }
	};

	var custom = {
		divisibleBy: function (schema, candidate) {
			var dvb = schema.$divisibleBy;16
			if (cndidate % dvb !== 0) {
				this.report('must be divisible by ' + dvb);
			}
		}
	};

	var c = [ 5, 10, 15, 16];
	si.validate(schema, candidate, custom); // Invalid: "@[3] must be divisible by 5"
```

## In the browser

```html
<script type="text/javascript" src="async.js"></script>
<script type="text/javascript" src="schema-inspetor.js"></script>

<script type="text/javascript">

	SchemaInspector.validate(schema, candidate, function (err, result) {
		alert(result.format());
	});

</script>
```

## Documentation

### Validation

* [type](#v_type)
* [optional](#v_optional)
* [pattern](#v_pattern)
* [minLength, maxLength, exactLength](#v_length)
* [lt, lte, gt, gte, eq, ne](#v_comparators)
* [someKeys](#v_someKeys)
* [strict](#v_strict)
* [exec](#v_exec)
* [properties](#v_properties)
* [items](#v_items)
* [alias](#v_alias)
* [error](#v_error)

### Sanitization

* [type](#s_type)
* [optional](#s_optional)
* [rules](#s_rules)
* [min, min](#s_comparators)
* [minLength, maxLength](#s_length)
* [exec](#s_exec)
* [properties](#s_properties)
* [items](#s_items)

## Validation

<a name="v_type" />
### type

* **type**: string, array of string.
* **usable on**: any.
* possible values
	* "string"
	* "number"
	* "integer"
	* "boolean"
	* "null"
	* "date" (constructor === Date)
	* "object" (constructor === Object)
	* "array" (constructor === Array)
	* "any" (it can be anything)

Allow to check property type. If the given value is incorrect, then type is not
checked.

__Example__

```javascript
	var si = require('schema-inspector');

	var schema = {
		type: 'object',
		properties: {
			lorem: {  type: 'number' },
			ipsum: { type: 'any' },
			dolor: { type: ['number' 'string', 'null'] }
		}
	};

	var c1 = {
		lorem: 12,
		ipsum: 'sit amet',
		dolor: 23
	};
	var c2 = {
		lorem: 12,
		ipsum: 34,
		dolor: 'sit amet'
	};
	var c3 = {
		lorem: 12,
		ipsum: [ 'sit amet' ],
		dolor: null
	};
	var c4 = {
		lorem: '12',
		ipsum: 'sit amet',
		dolor: new Date()
	};
	si.validate(schema, c1); // Valid
	si.validate(schema, c2); // Valid
	si.validate(schema, c3); // Valid
	si.validate(schema, c4); // Invalid: @.lorem must be a number, @dolor must be a number, a string or null

```

---------------------------------------

<a name="v_optional" />
### optional

* **type**: boolean.
* **default**: false.
* **usable on**: any.

This fields tell whether or not property has to exist.

__Example__

```javascript
	var si = require('schema-inspector');

	var schema1 = {
		type: 'object',
		properties: {
			lorem: { type: 'any', optional: true }
		}
	};

	var schema2 = {
		type: 'object',
		properties: {
			lorem: { type: 'any', optional: false } // default value
		}
	};

	var c1 = { lorem: 'ipsum' };
	var c2 = { };

	si.validate(schema1, c1); // Valid
	si.validate(schema1, c2); // Valid
	si.validate(schema2, c1); // Valid
	si.validate(schema2, c2); // Invalid: "@.lorem" is missing and not optional
```

---------------------------------------

<a name="v_uniqueness" />
### uniqueness

* **type**: boolean.
* **default**: false.
* **usable on**: array, string.

If true, then we ensure no element in candidate exists more than once.

__Example__

```javascript
	var si = require('schema-inspector');

	var schema = {
		type: 'array',
		uniqueness: true
	};

	var c1 = [12, 23, 34, 45];
	var c2 = [12, 23, 34, 12];

	si.validate(schema, c1); // Valid
	si.validate(schema, c2); // Invalid: 12 exists twice in @.
```

---------------------------------------

<a name="v_pattern" />
### pattern

* **type**: string, RegExp object, array of string and RegExp.
* **usable on**: string.
* Possible values as a string: `void`, `url`, `date-time`, `date`,
`coolDateTime`, `time`, `color`, `email`, `numeric`, `integer`, `decimal`,
`alpha`, `alphaNumeric`, `alphaDash`, `javascript`, `upperString`, `lowerString`.

Ask Schema-Inspector to check whether or not a given matches provided patterns.
When a pattern is a RegExp, it directly test the string with it. When it's a
string, it's an alias of a RegExp.

__Example__

```javascript
	var si = require('schema-inspector');

	var schema1 = {
		type: 'array',
		items: { type: 'string', pattern: /^[A-C]/ }
	};

	var c1 = ['Alorem', 'Bipsum', 'Cdolor', 'DSit amet'];

	var schema2 = {
		type: 'array',
		items: { type: 'string', pattern: 'email' }
	};

	var c2 = ['lorem@ipsum.com', 'dolor@sit.com', 'amet@consectetur'];

	si.validate(schema1, c1); // Invalid: @[3] ('DSit amet') does not match /^[A-C]/
	si.validate(schema2, c2); // Invalid: @[2] ('amet@consectetur') does not match "email" pattern.
```

---------------------------------------

<a name="v_length" />
### minLength, maxLength, exactLength

* **type**: integer.
* **usable on**: array, string.

__Example__

```javascript
	var si = require('schema-inspector');

	var schema = {
		type: 'object',
		properties: {
			lorem: { type: 'string', minLength: 4, maxLength: 8 },
			ipsum: { type: 'array', exactLength: 6 },
		}
	};
	var c1 = {
		lorem: '12345',
		ipsum: [1, 2, 3, 4, 5, 6]
	};

	var c2 = {
		lorem: '123456789',
		ipsum: [1, 2, 3, 4, 5]
	};

	si.validate(schema, c1); // Valid
	si.validate(schema, c2); // Invalid: @.lorem must have a length between 4 and 8 (here 9)
	// and @.ipsum must have a length of 6 (here 5)
```

---------------------------------------

<a name="v_comparators" />
### lt, lte, gt, gte, eq, ne

* **type**: number.
* **usable on**: number.

Check whether comparison is true:

* lt: `<`
* lte: `<=`
* gt: `>`
* gte: `>=`
* eq: `===`
* ne: `!==`

__Example__

```javascript
	var si = require('schema-inspector');

	var schema = {
		type: 'object',
		properties: {
			lorem: { type: 'number', gt: 0, lt: 5 }, // Between ]0; 5[
			ipsum: { type: 'number', gte: 0, lte: 5 }, // Between [0; 5]
			dolor: { type: 'number', eq: [0, 3, 6, 9] }, // Equal to 0, 3, 6 or 9
			sit: { type: 'number', ne: [0, 3, 6, 9] } // Not equal to 0, 3, 6 nor 9
		}
	};

	var c1 = { lorem: 3, ipsum: 0, dolor: 6, sit: 2 };
	var c2 = { lorem: 0, ipsum: -1, dolor: 5, sit: 3 };

	si.validate(schema, c1); // Valid
	si.validate(schema, c2); // Invalid
```

---------------------------------------

<a name="v_someKeys" />
### someKeys

* **type**: array of string.
* **usable on**: object.

Check whether one of the given keys exists in object (useful when they are
optional).

__Example__

```javascript
	var si = require('schema-inspector');

	var schema = {
		type: 'object',
		someKeys: ['lorem', 'ipsum']
		properties: {
			lorem: { type: 'any', optional: true },
			ipsum: { type: 'any', optional: true },
			dolor: { type: 'any' }
		}
	};

	var c1 = { lorem: 0, ipsum: 1, dolor: 2  };
	var c2 = { lorem: 0, dolor: 2  };
	var c3 = { dolor: 2  };

	si.validate(schema, c1); // Valid
	si.validate(schema, c2); // Valid
	si.validate(schema, c3); // Invalid: Neither @.lorem nor @.ipsum is in c3.
```

---------------------------------------

<a name="v_strict" />
### strict

* **type**: boolean.
* **default**: false.
* **usable on**: object.

Only key provided in field "properties" may exist in object.

__Example__

```javascript
	var si = require('schema-inspector');

	var schema = {
		type: 'object',
		strict: true,
		properties: {
			lorem: { type: 'any' },
			ipsum: { type: 'any' },
			dolor: { type: 'any' }
		}
	};

	var c1 = { lorem: 0, ipsum: 1, dolor: 2  };
	var c2 = { lorem: 0, ipsum: 1, dolor: 2, sit: 3  };

	si.validate(schema, c1); // Valid
	si.validate(schema, c2); // Invalid: @.sit should not exist.
```

---------------------------------------

<a name="v_exec" />
### exec

* **type**: function, array of function.
* **usable on**: any.

Custom checker =). "exec" functions take two three parameter
(schema, post [, callback]). To report an error, use `this.report([message])`.
Very useful to make some custom validation.

__Example__

```javascript
	var si = require('schema-inspector');

	var schema = {
		type: 'object',
		properties: {
			lorem: {
				type: 'number',
				exec: function (schema, post) {
					// here scheme === schema.properties.lorem and post === @.lorem
					if (post === 3) {
						// As soon as `this.report()` is called, candidate is not valid.
						this.report('must not equal 3 =('); // Ok...it's exactly like "ne: 3"
					}
				}
			}
		}
	};

	var c1 = { lorem: 2 };
	var c2 = { lorem: 3 };

	si.validate(schema, c1); // Valid
	si.validate(schema, c2); // Invalid: "@.lorem must not equal 3 =(".
```

---------------------------------------

<a name="v_properties" />
### properties

* **type**: object.
* **usable on**: object.

For each property in the field "properties", whose value must be a schema,
validation is called deeper in object.

__Example__

```javascript
	var si = require('schema-inspector');

	var schema = {
		type: 'object',
		properties: {
			lorem: {
				type: 'object',
				properties: {
					ipsum: {
						type: 'object',
						properties: {
							dolor: { type: 'string' }
						}
					}
				}
			},
			consectetur: { type: 'string' }
		}
	};

	var c1 = {
		lorem: {
			ipsum: {
				dolor: 'sit amet'
			}
		},
		consectetur: 'adipiscing elit'
	};
	var c2 = {
		lorem: {
			ipsum: {
				dolor: 12
			}
		},
		consectetur: 'adipiscing elit'
	};

	si.validate(schema, c1); // Valid
	si.validate(schema, c2); // Invalid: @.lorem.ipsum.dolor must be a string.
```

---------------------------------------

<a name="v_items" />
### items

* **type**: object, array of object.
* **usable on**: array.

Allow to apply schema validation for each element in an array. If it's an
object, then it's a schema which will be used for all the element. If it's an
array of object, then it's an array of schema and each element in an array will
be checked with the schema which has the same position in the array.

__Example__

```javascript
	var si = require('schema-inspector');

	var schema1 = {
		type: 'array',
		items: { type: 'number'	}
	};

	var schema2 = {
		type: 'array',
		items: [
			{ type: 'number'	},
			{ type: 'number'	},
			{ type: 'string'	}
		]
	};

	var c1 = [1, 2, 3];
	var c2 = [1, 2, 'string!'];


	si.validate(schema1, c1); // Valid
	si.validate(schema1, c2); // Invalid: @[2] must be a number.
	si.validate(schema2, c1); // Valid
	si.validate(schema2, c2); // Invalid: @[2] must be a string.
```

---------------------------------------

<a name="v_alias" />
### alias

* **type**: string.
* **usable on**: any.

Allow to display a more explicit property name if an error is encounted.

__Example__

```javascript
	var si = require('schema-inspector');

	var schema1 = {
		type: 'object',
		properties: {
			_id: { type: 'string'}
		}
	};

	var schema2 = {
		type: 'object',
		properties: {
			_id: { alias: 'id', type: 'string'}
		}
	};

	var c1 = { _id: 1234567890 };

	var r1 = si.validate(schema1, c1);
	var r2 = si.validate(schema2, c1);
	console.log(r1.format()); // Property @._id: must be string, but is number
	console.log(r2.format()); // Property id (@._id): must be string, but is number
```

---------------------------------------

<a name="v_error" />
### error

* **type**: string.
* **usable on**: any.

This field contains a user sentence for displaying a more explicit message if
an error is encounted.

__Example__

```javascript
	var si = require('schema-inspector');

	var schema1 = {
		type: 'object',
		properties: {
			_id: { type: 'string' }
		}
	};

	var schema2 = {
		type: 'object',
		properties: {
			_id: { type: 'string', error: 'must be a valid ID.' }
		}
	};

	var c1 = { _id: 1234567890 };

	var r1 = SchemaInspector.validate(schema1, c1);
	var r2 = SchemaInspector.validate(schema2, c1);
	console.log(r1.format()); // Property @._id: must be string, but is number.
	console.log(r2.format()); // Property @._id: must be a valid ID.
```

## Sanitization

<a name="s_optional" />
### optional

* **type**: boolean.
* **default**: false.
* **usable on**: any.

---------------------------------------

<a name="s_type" />
### type

* **type**: string, array of string.
* **usable on**: any.

---------------------------------------

<a name="s_rules" />
### rules

* **type**: string, array of string.
* **usable on**: string.

---------------------------------------

<a name="s_comparators" />
### min, min

* **type**: string, number.
* **usable on**: string, number.

---------------------------------------

<a name="s_length" />
### minLength, maxLength

* **type**: integer.
* **usable on**: string.

---------------------------------------

<a name="s_exec" />
### exec

* **type**: function, array of functions.
* **usable on**: any.

---------------------------------------

<a name="s_properties" />
### properties

* **type**: object.
* **usable on**: object.

---------------------------------------

<a name="s_items" />
### items

* **type**: object, array of object.
* **usable on**: array.
