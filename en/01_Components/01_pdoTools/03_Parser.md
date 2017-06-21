The Parser in pdoTools represents a separate class, which registers in the system and intercepts MODX parsing and processing tags on the page.  Within MODX this is usually done by modParser.

In older versions of the components the parser needed to set after the installation, but with version 2.1.1-pl it is enabled by default. If, for some reason, you do not like - delete the following system settings.

-   **parser\_class** - the name of the parser class
-   **parser\_class\_path** - path to the parser class

## Principle of operation

First of all - why do we need chunks? They are needed to separate the view from the logic. In most cases they are needed to allow users to edit the default behaviour of installed extras.

In MODX's chunks and templates you can use various types of placeholders:

* `[[+tag]]` - ususal placeholder that will be replaced with a value due chunk processing
* `[[++setting]]` - system settings from `modX::config`
* `[[*field]]` - field of current resource in `modX::resource` property
* `[[%lexicon]]` - entry from lexicon to display text strings from extras dictionaries
* `[[~number]]` - links to resources
* `[[snippet]]` - cached snippet
* `[[!snippet]]` - uncached snippet
* `[[$chunk]]` - chunk

Every tag can be called uncached if you prefix it with the exclamation sign. Also you might disable any tag by a leading dash.

* `[[!uncached_snippet]]`
* `[[-!disabled_snippet]]`

### And how exactly does MODX parser process this tags?

modParser collects all tags from chunk with regular expressions, creates modTag objects from them and calls the method `process()`. Each tag in your chunk is one modTag object instance for modParser.

If you are using output filters for tags - it is another xPDO object. If you are using a MODX snippet as an output filter it will be processed each time as modParser finds it in your chunk.

If you are using 3 snippets in a chunk and get 10 records from the database - you will get **30** snippet calls.

modParser is recursive parser, so if it can`t process some tag right away - it will leave it for the next iteration. First, it processes all of the cached tags, then additionally processes all the uncached.

If there is no value for a uncached placeholder on the current page - modParser will try to parse it up to 10 times (by default).  This is why a complicated chunk can be parsed for a very long time.

## pdoParser Chunk Processing

pdoParser can be used in two cases:

-   when rendering Chunk snippet - this is always and in all the snippets using pdoTools, regardless of the system settings.
-   at page rendering - only if the parser is included in the system settings.

In the class for pdoTools there are 2 methods which are very similar to those in the class modX:

-   **getChunk** - complete processing chunk can use native parser MODX
-   **parseChunk** - only replacing placeholders on values, modParser not called

The main feature of these methods is that the chunk is used to download protected method **\_loadChunk** , which can not only download a chunk from the database, but also makes it arbitrary strings.

### Chunk Options

So both parser methods support the following types of named chunks:

#### @INLINE or @CODE

One of the most popular options - the ability to imbed a body chunk right on the page. For example:

```
[[!pdoResources?
    &parents=`0`
    &tpl=`@INLINE <p>{{+id}} - {{+pagetitle}}</p>`
]]
```

When used INLINE there is a feature, which many people do not care for. For all the placeholders to be processed by the snippet they must be set to curly braces like `{{+id}}`.  Otherwise, the `[[+id]]` will be processed by the parser of the page and not the parser for the chunk.

Therefore on the page like this:

```
[[!pdoResources?
    &parents=`0`
    &tpl=`@INLINE <p>[[+id]] - [[+pagetitle]]</p>`
]]
```

The placeholders `[[+id]]`or `[[+pagetitle]]`, will be replaced with the id and pagetitle the current page.  If embedd the code above on resource 15 you will get the following results:

```
15 - Page Title
15 - Page Title
15 - Page Title
```

As in the example you must use the unusual placeholders - `{{+}}`instead `[[+]]`. The system parser with not touch them, and pdoParser replaces them during normal operation.

You can use the curly parenthesis as a placeholder frame in all chunks pdoTools - It will turn them in `[[+]]` immediately.

For the same reason you should never make calls to snippets and filters in INLINE chunks. **That will not work** :

```
[[!pdoResources?
    &parents=`0`
    &tpl=`@INLINE <p>[[+id]] - [[+pagetitle:default=`Default Title`]]</p>`
]]
```

This version will work without problems

```
[[!pdoResources?
    &parents=`0`
    &tpl=`@INLINE <p>{{+id}} - {{+pagetitle:default=`Default Title`}}</p>`
]]
```

Remember this nuance when using **INLINE** chunks.

#### @FILE

Many people complain that MODX can not keep the chunks in files and force the system to work with the database. This is inconvenient for any version control system and slower.

MODX Version 2.2 [proposes the use of static elements][0] for various reasons, this method may still be less convenient than direct work with files.

pdoTools opens that possibility when specifying @FILE:

```
[[!pdoResources?
    &parents=`0`
    &tpl=`@FILE resources/mychunk.tpl`
]]
```

For security purposes, you can use only files with the extension **html** and **tpl**, and only within a predetermined directory set by the system setting **pdotools\_elements\_path** .

You can specify your own directory for the file through the parameter **&elementsPath**:

```
[[!pdoResources?
    &parents=`0`
    &tpl=`@FILE resources/mychunk.tpl`
    &elementsPath=`/core/elements`
]]
```

The file will be downloaded from a file `/core/elements/resources/mychunk.tpl` from the root of the site.

#### @TEMPLATE

This type of chunk allows you to use the system templates (ie modTemplate objects) for processing output.

```
[[!pdoResources?
    &parents=`0`
    &tpl=`@TEMPLATE Base Template`
]]
```

If you specify a blank template and the selected record contains the field `template` which includes the id or name of the template, the record will be wrapped in the indentified template:

```
[[!pdoResources?
    &parents=`0`
    &tpl=`@TEMPLATE`
]]
```

This is analogous to the snippet [renderResources][3].

and you can specify a set of parameters in the derivation of the pattern (as with snippets):

```
[[!pdoResources?
    &parents=`0`
    &tpl=`@TEMPLATE Base Template@MyPropertySet`
]]
```

The values of this property set `MyPropertySet` will be inserted into the template.

#### Conventional Chunks

This is the default mode that loads a chunk from the database:

```
[[!pdoResources?
    &parents=`0`
    &tpl=`MyChunk`
]]
```

Similarly, supports and sets of parameters:

```
[[!pdoResources?
    &parents=`0`
    &tpl=`MyChunk@MyPropertySet`
]]
```

All of these methods work **in all native pdoTools snippets** and all other snippets that use the pdoTools methods `getChunk` and `parseChunk`.

### getChunk method

The method declaration looks like this:

```php
getChunk(string $chunkName, array $properties, bool $fastMode = false)
```

The method loads the specified chunk (following the @BINDING instructions, if any) and fully observing it, replacing all placeholders on the submitted values.

The third parameter `fastMode` cuts out any remaining rough placeholders to avoid unnecessary tags on the page. If this is not done, then the parser will try to recursively parse the tags (up to 10 iterations by default) using `modParser`, which can cause slowdowns.

Recursive parser - it is one of the advantages of MODX and specifically left tags are very common in the logic of the system snippets. Therefore `fastMode` is disabled by default, and should be used only if you are confident in what you are doing.

PdoTools parser will not use the system parser, `modParser`, if it is able to replace all the placeholders.  If, however, the chunk contains some additional filters or snippets, the job is transmitted to the `modParser`, which requires additional time for processing.

### parseChunk method

And this method is declared like this:

```php
parseChunk(string $name, array $properties, string $prefix = '[[+', string $suffix = ']]')
```

It also creates a chunk of a specified name, examining @BINDING then simply replaces the placeholders with values ​​without any special treatment.

This is the easiest and fastest way to design data in chunks.

## Processing pages

If pdoParser is enabled in the settings, it is called to process the entire page when outputting it to the user.

When you use this parser all chunks and MODX supplements are processed a little faster. Only "a little" because it does not process the conditions and filters and process only simple tags, such as `[[+id]]`and `[[~15]]`. However, it does it faster than `modParser`, because it does not create unnecessary objects.

In addition to the possible gain speed, you get more and more opportunities for convenient output data from different resources.

### tags fastField

At the end of 2012 the public [was presented][4] a small plug-in to add new tags parser MODX, which then grew into [fastField component][1].  It adds the ability to handle additional placeholders like `[[#15.pagetitle]]`. [With the permission of the author][2], this functionality is already included in pdoParser, and even slightly extended.

All tags start with fastField `#` further contain or **id required resource**, or the name of a global array.

Conventional resource fields:

```
[[#15.pagetitle]]
[[#20.content]]
```

TV resource settings:

```
[[#15.date]]
[[#20.some_tv]]
```

Fields miniShop2 goods:

```
[[#21.price]]
[[#22.article]]
```

Arrays of resources and goods:

```
[[#12.properties.somefield]]
[[#15.size.1]]
```

Superglobals:

```
[[#POST.key]]
[[#SESSION.another_key]]
[[#GET.key3]]
[[#REQUEST.key]]
[[#SERVER.key]]
[[#FILES.key]]
[[#COOKIE.some_key]]
```

You can specify any of the fields in arrays:

```
[[#15.properties.key1.key2]]
```

If you do not know what values ​​are located within an array - just point it and it will be printed in full:

```
[[#GET]]
[[#15.colors]]
[[#12.properties]]
```

Tags can be combined with fastField tags MODX:

```
[[#[[++site_start]].pagetitle]]

[[#[[++site_start]]]]
```

### Templating with Fenom

Support for templating using [Fenom][5] [appeared in pdoTools version 2.0][6], after which it began to require PHP 5.3+.

Fenom works much faster than the native `modParser`, and if you overwrite a chunk so that there is not longer any MODX tags, `modParser` does not run. Of course, the simultaneous operation of old tags, and new in one chunk **is allowed** .

The following system settings affect how templates are processed:

-   **pdotools\_fenom\_default** - all chunks processed by pdoTools `getChunk()` are processed by Fenom. Enabled by default.
-   **pdotools\_fenom\_parser** - includes Fenom template processing for all pages of the site.  Not only chunks, but the templates as well.
-   **pdotools\_fenom\_php** - includes support for PHP templating functions. Very dangerous function, as any manager will have access to PHP directly from the chunk.
-   **pdotools\_fenom\_modx** - adds the system variables `{$modx}` and `{$pdoTools}` in Fenom templates. Also very dangerous - any manager can manipulate objects from MODX chunks.
-   **pdotools\_fenom\_options** - a JSON string with an array of Fenom settings as described in [the official documentation][7]. For example:`{"auto_escape":true,"force_include":true}`
-   **pdotools\_fenom\_cache** - caching templates. Only makes sense for complex chunks.  It is disabled by default.

By default Fenom is enabled to operate only on chunks that pass through pdoTools. It is completely safe and system managers do not receive any additional features, but gain a more convenient syntax and higher speed.

Enabling **pdotools\_fenom\_parser** allows developers to use Fenom syntax directly in the content of the document and template pages, but there is one caveat - the template may incorrectly respond to the curly parenthesis.  In such cases, the author recommends using the tag {[ignore][8]}.

If you plan to include Fenom globally for the whole site, you need to check whether the pages normally work at all.

#### Syntax

To start we advise that you read [the official documentation][7], and then look at the syntax reference to MODX.

All variables from the snippets are made available in the chunk as it is, so to rewrite the old chunks to the new syntax is a real pleasure.

| MODX                                                                | Fenom                               |
| ------------------------------------------------------------------- | ----------------------------------- |
| **[[+Id]]**                                                         | `{$Id}`                             |
| **[[+Id: default = \`test\`]]**                                     | `{$Id:? 'Test'}`                    |
| **[[+Id: is = \`\`: then = \`test\`: else =\`[[+ pagetitle]]\` ]]** | `{$ Id == ''? 'Test': $ pagetitle}` |

To use a more complex entity, pdoParser provides an object in a variable **{$\_modx}**, which enables secure access to certain system variables and methods.

| MODX                      | Fenom                                                 |
| ------------------------- | ----------------------------------------------------- |
| **[[*Id]]**               | `{$_modx->resource.id}`                          |
| **[[*Tv\_param]]**        | `{$_modx->resource.tv_param}`                   |
| **[[%Lexicon]]**          | `{$_modx->lexicon('lexicon')}`                 |
| **[[~15]]**               | `{$_modx->makeUrl(15)}`                         |
| **[[~[[*Id]]**            | `{$_modx->makeUrl($_modx->resource.id)}` |
| **[[++system\_setting]]** | `{$_modx->config.system_setting}`               |

In addition you have access to the system settings variables: **{$\_modx-&gt;config}**

```
{$_modx->config.site_name}
{$_modx->config.emailsender}
{$_modx->config['site_url']}
{$_modx->config['any_system_setting']}
```

**$\_modx-&gt;user** - an array of the current user data if they are authenticated:

```
{if $_modx->user.id > 0}
    Hello, {$_modx->user.fullname}!
{else}
    Not logged in.
{/if}
```

**$\_modx-&gt;context** - an array containing the current context data

```
Current Context {$_modx->context.key}
```

**$\_modx-&gt;resource** - an array with the current resource data

```
{$_modx->resource.id}
{$_modx->resource.pagetitle}
{$_modx->makeUrl($_modx->resource.id)}
```

**$\_modx-&gt;lexicon** - the object (not an array!) `modLexicon`, which can be used to download arbitrary dictionaries:

```
{$_modx->lexicon->load('ms2gallery:default')}
Lookup from lexicon: {$_modx->lexicon('ms2gallery_err_gallery_exists')}
```

To withdrawal entries corresponding to the individual function use `{$_modx->lexicon()}`.

#### Placeholders with a dot or dash

There are many places where MODX uses placeholders which cannot be specified in `Fenom`, since they do not meet [the rules of PHP variable names][9]. For example, the placeholders containing a period or dash.

To access these placeholders the developer needs to use a second service variable - **$\_pls**:

```
{$_pls['tag.subtag']}

{var $tv_name = $_pls['tv-name']}
{$tv_name}
```

If the placeholder was put into a global array `modX::placeholders`, then the developer should use something like this:

```
{var $tag_sub_tag = $_modx->getPlaceholder('tag.sub_tag')}
{$tag_sub_tag}

{$_modx->getPlaceholder('tv_name')}
```

Incorrect specification of the variables will results in your template not being compiled which will be recorded in the system log.

#### Filling Placeholders

Fenom operates in a single pass, not recursively, as a native parser in MODX.

Performing the entire templating at one time results in a very high speed, but keep in mind that **placeholders** are available **only after execution** of the corresponding snippet.

For example: you need to get the value of the placeholder `{$mse2_query}`(search terms) in the form, but the search box should appear above the results display. To do this run snippet `mSearch2` and transmit the results placeholders, for example `searchResults`:

```
{'!pdoPage' | snippet : [
    'element' => 'mSearch2',
    'toPlaceholder' => 'searchResults'
]}
```

Next, call the snippet search form where Fenom parser will substitute the value of the placeholder `{$mse2_query}`:

```
{'!mSearchForm' | snippet}
```

Then use the results snippet mSearch2:

```
{'searchResults' | placeholder}
```

If the snippet is not able to save the results of its work into a placeholder, you can assign them to a variable in Fenom:

```
{var $date = 'dateAgo' | snippet : ['input' => '2016-09-10 12:55:35']}

...

Print the date: {$date}.
```

This is very similar to the logic of the usual script.

#### Snippets and Chunks

The variable `{$_modx}` is actually a simple and secure [class microMODX][10]

Therefore snippets and chunks are called like this:

```
{$_modx->runSnippet('!pdoPage@PropertySet', [
    'parents' => 0,
    'element' => 'pdoResources',
    'where' => ['isfolder' => 1],
    'showLog' => 1,
])}
{$_modx->getPlaceholder('page.total')}
{$_modx->getPlaceholder('page.nav')}
```

**RESUME HERE**

As you can see, the syntax is nearly identical to the PHP, which opens up new possibilities. For example, arrays can be specified instead JSON rows.

By default, snippets are called cached, but you can add `!`before the name - in tags MODX.

If the call snippet uses the native method MODX, it starts to display chunks pdoTools, and you can use all of its features:

    {$_modx->getChunk('MyChunk@PropertySet')}

    {$_modx->parseChunk('MyChunk', [
        'pl1' => 'placeholder1',
        'pl2' => 'placeholder2',
    ])}

    {$_modx->getChunk('@TEMPLATE Base Template')}

    {$_modx->getChunk('@INLINE
        Имя сайта: {$_modx->config.site_name}
    ')}

    {$_modx->getChunk(
        '@INLINE Передача перемнной в чанк: {$var}',
        ['var' => 'Тест']
    )}

    {$_modx->getChunk('
        @INLINE Передача переменной в вызов сниппета:
        {$_modx->runSnippet("pdoResources", [
            "parents" => $parents
        ])}
        Всего результатов: {$_modx->getPlaceholder("total")}
        ',
        ['parents' => 0]
    )}

Examples of the above a bit crazy, but it is currently running.

#### caching management

The object **{$ \_modx}** available SERVIS modX :: cacheManager, which allows you to set an arbitrary time caused by caching snippets:

    {if !$snippet = $_modx->cacheManager->get('cache_key')}
        {set $snippet = $_modx->runSnippet('!pdoResources', [
            'parents' => 0,
            'tpl' => '@INLINE {$id} - {$pagetitle}',
            'showLog' => 1,
        ])}
        {set $null = $_modx->cacheManager->set('cache_key', $snippet, 1800)}
    {/if}

    {$snippet}

View this cache can be `/core/cache/default/`, in the example, it is saved on 30 minutes.

`set $null = ...`I need to `cacheManager->set`put the 1 (ie true) on the page.

And you can run the system CPUs (if enough rights):

    {$_modx->runProcessor('resource/update', [
        'id' => 10,
        'alias' => 'test',
        'context_key' => 'web',
    ])}

#### authorization check

Since the object with the user `{$_modx}`is not present, methods of testing and authorization of access rights handed down directly into the classroom:

    {$_modx->isAuthenticated()}
    {$_modx->hasSessionContext('web')}
    {$_modx->hasPermission('load')}

#### Displays information about the speed

**Since version 2.1.4** you can use the method `{$_modx->getInfo(string, bool, string)}`to obtain data on the speed of the system at the moment. He has three options:

-   **: string** - to bring certain key from the data array (empty by default)
-   **bool** - bring all the data as a string rather than an array ( `true`default)
-   **string** - draw string in chunk can use any type of chunks pdoTools ( `@INLINE {$key}: ${value}`).

The default output (line with all the data):

    {$_modx->getInfo()}
    // queries: 39
    // totalTime: 0.1067 s
    // queryTime: 0.0069 s
    // phpTime: 0.0998 s
    // source: database

Output a specific value:

    {$_modx->getInfo('totalTime')}
    // 0.1067 s
    {$_modx->getInfo('source')}
    // database

Making strings in your chunk:

    {$_modx->getInfo('', true, '@INLINE {$key} => {$value}')}
    // queries => 39
    // totalTime => 0.1155 s
    // queryTime => 0.0091 s
    // phpTime => 0.1064 s
    // source => database

You can even add a string to the lexicon pdoTools (or any other) and remove the keys through it:

    {$_modx->lexicon->load('pdotools:default')}
    {$_modx->getInfo('', true, '@INLINE {$_modx->lexicon("pdotools_" ~ $key)} => {$value}')}

Or place without lexicons randomly. Simply assign a variable array of data and derive its keys:

    {set $info = $_modx->getInfo('', false)}
    Время работы: {$info.totalTime}
    Время запросов: {$info.totalTime}
    Количество запросов: {$info.queries}
    Источник: {$info.source}

Displays information only for users who are logged into the manager:

    {if $_modx->hasSessionContext('mgr')}
        {$_modx->getInfo()}
    {/if}

#### Other methods

These methods should be familiar to all developers MODX, so just show them examples:

    {$_modx->regClientCss('/assets/css/style.css')}
    {$_modx->regClientScript('/assets/css/script.js')}

    {$_modx->sendForward(10)}
    {$_modx->sendRedirect('http://yandex.ru')}

    {$_modx->setPlaceholder('key', 'value')}
    {$_modx->getPlaceholder('key')}

    {if $res = $_modx->findResource('url-to/doc/')}
        {$_modx->sendRedirect( $_modx->makeUrl($res) )}
    {/if}

You can even receive and display resources without snippets:

    {var $resources = $_modx->getResources(
        ['published' => 1, 'deleted' => 0],
        ['sortby' => 'id', 'sortdir' => 'ASC', 'limit' => 50]
    )}
    {foreach $resources as $resource}
        {$_modx->getChunk('@INLINE <p>{$id} {$pagetitle}</p>', $resource)}
    {/foreach}

Sometime in the class of the new methods added, so look them [in the file](https://github.com/bezumkin/pdoTools/blob/master/core/components/pdotools/model/pdotools/_micromodx.php)

### modifiers

Fenom able to use MODX snippets as modifiers. For example, to install the components **dateAgo** , you can immediately isoplzovat it to display dates:

    [[!pdoResources?
        &parents=`0`
&tpl=`@INLINE <p>{$id} - {$pagetitle} {$createdon | dateago}</p>`
    ]]

Also works **Jevix** , and any other snippets MODX filters that take parameters `$input`and `$options`, according to [the documentation](http://docs.modx.pro/system/the-basics/filters-input-and-output) .

    [[!pdoResources?
        &parents=`0`
        &includeContent=`1`
&tpl=`@INLINE <p>{$id} - {$pagetitle} {$createdon | dateago} {$content | jevix}</p>`
    ]]

Modifiers can be operated in series:

    [[!pdoResources?
        &parents=`0`
        &includeContent=`1`
&tpl=`@INLINE <p>{$id} - {$pagetitle} {$createdon | date_format:"%Y-%m-%d %H:%M:%S" | dateago}</p>`
    ]]

If the desired modifier is not found, then the variable will remain as it is, raw, and you get a record of this in the error log.

#### built modifiers

At this point you can use the following modifiers Fenom (in parentheses are the equivalent synonyms):

-   **date\_format** - the formatting of the date of the function `strftime`. If the variable is empty, you will receive the current time.

{'2015-01-10 12:45' | date_format : '%d.%m.%Y'} // 10.01.2015
{''                 | date_format : '%Y'} // текущий год     

-   **date** - the formatting of the date of the function `date`. If the variable is empty, you will receive the current time.

{'2015-01-10 12:45' | date : 'd.m.Y'} // 10.01.2015
{''                 | date : 'Y'} // текущий год   

-   **the escape (an e)** - the screening variable. The first parameter takes the mode of operation, the second - encoding.

{'<p>value</p>'               | escape : 'html' : 'utf-8'} // &lt;p&gt;value&lt;/p&gt;   
{'http://site.com/?key=value' | escape : 'url'} // http%3A%2F%2Fsite.com%2F%3Fkey%3Dvalue
{['key' => 'value']           | escape : 'js'}                                           

-   **The unescape** - decoding variable

{'&lt;p&gt;value&lt;/p&gt;'               | unescape : 'html' : 'utf-8'} // <p>value</p>   
{'http%3A%2F%2Fsite.com%2F%3Fkey%3Dvalue' | unescape : 'url'} // http://site.com/?key=value
{'{"key":"value"}'                        | unescape : 'js'}                               

-   **The truncate (ellipsis)** - Cuts the variable to a certain length, the default - 80 characters. As an optional second parameter, you can pass a string of text, which will be displayed at the end of the trimmed variable. This line characters are not included in the overall length to truncate. By default, truncate will attempt to cut the line in between words. If you want to truncate longer strictly to the instructions, pass an optional third parameter to true.

        {var $str = 'very very long string'}
{$str | truncate : 10 : ' read more...'} // very very read more...
{$str | truncate : 5 : ' ... ' : true : true} // very ... string  

-   **strip** - removes spaces at the beginning and end of the text. If the optional parameter is removed and all duplicate spaces in the text.

{'text     with      spaces' | strip : true} // text with spaces

-   **length (len, strlen)** - outputs the variable length. It may take a string or an array.

{'var'              | length} // 3
{['key' => 'value'] | len} // 1   

-   **in** - the operator check the substring or values in the array

        {var $key = '10'}
        // строка
{if $key | in : 'У меня 10 яблок'}
        Ключ найден
        {/if}
        // массив + тернарный оператор
{$key | in : [1, 3, 42] ? 'ключ найден' : 'не найден'}

-   **iterable** - is it possible to iterate through a variable

        {var $array = ['key' => 'value']}
{if $array | iterable}
        {foreach $array as $key => $value}
            <p>{$key} is {$value}</p>
        {/foreach}
        {/if}

-   **the replace** - replaces all occurrences of the search string in the replacement string

{$fruits | replace : "pear" : "orange"} // все pear будут заменены на orange

-   **ereplace** - performs search and replace regular expression.

{'April 15, 2014' | ereplace : '/(\w+) (\d+), (\d+)/i' : '${1}1, $3'} // April1, 2014

-   **the match** - verify that the line with the template of the function `fnmatch`. You can use a simple Mask:

{if $color | match : '*gr[ae]y'}
        какой-то оттенок серого
        {/if}

-   **ematch** - Perform a regular expression

{if $color | ematch : '/^(.*?)gr[ae]y$/i'}
        какой-то оттенок серого
        {/if}

-   **split** - splits the string and returns an array using the first parameter as a separator (default `,`).

{var $fruits = 'banana,apple,pear' | split} // ["banana", "apple", "pear"]   
{var $fruits = 'banana,apple,pear' | split : ',apple,'} // ["banana", "pear"]

-   **esplit** splits a string for a regular expression in the first parameter (the default `/,\s*/`).

{var $fruits = 'banana, apple, pear' | esplit} // ["banana", "apple", "pear"]          
{var $fruits = 'banana; apple; pear' | esplit : '/;\s/'} // ["banana", "apple", "pear"]

-   **join** - combines the elements of the array into a string using the first parameter as a connector (default `,`).

        {var $fruits1 = ["banana", "apple", "pear"]}
{$fruits1 | join} // banana, apple, pear                       
{$fruits1 | join : 'is not'} // banana is not apple is not pear

-   **number (number\_format)** - Formatting of php function`number_format()`

        {var $number = 10000}
{$number | number : 0 : '.' : ' '} // выведет 10 000

-   **the md5, sha1, the crc32** - count checksum line different algorithms

{'text' | md5} // 1cb251ec0d568de6a929b520c4aed8d1         
{'text' | sha1} // 372ea08cab33e71c02c651dbc83a474d32c676ea
{'text' | crc32} // 999008199                              

-   **urldecode, urlencode, rawurldecode** - Processing PHP function of the corresponding line.

{'http://site.com/?key=value'             | urlencode} // http%3A%2F%2Fsite.com%2F%3Fkey%3Dvalue
{'http%3A%2F%2Fsite.com%2F%3Fkey%3Dvalue' | urldecode} // http://site.com/?key=value            

-   **base64\_decode, base64\_encode** - encoding and decoding algorithm base line 64

{'text'     | base64_encode} // dGV4dA==
{'dGV4dA==' | base64_decode} // text    

-   **json\_encode (toJSON), json\_decode (fromJSON)**

{'{"key":"value"}'  | fromJSON} // ["key" => "value"]
{["key" => "value"] | toJSON} // {"key":"value"}     

-   **http\_build\_query** - generiratsiya URL-encoded string from an array

{['foo'=>'bar','baz'=>'boom'] | http_build_query} // foo=bar&baz=boom

-   **print\_r** - printing variable via PHP function. If given an option, you can save the result.

{var $printed = (['key' => 'value'] | print_r : true)} // Array ( [key] => value )

-   **var\_dump (The dump)** - printing with a variable type
-   **nl2br** - insert HTML-code for a line break before each newline

-   **lower (low, lcase, lowercase, strtolower)** - linefeed lowercase

{'Текст для проверки' | lower} // текст для проверки

-   **upper (up, ucase, uppercase, strtoupper)** - linefeed uppercase

{'Текст для проверки' | upper} // ТЕКСТ ДЛЯ ПРОВЕРКИ

-   **ucwords** - convert to upper case the first character of each word in the string, and the remaining characters to lower

{'теКст Для провЕрки' | ucwords} // Текст Для Проверки

-   **ucfirst** - convert to upper case the first character of the first word in the string, and the remaining characters to lower

{'теКст Для провЕрки' | ucfirst} // Текст для проверки

-   **htmlent (htmlentities)** - converts all possible characters to HTML-entities

{'<b>Текст для проверки</b>' | htmlent} // &lt;b&gt;Текст для проверки&lt;/b&gt;gt;

-   **limit** - trimming line to the specified number of characters

{'Привет, как дела' | limit : 6} // Привет

-   **esc (tag)** - shielded HTML and MODX tags

{'Привет, [[+username]]' | esc} // тег username не будет обработан

-   **notags (striptags, stripTags, strip\_tags)** - removal of HTML tags from line

{'<b>Привет!</b>' | notags} // Привет!

-   **stripmodxtags** - MODX removing tags from a string

{'Привет, [[+username]]' | stripmodxtags} // Привет,

-   **cdata** - wraps conclusion tags CDATA. Usually it is necessary to shield the values in the XML markup.
-   **reverse (strrev)** - turns the string character by character

{'mirrortext' | reverse} // txetrorrim

-   **wordwrap** - inserts a line break after each n-th character (words are not broken)

{'very very long text' | wordwrap : 3} // very<br />very<br />long<br />text

-   **wordwrapcut** - inserts a line break after each n-th character, even if the character is within a word

{'very very long text' | wordwrapcut : 3} // ver<br />y<br />ver<br />y<br />lon<br />g<br />tex<br />t

-   **fuzzydate** - displays the date with respect to yesterday and today, using the system dictionaries. If it has been more than 2 days, date formats through `strftime`.

{'2016-03-20 13:15' | fuzzydate : '%d.%m.%Y'} // 20.03.2016
{''                 | date : 'Y-m-d 13:15'                 

-   **declension (decl)** - declines word following a number of rules of the Russian language. For example: 1 apple, 2 apples, 10 apples. The second parameter specifies whether to display the number itself, by default, only the appropriate version of the word. Separator options, you can set the third parameter, the default `|`.

{6   | declension : 'яблоко                                  | яблока | яблок'} // яблок          
{3   | declension : 'яблоко                                  | яблока | яблок' : true} // 3 яблока
{101 | decl : 'яблоко,яблока,яблок' : false : ','} // яблоко |        |                           

-   **ismember (memberof, mo)** - verification of a user's group or groups MODX users. If the variable is empty, the check is performed for the current user.

{1 | ismember : 'Administrator'} // true              
{0 | ismember : ['Administrator', 'Manager']} // false

-   **isLoggedIn, isnotloggedin** - check the authorization of the current user in kontekte. If the context is not specified, then the current is checked.

{'' | isloggedin : 'web'} // true    
{'' | isnotloggedin : 'web'} // false

-   **url No** - generation of links to the document through the system `modX::makeUrl`.

{10 | url}                                              
{10 | url : ['scheme' => 'full'] : ['param' => 'value']}

-   **lexicon** - output record from the system through the dictionaries `modX::lexicon`. Some of the dictionaries is first necessary to download.

        {$_modx->lexicon->load('en:core:default')}
{'action_err_nfs' | lexicon : ['id' => 25] : 'en'} // No action with 25 found.

-   **user (userinfo)** - output field system user

{1 | user : 'fullname'} // Administrator

-   **resource** - the conclusion of the field document system. You can display and TV settings

{1 | resource : 'longtitle'} // Home page
{1 | resource : 'my_tv'} // My_tv value  

-   **print** - print function and screening of any variables. Very useful for debugging when you need to find out what is inside. The only parameter specifies whether to wrap the data tag is necessary `<pre>`(default - yes).

{1 | resource          | print} // распечатает массив с данными всего ресурса                                    
{0 | user : 'extended' | print : false} // распечатает массив поля `extended` текущего пользователя в одну строку

-   **setOption** - exposure values in an array `modX::config`. Be sure to specify the key values in the array.

{'Новое имя сайта' | setOption : 'site_name'}

-   **option (getOption)** - obtaining the value of the array`modX::config`

{'site_name' | option} // Новое имя сайта

-   **setPlaceholder (toPlaceholder)** - exposure value in the array `modX::placeholders`. Be sure to specify the key values in the array.

{'My text' | setPlaceholder : 'my_text'}

-   **placeholder (fromPlaceholder)** - obtaining the value of the array `modX::placeholders`.

{'my_text' | placeholder} // My text

-   **cssToHead** - the CSS code is registered in the page header
-   **htmlToHead** - registration arbitrary HTML in the page header
-   **htmlToBottom** - registration arbitrary HTML in the footer
-   **jsToHead** - javascript file registration in the page header. If you pass a parameter `true`, it is possible to register the code at once.
-   **jsToBottom** - javascript registration in the footer. If you pass a parameter `true`, it is possible to register the code at once.

{'<script>alert();</script>' | jsToBottom : true} // При загрузке страницы будет javascript alert

    *for registering functions on the page must have the tags `head`and`body`*

#### PCRE modifiers

-   **preg\_quote** - adds a backslash before each official symbol. This is useful if you are involved in drawing up the template string variables, which are important in the process of the script can be changed

    in regular expressions official considered following symbols

. \ + * ? [ ^ ] $ ( ) { } = ! < > | : -

-   **preg\_match** - Perform a regular expression

    For example, display *email* when matches the regular expression:

{if 'email@mail.ru' | preg_match : '/^.+@.+\\.\\w+$/' }
        email
        {/if}

-   **preg\_get** - performs a regular expression search result includes a first set of matches

    for example, we obtain a first image of the field *content* :

{set $image = $_modx->resource.content | preg_get : '!http://.+\.(?:jpe?g | png | gif)!Ui}

-   **preg\_get\_all** - searches for a regular expression, the result set contains all occurrences
-   **preg\_grep** - Return array entries that match the pattern

    for example, only the *date* :

{["26-04-1974", "Сергей", "27-11-1977", "Юля"] | preg_grep : "/(\d{2})-(\d{2})-(\d{4})/" | print_r : true}

-   **preg\_replace** - performs search and replace by regular expression

    For example, display the name of the site without the *http* :

Website: {'http://site.name' | preg_replace : '~^https?://~': ''}

-   **preg\_filter** - performs search and replace by regular expression

    identical to *preg\_replace* except that it returns only those values which match is found

-   **preg\_split** - Split string by a regular expression

    eg:

{'I love MODX' | preg_split : '/ /' | print_r : true}// Array ( [0] => I [1] => love [2] => MODX )

#### own modifiers

Snippet filter should take 2 parameters: required `$input`and optional `$options`. He can only return line.

As an example, let's create a snippet **randomize** , which will take the resulting string and add to it a random set of characters. The parameter we can pass the number of characters to be added:

    <?php
    if (empty($options)) {
        $options = 10;
    }
    $rand = md5(rand());

    return $input . ' ' . substr($rand, 0, $options);

Now you can safely use the new modifier in Fenom

{'any text' | randomize : 5}

and in MODX

    [[!*pagetitle:randimize=`5`]]

Starting with version **2.6.0** , to add modifiers, you can use the system event **pdoToolsOnFenomInit** :

    <?php
    /** @var modX $modx */
    switch ($modx->event->name) {
        case 'pdoToolsOnFenomInit':
            /** @var Fenom $fenom
            Мы получаем переменную $fenom при его первой инициализации, так что можем добавить
            модификатор "website" для вывода имени домена из произвольной ссылки.
            */
            $fenom->addModifier('website', function ($input) {
                if (!$url = parse_url($input)) {
                    return $input;
                }
                $output = str_replace('www.', '', $url['host']);

                return strtolower($output);
            });
            break;
    }

Now modifier *website* can be used in any chunks and templates Fenom:

<a href="{$link}" target="blank">{$link | website}</a>

Such modifiers are internal methods Fenom snippets and work faster because they do not need to make a request to the database to be retrieved, and then cache the code in the file to run.

### Expanding templates

Using a template allows Fenom include some chunks (templates etc.) and even expand them.

For example, you can simply load the contents of the chunk:

    Обычный чанк {include 'имя чанка'}
    Шаблон modTemplate {include 'template:имя шаблона'}
    Чанк с набором параметров
    {include 'chunk@propertySet'}
    {include 'template:Name@propertySet'}

Learn more about { [the include](https://github.com/fenom-template/fenom/blob/master/docs/ru/tags/include.md) }, see the official documentation.

Much more interesting feature - { [the extends](https://github.com/fenom-template/fenom/blob/master/docs/ru/tags/extends.md) } template, it requires a power-on settings **pdotools\_fenom\_parser** .

Writing basic template "Fenom Base":

    <!DOCTYPE html>
    <html lang="en">
    <head>
        {include 'head'}
    </head>
    <body>
        {block 'navbar'}
            {include 'navbar'}
        {/block}
        <div class="container">
            <div class="row">
                <div class="col-md-10">
                    {block 'content'}
                        {$_modx->resource.content}
                    {/block}
                </div>
                <div class="col-md-2">
                    {block 'sidebar'}
                        Sidebar
                    {/block}
                </div>
            </div>
            {block 'footer'}
                    {include 'footer'}
            {/block}
        </div>
    </body>
    </html>

It includes conventional chunks (which, incidentally, by conventional placeholders MODX component **Theme.Bootstrap** ) and defines a number of blocks `{block}`which can be expanded in another pattern.

Now write "Fenom Extended":

    {extends 'template:Fenom Base'}

    {block 'content'}
        <h3>{$_modx->resource.pagetitle}</h3>
        <div class="jumbotron">
            {parent}
        </div>
    {/block}

So you can write a basic template and extend its subsidiaries.

Similarly, you can write and expand chunks, just note that to work with modTemplate need the prefix **: template:** , and for no chunks - they work by default in all `{include}`and `{extends}`.

> **Important!** When inheriting Fenom requires mandatory advertisement of at least one block in the template. Those. just inherit the basic template string `{extends 'template:Fenom Base'}`can not be 502 error at the level of PHP. Enough to declare a block or any override of a base:
>
>     {extends 'template:Fenom Base'}
>     {block 'empty'}{/block}

### performance testing

Create a new site and add the 1000 resource like this script Console:

    <?php
    define('MODX_API_MODE', true);
    require 'index.php';

    $modx->getService('error','error.modError');
    $modx->setLogLevel(modX::LOG_LEVEL_FATAL);
    $modx->setLogTarget(XPDO_CLI_MODE ? 'ECHO' : 'HTML');

    for ($i = 1; $i <= 1000; $i++) {
        $modx->runProcessor('resource/create', array(
            'parent' => 1,
            'pagetitle' => 'page_' . rand(),
            'template' => 1,
            'published' => 1,
        ));
    }

Then create Chunk 2: `modx`and `fenom`with the following contents, respectively:

    <p>[[+id]] - [[+pagetitle]]</p>

and

    <p>{$id} - {$pagetitle}</p>

And add the two console test script. For native parser MODX

    <?php
    define('MODX_API_MODE', true);
    require 'index.php';

    $modx->getService('error','error.modError');
    $modx->setLogLevel(modX::LOG_LEVEL_FATAL);
    $modx->setLogTarget(XPDO_CLI_MODE ? 'ECHO' : 'HTML');

    $res = array();
    $c = $modx->newQuery('modResource');
    $c->select($modx->getSelectColumns('modResource'));
    $c->limit(10);
    if ($c->prepare() && $c->stmt->execute()) {
        while ($row = $c->stmt->fetch(PDO::FETCH_ASSOC)) {
            $res .= $modx->getChunk('modx', $row);
        }
    }
    echo number_format(microtime(true) - $modx->startTime, 4), 's<br>';
    echo number_format(memory_get_usage() / 1048576, 4), 'mb<br>';
    echo $res;

And pdoTools:

```php
<?php
define('MODX_API_MODE', true);
require 'index.php';

$modx->getService('error','error.modError');
$modx->setLogLevel(modX::LOG_LEVEL_FATAL);
$modx->setLogTarget(XPDO_CLI_MODE ? 'ECHO' : 'HTML');
$pdoTools = $modx->getService('pdoTools');

$res = array();
$c = $modx->newQuery('modResource');
$c->select($modx->getSelectColumns('modResource'));
$c->limit(10);
if ($c->prepare() && $c->stmt->execute()) {
    while ($row = $c->stmt->fetch(PDO::FETCH_ASSOC)) {
        $res .= $pdoTools->getChunk('fenom', $row);
        //$res .= $pdoTools->getChunk('modx', $row);
    }
}
echo number_format(microtime(true) - $modx->startTime, 4), 's<br>';
echo number_format(memory_get_usage() / 1048576, 4), 'mb<br>';
echo $res;
```

Since pdoTools understands both the syntax 2 test for it - in MODX tag mode and Fenom mode. The script is an indication limit = 10, on the table I present figures with his increase:

| Limit | MODX             | pdoTools (MODX)    | pdoTools (Fenom) |
| ----- | ---------------- | ------------------ | ---------------- |
| 10    | 0.0369s 8.1973mb | 0.0136s 7.6760mb   | 0.0343s 8.6503mb |
| 100   | 0.0805s 8.1996mb | 0.0501s 7.6783mb   | 0.0489s 8.6525mb |
| 500   | 0.2498s 8.2101mb | 0.0852s ​​7.6888mb | 0.0573s 8.6630mb |
| 1000  | 0.4961s 8.2232mb | 0.1583s 7.7019mb   | 0.0953s 8.6761mb |

Now, let's complicate chunks - add them to generate links to the resource and output `menutitle`:

    <p><a href="[[~[[+id]]]]">[[+id]] - [[+menutitle:default=`[[+pagetitle]]`]]</a></p>

and

    <p><a href="{$_modx->makeUrl($id)}">{$id} - {$menutitle ?: $pagetitle}</a></p>

| Limit | MODX             | pdoTools (MODX)  | pdoTools (Fenom) |
| ----- | ---------------- | ---------------- | ---------------- |
| 10    | 0.0592s 8.2010mb | 0.0165s 7.8505mb | 0.0346s 8.6539mb |
| 100   | 0.1936s 8.2058mb | 0.0793s 7.8553mb | 0.0483s 8.6588mb |
| 500   | 0.3313s 8.2281mb | 0.2465s 7.8776mb | 0.0686s 8.6811mb |
| 1000  | 0.6073s 8.2560mb | 0.4733s 7.9055mb | 0.1047s 8.7090mb |

As you can see, through the processing of chunks pdoTools in all cases faster. At the same time much that chunks Fenom have some minimum to start, which is due to the need to compile template.

## Disconnection

Parser pdoTools well tested and is not seen in any errors, but if you're in something of his suspect - with version 2.8.1, you can easily incorporate native parser MODX.

To do this, you need to change to change exactly one system configuration:

-   **parser\_class** - enter into it **modParser** .

To enable pdoParser need to specify its name back in this setting.

## Conclusion

Let's sum up the possibility parser pdoTools:

-   Quick work
-   Support fastField tags
-   Support templating Fenom
-   -   template inheritance

-   -   Expanding templates

-   -   Secure access to the advanced functions MODX

At this point pdoTools downloaded more than 114,000 times from [the official repository](http://modx.com/extras/package/pdotools) and more than 25 000 of [the repository modstore.pro](https://modstore.pro/pdotools) , which gives hope for widespread standardization of new technologies in the MODX.

[0]: https://rtfm.modx.com/revolution/2.x/administering-your-site/upgrading-modx/upgrading-to-2.2.x#Upgradingto2.2.x-StaticElements
[1]: http://modx.com/extras/package/fastfield
[2]: https://github.com/argnist/fastField/issues/5
[3]: https://rtfm.modx.com/extras/revo/renderresources
[4]: http://habrahabr.ru/post/161843/
[5]: https://github.com/fenom-template/fenom
[6]: https://modx.pro/components/5598-pdotools-2-0-0-beta-c-templating-fenom/
[7]: https://github.com/fenom-template/fenom/blob/master/docs/en/readme.md
[8]: https://github.com/fenom-template/fenom/blob/master/docs/en/tags/ignore.md
[9]: http://php.net/manual/ru/language.variables.basics.php
[10]: https://github.com/bezumkin/pdoTools/blob/master/core/components/pdotools/model/pdotools/_micromodx.php
