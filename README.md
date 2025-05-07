# Fractional framebuffer.
A "new" approach to get rid of the flickering on ESP8266.

Allocates only 1/N of the total display area, and calls the drawing code N times, looping over fragments.

This is based on the behavior of [u8glib](https://docs.arduino.cc/libraries/u8glib/).

## Pros:

* No flicker.
* Lower RAM usage.
* Faster than framebuffer-less approach.

## Cons:

* Slower than full framebuffer.
* Lambda will be called once per fragment (not the usual once per frame).
* Based implemented only on the `st7789v` driver, which is deprecated.

## Usage:

* Setup external_components as below.
```yaml
external_components:
  - source:
      type: git
      url: https://github.com/lhartmann/esphome-st7789v-fractional-framebuffer
      ref: main
    refresh: 0s
    components: [st7789v]
    ...
```
* Add fragmentation: 3 or more to the display entry.
```yaml
display:
  - platform: st7789v
    fragmentation: 30
    ...
```

## Optional, For Extra Performance on ESP8266
* Bump the ESP8266 to 160MHz (see `platformio_options` in the example).
* Bump the SPI to 40MHz (see `data_rate` in the example).

## Full Example

```yaml
esphome:
  name: geekmagic-smalltv-01
  friendly_name: geekmagic-smalltv-01
  platformio_options: 
    board_build.f_cpu: 160000000L
  on_boot:
    - priority: 600
      then:
        - delay: 1s
        - output.turn_on: pwm_output
        - output.set_level:
            id: pwm_output
            level: 50%

esp8266:
  board: esp12e

external_components:
  - source:
      type: git
      url: https://github.com/lhartmann/esphome-st7789v-fractional-framebuffer
      ref: main
    refresh: 0s
    components: [st7789v]

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: !secret api_key

ota:
  - platform: esphome
    password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  # ap:
  #   ssid: "Geekmagic-Smalltv-01"
  #   password: !secret fallback_password

captive_portal:

time:
  - id: ha_time
    platform: homeassistant
    timezone: America/Recife

font:
  - id: roboto
    file: "gfonts://Roboto"
    size: 20
  - id: spleen64
    file:
      type: web
      url: https://raw.githubusercontent.com/fcambus/spleen/refs/heads/master/spleen-32x64.bdf
    glyphs: "0123456789:"
      
image:
  - id: bg_image
    file: https://lcsvh.com/ironman.png
    type: rgb565
    resize: 240x240

spi:
  clk_pin: GPIO14
  mosi_pin: GPIO13
  interface: hardware
  id: spihwd

output:
  - id: pwm_output
    platform: esp8266_pwm
    pin: GPIO05
    inverted: true
    frequency: 1000 Hz

light:
  - platform: monochromatic
    output: pwm_output
    name: "Backlight"

color:
  - id: color_ironman_red
    hex: eb1c24
  - id: color_ironman_yellow
    hex: fcb712

interval:
  - interval: 5s
    then:
      - display.page.show_next: my_display
      - component.update: my_display

display:
  - id: my_display
    platform: st7789v
    model: custom
    spi_id: spihwd
    height: 240
    width: 240
    offset_height: 0
    offset_width: 0
    fragmentation: 30
    dc_pin: GPIO00
    reset_pin: GPIO02
    eightbitcolor: False
    update_interval: never
    spi_mode: mode3
    data_rate: 40000000
    auto_clear_enabled: False
    pages:
      - id: page_clock
        lambda: |-
          // Calculate the center positions
          int W = it.get_width();
          int H = it.get_height();

          // Background image (also replaces autoclear)
          it.image(0, 0, id(bg_image), ImageAlign::TOP_LEFT);

          // This is a bad-example, to demonstrate side-effect multiplication.
          static int framecounter = 0;
          it.printf(W, H, id(terminus12n), id(color_ironman_yellow), TextAlign::BOTTOM_RIGHT, "%d", framecounter++);

          auto now = id(ha_time).now();
          if (now.is_valid()) {
            it.printf( +12, 12, id(spleen64), id(color_ironman_red),    TextAlign::TOP_LEFT,  "%02d", now.hour);
            it.printf(W-12, 12, id(spleen64), id(color_ironman_yellow), TextAlign::TOP_RIGHT, "%02d", now.minute);

            it.printf(W/2, H, id(roboto), id(color_ironman_yellow), TextAlign::BOTTOM_CENTER, "%02d/%02d/%04d", now.day_of_month, now.month, now.year);
          }

      - id: page_blankish
        lambda: |-
          // This is a bad-example, will not be a solid color, but stripes.
          it.fill(Color(rand()));

```
