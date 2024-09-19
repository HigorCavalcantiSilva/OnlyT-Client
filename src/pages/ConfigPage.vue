<template>
  <q-page>
    <q-input
      style="width: 80%"
      label="IP/Host OnlyT"
      placeholder="(Ex.: 192.169.1.2)"
      @blur="validateHost()"
      @update:model-value="saveLocalhost"
      v-model="host"
    />
    <q-input
      style="width: 80%"
      label="Porta"
      @update:model-value="saveLocalhost"
      v-model="port"
    />
  </q-page>
</template>

<script>
import { defineComponent } from "vue";

export default defineComponent({
  name: "ConfigPage",
  data() {
    return {
      host: "",
      port: 8096,
    };
  },
  mounted() {
    this.host =
      localStorage.getItem("ip_onlyt") != null &&
      localStorage.getItem("ip_onlyt") != "null"
        ? localStorage.getItem("ip_onlyt")
        : "";
    this.port =
      localStorage.getItem("port_onlyt") != null &&
      localStorage.getItem("port_onlyt") != "null"
        ? localStorage.getItem("port_onlyt")
        : "";
  },
  methods: {
    validateHost() {
      const regex =
        /^(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$/;

      if (
        this.host != null &&
        this.host != "" &&
        this.host != "localhost" &&
        !regex.test(this.host)
      )
        return alert("ERRO! FORMATO DE IP INVÁLIDO");

      localStorage.setItem("ip_onlyt", this.host);
      localStorage.setItem("port_onlyt", this.port);
    },
    saveLocalhost() {
      localStorage.setItem("ip_onlyt", this.host);
      localStorage.setItem("port_onlyt", this.port);
    },
  },
});
</script>
