The main component is a class that inherits all the others (except pdoParser - it inherits modParser).

## Initialization

Simple class initialization:

```php
$pdoTools = $modx->getService('pdoTools');
```

This method always returns the original pdoTools.

You can specify a different path to the class in the system settings to replace it then initialize it in a better way:

```php
$fqn = $modx->getOption('pdoTools.class', null, 'pdotools.pdotools', true);
if ($pdoClass = $modx->loadClass($fqn, '', false, true)) {
    $pdoTools = new $pdoClass($modx, $scriptProperties);
}
elseif ($pdoClass = $modx->loadClass($fqn, MODX_CORE_PATH . 'components/pdotools/model/', false, true)) {
    $pdoTools = new $pdoClass($modx, $scriptProperties);
}
else {
    $modx->log(modX::LOG_LEVEL_ERROR, 'Could not load pdoTools from "MODX_CORE_PATH/components/pdotools/model/".');
    return false;
}
$pdoTools->addTime('pdoTools loaded');
```

This initialization method is used in all the snippets of pdoTools, so you can change the functionality and test a new version pdoTools.

## Logging

An important feature of pdoTools - it knows how to keep a log of what happens.  To do this, you have access to the following methods:

* **addTime** (: string $ message) - add a new entry in the log
* **getTime** (bool $: string [to true]) - adds a final time and returns a formatted string (the default) or an array of [time => Message]

For example, here's the code:

```php
$pdo = $modx->getService('pdoTools');
$pdo->addTime('pdoTools loaded');
print_r($pdo->getTime());
```

```
Outputs:

0.0000150: pdoTools loaded
0.0000272: Total time
1 572 864: Memory usage
```

Every pdoTools snippet logs events which can be viewed in the manager by adding the **&showlog = \`1\`** option.

## Caching

pdoTools able to cache data during script execution. You too can enjoy this.

* **setStore** (: string $name, mixed $object, $: string of the type [ "data"]) - add any data to temporary storage.
* **getStore** (: string $name, $: string of the type [ "data"]) - or receives data or null.

For example, during the execution of the snippet a developer may need to cache the names of users, so they don't need to select them every time. The developer can check whether the correct user in the cache and if not - retreive it:

```php
foreach ($users as $id) {
    $user = $pdo->getStore($id, 'user')
    if ($user === null) {
        if (!$user = $modx->getObject('modUser', $id)) {
            $user = false;
        }
        $pdo->setStore($id, $user, 'user');
    }
    elseif ($user === false) {
        echo 'User not found id = ' . $id;
    }
    else {
        echo $user->get('username');
    }
}
```

In this code, we reserve the namespace for users, so as not to interfere with other snippets, and check the presence of the user in the cache. Note that the cached value can be returned, `null` (the user has not yet obtained), or `false` (user was not found). In any case, the database query will be executed only once for each user.

pdoTools caches chunks. The data is stored only for the duration of the script, that is, they are not written to the hard disk.

There are also more advanced caching methods in MODX:

* **setCache** (mixed $ data, $options of array) - saves the data `$data` in the cache, generating a key from `$options`
* **getCache** (array $ options) - outputs data according `$options`

Here, the data already stored on the disk cache time can be passed in an array of parameters:

```php
$pdo = $modx->getService('pdoTools');
$options = array(
    'user' => $modx->user->get('id'),
    'page' => @$_REQUEST['page'],
    'cacheTime' => 10,
);
$pdo->addTime('pdoTools loaded');
if (!$data = $pdo->getCache($options)) {
    $pdo->addTime('Generate data');
    $data = array();
    for ($i = 1; $i <= 100000; $i ++) {
        $data[] = rand();
    }
    $data = md5(implode($data));
    $pdo->setCache($data, $options);
    $pdo->addTime('Cache is set');
}
else {
    $pdo->addTime('Data retrieved from cache');
}
print_r($data);
```

Thus, any data stored in the cache will be obtained, depending on the user and pages.

The first time our code runs it will show roughly the following:

```
0.0000281: pdoTools loaded
0.0004001: No cached data for key "default/e713939a1827e7934ff0242361c06b4b10c53d97"
0.0000079: Generate data
0.0581820: Saved data to cache "default/e713939a1827e7934ff0242361c06b4b10c53d97"
0.0000181: Cache is set
0.0586412: Total time
1 835 008: Memory usage
```

The following times:

```
0.0000310: pdoTools загружен
0.0007479: Retrieved data from cache "default/e713939a1827e7934ff0242361c06b4b10c53d97"
0.0000081: Data retrieved from cache
0.0007918: Total time
1 572 864: Memory usage
```

As you can see, pdoTools wrote the cache handling in the log, so that you can see how it was handled.

## Utilities

There are only two methods.

* **&makePlaceholders** (array $ data, string $ plPrefix, string $ prefix [ '[[+'], string $ suffix [ ']]'], bool $ uncacheable [true]) takes an array of key => value and returns two arrays placeholders = > value is used for parsing.

The first option - an array of data, then you can specify a prefix for the placeholder, the opening and closing characters, as well as to disable the generation of non-cached placeholders.

```php
$data = array(
    'key1' => 'value1',
    'key2' => 'value2',
);

$pls = $pdo->makePlaceholders($data);
print_r($pls);
```

Result:

```
Array
(
    [pl] => Array
        (
            [key1] => [[+key1]]
            [!key1] => [[!+key1]]
            [key2] => [[+key2]]
            [!key2] => [[!+key2]]
        )

    [vl] => Array
        (
            [key1] => value1
            [!key1] => value1
            [key2] => value2
            [!key2] => value2
        )

)
```

Then you can handle some html templates like this:

```php
$html = str_replace($pls['pl'], $pls['vl'], $html);
```

* **buildTree** ($ of array resources) builds a hierarchical tree of the resources in the array.  It is used by pdoMenu.

```php
$pdo = $modx->getService('pdoFetch');
$resources = $pdo->getCollection('modResource');
$tree = $pdo->buildTree($resources);
print_r($tree);
```

You will see a tree of resources in your site. Note the use of `getCollection()` to download resources using `pdoFetch`.

## Parsing (work with chunks)

This is probably the most interesting part of the class pdoTools.

The method here is only one - `getChunk()`, but it provides enhanced performance and functionality when parsing chunks.

All placeholders in chunks can be handled by `pdoParser` with one condition - placeholders must be without conditions and filters. I.e:

* `[[% Tag]]` - string lexicon
* `[[~ Id]]` - link
* `[[+ Tag]]` - the usual placeholders
* `[[++ tag]]` - the system placeholders
* `[[* Tag]]` - placeholders resource

`getChunk()` in pdoTools is able to work with different types of chunks:

* **@INLINE** , **@CODE** - chunk is created from the resulting string.
* **@FILE** - chunk is obtained from the file. To exclude the injections, the files can only be with the extension html and tpl, and the directory for their sample is given by the system settings pdotools_elements_path .
* **@TEMPLATE** - chunk is created from a site template, you can specify its id or name.
* **@CHUNK** or just a string without @ prefix - the conventional format for chunks in the database.

Working Example:

```php
$tpl = '@INLINE <p>[[+param]] - [[+value]]</p>';
$res = '';
for ($i = 1; $i <= 10000; $i++) {
    $pls = array('param' =>$i, 'value' => rand());
    $res .= $pdo->getChunk($tpl, $pls);
}
print_r($pdo->getTime());
print_r($res);
```

Here you have a simple templating using pdoTools.

This code displays the 10,000 lines in just 0.17 seconds! And, no matter what chunk code is used @INLINE, usually works with the same speed. And if you replace `$pdo->getChunk()` with `$modx->getChunk()`, it is published in 8 seconds!

In this particular example, parsing chunks with MODX is slower than pdoTools by 3000 times - 8 seconds, against 0.17. This suggests that developers benefit by simplifying chunks when using pdoTools.

The simplest "empty \ is not empty" can be replaced by the "fast placeholder". It works like this:

The chunk to be any tag, e.g. `[[+tag]]`.
The chunk must be a special html comments such as:

```html
<!--pdotools_tag [[+tag]] - The tag value is known-->
<!--pdotools_!tag The tag value is empty-->
```

As you can see, the comment is formatted with the prefix `pdotools_` and the tag name. Prefix can be changed using the parameter **&nestedChunkPrefix = \`df_\`** .

Why keep a quick placeholder in the comments?  Very simply so that it does not conflict with `modParser`.

Example:

```
$tpl = '@INLINE
    <p>[[+tag]]</p>
    <!--pdotools_tag [[+tag]] - The tag value is known-->
    <!--pdotools_!tag The tag value is empty-->
';

$pls = array('tag' => 1);
echo $pdo->getChunk($tpl, $pls);

$pls = array('tag' => 0);
echo $pdo->getChunk($tpl, $pls);
```

get

```
1 - The tag value is known

The tag value is empty
```

As you can see, the placeholders can be inserted quickly with its original value. Of course, this is minimal functionality when compared with MODX filters, but it is very fast.

There is another interesting parameter for placeholders processing - **&fastMode** . This setting turns off the passing of placeholders to the MODX native parser and if pdoParser can't handle the placeholder it will simply be ignored.

The latest versions of pdoTools do not require the use of **&fastMode**  because if pdoParser has processed all the placeholders like `[[+ tag]]`, it immediately returns the result without using `modParser`.  You can enable it as a mandatory requirement so that developers will be forced to not fallback to `modParser`.

With version 2.0, pdoTools includes in its composition template [Fenom][0], potentially eliminating the need for MODX tags and enabling developers to write chunks with more advanced logic.  For details about using Fenom see the [pdoParser][1] documentation.

[0]: https://github.com/fenom-template/fenom/tree/master/docs/ru#readme
[1]: /en/01_Components/01_pdoTools/03_Parser