---
layout: slides
title: "A coffeescript rewritten of classical FC tank game"
date: 2013-05-07 17:48
sidebar: false
deck_theme: web-2.0
deck_transition: fade
deck_navigation: true
deck_status: true
deck_goto: true
deck_hash: true
deck_menu: true
deck_scale: false
---
{% slide first %}
# Coffee Tank
<p>May 8th, 2013 @FW</p>
{% endslide %}

{% slide %}
# Play [it](http://localhost:8000)!

{% endslide %}

{% slide %}
## Game Basics

- The law of frame
- Path finding
- Enemy AI
- User input handling
- Scenes
- Stage design and Tiled map
- Tunning numbers
- Tunning performance

{% endslide %}

{% slide %}
## The law of frame

- FPS = frames per second
- The higher the fps is, the smoother users feel
- LCD screen refresh rate is 60Hz, so it's good enough to set our goal to 60 fps
- less than 17ms per frame, but need to do calculation and redraw, challenge!

{% endslide %}

{% slide %}
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
{% endslide %}

{% slide %}
## Path finding

- single source shortest path problem
- Dijikstra and A*
- [PathFinding.js](http://qiao.github.io/PathFinding.js/visual/) for more
{% endslide %}

{% slide %}
## Enemy AI

- Where to move
- When to fire
- Adjustable AI
{% endslide %}

{% slide %}
{% codeblock Enemy AI lang:coffeescript %}
next: ->
  # move towards home
  if _.size(@path) == 0
    end_vertex = if (Math.random() * 100) <= @map_unit.iq
      @map.home_vertex
    else
      @map.random_vertex()
    @path = @map.shortest_path(@map_unit, @current_vertex(), end_vertex)
    @next_move()
    setTimeout((() => @reset_path()), 2000 + Math.random()*2000)
  else
    @next_move() if @current_vertex().equals(@target_vertex)

  # more chance to fire if can't move
  if @map_unit.can_fire() and @last_area and @last_area.equals(@map_unit.area)
    @fire() if Math.random() < 0.08
  else
    @fire() if Math.random() < 0.01

  @last_area = @map_unit.area

{% endcodeblock %}
{% endslide %}

{% slide %}
## User input handling

- the challenge is to transfer simulated input signals(by user) to digital input signals
- remove noise such as press left and right together
- tank user input is keyboard, which has 3 types of events: keyup, keydown, keypress
{% endslide %}

{% slide %}
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
{% endslide %}

{% slide %}
## Scenes

- each scene should be designed as individual as possible
- also need to provide a mechanism for scenes to share data

Tank is seperated into

- welcome scene
- stage scene
- game scene
- report scene
- hi_score scene

{% endslide %}

{% slide %}
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
{% endslide %}

{% slide %}
## Stage design and Tiled map

- It's possible to design stage by a few lines of code, ok for test but not good for human.
- GUI tool helps. I found Tiled map.
- Tiled map is an ecosystem which contains a 2D sprite based map format standard and a set of cross platform GUI tools and cross language map readers and writers.
- Its json definition is so easy readable, tank read the json output directly.
{% endslide %}

{% slide %}
{% img /images/posts/tiled_screen_capture.png 949 665 %}
{% endslide %}

{% slide %}
## Tunning numbers

- Most numbers were borrowed from origin FC tank game
- story about enemy tank IQ
{% endslide %}

{% slide %}
## Tunning performance

- Drawing Library is the most important factor to final performance
- Improve path finding speed using binomial heap, 700ms to less than 50ms.
{% endslide %}

{% slide %}
## What's more

- Why I love Coffeescript more than Javascript
- Unit Test and Continuous Integration

{% endslide %}

{% slide %}
## Why I love Coffeescript more than Javascript

- human friendly
- easy and intuitive class inheritance
- simpler array/hash iteration
- @ instead of this
- shorthand for function definition
- popular now
{% endslide %}

{% slide %}
## Unit Test and Continuous Integration

You really should try the excellent CI tool [Travis-CI](https://travis-ci.org/). It support running unit tests in a variety of environments and is already seamlessly integrated with Github.

The unit test tool I used is [QUnit](http://qunitjs.com/).
{% endslide %}

{% slide %}
## References

- [Game Physics](http://gafferongames.com/game-physics/)
- [Tiled Map](http://www.mapeditor.org/)
- [KineticJS](http://kineticjs.com/)
- [Coffeescript](http://coffeescript.org/)
- [LoDash](http://lodash.com/)
- [QUnit](http://qunitjs.com/)
- [PathFindingJs](https://github.com/qiao/PathFinding.js)
- [How to be a game programer](http://www.gamengines.com/article-549.html)
{% endslide %}

{% slide %}
## Ideas and Questions?
{% endslide %}
