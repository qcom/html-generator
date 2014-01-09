# HTML Generator

## Module Dependencies

	fs = require 'fs'
	_ = require 'underscore'

## Data Variables

* an object representing all data exported from the site database, including
 * list of available ("Visible" according to the db) skus
 * map of available "Best Used For" category names to their URLs
* an export of the api
* a subset of the list of available skus including only those with available api data
* a map of every SKU to its generated HTML

=

	site = {
		skus : require '../data/site'
		bestUsedFors : require '../data/site-bestUsedFors'
	}
	api = require '../data/api-post-import'
	inApi = site.skus.filter (sku) -> sku in Object.keys(api)
	data = {}

## Utility Functions

Cache the identity function for the api values that need to be included in our template object but need no change.

	ident = (x) -> x

Convert a string formatted in camelCase style to title style (each term separated by spaces with each word beginning with a capital letter).

	camelToDisplay = (s) ->
		result = s[0].toUpperCase()
		uppered = s.toUpperCase()
		for i in [1...s.length]
			result += (if s[i] == uppered[i] then " #{uppered[i]}" else s[i])
		resultA

Given an array of objects, return an object with key value pairs taken from accessing properties on each object in the object array.  
The properites that are accessed are passed as string arguments (key and valKey).

	arrToObj = (arr, key, valKey) ->
		obj = {}
		arr.forEach (o) ->
			obj[o[key]] = o[valKey]
		obj

Simple function wrapper over the (typeof value) JavaScript operator, which mainly be used for the distinction between arrays and objecs.

	getType = (val) ->
		type = typeof val
		if Array.isArray val
			'array'
		else if type == 'object'
			if not val? then 'null' else 'object'
		else
			type

Determine whether or not the passed value is of type array or object and is thus composite in nature.

	isComposite = (val) ->
		getType(val) in ['array','object']

As long as the single argument passed is either an object or an array, ensure that the composite value is not empty.

	notEmpty = (obj) -> Object.keys(obj).length > 0

## Global (Helper) Variables

Thanks to the exclusion of unnecessary or incomplete keys, once the handling of attributes like relatedProductNumbers is complete,  
where specialized processing must occur, the remaining, "default" iteration over the product object will be entirely for the  
"details" section of the template, in which a grab bag of attributes are listed.

	exclude = ['name','url','baseProduct','productNumber','testimonials','pricing','faqs','packaging']

Two dispatch objects, both of which contain functions for accepting values of the appropriate keys,  
return those values with the necessary modifications performed.

The first dispatch object does *not* include a reference-type apiProduct parameter; as a result, no side effects are performed.

	handlers = {
		features : ident
		relatedProductNumbers : (arr) ->
			arr
		lockDimensions : (arr) -> arrToObj(arr, 'name', 'dimention')
		bestUsedFors : (arr) -> arrToObj (arr.filter (obj) -> site.bestUsedFors[obj.name]), 'name', 'image'
		videos : ident
	}

The second dispatch object _does_ accept an apiProduct reference, and mutates under conditions appropriate for each attribute.

	mutateHandlers = {
		images : (imageObj, apiProduct) -> apiProduct.schematic = imageObj.schematic if imageObj.schematic?
	}

## Iteration

Now we begin iterating over all product SKUs on the site, but only those that are in the api.  
Additionally, we set up a few variables needed for the current iteration:

* a reference to the API object identified by the current SKU is set
* an empty object for the (massaged) data necessary for the template
* an empty object for namespacing the random leftover attributes under one key, "details"

=

	filtered.forEach (sku) ->
		apiProduct = api[sku]
		product = {}
		details = {}



		for key, val of apiProduct
			val = apiProduct[key]
			type = getType val
			if val? and (not isComposite(val) or notEmpty val)
				if key of handlers
					value = handlers[key](val)
					if (notEmpty value)
						product[key] = value
				else if key of mutateHandlers
					mutateHandlers[key](val, product)
				else if key not in exclude
					details[camelToDisplay key] = val

		product.details = details if (notEmpty details)

		data[apiProduct.productNumber] = product;

	fs.writeFileSync './out.json', JSON.stringify(data)