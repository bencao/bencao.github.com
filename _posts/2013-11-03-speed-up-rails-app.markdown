---
layout: post
title: "Way to speed up rails apps"
date: 2013-11-03 21:51
comments: true
external-url:
categories:
---

Performance is always a bottleneck for rails apps.
Why?
Maybe calling a method in other active records is way too easy, so we're apt to call more, and delegate more.
But call and delegate is not free, it's db operations under the hood, which are expensive.
Besides that, it's very common that we bind many 'before' and 'after' callbacks on active record, which results in more and more CPU time during load.
To sum up, as rails developers, we're very likely to encounter performance issue, so it's better to equip ourselves with some swiss army knife.

## Find out the problem

Before start optimizing something, the first step is to find out what the problem is.

Ok, it's not a rocket science since we have ruby prof gem in hand.

### Ruby Prof

Some background is needed here - you'd better go [ruby prof website](https://github.com/ruby-prof/ruby-prof) to know more.
With the help of ruby prof, you can easily wrap some pieces of your code, then trigger one run, and you will see the performance data for this piece of code including:
- time percentage spent by each sub function call
- the call counts for each function

I recommend using the "RubyProf::GraphHtmlPrinter - Creates a call graph report in HTML (separate files per thread)" to print result to a html page.

## Cache

We can see from ruby prof result that some methods were consuming more resource that we'd thought.
One possibility is it's called too many times.

If one method is called many times and for each time it's called, it returns the same result, it's the scenario that we can cache the return of method.
Before doing cache, please remember cache should not break the system, and should have the minimal intrusiveness as possible.

### the naive way

Just save method result to a instance variable, this can be achieved by wrapping old model method, as following:

```ruby
# BEFORE
def expensive_method
  sleep 30
end

# AFTER
def expensive_method
  @expensive_method_result ||= _expensive_method
end

def _expensive_method
  sleep 30
end
```

It works and observably improved total time. But there's some disadvantages:
- when _expensive_method returns false, the optimization is totally useless
- it invade the original codes, make it harder to read, and will potentially break the specs, this can be a big problem to a large project

### a improved way

Assume we can write like this:

```ruby
# define methods need to cache in class
acts_as_method_cacheable :methods => [:expensive_method, :another_expensive_method]
```

In this way we achieved two goals:
- we can handle the case when result is false without writing if/else everywhere
- it's less intrusive, even we can rewrite 'reload' method to clear the cache, so it's easy to handle cached method in specs

### ultimate way

If a method is called for a feature only, do we need to cache it for other cases?
Yes, No. We need a way that can do cache on demand, like this:

```ruby
@post.cache_method(:expensive_method)
10.times { @post.expensive_method }
```

Got it. You can cache on a instance instead in class level.
And where should we write the codes to do cache_method? The answer can be chosen from Model, View, Controller.
The answer is Controller:
- only in controller you knowing exactly which model methods will be called
- and in controller the impact is partial, anything you're doing with cache won't break other actions, and specs

### do we have helper methods mentioned above as a gem?

Yes, we do. [Go there](https://github.com/bencao/acts_as_method_cacheable) to learn more.

## Improve algorithm

If you've tried the above ways and the performance is still not reasonable, you can consider to rewrite some key component in a higher efficient algorithm.
Take copy a active record with its association as an example.
We have a grade with many teachers, we need to copy this grade, and its teachers, having the same relationships between grade and teachers.

### traditional way

Let's first do this in a traditional way.

```ruby
copied_grade = @grade.dup.save!
@grade.teachers.each do |teacher|
  copied_teacher = teacher.dup.save!
  copied_grade.teachers << copied_teacher
end
```

Any problem to this piece of code? From logic it's perfectly true.
But let's switch to DB perspective:

```ruby
# INSERT INTO `grade` (name) VALUES ("grade")
# INSERT INTO `teacher` (name) VALUES ("teacher")
# INSERT INTO `grade_teacher_assignment` (grade_id, teacher_id) VALUES (10, 13)
# INSERT INTO `teacher` (name) VALUES ("teacher")
# INSERT INTO `grade_teacher_assignment` (grade_id, teacher_id) VALUES (10, 13)
# ...
```

Yes, it results in too many inserts. Is it possible to insert all teachers in one 'INSERT' statement?
Seems some foundamental changes are needed.

### alternative way

[Go there](https://github.com/bencao/acts_as_brand_new_copy) you can see an alternative way of doing copy.

Not only efficiency is improved, code also became cleaner.
So remember, when time comes, write you own algorithm to save the world!

## Improve GC

Gabage colletor for ruby(MRI) works in a 'Stop the world, Mark-Sweep' way.
That means, all ruby code will stop being executed and all CPU resources are used to find out gabage objects and free the memory.
To reduce GC time, we can work from two directions:
- see how we can generate less objects in app
- see how we can adjust GC make it run faster

### generate less objects in app

When you write '@grade.teachers' you're creating dozens of 'Teacher' and 'GradeTeacherAssignment' active record objects.
Say if we need teacher ids only we can write '@grade.grade_teacher_assignments.map(&:teacher_id)', this save us from constructing and GCing 'Teacher' objects.

Another one is the above copy sample. Brand new copy create only a half active record objects comparing to naive one, it uses hash instead.
So we can always try to find rooms for cutting unnecessary object constructions.

### adjust GC make it run faster

Digging into [ruby GC implementation](https://github.com/ruby/ruby/blob/trunk/gc.c) you will find that there're several tunable params to control its behaviours.
For example, I used these settings in my local machine which accelerates visiting speed about 50%
```bash
export RUBY_HEAP_MIN_SLOTS=300000
export RUBY_HEAP_SLOTS_INCREMENT=100000
export RUBY_HEAP_SLOTS_GROWTH_FACTOR=1
export RUBY_HEAP_FREE_MIN=100000
export RUBY_GC_MALLOC_LIMIT=30000000
```

Many articles could be found in internet, for example [Eating the 1.9 elephant](http://blog.newrelic.com/2012/10/23/eating-the-1-9-elephant/).

## Upgrade to latest ruby version

Ruby is always improving. Later version has many improvements that can speed up your program, such as GC redesign.
Just keep ruby version up to date you could benifit from the effort of whole community.

## Conclusion

I hope the above tips helps you improve your rails apps' performance.
Beside of technics, the more important thing is
- the need for speed
- keep an open eye on latest community progress, for example trending repos on github

Thank you for reading!
