# Fade Media Player Volume

I struggled to find any comprehensive solutions on the Home Assistant forums for fading media player volumes with common attenuation curves, so put this script together.

This script fades the volume of a `target_player` media player, starting at it's current `volume_level`, to a user-defined `target_volume` over the user-defined `duration` in seconds. It also applies one of three `curve` algorithms to shape fade, defaulting to `logarithmic`, which is often considered the most natural sounding fade.

For those interested, the script is fully commented, and I've put together a quick explanation of the script's working below.

## Input Parameters
- **target_player**: media_player entity
- **target_volume**: float 0 - 1 --- volume to end at
- **duration**: float 0.1 - 60 --- duration of fade in seconds 
- **curve**: selector [logarithmic, bezier, linear]

## Timing Limitation
From what I could gather, Home Assistant calls its services on a 1 second clock. I don't know the details, however it's clear that sub-second delay calls aren't timed perfectly. So don't expect the fade duration to be perfect. The larger the duration, the more noticeable the duration discrepancy will be.

To make this script duration-accurate, instead of defining `total_steps`, a `steps_left` value could be used, defined by the script's `start_time`, `end_time` (which would be fixed), and the `current_time` for each iteration of the loop. The repeat condition could then use a pre-defined end-time, with the fade steps increasing or decreasing depending on if the calls are lagging or ahead..... but I've already spent way too much time on this, so be my guest :)

## Working

The script works first calculating the number of `total_steps` required to fade based on the user-defined `duration` multiplied by a hard-coded `step_duration` of `100ms` (or 10 per second). For example, a duration of 5 seconds equates to 500 steps.

It determines the difference between the media player's current volume and the user-defined `target_volume`. It applies this value as a factor to the shaped fade amount, adds it to the original volume, and applies it to the media player entity `volume_level` for each step in a `while` loop.

## Algorithms:
Where `x` is the normalised time value based on the `duration`, starting at `0` and ending at `1`.
- (red) Linear: `f(x) = x` 
- (blue) Bezier: `f(x) = x / (1 + (1 - x ))`
- (green) Logarithmic: `f(x) = x * x * (3 - 2x)`

<img src="https://bs.chrisvik.com/uploads/images/gallery/2021-10/scaled-1680-/fade-curves.png" alt="drawing" width="300"/>


## Script code

Add this to your Home Assistant `script` config `.yaml`, and use it anywhere that allows a service call (such as automations):

```yaml
fade_volume:
  alias: Fade the volume of a media player
  mode: restart
  # User-defined inputs to use.
  fields:
    target_player:
      name: Target media player
      description: "Target media player of volume fade."
      required: true
      example: media_player.lounge_sonos
      selector:
        entity:
          domain: media_player
    target_volume:
      name: Target volume
      description: "Volume the media play will be at the end of the fade duration."
      required: true
      default: 0.5
      example: '0.5'
      selector:
        number:
          max: 1.0
          min: 0.0
          step: 0.01
          mode: slider
    duration:
      name: Fade duration
      description: "Length of time in seconds the fade should take."
      required: true
      default: 5
      example: '5'
      selector:
        number:
          max: 60
          min: 0.1
          step: 0.1
          mode: box
          unit_of_measurement: "s"
    curve:
      name: Fade curve algorithm
      description: "Shape of the fade curve to apply."
      required: true
      default: 'logarithmic'
      example: 'logarithmic'
      selector:
        select:
          options:
            - logarithmic
            - bezier
            - linear
  variables:
    # Hard-coded temporal granularlity.
    steps_per_second: 10
    # An integer denoting the total steps required to fade based on the user-defined duration and steps per second.
    total_steps: "{{ (steps_per_second * duration) | int(0) }}"
    # Define the difference between start point and target, used to scale each fade step.
    start_volume: "{{ state_attr(target_player, 'volume_level') | float(0) }}"
    start_diff: "{{ (target_volume - start_volume) | float(0) }}"
  sequence:
    - repeat:
        # Only continue if the following conditions are true:
        while:
            # Pre-calculated total step index has not been reached.
          - condition: template
            value_template: "{{ repeat.index < total_steps }}"
            # Media player's current volume is not close to the target, otherwise we're just wasting processing time.
          - condition: template
            value_template: "{{ ((state_attr(target_player, 'volume_level') - target_volume) | abs) > 0.001 }}"
        sequence:
          - service: media_player.volume_set
            data_template:
              entity_id: '{{ target_player }}'
              # Defines x as the normalised time over the duration based on the repeat index.
              # Then applies the fade curve on each step, multiplied by the difference factor.
              volume_level: >
                {% set t = repeat.index / total_steps %}
                {% if curve == 'logarithmic' %}
                  {{ (start_volume + (t / (1 + (1 - t))) * start_diff) | float(0) }}
                {% elif curve == 'bezier' %}
                  {{ (start_volume + (t * t * (3 - 2 * t)) * start_diff) | float(0) }}
                {% else %}
                  {{ (start_volume + t * start_diff) | float(0) }}
                {% endif %}
          # Pause to limit the update rate.
          # Apparently HA has issues with sub-second accuracy, so 100ms will have to do.
          - delay: '00:00:00.1'
    - service: media_player.volume_set
      data_template:
        entity_id: '{{ target_player }}'
        volume_level: '{{ target_volume }}'
```
