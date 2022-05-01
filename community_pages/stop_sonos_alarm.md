This script blueprint provides a convenient way to stop the current or upcoming 
alarm on a Sonos speaker. When called, the script checks if audio is currently 
playing and if so, stops the audio. If not, the script checks if an alarm is coming soon 
and prevents it from going off by disabling the alarm and re-enabling after it
would have gone off. At most, one alarm is impacted.

The following field parameters can be given when the script is called:
* _[required]_ Sonos speaker (that the alarms are attached to)
* _[optional]_ Time window that is used to look forward when disabling a future alarm. 60 minutes if not specified.

This script is particularly convenient when:
* You wake up earlier than your alarm and don't want to wake others
* You don't want to fiddle with your phone or apps when you wake up
* You have have a phyiscal button near your bed (e.g. a Lutron Caseta Pico remote)

## Additional notes ##

* I recommend setting the mode to `parallel` when using the same script for multiple automations
* This blueprint has no inputs, but is meant to be a script to call from automations  
&nbsp;

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fscript%2Fstop_sonos_alarm.yaml)

````yaml
# ***************************************************************************
# *  Copyright 2022 Joseph Molnar
# *
# *  Licensed under the Apache License, Version 2.0 (the "License");
# *  you may not use this file except in compliance with the License.
# *  You may obtain a copy of the License at
# *
# *      http://www.apache.org/licenses/LICENSE-2.0
# *
# *  Unless required by applicable law or agreed to in writing, software
# *  distributed under the License is distributed on an "AS IS" BASIS,
# *  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# *  See the License for the specific language governing permissions and
# *  limitations under the License.
# ***************************************************************************

# Stops a currently playing or future alarm on a Sonos speaker.

blueprint:
  name: "Stop Sonos Alarm"
  description:
    This blueprint is used to add a script that stops current audio on the specified Sonos speaker 
    but if nothing is playing the script checks if an alarm is coming soon and if so, prevents it 
    from going off by disabling the alarm and then re-enabling after it would have gone off. Only 
    one alarm is impacted. 
    
    This is perfect for when you wake up earlier than your alarm and don't want the alarm to wake 
    others. 

    I recommend setting the mode to parallel if you will use this script on more than one speaker.
  domain: script

fields:
  entity_id:
    description: The entity id of the device to stop the alarm on.
    name: Entity
    required: true
    selector:
      entity:
        domain: media_player
        integration: sonos
  time_window:
    name: Time Window
    description:
      If music isn't playing this is the number of minutes (up to 4 hours) the script will look forward
      to find an active alarm to disable. The alarm is re-enabled about 5 minutes after the time it
      would have gone off.
    required: false
    default: 60
    selector:
      number:
        min: 0
        max: 240
        unit_of_measurement: minutes
        mode: slider
variables:
  alarm: >-
    {%- set entities = device_entities( device_id( entity_id ) ) -%}
    {# since the current day can flip while running this script and we want this as accurate #}
    {# as possible, we capture the current time right away and base all calculations in this #}
    {# script from that captured time, plus means we calculate just once #}
    {%- set current_time = now() -%}
    {# set for current midnight to help with calculating alarms for today below #}
    {# again, doesn't use today_at() to ensure no cross-over in the day during execution #}
    {%- set current_midnight = current_time.replace(hour=0,minute=0,second=0,microsecond=0) %} 
    {# we track tomorrow to help with cross-overs at midnight and we force no more than 4 hours for future alarm #}
    {%- set tomorrow_midnight = current_midnight + timedelta(days=1) -%} 
    {%- set current_weekday = current_midnight.isoweekday() -%}
    {%- if current_weekday == 7 %}
      {# the sonos alarms start at 0 and end at 6, while jinja starts at 1 and ends at 7#}
      {# making it convenient that we only need to fix up Sunday #}
      {%- set current_weekday = 0 %}
    {%- endif -%}
    {# we use this namespace track the alarm time info we find #}
    {%- set alarm_timing = namespace( days = "", time={}) -%}
    {# we set delta in this structure to help processing later, this is also where we cap time to 4 hours #}    
    {%- set found_alarm = namespace( found = false, entity_id = "", delta = timedelta(minutes=60 if time_window is undefined else 240 if time_window > 240 else time_window )) -%} 
    {# we iterate through all the entities on the specified entity looking for alarms #}
    {%- for entity_id in entities -%}
      {%- set entity_id_parts = entity_id.split('.') -%}
      {%- set entity = states[entity_id_parts[0]][entity_id_parts[1]] -%}
      {%- set alarm_id = state_attr( entity_id, "alarm_id") -%}
      {# we see if we found a switch that has an alarm #}
      {%- if entity.domain == "switch" and alarm_id != None and entity.state == "on" -%}
        {# we need to figure out which days the alarm is on #}
        {%- set recurrence = state_attr( entity_id, "recurrence") -%}
        {%- if recurrence == "DAILY" -%}
          {# if a daily alarm, we just indicate we will look across all the days #}
          {%- set alarm_timing.days = "1234567" -%}
        {%- else -%}
          {# if not a daily alarm, the format is ON_#### (e.g. ON_1236), where ### are numbers of the weekdays #}
          {%- set alarm_timing.days = recurrence.split('_')[1] -%}
        {%- endif -%}
        {# we need the time as a delta so we can add later, yes this is hefy work #}
        {# but I don't know another way to take a string and turn into a timedelta #}
        {%- set alarm_time_parts = state_attr( entity_id, "time").split(':') -%}
        {%- set alarm_timing.time = timedelta( hours=alarm_time_parts[0] | int, minutes=alarm_time_parts[1] | int ) -%}
        {# so now we loop through each day to see if we have a matching alarm #}
        {# ideally we'd shortcircuit once we find it or go too far #}
        {%- for day_string in alarm_timing.days %}
          {%- set day = day_string | int -%}
          {%- if day == current_weekday -%}
            {# if the alarm is for today then we use current_midnight as a basis for time comparison #}
            {%- set alarm_timestamp = current_midnight + alarm_timing.time-%}
            {%- set delta = alarm_timestamp - current_time -%}
            {# delta can't be negative, and replace alarm if delta is less than the last found (which includes never found) #}
            {%- if delta >= timedelta() and found_alarm.delta >= delta -%}
              {%- set found_alarm.found = true -%}
              {%- set found_alarm.entity_id = entity_id %}
              {%- set found_alarm.delta = delta %}
            {%- endif -%}
          {%- elif day == ( current_weekday + 1 ) or (day == 0 and current_weekday == 6) %}
            {# if the alarm is for tomorrow then we use tomorrow_midnight as a basis for time comparison #}
            {%- set alarm_timestamp = tomorrow_midnight + alarm_timing.time-%}
            {%- set delta = alarm_timestamp - current_time -%}
            {# delta can't be negative, and replace alarm if delta is less than the last found (which includes never found) #}
            {%- if delta >= timedelta() and found_alarm.delta >= delta -%}
              {%- set found_alarm.found = true -%}
              {%- set found_alarm.entity_id = entity_id %}
              {%- set found_alarm.delta = delta %}
            {%- endif -%}
          {%- endif -%}
        {%- endfor %}
      {%- endif -%}
    {%- endfor -%}
    {# if we find the alarm then we can stop / pause it#}
    {%- if found_alarm.found == true %}
      {# return back the delta to help calculate when to re-enable the alarm, yes missing microseconds, but close enough #}
      {% set delta_minutes = ( found_alarm.delta.seconds/60 ) | int %}
      {% set delta_seconds = found_alarm.delta.seconds - (delta_minutes * 60) %}
      {{ { "found": true, "entity_id": found_alarm.entity_id, "delta": { "minutes": delta_minutes, "seconds" : delta_seconds } } }}
    {%- else -%}
      {{ { "found": false } }}
    {%- endif -%}

sequence:
  - choose:
      # we see if the speaker is playing and if so stop it
      - conditions:
          - condition: template
            value_template: >
              {{ is_state(entity_id, 'playing') }}
        sequence:
          - service: media_player.media_stop
            data:
              entity_id: >
                {{ entity_id }}
      # if device isn't playing then we see if we found an alarm and if not, do nothing
      - conditions:
          - condition: template
            value_template: >
              {{ alarm.found == false }}
        sequence: []
                  
    default:
      # if we got here the speaker isn't playing and we found a suitable alarm
      - service: switch.turn_off
        data:
          entity_id: >
            {{ alarm.entity_id }}
      - delay:
          # we now wait to re-enable the alarm by using the delta to when the alarm
          # was suppose to go off, but adding five minutes, just to be safe
          hours: 0
          minutes: >
            {{ alarm.delta.minutes + 5 }} 
          seconds: >
            {{ alarm.delta.seconds }} 
      - service: switch.turn_on
        data:
          entity_id: >
            {{ alarm.entity_id }}

# we run in parallel since this may take some time to run and more than one alarm
# may be attempted to be turned off in a household
mode: parallel
icon: mdi:alarm
````
&nbsp;
# Additional Bluepints #
* [Play Media Script w/ Shuffle, Repeat, Multi-room, Volume support](https://community.home-assistant.io/t/play-media-script-w-shuffle-repeat-multi-room-volume-support/415234)
* Others in my [GitHub repository](https://github.com/Talvish/home-assistant-blueprints)