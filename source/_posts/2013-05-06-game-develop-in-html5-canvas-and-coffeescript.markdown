---
layout: post
title: "Game develop in HTML5 canvas and coffeescript"
date: 2013-05-06 18:54
comments: true
categories:
---

In the past a few months, I tried to rebuild the classical FC tank game in coffeescript.

The final work is [here](http://blog.bencao.it/fc_tank).

{% img /images/posts/tank_welcome_scene.png %}
{% img /images/posts/tank_game_scene.png %}

It's an interesting journey, and following aspects are what I feel worth mentioning.

# Game Basics

## The law of frame
The most foundamental thing before developing a game is to understand FPS - frame per second, and how the game world are running in a frame by frame law.

Before each frame was drawn, game program need to calculate the position, direction, speed, and other attributes for each moving objects(display object) in screen.
The higher the fps is, the smoother users feel. Typical refresh rate of LCD screen is 60Hz, so it's good enough to set 60 fps as our goal.
This set a great challenge for game developers because some parts may sometimes take a lot of CPU time.
Great game need a great performance optimization.

Here's the core loop of Tank time line:
{% codeblock Tank Core Loop lang:coffeescript %}
start_time_line: () ->
  last_time = new Date()
  @timeline = setInterval(() =>
    current_time = new Date()
    # assume a frame will never last more than 1 second
    delta_time = current_time.getMilliseconds() - last_time.getMilliseconds()
    delta_time += 1000 if delta_time < 0
    _.each(@map.missiles, (unit) -> unit.integration(delta_time))
    _.each(@map.gifts.concat(@map.tanks), (unit) -> unit.integration(delta_time))
    _.each(@map.missiles, (unit) -> unit.integration(delta_time))
    last_time = current_time
    @frame_rate += 1
  , parseInt(1000/@fps))
  # show frame rate
  @frame_timeline = setInterval(() =>
    @frame_rate_label.setText(@frame_rate + " fps")
    @frame_rate = 0
  , 1000)
{% endcodeblock %}

## Path finding
In many kinds of games, enemies have some degree of AI. They need to move across certain paths or towards some targets.
Terrains might be different, also they may be changing all the time, we need to do frequent calculation in order to find out the best path.

There are many path finding algorithm, the most common ones are: Dijikstra and A star, while A star is sightly improve Dijikstra by adding a weight base on the "distance" of current position to target position.
Here's a wonderful site demonstrate the Path finding algorithm very well: [PathFinding.js](http://qiao.github.io/PathFinding.js/visual/).

## Enemy AI

AI differs in different kind of games.

For tank, here is the list I considered to be important:

- Where to move
- When to fire
- Adjustable AI


## Scenes

A game can be divided into several scenes, these scenes should be easy switchable and be properly designed to have its own preparation and cleaning functions.
While each scene should be designed as individual as possible, a global storage must be there which can be used by scenes to share data.

For Example, tank game is divided into

- welcome scene
- stage scene
- game scene
- report scene
- hi_score scene

Each scene has a start/stop method which doing prepare/clearing stuffs. The global storage is the instance of game. Take stage scene as example:
{% codeblock Stage start and stop lang:coffeescript %}
class StageScene extends Scene
  constructor: (@game) ->
    super(@game)
    @init_stage_nodes()

  start: () ->
    super()
    @current_stage = @game.get_config('current_stage')
    @update_stage_label()
    if @game.get_config('stage_autostart')
      setTimeout((() => @game.switch_scene('game')), 2000)
    else
      @enable_stage_control()

  stop: () ->
    super()
    @disable_stage_control()
    @prepare_for_game_scene()
{% endcodeblock %}

## Stage design and Tiled map

It's possible to design stage by a few lines of code. But normal users can not easily do that.
I started to think whether a GUI tool exists. And I found Tiled map.

Tiled map is an ecosystem which contains a 2D sprite based map format standard and a set of cross platform GUI tools and cross language map readers and writers.
Its json definition is so easy readable that I directly read the data from the json export from a tiled output.

Use tiled made adding new stage easy and possible for non-programers to get involved.

{% img /images/posts/tiled_screen_capture.png %}

## User input handling

The challenge is to transfer simulated user input signals to digital signals.

A variety of user input devices are available now. The most frequently used devices today are keyboard, mouse and touch screen.
I choosed keyboard. There are 3 types of keyboard events, keyup, keydown, keypress.

Here's the code logic I used to handle keyboard input:
{% codeblock Input handling lang:coffeescript %}
class UserCommander extends Commander
  constructor: (@map_unit, key_setting) ->
    super(@map_unit)
    @key_map = {}
    for key, code of key_setting
      @key_map[code] = key
    @key_status = {
      up: false,
      down: false,
      left: false,
      right: false,
      fire: false
    }
    @reset_input()

  reset_input: () -> @inputs = { up: [], down: [], left: [], right: [], fire: [] }
  is_pressed: (key) -> @key_status[key]
  set_pressed: (key, bool) -> @key_status[key] = bool

  next: ->
    @handle_key_up_key_down()
    @handle_key_press()

  handle_key_up_key_down: () ->
    for key, types of @inputs
      continue if _.isEmpty(types)
      switch (key)
        when "fire"
          @fire()
        when "up", "down", "left", "right"
          if @direction_changed(key)
            @turn(key)
            break
          keyup = _.contains(@inputs[key], "keyup")
          keydown = _.contains(@inputs[key], "keydown")
          if keydown
            @start_move()
          else
            @stop_move() if keyup
    @reset_input()

  handle_key_press: () ->
    for key in ["up", "down", "left", "right"]
      if @is_pressed(key)
        @turn(key)
        @start_move()
    if @is_pressed("fire")
      @fire()

  add_key_event: (type, key_code) ->
    return true if _.isUndefined(@key_map[key_code])
    key = @key_map[key_code]
    switch type
      when "keyup"
        @set_pressed(key, false)
        @inputs[key].push("keyup")
      when "keydown"
        @set_pressed(key, true)
        @inputs[key].push("keydown")
{% endcodeblock %}

## Tunning Numbers

Most number settings that are obvious were borrowed from origin FC tank game, such as tank life, missile power, terrain properties and so on.

Others such as enemy tank IQ are defined according to feelings. At first there's no enemy IQ concept and I found enemies always rush to me in shortest paths, what a nightmare!
So I have to import an IQ concept which is actually the hundred percent rate that they choose the right target - user's home, otherwise they rush to a random area in map.
Enemies are becoming smarter and smarter along with user entering into later stages.

## Tunning Performance

### Drawing Library is the most important factor to final performance

I've tried [Rapheal](http://raphaeljs.com/), [oCanvas](http://ocanvas.org/) and [Kinetic](http://kineticjs.com/).
And Kinetic is my final choice because it's base on HTML5 canvas, its API is programer friendly and its performance is much better than oCanvas.

### Improve path finding using advance data structure

I can't find a existing implementation of Binomial Heap in Javascript, so I wrote one.
It reduces each path finding time from around 700ms to less than 50ms.

# Why I love Coffeescript more than Javascript

- easy and intuitive class inheritance
- simpler array/hash iteration
- @ instead of this
- shorthand for function definition

# Unit Test and Continuous Integration

You really should try the excellent CI tool [Travis-CI](https://travis-ci.org/). It support running unit tests in a variety of environments and is already seamlessly integrated with Github.

The unit test tool I used is QUnit.

# Possible improvements

## Larger and custom map
## Multi-user online game
