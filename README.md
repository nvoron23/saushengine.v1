# SaushEngine

Most people who know me realises at some point in time that I go through cycles where I dive into things that catch my imagination and obsess over it for days, weeks or even months. Some things like origami and programming never really go away while others like bread-making make cometary orbits that come once in a few years. In the past long gone, this was fueled mainly through reading copious amounts of books and magazines, but since the arrival of the Internet, my obsessive cycles reach new peaks. It allowed me to reach out to like minded people and provided a massive amount of data that I can never hope to conquer.

Fortunately with search engines, I don't need to conquer them by myself. Search engines are almost ubiquitous on the Internet, the most famous one has since become a verb. In June 2006, the verb 'to google' was officially added to the Oxford English Dictionary. The barrier to entry for obsessing over something is now nothing more than an Internet connection away. So it's only natural that search engines has now become my new obsessive interest. This of course has been enhanced in particular by the fact that I work for Yahoo!, who owns second most popular web search engine in the world.

The topics on search engines are pretty vast. Much of it is centered around search engine optimization (SEO), which contrary to its name, is not about optimizing search engines but about optimizing web sites to make it rise to the top of a search engine results list. However this is not what I'm interested in (at this moment). What I'm interested in is how search engines work, what makes it tick, basically the innards of a search engine. As part of my research, I wrote a search engine using Ruby, called SaushEngine, which I will describe in full here.

Surprisingly, as I delved deeper into the anatomy of a search engine, most search engines are pretty much the same. Sure, the implementation can be vastly different but the basic ideas are pretty much universal in all the search engines I looked at, including web search engines such as Yahoo! and Google as well as standalone ones such as Sphinx, ht://Dig, Lucence and Nutch. In essence, any search engine has these 3 parts:

* Crawl - the part of the search engine that goes around extracting data from various data sources that are being searched
* Index - the part of the search engine that processes the data that has been extracted during the crawl and stores it
* Query - the part of the search engine that allows a user to query the search engine and result ranked results

This article will describe how each part works and how my Ruby search engine implements it. SaushEngine (pun intended) is a simple web search engine, that is around 200 lines of Ruby code. However I do cheat a bit and use various libraries extensively including Hpricot, DataMapper and the Porter Stemmer. SaushEngine is a web search engine which means it goes out to Internet and harvests data on Internet web sites. Having said that, it's definitely not production grade and lacks much of the features of a proper search engine (including high performance) in exchange for a simpler implementation.

## Crawl

Let's start with the crawl phase of SaushEngine. SaushEngine has a web crawler typically called Spider (spider crawls the web -- get it? I know, it's an old joke), in 2 files named `spider.rb` and `index.rb`. This crawler doesn't just extract the information only though, it partly processes the data as well. It is designed this way to simplify the overall design. In this section we'll concentrate on the crawling parts first.

There are a number of things to consider when designing a crawler, the most important amongst which are:

* How do we crawl?
* What kind of documents do we want to get?
* Which sites are we allowed to visit?
* How do we re-crawl and how often?
* How fast can we crawl?

Many search engine designs approach crawling modularly by detaching the dependency of the crawler to the index and query design. For example in Lucene, the crawler is not even part of the search engine and the implementer is expected to provide the data source. On the other hand, some crawlers are not built for search engines, but for research into the Internet in general. In fact, the first crawler, the World Wide Web Wanderer, was deployed in 1993 to measure the size of the Internet. As mentioned before, to simplify the design, SaushEngine's crawler is an integral part of the search engine and the design is not modular. The subject of web crawlers is a topic of intensive research, with tons of existing literature dating from the late 1990s till today. Without going extensively into them, let's see how SaushEngine is designed to answer some of those questions.

### How do we crawl?

This is the central question. Google claimed in a blog entry in the official Google blog in July 2008 that they found 1 trillion unique URLs. That's a whole lot of web pages. As a result, there is always the question of the value of visiting a particular page and prioritizing which sites to crawl are important to get the strategic sites in an evenly distributed way.

Most crawlers work by starting with bunch 'seed' URLs and following all links in those seeds, so this means choosing the correct seeds are critical to having a good index. SaushEngine uses a breadth-first strategy in crawling sites from its seed URLs.

While we're discussing the generic Internet crawler here, there exist a more focused type of crawler that only crawls specific sites or topics etc. For example, I could crawl only food-related sites or anime-related sites. The specifics in implementation will be different but the core mechanism will remain the same.

### What kind of documents do we want to get?

Obviously for an Internet web crawler, we what are looking are all the web pages on the various web sites on the Internet. Some more advanced search engines include Word documents, Acrobat documents, Powerpoint presentations and so on but predominantly the main data source are HTML-based web pages that are reference by URLs. The SaushEngine design only crawls for html documents and ignore all other types of documents. This has an impact when we go into the indexing phase. Besides blocking off URLs that end with certain types such as .doc, .pdf, .avi etc, SaushEngine also rejects URLs with '?', '&' and '/cgi-bin/' and so on, in them as the likelihood of generated content or it being a web application is high.

### Which sites are we allowed to visit?

A major concern many web sites have against web crawlers are that they will consume bandwidth and incur costs for the owners of the various site it crawls. As a response to this concern, the Robots Exclusion Protocol was created where a file named robots.txt is placed at the root level of a domain and this file tells compliant crawlers which parts of the site is allowed or not allowed. SaushEngine parses the robots.txt file for each site to extract the permissions and complies with the permissions in the file.

### How do we re-crawl and how often?

Web pages are dynamic. Content in web pages can change regularly, irregularly or even never. A good search engine should be able to balance between the freshness of the page (which makes it more relevant) to the frequency of the visits (which consumes resources). SaushEngine make a simple assumption that any page should be revisited if it is older than 1 week (a controllable parameter).

### How fast can we crawl?

Performance of the crawler is critical to the search engine. It determines the number of pages in its index, which in turn determines how useful and relevant the search engine is. The SaushEngine crawler is rather slow because it's not designed for speed but for ease of understanding of crawler concepts.

Let's go deeper into the design. This is the full source code for index.rb

```ruby
require 'rubygems'
require 'dm-core'
require 'dm-more'
require 'stemmer'
require 'robots'
require 'open-uri'
require 'hpricot'

DataMapper.setup(:default, 'mysql://root:root@localhost/saush')
FRESHNESS_POLICY = 60 * 60 * 24 * 7 # 7 days

class Page
  include DataMapper::Resource

  property :id,          Serial
  property :url,         String, :length => 255
  property :title,       String, :length => 255
  has n, :locations
  has n, :words, :through => :locations
  property :created_at,  DateTime
  property :updated_at,  DateTime  

  def self.find(url)
    page = first(:url => url)
    page = new(:url => url) if page.nil?
    return page
  end

  def refresh
    update_attributes({:updated_at => DateTime.parse(Time.now.to_s)})
  end

  def age
    (Time.now - updated_at.to_time)/60
  end

  def fresh?
    age > FRESHNESS_POLICY ? false : true 
  end
end

class Word
  include DataMapper::Resource

  property :id,         Serial
  property :stem,       String
  has n, :locations
  has n, :pages, :through => :locations

  def self.find(word)
    wrd = first(:stem => word)
    wrd = new(:stem => word) if wrd.nil?
    return wrd
  end
end

class Location
  include DataMapper::Resource

  property :id,         Serial
  property :position,   Integer

  belongs_to :word
  belongs_to :page
end

class String
  def words
    words = self.gsub(/[^\w\s]/,"").split
    d = []
    words.each { |word| d << word.downcase.stem unless (COMMON_WORDS.include?(word) or word.size > 50) }
    return d
  end

  COMMON_WORDS = ['a','able','about','above','abroad','according','accordingly','across','actually','adj','after','afterwards','again','against','ago','ahead','aint','all','allow','allows','almost','alone','along','alongside','already','also','although','always','am','amid','amidst','among','amongst','an','and','another','any','anybody','anyhow','anyone','anything','anyway','anyways','anywhere','apart','appear','appreciate','appropriate','are','arent','around','as','as','aside','ask','asking','associated','at','available','away','awfully','b','back','backward','backwards','be','became','because','become','becomes','becoming','been','before','beforehand','begin','behind','being','believe','below','beside','besides','best','better','between','beyond','both','brief','but','by','c','came','can','cannot','cant','cant','caption','cause','causes','certain','certainly','changes','clearly','cmon','co','co.','com','come','comes','concerning','consequently','consider','considering','contain','containing','contains','corresponding','could','couldnt','course','cs','currently','d','dare','darent','definitely','described','despite','did','didnt','different','directly','do','does','doesnt','doing','done','dont','down','downwards','during','e','each','edu','eg','eight','eighty','either','else','elsewhere','end','ending','enough','entirely','especially','et','etc','even','ever','evermore','every','everybody','everyone','everything','everywhere','ex','exactly','example','except','f','fairly','far','farther','few','fewer','fifth','first','five','followed','following','follows','for','forever','former','formerly','forth','forward','found','four','from','further','furthermore','g','get','gets','getting','given','gives','go','goes','going','gone','got','gotten','greetings','h','had','hadnt','half','happens','hardly','has','hasnt','have','havent','having','he','hed','hell','hello','help','hence','her','here','hereafter','hereby','herein','heres','hereupon','hers','herself','hes','hi','him','himself','his','hither','hopefully','how','howbeit','however','hundred','i','id','ie','if','ignored','ill','im','immediate','in','inasmuch','inc','inc.','indeed','indicate','indicated','indicates','inner','inside','insofar','instead','into','inward','is','isnt','it','itd','itll','its','its','itself','ive','j','just','k','keep','keeps','kept','know','known','knows','l','last','lately','later','latter','latterly','least','less','lest','let','lets','like','liked','likely','likewise','little','look','looking','looks','low','lower','ltd','m','made','mainly','make','makes','many','may','maybe','maynt','me','mean','meantime','meanwhile','merely','might','mightnt','mine','minus','miss','more','moreover','most','mostly','mr','mrs','much','must','mustnt','my','myself','n','name','namely','nd','near','nearly','necessary','need','neednt','needs','neither','never','neverf','neverless','nevertheless','new','next','nine','ninety','no','nobody','non','none','nonetheless','noone','no-one','nor','normally','not','nothing','notwithstanding','novel','now','nowhere','o','obviously','of','off','often','oh','ok','okay','old','on','once','one','ones','ones','only','onto','opposite','or','other','others','otherwise','ought','oughtnt','our','ours','ourselves','out','outside','over','overall','own','p','particular','particularly','past','per','perhaps','placed','please','plus','possible','presumably','probably','provided','provides','q','que','quite','qv','r','rather','rd','re','really','reasonably','recent','recently','regarding','regardless','regards','relatively','respectively','right','round','s','said','same','saw','say','saying','says','second','secondly','see','seeing','seem','seemed','seeming','seems','seen','self','selves','sensible','sent','serious','seriously','seven','several','shall','shant','she','shed','shell','shes','should','shouldnt','since','six','so','some','somebody','someday','somehow','someone','something','sometime','sometimes','somewhat','somewhere','soon','sorry','specified','specify','specifying','still','sub','such','sup','sure','t','take','taken','taking','tell','tends','th','than','thank','thanks','thanx','that','thatll','thats','thats','thatve','the','their','theirs','them','themselves','then','thence','there','thereafter','thereby','thered','therefore','therein','therell','therere','theres','theres','thereupon','thereve','these','they','theyd','theyll','theyre','theyve','thing','things','think','third','thirty','this','thorough','thoroughly','those','though','three','through','throughout','thru','thus','till','to','together','too','took','toward','towards','tried','tries','truly','try','trying','ts','twice','two','u','un','under','underneath','undoing','unfortunately','unless','unlike','unlikely','until','unto','up','upon','upwards','us','use','used','useful','uses','using','usually','v','value','various','versus','very','via','viz','vs','w','want','wants','was','wasnt','way','we','wed','welcome','well','well','went','were','were','werent','weve','what','whatever','whatll','whats','whatve','when','whence','whenever','where','whereafter','whereas','whereby','wherein','wheres','whereupon','wherever','whether','which','whichever','while','whilst','whither','who','whod','whoever','whole','wholl','whom','whomever','whos','whose','why','will','willing','wish','with','within','without','wonder','wont','would','wouldnt','x','y','yes','yet','you','youd','youll','your','youre','yours','yourself','yourselves','youve','z','zero']  
end

DataMapper.auto_migrate! if ARGV[0] == 'reset'
```

Note the various libraries I used for SaushEngine:

* DataMapper (dm-core and dm-more)
* Stemmer
* Robots
* Hpricot

I also use the MySQL relational database as the index (which we'll get to later). In the Page class, note the fresh?, age and refresh methods. The fresh? method checks if the page is fresh or not, and the freshness of the page is determined by the age of the page since it was last updated. Also note that I extended the String class by adding a words method that returns a stem of all the words from the string, excluding an array of common words or if the word is really long. I create the array of word stems using the Porter stemmer. We'll see how this is used in a while.

Now take a look at spider.rb, which is probably the largest file in SaushEngine, around 100 lines of code:

```ruby
require 'rubygems'
require 'index'

LAST_CRAWLED_PAGES = 'seed.yml'
DO_NOT_CRAWL_TYPES = %w(.pdf .doc .xls .ppt .mp3 .m4v .avi .mpg .rss .xml .json .txt .git .zip .md5 .asc .jpg .gif .png)
USER_AGENT = 'saush-spider'

class Spider

  # start the spider
  def start
    Hpricot.buffer_size = 204800
    process(YAML.load_file(LAST_CRAWLED_PAGES))
  end

  # process the loaded pages
  def process(pages)
    robot = Robots.new USER_AGENT
    until pages.nil? or pages.empty? 
      newfound_pages = []
      pages.each { |page|
        begin
          if add_to_index?(page) then          
            uri = URI.parse(page)
            host = "#{uri.scheme}://#{uri.host}"
            open(page, "User-Agent" => USER_AGENT) { |s|
              (Hpricot(s)/"a").each { |a|                
                url = scrub(a.attributes['href'], host)
                newfound_pages << url unless url.nil? or !robot.allowed? url or newfound_pages.include? url
              }
            } 
          end
        rescue => e 
          print "\n** Error encountered crawling - #{page} - #{e.to_s}"
        rescue Timeout::Error => e
          print "\n** Timeout encountered - #{page} - #{e.to_s}"
        end
      }
      pages = newfound_pages
      File.open(LAST_CRAWLED_PAGES, 'w') { |out| YAML.dump(newfound_pages, out) }
    end    
  end

  # add the page to the index
  def add_to_index?(url)
    print "\n- indexing #{url}" 
    t0 = Time.now
    page = Page.find(scrub(url))

    # if the page is not in the index, then index it
    if page.new_record? then    
      index(url) { |doc_words, title|
        dsize = doc_words.size.to_f
        puts " [new] - (#{dsize.to_i} words)"
        doc_words.each_with_index { |w, l|    
          printf("\r\e - %6.2f%",(l*100/dsize))
          loc = Location.new(:position => l)
          loc.word, loc.page, page.title = Word.find(w), page, title
          loc.save
        }
      }

    # if it is but it is not fresh, then update it
    elsif not page.fresh? then
      index(url) { |doc_words, title|
        dsize = doc_words.size.to_f
        puts " [refreshed] - (#{dsize.to_i} words)"
        page.locations.destroy!
        doc_words.each_with_index { |w, l|    
          printf("\r\e - %6.2f%",(l*100/dsize))
          loc = Location.new(:position => l)
          loc.word, loc.page, page.title = Word.find(w), page, title
          loc.save
        }        
      }
      page.refresh
  
    #otherwise just ignore it
    else
      puts " - (x) already indexed"
      return false
    end
    t1 = Time.now
    puts "  [%6.2f sec]" % (t1 - t0)
    return true          
  end

  # scrub the given link
  def scrub(link, host=nil)
    unless link.nil? then
      return nil if DO_NOT_CRAWL_TYPES.include? link[(link.size-4)..link.size] or link.include? '?' or link.include? '/cgi-bin/' or link.include? '&' or link[0..8] == 'javascript' or link[0..5] == 'mailto'
      link = link.index('#') == 0 ? '' : link[0..link.index('#')-1] if link.include? '#'
      if link[0..3] == 'http'
        url = URI.join(URI.escape(link))                
      else
        url = URI.join(host, URI.escape(link))
      end
      return url.normalize.to_s
    end
  end

  # do the common indexing work
  def index(url)
    open(url, "User-Agent" => USER_AGENT){ |doc| 
      h = Hpricot(doc)
      title, body = h.search('title').text.strip, h.search('body')
      %w(style noscript script form img).each { |tag| body.search(tag).remove}
      array = []
      body.first.traverse_element {|element| array << element.to_s.strip.gsub(/[^a-zA-Z ]/, '') if element.text? }
      array.delete("")
      yield(array.join(" ").words, title)
    }    
  end  
end

$stdout.sync = true 
spider = Spider.new
spider.start
```

The Spider class is a full-fledged Internet crawler. It loads its seed URLs from a YAML file named seed.yml and processes each URL in that list. To play nice and comply with the Robots Exclusion Protocol, I use a slightly modified Robots library based on Kyle Maxwell's robots library, and set the name of the crawler as 'saush-spider'. As the crawler goes through each URL, it tries to index each one of them. If it successfully indexes the page, it goes through each link in the page and tries to add it to newly found pages bucket if it is allowed (checking robots.txt), or if if it not already added. At the end of the original seed list, it will take this bucket and replace it in the seed URLs list in the YAML file. This is effectively a breadth-first search that goes through each URL at every level before going down to the next level. Please note that I did some minor modifications to the robots library and the link to this is my fork out of GitHub, not Kyle's.

The flow of adding to the index goes like this. First, we need to scrub the URL by taking out URLs that are suspected to be web applications (like Google Calendar, Yahoo Mail etc) we check if the URL is in the index already, if it's not, we proceed to index it. If it already is, we'll check for the page's freshness and re-index it if it's stale. Otherwise, we'll just leave it alone.

That's it! The crawling part is rather simple but slow. To use this effectively we will need to add in capabilities to run a whole bunch of spiders at the same time but this will do for now. Let's get to the indexing part next.

## Index

Indexing involves parsing and storing data that has been collected by the crawler. The main objective here is to build a index that can be used to query and retrieve the stored data quickly. But what exactly is an index?

An index in our context is a data structure that helps us to look up a data store quickly. Take any data store that has N number of records. If we want to find a document in this data store we would need to go through each record to check if that's the one we're looking for before proceeding to the next. If it takes 0.01s to check each document and if we have 100,000 records, we'll take 0.01 second at best case, 1,000 seconds to search through the data store at worst case and 500 seconds on average. Performance is therefore time linear or we say it takes O(N) time. That's way too slow for any search engine to effectively handle millions or billions of records.

To improve the search performance, we insert an intermediate data structure between the query and the actual data store. This data structure, which is the index, allows us to return results in much improved times.

There are many different types of data structures that can be used as an index to the data store but the most popular one used by search engines is the inverted index. A normal way of mapping documents to words (called the forward index) are to have each document map to a list of words. For example:

[](images/se01.png)

An inverted index however maps words to the documents where they are found.

[](images/se02.png)

A quick look at why inverted indices is much faster. Say we want to find all documents in the data store that has the word 'fox' in it. To do this on a normal forward index, we need to go through each record to check for the word 'fox' i.e. in our example above, we need to check the forward index 4 times to find that it exists in document 2 and 3. In the case of the inverted index, we need to just go to the 'fox' record and find out that it exists in document 2 and 3. Imagine a data store of a millions of records and you can see why an inverted index is so much more effective.

In addition, a full inverted index also includes the position of the word within the document:

[](images/se03.png)

This is useful for us later during the query phase.

SaushEngine implements an inverted index in a relational database. Let's look at how this is implemented. I used MySQL as it is the most convenient but with some slight modifications you can probably use any relational database.

[](images/se04.png)

The Pages table stores the list of web pages that has been crawled, along with its title and URL. The `updated_at` field determines if the page is stale and should be refreshed. The Words table stores a list of all words found in all documents. The words in this table however are stemmed using the Porter stemmer prior to storing, in order to reduce the number of words stored. An improved strategy could be to also include alternate but non-standard synonyms for e.g. database can be referred as DB but to keep things simple, only stemming is used. By this time you should recognize that Locations table is the inverted index, which maps words and its positions to the document. To access and interact with these tables through Ruby, I used DataMapper, an excellent Ruby object-relational mapping library. However a point to note is that DataMapper doesn't allow you to create indices on the relationship foreign keys, so you'll have to do it yourself. I found that putting 3 indices -- 1 for `word_id`, 1 for `page_id` and 1 with `word_id` and `page_id` improves the query performance tremendously. Also please note that the search engine requires you to have both dm-core and dm-more so remember to install those 2 gems.

Now that we have the index, let's go back and look at parsing the information that is retrieved through the spider. The bulk of the work is in the index and add_to_index? methods in the Spider class. The index method is called from the add_to_index? method, along with the scrubbed URL and a code block that utilizes DataMapper to add the words and its locations into the index.

Using Open URI, the web page is extracted and Hpricot is used to parse the HTML. I take only the body of the document, and removed non-text portions of the document like inline scripts and styles, forms and so on. Going through each remaining element, I take only the explicitly text elements and drop the rest. Next, I tokenize the text. My strategy is pretty simple, which is to delimit the text by white spaces only. As a side note, while this works pretty ok in English text, it doesn't work as well in other languages especially Asian languages such as Chinese or Korean (where sometimes there are no white space delineation at all). The resulting tokens, which are words, are run through the Porter stemmer to produce word stems. These word stems are the ones that are added to the index.

Another side note -- if you're not a Rubyist or a beginner Rubyist you might interested to see that I actually passed an entire code block into the index method. This block differs if it's a new web page or a refreshed web page. Doing this reduces the amount of duplicate code I had to write which makes the code easier to maintain and to read.

Also if you're wondering what happened to keyword meta-tags, you're right, I didn't use them. There is no big philosophical reasoning why I skipped them -- I just wanted to focus on the document text and not give any special consideration to keywords in the meta tag.

Finally let's look at the query phase of the search engine, which all that we've talked about before comes together.

## Query

The query phase of the search engine allows a user to express his intent and get relevant results. The search query input is most often limited to a single text box. As a result of this constraint, there are a number of problems. Firstly, because the user can only express his question in a single line of text, the true intention of the user can vary widely. For example, if I type in 'apple' as the search input, do I mean the computer company or the fruit? There is no way the search engine can find out absolutely from the user. Of course, a more seasoned search user would put in more clues, for example, 'apple mac' or 'apple ipod' will return the more accurate intention.

Also, all intentions must be express only in text. For example, if I want to find a document that was written after a certain date, or a document written by at least 3 collaborators, or a photo with a particular person in it, expressing it in a text query is pretty challenging. This is the problem posed in parsing and identifying the search query, the first part of the query phase. For SaushEngine, I made broad assumptions and parsing the search query becomes only a matter of tokenizing the search input into words and stemming the words.

The second part of the query phase is to return relevant results. This is usually implementation specific and is dependent on the index that has been generated. The challenge in this part doesn't really lie here, it lies in sorting the results by relevance. Although this sounds simple, it can be a truly profound problem with an amazing variety of solutions. Choosing the correct sorting solutions (also called ranking algorithms), generating the correct sorted relevance and responding with reasonable performance is the balance a search engine designer must make.

There are a large number of ranking algorithms that can be used for a search engine. The most famous one is probably PageRank, the ranking algorithm created by the Google founders, Larry Page and Sergey Brin. PageRank assigns a score for every page depending on the number and importance of pages that links to it and uses that to rank the page. Other algorithms use different methods, from getting feedback from the search engine users to rank the search results (the more times a user clicks on a search result, the higher it is ranked) to inspecting the contents of the page itself for clues on its relevance.

For SaushEngine, I chose 3 very simple ranking algorithms that are based on the content of the page itself:

* Frequency of the search words
* Location of the search words
* Distance of the search words
* Frequency

The frequency ranking algorithm is also quite simple. The page that has more of the search words is assumed to be more relevant.

### Location

The location ranking algorithm is very simple. The assumption made here is that if the search word is near to the top of the document, the page is more relevant.

### Distance

The distance ranking algorithm inspects the distance of the search words between each other on every page. The closer the words are to each other on a page, the higher that page will be ranked. For example, if I search for 'brown fox' in these 2 documents:

1. The quick brown fox jumped over the lazy dog
2.  The brown dog chased after the fox.

The will both turn up the search results, but document 1 will be more relevant as the distance between 'brown' and 'fox' in document 1 is 0 while in document 2 it is 4.

Let's see how SaushEngine implements these 2 ranking algorithms. SaushEngine's ranking algorithms are in a class named Digger, in a file named `digger.rb`. Here's the full source code to `digger.rb`:

```ruby
require 'index'

class Digger
  SEARCH_LIMIT = 19  

  def search(for_text)
    @search_params = for_text.words
    wrds = []
    @search_params.each { |param| wrds << "stem = '#{param}'" }
    word_sql = "select * from words where #{wrds.join(" or ")}"
    @search_words = repository(:default).adapter.query(word_sql)    
    tables, joins, ids = [], [], []
    @search_words.each_with_index { |w, index|
      tables << "locations loc#{index}"
      joins << "loc#{index}.page_id = loc#{index+1}.page_id"
      ids << "loc#{index}.word_id = #{w.id}"    
    }
    joins.pop        
    @common_select = "from #{tables.join(', ')} where #{(joins + ids).join(' and ')} group by loc0.page_id"    
    rank[0..SEARCH_LIMIT]
  end

  def rank
    merge_rankings(frequency_ranking, location_ranking, distance_ranking)
  end

  def frequency_ranking
    freq_sql= "select loc0.page_id, count(loc0.page_id) as count #{@common_select} order by count desc"
    list = repository(:default).adapter.query(freq_sql)
    rank = {}
    list.size.times { |i| rank[list[i].page_id] = list[i].count.to_f/list[0].count.to_f }  
    return rank
  end  

  def location_ranking
    total = []
    @search_words.each_with_index { |w, index| total << "loc#{index}.position + 1" }
    loc_sql = "select loc0.page_id, (#{total.join(' + ')}) as total #{@common_select} order by total asc" 
    list = repository(:default).adapter.query(loc_sql) 
    rank = {}
    list.size.times { |i| rank[list[i].page_id] = list[0].total.to_f/list[i].total.to_f }
    return rank
  end

  def distance_ranking
    return {} if @search_words.size == 1
    dist, total = [], []
    @search_words.each_with_index { |w, index| total << "loc#{index}.position" }    
    total.size.times { |index| dist << "abs(#{total[index]} - #{total[index + 1]})" unless index == total.size - 1 }    
    dist_sql = "select loc0.page_id, (#{dist.join(' + ')}) as dist #{@common_select} order by dist asc"  
    list = repository(:default).adapter.query(dist_sql) 
    rank = Hash.new
    list.size.times { |i| rank[list[i].page_id] = list[0].dist.to_f/list[i].dist.to_f }
    return rank
  end

  def merge_rankings(*rankings)
    r = {}
    rankings.each { |ranking| r.merge!(ranking) { |key, oldval, newval| oldval + newval} }
    r.sort {|a,b| b[1]<=>a[1]}    
  end
end
```

The implementation is mostly done in SQL, as can be expected. The basic mechanism is to generate the SQL queries (one for each ranking algorithm) from the code and send it to MySQL using a DataMapper pass-through method. The results from the queries are then processed as ranks and are sorted accordingly by a rank merging method.

Let's look at each method in greater detail. The search method is the main one, which takes in a text string search query as input. This search query is then broken down into different word stems and we try to look for these words in our index, to get their row IDs for easier and faster manipulation. Next, we create the common part of the search query, something that is needed for each subsequent query and call the rank method.

The rank method is an aggregator that calls a number of ranking methods in sequence, and merges the returned rank lists together. A rank list is nothing more than an array of 2 item arrays, with the first item being the word ID and the second item being its rank by that algorithm:

    [ [1123, 0.452],
    [557 , 0.314],
    [3263, 0.124] ]

The above means that there are 3 words in the rank list, the first being a word with the word ID 1123 and this word is ranked with a number 0.452, the second word is 557 and so on. Merging the returned rank lists would just mean that if we have the same words in the different rank lists but given a different ranking number, we add the rank numbers together.

Let's look at the ranking algorithms. The easiest is the frequency algorithm. In this algorithm, we rank the page according to the number of times the searched words appear in that page. This is the SQL query that is normally generated from a search with 2 effective terms:

```ruby
SELECT loc0.page_id, count(loc0.page_id) as count FROM locations loc0, locations loc1
WHERE loc0.page_id = loc1.page_id AND
  loc0.word_id = 1296 AND
  loc1.word_id = 8839
GROUP BY loc0.page_id ORDER BY count DESC
```

This returns a resultset like this:

[](images/se05.png)

which obviously tells us which page has the most count of the given words. To normalize the ranking, just divide all the word counts with the largest count (in this case it is 1575). The highest ranked page has a rank count of 1, while the rest of the pages are ranked at numbers smaller than 1.

The location algorithm is about the same as the frequency algorithm. Here, we rank the page according to how close the word is to the top of the page. With multi-word searches, this becomes an exercise in adding the locations of each word. The is the SQL query from the same search:

```sql
SELECT loc0.page_id, (loc0.position + 1 + loc1.position + 1) as total FROM locations loc0, locations loc1
WHERE loc0.page_id = loc1.page_id AND
  loc0.word_id = 1296 AND
  loc1.word_id = 8839
GROUP BY loc0.page_id ORDER BY total ASC
```

This results a result set like this:

[](images/se06.png)

which again obviously tells us which page has those words closest to the top of the page. To normalize the ranking however, we can't use the same strategy as before, which is to divide each count by the largest count, as the lower the number is, the higher it should be ranked. Instead, I inversed the results (dividing the smallest total by each page total). For the smallest total this produces a rank count of 1 again, and the rest are ranked at numbers smaller than 1.

The previous 2 algorithms were relatively simple, the word distance is slightly tricky. We want to find the distance between all the search terms. For just 1 search term, this is a non-event as there is no distance. For 2 search terms, this is also pretty easy as it is only searching the distance between 2 words. It becomes tricky for 3 or more search terms. Should I find the distance between word 1 and 2 then word 1 and 3 then word 2 and 3? What if there are 4 or more terms? The growth in processing becomes factorial. I opted for a simpler approach, which is, for 3 terms, to find the distance between word 1 and 2 and then word 2 and 3. For 4 terms, it will be the distance between word 1 and word 2, then word 2 and word 3 and finally word 3 and word 4. The growth in processing then becomes a more manageable linear growth.

This is the SQL generated by this approach for 3 terms (adding 1 more additional term from above):

```ruby
SELECT loc0.page_id, (abs(loc0.position - loc1.position) + abs(loc1.position - loc2.position)) as dist
FROM locations loc0, locations loc1, locations loc2
WHERE loc0.page_id = loc1.page_id AND
  loc1.page_id = loc2.page_id AND
  loc0.word_id = 1296 AND
  loc1.word_id = 8839 AND
  loc2.word_id = 8870
GROUP BY loc0.page_id ORDER BY dist ASC
```

I use the abs function in MySQL to get the distance between 2 word locations in the same page. This returns a resultset like this:

[](images/se07.png)

Like the location algorithm, the smaller the distance, the more relevant the page is, so the same strategy in creating the page rank.

The rankings that are returned by the different ranking algorithms are added up together equally to give a final ranking of each page. While I put equal emphasis on each ranking algorithm, this doesn't necessarily need to be so. I could have easily put in parameters that would weigh the algorithms differently.

We've finished up the search engine proper. However search engines need an interface for users to interact with. We need wrap a nice web application around SaushEngine.

## User Interface

I chose Sinatra, a minimal web application framework, to build a single page web application that wraps around SaushEngine. I chose it because of its style and simplicity and its ability to let me come up with a pretty good interface in just a few lines of code.

For this part of SaushEngine, you'll need to install the Sinatra gem. We have 2 files, this is the Sinatra file, ui.rb:

```ruby
require 'rubygems'
require 'digger'
require 'sinatra'

get '/' do
  erb :search
end

post '/search' do
  digger = Digger.new
  t0 = Time.now
  @results = digger.search(params[:q])
  t1 = Time.now
  @time_taken = "#{"%6.2f" % (t1 - t0)} secs"
  erb :search
end

error MysqlError do
    'Can\'t find this in the index, try <a href=\'/\'>again</a>'
end

error do
    'Something whacked happened dude, try <a href=\'/\'>again</a>'
end

not_found do
    'Can\'t find this dude, try <a href=\'/\'>again</a>'
end

get '/info' do
  "There are #{Page.count} pages in the index."
end
```

Finally you need a view template `search.erb` to be its main user interface. I use ERB for templating because it's the most familiar to me. This is how it turns out:

[](images/se08.png)

This is obviously not an industrial strength or commercially viable search engine. I wrote this as a tool to describe how search engines are written. There are much things to be improved, especially on the crawler, which is way too slow to be effective (unless it is a focused crawler) or the index (stored in a few MySQL tables but would have problems with a large index) or the query (ranking algorithms are too simplistic).

There are obviously a lot of literature out there I have referenced, many of them are now hyperlinked from this article. However, a book that have influenced me greatly especially in designing the index in MySQL is Toby Segaran's Programming Collective Intelligence.

If you're interested in extending it please feel free to take this code but would appreciate it if you drop me a note here. 