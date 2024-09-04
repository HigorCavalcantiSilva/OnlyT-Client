<template>
  <q-page class="flex flex-center">
    <q-card
      :class="`my-card text-${color_timer} q-mb-md q-mt-md text-center`"
      style="
        background: radial-gradient(circle, #35a2ff 0%, #014a88 100%);
        position: fixed;
        z-index: 999;
        top: 35px;
        width: 96%;
      "
    >
      <q-card-section>
        <div class="text-h1">
          {{ minutesRemaining.toString().padStart(2, "0") }}:{{
            secondsRemaining.toString().padStart(2, "0")
          }}
        </div>
      </q-card-section>
    </q-card>
    <div v-if="!connected" class="text-h5 text-center q-mt-md">
      {{ status }}
    </div>
    <q-list
      bordered
      separator
      style="width: 95%; margin-top: 160px"
      class="q-mb-md"
    >
      <q-item clickable v-ripple v-for="talk in talks" :key="talk.talkId">
        <q-item-section
          >{{ talk.talkTitle }} <br />
          ({{ convertMinutes(talk.originalDurationSecs) }} min)</q-item-section
        >
        <q-item-section top side>
          <div class="text-grey-8 q-gutter-xs">
            <q-btn
              v-if="!isRunning[talk.talkId]"
              color="secondary"
              :label="
                done[talk.talkId].minutes === 0 &&
                done[talk.talkId].seconds === 0
                  ? 'INICIAR'
                  : 'REINICIAR'
              "
              icon="play_arrow"
              @click="startTalk(talk.talkId)"
            />
            <q-btn
              v-if="isRunning[talk.talkId]"
              color="deep-orange"
              label="PARAR"
              icon="stop"
              @click="stopTalk(talk.talkId)"
            />
            <div
              class="text-h6"
              :style="
                'color: ' +
                (done[talk.talkId].minutes === 0 &&
                done[talk.talkId].seconds === 0
                  ? 'red'
                  : 'green')
              "
            >
              Feito:
              {{ done[talk.talkId].minutes.toString().padStart(2, "0") }}:{{
                done[talk.talkId].seconds.toString().padStart(2, "0")
              }}
            </div>
          </div>
        </q-item-section>
      </q-item>
    </q-list>
  </q-page>
</template>

<script>
import { defineComponent, ref, reactive } from "vue";

export default defineComponent({
  name: "IndexPage",
  data() {
    return {
      talks: [],
      minutesRemaining: 0,
      secondsRemaining: 0,
      countdown: null,
      isRunning: reactive({}),
      done: reactive({}),
      host: localStorage.getItem("ip_onlyt"),
      port: localStorage.getItem("port_onlyt"),
      status: "Tentando se conectar ao OnlyT remoto...",
      connected: false,
      currentDone: 0,
      negative: false,
      readyNegative: false,
      color_timer: "white",
    };
  },
  created() {
    if (this.port == null || this.port == "")
      localStorage.setItem("port_onlyt", "8096");

    if (this.host == null || this.host == "" || this.host == "null") {
      alert("Por favor insira o IP/Host na tela de configurações");
      this.$router.push("/config");
      return;
    }

    this.getTalks();
  },
  methods: {
    getTalks() {
      this.$axios
        .get(`http://${this.host}:${this.port}/api/v1/timers`)
        .then((res) => {
          this.connected = true;
          this.talks = res.data.timerInfo.filter(
            (item) => item.originalDurationSecs !== 0
          );

          this.fillRunning();
          this.fillDone();
          this.verifyTalkIsRunning(res.data.status);
          this.verifyTalkDones();
        })
        .catch((err) => {
          this.status =
            "Erro ao se conectar. Recarregue a página e/ou verifique as configurações";
          alert(
            "Erro ao tentar conectar-se ao OnlyT. Verifique o IP/Host e a porta na tela de configurações"
          );
          console.error(err);
        });
    },
    startTalk(talkId) {
      let idx = this.verifyCountdownActive();
      if (idx !== null) {
        this.stopTalk(idx);
      }

      this.done[talkId].minutes = 0;
      this.done[talkId].seconds = 0;

      setTimeout(() => {
        this.$axios
          .post(`http://${this.host}:${this.port}/api/v1/timers/${talkId}`)
          .then((res) => {
            let talk = res.data;
            let time = talk.currentStatus.targetSeconds;

            this.adaptTimer(time);

            setTimeout(() => {
              this.initCountdown(talkId);
              this.isRunning[talkId] = true;
            }, 500);
          })
          .catch((err) => {
            console.error(err);
          });
      }, 1000);
    },
    stopTalk(talkId) {
      fetch(`http://${this.host}:${this.port}/api/v1/timers/${talkId}`, {
        method: "DELETE",
        headers: {
          "Content-Type": "application/json",
        },
      })
        .then((response) => {
          if (!response.ok) {
            throw new Error("Network response was not ok");
          }
          return response.json(); // ou `response.text()` se a resposta não for JSON
        })
        .then((res) => {
          clearInterval(this.countdown);
          this.isRunning[talkId] = false;

          this.done[talkId].minutes = Math.floor(this.currentDone / 60);
          this.done[talkId].seconds =
            this.currentDone - this.done[talkId].minutes * 60;

          this.currentDone = 0;
          this.color_timer = "white";

          this.negative = false;
          this.readyNegative = false;

          this.minutesRemaining = 0;
          this.secondsRemaining = 0;
        })
        .catch((err) => {
          console.error(err);
        });
    },
    adaptTimer(time) {
      this.minutesRemaining = Math.floor(time / 60);
      this.secondsRemaining = time - this.minutesRemaining * 60;
    },
    initCountdown(talkId) {
      this.countdown = setInterval(() => {
        this.verifyCountdownRunning(talkId);

        if (
          (this.secondsRemaining === 0 && !this.negative) ||
          (this.secondsRemaining === 59 && this.negative)
        ) {
          this.minutesRemaining--;

          if (this.minutesRemaining < 3 && !this.negative) {
            this.color_timer = "yellow";
          }

          if (!this.readyNegative && this.minutesRemaining == -1) {
            this.color_timer = "red";
            this.minutesRemaining = 0;
            this.negative = true;
            this.readyNegative = true;
          } else {
            this.secondsRemaining = this.negative ? -1 : 60;
          }
        }

        if (!this.negative) {
          this.secondsRemaining--;
        } else {
          this.secondsRemaining++;
        }

        this.currentDone++;
      }, 1000);
    },
    verifyCountdownRunning(talkId) {
      this.$axios
        .get(`http://${this.host}:${this.port}/api/v1/timers`)
        .then((res) => {
          if (!res.data.status.isRunning) {
            clearInterval(this.countdown);
            this.isRunning[talkId] = false;
            this.minutesRemaining = 0;
            this.secondsRemaining = 0;
          }
        })
        .catch((err) => {
          console.error(err);
        });
    },
    fillRunning() {
      this.talks.forEach((talk) => {
        this.isRunning[talk.talkId] = false;
      });
    },
    fillDone() {
      this.talks.forEach((talk) => {
        this.done[talk.talkId] = { minutes: 0, seconds: 0 };
      });
    },
    verifyCountdownActive() {
      for (let key in this.isRunning) {
        if (this.isRunning[key]) {
          return key;
        }
      }
      return null;
    },
    convertMinutes(secs) {
      return secs / 60;
    },
    verifyTalkIsRunning(status) {
      if (status.isRunning) {
        this.isRunning[status.talkId] = true;
        let [hour, minutes, seconds] = status.timeElapsed
          .split(".")[0]
          .split(":");

        this.currentDone = minutes * 60 + seconds;
        this.adaptTimer(status.targetSeconds - this.currentDone);
        this.initCountdown(status.talkId);
      }
    },
    verifyTalkDones() {
      this.talks.forEach((talk) => {
        this.done[talk.talkId].minutes = Math.floor(
          talk.completedTimeSecs / 60
        );
        this.done[talk.talkId].seconds =
          talk.completedTimeSecs - this.done[talk.talkId].minutes * 60;
      });
    },
  },
});
</script>
