Other browsers have something similar even if they're not based on chromium. The shortcut keys vary in chrome on a mac command + alt + I will open dev tools. You can also right click a page and click inspect to open it up.

## Elements Tab

The elements tab shows the DOM tree this is like viewing the source but it's live and updates as things change on the page. If you edit something here what's shown on the page changes immediately. There is also a section that shows css properties for the selected element this updates immediately as well. This is a good place to figure out your dom/css without having to wait for anything to build.

## Console Tab

The console tab lets you run any javascript in the page. It's useful for inspecting elements, prototyping short functions, and forcing events to happen. It's similar to the python REPL.

## Sources Tab

The sources tab let's you open files loaded by the browser. If you set a breakpoint you can execute code line by line here to trace what's happening.

## Network Tab

This shows you network requests made to load the page. You can see their responses, timing and the url and headers sent to make the request.

## Other than debugging your own pages these can be really helpful for ripping content from other sites.

Open https://www.d20pfsrd.com/magic/spell-lists-and-domains/spell-lists-sorcerer-and-wizard and open the elements tab. We're going to scrape this data and store turn it into a JSON object we can use later.

This page uses jQuery but tries to hide it. Probably for compatibility. You can restore the default behavior by running this in the console. 
```
$ = jQuery
````

jQuery is an older framework used to build sites. It lets you find specific dom nodes and do things to them. The main jQuery function is now `$()`. It's stupid syntax but totally legal.

To find elements on the page you can use css like queries so to find all the `<table>` tags run this in the console.
```
tables = $('table')
```
`tables` is now an array like object. That's a javascript specific thing. It's not actually an array but anything that can work with an array can work with it because it has all the properties and methods of an array. Javascript has other similar types like thennable and promiselike, but we won't use them right away. Because tables is an array like object we can check it's length to see how many tables are on the page with `tables.length`. The result should be 11 one each for the levels of spell 0-9 and one extra that's an unrelated table on the screen.

This tables object is a special jQuery object that we can filter further or use to look up other elements, but we just want the individual table tags. To get the table tags you just access them like array elements so `tables[0]` is the first table tag. If you run that in the console you'll see the html fragment for the first table. If you expand the table tag you can see this page uses a caption tag on the tables to label them. That will help us filter out the table we don't need.

Javascript has a few kinds of for loop the generally preferred one is the "for of" loop. This loops over the elements of something. In this case the elements of an array. Try this in the console.
```
for(table of tables) {
    console.dir(table);
}
```

This loop creates a temporary local variable `table` that's only available in the body of the loop. It then sets table to the first item in tables and calls `console.dir()` on that table to print out it's value.

`console.dir` is non-standard but available in most browsers and nodejs. Instead of just printing text it outputs an object you can manipulate.  You'll notice this printed out 11 table tags and then undefined at the end. Whenever you run something in the console it's return value is printed, but our for loop doesn't return anything. As far as javascript is concerned not returning anything is the same as returning the special value undefined. For now it's enough to know that the undefined isn't one of the tables and it's safe to ignore.

If we look at the tables they are shown differently than when we just ran `tables[0]` above. This shows us all the properties on each table object. Our goal right now is to find all of the captions and find a way to get just the 10 tables we are interested in. So expand the first table and look at it's properties. There's one called children that's an array of all of the child nodes. If we expand that we'll see the first one is our caption. It's displayed as '0: caption' the 0 is the index in the array. caption is the type of node but that won't always be displayed. If we scroll down we'll see that the caption has a children property but there is nothing in the array.

The caption does have a few other ways to get what we need. First, right above the children property is a childNodes property that does have one element in it. The children property doesn't include plain text nodes, but childNodes does. The childNodes property contains 1 text node with a few properties containing the data we need. Since we're not trying to make a general purpose scraper we can pick any one of `data`, `nodeValue`, `textContent`, or `wholeText` on that text node to get the value. Also if we look further the caption element does have the text in a few properties `innerHTML`, `innerText`, and `outerText`. Usually when working with the DOM tree there are a lot of options to get the data we need, but they usually have specifics that make us want to use one over another. Some can only be read and if you write to them it will be overridden by another value or throw an error.

jQuery also gives us a few ways to get the text as well.
```
// For each caption inside of one of our tables
// This looks at child nodes of tables for caption tags and returns a jQuery result set of those tags.
for (caption of tables.find('caption')) {
    // Turn our caption dom node into a jQuery result set containing just that one node so we can use jQuery helper functions on it.
    var $caption = $(caption);
    // print out the text of the caption
    console.dir($caption.text());
}
```

That breaks the link between a table and it's associated caption. If we need to remember which table the caption is in we can do a longer form of that search.

```
// For each of our tables
for (table of tables) {
    var captions = $(table).find('caption');
    if (captions.length > 0) {
        for (caption of captions) {
            // This creates an anonymous object with two properties table and caption. table is set to the current value of table because we left out the value it defaults to the value of the variable with the same name. caption is set to the text of the caption node using the jQuery .text() helper.
            console.dir({table, caption: $(caption).text()});
        }
    } else {
        // In this case we're printing the table to see that it has no caption.
        console.dir({table, caption: ''})
    }
}
```

If we examine the output of the second loop we'll see a bunch of 'Object' objects. Expand them and check their captions to find the one that is empty. That's the one we want to remove so let's examine it a bit further. If we check the innerText of the table it starts with a pretty searchable string "Become an Editor" so search for that on the page itself not in dev tools. You'll find it's a table of links near the bottom of the page.

Now that we have a way to filter out or reject table let's take a look at the tables we want to keep and find out what's similar about them.


```
$tables
for (table of tables) {
    var captions = $(table).find('caption');
    if (captions.length > 0) {
        for (caption of captions) {
            // This creates an anonymous object with two properties table and caption. table is set to the current value of table because we left out the value it defaults to the value of the variable with the same name. caption is set to the text of the caption node using the jQuery .text() helper.
            console.dir({table, caption: $(caption).text()});
        }
    } else {
        // In this case we're printing the table to see that it has no caption.
        console.dir({table, caption: ''})
    }
}
```

Without looking this up lets try a normal array filter on this array like object.
```
tables.filter(tbl => {
    console.dir(tbl);
    return false;
})
```
If tables were a normal array this would print all the values and return an empty array. However this isn't a normal array. It does have a filter method but it works differently. Unfortunately for us this just prints the indexes.
We still don't need to lookup the docs to figure out how it works though. We can be pretty sure the first parameter is the index so we could use that to lookup each value, but we are probably getting more parameters that we aren't using yet. To check let's add a few more to our function.

```
tables.filter((a, b, c, d) => {
    console.dir({a, b, c, d});
    return false;
})
```

This will take the first four parameters passed to us and print them. Because this is javascript we won't get an error if we have too many parameters. They will have the undefined value. Sometimes a parameter in the middle might be undefined intentionally so it's not a good idea to assume there are no more parameters after the first undefined one. This is a special type of function called an arrow function in javascript or a lambda or anonymous function in other languages. If this was a normal function there is a special arguments object we could query to find the number of arguments.

The output we get for each is an intex as a, the table as b, and undefined as both c and d. Using that knowledge let's rewrite the loop with some better names.
```
tables.filter((index, table) => {
    console.dir({index, table});
    return false;
})
```

We know from looking at our data that each of our target tables has a single caption tag so lets look for that in our filter.
```
filteredTables = tables.filter((index, table) => $(table).find('caption').length)
```

This checks each table for a caption tag and if there is at least one the function returns something truthy, e.g. the number 1, otherwise the table is filtered out because the function returned something falsy, e.g. 0. In javascript true is truthy, and false is falsy, but not the other way around. Anything that when converted to a boolean is true is considered truthy and anything that converts to false is falsy. Specifically false, undefined, null, 0, the empty string '', an empty array, and the constant NaN are falsy, and everything else is truthy.

Now that we have our 10 tables in filteredTables we need to parse their contents into something useful. Lets create some objects for further processing. Javascript doesn't have many constraints on what we can do to an object at runtime so lets make the most basic object we can that just holds the table and has some space for everything else we want to store.

```
// This makes data an array of objects with a single property called table that points to our table
data = Array.prototype.map.call(filteredTables, table => ({table}))
```

This uses some new syntax. Since filteredTables isn't a real array we can't use the normal array map method on it. Instead we can find that method which is also called Array.prototype.map. Since functions are objects they can have methods and properties themselves. Javascript functions have two special methods call and apply. These methods execute the function. The only difference between them is how they take parameters. The call method takes parameters as it's own parameters the first parameter to call is treated as the object the function was called on and the rest are shifted one left and passed into the function. `myObj.myFn(a, b, c)` is the same as `MyObj.prototype.myFn.call(myObj, a, b, c)`. The apply function takes an array and uses the first element as the objecgt the function was called on and the rest as arguments. This is how we can call an array method on an array like object even though it's not an array.

Another important thing here is the syntax of the lambda. If there is exactly one parameter we can omit the parentheses around the parameter block before the arrow. If there is only one statement in the body we can omit the {} around the function body and the return statement. If however we want to return an anonymous object like this or something else with ambiguous syntax we need to wrap the return value in parentheses so combining all of our shortcuts so far `(table) => { return {table: table};}` becomes `table => ({table})`.

We could pull out the caption because we know how, but what we really want out of the caption is the level. So instead let's print out all the captions and find a way to pull out the level numbers.

```
// The console.log here is important so the newlines actually show as newlines and not '\n' in the text.
console.log(Array.prototype.map.call(filteredTables, table => $(table).find('caption').text()).join('\n'))
```

It should be pretty obvious we can just take the first character, but I'm going to explore a helpful option for more complicated situations using this one. Copy the text output, go to https://regexr.com and paste the text into the middle text box. This site lets you play with regular expressions live and see the output. Click the flags button and make sure global and multiline are on and the rest are off. Type `^(\d+).*` into the expression box. This should show a match highlighted in blue for each line. Click the details button below to see details about the match. What we want is for one of the groups to match our specific data. In this case it's the only group.

If we try this regular expression in the console with one of our strings we'll see the output is an array and group 1 is the second element.
```
'0-Level Sorcerer/Wizard Spells (Cantrips)'.match(/^(\d+).*/)
```

So lets use that to update our mapping function
```
data = Array.prototype.map.call(filteredTables, table => {
    const caption = $(table).find('caption').text();
    const level = parseInt(caption.match(/^(\d+).*/)[1]);
    return {table, level};
})
```

Now to start pulling out the rest of our data we need to look at the dom itself. Each of these tables has multiple thead and tbody sections. with a single header row naming the columns and an additional header row for each school. The spells are organized alphabetically within their schools. We can either hard code the names of the schools or pull them out of the tables as well. I'm going to explain how to pull them out.

```
data = Array.prototype.map.call(filteredTables, table => {
    const caption = $(table).find('caption').text();
    const level = parseInt(caption.match(/^(\d+).*/)[1]);
    const schools = {};
    const allSchools = [];
    const currentSchool = null;
    const schoolSpells = []
    const allSpells = [];
    for(row of $(table).find('tr')) {
        const $row = $(row);
        const $headerCells = $row.find('th');
        if ($headerCells.length > 1) {
            // This is our main header row so do nothing.
        } else if ($headerCells.length === 1) {
            // This is a bit of a cheat because we know what text is in the row.
            const schoolName = $headerCells.text();
            // This is a school header row.
            allSchools.push(schoolName)
            if (currentSchool !== null) {
                schools[currentSchool] = schoolSpells;
                schoolSpells = [];
            }
        } else {
            const $cells = $row.find('td');
            // This is a spell row.
            const spell = {
                name: $($cells[0]).text().trim(),
                components: $($cells[1]).text().trim(),
                description: $($cells[2]).text().trim(),
                source: $($cells[3]).text().trim(),
                // This is how we can get the linked url using jQuery helpers.
                sourceUrl: $($cells[3]).find('a').attr('href'),
                url: $($cells[0]).find('a').attr('href'),
            }
            schoolSpells.push(spell);
            allSpells.push(spell);
        }
    }
    // Cleanup we have to do this here because we process this when going on to the next school and that step doesn't exist for the last school
    if (currentSchool !== null) {
        schools[currentSchool] = schoolSpells;
        schoolSpells = [];
    }
    return {level, schools, allSchools, allSpells};
})
```

## Where do we go from here?
At this point we can technically use jQuery to fetch those pages and either parse their responses as plain text or load them in div's or iFrames and continue parsing them. It's possible to load all of those pages. Send each of them a script to parse the contents of the page and return that data to a script on this page and update our data object with the results.

## What if the page didn't have jquery.
You can dynamically import almost any javascript into a running page and most javascript libs are available on a public CDN.
```
const myNewScriptTag = document.createElement('script');
myNewScriptTag.src = 'https://cdnjs.cloudflare.com/ajax/libs/jquery/3.6.0/jquery.min.js'
document.body.append(myNewScriptTag);
```
Another library that's useful for manipulations like this is lodash.
`myNewScriptTag.src = 'https://cdnjs.cloudflare.com/ajax/libs/lodash.js/4.17.21/lodash.min.js';`

## Homework

Modify the last function so it returns something like this
{
    allSpells: [
        ...spells
    ],
    spellsByLevel: {
        0: [
            ...spells
        ],
        1: [
            ...spells
        ]
    },
    spellsBySchool: {
        Abjuration: [
            ...spells
        ],
        Conjuration: [
            ...spells
        ],
        Divination: [
            ...spells
        ]
    },
    spellsBySource: {
        PZO1110: [
            ...spells
        ],
        Blog: [
            ...spells
        ],
        'PPC:AoE': [
            ...spells
        ]
    }
}

# Referneces & Links

[Array methods](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array)  
[Function methods](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function)  
[Object methods](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object)  
[Lodash](https://lodash.com/docs/4.17.15)  
[jQuery](https://api.jquery.com/)  
