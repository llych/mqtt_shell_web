<template>
  <div class="xterm-main">
    <div style="display: inline-block">
      <el-button
        style="padding: 5px"
        :type="isConnect ? 'success' : 'danger'"
        @click="dialogVisible = true"
        >{{ isConnect ? "已连接" : "未连接" }}
      </el-button>
      <div style="color: #645f5f;display: inline-block; margin-left: 20px;">
        <span>{{ mqttUrl }} || {{ device }}</span>
      </div>
    </div>
    <div id="xterm" class="xterm" />
    <el-dialog v-model="dialogVisible" title="连接配置" width="30%">
      <el-form :model="form" label-width="120px">
        <el-form-item label="broker">
          <el-input v-model="form.broker" placeholder="wss[mqtts]://xx:port" />
        </el-form-item>
        <el-form-item label="username">
          <el-input v-model="form.username" />
        </el-form-item>
        <el-form-item label="password">
          <el-input v-model="form.password" type="password" />
        </el-form-item>
        <el-form-item label="device">
          <el-input v-model="form.device" placeholder="设备名" />
        </el-form-item>
      </el-form>
      <template #footer>
        <span class="dialog-footer">
          <el-button type="primary" @click="onSubmit"> 确定 </el-button>
        </span>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { onMounted, ref } from "vue";
import * as mqtt from "mqtt/dist/mqtt.min";
import "xterm/css/xterm.css";
import { Terminal } from "xterm";
import { FitAddon } from "xterm-addon-fit";
import { SearchAddon } from "xterm-addon-search";
import { WebLinksAddon } from "xterm-addon-web-links";

const customKeyEventHandler = (e) => {
  if (e.type !== "keydown") {
    return true;
  }
  if (e.ctrlKey && e.shiftKey) {
    const key = e.key.toLowerCase();
    if (key === "v") {
      // ctrl+shift+v: paste whatever is in the clipboard
      navigator.clipboard.readText().then((toPaste) => {
        term.value.writeText(toPaste);
      });
      return false;
    } else if (key === "c" || key === "x") {
      // ctrl+shift+x: copy whatever is highlighted to clipboard

      // 'x' is used as an alternate to 'c' because ctrl+c is taken
      // by the terminal (SIGINT) and ctrl+shift+c is taken by the browser
      // (open devtools).
      // I'm not aware of ctrl+shift+x being used by anything in the terminal
      // or browser
      const toCopy = term.value.getSelection();
      navigator.clipboard.writeText(toCopy);
      term.value.focus();
      return false;
    }
  }
  return true;
};

const dialogVisible = ref(false);
const form = ref({ broker: "", device: "", username: "", password: "" });
const term = ref();
const fitAddon = ref();

const initTerm = () => {
  fitAddon.value = new FitAddon();
  term.value = new Terminal({
    fontSize: 14,
    cursorBlink: true,
  });

  term.value.loadAddon(fitAddon.value);
  term.value.loadAddon(new SearchAddon());
  term.value.loadAddon(new WebLinksAddon());
  term.value.open(document.getElementById("xterm"));
  fitAddon.value.fit();
  term.value.focus();
  term.value.attachCustomKeyEventHandler(customKeyEventHandler);

  term.value.onData((data) => {
    sendMsq(data);
  });
};

const device = ref("");

const sendMsq = (msg) => {
  client.value.publish(
    `/shell/${device.value}/input`,
    JSON.stringify({ data: msg, type: "data" })
  );
};

const isConnect = ref(false);
const client = ref();
const mqttUrl = ref("");
const username = ref("");
const password = ref("");
const initMqtt = () => {
  client.value = mqtt.connect(mqttUrl.value, {
    username: username.value,
    password: password.value,
  });

  const mqttc = client.value;
  mqttc.subscribe(`/shell/${device.value}/output`);
  mqttc.on("message", function (topic, payload) {
    const data = JSON.parse(payload);
    term.value.write(data.data);
  });

  mqttc.on("connect", function () {
    isConnect.value = true;
  });

  mqttc.on("disconnect", () => {
    isConnect.value = false;
  });

  mqttc.on("error", (err) => {
    console.log(err);
  });

  resetSize();
};

const onSubmit = () => {
  initConfig();
  initMqtt();
  dialogVisible.value = false;
  term.value.focus();
};

const debounce = (func, waitMs) => {
  let timeout;
  return function (...args) {
    const context = this;
    clearTimeout(timeout);
    timeout = setTimeout(() => func.apply(context, args), waitMs);
  };
};

const resetSize = () => {
  fitAddon.value.fit();
  client.value.publish(
    `/shell/${device.value}/input`,
    JSON.stringify({
      type: "size",
      rows: term.value.rows,
      cols: term.value.cols,
    })
  );
};

const initConfig = () => {
  if (client.value) {
    client.value.end();
  }
  mqttUrl.value = form.value.broker;
  password.value = form.value.password;
  username.value = form.value.username;
  device.value = form.value.device;
  localStorage.setItem("config", JSON.stringify(form.value));
};

onMounted(() => {
  initTerm();
  try {
    const data = localStorage.getItem("config");
    const jsonData = JSON.parse(data);
    form.value = { ...form.value, ...jsonData };
  } catch (e) {
    console.log(e);
  }
  window.onresize = debounce(resetSize, 1000);
});
</script>

<style scoped>
.xterm {
  height: 95vh;
}
</style>
