---
layout: post
title: "First time to contribute a Gem"
date: 2013-04-10 11:49
comments: true
categories:
---

Recent days I just wrote an active record extension to make cache optimization easier for company.
I realized it's a useful tool and publish it as a gem might be a good idea.
So I start writing the gem 'acts_as_method_cacheable' yesterday afternoon, and I found it's an interesting journey.
The home page is [here](https://github.com/bencao/acts_as_method_cacheable).

Here are the steps I did it:

## create a Gem with bundle

bundle gem xxx will generate a skeleton of a gem, the gem has a hierachy like this

{% img /images/posts/gem_hierachy.png %}

## write info in gemspec file

A typical gemspec file looks like this, which is quite straight forward.
{% codeblock "acts_as_method_cacheable.gemspec" lang:ruby %}
Gem::Specification.new do |spec|
  spec.name          = "acts_as_method_cacheable"
  spec.version       = ActsAsMethodCacheable::VERSION
  spec.authors       = ["Ben Cao"]
  spec.email         = ["benb88@gmail.com"]
  spec.description   = "Make cache methods on ActiveRecord easy!"
  spec.summary       = "Instead of writing def expensive { @cached_expensive ||= original_expensive }, now you can write instance.cache_method(:expensive) instead. Also support nested >
  spec.homepage      = "https://github.com/bencao/acts_as_method_cacheable"
  spec.license       = "MIT"

  spec.files         = `git ls-files`.split($/)
  spec.executables   = spec.files.grep(%r{^bin/}) { |f| File.basename(f) }
  spec.test_files    = spec.files.grep(%r{^(test|spec|features)/})
  spec.require_paths = ["lib"]

  spec.add_development_dependency "bundler", "~> 1.3"
  spec.add_development_dependency "rake", "~> 10.0.4"
  spec.add_development_dependency "rspec", "~> 2.13.0"
  spec.add_development_dependency "mocha", "~> 0.13.3"
  spec.add_development_dependency "sqlite3", "~> 1.3.7"
  spec.add_development_dependency "pry"
  spec.add_development_dependency "pry-theme"
  spec.add_development_dependency "pry-nav"
  spec.add_dependency "activesupport", "~> 3.2.13"
  spec.add_dependency "activerecord", "~> 3.2.13"
end
{% endcodeblock %}

Note the dependencies section. development_dependencies will only be installed when you want to develop and runs
{% codeblock lang:bash %}
bundle install
{% endcodeblock %}
in gem source directory.

## add test

In order to run spec, I introduced a few development_dependencies, including rspec/sqlite3/pry.
Then I created "db" and "spec" folder, where hosts database/spec related files.

Before that, I need a sqlite db and setting up stand alone active record without rails.
After some digest, I wrote this spec_helper, which setup up active record to a empty db.

{% codeblock "spec_helper.rb" lang:ruby %}
require 'active_support'
require 'active_record'
require 'sqlite3'
require 'pry'

db_config = YAML::load(IO.read('db/database.yml'))
db_file = db_config['development']['database']
File.delete(db_file) if File.exists?(db_file)
ActiveRecord::Base.configurations = db_config
ActiveRecord::Base.establish_connection('development')
{% endcodeblock %}

For a simple gem like this, I just defined two active record class "Post" and "Comment" inside my spec file, and use them inside a describe block.
To make these model working, we need a ActiveRecord::Migration do create these tables.
The spec might be run several times, also migration will be run several times.
So you understand why I have to delete the sqlite db file in spec_helper.

Here is how the spec looks like:
{% codeblock "acts_as_method_cacheable_spec.rb" lang:ruby %}
require 'spec_helper.rb'

class Schema < ActiveRecord::Migration
  def change
    create_table :posts do |t|
      t.string :title
      t.string :date
    end

    create_table :comments do |t|
      t.string :content
      t.string :author
      t.string :date
      t.references :post
    end
  end

end
Schema.new.change

require 'acts_as_method_cacheable'

class Post < ActiveRecord::Base
  has_many :comments

  def comment_authors
    comments.map(&:author).join(" ")
  end

  def comment_contents
    comments.map(&:content).join(" ")
  end

  def comment_dates
    comments.map(&:date).join(" ")
  end

  def comment_signatures
    comments.map(&:signature).join(" ")
  end

  acts_as_method_cacheable :methods => [:comment_authors, :comment_contents]
end

class Comment < ActiveRecord::Base
  belongs_to :post

  def signature
    sub_signature
  end

  def sub_signature
    "cool!"
  end

  acts_as_method_cacheable
end

describe ActsAsMethodCacheable do

  before(:each) do
    @post = Post.create!(:title => 'test', :date => '2013-04-04')
    @post.comments.create!(:content => 'ct1', :author => 'ben', :date => '2013-04-05')
    @post.comments.create!(:content => 'ct2', :author => 'feng', :date => '2013-04-06')
    @comment1, @comment2 = @post.comments.to_a
  end

  context "class" do
    it "should return the same result as without cache" do
      @post.comment_authors.should == @comment1.author + " " + @comment2.author
      @post.comment_authors.should == @comment1.author + " " + @comment2.author
    end

    # ...
  end
end
{% endcodeblock %}

## build and publish

build gem is easy.

{% codeblock lang:bash %}
gem build acts_as_method_cacheable.gemspec
{% endcodeblock %}

publish gem is easy once you've registered a rubygem.org account and downloaded credentials to localhost.

{% codeblock "download credentials" lang:bash %}
curl -u USERNAME https://rubygems.org/api/v1/api_key.yaml > ~/.gem/credentials
{% endcodeblock %}

{% codeblock "publish gem" lang:bash %}
gem push acts_as_method_cacheable-0.1.0.gem
{% endcodeblock %}

And it's all done.
