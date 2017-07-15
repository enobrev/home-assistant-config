# Tracking Approximate "Darkness"

Originally Posted on [Reddit](https://www.reddit.com/r/homeautomation/comments/67680g/is_it_possible_to_save_alias_triggers_in_home/dgob0tc/) and the [HA Forums](https://community.home-assistant.io/t/is-it-possible-to-save-alias-triggers/16285/4)

I have a sensor for tracking the sun angle. This was directly from the docs [somewhere](https://home-assistant.io/components/sensor.template/#sun-angle). I found it useful for getting the elevation as a float.  I also have a sensor that takes a couple other sensors (sun angle, dark sky cloudiness, dark sky precipitation probability) and generates a “darkness percentage”.

A cloudy afternoon would be about a 30%.  Midnight, Overcast, and Raining is 100%.  Sunny Day, Clear Sky is 0.

I use an input_slider to track that percentage, and my lights are based upon the input-slider for whether they should be on, and how bright they should be.  The outside lights use the whole scale from 0 to 10, while my indoor motion sensors don’t start affecting lights until darkness is at least 2 (20%).  I like using the slider rather than directly using the darkness sensor, as it allows me to adjust the slider in the interface to test the brightness while looking at the lights.

## Sensors

Here are the solar angle and darkness sensors.  It looks hairier than it is. I'll explain it below::

    - platform: template
      sensors:
          solar_angle:
              value_template: '{{ "%+.1f"|format(states.sun.sun.attributes.elevation) }}'
              friendly_name: 'Sun Angle'
              unit_of_measurement: '°'
          darkness:
              friendly_name: 'Darkness'
              unit_of_measurement: '%'
              value_template: >
                  {# ##### Max Angle is Appx Sun in the Sky, Min Angle is Appx Night Time #}
                  {% set min_angle = -20 %}
                  {% set max_angle =  8 %}
                  {# ##### Our Starting Values #}
                  {% set angle  = states.sensor.solar_angle.state                 | float %}
                  {% set clouds = states.sensor.dark_sky_cloud_coverage.state     | float %}
                  {% set p_rain = states.sensor.dark_sky_precip_probability.state | float %}
                  {# ##### Crop our Angle to our Range #}
                  {% if angle > max_angle %}
                      {% set angle = max_angle %}
                  {% elif angle < min_angle %}
                      {% set angle = min_angle %}
                  {% endif %}
                  {# ##### Figure out the percentage our angle falls within our range, and then subtract if from 1 for it to mean 'darkness' rather than 'brightness' #}
                  {% set p_angle   = 1 - ((angle - min_angle) / (max_angle - min_angle)) %}
                  {% set p_clouds  = clouds / 100 %}
                  {# ##### Get the average of the three, with adjustments for importance in our final value #}
                  {% set p_average = ((7 * p_angle) + (2 *  p_clouds) + p_rain) / 10 %}
                  {# ##### Multiply our average percentage by 10 and round it for our final result #}
                  {{ (p_average * 100) }}

## Slider

    darkness:
        name: Outside Darkness
        initial: 0
        min: 0
        max: 10
        step: 1

## Automation

        - alias: "Darkness Sensors Changed"
          trigger:
              platform: state
              entity_id: sensor.darkness
          action:
              - service: input_slider.select_value
                data_template:
                  entity_id: input_slider.darkness
                  value: '{{ states.sensor.darkness.state | int / 10 | round }}'

## Light Automation Example

        - alias: "Front Porch Light Off"
          trigger:
              platform: state
              entity_id: input_slider.darkness
          condition:
              condition: template
              value_template: '{{ states.input_slider.darkness.state | int < 1 }}'
          action:
              service: light.turn_off
              entity_id: light.front_porch_level # find-light-front_porch

        - alias: "Front Porch Light On"
          trigger:
              platform: state
              entity_id: input_slider.darkness
          condition:
              condition: template
              value_template: '{{ states.input_slider.darkness.state | int > 0 }}'
          action:
              service: light.turn_on
              data_template:
                  entity_id: light.front_porch_level # find-light-front_porch
                  brightness: '{{ states.input_slider.darkness.state | int * 15 }}'



----------

# The Darkness Math

Setting the "Max" and "Min" values. My goal is to set this value to basically mean 100% Daytime and 100% Nighttime. So, after, say 9am, it's daytime. I don't need my lights to dim or brighten or whatever. It's day. And for night, after, say 11pm, it's fully night-time. No need to worry about it beyond then.

So using this mix / max angle values is meant to say these are the important values for me. Anything beyond them should just be considered 100% day or 100% night. I didn't do anything magical to come up with these values. I just looked at the graph for my "sun_angle" sensor and picked where I wanted my range to be.

```
    {# ##### Max Angle is Appx Sun in the Sky, Min Angle is Appx Night Time #}
    {% set min_angle = -20 %}
    {% set max_angle =  8 %}
```

These are just variables. I'm going to use them later and I don't want to deal with "states.sensor.etc.etc". The "p_" prefix means percentage (actually a number between 0 and 1). The Precipitation Intensity is already a percentage. The other two are not yet, and I want them to be, which is coming...

```
    {# ##### Our Starting Values #}
    {% set angle  = states.sensor.solar_angle.state                 | float %}
    {% set clouds = states.sensor.dark_sky_cloud_coverage.state     | float %}
    {% set p_rain = states.sensor.dark_sky_precip_probability.state | float %}
```

If the angle is beyond the bounds I described above, just set them to the bounds. So if the sun angle is 10 and my max angle is 8, just set it to 8. Anything over 8 is useless information

```
    {# ##### Crop our Angle to our Range #}
    {% if angle > max_angle %}
      {% set angle = max_angle %}
    {% elif angle < min_angle %}
      {% set angle = min_angle %}
    {% endif %}
```

Here, I convert the angle and cloud cover value to decimals between 0 and 1 so they're all in the same realm. So .8 is 80%. .2 is 20%. clouds is already just a number between 0 and 100 so I divide it by 100.

Angle is going to be a number in between -20 and 8 (my min and max angle). So the equation figures out where it is on a percentage scale. So if angle is -20 it's 0, and if it's 8 it's 1. If it's -11, it's .5 (50% of the way between -20 and 8). Then I subtract it from 1 because I want percentage darkness, not brightness.

```
    {# ##### Figure out the percentage our angle falls within our range, and then subtract if from 1 for it to mean 'darkness' rather than 'brightness' #}
    {% set p_angle   = 1 - ((angle - min_angle) / (max_angle - min_angle)) %}
    {% set p_clouds  = clouds / 100 %}
```

This came from some trial and error, but the end result is 7 parts sun angle, 2 parts cloudiness, and 1 part rain. I add them all up and divide by 10 to get the average. There are probably better ways to do this. Anyways, the basic idea is that the sun angle is the most important value. The cloudiness matters, but not much. And precipitation counts, but barely. So if it's day time, but raining, the value is probably going to be 2 or 3.

```
    {# ##### Get the average of the three, with adjustments for importance in our final value #}
    {% set p_average = ((7 * p_angle) + (2 *  p_clouds) + p_rain) / 10 %}
```

My average is a number between 0 and 1 again. The 10 here isn't referring to the 10 inputs like in the math for the average just above this note, but instead for the slider. If my slider was 0-50, then this would be p_average * 50. I think the slider can take non-integers (decimals), but just in case it can't, I round the number as well.

```
    {# ##### Multiply our average percentage by 10 and round it for our final result #}
    {{ (p_average * 10) | round }}
```

That last line is just "printed" as the template_value.  It’s the only value in that whole block that is output, so that's what the sensor value is set to.


----------

The reason I went with % darkness instead of % brightness is because it correlated best with my uses. For instance, I control my outside lights with this. And I want my lights to get brighter the darker it is. So a 2 on the slider means 20% brightness, and a 10 on the slider means 100% brightness for my lights. The darker it is outside, the brighter i want my lights.

I also have a motion sensor in my dining room that doesn't activate my lights unless "darkness" is more than 2.