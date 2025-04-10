---
layout: default
title: "Accessing NHANES data locally"
editor_options: 
  chunk_output_type: console
---



In its default mode of operation, functions in the __nhanesA__ package
scrape data directly from the CDC website each time they are invoked.
The advantage is simplicity; users only need to install the nhanesA
package without any additional setup.  However, the response time is
contingent upon internet speed and the size of the requested data.

Starting with version `0.8.x`, __nhanesA__ offers two alternatives:
using a prebuilt SQL database and using a mirror.

# Using SQL database

Functions in the __nhanesA__ package can obtain (most) data from a
suitably configured Microsoft SQL Server database instead of accessing
the CDC website directly. The easiest way to obtain such a database is
to use the [docker image](https://github.com/ccb-hms/NHANES) created
as part of the Epiconductor project. This docker image includes
versions of R and RStudio, and is configured in a way that causes
__nhanesA__ to use the database when it is run inside the docker
instance.

It is also possible to configure __nhanesA__ to use a SQL database
when running _outside_ a docker instance, provided the machine has
access to the database, which could be running in a docker instance on
the same machine, or on another machine in the local network. To do
so, the following environment variables need to be define prior to
loading the __nhanesA__ package:

- `EPICONDUCTOR_CONTAINER_VERSION` (e.g., `v0.12.0`)
- `EPICONDUCTOR_COLLECTION_DATE` (e.g., `2023-11-21`)
- `EPICONDUCTOR_DB_DRIVER` (e.g., `FreeTDS` on Linux)
- `EPICONDUCTOR_DB_SERVER` (e.g., `localhost`)
- `EPICONDUCTOR_DB_PORT` (e.g., `1433`)

The first two are for information, and need not actually match the
version of the database. They indicate the date on which a snapshot of
the NHANES data was collected from the CDC website, and are defined
suitably when running inside the docker image. However, they must be
specified explicitly when trying to connect to the database from an
instance of R running outside docker.

The last three environment variables define the details of how to
connect to the database. For details, see the
[DBI](https://github.com/r-dbi/DBI) and
[odbc](https://github.com/r-dbi/odbc) packages (the latter is the
backend that allows R to communicate with a Microsoft SQL server).


## Usage 

Once a database is successfully configured (which is most easily done
by using the docker version), the __nhanesA__ package should ideally
behave similarly whether or not a database is being used. When a
database is successfully found on startup, the package sets an option
called `use.db` to `TRUE`.


```r
library(nhanesA)
nhanesOptions()
```

```
$use.db
[1] TRUE
```

Even in this case, it is possible to pause use of the database and
revert to downloading from the CDC website by setting


```r
nhanesOptions(use.db = FALSE, log.access = TRUE)
```

The `log.access` option, if set, causes a message to be printed every
time a web resource is accessed.

With these settings, we get


```r
bpq_b_web <- nhanes("BPQ_B")
```

```
Downloading: https://wwwn.cdc.gov/Nchs/Nhanes/2001-2002/BPQ_B.XPT
```

On the other hand, if we use the database, we get


```r
nhanesOptions(use.db = TRUE)
bpq_b_db <- nhanes("BPQ_B")
```

The two versions have minor differences: The order of rows and columns
may be different, and categorical variables may be represented either
as factors of character strings. However, as long as the data has not
been updated on the NHANES website since it was downloaded for
inclusion in the database, the contents should be identical.


```r
str(bpq_b_web[1:10])
```

```
'data.frame':	6634 obs. of  10 variables:
 $ SEQN   : num  9966 9967 9968 9969 9970 ...
 $ BPQ010 : Factor w/ 7 levels "Less than 6 months ago,",..: 1 1 1 1 2 2 1 1 3 1 ...
 $ BPQ020 : Factor w/ 3 levels "Yes","No","Don't know": 2 2 1 2 2 1 2 2 2 2 ...
 $ BPQ030 : Factor w/ 3 levels "Yes","No","Don't know": NA NA 1 NA NA 2 NA NA NA NA ...
 $ BPQ040A: Factor w/ 3 levels "Yes","No","Don't know": NA NA 1 NA NA 1 NA NA NA NA ...
 $ BPQ040B: Factor w/ 3 levels "Yes","No","Don't know": NA NA 2 NA NA 1 NA NA NA NA ...
 $ BPQ040C: Factor w/ 3 levels "Yes","No","Don't know": NA NA 1 NA NA 1 NA NA NA NA ...
 $ BPQ040D: Factor w/ 3 levels "Yes","No","Don't know": NA NA 2 NA NA 1 NA NA NA NA ...
 $ BPQ040E: Factor w/ 3 levels "Yes","No","Don't know": NA NA 2 NA NA 1 NA NA NA NA ...
 $ BPQ040F: Factor w/ 3 levels "Yes","No","Don't know": NA NA 2 NA NA 2 NA NA NA NA ...
```

```r
str(bpq_b_db[1:10])
```

```
'data.frame':	6634 obs. of  10 variables:
 $ SEQN   : int  9975 10025 10060 10074 10077 10093 10410 10542 10592 10593 ...
 $ BPQ010 : chr  "Less than 6 months ago" "Less than 6 months ago" "Less than 6 months ago" "Less than 6 months ago" ...
 $ BPQ020 : chr  "No" "No" "No" "No" ...
 $ BPQ030 : chr  NA NA NA NA ...
 $ BPQ040A: chr  NA NA NA NA ...
 $ BPQ040B: chr  NA NA NA NA ...
 $ BPQ040C: chr  NA NA NA NA ...
 $ BPQ040D: chr  NA NA NA NA ...
 $ BPQ040E: chr  NA NA NA NA ...
 $ BPQ040F: chr  NA NA NA NA ...
```


# Using a local mirror

A conceptually simple alternative that also avoids repetitive
downloads from the CDC website is to maintain a local mirror from
which the data and documentation files can be retrieved as needed.

As noted [here](nhanes-introduction.html), data and documentation URLs
for a particular table are determined by the table's name and the
cycle it represents. For example, the URLs for table `DEMO_C`, which
is from cycle 3, i.e., `2003-2004`, would be

- Data: <https://wwwn.cdc.gov/nchs/nhanes/2003-2004/DEMO_C.XPT>

- Documentation: <https://wwwn.cdc.gov/nchs/nhanes/2003-2004/DEMO_C.htm>

It is possible to change the "base" of the server from where
__nhanesA__ tries to download these files by setting an environment
variable called `NHANES_TABLE_BASE`, which defaults to the value
`"https://wwwn.cdc.gov"`.

The steps needed to create such a mirror is beyond the scope of this
document, but tools such as `wget`, or even the R function
`download.file()` in conjunction with the list of relevant URLs
obtained using `nhanesManifest()`, may be used to download all files
locally. Note that just downloading the files is not sufficient, and
they must also be made available through a HTTP server running
locally.


## Dynamic caching using __httpuv__ and __BiocFileCache__

Both the database and local mirroring options can get outdated when
CDC releases new files or updates old ones. The
[__BiocFileCache__](https://bioconductor.org/packages/release/bioc/html/BiocFileCache.html)
package can cache downloaded files locally in a persistent manner,
updating them automatically when the source file has been updated. The
experimental [__cachehttp__](https://github.com/ccb-hms/cachehttp) package
uses the __BiocFileCache__ package in conjunction with the
[httpuv](https://github.com/rstudio/httpuv/#readme) package to run a
local server that downloads files from the CDC website the first time
they are requested, but uses the cache for subsequent requests.

To use this package, first install it using

```r
BiocManager::install("BiocFileCache")
remotes::install_github("ccb-hms/cachehttp")
```

Then, run the following in a separate R session.

```r
require(cachehttp)
add_cache("cdc", "https://wwwn.cdc.gov",
          fun = function(x) {
              x <- tolower(x)
              endsWith(x, ".htm") || endsWith(x, ".xpt")
          })
s <- start_cache(host = "0.0.0.0", port = 8080,
                 static_path = BiocFileCache::bfccache(BiocFileCache::BiocFileCache()))
## stopServer(s) # to stop the httpuv server
```

This session must be kept active for the server to work. It can even
run on a different machine, as long as it is accessible via the
specified port, and does not require the __nhanesA__ package to work.

While the server is running, we can set (in a different R session)


```r
Sys.setenv(NHANES_TABLE_BASE = "http://127.0.0.1:8080/cdc")
```

(changing host IP and port as necessary) to use this server instead of
the primary CDC website to serve `XPT` and `htm` files. Although the
each file is downloaded from the CDC website the first time it is
requested, subsequent downloads should be faster, as indicated by the
elapsed times in the following code.


```r
nhanesOptions(use.db = FALSE, log.access = TRUE)
system.time(foo <- nhanes("DEMO"))
```

```
Downloading: http://127.0.0.1:8080/cdc/Nchs/Nhanes/1999-2000/DEMO.XPT
```

```
   user  system elapsed 
  2.237   0.112   9.991 
```

```r
system.time(foo <- nhanes("DEMO"))
```

```
Downloading: http://127.0.0.1:8080/cdc/Nchs/Nhanes/1999-2000/DEMO.XPT
```

```
   user  system elapsed 
  2.327   0.109   2.954 
```

