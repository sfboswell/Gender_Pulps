# Network Analysis of Gender in Publication  

This code uses R to analyze (gendered) patterns in publication, primarily through networks. It relies on tsv files of bibliographic material. The code could be used to investigate almost any publication pattern, depending both on the user's interests, and the information within the bibliographic files. 

I used this code in order to analyze gendered patterns of publication in pulp science fiction magazines (1926-1946) for an article forthcoming in *Extrapolation.* You can follow along my process through this README.md file, and read the entire code at "pulp project attempt.Rmd."

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

**Prepare the data for the networks** 
*** 

Once you've got tsv files with bibliographic (or other) data, you can move on to creating networks. But before you can even create some networks, you've got to filter the data. There's a tremendous amount of information within the tsv files, and only a small percentage of it will be of interest. 

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
ISFDB's database had a lot of small inconsistencies (for example, the magazine "Astounding" was called  "Astounding Science Fiction" for one entry
and "Astounding Science-Fiction" for the next. Of course, Wonder Stories (for example) had multiple names over its run, so I needed a way to search for all of them at once. In order to make it possible to create networks of *all* the instances of Astounding (or any other magazine), I 
1. changed the names of the publication to one word for easier graphing/ searching using ```mutate``` and ```gsub``` 
2. changed names of authors/magazines to compensate for database inconsistencies. 

Some of those inconsistencies I only discovered mid-way through the research (for example, I'd make a network where I *knew* John Campbell should appear, and then he wasn't there, which is how I discovered his name had multiple spellings. The smoothing out of the bibliographic data was an iterative process. 

```
giant_network <- our_canon_df   %>%  
  mutate (pub_title= gsub ("Weird Tales.*", "Weird", pub_title))  %>% 
  mutate (pub_title =gsub ("Amazing Stories, .*", "Amazing", pub_title))  %>%  
  mutate (pub_title=gsub ("Amazing Stories.*, .*", "Amazing", pub_title)) %>% 
  mutate (pub_title=gsub (".*Wonder Stories, .*", "Wonder", pub_title))  %>% 
  mutate (pub_title=gsub ("Wonder Stories.*, .*", "Wonder", pub_title))  %>% 
  mutate (pub_title=gsub ("Astounding Stories .*, .*", "Astounding", pub_title))   %>% 
  mutate (pub_title=gsub ("Astounding Science-Fiction, .*", "Astounding", pub_title)) %>%
  mutate (pub_title=gsub ("Astounding Stories, .*", "Astounding", pub_title)) %>%
  mutate (pub_title=gsub ("Startling Stories, .*", "Startling", pub_title)) %>%
  mutate (author_canonical=gsub ("^John W. .*", "Campbell", author_canonical)) %>%
    mutate (pub_title=gsub ("^Planet Stories, .*", "Planet", pub_title)) %>%
   mutate (pub_title=gsub ("^Fantastic Adventures, .*", "Fantastic", pub_title)) 
   ``` 
Now you should have the basic data, represented here by ```giant_network```, to create publishing networks. Obviously, if you were interested in different magazines, you could play around with substitutions - the limits here are only within the database itself. 

**Creating the Networks** 
*** 

Finally, it's time to create the networks. Here, your interests will dictate what kind of networks you create. Assuming you're interested in replicating my process, here are the networks I created. 

First, boilerplate information for RStudio to generate the figures (the networks) - I wanted something square. 
``` 
{r, fig.height=8, fig.width=8, fig.cap= "All authors with more than 5 works (1926-1946)" }
``` 
Then I selected the years I was interested in (```filter (str_detect (pub_year) ``` )- in this case, I wanted to know publication patterns for the SF pulp period to the golden age (1926-1946). These years changed over the nearly half dozen years I worked on this project, but this ten year spread ended up being the easiest to work within. 


I also filtered the data so it only included authors who wrote 5 or more works (```filter (n() > 5) %>%```) during the 1926-1946 period. Before I did this, the networks were nigh-unreadable because there were just so many authors.  

``` 
giant_network_df <- giant_network  %>%  
  select (pub_year, pub_title, author_canonical) %>%
  filter (str_detect (pub_year, ("^1926|^1927|^1928|^1929|^1930|^1931|^1932|^1933|^1934|^1935|^1936|^1937|^1938|^1939|^1940|^1941|^1941|^1942|^1943|^1944|^1945|^1946")))  %>%
  distinct ()  %>% group_by (author_canonical)  %>% filter (n() > 5) %>%
  select (pub_title, author_canonical)
  ``` 
  The next task is to convert the data into a matrix, and then the matrix into a graph. I simplified the edges of the network so that there was one edge per author connection to a publication (rather than one edge per publication). In other words, if an author (say, H.P. Lovecraft) ever published in *Weird Tales* , they were connected to the magazine through an edge. Unsimplified, if Lovecraft published in *Weird Tales* 50 times, he would have 50 edges, whereas if he published one time, he would have one edge. 
  
  ``` 
  (set.seed(42)) 

 giant_network_matrix <- as.matrix (giant_network_df) 
 giant_network_graph <- graph.edgelist (giant_network_matrix, directed=F) 
  giant_network_graph <- simplify (giant_network_graph)
  
  ``` 
 
For *aesthetic* purposes - and, in fact, for access purposes - I added both shapes and colors to the nodes. This proved to be one of the most difficult parts of the process. While the printed version of the article version of this project will appear in B/W, if you do network analysis of any stripe, I highly recommend that you not only commit to color, but consider accessibility (i.e: which colors work for colorblind readers). 

``` 
#add shapes to nodes
V(giant_network_graph)$size <- 3
vshape <- rep("circle", vcount(giant_network_graph))
vshape [V(giant_network_graph)$name == "Astounding"] <- "rectangle"
vshape [V(giant_network_graph)$name=="Amazing"]<-"rectangle"
vshape [V(giant_network_graph)$name=="Weird"]<-"square"
vshape [V(giant_network_graph)$name=="Wonder"]<-"square"
vshape [V(giant_network_graph)$name=="Fantastic"]<-"rectangle"
vshape [V(giant_network_graph)$name=="Startling"]<-"square"
vshape [V(giant_network_graph)$name=="Planet"]<-"rectangle"
V(giant_network_graph)$shape <-vshape

#add colors to nodes

vcolors <- rep("orange", vcount(giant_network_graph))
vcolors [V(giant_network_graph)$name == "Astounding"] <- "yellow"
vcolors [V(giant_network_graph)$name == "Amazing"] <- "cyan"
vcolors [V(giant_network_graph)$name == "Weird"] <- "mediumvioletred"
vcolors [V(giant_network_graph)$name == "Wonder"] <- "plum"
vcolors [V(giant_network_graph)$name == "Fantastic"] <- "gray"
vcolors [V(giant_network_graph)$name == "Startling"] <- "red"
vcolors [V(giant_network_graph)$name == "Planet"] <- "springgreen4"
``` 
Finally, we plot the network! 
```
plot (giant_network_graph, vertex.color=vcolors, vertex.size=4, vertex.label=NA) 
``` 
This particular network will show the "prolific" authors who published in SF magazines between 1926-1946, and which magazines they published in. 

Other networks in this project were created by changing the number of publications for authors (```filter (n() > ?)```), the years of interest, and even unsimplyfying the network. 

**Gender in the Networks** 
*** 

The major variable within the networks was tracking how *women* published in SF. The trickiest thing to studying gender and publication patterns is that ISFDB does not track gender. If you recall the creation of the tsv files, where 

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
None of the database informations contains anything about gender (there's more information contained in the ISFDB files than we pulled into the tsv files - for example, the author's birthplace - but none of it was useful). 

Because the database had no way to filter by gender, in this case, I relied on archival research - my own, and that done by other SF scholars - to create a list of female pulp writers active in the 1926-1946 period. I then used the power of ```str_detect``` to find these women and create new networks focused on their publication patterns - not without some bumps along the way. Here's an example: 

``` 
#filter known female authors and years of interest

pulp_women_df <- giant_network  %>%  
  select (pub_year, pub_title, author_canonical) %>%
  filter (str_detect (author_canonical, ("^Leigh Brackett|^Frances M. Deegan|^Merab Eberle|^Sophie Wenzel Ellis|^Mona Farnsworth|^Mrs. Lee Hawkins Garby|^Frances Garfield|^L. Taylor Hansen|^Clare Winger Harris|^Hazel Heald|^E. Mayne Hull|^Amelia Reynolds Long|^Lilith Lorraine|^Kathleen Ludwick|^C. L. Moore|^Dorothy Quick|^Kaye Raymond|^Margaretta W. Rea|^Jane Rice|^Louise Rice|^Margaret Ronan|^Babette Rosmond|^Margaret St. Clair|^Lewis Padgett|^G. St. John-Loe|^I. M. Stephens|^Leslie F. Stone|^Doris Thomas|^Emma Vanne|^Ruth Washburn|^Helen Weinbaum|^Gabriel Wilson|^Lillian M. Ainsworth|^Thaedra Alden|^Pansy E. Black|^Dorothy Donnell Calhoun|^Clara E. Chesnutt|^Mollie Claire|^Clemence Dane|^Dorothy de Courcy|^Theodora Du Bois|^Norma Lazell Easton|^Augusta Groner|^Inez Haynes Irwin|^Millicent Holmberg|^Minna Irving|^Jessie Douglas Kerruish|^LesTina|^Winona McClintic|^Katherine Maclean|^Judith Merril|^Keith Hammond|^Hudson Hastings|^Maria Moravsky|^Marian O'Hearn|^Leslie Perri|^Margaretta W. Rea|^Amabel Redman|^Jane Rice|^Louise Rice|^Margaret Rogers|^M. F. Rupert|^Wilmar H. Shiras|^Francis Stevens|^Elma Wentz|^Gabriel Wilson|^Laura Withrow|^Frances Yerxa|^Vida Taylor Adams|^Marguerite Lynch Addis|^Edith M. Almedingen|^Frances Arthur|^Meredith Beyers|^Anne M. Bilbro|^Zealia B. Bishop|^Lady Anne Bonny|^Edna Goit Brintnall|^Mary S. Brown|^Dulcie Browne|^Loretta G. Burrough|^Brooke Byrne|^Grace M. Campbell|^Lenore E. Chaney|^Valma Clark|^Martha May Cockrill|^Ethel Helene Coen|^Eli Colter|^Mary Elizabeth Counselman|^Florence Crow|^Marjorie Darter|^Meredith Davis|^Miriam Allen de Ford|^Edith de Garis|^Leah Bodine Drake|^Elsie Ellis|^Mollie Frank Ellis|^Sophie Wenzel Ellis|^Betsy Emmons|^Mary McEnnery Erhard|^Caroline Evans|^Alice Drayton Farnham|^Effie W. Fifield|^Frances Garfield|^Louise Garwood|^Elizabeth Cleghorn Gaskell|^Myrtle Levy Gaylord|^Dorothea Gibbons|^Victoria Glad|^Gertrude Gordon|^Sonia H. Greene|^Allison V. Harding|^Lyllian Huntley Harris|^Margaret McBride Hoss|^Helen Rowe Henze|^Vennette Herron|^Terva Gaston Hubbard|^Mindret Lord|^Maebelle McCalment|^Laurie McClintock|^Sylvia Leone Mahler|^Isa-Belle Manzer|^Rachael Marshall|^Kadra Maysi|^Violet M. Methley|^Frances Bragg Middleton|^Bassett Morgan|^Sarah Newmeyer|^Dorothy Norwich|^Alice Olsen|^G. G. Pendarves|^Stella G. S. Perry|^Suzanne Pickett|^Mearle Prout|^Edith Lyle Ragsdale|^Ellen M. Ramsay|^Alicia Ramsey|^Sybla Ramus|^Helen M. Reid|^Susan Andrews Rice|^Eudora Ramsay Richardson|^Flavia Richardson|^Jean Richepin|^Katharine Metcalf Roof|^Gretchen Ruediger|^Edgar Saltus|^Sylvia B. Saltzberg|^Jane Scales|^Mary Sharon|^Schnirring|^Edna Bell Seward|^Ann Sloan|^Mrs. Chetwood Smith|^Lady Eleanor Smith|^Mrs. Harry Pugh Smith|^Emma-Lindsay Squier|^Marjorie Murch Stanley|^Francis Stevens|^Edith Lichty Stewart|^Pearl Norton Swet|^Gertrude Macaulay Sutton|^Tessida Swinges|^Fanny Kemble Johnson|^Mildred Johnson|^Helen W. Kasson|^Theda Kenyon|^Ida M. Kier|^Lois Lane|^Genevieve Larsson|^Greye La Spina|^Nadia Lavrova|^Helen Liello|^Signe Toksvig|^Louise Van de Verg|^Marilyn Venable|^Evangeline Walton|^Elizabeth Adt Wenzler|^Phyllis A. Whitney|^Everil Worrell|^Stella Wynne|^Katherine Yates|^Aalla Zaata|^L. M. Montgomery|^O. M. Cabral"))) %>%
  filter (str_detect (pub_year, ("^1926|^1927|^1928|^1929|^1930|^1931|^1932|^1933|^1934|^1935|^1936|^1937|^1938|^1939|^1940|^1941|^1942|^1943|^1944|^1945|^1946"))) %>%
  select (pub_title, author_canonical)

#turn data into matrix, prepare for graphing

(set.seed(42))

pulp_women_matrix <- as.matrix (pulp_women_df) 
pulp_women_graph <- graph.edgelist (pulp_women_matrix, directed=F)
pulp_women_graph <- simplify (pulp_women_graph)

(...add shapes and colors to graph) 

plot (pulp_women_graph, vertex.color=vcolors, vertex.label=NA, vertex.size=5) 
```
This allowed me to get a pretty good measure of women's publication patterns in the 1926-1946 period in the SF pulps (and then to slice that data in different ways). 

There are some big awkwardnesses with this code, the most of obvious of which is the overreliance on ```str_detect```. The use of ```str_detect``` for both filtering out women and years gave me some huge regular expressions to write, rewrite and copy. In the future, I would have first converted the string format dates to a numerical format (potentially ``` mutate(year=as.numeric(str_sub(pubdate, 1, 4)))```. Then you could limit the date range with ```filter(year >= 1938 & year <= 1946)```. Authors would be trickier, but you could write out a vector of author names, or store it in a separate text file, and then filter with ```filter (authors %in% women)```. 

The entirety of the code I used to create every network is found on github, under "pulp project attempt.Rmd." (with annotations). Much credit is due to Rebecca Lipperini, my initial partner in codewriting (back in 2015), and to Andrew Goldstone, both for the tsv files and for help at critical junctures. The networks to which this code is attached are forthcoming in an article in *Extrapolation.* 
