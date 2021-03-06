As a doctoral student studying history, I have been regularly frustrated when archival collections are difficult to access. They often require travel, something I have difficulty planning with a daughter and limited funding. Fortunately, many of my archival sources are available closer to home, at the libraries of the City University of New York, the subject of my dissertation. However, accessing archives then becomes a slow and uneven process of navigating out-of-date finding aids and bureaucracy.

My habit has been to scan or photograph as many of the primary documents I find related to my research questions. This is partly a result of my fear of missing out on a piece of evidence I don't take not of in my research journal. I also do this because I hope these primary document could later be converted to searchable PDFs, over which I could efficiently search and categorize primary research materials for my own further research or for others interested in the same primary materials.

If I am lucky, a collection has already been digitized. I am thinking specifically here of how even digitized archival collections are difficult to process. This can potentially save me time, allowing me to avoid the more tedious task of reading document after document from an archive because the collection can be searched by some key terms, but this has proven a mixed bag. Digitized documents might have poor quality scans or the scans have poor quality searchable content. While I can control my own process for digitizing, in the case of collections already digitized, I needed less than ideal solutions that would improve the quality of the searchable content.

In this post, I will illustrate the sort of errors in optical character recognition (OCR) that I have found in my archival research and one workaround I have used with some success ([Abbyy](https://www.abbyy.com)). The source material I will focus on are digitized historical newspapers, specifically those produced by students at CUNY. In the spirit of open digital scholarship, I have included R source code I used to analyze the problem.

Scraping data then cleaning it
------------------------------

Recently, as I was writing an essay on the relationship of CUNY students to the civil rights movement, I attempted to search for mentions of important people and events. One such individual was [James Meredith](https://en.wikipedia.org/wiki/James_Meredith), whose barring from admission by the University of Alabama was national news. One of my sources, the [Baruch Ticker](http://ticker.baruch.cuny.edu/) conveniently has an online search interface, which I decided to use for my query. A fulltext search for "James Meredith" turned up an editorial on page 3 of the [November 20, 1962](http://ticker.baruch.cuny.edu/files/articles/ticker_19621120.pdf) issue. But the results proved unreliable.

As I wanted to automate the process of searching for key terms to some extent, I decided to scrape the information from the web search interface. I used the [rvest](https://github.com/hadley/rvest) and identified XPaths from the search result to query and extract results.

``` r
library(tidyverse)
library(stringr)
library(rvest)
library(httr)
library(imager)
```

``` r
# set user agent string to make sure web server replies with full page
uastring <- "Mozilla/5.0 (Windows NT 6.1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/41.0.2228.0 Safari/537.36"
baseurl <- "http://ticker.baruch.cuny.edu/" 
session <- 
  baseurl %>%
  html_session(user_agent(uastring))
form <- session %>%
  # index page has malformed forms so I cannot directly use html_form on the session
  html_node(xpath = '//*/form') %>%
  html_form
form <- set_values(form, 
                   'data[Search][keywords]' = "James Meredith",
                   'data[Search][field]' = "fulltext")
results_session <- submit_form(session, 
                               form, 
                               user_agent(uastring))
```

    ## Warning in if (!(submit %in% names(submits))) {: the condition has length >
    ## 1 and only the first element will be used

``` r
results_urls <- 
  results_session %>% 
  # the search result page unfortunately has a complicated structure
  html_nodes(xpath='//*/div[@id="search_results_div"]/div[@class="results"]/div[contains(@class,"result_")]/div[@class="result_title"]/a') %>%
  html_attr("href") %>%
  # remove the query string that is appended to the pdf url
  str_replace("#.*$", "")
results_contexts <-
  results_session %>% 
  html_nodes(xpath='//*/div[@id="search_results_div"]/div[@class="results"]/div/div[@class="context"]') %>%
  html_text
results <- tibble(url = results_urls,
                  context = results_contexts)
```

Though there are 13 results, only one of them actually contains the exact phrase "James Meredith" contiguously. The only other result relevant to our query actually was erroneously recognized as "J a m e s Meredith" (with inserted spaces), but "James" was recognized elsewhere on the page.

``` r
results %>%
  mutate(
    # extract the phrase and the ten characters before
    extract = str_extract(context, ".{10}Meredith")
  ) %>%
  select(url, extract)
```

    ## # A tibble: 13 x 2
    ##                                    url               extract
    ##                                  <chr>                 <chr>
    ##  1 /files/articles/ticker_19630923.pdf    James II. Meredith
    ##  2 /files/articles/ticker_19830426.pdf     Possibly Meredith
    ##  3 /files/articles/ticker_19871117.pdf    eason for Meredith
    ##  4 /files/articles/ticker_19761214.pdf    ICATION a Meredith
    ##  5 /files/articles/ticker_19980401.pdf     1907 Don Meredith
    ##  6 /files/articles/ticker_19991201.pdf     I kissed Meredith
    ##  7 /files/articles/ticker_19770103.pdf    or- James Meredith
    ##  8 /files/articles/ticker_19870217.pdf    f Burgess Meredith
    ##  9 /files/articles/ticker_19640421.pdf     i c s by Meredith
    ## 10 /files/articles/ticker_19691209.pdf    inds of'^ Meredith
    ## 11 /files/articles/ticker_19621010.pdf    J a m e s Meredith
    ## 12 /files/articles/ticker_19640225.pdf    APriL The Meredith
    ## 13 /files/articles/ticker_19761112.pdf "c M a n,\" Meredith"

Problem of text quality
-----------------------

Note that `ticker_19621010.pdf` was the document that matched when searching "James Meredith" but only because "James" had occurred elsewhere in the document and not immediately preceding "Meredith". We will now more closely examine the text quality of that pdf to understand why this problem occurred.

I used the [pdftools](https://github.com/ropensci/pdftools) package to extract the text from the PDF. Behind the scenes, this package is using the [poppler](https://poppler.freedesktop.org/) library, commonly used on Linux systems.

``` r
pdffile <- tempfile("")
curl::curl_download('http://ticker.baruch.cuny.edu/files/articles/ticker_19621120.pdf', pdffile)
page <- pdftools::pdf_render_page(pdffile, page = 3)
pngfile <- tempfile()
png::writePNG(page, pngfile)
load.image(pngfile) %>% plot
```

<img src="01-ocr-quality_files/figure-markdown_github/unnamed-chunk-3-1.png" width="400px" />

``` r
# knitr::include_graphics(pngfile)
# plot(page)
```

As is apparent from the above image, any OCR technology would have difficult with such scans. The scan has lines across in random places and some of the text is almost unreadable. Let's try to confirm that the text extracted from the PDF has the string "James Meredith".

We first confirm that "Meredith" occurs in the text extracted. But the quality of the PDF is such that we find only two of the three occurrences of Meredith, once in the header, and another in the second paragraph. Missing in the results is the "Meredith" immediately following "James" in the first paragraph.

``` r
pagetext <- pdftools::pdf_text(pdffile)[3] 
pagetext %>%
  str_extract_all(".{10}Meredith.{10}")
```

    ## [[1]]
    ## [1] "          Meredith          " "      Mr! Meredith.         "

One problem is that there are unknown characters in the output. But a more basic problem in the text becomes apparent when we look at a slice of the data. We quickly find how often spaces (" ") are inserted into the extracted text. Any attempt at tokenizing this text would prove difficult because of the poor quality of the source data.

``` r
pagetext %>%
  # grab some random window of 2500 characters
  str_sub(10000, 12500) %>%
  str_wrap(80) %>%
  cat
```

    ## His talk concentrated on t w o Blind Students S t e v e R a p p a p o r t *63
    ## Mike Kreitzer '63 Managing Joe Traum '64 Editor Business Manager s i g n e d b
    ## y D e a n S a x e , w a s circu-» Leonard T a s h m a n '63 lated throughout t
    ## h e school: Perhaps this column will give you a n insight a s to tricks that,
    ## various people Aise to cover UP w h a t they t. H o w e v e r , l e t u s t a k
    ## e ' t h e c a s e o f o n e m a n . T h e t i m e h e l i v e d in

One solution would be to attempt to repair this text using a set of simple rewrite rules. For instance, if there is a series of characters separated by spaces, we could check to see whether removing the spaces would yield a valid word. But the original motivation for this query was to find references to an individual, and we wouldn't easily be able to decide if the word yielded was a valid word if that word in fact is a proper name. So instead I will show how we can reconvert the PDF to a searchable PDF using an alternative OCR technology.

Re-processing documents with Abbyy
----------------------------------

Before processing the document page with Abbyy, it is helpful to highlight that the document we are examining is black and white. What might have been faint marks on the scan in a color or grayscale image become lines that cut across the page. The choice to give users access to the black and white version of the scans could make sense given how the size of grayscale and color PDFs would necessarily be larger. Size certainly matters for the institution hosting the files, be it the storage consumed by a collection or the network bandwidth used when sending those files. But for the academic researchers, such a decision forces them to use smaller PDFs at the expense of quality. Being that archives go through a length and expensive process to digitize material, I would hope they maintain full quality, color scans that could be used for future re-processing as OCR technology improves.

Given these reservations, I was still pleased with the results from Abbyy. I have been using their desktop application, [FineReader](https://www.abbyy.com/en-eu/finereader/), for a while now, and would certainly recommend it to all academic researchers working with digitized primary documents. But here I wanted to explore features that Abbyy only provides through its SDK, which is available with through the [FineReader Engine](https://www.abbyy.com/en-us/ocr-sdk/) as well as the [Cloud SDK](http://ocrsdk.com/). I chose to give the web api a try since there is a trial package of 50 free pages, which was plenty for my purposes in this exercise.

I made use of the [abbyyR](https://github.com/soodoku/abbyyR) package to easily call the web api. The developers of the package provide a straightforward [example](http://soodoku.github.io/abbyyR/articles/example.html) for using the library. I set environment variables for the application name and password that I created in the Cloud SDK.

``` r
library(abbyyR)
setapp(c(Sys.getenv('ABBYYSDK_APP'), Sys.getenv('ABBYYSDK_PW')))
getAppInfo()
```

    ## Name of Application: 1
    ## No. of Pages Remaining: 1
    ## No. of Fields Remaining: 1
    ## Application Credits Expire on: 1
    ## Type: 1

    ##         name pages fields             expires   type
    ## 1 whatsupdoc    38    190 2017-11-18T00:00:00 Normal

I will submit the image of page 3 to be processed, which creates a task that I can monitor till it is complete.

``` r
processImage(file_path = pngfile)
```

    ## Status of the task:  1 
    ## Task ID:  1

    ##    .id                                   id     registrationTime
    ## 1 task e56fe725-bc21-45b3-99c9-7a1d2d77021e 2017-09-01T13:57:25Z
    ##       statusChangeTime status filesCount credits estimatedProcessingTime
    ## 1 2017-09-01T13:57:26Z Queued          1       0                       5

``` r
# keep on checking if task is finished, waiting for 5 seconds in case it isn't
i <- 0
while(i < 1){
  i <- nrow(listFinishedTasks())
  if (i == 1){
    print("All Done!")
    break;
  }
  Sys.sleep(5)
}
```

    ## No. of Finished Tasks:  11

Once the processing is completed (which took about 90 seconds), the file can be downloaded and we can extract the text as we did above. We notice immediately that we are now getting three matches for "Meredith" rather than just two and that in this output we do have the exact phrase "James Meredith". Great!

``` r
finishedlist <- listFinishedTasks()
```

    ## No. of Finished Tasks:  12

``` r
resultUrl <- finishedlist$resultUrl %>% as.character()
abbyyFile <- tempfile()
curl::curl_download(resultUrl, abbyyFile)
abbyyText <- read_file(abbyyFile)
abbyyText %>%
  str_extract_all(".{10}Meredith.{10}")
```

    ## [[1]]
    ## [1] "          Meredith          " " to James Meredith's presenc"
    ## [3] " with Mr. Meredith.! that th"

If we examine a slice of the text, we again see a vast improvement over the quality of the original document. However, there are still many errors in the text output that a human reader could likely fix. But we will leave that problem for another time.

``` r
abbyyText %>%
  # grab some random window of 2500 characters
  str_sub(10000, 12500) %>%
  str_wrap(80) %>%
  cat
```

    ## ity in which h* lived is of little Jeffers. Jeffers, a poet on whom j Charity
    ## Drive .Vries Editor Assoc. Bus. Man. A short Memorial Service for desire to
    ## say. - --------- tiortancf. We would not want this individual to be ari anomaly.
    ## Levy is a noted authority, was Mike Del Gindie# '64 Jody Bernstein 1  *  #
    ## Iking any identification with his counterpart* in society, so we shall honest.
    ## the professor said. _ Alpha Phi Omega, the national the late Anna Eleanor
    ## Roosevelt Student ConacU representative to one of his clone conatltn. II him
    ## Mr. Zacaz. # The other, Mark Van Doren. Era!hit* Editor wttt be heftt-fa the
    ## Auditorium of- I We. must next act the scene. Mr. Znraz return* btfm^Troni work,

We can also request that the image processing to generate an Output XML document that provides even detailed information about the resulting document. The XML represents documents as a nested structure of pages, blocks, regions, rectangles, collections of paragraphs, lines, and ultimately characters. Most importantly the chracter level data includes variants of the character recognized and the confidence that the OCR engine had for each variant.

One could imagine taking the character level variations as input to a post-processing step where the text content of a document are automatically corrected. Such post-processing could use the lexical information from other documents in a collection to substitute the output of the OCR engine with more likely words and phrases. For instance, after building a dictionary of words after tokenizing all documents in a collection, we could merge similar words in the hopes of catching the sort of errors made during recognition. This is an area that I frankly need to learn more, not just specific to what can be extracted from the Abbyy SDK but how OCR technology generally approaches the problem of word-level correction.

For now, one useful piece of information that can be pulled out of the Abbyy output XML is that blocks of text on a page. Our case study has been a historical newspaper. I have for now ignored that text on the page is layed out in blocks of articles. Lines of two or more articles will be vertically aligned, making it hard for OCR technology to extract sentences and paragraphs reliably. We can see this by going back to our original PDF and extracting a window of text after heading for the editorial on Meredith we have examined above.

First, let's crop out the article. The dimensions for such a crop were arrived at by visually approximating where the editorial appears on the page. Of course, this is not a process that would be easily reproducible by a computer program.

``` r
library(imager)
cropped <- tempfile(fileext=".png")
load.image(pngfile) %>% 
  imsub(x < (2466 / 3), y > (1473 / 2)) %>% 
  save.image(cropped)
knitr::include_graphics(cropped)
```

<img src="C:\Users\tahir\AppData\Local\Temp\RtmpQj0jSR\file77403514be5.png" width="400px" />

The text extracted from the original clearly did not break distinguish between the text of the editorial and the text of the letters to its right.

``` r
pagetext %>% str_extract("Meredith.{100}")
```

    ## [1] "Meredith                                                                       n o t t h e a d m i n i s t r"

Did the output of the Abbyy Cloud SDK fare any better with the headline?

``` r
abbyyText %>% str_extract("Meredith.{100}")
```

    ## [1] "Meredith                                                               should have the right to decide, and "

However, the output text from the desktop FineReader proved far better, with the article seeming to be segmented properly.

    Meredith

    Reaction to James Meredith’s presence on the Univer­sity of Mississippi campus is still strong. However, we believe that it has reached the peak of cowardice when a dean of the school believes it is ill-advised for white students to eat with Mr? Meredith. w ----------
    Last week a group of students at the university had the courage and intelligence to eat -lunch with Mr. Meredith.

What next?
----------

Using Abbyy, I was able to dramatically improve the character recognition of documents from my archival research. With better text extraction, documents can be analyzed using machine learning techniques. In the case of an archival collection, machine learning techniques could group similar documents together so that a researcher who finds one document of interest could immediately locate other documents that might also be of interest. At the moment, the search interface for digitized archival collection is sorely limited, which is not to knock the effort archivists and librarians have put towards it. And, as someone consuming rather than producing these archives, I suspect there are major gaps in my understanding of the problems I have been presented with, perhaps even that I have missed far simpler solutions.

Still, my interest in opening historical scholarship to better automation and machine learning is not to replace the researcher (myself). My hope is to craft computational tools that could help me and others researching with large archival sources. Most of our computers now have sophisticated built-in file search features that can return a list of documents based on some matching expression. But, as is evident with the quality of some of the digitized archival collections I have been researching, better tools requires we first tackle a prior problem, that of reliably extracting text from archival material. This is not a simple problem, since it involves improving character-level as well as word-level recognition, as well as in this particular case of historical newspapers it requires segmenting pages into articles rather than just arbitrary blocks of texts without any understanding of page layout. If you have been tackling these and related problems, please introduce yourself over a email or direct message on Twitter.
