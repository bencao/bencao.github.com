---
layout: post
title: "Some thoughts about coding"
date: 2014-05-29 15:33
comments: true
external-url:
categories:
---

While browsing code these days, reading some wonderful pieces and some awful lines, some thoughts came into my mind.

Let’s start from some code smells first.

## Smell 1 - Hide complexity/detail in a bad way

### Case 1:

```ruby
check_point!(:beauty)
```

What does it mean here? I have no choice but follow to the actual check_beauty method to see what happens.
Why not

```ruby
should_have!(:perfect_shape)
should_not_have!(:big_mouth)
should_be!(:tender)
should_not_be!(:angry)
```

It’s low efficient to switch between different stack levels when reading code, human brain isn’t designed for that

### Case 2:

```ruby
if placement_status_active?
  register_nielsen_placements
end
```

All active placement will register nielsen placement? No matter it is rating base buying or not? Sounds buggy. But actually here is register_niesen_placements

```ruby
def register_nielsen_placements
  return if template?
  return unless rating_based_buying_enabled?
  nielsen_campaign = NetworkNielsenCampaign.where(:mrm_campaign_id => self.campaign.id).first
  return if nielsen_campaign.blank?
  # ignore more
end
```

Why not

```ruby
if rating_base_buying_enabled? && placement_status_active?
  register_nielsen_placements
end
```

Don’t frighten your reader.


## Smell 2 - Expose complexity/detail in a bad way

### Case 1:

```ruby
@ad_unit_nodes.each do | ad_unit_node |
  if ad_unit_node.is_display? && ad_unit_node.ad_unit.width && ad_unit_node.ad_unit.height
    @creatives.each do | creative |
      if creative.param_only? && creative.alive_creative_renditions.size > 0
        ad_unit_dimension = ad_unit_node.ad_unit.width.to_s + "*" + ad_unit_node.ad_unit.height.to_s
        creative_dimension = creative.alive_creative_renditions[0].width.to_s + "*" + creative.alive_creative_renditions[0].height.to_s
        if ad_unit_dimension != creative_dimension
          errors.push( I18n.t("campaign_mgmt.param_only_w_h_not_match_ad_unit", :ad_unit => "#{ad_unit_node.ad_unit.name}(##{ad_unit_node.id})" , :creative =>"#{creative.name}(##{creative.id})" ))
        end
      end
    end
  end
end
```

Can you read it? We have code wrapped inside 2 each and 3 if.
Why not

```ruby
@ad_unit_nodes.each do |ad_unit_node|
  @creatives.each{|creative| creative.should_have!(:the_same_dimension_as_ad_unit)} if ad_unit_node.is_valid_display?
end
```

Your reader might not know the business logic detail as much as you are. Assume them to be new hires.


## Conclusion

Ok, did you notice what I’m talking here is all about hide/expose complexity?

It’s my understanding, good code is an elegant expressing of the concept in your mind, for each level(Class, public method, private helper method) the complexity/detail has been exposed to the most proper level, no more no less.

With this principle in mind, the most difficult tasks for coding are now answering those questions:

- Which level is most appropriate for putting info the concept?
- Does this Class have the proper responsibility, isn’t it handling too much or too few?
- Any separation needed for that complex Class or method?

So finally coding is becoming an art. Just as how a beauty dresses, too much is not fashion but also too few is not widely acceptable(although some bachelor will like it).
