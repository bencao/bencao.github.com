---
layout: post
title: "Say hello world to new blog"
date: 2013-03-27 10:19
comments: true
categories:
---

# Setup
After clone octopress to local, it's easy to deploly to github.

{% codeblock Setup octopress lang:bash %}
rake setup_github_pages
rake install
git commit
rake generate
rake deploy
{% endcodeblock %}

And it's done.

## Customized domain name
I want to redirect my domain [blog.bencao.it](http://blog.bencao.it) to the newly setup site.
It's simple, just add a file under *source* directory named "CNAME" with following content

{% codeblock %}
blog.bencao.it
{% endcodeblock %}

## New post
Rake is the core of doing things in octopress. To create a post, just

{% codeblock lang:bash %}
rake new_post["say hello world to new blog"]
{% endcodeblock %}

and I can start editing this post, with a YAML header showing meta data for this article.

## Writing Code Block
As a blog framework for geek, writing code block should be easy. Let's say helloworld in Ruby.

{% codeblock lang:ruby %}
class SayHello
  def say
    puts "Hello World!"
  end
end
{% endcodeblock %}

## Writing Blockquote
Did you remember the words from Steve Jobs?

{% blockquote %}
Stay hungry, stay foolish.
{% endblockquote %}

## Choose a Theme
Take [greyshade](https://github.com/shashankmehta/greyshade) as example

{% codeblock lang:bash %}
cd octopress
git clone git@github.com:shashankmehta/greyshade.git .themes/greyshade
echo "\$greyshade: #AA3;" >> sass/custom/_colors.scss
rake "install[greyshade]"
{% endcodeblock %}

