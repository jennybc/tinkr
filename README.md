
<!-- README.md is generated from README.Rmd. Please edit that file -->

# tinkr

<!-- badges: start -->

[![Project Status: WIP – Initial development is in progress, but there
has not yet been a stable, usable release suitable for the
public.](https://www.repostatus.org/badges/latest/wip.svg)](https://www.repostatus.org/#wip)
[![R build
status](https://github.com/ropenscilabs/tinkr/workflows/R-CMD-check/badge.svg)](https://github.com/ropenscilabs/tinkr/actions)
[![Coverage
status](https://codecov.io/gh/ropenscilabs/tinkr/branch/master/graph/badge.svg)](https://codecov.io/github/ropenscilabs/tinkr?branch=master)
<!-- badges: end -->

The goal of tinkr is to convert (R)Markdown files to XML and back to
allow their editing with `xml2` (XPath\!) instead of numerous
complicated regular expressions. Would you like to kknow more? [This is
great intro if you are new to
XPath](https://www.w3schools.com/xml/xpath_intro.asp) and [this is a
good resource on XSLT for XML
transformations](https://www.w3schools.com/xml/xsl_intro.asp).

## Use-Cases

Possible applications are R scripts using this and XPath in `xml2` to:

  - change levels of headers, cf [this
    script](inst/scripts/roweb2_headers.R) and [this pull request to
    roweb2](https://github.com/ropensci/roweb2/pull/279)
  - change chunk labels and options
  - extract all runnable code (including inline code)
  - insert arbitrary markdown elements
  - modify link URLs
  - your idea, feel free to suggest use cases\!

## Workflow

Only the body of the (R) Markdown file is cast to XML, using the
Commonmark specification via the [`commonmark`
package](https://github.com/jeroen/commonmark). YAML metadata could be
edited using the [`yaml` package](https://github.com/viking/r-yaml),
which is not the goal of this package.

We have created an [R6 class](https://r6.r-lib.org/) object called
**yarn** to store the representation of both the YAML and the XML data,
both of which are accessible through the `$body` and `$yaml` elements.
In addition, the namespace prefix is set to “md” in the `$ns` element.

You can perform XPath queries using the `$body` and `$ns` elements:

``` r
library("tinkr")
library("xml2")
path <- system.file("extdata", "example1.md", package = "tinkr")
ex1 <- yarn$new(path)
# find all ropensci.org blog links
xml_find_all(
  x = ex1$body, 
  xpath = ".//md:link[contains(@destination,'ropensci.org/blog')]", 
  ns = ex1$ns
)
#> {xml_nodeset (7)}
#> [1] <link destination="https://ropensci.org/blog/2018/08/21/birds-radolfzell/" title="" ...
#> [2] <link destination="https://ropensci.org/blog/2018/09/04/birds-taxo-traits/" title=" ...
#> [3] <link destination="https://ropensci.org/blog/2018/08/21/birds-radolfzell/" title="" ...
#> [4] <link destination="https://ropensci.org/blog/2018/08/14/where-to-bird/" title="">\n ...
#> [5] <link destination="https://ropensci.org/blog/2018/08/21/birds-radolfzell/" title="" ...
#> [6] <link destination="https://ropensci.org/blog/2018/08/28/birds-ocr/" title="">\n  <t ...
#> [7] <link destination="https://ropensci.org/blog/2018/09/04/birds-taxo-traits/" title=" ...
```

## Installation

Wanna try the package and tell me what doesn’t work?

``` r
remotes::install_github("ropenscilabs/tinkr")
```

## Examples

### Markdown

This is a basic example. We read “example1.md”, change all headers 3 to
headers 1, and save it back to md. Because the {xml2} objects are
[passed by
reference](https://blog.penjee.com/wp-content/uploads/2015/02/pass-by-reference-vs-pass-by-value-animation.gif),
manipulating them does not require reassignment.

``` r
library("magrittr")
library("tinkr")
# From Markdown to XML
path <- system.file("extdata", "example1.md", package = "tinkr")
# Level 3 header example:
tail(readLines(path, 40))
#> [1] "### Getting a list of 50 species from occurrence data"                  
#> [2] ""                                                                       
#> [3] "For more details about the following code, refer to the [previous post" 
#> [4] "of the series](https://ropensci.org/blog/2018/08/21/birds-radolfzell/)."
#> [5] "The single difference is our adding a step to keep only data for the"   
#> [6] "most recent years."
ex1  <- yarn$new(path)
# transform level 3 headers into level 1 headers
ex1$body %>%
  xml2::xml_find_all(xpath = ".//md:heading[@level='3']", ex1$ns) %>% 
  xml2::xml_set_attr("level", 1)

# Back to Markdown
tmp <- tempfile(fileext = "md")
ex1$write(tmp)
# Level three headers are now Level one:
tail(readLines(tmp, 40))
#> [1] "# Getting a list of 50 species from occurrence data"                    
#> [2] ""                                                                       
#> [3] "For more details about the following code, refer to the [previous post" 
#> [4] "of the series](https://ropensci.org/blog/2018/08/21/birds-radolfzell/)."
#> [5] "The single difference is our adding a step to keep only data for the"   
#> [6] "most recent years."
unlink(tmp)
```

### R Markdown

For R Markdown files, to ease editing of chunk label and options,
`to_xml` munges the chunk info into different attributes. E.g. below you
see that `code_blocks` can have a `language`, `name`, `echo` attributes.

``` r
path <- system.file("extdata", "example2.Rmd", package = "tinkr")
rmd <- yarn$new(path)
rmd$body
#> {xml_document}
#> <document xmlns="http://commonmark.org/xml/1.0">
#>  [1] <code_block xml:space="preserve" language="r" name="setup" include="FALSE" eval="T ...
#>  [2] <heading level="2">\n  <text xml:space="preserve">R Markdown</text>\n</heading>
#>  [3] <paragraph>\n  <text xml:space="preserve">This is an </text>\n  <strikethrough>\n  ...
#>  [4] <paragraph>\n  <text xml:space="preserve">When you click the </text>\n  <strong>\n ...
#>  [5] <code_block xml:space="preserve" language="r" name="" eval="TRUE" echo="TRUE">summ ...
#>  [6] <heading level="2">\n  <text xml:space="preserve">Including Plots</text>\n</heading>
#>  [7] <paragraph>\n  <text xml:space="preserve">You can also embed plots, for example:</ ...
#>  [8] <code_block xml:space="preserve" language="python" name="" fig.cap="&quot;pretty p ...
#>  [9] <code_block xml:space="preserve" language="python" name="">plot(pressure)\n</code_ ...
#> [10] <paragraph>\n  <text xml:space="preserve">Non-RMarkdown blocks are also considered ...
#> [11] <code_block info="bash" xml:space="preserve" name="">echo "this is an unevaluted b ...
#> [12] <code_block xml:space="preserve" name="">This is an ambiguous code block\n</code_b ...
#> [13] <paragraph>\n  <text xml:space="preserve">Note that the </text>\n  <code xml:space ...
#> [14] <table>\n  <table_header>\n    <table_cell align="left">\n      <text xml:space="p ...
#> [15] <paragraph>\n  <text xml:space="preserve">blabla</text>\n</paragraph>
```

### Inserting new markdown elements

Inserting new nodes into the AST is surprisingly difficult if there is a
default namespace, so we have provided a method in the **yarn** object
that will take plain markdown and translate it to XML nodes and insert
them into the document for you. For example, you can add a new code
block:

```` r
path <- system.file("extdata", "example2.Rmd", package = "tinkr")
rmd <- yarn$new(path)
xml_find_first(rmd$body, ".//md:code_block", rmd$ns)
#> {xml_node}
#> <code_block space="preserve" language="r" name="setup" include="FALSE" eval="TRUE">
new_code <- c(
  "```{r xml-block, message = TRUE}",
  "message(\"this is a new chunk from {tinkr}\")",
  "```")
# Add chunk into document after the first chunk
rmd$add_md(new_code, where = 1L)
tmp <- tempfile(fileext = ".Rmd")
rmd$write(tmp)
readLines(tmp, 15)
#>  [1] "---"                                          
#>  [2] "title: \"Untitled\""                          
#>  [3] "author: \"M. Salmon\""                        
#>  [4] "date: \"September 6, 2018\""                  
#>  [5] "output: html_document"                        
#>  [6] "---"                                          
#>  [7] ""                                             
#>  [8] "```{r setup, include=FALSE, eval=TRUE}"       
#>  [9] "knitr::opts_chunk$set(echo = TRUE)"           
#> [10] "```"                                          
#> [11] ""                                             
#> [12] "```{r xml-block, message = TRUE}"             
#> [13] "message(\"this is a new chunk from {tinkr}\")"
#> [14] "```"                                          
#> [15] ""
````

## Loss of Markdown style

### General principles and solution

The (R)md to XML to (R)md loop on which `tinkr` is based is slightly
lossy because of Markdown syntax redundancy, so the loop from (R)md to
R(md) via `to_xml` and `to_md` will be a bit lossy. For instance

  - lists can be created with either “+”, “-” or "\*“. When using
    `tinkr`, the (R)md after editing will only use”-" for lists.

  - Links built like `[word][smallref]` and bottom `[smallref]: URL`
    become `[word](URL)`.

  - Characters are escaped (e.g. “\[” when not for a link).

  - Block quotes lines all get “\>” whereas in the input only the first
    could have a “\>” at the beginning of the first line.

  - For tables see the next subsection.

Such losses make your (R)md different, and the git diff a bit harder to
parse, but should *not* change the documents your (R)md is rendered to.
If it does, report a bug in the issue tracker\!

A solution to not loose your Markdown style, e.g. your preferring "\*"
over “-” for lists is to tweak [our XSL
stylesheet](inst/extdata/xml2md_gfm.xsl) and provide its filepath as
`stylesheet_path` argument to `to_md`.

### The special case of tables

  - Tables are supposed to remain/become pretty after a full loop
    `to_xml` + `to_md`. If you notice something amiss, e.g. too much
    space compared to what you were expecting, please open an issue.

## Meta

Please note that the ‘tinkr’ project is released with a [Contributor
Code of Conduct](CODE_OF_CONDUCT.md). By contributing to this project,
you agree to abide by its terms.
