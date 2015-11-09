# Depsy: valuing the software that powers science
Heather Piwowar and Jason Priem  

*This paper is still in progress. Feel free to submit a pull request with updates and changes.*










# Introduction

Today's cutting-edge science is built on an array of specialist research software. This research software is often as important as traditional scholarly papers--[but it's not treated that way when it comes to funding and tenure](http://sciencecodemanifesto.org/). There, the traditional publish-or-perish, show-me-the-Impact-Factor system still rules.

We need to fix that. We need to provide meaningful incentives for the [scientist-developers](http://dirkgorissen.com/2012/03/26/the-researcher-programmer-a-new-species/) who make important research software, so that we can keep doing important, software-driven science.

[Lots of things have to happen](http://rev.oxfordjournals.org/content/early/2015/07/26/reseval.rvv014.full) to support this change. Depsy is a shot at making one of those things happen: a system that tracks the impact of software in *software-native ways*.

That means not just counting up citations to a hastily-written paper *about* the software, but actual mentions of *the software itself* in the literature. It means looking how software gets reused by other software, even when it's not cited at all. And it means understanding the full complexity of software authorship, where one project can involve hundreds of contributors in multiple roles that don't map to traditional paper authorship.

Depsy doesn't do any of these things perfectly, and it's not supposed to. Instead, Depsy is a proof-of-concept to show that we can do them at all. The data and tools are there. We *can* measure and reward software impact, like we measure and reward the impact of papers. 

Given that, it's not a question of *if* research software becomes a first-class scientific product, but *when* and *how*. Depsy is meant to encourage conversations about when and how. Hopefully it will prompt folks to improve Depsy, to build systems better than Depsy, and to (most importantly) build the cultural and political structures that can use these systems.


## Related work

*This section is very much still in progress! We're missing a lot of stuff. If you think your paper/app/thing should be mentioned here, submit a pull request and we'll put it in.*

+ There are a few commercial services (http://libraries.io, https://www.versioneye.com) that track dependencies. These are missing a focus on research software, and also find fewer dependencies then they should.
+ There are several smaller attempts to catalog and understand PyPI, including
    + A browsable and searchable interface on top of PyPI: http://pydoc.net/.
    + A ranking of most impactful PiPI projects at http://pypi-ranking.info. Only uses download information though.
    + Maps of the PyPI dependency graph: [this blog post](https://ogirardot.wordpress.com/2013/01/05/state-of-the-pythonpypi-dependency-graph/), 
+ Similarly, there are several projects analyzing CRAN and the R dependency graph:
    + https://github.com/metacran/cranlogs.app provides an API on CRAN download data.
    + Several blog posts and presentations on visualizing the CRAN dependency graph, including [this one](http://www.r-bloggers.com/differences-in-the-network-structure-of-cran-and-bioconductor/) and [this one](http://librestats.com/2012/05/17/visualizing-the-cran-graphing-package-dependencies/)
    + several apps to provide better interface to CRAN, with some impact information including http://www.crantastic.org/ and https://mran.revolutionanalytics.com/packages/ 
    + There've been a few efforts to make systems that assess the scholarly impact of code. http://sciencetoolbox.org/ is one of the best.
+ There's a lot of writing about this. TODO add lots of it. 
    + James Howison's work is a good start
    + Dan Katz' work about transitive credit (Dan is happy to help - let him know what you want from him)
    + Another question is what to count - see https://danielskatzblog.wordpress.com/2015/10/29/software-metrics-what-to-count-and-how/  For example, Dan thinks it's important to count actual usage more than downloads, though this can be orthogonal to reuse.

## Coverage

Depsy tracks research software packages hosted on either [CRAN](https://cran.r-project.org/web/packages/) (the main [software repository](https://en.wikipedia.org/wiki/Software_repository) for the R programming language) or [PyPI](http://pypi.python.org) (the software repository for Python-language software).

R is a language for doing statistics.  As such, almost all of its packages are written by academics, for academics, to do reasearch--so we consider all R software on Depsy to be research software. We cover the *7057* R software packages available on [CRAN](https://cran.r-project.org/web/packages/), the main library repository for R.

Python is a more general purpose programming language.  We try to establish whether a given Python package is research software by searching its metadata (package name, description, tags) for researchy-words (see [code on GitHub](https://github.com/Impactstory/depsy/blob/870c85ee4598643f496bca76e5a7dff994e53837/models/academic.py)). We cover all *57243* active Python packages on [PyPI](http://pypi.python.org), Python's main package repository; of those, we count *4166* as research software.


## Package impact

Depsy calculates impact along three dimensions (and a fourth, coming later, that will capture social signals like GitHub watchers and Stack Overflow discussions). The overall impact score of a package is calculated as the average of these three: **downloads** from software repositories, **software reuse** via reverse dependencies, and **literature reuse** via mentions within papers. 

### Downloads

Gathering download statistics is easy.  We gathered monthly download numbers for python packages from PyPI (ie [numpy](https://pypi.python.org/pypi/numpy) ), and from the [CRAN logs](http://cranlogs.r-pkg.org/) api for R (ie [knitr](http://cranlogs.r-pkg.org/downloads/daily/1900-01-01:2020-01-01/knitr)).  We took a snaphost of these numbers in September 2015.


### Software reuse

Quantifying software reuse is less easy.

The main way software is reused by other software is by being *imported*. (Dan: Does use of a library count as reuse?) For instance, let's say I'm doing a project in R that tracks change in something over time. I end up working with dates a lot (yesterday, a year ago, March 21 1981, and so on). Out of the box, R's support for this is only so-so. I could write a bunch of code to improve it, but instead I'll *import* Hadley Wickham's very nice ```lubridate``` library. Now ```lubridate`` is a *dependency* of my software, because I depend on it. I can reuse his code to make dealing with dates faster and easier. It's a bit like citation on steroids.

If we wanted to do a great job of tracking how reused my software was, we'd want to measure three things:

1. How many times my package is imported by other projects. All else equal, more is better.
2. How crucial those imports are. If you import my software package and *only* my package, that suggests more impact than if you import 100 packages and I'm only one of them.
3. How important the projects are that reuse my package. I'd rather be depended on by a single Nobel-quality software project than reused in a hundred hacky little throwaway projects.

Here's where it gets cool: there's a metric that measure all three of these together: the famous PageRank algorithm, used by Google to rank the importance of webpages. But where PageRank uses hyperlinks to define the relationships between webpages, we'll use imports to define the relationship between software projects. That'll get us our number to define the true importance of a software package in the dependency network.

To build this number, we'll need three steps, which are covered in detail below:

1. Identify dependencies
2. Calculate PageRank
3. Display PageRank in a way that is interpretable

#### Identify dependencies

Ideally, we'd like to find absolutely all of the code that reuses a given research software library.  Alas this is impossible -- most software lives only on private harddrives and servers, so we don't know that it exists, let alone that it depends on a given software library.

That said, there's great news: more and more code is becoming Open Source, available for anyone to browse and search.  GitHub is by far the largest such repository, with over 9 million users and over 21.1 million repositories [[wikipedia]("https://en.wikipedia.org/wiki/GitHub")].  By analyzing how much a given software package is reused on GitHub, we can get a pretty good idea of how much that software is reused in general. 

We can also analyze the dependency information in the open source code of packages availale on PyPi and CRAN, the main software repositories for Python and R respecively.

So then we've got three places to look for software reused via imports:  GitHub, PyPI, and CRAN.  The specifics of how we do this differ by language:


##### Finding dependencies on R libraries: CRAN and GitHub

The CRAN website lists reverse depencencies for each library (see [the CRAN page for knitr](https://cran.r-project.org/web/packages/knitr/) for what this looks like), so that part is easy. We count the "Reverse depends" and "Reverse imports" fields since those reflect actual use, but not the "Reverse enhances" or "Reverse suggests" fields since those are optional.

We also wanted to find every time that a software project on GitHub uses an R library.  We started by getting a list of all R GitHub repositories using [Google BigQuery's GitHub Archive](https://bigquery.cloud.google.com/table/githubarchive:github.timeline) and exporting the results of [this query](https://github.com/Impactstory/depsy/blob/master/sql/bigquery_all_github_r.sql).
  
Then for each GitHub repository we downloaded its zipped source code, extracted lines that included the words "library" or "requires" from files that end in R and R/Sweave/knitr filename extensions, discarded comments, and then used regular expressions to extract the library name. [Here's the code that does that](https://github.com/Impactstory/depsy/blob/master/models/github_repo.py) if you'd like more details.

##### Finding dependencies on Python libraries: PyPI and GitHub

Building the Python dependency graph is a bit harder.

To start with, we downloaded the source distribution of each PyPI library using its most recent release url listed on PyPI.  For the recently introduced ["wheel" format"](http://pythonwheels.com/), we extracted the list of required libraries from the package's metadata.json file. For other source distributions we extracted the list from the older requires.txt file.  

Python has a number of libraries we didn't to track: ones that come with a base Python installation, ones  used only by certain environments (Windows pywin for example), auxilliary install options (like development or test options). So we excluded these. That took care of finding all the dependencies in PyPI.

Next, we wanted to find all the times that sofware in GitHub depends on a Python library.  This starts off simple:  we got a list of all Python GitHub repositories, using Google BigQuery, as above with R.  For each GitHub repository we downloaded its source code, again just like we did with R.  

From there, we looked for dependencies in two places to make sure we didn't miss anything.  First, if the software repository included a setup.py or the newer requirements.txt file, we recorded the libraries listed there.  However, we found that Python code often has dependencies *that are never explicitly specified*. We didn't want to miss these, so we used a second technique: actually reading the Python source code to find import statements.

To do this, we extracting lines that included the word "import" from files that end in ```.py```. We discarded comments, then used a set of regular expressions to extract the import name from these lines (which, annoyingly, use a pretty diverse syntax).  Once we had the names of the imported libraries, we weren't done though--in Python, the import statement is often used to manipulate local code, as well as to bring in external dependencies. So we compared the import names we found to the source code directory tree.  If the import name was a subdirectory or a filename we assumed they were importing a local module, so we discarded the import. You can see  details of this in the [Python source file.](https://github.com/Impactstory/depsy/blob/master/models/pypi_package.py)

The final task was mapping the import name to a PyPI library. This was harder than we'd hoped, because import names can be completely different from package names, one of the (many) [painful bits of Python's packaging system.](http://lucumr.pocoo.org/2012/6/22/hate-hate-hate-everywhere/). Even worse, in some cases more than one PyPI library  uses the *same* import name (for example, pipeline, python-pipeline, and django-pipeline all distribute a package or module called "pipeline", as [PEP 423](https://www.python.org/dev/peps/pep-0423/#use-a-single-name) discusses).

So, the approach we finalized on was this:

1. Run through all Python package on PyPI and extract the package or module name that each package specifies in its setup.py. That's it's import name. Sometimes the setup.py runs code to generate the value, in which case we ccouldn't extract it from the static setup.py file -- in these cases we made a best guess, and scraped the highest level directory from http://pydoc.net.  For example, check out the python library [python-igraph](https://pypi.python.org/pypi/python-igraph): its import name is actually ```igraph```, so you import it as ```import igraph```.  We built a lookup from these import names back to the PyPI library names.  
2. If there is more than one PyPI package with the same import name, we figured out which one had the most downloads, then assign all uses to that package. Where these name collisions keep us from getting *any* data on imports (max 0), we estimate their PageRank percentile from their download percentile.

#### Calculate PageRank

We load the dependencies into Python's [igraph](http://depsy.org/package/python/python-igraph) library as a directed network, then calculate PageRank using the ```personal_pagerank``` method.  You can play with it too by exporting the two ```dep_nodes_ncol_reverse``` tables from the PostgreSQL database ([this script](https://github.com/Impactstory/depsy/blob/master/scripts/run_igraph.sh) can help).

#### Display PageRank in a way that is interpretable

The last step is to give the PageRank measure some context. PageRank values are really hard to grok by themselves. What does it mean if I tell you a given package has a PageRank of 1.7 * 10<sup>-6</sup>?

So we make the PageRank value more understandable by transforming it into a number between 0 and 10, and then also providing its percentile compared to other libraries.  

There are a few steps to this. 

1. We scale scores by the max score in the network by dividing PageRank score by the maximum PageRank score (Python libraries are compared to Python libraries, and R to R).
2. We taking the log10 transform. The dependency graph exhibits properties of a [scale-free network](https://en.wikipedia.org/wiki/Scale-free_network), including an extremely skewed distribution. The log transform eases interpretation and plotting.
3. We add an offset so all numbers are positive
4. We  multiply everything by a scaling factor that places values into the range 0 to 10.  

So, after all this conditioning, we can see that a package with a PageRank Score of 0 isn't depended on by any code, whereas a PageRank Score of 10.0 is depended on heavily by a lot of projects, including a lot of important projects. Even after the transformation, the values are quite skewed (TODO: plot).


### Literature reuse

If software is making an impact on research, we'd expect to see it cited in the literature. And it is. But it's cited haphazardly--instead of formal citation, we often see things like "we used foo library for the figures" in papers.  In fact, Howison and Bullard find [only around a third of software mentions are formal citations](http://onlinelibrary.wiley.com/doi/10.1002/asi.23538/abstract). These informal citations don't show up in citation indexes, and so it's hard to trace--and hard for authors to get credit.

One solution is to write "software papers" that act as hooks for citations instead of the software itself. This doesn't try to change the current paper-oriented publication system, but rather leverage it. However, often these papers don't get cited either, and even more often they don't get written (and why should they? software shouldn't have to pretend to be something else to be cited). So while this approach has value, it's too mired in the past.

A different, more radical solution is to mint DOIs for software, counting on the symbolic value "it has a DOI" to encourage better citation rates and better traceability of citation. This anticipates a new, more diverse publication landscape in which software and traditional papers coexist as first-class citizens. However, though this may pay off in the long term, for now software DOIs are uncommon--and often remain uncited even when they exist. While this approach has value, it's too dependent on the future.  (Dan: I fairly strongly think this is the way things will go.)

A third approach can combine the virtues of both of these: text-mining the literature to track mentions that indicate software reuse. This works within the currunt citation culture, but also treats software as the first-class scholarly output it deserves to be.

Depsy implements a proof-of-concept version of this third approach, searching the full text of literature hosted in two places: Europe PMC and ADS (which includes arXiv, some other OA corpora, and a few toll-access sources).  Because these sources have different abilities, we search differently in both places.

#### Europe PMC

To find mentions of libraries, we simply use Europe PMC's (excellent) API to search for the name of the library. However, many packages have names that give false positives: naming your project "THE" should not result in millions of "uses." 

So, we require names be quite distinctive before we accept search matches as accurate. To do this, we do a "filtered search," looking for mentions of each package AND "python" or "r statistical". These are assumed to be legitimate mentions. We then do the search again, but this time with the package name by itself, with no filter.  

Package name with commonly-used words (```requests```, ```plot```, etc) have very small numbers of filtered mentions compared to overall mentions, while packages with uncommon names (```astropy```, ```rWBclimate```) have more similar counts whether filtered or not. We discard search counts when filtered searches deliver less than 20% as many hits as unfiltered. In those cases, we instead just look up its PyPI URL and GitHub url (if known), which is much more precise (at the cost of greater recall).

#### ADS

We found ADS has poor precision when looking up software names, because it ignores puncutation.  This results in many incorrect matches, where it finds software names inside equations, for example.  

So for ADS we instead search for a specific set of phrases that more conclusively identify software mentions. You can see an example in this search for [ADS mentions of the Astropy library](http://labs.adsabs.harvard.edu/adsabs/search/?q=(%20(=%22import%20astropy%22%20OR%20=%22github%20com%20astropy%22%20OR%20=%22pypi%20python%20org%20astropy%22%20OR%20=%22available%20in%20the%20astropy%20project%22%20OR%20=%22astropy%20a%20community-developed%22%20OR%20=%22library%20astropy%22%20OR%20=%22libraries%20astropy%22%20OR%20=%22package%20astropy%22%20OR%20=%22packages%20astropy%22%20OR%20=%22astropy%20package%22%20OR%20=%22astropy%20packages%22%20OR%20=%22astropy%20library%22%20OR%20=%22astropy%20libraries%22%20OR%20=%22astropy%20python%22%20OR%20=%22astropy%20software%22%20OR%20=%22astropy%20api%22%20OR%20=%22astropy%20coded%22%20OR%20=%22astropy%20new%20open-source%22%20OR%20=%22astropy%20open-source%22%20OR%20=%22open%20source%20software%20astropy%22%20OR%20=%22astropy%20modeling%20frameworks%22%20OR%20=%22astropy%20modeling%20environment%22%20OR%20=%22modeling%20framework%20astropy%22%20OR%20=%22astropy:%20sustainable%20software%22%20OR%20=%22astropy%20component-based%20modeling%20framework%22)%20-author:%22astropy%22)&year_from=1997&month_to=&year_to=2016). Despite a great deal of testing various phrase combinatinos, this approach still results in many incorrect hits for common-word package names, so we use the ratio approach described above for PMC to identify and exclude these, improving accuracy.  The fallback for common word package names for ADS is simply to return 0 hits because the URL searches we use for PMC are not precise enough. 

#### Limitations

This approach has several important limitations: 

* most obviously, __Depsy finds way fewer mentions than Google Scholar does__. That's because we only mine a very small segment of the literature compared to Google Scholar (GS). However, Depsy's data is valuable because it often finds mentions that GS does not, so can be used to improve GS counts. Moreover, Depsy's counts are based on open data, so they can be automatically gathered, remixed, and reused in other applications. You can't do that with GS data, which is released under very restrictive terms of use. Finally, Depsy evaluates software mentions more specifically than GS, using the heuristics described above, which in some cases results in greater accuracy.
* Depsy does not find citations to traditional software papers (unless their titles include the name of the software).
* Depsy's heuristic-based approach can and does make errors in both precision and recall.

#### Future work

Although Depsy's text-mining approach to assessing literature impact has some big holes that restrict it to primarily proof-of-concept use right now, it's got a lot of potential, and we've got big plans of future improvements. We're going to include papers that often serve as proxies for software name mentions, particularly those identified within CRAN as the author's prefered attribution. We're also going to incorporate more network information, using not just how many people have cited something, but *who*. Eventually we'll incorporate citation information into the software reuse network, building a comprehensive, heterogeneous impact map. Finally, we'll be working with publishers to text-mine larger subsets of the research literature. (Dan: while text mining will always pick up mentions that are not cites, it will also alway overcount, while cites always will undercount.  I wonder if this range idea has value.)



## Person impact

There are lots of ways to represent differing contributions among authors of a paper, including [author order](http://www.phdcomics.com/comics/archive.php?comicid=562), corresponding authorship, acknowledgements, and more explicit contribution statements.

Representation of software authorship has evolved, for the most part, seperately. So if we want to look at software authorship in an academic context, we need to translate software-native authorship representations into something academics can understand and work with.

Depsy does this by calculating transitive credit for the authors of research software.  That means we look at the impact of the research software they've contributed to, times the size of the contribution they've made to each project, summed accross everything they've worked on. For more on transitive credit, see [Katz and Smith 2014](http://arxiv.org/abs/1407.5117).

### Authorship

First, we calculate the authors of software: 

+ The author names are parsed from the author bylines in the PyPI and CRAN metadata
+  If the package lists its GitHub repo, we associate the two. We also use some automated methods to connect GitHub repos with CRAN and PyPI packages where they aren't explicitly made in the package metadata. Finally, we manually inspect important packages and connect them with GitHub repos using Google searches.
+ For each package that has an associated GitHub repository, we augment the author list with the GitHub owner and GitHub commiters.  We get these using the GitHub api.  We consider an author identicial when they have the same email address or github credentials. We also parse their human names and consider people identical when they have matching first and last names. 
+ Within a given project, authors are deduplicated using identical first and last names, even if they have different email addresses

### Contribution-level quantification

Once we have a package's authors, we figure out their respective levels of contribution to the project:

+ GitHub committers assigned a contribution-level proportional to their proportion of the software commits
+ Authors who aren't GitHub committers assigned contribution-level equal to that of the most-contributing GitHub committer
+ GitHub owners are assigned a token 1% contribution-level
+ Credits are weighted such that the contribution-level assigned across all contributors on a package sums to 1.


### Component scores

We calculate research software impact subscore for downloads, software reuse, and literature citations for each author.  

We do this by summing a fraction of the percentile scores across each of the research software packages they've contributed to, using a fraction equal to the contribution-level they've made to the package, as we calculate above -- if you contribute more to a project, you inherit more of its impact.  

Next, more percentiles: we calculate the percentile of each of these aggregated fractional percentile scores against those of all the other people in the network who primarily code in the same language as the given author.  

The result of the calculations to this point is that each author has three percentiles that represent the impact they've made in research software: one for download, another for software reuse, and a third for citatons.

### Overall transitive credit

Finally, we calculate an overall person-level score.  We get this by averaging the three componenet subscore percentiles, then, yes, you guessed it, then taking the percentile of *that* across all people.

It is a lot of percentiles, we agree.  :)  We tried several approaches, but ended up with percentiles mostly because 1) the data is very skewed, with different distributions across the subscores and 2) it provides instantly-accessible context for users of the scores.  Stay tuned for blog posts exploring this in more detail.

## Case studies

TODO.


## Known limitations

- we get contribution/commit information for packages hosted on GitHub.  We don't yet get contribution information from BitBucket or any other place where package contributor information might be stored.  We gather information on only the first 100 committers. 
- we're missing GitHub links for many packages, which unfortunately means we are missing their detailed contributor/commit information. It's surprising how many packages have a GitHub repo but do not link to it in the package metadata. We attempted to link more by automatically comparing the contents of setup.py files on github and PyPI, but this unfortuantely provided many false matches so we aren't including those links until we can improve the recall.
- we look for dependency information on all python and R GitHub repositories created before Jan 1 2015 (when the GitHub API changed, and Google Big Query changed their GitHub data structure).  
- we include active packages, which means we've deleted those without source code available on CRAN/pypi, or in the case of pypi libraries with fewer than 50 downloads (affects about 100 packages).
- We count self citations and self-imports. Depending on how you look at it, this is a feature of a bug. Either way, it can have a significant affect, especially when dealing with low *n* as we often are.
- we do our best to guess whether a package is a research package or not. Sometimes we get it wrong. Also, it's hard to know what this is for R; right now we consider all R packages to be academic.  
- download numbers are notoriously imperfect. They are inflated by automatic downloads (continuous integration servers are a chief culprit) and deflated by downloading from mirrors, among other sources of flakiness.
- somewhat obviously, this is currently limited to python and R, which is a subset of more general scientific software.

# Data availability

Everything in Depsy is open, and we encourage reuse of the data and code. You can use the API (see "View in Api" buttons on the package, person, and tag pages), or get larger-scale access via our read-only postgres database. The database connection details (plus an example of how to connect in R) are in [the Knitr source for this  paper](https://github.com/Impactstory/depsy-research/blob/master/introducing_depsy.Rmd).

# Funding

Depsy is funded by an [EAGER grant (number 1346575)](http://blog.impactstory.org/impactstory-awarded-300k-nsf-grant/) from the National Science Foundation.


