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

# Customized domain name
I want to redirect my domain [blog.bencao.it](http://blog.bencao.it) to the newly setup site.
It's simple, just add a file under *source* directory named "CNAME" with following content

{% codeblock %}
blog.bencao.it
{% endcodeblock %}

# Choose a Theme
Take [greyshade](https://github.com/shashankmehta/greyshade) as example

{% codeblock lang:bash %}
cd octopress
git clone git@github.com:shashankmehta/greyshade.git .themes/greyshade
echo "\$greyshade: #AA3;" >> sass/custom/_colors.scss
rake "install[greyshade]"
{% endcodeblock %}

# New post
Rake is the core of doing things in octopress. To create a post, just

{% codeblock lang:bash %}
rake new_post["say hello world to new blog"]
{% endcodeblock %}

and I can start editing this post, with a YAML header showing meta data for this article.

# Writing Code Block
As a blog framework for geek, writing code block should be easy. Let's say helloworld in Ruby.

{% codeblock lang:ruby %}
class SayHello
  def say
    puts "Hello World!"
  end
end
{% endcodeblock %}

Also we can including code from file, which even support download

{% include_code tank_index.html %}

# Writing Blockquote
Did you remember the words from Steve Jobs?

{% blockquote %}
Stay hungry, stay foolish.
{% endblockquote %}

# Writing Pullquote

{% pullquote %}
What is the meaning of life? This is the final question we will face when we are walking on the road.
Someone says it's {"DREAM"}, do you agree?
{% endpullquote %}

# Include partial
{% render_partial ./_partials/2013-03-27-i-am-a-partial.markdown %}

# Include multi media
Embed a image is simple

{% img http://placekitten.com/900/200 %}

Pull it left/right if you wanted.
{% img left http://placekitten.com/120/80 %}

No matter where we wanted to go, here are some guides.

- Read books
- Thinking in Mind
- Practice, practice, practice

# Preview posts
To preview posts @localhost, there's also a rake task for help:
{% codeblock lang:bash %}
rake preview
{% endcodeblock %}
