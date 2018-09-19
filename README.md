# Clawling Liferay with Apache Nutch 1.x

Those are the steps to obtain content from Liferay by crawling a portal with Nutch.

## Downloading and setting up
1. Download Nutch from [here](http://www.apache.org/dyn/closer.cgi/nutch/).
2. Extract the contents of the file contents. We will call the resulting directory `$NUTCH_HOME` and it will be named as something like `apache-nutch-1.123`.
3. To ensure the unpacked content is working, run `NUTCH_HOME/bin/nutch`. Should get something like this

        Usage: nutch COMMAND where command is one of:
        readdb            read / dump crawl db
        mergedb           merge crawldb-s, with optional filtering
        readlinkdb        read / dump link db
        inject            inject new urls into the database
        generate          generate new segments to fetch from crawl db
        freegen           generate new segments to fetch from text files
        fetch             fetch a segment's pages
        Set up

4. You can give a name for the cluster, by adding the lines below to `$NUTCH_HOME/conf/nutch-site.xml`:

        <property>
         <name>http.agent.name</name>
         <value>My Nutch Spider</value>
        </property>

5. We tried to test Nutch on a local Liferay, but there were problems making Nutch crawl the “localhost” domain. So we added example.com to `/etc/hosts`, pointing to localhost?:

        127.0.0.1   example.com

6. Let’s only accpet URLs in `example.com`, so in `$NUTCH_SITE/conf/regex-urlfilter.xml` we replace

        +.

    with

        +^http://example.com:8080/
7. Liferay uses a lot of query strings, but the defaut Nutch setup filter them out. We change this configuration in `$NUTCH_SITE/conf/regex-urlfilter.xml`, commenting out the line:

        #-[?*!@=]

## Seeding and crawling
1. Let’s create some seed URLs. Those are the URLs to be first retrieved, and from them we get the links to other pages. The commands below are enough:

        mkdir -p $NUTCH_HOME/urls
        echo 'http://example.com:8080/' > $NUTCH_HOME/urls/seed.txt

2. We should “inject” the seed URLs into the crawl database (or “crawldb”). Crawldb has information about every URL Nutch already knows. It is from there that Nutch takes URLs to check. To inject the seed URLs there, just call

        $NUTCH_HOME/bin/nutch inject $NUTCH_HOME/crawl/crawldb $NUTCH_HOME/urls/seed.txt

    With this, a directory called `crawl/` is created, and inside it another one, `crawldb`, that stores the data.

3. Nutch has the concept of “segments:” a set of URLs that are grabbed together. We suspect the URLs are bundled this way so different segments can be distributed through a cluster. We need to generate a segment to make the search:

        $NUTCH_HOME/bin/nutch generate $NUTCH_HOME/crawl/crawldb $NUTCH_HOME/crawl/segments

    This will create a `$NUTCH_HOME/crawl/segments` directory, and inside it another one whose name is a timestamp. There is our segment.

4. Given a segment, we have to fetch and parse it. So we use the fetch command with the path of the segment:

        $NUTCH_HOME/bin/nutch fetch $NUTCH_HOME/crawl/segments/20180905171629/

5. And then we use the `parse` command:

        $NUTCH_HOME/bin/nutch parse $NUTCH_HOME/crawl/segments/20180905171629/

    Those commands download and process pages, and save the data in the segment’s directory.

6. We can use the new segment to populate again the crawldb. That’s the magic of Nutch ;) We just need the updatedb command:

        $NUTCH_HOME/bin/nutch updatedb $NUTCH_HOME/crawl/crawldb  $NUTCH_HOME/crawl/segments/20180905171629/

    After that, we can generate a new segment, fetch it, parse it etc. as usual.

## Reading data from CrawlDB and segments
We can take a peek at the database content with the `readdb` command. With it, we can

1. see some statistics with this command:

        $NUTCH_HOME/bin/nutch readdb $NUTCH_HOME/crawl/crawldb -stats

2. Get a dump of the content of the database:

        $NUTCH_HOME/bin/nutch readdb $NUTCH_HOME/crawl/crawldb -dump output-directory/

    This will create a directory named `output-directory/` and inside it a file with the data.

3. Take a listing of URLs only (since a dump is a lot of info):

        $NUTCH_HOME/bin/nutch readdb $NUTCH_HOME/crawl/crawldb -topN 100 output-directory

4. request info about only one URL:

        $NUTCH_HOME/bin/nutch readdb $NUTCH_HOME/crawl/crawldb -url http://example.com:8080/

5. We can spect the segment data with the readseg command. For example, this one below:

        $NUTCH_HOME/bin/nutch readseg -dump $NUTCH_HOME/crawl/segments/20180905171629/ output-directory

    This generates a dump in the output-directory. An example of such a dump (which has both content and metadata) can be seen in [this document](https://drive.google.com/file/d/1SCRNjHt--EGglvKrA_wLozpXTEss_HbD/view).

## Commands summary:

        bin/nutch inject crawl/crawldb urls
        bin/nutch generate crawl/crawldb crawl/segments
        bin/nutch fetch crawl/segments/20180905182415/
        bin/nutch parse crawl/segments/20180905182415/
        bin/nutch updatedb crawl/crawldb crawl/segments/20180905182415/

## References
* [Nutch Tutorial](https://wiki.apache.org/nutch/NutchTutorial).

