# HTML Generator

## Module Dependencies

	fs = require 'fs'
	jade = require 'jade'

## Data Variables

* a map of every SKU to its cleaned api data
* path to our template file
* compiled template function from our jade template file
* a map of product SKUs to the width value of their schematic images
* a map of product SKUs to their website URLs

=

	data = require '../out'
	templateFile = './template.jade'
	template = jade.compile(fs.readFileSync(templateFile), filename : templateFile, pretty : false)
	schematicWidths = require '../data/site-schematicWidths'
	urls = require '../data/site-urls'
	s = 'sku\thtml\n'
	output = {}

## Generation

	for sku, productData of data
		templateObj = 
			data : productData
			dataset : data
			sectionOrder : {
				features :
					['features']
				specifications :
					['lockDimensions','schematic']
				details :
					['details']
				bestUsedFors :
					['bestUsedFors']
				videos :
					['videos']
			}
			isAvailable : (keys, obj) ->
				length = (keys.filter (key) -> key of obj).length
				length > 0 and length == keys.length
			schematicWidths : schematicWidths
			urls : urls
			sku : sku
		console.log sku
		html = template(templateObj).replace('\n','')
		output[sku] = html
		s = "#{s}#{sku}\t#{html}\n"
		fs.writeFileSync "./pages/#{sku}.html", html, 'utf8'
	fs.writeFileSync './out.csv', s, 'utf8'
	fs.writeFileSync './output.json', JSON.stringify(output), 'utf8'
