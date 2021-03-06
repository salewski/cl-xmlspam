XML Stream PAttern Matcher - concise, regexp-like pattern matching on streaming XML

[Cxml](http://common-lisp.net/project/cxml/) does an excellent job of parsing XML elements, but what do you do when you have a XML file that's larger than you want to fit in memory, and you want to extract some information from it? Writing code to deal with SAX events, or even using [Klacks](http://common-lisp.net/project/cxml/klacks.html) quickly becomes tedious. Cl-xmlspam is designed to make it easy to write code that mirrors the structure of the XML that it's parsing. It also makes it easy to shift paradigms when necessary - the usual Lisp control constructs can be used interchangeably with pattern matching, and the full power of CXML is available when necessary.

It is based on a non-backtracking regular-expression-like parser.  It
provides a compact syntax for matching, borrowing terminology from [RELAX-NG](http://www.lichteblau.com/cxml-rng/), and also for regular-expression-based parsing of text elements, inspired by Rob Pike's [structural regular expressions](http://doc.cat-v.org/bell_labs/structural_regexps/se.pdf).

Here is a simple example of its use:
```
* (with-xspam-source "&lt;?xml version=\"1.0\"?> &lt;a> &lt;b>hello there&lt;/b> &lt;b>goodbye&lt;/b>&lt;e/> &lt;p>345x675 234x754 786x532&lt;/p> &lt;q when=\"23-11-2008\"/>&lt;/a>"
    (element :a
      (group
        (one-or-more
          (element :b (text (matches "[^ ]+" (format t "word ~s~%" _)))))
        (element :p
          (text
            (matches "([0-9]+)x([0-9]+)"
              (format t "got point [~s ~s]~%" (_ 0) (_ 1)))))
        (element :q
          (attribute :when
            (match "([0-9]+)-([0-9]+)-([0-9]+)"
              (format t "got date ~s/~s/~s~%" (_ 0) (_ 1) (_ 2))))))))
word "hello"
word "there"
word "goodbye"
got point ["345" "675"]
got point ["234" "754"]
got point ["786" "532"]
got date "23"/"11"/"2008"
NIL
*
```

Some preliminary documentation can be found [here](documentation.md) and one or two examples [here](examples.txt).

Cl-xmlspam is currently in a young state - although it has been used to
do useful work on real-world XML data, it has so far only been tested under
SBCL.

API documentation
--------

Cl-xmlspam, "XML Stream PAttern Matcher"†, (also known as xspam) is an
adjunct to CXML to allow simple on-the-fly parsing of streaming XML
based on a non-backtracking regular- expression-like parser.  It
provides a compact syntax for this matching, and also for
regular-expression-based parsing of text elements, inspired by Rob
Pike's structural regular expressions. (see http://doc.cat-v.org/bell_labs/structural_regexps/se.pdf)

Xspam layers on top of the CXML Klacks parser, and is designed to
interoperate easily with other packages that use CXML.

It requires the following supporting packages:

	cl-ppcre
	cxml

The following symbols are exported from the xspam package:

`make-xspam-source (source &rest args)`

	Function. Creates a new xspam source. If source is already
	an xspam source, this just returns source; if it is an
	existing cxml klacks source, the xspam source layers on
	top of that; otherwise the arguments are exactly as
	accepted by cxml:make-source. For instance, source
	may be a string, giving literal XML, a pathname object,
	giving the path to a file containing XML or a stream.

	Returns the new source.

`with-xspam-source (source &body body)`

	Macro.  Creates a new xspam source and evaluates body (an
	implicit progn) making the source available to xspam macros
	within its lexical scope; source is as accepted by
	make-xspam-source.

	Returns the result of evaluating body.

`with-namespace (names &body body)`

	Macro.  Names is a form which is evaluated to get a list of
	(symbol . uri) pairs, where the name of each symbol specifies
	a prefix used within element and attribute names to refer to
	the namespace given by the string uri.  With-namespace
	evaluates the body form (an implicit progn); the namespace
	will be used within its lexical scope.
	With-namespace will be ignored unless it is used inside a with-xspam-source
	context.

	Returns the result of evaluating body.

`element (name &body body)`

	Macro.  Name is a designator for an XML element name (see
	below).  Element reads from the current xspam source at the
	top level within the current element until it finds an element with a matching
	name, whereupon it evaluates body (an implicit progn).  The
	current klacks context will be positioned at the
	:start-element of the element.  If no such element can be
	found, an error is raised.

	Returns the result of evaluating body, upon which the current
	klacks context will be positioned just after the :end-element
	of the matched element.

`attribute (name &body body)`
`optional-attribute (name &body body)`

	Macro.  Name is a designator for an XML attribute name (see below)  The
	current klacks context must be positioned at the start of an
	element.  If that element holds an attribute named name, then
	body (an implicit progn) is evaluated.  Within body, the
	variable _ gives the text of the attribute, and the current
	text is set accordingly. Attribute requires the element to hold
	the named attribute, and signals an error if it does not;
	whereas optional-attribute just ignores elements without the
	attribute.

	Returns the result of evaluating body.

`text (name &body body)`

	Macro.  Text reads from the current xspam source, within the
	current element, until it finds a non-empty text element,
	whereupon it evaluates body (an implicit progn).  Within body,
	the variable _ gives the text and that of all directly
	adjoining text elements, and the current text is set
	accordingly.

`match (regex &body body)`

	Macro. Regex is a form that is evaluated to give
	a cl-ppcre regular expression, which may be a simple
	string. If the current text matches this, then
	body (an implicit progn) is evaluated, with the current
	text set to the bounds of the match. Within body,
	the variable _ gives the matched text, and the function _
	is defined with one integer argument, say n,  to give the
	text of the nth (zero-based) subexpression of regex, or nil if that
	was not matched.

	Match raises an error if the regular expression does not contain
	a match.

	Returns the result of evaluating body, whereupon the
	current text is set to the original current text from the
	end of the match onwards.

`matches (regex &body body)`

	Macro. Similar to match, except that body
	is evaluated once for each match of regex within
	the current text, and no error is raised if there
	is no match.

	Returns nil.

`matches-not (regex &body body)`

	Macro. Similar to matches, except that body
	is evaluated once for every range of text
	*not* matched by regex.

	Returns nil.

`guard (regex &body body)`
`guard-not (regex &body body)`

	Macro. Guard (guard-not) evaluates body, an implicit progn, if the
	current text contains (does not contain) a match for regex. The current
	text is left unchanged.

	Returns the result of evaluating body, or NIL if body is not evaluated.

`one-of (&rest pattern)`
`zero-or-more (&rest pattern)`
`one-or-more (&rest pattern)`
`optional (&rest pattern)`
`group (&rest pattern)`

	Macros. Continuously read XML from the current
	source and match it against pattern, evaluating
	the bodies of element subforms as they are matched.
	No lookahead is performed - the first matching
	clause will be selected, even if that means that
	a subsequent clause that could have matched will fail.

	Pattern should conform to the following grammar:

		pattern ::= {
			one-of pattern {pattern}* |
			zero-or-more pattern |
			one-or-more pattern |
			optional pattern |
			group pattern {pattern}* |
			element name {form}* |
			text {form}* }

	where forms is an implicit progn, evaluated when the relevant clause
	is matched. Macros within pattern are expanded until one
	of the above forms is generated.

	The pattern names are those of RELAX NG, except one-of is used
	instead of interleave, to make it clear that there is no
	conjunction of subelements:

		one-of matches exactly one of its constituent patterns.
		zero-or-more matches zero or more repetitions.
		one-or-more matches one or more repetitions.
		optional matches one element, optionally.
		group matches each pattern in turn.
		element matches a single element, as for the element macro above.
		text matches non-empty text elements, as for the text macro above.


`xspam-context (&rest which)`

	Macro. Gathers the current xspam lexical context up into
	an object suitable for passing to with-xspam-context.
	It does not evaluate its arguments, which should be
	one or both of the keywords :source and :text.
	If :source is given, the context will contain the current xspam source.
	If :text is given, the context will contain the current xspam text context.

	Returns the new context object.

`xspam-source ()`

	Macro. Returns the current xspam source, a Klacks source.

`with-xspam-context ((context &rest which) &body body)`

	Macro. Creates a new xspam lexical context for body
	(an implicit progn). Context is a form evaluated to get
	context object, as returned by xspam-context.
	Which is not evaluated, and may hold one or both of the keywords :source
	and :text, determining which primitives are valid in the new context.
	For instance, if :text is not specified, none of the text primitives,
	such as matches, guard, etc will be valid inside body.

	Returns the result of evaluating body.

Element and attribute names.
----------------------------

The names of elements and attributes can be specified in several
ways, and the way they are matched depends on whether there
is a namespace in scope (from the with-namespace macro).

A namespace prefix is optional - if given, it specifies a
namespace with the given prefix. If there is
no namespace in scope, then it is an error to specify a prefix;
in that case, a name matches itself in all namespaces.
All matching of names is case sensitive.

An element name may be a string, a keyword or a non-keyword symbol.

A string is of the form "name", or "prefix:name".

A non-keyword symbol's name is used as if a string had been given,
except that all-uppercase names are not allowed.

A keyword symbol is transformed to a string in the following
way:
	If its name is uniformly upper case, it is transformed lower case.
	The prefix (if given) taken from its name up to, but not including
	the first dot (.).
	"Camel case" conversion then converts to upper case any
	character following a minus-sign (-).

Thus:
	:ns.funky-name
translates to
	"ns:funkyName"


Xspam sources
-------------

An xspam source is just like a CXML Klacks source, except that
it refuses to read beyond the bounds of the current element
(defined by the most recently enclosing
element construct). Attempts to do this (using klacks:peek-next, for example)
result in :end-of-document.

On exit from the enclosing element tag, no matter how much of the
element has been read, the next read from the xspam source
will yield the token *after* the end of the previous element. This enables
normal Lisp control-flow constructs to be used, such as RETURN-FROM,
even when deeply embedded inside a piece of XML. (Note that the
current token is advanced lazily - if the parsing of an element is
aborted part-way through, the rest of the element will only be
read if a token beyond its end is required.)

This means that it is perfectly OK to use an xspam source
with arbitrary software that reads a Klacks source (or to turn the
xspam source into a SAX source and use that) without needing
to worry about the integrity of the surrounding matching code.

For instance, the following code will return ("1.5" . "London"):

	(with-xspam-source "<?xml version=\"1.0\"?>
	<body>
		<people>
			<person>
				<name>John</name>
				<height>1.7</height>
			</person>
			<person>
				<name>Susan</name>
				<height>1.5</height>
			</person>
			<person>
				<name>Bob</name>
				<height>1.9</height>
			</person>
		</people>
		<location>London</location>
	</body>"

	  (element :body
	    (let (height)
	      (element :people
	        (loop with name do
	          (element :person
	            (element :name
	              (text
	                (setf name _)))
	            (element :height
	              (when (equal name "Susan")
	                (text
	                  (setf height _)
	                  (return)))))))
	      (element :location
	        (text
	          (cons height _))))))


† name courtesy of michaelw on #lisp.
