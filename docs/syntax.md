# How to Write Custom Syntax

PostCSS can transform not only CSS. You can write a custom syntax
and transform styles in other format.

Writing a custom syntax is much harder rather then writing just
a PostCSS plugin. But it is a awesome adventure.

You can provide 3 types of package:

* **Parser** to parse input string to node’s tree.
* **Stringifier** to generate output string by node’s tree.
* **Syntax** contains parser and stringifier.

## Syntax

Good example of syntax is [SCSS]. Some users may want to change properties order
or add prefixes directly to SCSS sources. So we need SCSS input and output.

Syntax API is very simple. It is just a object with `parse` and `stringify`
functions:

```js
module.exports = {
    parse:     require('./parse'),
    stringify: require('./stringify')
};
```

[SCSS]: https://github.com/postcss/postcss-scss

## Parser

Main example of parse is [Safe Parser], to parse broken CSS. There is not sense
to generate string with broken CSS, so package should provide only parser.

Parser API is a function, that receives string and returns `Root` node.
The second function argument will be a object with PostCSS options.

```js
var postcss = require('postcss');

module.exports = function (css, opts) {
    var root = postcss.root();
    // Some magic with css
    return root;
};
```

[Safe Parser]: https://github.com/postcss/postcss-safe-parser

### Main Theory

There are many books about parses. But do not scary, CSS syntax is very easy,
so parser will be much simpler than usual.

The default PostCSS parser contains two steps:

* [Tokenizer] to read input string char by char and build tokens array.
  For example, it join spaces symbols to `['space', '\n  ']` token,
  or detect strings to `['string', '"\"{"']`.
* [Parser] to read tokens array, create node instances and build tree.

Look at this parts for a parser example.

[Tokenizer]: https://github.com/postcss/postcss/blob/master/lib/tokenize.es6
[Parser]:    https://github.com/postcss/postcss/blob/master/lib/parser.es6

### Performance

Parsing input is a longest task in CSS usual processor. So it is very important
to have fast parser.

The main rule of optimization: there is no performance without benchmark.
You can look at [PostCSS benchmarks] as example.

If we will look deeper, the tokenize step will be a longest part of parsing.
So you should focus only on tokenizer performance. Other parts could be
well written maintainable code.

Unfortunately, classes, functions and high level structures can slow down
your tokenizer. Be ready to write dirty code with code repeating.
This is why it is difficult to extend default tokenizer.
Copy paste will be a necessary evil.

Second optimization is using char codes instead of string.

```js
// Slow
string[i] === '{';

// Fast
const OPEN_CURLY = 123; // `{'
string.charCodeAt(i) === OPEN_CURLY;
```

Third optimization is “fast jumps”. If you find open quote, you can find
closed quote much faster by `indexOf`:

```js
// Simple jump
next = string.indexOf('"', currentPosition + 1);

// Complicated jump
regexp.lastIndex = currentPosion + 1;
regexp.text(string);
next = regexp.lastIndex;
```

Parser can be well written class. So there is no need in copy-paste and
hardcore optimization there. You can safely extend default class.

[PostCSS benchmarks]: https://github.com/postcss/benchmark

### Node Source

To generate source map, every node instance should have `source` property.
It should have `{ line, column }` object in `start` and `end` properties
and `input` property with [`Input`] instance.

So your tokenizer should provide token type, content and positions.

[`Input`]: https://github.com/postcss/postcss/blob/master/lib/input.es6

### Raw Values

Good PostCSS parser should provide all information (including spaces symbols)
to generate byte-to-byte equal output. It is not so difficult, but respectful
for user input and allow integration smoke tests.

Parser should save all addition symbols to [`node.raws`] object.
It is a open structure to you, you can add addition keys.
For example, [SCSS parser] saves comment types there (`/* */` or `//` comment).

Also, default parser clean some CSS values from comments and spaces.
It save origin value with comments to `node.raws.value.raw` and use it,
if node values were not changed.

[`node.raws`]: https://github.com/postcss/postcss/blob/master/docs/api.md#node-raws

### Tests

Of course, all parsers must have a tests.

If your parser just extend CSS syntax (like [SCSS] or [Safe Parser]),
you should use [PostCSS Parser Tests]. It contains unit and integration tests.

[PostCSS Parser Tests]: https://github.com/postcss/postcss-parser-tests

## Stringifier

Style guide generator is good example of stringifier. It generates output HTML
with usage examples by component styles.

Stringifier API is little bit more complicated, that parser API.
PostCSS generates source map. So stringifier can’t just return a string.
It must link every string with source node.

Stringifier is a function, that receives `Root` node and builder callback.
Then it calls builder with every node’s string and this node.

```js
module.export = function (root, builder) {
    // Some magic
    var string = decl.prop + ':' + decl.value + ';';
    builder(string, decl);
    // Some science
};
```

### Main Theory

PostCSS [default stringifier] is just a class with method for each node type
and many methods to detect raw properties (if node was built manually)
by other nodes.

In most cases it will be enough just to extent this class,
like in [SCSS stringifier].

[default stringifier]: https://github.com/postcss/postcss/blob/master/lib/stringifier.es6
[SCSS stringifier]:    https://github.com/postcss/postcss-scss/blob/master/lib/scss-stringifier.es6

### Builder Function

Builder function will be pass to `stringify` function as second argument.
Stringifier class saves it to `this.builder` property.

It receive output substring and source node and accept this substring
to final output.

Some nodes string can contains other nodes in the middle.
For example, rule has `a {` beginning, many declarations inside
and `}` at the end.

For this cases, you should pass a third argument to builder function:
`'start'` or `'end'` string.

### Raw Values

Good PostCSS syntax saves all symbols and provide byte-to-byte equal output
if there were not AST changes.

This is why every node has [`node.raws`] object to store space symbol, etc.

But be ready, that any of this `raws` properties can be missed. Some nodes
can be built manually. Some nodes can loose indent on moving between parents.

This is why default stringifier has `raw()` method to autodetect raw property
by other nodes. For example, it will look at other nodes to detect indent size
and them multiply it with current node depth.

[`node.raws`]: https://github.com/postcss/postcss/blob/master/docs/api.md#node-raws

### Tests

Stringifier must have a tests too.

You can use unit and integration cases from [PostCSS Parser Tests]
in smoke test. Just compare input CSS with CSS after you parser and stringifier.

[PostCSS Parser Tests]: https://github.com/postcss/postcss-parser-tests