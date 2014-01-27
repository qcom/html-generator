# HTML Generator

## Module Dependencies

	fs = require 'fs'

## Data Variables

* an object namespacing all data exported from the site database, which includes
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

Cache the identity function for the api attributes that need to be included in our template object but can be done so without processing.

	ident = (x) -> x

Convert a string formatted in camelCase style to title/display style (each term separated by spaces with each word beginning with a capital letter).

	camelToDisplay = (s) ->
		result = s[0].toUpperCase()
		uppered = s.toUpperCase()
		for i in [1...s.length]
			result += (if s[i] == uppered[i] then ' ' + uppered[i] else s[i])
		result

Given an array of objects, return an object with key value pairs taken from accessing properties on each object in the object array. The properites that are accessed are passed as string arguments `key` and `valKey`. This function assumes that each `o[key]` value is unique, otherwise overwrites on `obj` will occur.

	arrToObj = (arr, key, valKey) ->
		obj = {}
		arr.forEach (o) ->
			obj[o[key]] = o[valKey]
		obj

Simple function wrapper over `typeof`, which will be utilized primarily for its distinction of arrays and objects.

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
	notComposite = (val) -> not isComposite(val)

Check if a value is meaningful by ensuring the input passes three tests:

* it is not null or undefined
* if it is a simple type, it is not the empty string
* if it is either an array or object, it has at least one key/value pair

=

	notEmpty = (obj) -> obj? and ((notComposite(obj) and obj != "") or Object.keys(obj).length > 0)

## Global (Helper) Variables

Thanks to the exclusion of unnecessary or incomplete (regarding the api implementation) keys, once the handling of "special case" attributes (e.g. `relatedProductNumbers`) is complete, where processing specific to those attributes must occur, the remaining iteration over the product object will be entirely dedicated to the `details` section of the template, in which a grab bag of attributes are listed.

	exclude = ['name','url','baseProduct','productNumber','testimonials','pricing','faqs','packaging','videos']

Two dispatch objects, both of which contain functions for accepting values of the appropriate keys, handle those values appropriately.

The first dispatch object does *not* support a reference-type `product` parameter; as a result, no side effects are performed.

	handlers =
		features : ident
		relatedProductNumbers : (arr) -> arr.filter (sku) -> sku in site.skus and sku of api
		lockDimensions : (arr) -> arrToObj(arr, 'name', 'dimention')
		bestUsedFors : (arr) -> arrToObj (arr.filter (obj) -> site.bestUsedFors[obj.name]), 'name', 'image'

		# videos : ident

The second dispatch object _does_ accept a reference to `product`, and mutates under conditions appropriate for each attribute.

	mutateHandlers =
		images : (imageObj, product) -> product.schematic = imageObj.schematic if imageObj.schematic?

## Iteration

We now begin iterating over all product SKUs on the site, but only those that are in the api. Additionally, we set up a few variables needed for the current iteration:

* a reference to the API object identified by the current SKU
* an empty object for the (massaged) data necessary for the template
* an empty object for namespacing the leftover attributes under one `details` key

=

	inApi.forEach (sku) ->
		apiProduct = api[sku]
		product = {}
		details = {}

For each product, we iterate over every key and value on our api handle, but only if the value is not empty (in other words, it is not null, undefined, or the empty string, and if it is an array or object, it has at least one key/value pair).

		for key, val of apiProduct when notEmpty(val)

If the current key has an associated function in `handlers`, evaluate the filter and update `product` as long as the resultant value is not empty. The inclusion of this assignment and validation boilerplate in the `apiProduct` iteration frees the filter functions from having to obfuscate their meaning with irrelevant and redundant assginment statements.

			videoString = '<param name="bgcolor" value="#FFFFFF" />\t'

			if key of handlers
				result = handlers[key](val)
				if notEmpty(result)
					# if key is 'videos'
						# result.forEach (videoObj) -> videoObj.embededCode = videoObj.embededCode.replace(videoString, '')
					product[key] = result

If the current key was not found in `handlers`, check to see if it exists in `mutateHandlers`. If so, invoke the side-effect-inducing function with the current iteration value as well as a reference to the `product` template object. The reason `mutateHandlers` exists is because certain attributes are not replicated key-for-key on `product` and thus the generic form of assignment explained above will not suffice.

			else if key of mutateHandlers
				mutateHandlers[key](val, product)

All remaining keys, provided they are not in `exclude`, will have the display-appropriate version of their key stored in the `details` object mapped to the current value.

			else if key not in exclude
				details[camelToDisplay(key)] = val

As long as `details` contains at least one attribute under its namespace, attach `details` to `product`.

		product.details = details if notEmpty(details)

As the last step in the current product iteration, add the template object to `data`.

		data[sku] = product

After every sku in `inApi` has been iterated over, write the data object to disk.

	fs.writeFileSync './out.json', JSON.stringify(data)
