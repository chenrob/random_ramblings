odl-tiles2.1
============
In this article, we examine the current "tile-centric" approach to laying out tiles and presenting an alternate "column-centric" approach to accomplish the same task.

Current Approach
----------------
When laying out content primarily involving images, one common pattern is to treat each image (and possibly some additional content such as price, author, or social indicators) as a "tile" to be laid out in some form of grid like structure.

Traditionally, each tile would be uniformly sized, allowing the viewer to observe distinct rows and columns.  Such designs are common on e-commerce websites such as [Newegg](http://www.newegg.com/) and [Target](http://www.target.com/).

A recent design trend is emerging which some refer to as "Pinterest-style grid layouts" based on the website that popularized the design.  This can be seen as an evolution of the traditional grid.  The core of this design is to embrace the differing dimensions of the images and position the tiles asymmetrically, with each column having tiles of differing heights.

The straightforward approach to implementing this design is to write Javascript to absolutely position each tile, resulting in HTML that looks similar to:

```html
<div id="ebay-style-accessed-on-2012Nov1">
	<div style="left: 0px; top: 0px; position: absolute;" realheight="275">content</div>
</div>

<div id="pinterest-style-accessed-on-2012Nov1">
	<div data-width="500" data-height="750" style="top: 1158px; left: 474px;" data-col="2">content</div>
</div>
```

To my knowledge, existing libraries such as [Masonry](http://github.com/desandro/masonry/) and [Wookmark](http://github.com/GBKS/Wookmark-jQuery/) also absolutely position each tile.

I refer to this class of implementation (where the styling is applied to each tile) as a "tile-centric" approach/design/implementation.

Problem Domain
--------------
In June and July 2012, we updated our old "seller pages" with a new "profile page" feature at [Oodle](http://www.oodle.com/).  The seller pages presented a user's listings in the same format as the search pages, but we decided to take a tile-centric approach instead for the profile pages.

During development, we noticed a couple of annoyances with a tile-centric implementation.

1.	One of our first observations was that we would render the web page with tiles overlapping the tile below it.  We traced this down to our `<img>` tags not having a defined height.  At first, we dealt with it by deferring the display of the tiles until all the images fully load (with detrimental effect to our page load times).  Fortunately, as all our images are stored in a 4:3 aspect ratio, we were able to specify the dimensions of the tile images via CSS.

	Looking at the example HTML above, we hypothesize that tile-centric websites dealing with images with unknown aspect ratios must find out the image's aspect ratio, calculate a resized dimension, and "stuff" the HTML tag with additional attributes such as `realheight` or `data-height`.

2.	As development progressed, we needed to hook up our in-house comment module with the tiles.  For improved usability, after a user submits a comment, we would update the tile with the comment to serve a visual confirmation.  This causes the height of the tile to increase.  As all the tiles are absolutely positioned, this causes the tile to overlap with the tile underneath it.

	**We noticed that it is non-trivial to update a tile's contents with a tile-centric implementation**

	We were able to work around this by writing a `retile(tile)` function that pushes down all the tiles underneath the given (updated) tile.  As we own both the tiling code and the comment module, we interfaced the two via firing a custom jQuery event.

Column-centric Approach
-----------------------
I perceived the tile-centric approach to be somewhat limiting and required extra code that I felt was unnecessary.  However, I was not able to come up with a better implementation before the feature went out to production.

I wanted a solution that:

* kept additional HTML tags/attributes to a minimum
* didn't need to wait for all the images to load  
(impacts users' perceived page load time)
* didn't need the images' height to be known  
(what if we are linking to off-site images with unknown heights)
* didn't need extra code to handle retiling  
(what if we're working with some minified Javascript library and cannot modify it)

### Basic Principle ###
We start off with this sample HTML:

```html
<section class="tiles-container" data-tileui-options='{"infiniteScroll": true, "maxPages": 10, "minCols": 3, "nextUrl": "/?o=25"}'>
	<section class="tile">
		<header class="bg-rough4 tile-header h3">...</header>

		<section class="item">
			...
			<div class="item-img-wrap">
				<a href="...">
					<img src="..." class="item-img" alt="..." />
				</a>
			</div>
			...
		</section>
	</section>
</section>
```

Instead of taking a tile-centric approach and directly set the position of each tile, we add a layer of indirection and create absolutely positioned column `<div>`s with the same width as the tiles.  We then detach the tiles from the DOM and insert them into the columns.

After the Javascript manipulates the DOM, we end up with something similar to:

```html
<section class="tiles-container" data-tileui-options='{"infiniteScroll": true, "maxPages": 10, "minCols": 3, "nextUrl": "/?o=25"}'>
	<div class="tile-column" style="position: absolute; left: 0px; width: 240px;">
		<section class="tile">...</section>
		<section class="tile">...</section>
	</div>
	<div class="tile-column" style="position: absolute; left: 255px; width: 240px;">
		<section class="tile">...</section>
		<section class="tile">...</section>
	</div>
</section>
```

The column-centric approach, in effect, offloads a bunch of work to the browser (such as retiling).  In theory, this results in smaller Javascript (no need to implement something that the browser gives us for free) which then further results in easier development (less code to read, less code that can break something) and marginally faster downloads.

Notice that the tiles are standard block level elements with the default static positioning.  When we give the columns the same width as the tiles, some cool stuff happens:

* *Invariant*: tiles automatically stack on top of each other
* *Invariant*: tiles will not overlap
* There is no need to calculate and set top positioning
* When a tile's content is changed, the other tiles in the same column will automatically be pushed down to maintain the invariant conditions  
(Note that this also works if we add/remove *entire tiles*)