---
layout: post
title:  "Making a Ruby Gem"
date:   2016-01-19
categories: jekyll theme
---

In this post, I will outline the basic set-up and process to create a gem that displays the air quality in a chosen zipcode of the US. In creating this gem, we need to consider questions like: 

> *Where will we get the data? How will we store this data to show the user? What will the interface for the user look like? What if a zipcode is not recognized by the database? What if information is unavailable?*

Let's begin.

**Initialization**
---

As is in the proverbial example of <a href="http://static.zerorobotics.mit.edu/docs/team-activities/ProgrammingPeanutButterAndJelly.pdf" target="_blank">'how to make a peanut butter and jelly sandwich'</a>, let me not pass over some assumed steps that have been taken.

We will be using the <a href="http://www.nokogiri.org" target="_blank">Nokogiri</a> and <a href="http://pryrepl.org" target="_blank">Pry</a> gems and <a href="http://ruby-doc.org/stdlib-2.1.0/libdoc/open-uri/rdoc/OpenURI.html" target="_blank">Open-Uri</a> to scrape the information from the website that holds the data. More on this later.

The basic file structure for the gem was initialized by running `bundle gem [new_gem_name]`, a very convenient tool available through the magic of Bundler. The structure of our gem will be as follows:

    .
    ├── Gemfile
    ├── README.md
    ├── Rakefile
    ├── bin
    │   ├── breathe_in
    │   ├── console
    │   └── setup
    ├── breathe_in.gemspec
    ├── config
    │   └── environment.rb
    ├── lib
    │   ├── breathe_in
    │   │   ├── city.rb
    │   │   ├── cli.rb
    │   │   ├── scraper.rb
    │   │   └── version.rb
    │   └── breathe_in.rb
    └── spec
        ├── breathe_in_spec.rb
        └── spec_helper.rb
    
**Scraping Data**
---

The first step is to obtain data. The gem must display: 1.) the expected air quality conditions for today as well as 2.) the current conditions, if available. To gather the data, we will utilize the resources available at <a href="http://airnow.gov" target="_blank">AirNow.gov</a>, which is compiled by "the U.S. Environmental Protection Agency, National Oceanic and Atmospheric Administration, National Park Service, tribal, state, and local agencies" - in other words, we can trust the data.

To begin, we can define a simple scraping method inside of our `scraper.rb` file. The method can be as simple as: 

{% highlight ruby linenos%}
def get_page
      doc = Nokogiri::HTML(open("http://airnow.gov/?action=airnow.local_city&zipcode=90101&submit=Go"))
      binding.pry
    end
{% endhighlight %}

Notice that we have already input a zipcode and are opening the HTML with `Open-Uri`'s `#open` method and getting ready to scrape it with `Nokogiri`. We will use `Pry` to experiment with obtaining the correct tags to extract the necessary data. Awesome how powerful that combination is, right??

Now that we have our environment set up, we need to determine what to scrape from the website:

    - city name
    - today's AQI high (numerical value)
    - today's high index (Good, Moderate, etc.)
    - current conditions (last updated)
    - current AQI conditions (numerical value)
    - current conditions index (Good, Moderate, etc.)

After a lot of trial and error that isn't shown here (like how the chef pulls out a perfectly roasted chicken minutes after preparing it on the cooking channel), we skip ahead to the text values that have been obtained and stripped of unnecessary characters and converted to integers (if applicable) and finally, we end up with six methods that ultimately look like this: 

{% highlight ruby linenos%}
  def self.city_name
    city = scraped_pg.css("#pageContent .ActiveCity")
    #returns array of an object with attributes including city name
    city.empty? ? nil : air_quality[:city_name] = city.text.strip
  end
{% endhighlight %}

What's all the extra logic? Well, we know that sometimes, data can be unavailable for the particular zipcode or the city name doesn't exist (due to a wrongly inputted zipcode). We will account for that by using the `#empty?` method and calling it on each Nokogiri result to see if it has returned an empty array instead of meaningful data.

Ultimately, we want to save this extracted data as a hash of attributes that can be assigned to a city, so we will create a class variable called `.air_quality_info` (saved as a class method called `.air_quality`). Within this hash, we will assign the attributes with the correctly extracted values for that particular zipcode.

To simplify this, we create a method `.city_air_quality` that will run all the other extraction methods and returns `.air_quality_info`, the hash of attributes for the particular zipcode, saved as a class method `.air_quality`.

{% highlight ruby linenos%}
  def self.city_air_quality
    city_name
    today_high
    index_level
    current_conditions_time
    current_conditions_value
    current_conditions_index 
    air_quality
  end
{% endhighlight %}

Also, we want to add meaning to the 'index' words (Good, Moderate, etc.) that we pull from the website, so we will create a few methods that `prints` the relevant health information. We might as well include a method that outlines detailed information about the numerical range for the AQI value. 

{% highlight ruby linenos%}
  def self.index_good
    print "Air quality is considered satisfactory, and air pollution poses little or no risk."
  end

  etc..

  def self.AQI_range_information
    information = <<-Ruby
      The Air Quality Index (AQI) translates air quality data into an easily understandable number to identify how clean or polluted the outdoor air...
      Ruby 
    ...
  end
{% endhighlight %}

One more thing: we will refactor the `#get_page` method to take in an argument (since we will be inputting the zipcode that the user types) and pass it into the website address through string interpolation. 

{% highlight ruby linenos%}
  def self.scraped_page(zipcode)
    begin
      @@scraped = Nokogiri::HTML(open("http://airnow.gov/?action=airnow.local_city&zipcode=#{zipcode}&submit=Go"))
    ...
  end 
{% endhighlight %}

We will further refactor this and assign this `Nokogiri` request as a class variable (`.scraped`) to make our scraping methods less cluttered.

*Special Consideration*

You may notice a method at the bottom of the file called `.under_maintenance`. 

{% highlight ruby linenos%}
  def self.under_maintenance #returns true if under maintenance 
    scraped_pg.css("#pageContent .TblInvisibleFixed tr p[style*='color:#F00;']").text.include?("maintenance")
  end
{% endhighlight %}

The AirNow.gov website undergoes maintenance every day from 12am-4am EST and data can be sporadically available. I will talk more about this later when we get to the CLI, but know that it exists.

**Making a City**
---

Great, we have now established a class that scrapes the data. Next, we need to create a `City` class that will hold the scraped data. 

{% highlight ruby linenos%}
  def initialize(city_hash={})
    city_hash.each { |key, value| self.send(("#{key}="), value) }
    @@cities << self
  end
{% endhighlight %}

Every new city will be initialized with a default empty hash, but if a hash is passed in, the city object will be assigned with the respective hash keys and values using the `#send` method. Also, every new instance of a city will be pushed into a class variable called `.cities` that will hold all created cities.

{% highlight ruby linenos%}
  def add_city_air_quality(air_quality_hash)
    air_quality_hash.each { |key, value| self.send(("#{key}="), value) }
  end
{% endhighlight %}

The other important method to note in this file is the `#add_city_air_quality` method that takes in a hash. For our use, this method takes in the hash of attributes that we created in the `Scraper` class (`.air_quality_info`) and assigns these attributes to the city instance. So after this method is run, each city will have a `:city_name`, `:today_high`, `:today_index`, `:last_update_time`, `:last_update_value`, and `:last_update_index` (if the values are not `nil`). It will also have a `:zipcode` attribute that gets passed in through our CLI, but more on that later.

**Putting It All Together**
---

We have scraped the necessary data and now have the ability to assign a city with this data. How do we do this? With a CLI class, of course! Open up `cli.rb` to see the magic. 

Let's begin with a broad overview to understand the objective of this class. What we want to do:

<ol>
  <li>Greet the user and ask for a zipcode </li>
  <li>Scrape the website with that zipcode</li>
  <li>Assign the scraped attributes to a city</li>
  <li>Display the information</li>
  <li>Ask the user if they want to search another zipcode, get AQI information, or exit</li>
</ol>

Let's break it down by evaluating the control flow of the class.

{% highlight ruby linenos%}
  def run
    puts "*Data provided courtesy of AirNow.gov*"
    puts "How safe it is to breathe today?"
    puts ""
    get_information
    check_site_availability
    menu
  end
{% endhighlight %}

First, the CLI will be started with the `#run` method, which initially greets the user with a couple of `puts` statements. 

Then the `#get_information` method will be invoked.

{% highlight ruby linenos%}
  def get_information
    get_zipcode
    scrape_data
    if BreatheIn::Scraper.city_name == nil
      puts "That zipcode is not recognized by Air.gov."
      get_information
    else
      new_city = BreatheIn::City.new({zipcode: self.class.zipcode})
      assign_attributes(new_city)
      display_information
    end
  end
{% endhighlight %}

In turn, this method invokes the `#get_zipcode` method, which will save the user inputted zipcode in a class variable `.zipcode` that will be referenced in several other methods. 

{% highlight ruby linenos%}
  def get_zipcode
    input = ""
    until input.match(/\b\d{5}\b/)
      puts "Please enter a valid zipcode and wait a few seconds:"
      puts ""
      input = gets.strip
    end
    @@zipcode = input.to_s.rjust(5, '0')
  end
{% endhighlight %}

After getting a valid zipcode, `#get_information` will invoke `#scrape_data`, which will actually call on the `#scraped_page` method in the Scraper class. `.zipcode` will be passed into this method and through string interpolation, AirNow.gov will load the relevant data for that zipcode.

{% highlight ruby linenos%}
  def scrape_data
    BreatheIn::Scraper.scraped_page(self.class.zipcode)
  end
{% endhighlight %}

Next in the control flow is a conditional statement that checks if the `#city_name` method is `nil`. 

{% highlight ruby linenos%}
  def get_information
    ...
    if BreatheIn::Scraper.city_name == nil
      puts "That zipcode is not recognized by Air.gov."
      get_information
    ...
{% endhighlight %}

In essence, this weeds out zipcodes that are valid zipcodes, but are not recognized by AirNow.gov.

{% highlight ruby linenos%}
  def get_information
    ...
    else
      new_city = BreatheIn::City.new({zipcode: self.class.zipcode})
      assign_attributes(new_city)
      display_information
    end
  end
{% endhighlight %}

Else, if there is indeed data available for the zipcode, a new city instance will be initialized with a hash of the zipcode value.

This newly created city will now be assigned the attributes that we scraped in the `#scrape_data` method, by calling on the `#assign_attributes` method.

{% highlight ruby linenos%}
  def assign_attributes(new_city)
    attributes = BreatheIn::Scraper.city_air_quality
    ...
  end
{% endhighlight %}

`#assign_attributes` takes in an argument of the new city object that was just created - more on this in a second. But first, this method will call on `#city_air_quality` in the Scraper class - which scrapes the relevant data from the website and creates a hash of attributes from it.

{% highlight ruby linenos%}
  def assign_attributes(new_city)
    ...
    attributes[:today_high] = "Data currently unavailable." if !attributes.has_key?(:today_high)
    attributes[:today_index] = "Data currently unavailable." if !attributes.has_key?(:today_index)
    attributes[:last_update_value] = "Data currently unavailable." if !attributes.has_key?(:last_update_value)
    attributes[:last_update_time] = "Data currently unavailable." if !attributes.has_key?(:last_update_time)
    attributes[:last_update_index] = "Data currently unavailable." if !attributes.has_key?(:last_update_index)      
    ...
  end
{% endhighlight %}

Data from AirNow.gov is intermittently unavailable for certain zipcodes, certain time periods, and sometimes, random system glitches. These conditional statements check to see if the data for each scraped value of the city object is `nil` and if it is, assigns that key a string statement. It must check every possible hash key because not every value is `nil` at the same time. Without this logic, a `nil` data value will be populated with the previous search's results.

{% highlight ruby linenos%}
  def assign_attributes(new_city)
    ...
    city_info_hash = new_city.add_city_air_quality(attributes)
  end
{% endhighlight %}

 Finally, the `#add_city_air_quality` method from the City class will be invoked on the city object, new_city, passing in the scraped data hash. As a result, the city will now be associated with a hash of attributes.

{% highlight ruby linenos%}
  def display_information
    BreatheIn::City.cities.each do |city|
      puts "---------------------"
      puts "City/Area: #{city.city_name}, Zipcode: #{city.zipcode}"
      puts "---------------------"
      puts "Today's High AQI: #{city.today_high}"
      puts "Today's Index: #{city.today_index}"
      health_description(city.today_high) if city.today_high.is_a?(Integer)
      puts "---------------------"
      puts "Last #{city.last_update_time}"
      puts "Current AQI: #{city.last_update_value}"
      puts "Current Index: #{city.last_update_index}"
      health_description(city.last_update_value) if city.last_update_value.is_a?(Integer) 
       puts "---------------------"      
    end
  end
{% endhighlight %}

Finally, the information will be displayed through the `#display_information` method. This method iterates through the `.cities` class variable and `puts` out the information. 

This method also calls on `#health_description`, which will evaluate the value of `:today_index` and `:last_update_index` and return the relevant health message.

{% highlight ruby linenos%}
  def health_description(level)
      if level.between?(0,50)
        puts "#{BreatheIn::Scraper.index_good}"
      elsif level.between?(51,100)
        puts "#{BreatheIn::Scraper.index_moderate}"
      elsif level.between?(100,150)
        puts "#{BreatheIn::Scraper.index_sensitive}"
      elsif level.between?(151,200)
        puts "#{BreatheIn::Scraper.index_unhealthy}"
      elsif level.between?(201,300)
        puts "#{BreatheIn::Scraper.index_very_unhealthy}"
      elsif level.between?(301,500)
        puts "#{BreatheIn::Scraper.index_hazardous}"   
      end
  end
{% endhighlight %}

Going back to the control flow, the gem has executed `#get_information` and returns to `#run`. The next method `#check_site_availability` is invoked. This method calls on `#under_maintenance` in the Scraper class to check if there is a maintenance message present on the page. If so, it will print a disclaimer notice to the user.

{% highlight ruby linenos%}
  def check_site_availability
    if BreatheIn::Scraper.under_maintenance
      disclaimer = <<-Ruby
        ***AirNow.gov undergoes maintenance from midnight to 4am EST. 
        If information is currently unavailable, please try again later.***
        Ruby
      puts disclaimer
    end
  end
{% endhighlight %}


The last method to be called is `#menu`, which will display a pretty self-explanatory list of the user's next choices after one search. 

{% highlight ruby linenos%}
  def menu
    input = nil

    while input != 3
      puts ""
      puts "1. Learn more about the AQI values and the ranges."
      puts "2. Choose another zipcode."
      puts "3. Exit."
      puts "Please make a selection:"
      puts ""
      
      input = gets.strip.to_i
      case input
        when 1
          BreatheIn::Scraper.AQI_range_information
        when 2
          BreatheIn::City.reset
          get_information
          check_site_availability
        when 3
          puts "Breathe safely!"
        else
          puts "Please choose 1, 2, or 3."
      end
    end
  end
{% endhighlight %}

One note: when the user decides to perform another search, the `.cities` array class variable will be cleared, as otherwise, the previous results will be listed as well.

There you have it! Use this gem every morning to see how toxic that outside air really is. Maybe you'll find out that on some days, it isn't better for your health to get off that computer and run around outside for a bit.

View the <a href="https://github.com/eric-an/breathe-in" target="_blank">GitHub repository for this gem.</a>