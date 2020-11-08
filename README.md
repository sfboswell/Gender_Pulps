# Network Analysis of Gender in Publication  

This code uses R to analyze (gendered) patterns in publication, primarily through networks. It relies on tsv files of bibliographic material. The code could be used to investigate almost any publication pattern, depending both on the user's interests, and the information within the bibliographic files. 

This software is free to use, copy, modify, and distribute under the MIT license (in other words, please credit me if you do use it). 

**The Files**  

***

Before you do anything else, you're going to need some tsv files for R to analyze. If you already have tsv files of bibliographic information, you're ahead of the curve. In the case of the project I worked on, the bibliographic data I worked with did not exist in tsv files. It existed on the Internet Science Fiction Database (ISFDB) in MySQL format. 

To replicate the process for getting these files - which is really Andrew Goldstone's process, since he initially came up with the MySQL to tsv transformation -  download  http://isfdb.s3.amazonaws.com/backups/backup-MySQL-55-2015-05-02.zip and unzip it. This will yield a big 466MB file, backup-MySQL-55-2015-05-02, which can be imported into a local MySQL database. One can then interact with the MySQL database from R using dyplr's "database connections" 

```
library ("dyplr") 
db <- sr_mysql ("isfdb")
``` 
where db = database 

you can see all the tables in the database with 
``` 
src_tbls(db)
``` 
In order to get the data-frame objects representing individual tables, you can use the ```tbl``` function. The trick here is that you need to know what information you want in each table. Because we're exploring relationships and not just individual tables, you quickly run into a problem whereupon you have one table with the publication ID in it, one table with the publication name in it, one table with the publication year in it (etc.) - and no table that brings each of these pieces of information together. In other words, you're going to need to join some tables. 

This will require exploring each table a little bit to figure out what information you want from each. For simplicity's sake, I've included how Andrew Goldstone initially created the tsv files for the gender_pulp project. 

```
authors %>% select(author_id,
           author_canonical,
           author_legalname,
           author_lastname,
           author_birthplace,
           author_birthdate,
           author_deathdate) %>%
collect() %>% mutate(author_legalname=
str_replace_all(author_legalname, regex("\t"), " ")) %>% write.table("tsv/authors.tsv", sep="\t", quote=F,
                row.names=F, fileEncoding="UTF-8")
``` 
A table of relations between authors and their titles: 

``` 
tbl(db, "canonical_author") %>% 
  collect() %>%
    write.table("tsv/canonical_author.tsv", sep="\t", 
    quote=F, row.names=F, fileEncoding="UTF-8")
```
A table for the titles (independent of authors) 

``` 
titles %>% select(title_id,
           title_title,
           title_copyright,
           title_parent) %>%
collect() %>% mutate(title_title=
str_replace_all(title_title, regex("\t"), " ")) %>% write.table("tsv/titles.tsv", sep="\t", quote=F, row.names=F,
                fileEncoding="UTF-8")
``` 
Now come the volumes, or "publications" (one magazine = one publication) 

``` 
pubs %>% select(pub_id,
           pub_title,
           pub_tag,
           pub_year,
           pub_ptype,
           pub_ctype) %>%
collect() %>%
write.table("tsv/pubs.tsv", sep="\t", quote=F, row.names=F,
            fileEncoding="UTF-8")
``` 
And the link table between publications and individual items (i.e: publications and the stories/ articles within them) 

```
tbl(db, "pub_content") %>%
collect() %>%
write.table("tsv/pubs.tsv", sep="\t", quote=F, row.names=F,
            fileEncoding="UTF-8")
``` 
Now you have a series of tsv files! 

You can explore these files using read.table, with this code: 

``` 
# filename is whatever you need it to be
read.table(filename, sep="\t", header=T, 
stringsAsFactors=F, quote="", comment.char="", 
encoding="UTF-8", fileEncoding="UTF-8")

``` 

Crucially, when you move on to the network analysis portion, the tsv folder of the bibliographic data must be in the same folder as the filename.rmd., or else everything else goes to hell in a handbasket. Choose your working directory with care. 

**Creating the networks** 

Once you've got tsv files with bibliographic (or other) data, you can move on to creating networks. My process for creating networks of publishing relationships went as follows: 

First, I read in the data from the tsv files (again, this is why it's crucial to choose your working directory with care, and to make sure the tsv folder is in the same folder as your file). 

The author data: 
``` 
remember that "tsv/author.tsv" will look different depending on your filename 

authors <- read.table("tsv/authors.tsv", sep="\t", 
                      header=T, stringsAsFactors=F, quote="", 
                      comment.char="", encoding="UTF-8", fileEncoding="UTF-8")
```

Publication data: 

``` 
#remember that "tsv/pubs.tsv" will look different depending on your filename 

pubs <- read.table("tsv/pubs.tsv", sep="\t", 
                   header=T, stringsAsFactors=F, quote="", 
                   comment.char="", encoding="UTF-8", fileEncoding="UTF-8")
``` 
Titles data 

```
#remember that "tsv/titles.tsv" will look different depending on your filename 

titles <- read.table("tsv/titles.tsv", sep="\t", 
                     header=T, stringsAsFactors=F, quote="", 
                     comment.char="", encoding="UTF-8", fileEncoding="UTF-8")
``` 
Repeat as needed for every table you need - which in this case was canonical_authors and pub_content. At the end, you'll have five "tables" accessible by your code - "titles," "pubs," "authors," "canonical_authors," and "pub_content." 

You may want to filter down magazines or publications of interest at this point. In the case of this project, ISFDB contains thousands more magazines than any network could possibly handle - so I choose a sampling to focus on, using ```str_detect``` to pull information out of ```pubs```. 

``` 
our_canon <- pubs  %>% 
           filter(str_detect(pub_title, ("^Amazing Stories|^Astounding Stories|^Wonder Stories|^Thrilling Wonder|^Planet Stories|^Startling Stories|^Fantastic       Adventures|^Weird Tales|^Astounding Science-Fiction")))
```
Having created a new canon ```our_canon```, now it's time to make sure the publication information we pulled out is tied to information about titles and authors 

``` 
#join publication information to information about the titles

our_canon_df <- pub_content  %>% inner_join (our_canon, by="pub_id")  %>%
inner_join (titles, by="title_id")

#join that dataframe to information about the authors

our_canon_df <- canonical_authors  %>%
inner_join (our_canon_df, by="title_id")  %>%
inner_join (authors, by="author_id")

``` 

