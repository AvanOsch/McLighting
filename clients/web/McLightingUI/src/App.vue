<template>
  <div id="app">

    <v-app v-bind:dark="dark_theme">
      <v-toolbar color="primary" dark>
        <!-- <v-toolbar-side-icon></v-toolbar-side-icon> -->
        <v-toolbar-title>Mc Lighting UI</v-toolbar-title>
        <v-spacer></v-spacer>
        <v-spacer></v-spacer>
        <span class="group pa-2" v-if="is_connected"><v-icon>wifi</v-icon>&nbsp;<span class="hidden-xs-only">&nbsp;McLighting </span>connected</span>
        <span class="group pa-2" v-else><v-icon>signal_wifi_off</v-icon>&nbsp;&nbsp;McLighting disconnected</span>
        <span v-if="num_additional_connections > 0">+{{ num_additional_connections }}</span>
        <span class="group ml-2" v-if="connection_failed" @click="ws_reconnect()"><v-btn color="secondary" ><v-icon>autorenew</v-icon>&nbsp;Reconnect</v-btn></span>
      </v-toolbar>

      <Loader v-if="loader_state > 0" :state="loader_state" max_states="4" />

      <v-content v-if="loader_state === 0" id="content" class="mx-auto">
        <v-card>
          <!-- Color selector -->
          <v-layout row wrap>
            <v-flex xs12 sm5>
              <ColorPicker v-on:selected="onColorSelected" :prop_color="color" :prop_type="picker_type"/>
            </v-flex>
            <v-flex xs12 sm7>
              <v-card-text>
                <v-slider v-model="color.r" label="R" :max="255" v-on:input="set_color" color="red" persistent-hint :hint="(color.r || 0)+''" :disabled="auto_mode"></v-slider>
                <v-slider v-model="color.g" label="G" :max="255" v-on:input="set_color" color="green" persistent-hint :hint="(color.g || 0)+''" :disabled="auto_mode"></v-slider>
                <v-slider v-model="color.b" label="B" :max="255" v-on:input="set_color" color="blue" persistent-hint :hint="(color.b || 0)+''" :disabled="auto_mode"></v-slider>
                <v-slider v-model="brightness" prepend-icon="wb_incandescent" :max="255" v-on:input="set_brightness" persistent-hint :hint="(brightness || 0)+''" :disabled="auto_mode"></v-slider>
                <v-slider v-model="speed" prepend-icon="slow_motion_video" :max="255" v-on:input="set_speed" persistent-hint :hint="(speed || 0)+''" :disabled="auto_mode"></v-slider>
                <v-switch v-model="auto_mode" prepend-icon="play_circle_outline" :label="`AUTO: ${(auto_mode) ? 'ON' : 'OFF'}`" color="primary"></v-switch>
              </v-card-text>
            </v-flex>
          </v-layout>

          <!-- Mode list -->
          <v-container pb-5>
            <v-layout row wrap>
              <v-flex xs12 sm6 lg4 xl3 v-for="(mode, index) in modes" :key="mode.id" v-if="edit_mode || !mode.hidden">
                <span v-if="edit_mode">
                  <v-icon
                    color="grey lighten-1"
                    @click="toggle(mode, index)"
                    v-if="mode.hidden === true"
                    >star_border</v-icon>
                  <v-icon
                    color="yellow darken-2"
                    @click="toggle(mode, index)"
                    v-else>star</v-icon>
                </span>

                <v-btn :id="mode.id" v-on:click="set_mode(mode.id)" :disabled="auto_mode" class="elevation-6 white--text" v-bind:class="[modeIsActive(mode) ? 'red' : 'blue']">{{ mode.title }} ({{ mode.id }})</v-btn>
              </v-flex>
            </v-layout>
          </v-container>

          <!-- Settings button -->
          <v-fab-transition>
            <v-btn
              v-show="!edit_mode"
              transition="scale-transition"
              @click.native.stop="edit_mode = !edit_mode"
              color="pink"
              dark
              absolute
              bottom
              right
              fab
              hover>
              <v-icon>settings</v-icon>
            </v-btn>
          </v-fab-transition>
        </v-card>

        <!-- Settings tab -->
        <v-card row wrap v-if="edit_mode" id="settings">
          <v-layout row wrap class="pa-3 my-2">
            <v-flex xs12 sm6>
              <p>Dark mode:</p>
              <v-switch :label="`${(dark_theme) ? 'ON' : 'OFF'} `" v-model="dark_theme" color="primary"></v-switch>
            </v-flex>
            <v-flex xs12 sm6>
              <p>Select color picker:</p>
              <v-btn-toggle v-model="picker_type" mandatory>
                <v-btn flat value="circle">
                  Circle
                </v-btn>
                <v-btn flat value="wheel">
                  Wheel
                </v-btn>
              </v-btn-toggle>
            </v-flex>
            <v-flex xs12>
              <v-text-field v-model="slave_nodes" label="Additional McLighting Slave Nodes (experimental)"></v-text-field>
            </v-flex>
            <v-flex xs12>
              <v-btn block color="green" dark class="elevation-6 mb-2" @click="saveSettings()"><v-icon>save</v-icon> Save settings</v-btn>
            </v-flex>
          </v-layout>
        </v-card>

        <v-footer color="primary white--text" class="pa-3">
          <div><a href="https://github.com/toblum/McLighting/" target="_blank" class="white--text">Project home</a> - &copy; {{ new Date().getFullYear() }}</div>
          <v-spacer></v-spacer>
        </v-footer>
      </v-content>

      <!-- Snackbar -->
      <v-snackbar
        :timeout="snackbar_timeout"
        top
        :color="snackbar_color"
        v-model="snackbar"
      >
        {{ snackbar_text }}
      </v-snackbar>
    </v-app>

  </div>
</template>

<script>
/* eslint-disable */
import ColorPicker from "./components/ColorPicker";
import Loader from "./components/Loader";
import ReconnectingWebSocket  from 'reconnecting-websocket';  // https://github.com/pladaria/reconnecting-websocket

var host = window.location.hostname;
if (host === "localhost") {
  host = "192.168.0.49";
}
const ws_url = "ws://" + host + ":81";

let ws_options = {
  connectionTimeout: 1000,
  maxRetries: 2,
  reconnectionDelayGrowFactor: 2,
  debug: true
};

export default {
  name: "McLightingUI",
  
  components: {
    ColorPicker,
    Loader
  },

  data: () => ({
    color: {r:0, g:0, b:0, hex:"000000", r2:0, g2:0, b2:0, hex2:"000000", r3:0, g3:0, b3:0, hex3:"000000"},
    brightness: 192,
    speed: 192,
    modes: [{ title: "OFF", id: "off" }, { title: "TV", id: "tv" }],
    connection: null,
    connection_failed: false,
    additional_connections: [],
    num_additional_connections: 0,
    is_connected: false,
    dark_theme: false,
    ws2812fx_mode: null,
    auto_mode: false,
    settings: {},
    edit_mode: false,
    picker_type: "circle",
    slave_nodes: "",
    snackbar: false,
    snackbar_text: "",
    snackbar_color: "info",
    snackbar_timeout: 2000,
    loader_state: 1
  }),

  watch: {
    auto_mode: {
      handler: function(val, oldVal) {
        this.toggleAutoplay(val);
      }
    }
  },

  methods: {
    getModes() {
      var that = this;
      this.$http.get("//" + host + "/get_modes").then(data => {
        // console.log("Getting modes list via REST:", data);
        data.body.forEach(item => {
          if (item.name && item.name.length > 0) {
            that.modes.push({ title: item.name, id: item.mode });
          }
        });
        this.readSettings();
      }, response => {
        console.error("ERROR loading modes", response);
        this.showSnackbar("Error loading animation modes, please try again...", "error", 5000);
      });
    },
    modeIsActive(mode) {
      return mode.id === this.ws2812fx_mode;
    },

    showSnackbar(message, color, timeout) {
      this.snackbar_text = message;
      this.snackbar_color = color;
      this.snackbar_timeout = timeout;
      this.snackbar = true;
    },

    readSettings() {
      let that = this;
      this.loader_state = 2;
      this.$http.get("//" + host + "/uistate.json").then(data => {
        console.log("readSettings()", data.body);
        this.settings = data.body || {};
        this.applySettings();
        this.connectAdditionalNodes();
        this.ws_connect();
      }, response => {
        console.warn("ERROR loading settings", response);
        this.ws_connect();
      });
    },
    applySettings() {
      this.modes.forEach((mode, index) => {
        if (this.settings.visibility.includes(mode.id)) {
          mode.hidden = true;
          this.$set(this.modes, index, mode);
        }
      });
      this.dark_theme = this.settings.dark_theme || false;
      this.picker_type = this.settings.picker_type || "circle";
      this.slave_nodes = this.settings.slave_nodes || "";
    },
    saveSettings() {
      var visibility = this.modes.map(mode => {
        if (mode.hidden) {
          return mode.id;
        }
      }).filter((x) => {return x >= 0});
      this.settings.visibility = visibility;
      this.settings.dark_theme = this.dark_theme;
      this.settings.picker_type = this.picker_type;
      this.settings.slave_nodes = this.slave_nodes;

      var formData = new FormData();
      var blob = new Blob([JSON.stringify(this.settings)], {type: 'application/json'});
      formData.append('settings', blob, '/uistate.json');

      this.$http.post("//" + host + "/edit", formData).then(data => {
        console.log("SUCCESS saveSettings()", data, this.settings);
        this.showSnackbar("Settings saved", "success", 1500);
        this.connectAdditionalNodes();
        this.edit_mode = false;
      }, err => {
        console.error("ERROR saveSettings()", err);
        this.showSnackbar("Error saving settings, please try again...", "error", 5000);
      });
    },

    toggle(mode, index) {
      mode.hidden = !mode.hidden;
      this.$set(this.modes, index, mode);
    },
    toggleAutoplay(mode) {
      var message = (mode) ? "start" : "stop";
      this.ws_send(message);
      if (!mode) {
        this.ws_send("$");
      }
    },

    disconnectAllAdditionalNodes() {
      // Close potentially open connections
      this.additional_connections.forEach((conn) => {
        conn.close(1000, "Manually closed::disconnectAllAdditionalNode()", {keepClosed: true});
      });
      this.additional_connections = [];
      this.num_additional_connections = 0;
    },
    connectAdditionalNodes() {
      var that = this;
      this.disconnectAllAdditionalNodes();

      // Connect to the configured nodes
      let nodes = this.slave_nodes.split(";");
      nodes.forEach((host) => {
        host = host.trim();
        if (host !== "") {
          let conn = new ReconnectingWebSocket(ws_url, "arduino", ws_options);
          console.log("Connecting to additional node", host);
          that.additional_connections.push(conn);
  
          conn.onopen = () => {
            console.log("Connected to additional node", host);
            this.num_additional_connections++;
          };
          conn.onclose = () => {
            console.log("Disconnected to additional node", host);
            that.num_additional_connections--;
          };
          conn.onerror = () => {
            console.log("Error on additional node", host);
          };
        }
      });
    },
    ws_reconnect() {
      this.connection.reconnect();
      this.connection_failed = false;
    },

    ws_connect() {
      var that = this;
      this.connection = new ReconnectingWebSocket(ws_url, "arduino", ws_options);
      this.loader_state = 3;

      // When the connection is open, send some data to the server
      this.connection.onopen = function() {
        console.log("WebSocket open");
        that.is_connected = true;
        that.loader_state = 4;
        setTimeout(() => {
          that.loader_state = 0;
          that.ws_send("$");
        }, 500);
      };

      // When the connection is open, send some data to the server
      this.connection.onclose = function() {
        console.log("WebSocket closed");
        that.is_connected = false;
      };

      // Log errors
      this.connection.onerror = function(error) {
        console.error("WebSocket error", error);
        that.is_connected = false;

        if (that.connection.retryCount >= ws_options.maxRetries) {
          that.connection_failed = true;
          that.showSnackbar("Could not connect to McLighting. Tried " + ws_options.maxRetries + " times. Please use the RECONNECT button to try again.", "error", 15000);
        }
      };

      // Log messages from the server
      this.connection.onmessage = function(e) {
        console.log("WebSocket from server:", e.data);
        try {
          var res = JSON.parse(e.data);
          // console.log("res", res);
          if (res && res.ws2812fx_mode) {
            that.ws2812fx_mode = res.ws2812fx_mode;
          }
          if (res && res.speed) {
            that.speed = res.speed;
          }
          if (res && res.brightness) {
            that.brightness = res.brightness;
          }
          if (res && res.color) {
            that.color.r = res.color[0];
            that.color.g = res.color[1];
            that.color.b = res.color[2];
            that.color.hex = that.rgbToHex([that.color.r, that.color.g, that.color.b]);
            that.color.r2 = res.color[3];
            that.color.g2 = res.color[4];
            that.color.b2 = res.color[5];
            that.color.hex2 = that.rgbToHex([that.color.r2, that.color.g2, that.color.b2]);
            that.color.r3 = res.color[6];
            that.color.g3 = res.color[7];
            that.color.b3 = res.color[8];
            that.color.hex3 = that.rgbToHex([that.color.r3, that.color.g3, that.color.b3]);
          }
        } catch (e) {}
      };
    },

    ws_send(message) {
      console.log("WS send: ", message);
      this.connection.send(message);

      this.additional_connections.forEach((conn) => {
        conn.send(message);
      });
    },

    set_mode(mode_id) {
      this.ws2812fx_mode = mode_id;
      if (Number.isInteger(mode_id)) {
        this.ws_send("/" + mode_id);
      } else {
        // For named modes
        this.ws_send("=" + mode_id);
      }
    },
    set_speed(speed) {
      this.ws_send("?" + speed);
    },
    set_brightness(brightness) {
      this.ws_send("%" + brightness);
    },
    set_color() {
      if (this.color.num === 1) {
        this.ws_send(
          "#" + this.rgbToHex([this.color.r, this.color.g, this.color.b])
        );
      }
      if (this.color.num === 2) {
        this.ws_send(
          "##" + this.rgbToHex([this.color.r2, this.color.g2, this.color.b2])
        );
      }
      if (this.color.num === 3) {
        this.ws_send(
          "###" + this.rgbToHex([this.color.r3, this.color.g3, this.color.b3])
        );
      }
    },

    componentToHex(c) {
      return ("0" + Number(c).toString(16)).slice(-2).toUpperCase();
    },
    rgbToHex(rgb) {
      return (
        this.componentToHex(rgb[0]) +
        this.componentToHex(rgb[1]) +
        this.componentToHex(rgb[2])
      );
    },

    onColorSelected(color) {
      if (color.num === 2) {
        this.color.num = color.num;
        this.color.r2 = color.r;
        this.color.g2 = color.g;
        this.color.b2 = color.b;
        this.color.hex2 = color.hex;
      } else if (color.num === 3) {
        this.color.num = color.num;
        this.color.r3 = color.r;
        this.color.g3 = color.g;
        this.color.b3 = color.b;
        this.color.hex3 = color.hex;
      } else {
        this.color.num = color.num;
        this.color.r = color.r;
        this.color.g = color.g;
        this.color.b = color.b;
        this.color.hex = color.hex;
      }
      this.set_color();
    }
  },

  mounted() {
    this.getModes();
  }
};
</script>

<style>
  #content {
    max-width: 1380px;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
  }

  /* fallback */
  @font-face {
    font-family: 'Material Icons';
    font-style: normal;
    font-weight: 400;
    /* src: url(https://fonts.gstatic.com/s/materialicons/v29/2fcrYFNaTjcS6g4U3t-Y5ZjZjT5FdEJ140U2DJYC3mY.woff2) format('woff2'); */
    src: url(../node_modules/material-design-icons-iconfont/dist/fonts/MaterialIcons-Regular.woff2) format('woff2');
  }

  .material-icons {
    font-family: 'Material Icons';
    font-weight: normal;
    font-style: normal;
    font-size: 24px;
    line-height: 1;
    letter-spacing: normal;
    text-transform: none;
    display: inline-block;
    white-space: nowrap;
    word-wrap: normal;
    direction: ltr;
    -webkit-font-feature-settings: 'liga';
    -webkit-font-smoothing: antialiased;
  }

  .btn__content {
    height: 100% !important;
  }
</style>
