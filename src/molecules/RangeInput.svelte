<script>
  export let range = [0, 100];
  export let disabled;
  export let onChange;
  export let name;
  export let step = 1;
  export let defaultValue = range[0];
  export let skippedValues = [];

  $: min = Math.min.apply(null, range);
  $: max = Math.max.apply(null, range);
  $: diff = max - min;
  $: precision = Math.abs(Math.min(0, +step.toExponential().split('e')[1]));
  $: value = Math.min(Math.max(defaultValue, min), max);

  let timeout,
    interval,
    showControls = false;

  function increment() {
    if (value + step <= max) {
      do {
        value = +(value + step).toFixed(precision);
      } while (skippedValues.includes(value));
    } else {
      clearTimers();
    }
  }

  function decrement() {
    if (value - step >= min) {
      do {
        value = +(value - step).toFixed(precision);
      } while (skippedValues.includes(value));
    } else {
      clearTimers();
    }
  }

  function stickyCall(fn) {
    fn();
    timeout = setTimeout(() => {
      fn();
      interval = setInterval(fn, 50);
    }, 500);
  }

  function clearTimers() {
    clearTimeout(timeout);
    clearInterval(interval);
  }

  function pressIncrement(e) {
    stickyCall(increment);
    e.target.setPointerCapture(e.pointerId);
  }

  function pressDecrement(e) {
    stickyCall(decrement);
    e.target.setPointerCapture(e.pointerId);
  }

  function release(e) {
    clearTimers();
    e.target.releasePointerCapture(e.pointerId);
    onChange(value, name);
  }
</script>

<span class="input-wrapper" class:disabled>
  <button
    disabled={value <= min || disabled}
    class="decrementer"
    on:pointerdown={pressDecrement}
    on:pointercancel={release}
    on:pointerup={release}>
    <span>-</span>
  </button>
  <span class="input">{value.toFixed(precision)}</span>
  <button
    disabled={value >= max || disabled}
    class="incrementer"
    on:pointerdown={pressIncrement}
    on:pointercancel={release}
    on:pointerup={release}>
    <span>+</span>
  </button>
</span>

<style>
  .input-wrapper {
    width: 14rem;
    border-radius: 4px;
    border: 1px solid var(--corporate-blue);
    height: 3.2rem;
    line-height: 3.2rem;
    display: flex;
    overflow: hidden;
  }
  .input-wrapper.disabled {
    opacity: 0.6;
  }
  .input {
    flex-grow: 1;
    padding: 0 1rem;
    border: none;
    font-size: 2rem;
    text-align: center;
    display: inline-block;
  }
  button {
    border: none;
    background-color: transparent;
    width: 3.2rem;
    font-size: 2.4rem;
    line-height: 3.2rem;
    font-weight: 300;
    outline: none;
    background-color: var(--corporate-blue);
    color: var(--bg-color);
  }
  button:focus {
    outline: none;
  }
  button:disabled {
    opacity: 0.5;
  }
</style>
