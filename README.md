# HTML Generator

## Module Dependencies

	fs = require 'fs'

## Data Variables

* an object representing all data exported from the site database which including
 * a list of available ("Visible" according to the db) skus
 * a map of available "Best Used For" category names to their URLs
* an export of the api
* a subset of the list of available skus including only those with available api data
* a map of every SKU to its generated HTML

=

	site =
		skus : require '../data/site-skus'
		bestUsedFors : require '../data/site-bestUsedFors'
	api = require '../data/api'
	inApi = site.skus.filter (sku) -> sku of api
	data = {}

## Utility Functions

Cache the identity function for the api values that need to be included in our template object but no processing.

	ident = (x) -> x

Convert a string formatted in camelCase style to title/display style (each term separated by spaces with each word beginning with a capital letter).

	camelToDisplay = (s) ->
		result = s[0].toUpperCase()
		uppered = s.toUpperCase()
		for i in [1...s.length]
			result += (if s[i] == uppered[i] then ' ' + uppered[i] else s[i])
		result

Given an array of objects, return an object with key value pairs taken from accessing properties on each object in the object array. The properites that are accessed are passed as string arguments `key` and `valKey`. This function assumes that each `o[key]` value is unique, otherwise overwrites will occur.

	arrToObj = (arr, key, valKey) ->
		obj = {}
		arr.forEach (o) ->
			obj[o[key]] = o[valKey]
		obj

Simple function wrapper over the tyepof operator, which will be utilized primarily for its distinction of arrays and objecs.

	getType = (val) ->
		type = typeof val
		if Array.isArray val
			'array'
		else if type == 'object'
			if not val? then 'null' else 'object'
		else
			type

Determine whether the passed argument is of type array or object and is thus composite in nature.

	compositeTypes = ['array','object']
	isComposite = (val) -> getType(val) in compositeTypes
	notComposite = (val) -> getType(val) not in compositeTypes

As long as the single argument is of a composite data type, ensure that the value is not empty.

	notEmpty = (obj) -> Object.keys(obj).length > 0

## Global (Helper) Variables

Thanks to the exclusion of unnecessary or incomplete (regarding the api implementation) keys, once the handling of "special case" attributes (e.g. `relatedProductNumbers`) is complete, where processing specific to those attributes must occur, the remaining iteration over the product object will be entirely dedicated to the `details` section of the template, in which a grab bag of attributes are listed.

	exclude = ['name','url','baseProduct','productNumber','testimonials','pricing','faqs','packaging']

Two dispatch objects, both of which contain functions for accepting values of the appropriate keys, handle those values appropriatley.

The first dispatch object does *not* support a reference-type `apiProduct` parameter; as a result, no side effects are performed.

	handlers =
		features : ident
		relatedProductNumbers : (arr) ->
			arr
		lockDimensions : (arr) -> arrToObj(arr, 'name', 'dimention')
		bestUsedFors : (arr) -> arrToObj (arr.filter (obj) -> site.bestUsedFors[obj.name]), 'name', 'image'
		videos : ident

The second dispatch object _does_ accept an `apiProduct` reference, and mutates under conditions appropriate for each attribute.

	mutateHandlers =
		images : (imageObj, apiProduct) -> apiProduct.schematic = imageObj.schematic if imageObj.schematic?

## Iteration

We now begin iterating over all product SKUs on the site, but only those that are in the api. Additionally, we set up a few variables needed for the current iteration:

* a reference to the API object identified by the current SKU
* an empty object for the (massaged) data necessary for the template
* an empty object for namespacing the random leftover attributes under one key, "details"

=

	inApi.forEach (sku) ->
		apiProduct = api[sku]
		product = {}
		details = {}

For each product, we iterate over every key and value on our api handle, but only if the value is not null or undefined and if it is an array or object, it is not empty.

		for key, val of apiProduct when val? and (notComposite(val) or notEmpty val)

If the current key has an associated function in `handlers`, then evaluate the filter mapped to the key and update `product` as long as the resultant value from the filter function is not empty.

			if key of handlers
				result = handlers[key](val)
				if notEmpty(result)
					product[key] = result

If the current key was not found in `handlers`, check to see if it exists in `mutateHandlers` and if so, invoke the side-effect-inducing function with the current iteration value as well as a reference to the template obj, `product`.

			else if key of mutateHandlers
				mutateHandlers[key](val, product)

All remaining keys, provided they are not in `exclude`, will have their display-appropriate version of their key stored in the `details` object mapped to the current value.

			else if key not in exclude
				details[camelToDisplay key] = val

Attach the details object to our template object, but only if it actually contains any attributes under its namespace.

		product.details = details if notEmpty(details)

As the last step in the current product iteration, add the template object to `data` under the sku as indicated by the api.

		data[apiProduct.productNumber] = product;

After every sku in `inApi` has been iterated over, write the data object to disk.

	fs.writeFileSync './out.json', JSON.stringify(data)