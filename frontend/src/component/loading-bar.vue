<template>
  <transition id="p-loading-bar" :css="false" @before-enter="beforeEnter" @enter="enter" @after-enter="afterEnter">
    <div v-if="visible" class="top-progress" :style="barStyle">
      <div class="peg" :style="pegStyle"></div>
    </div>
  </transition>
</template>
<script>
function clamp(n, min, max) {
  if (n < min) {
    return min;
  }
  if (n > max) {
    return max;
  }
  return n;
}

let queue = (() => {
  let pending = [];

  function next() {
    let fn = pending.shift();

    if (fn) {
      fn(next);
    }
  }

  return (fn) => {
    pending.push(fn);

    if (pending.length === 1) {
      next();
    }
  };
})();

export default {
  name: "PLoadingBar",

  props: {
    speed: {
      type: Number,
      default: 350,
    },

    color: {
      type: String,
      default: "#29d",
    },

    errorColor: {
      type: String,
      default: "#f44336",
    },

    trickle: {
      type: Boolean,
      default: true,
    },

    trickleSpeed: {
      type: Number,
      default: 250,
    },

    easing: {
      type: String,
      default: "linear",
    },

    height: {
      type: [Number, String],
      default: 3,
    },

    minimum: {
      type: Number,
      default: 0.8,
    },

    maximum: {
      type: Number,
      default: 97.5,
    },

    zIndex: {
      type: Number,
      default: 9999,
    },
  },
  data() {
    return {
      visible: false,
      error: false,
      progress: 0,
      opacity: 1,
      status: null,
      isPaused: false,
    };
  },

  computed: {
    progressColor() {
      return this.error ? this.errorColor : this.color;
    },

    isStarted() {
      return typeof this.status === "number";
    },

    barStyle() {
      return {
        position: "fixed",
        top: "0",
        left: "0",
        right: "0",
        width: `${this.progress}%`,
        height: `${this.height}px`,
        backgroundColor: this.progressColor,
        transition: `all ${this.speed}ms ${this.easing}`,
        opacity: `${this.opacity}`,
        zIndex: `${this.zIndex}`,
      };
    },

    pegStyle() {
      return {
        display: "block",
        position: "absolute",
        right: "0",
        width: "100px",
        height: "100%",
        opacity: this.progress ? "1" : "0",
        boxShadow: `0 0 10px ${this.progressColor}, 0 0 5px ${this.progressColor}`,
        transform: "rotate(3deg) translate(0px, -4px)",
      };
    },
  },

  mounted() {
    let stackSize = 0;

    this.$event.subscribe(
      "ajax.start",
      function () {
        stackSize++;

        if (stackSize === 1) {
          this.start();
        }
      }.bind(this)
    );

    this.$event.subscribe(
      "ajax.end",
      function () {
        stackSize--;

        if (stackSize === 0) {
          this.done();
        }
      }.bind(this)
    );
  },

  methods: {
    beforeEnter(el) {
      this.opacity = 0;
      this.progress = 0;
      this.width = 0;
    },

    enter(el, done) {
      this.opacity = 1;
      done();
    },

    afterEnter(el) {
      this._runStart();
    },

    _work() {
      setTimeout(() => {
        if (!this.isStarted || this.isPaused) {
          return;
        }
        this.increase();
        this._work();
      }, this.trickleSpeed);
    },

    _runStart() {
      this.status = this.progress === 100 ? null : this.progress;

      if (this.trickle) {
        this._work();
      }
    },

    start() {
      this.isPaused = false;

      if (this.visible) {
        this._runStart();
      } else {
        this.visible = true;
      }
    },

    set(amount) {
      this.isPaused = false;

      let o;
      if (this.isStarted) {
        o = amount < this.progress ? clamp(amount, 0, 100) : clamp(amount, this.minimum, 100);
      } else {
        o = 0;
      }

      this.status = o === 100 ? null : o;

      queue((next) => {
        this.progress = o;
        if (o === 100) {
          setTimeout(() => {
            this.opacity = 0;
            setTimeout(() => {
              this.visible = false;
              this.error = false;
              next();
            }, this.speed);
          }, this.speed);
        } else {
          setTimeout(next, this.speed);
        }
      });
    },

    increase(amount) {
      let o = this.progress;

      if (o < 100 && typeof amount !== "number") {
        if (o >= 0 && o < 25) {
          amount = Math.random() * 3 + 3;
        } else if (o >= 25 && o < 50) {
          amount = Math.random() * 3;
        } else if (o >= 50 && o < 85) {
          amount = Math.random() * 2;
        } else if (o >= 85 && o < 99) {
          amount = 0.5;
        } else {
          amount = 0;
        }
      }
      this.set(clamp(o + amount, 0, this.maximum));
    },

    decrease(amount) {
      if (this.progress === 0) {
        return;
      }
      this.increase(-amount);
    },

    done() {
      this.set(100);
    },

    getProgress() {
      return this.status ? this.progress : 0;
    },

    pause() {
      this.isPaused = true;
    },

    fail() {
      this.error = true;
      this.done();
    },
  },
};
</script>
