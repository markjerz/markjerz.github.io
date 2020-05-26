---
title: How to automatically translate ASP.Net Mvc razor views
---

# Introduction

One of our larger projects started off without any internationalisation - it was built for a UK client, and written in English using ASP.Net MVC 5. This application consists of several front end applications, some of them customer facing, others for users of the business only. 

Our first requirement for translation was in 2 of the customer facing applications. We built a small framework to handle the runtime translation and then used sweat, and tears, to replace the hard-coded text inside the views with calls to the translation engine.

This change turns this code

```
<h2>Process @model.VoucherId</h2>
```

in to this code:

```
<h2>@T(w => w.Translation.Vouchers.Process, new { VoucherId = model.VoucherId })</h2>
```

# Round 2

Fast forward a year or so and we were in the position of needing to internationalise a large amount of one of the back office applications.

Previously, we had used human grunt to do the string matching. We needed something quicker this time.

We decided to write a little tool to automatically re-work the views from being hard-coded to being written as in the example above.

As we know [parsing html is hard](https://stackoverflow.com/a/1732454), but that's what we need to do to understand whether our strings need translating or not.

# Parsing Razor

Razor, being a templating language, already has a parser! It's in Microsoft.AspNet.Razor and is called `System.Web.Razor.Parser.RazorParser`. It's one of those bits of code that's public but you're not meant to use it:

> This type/member supports the .NET Framework infrastructure and is not intended to be used directly from your code.Represents a Razor parser.

So, you can make use of this to parse razor files and get the abstract syntax tree:

```
var razorParser = new RazorParser(new CSharpCodeParser(), new HtmlMarkupParser());
    this.results = razorParser.Parse((TextReader)new SeekableTextReader(this.razor));
```

From this, we can then look at the tree of nodes, extract out things that look like text strings and replace those pieces of code with the strongly typed method invocations instead. At the same time, we generate the strongly typed helper classes which contain the English fallback.

There are lots of different places that we can find text strings to translate inside razor page. There are obvious ones like inside a `<p></p>`, slightly less obvious like inside the `alt` tag of an `<img>` element and then harder things like arguments to HtmlHelper methods e.g. `@Html.ActionLink("Click here")`.

For each one of these we write a class that implements the following class:

```
interface IExtractor {
    Possibly IsMatch(Span span, ISymbol symbol, Context context);

    IEnumerable<(Span, ISymbol, Context)> Match();

    void Reset();
}
```

We recurse down through the tree, checking for matches and once we've got some candidates for a particular section of code we pick the one that's the longest. We do that so that we get a single translation entry for an entire block of text, even it's split up over multiple paragraphs.

The piece of code walks the parser results is:

```
private void WalkBlock(Block block, Context context) {
    // try to greedily match internationalisation patterns
    // we care about:
    //      text content of html elements
    //      title attribute content
    //      alt attribute content
    //      input placeholder
    //      input[type=submit] value
    //      @:XYZ
    //      <text>XYZ</text>
    //      @Html.ActionLink("X",...)

    // track: 
    //      are we inside an element definition?            <div .X. >
    //      are we inside an attribute definition?          <div style="..X..">
    //      are we inside an element                        <div ...>X</div>
    //      

    foreach (var syntaxTreeNode in block.Children) {
        if (syntaxTreeNode is Span span) {
            foreach (var spanSymbol in span.Symbols) {
                context.Update(span, spanSymbol);
                foreach (var translator in this.extractors) {
                    var match = translator.IsMatch(span, spanSymbol, context);
                    if (match == Possibly.Yep) {
                        this.matches.Add(translator.Match());
                    }

                    if (match != Possibly.Maybe) {
                        translator.Reset();
                    }
                }
            }
        }
        else {
            var childBlock = (Block)syntaxTreeNode;
            this.WalkBlock(childBlock, context);
        }
    }
}
```

You'll notice that there's a `Context` object. This keeps track of the "context" of the parser and provides that to the extractors so that they know what type of piece of code we're in. This is because the parser doesn't provide us with a DOM, like in html, but a much lower level syntax, so we need this extra layer to track that we are, for example, inside an html element attribute.

Once this piece of code has executed we now have our list of candidate strings, and their locations in the original source code.

The rest of the code then goes through the matches, selecting the longest ones, finding existing translations where they already exist (for example, a "Save" button) or creating new ones where they don't and then generating the code and inserting into the razor file.

Finally, with the help of a good diff tool, or in our case the use of a good git gui, you can quickly review the changes, tweak where necessary and then commit to the repository.

