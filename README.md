formX
=====

FormX is a Javascript-based mapping between XML content and web forms.
It consists of three parts, or rather two, and you're responsible for
the third.  There is the formX object, which provides methods for
loading data from an XML file into a web form, a mapping file, the
spcification for which may be found below, and a web form, for which you
are completely responsible. FormX doesn't say anything about what it has
to look like, except that you have to use the same naming conventions
you use in the mapping. The goal of formX is to allow the development of
HTML forms capable of editing any reasonably data-oriented XML while
preserving the formatting of that XML without the overhead of using XForms.

## The formX object ##

The most important functions are
    formX.populateForm()
and
    formX.updateXML()
The first of these uses the mapping file stored in `formX.mapping` to
load data from the XML file stored in `formX.xml` into the an HTML form
in the current web page. The second pulls data out of the form and
writes it back into the XML, again using the mapping file to decide
where to put it.

## The mapping file ##

The mapping file is a javascript file looking something like:
    var map = {
      namespaces: {"#default": "http://www.tei-c.org/ns/1.0",
                   "t": "http://www.tei-c.org/ns/1.0"},
      indent: "  ",
      elements: [
                 {xpath: "/t:TEI/t:teiHeader/t:fileDesc/t:titleStmt/t:title",
                  tpl: "<title>$title</title>"},
                 {xpath: "/t:TEI/t:teiHeader/t:fileDesc/t:titleStmt/t:author",
                  tpl: "<author>$author</author>"}]
      models: { "http://www.tei-c.org/ns/1.0": 
                {titleStmt: ["title", "author"]}},
      functions: {
        // A list of function definitions that may be applied to data on
        // its way out of the XML or on its way back in.
      }}

### namespaces ###

A map of namespaces used in the source document. To set a default
namespace, add a member named "#default" and this namespace will be
returned whenever no prefix is used.

### indent ###

The amount of whitespace (spaces or tabs) by which child elements should
be indented.

### elements ###

Individual `elements` in formX are Javascript objects that may possess
the following members:
    {name: "language",
     xpath: "/t:TEI/t:teiHeader/t:fileDesc/t:sourceDesc/t:msDesc/t:msContents/t:msItem/t:textLang",
     children: ["@mainLang", "@otherLangs", "."],
     tpl: "<textLang mainLang=\"$mainLang\"[ otherLangs=\"$otherLangs\"]>$textLang</textLang>",
     multi: false}
Only two of these, `xpath` and `tpl` are always required. `xpath` is
simply the XPath (using a namespace prefix defined in the mapping's
`namespaces` member) to the data you want in the XML file. `tpl` is a
template to be used when the XML is written back. Variables to be
substituted for values in the form are prefixed with a `$`. In the HTML
form, the `input`s or `textarea`s will be given names corresponding to
the variable names (prefixed by the map's prefix, so
    <input name="your_prefix_textLang" type="text">
for the `$textLang` variable above. Optional
sections are surrounded by brackets, so the `otherLangs` attribute above
may be populated or not. Optional sections may nest, as we'll see below.
`children` contains an array of nodes relative to the XPath supplied in `xpath`, 
given in the same order as the variables in the template. A `name` is
only needed when there is either a group of values that go together, as
above, or when there may be multiple values, so that form fields will
need to be duplicated. In this case, the HTML form must wrap such groups
or fields to be duplicated in a div or span with a @class attribute that
uses the value in the `name`. Finally, `multi`, if true, denotes an
`element` that may be duplicated.

### models ###

In order to know exactly where new elements should be inserted, formX
will sometimes need a simplified content model for the parent element.
This is a Javascript object where element names are mapped to ordered
lists of possible child elements. This is so that, given an XML document
where a parent element has a child already, and another child element of
a different type is to be inserted, formX will know whether to insert
the new element before or after the existing one. 

### functions ###

Functions that may be called on either values extracted from the XML or
on values from the form. For example, in TEI, dates may be given in the
following way:
    <date when="2012-05-08">May 8th, 2012</date>
with a `@when` (or `@notBefore`/`@notAfter`) attribute that uses an ISO 
date format (YYYY-MM-DD with a minus sign before for BCE dates). But in 
an HTML form, you might want to split out the year, month, and
day values in `@when` so that it's easier for a layperson to edit the
values, and so that they can ignore leading zeroes. Maybe you'd want to
give them a dropdown to choose between BCE and CE, and so on. You can
write functions that do this kind of pre- and post-processing on form
values.  For example:
    {name: "origDate",
     xpath: "/t:TEI/t:teiHeader/t:fileDesc/t:sourceDesc/t:msDesc/t:history/t:origin/t:origDate",
     children: [["@when","getYear"],["@when", "getMonth"], ["@when", "getDay"], ["@notBefore","getYear"],["@notBefore", "getMonth"], ["@notBefore", "getDay"], ["@notAfter","getYear"],["@notAfter", "getMonth"], ["@notAfter", "getDay"], "."],
     tpl: "<origDate[ when=\"~pad('$year', 4)~{-~pad('$month', 2)~}{-~pad('$day',2)~}\"][ notBefore=\"~pad('$year1',4)~{-~pad('$month1',2)~}{-~pad('$day1',2)~}\" notAfter=\"~pad('$year2',4)~{-~pad('$month2',2)~}{-~pad('$day2',2)~}\"]>$origDate</origDate>"},

In the `children` property, you'll notice that instead of just relative
XPaths, we have an array containing an XPath and a function name, so
when the child corresponding to the `$year` variable (i.e. the `@when`
attribute) is retrieved, it will be piped through the `getYear()`
function defined in the `functions` section. This function pulls out
just the year from the full date. 

In the origDate template, we see:

    <origDate[ when=\"~pad('$year', 4)~{-~pad('$month', 2)~}{-~pad('$day',2)~}\"]

meaning that the `@when` attribute is optional, but if you have one, it
must have at least a year, followed by an optional month and day (there
are those nested options using curly braces). The value of `$year` is
piped through the `pad()` function, again defined in the `functions`
section, and this zero-pads the year from the form so that it is four
digits long. So the values in the form might be 
    Year:  -99
    Month: 8
    Day:   7
But what would go into the XML would be "-0099-08-07". These functions 
can also perform validation tasks.

## The HTML Form ##

Unlike XForms, formX has nothing at all to say about the construction of
the HTML form to be used in editing the XML, except that its fields'
`@name` attributes must correspond to variable names in the mapping, and
groups of fields and/or fields that may be duplicated must be enclosed by
an element bearing a `@class` attribute with a value corresponding to
the `name` of the `element` in the mapping. Apart from these naming
conventions, you are free to build the form in any way you see fit.
