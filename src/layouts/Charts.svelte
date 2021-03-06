<script>
  import Button from '../atoms/Button';
  import RadioGroup from '../molecules/RadioGroup';
  import SaveButton from '../organisms/SaveButton';
  import { data } from '../stores';
  import { ipcRenderer } from 'electron';
  import { PELTIER_PARAMS } from '../constants';
  import Chart from 'chart.js';
  import 'chartjs-plugin-zoom';
  import { onMount, onDestroy } from 'svelte';
  import configureChart from './chart.config';
  import PointsStorage from '../utils/PointsStorage';
  import { __ } from '../utils/translations';
  export let goBack;

  const xCaption = 'T, ' + PELTIER_PARAMS.temperatureHot.units;
  const pointsStorage = new PointsStorage();

  let logId,
    chart,
    unsubscribeData,
    timeStart,
    isDrawing;

  onMount(createChart);
  onDestroy(() => chart && chart.destroy());

  function createChart() {
    chart = new Chart(
      document.getElementById('measures-chart').getContext('2d'),
      configureChart(pointsStorage.points, {
        x: xCaption,
        y: selectedSensor.caption,
      })
    );
    chart.options.onClick = chart.resetZoom;
  }

  const faceOptions = {
    name: 'face',
    elements: [
      { label: '', value: 'Hot', icon: 'hot' },
      { label: '', value: 'Cool', icon: 'cool' },
    ],
  };

  const sensorsOptions = {
    name: 'sensors',
    elements: [
      {
        value: 0,
        label: 'thermoresistor',
        name: 'thermoresistor',
        caption:
          $data.thermoresistorCool.symbol +
          ', ' +
          $data.thermoresistorCool.units,
      },
      {
        value: 1,
        label: 'thermocouple',
        name: 'thermocouple',
        caption:
          $data.thermocoupleCool.symbol + ', ' + $data.thermocoupleCool.units,
      },
      {
        value: 2,
        label: 'thermistor',
        name: 'thermistor',
        caption:
          $data.thermistorCool.symbol + ', ' + $data.thermistorCool.units,
      },
    ],
  };

  const storedValues = faceOptions.elements
    .map(face =>
      ['temperature' + face.value].concat(
        sensorsOptions.elements.map(sensor => sensor.name + face.value)
      )
    )
    .flat();

  let selectedFace = 'Cool',
    selectedSensor = sensorsOptions.elements[0];

  function selectFace(e) {
    selectedFace = e.target.value;
    updateXAxis();
  }

  function selectSensor(e) {
    selectedSensor = sensorsOptions.elements[e.target.value];
    updateYAxis();
  }

  function toggleDrawing() {
    if (isDrawing) stopDrawing();
    else startDrawing();
  }

  function stopDrawing() {
    unsubscribeData();
    pointsStorage.drain();
    isDrawing = false;
  }

  function updateXAxis() {
    pointsStorage.setX(storedValues.indexOf('temperature' + selectedFace));
    pointsStorage.setY(
      storedValues.indexOf(selectedSensor.name + selectedFace)
    );
    chart.data.datasets[0].data = pointsStorage.points;
    chart.update();
  }

  function updateYAxis() {
    if (!chart) return;
    pointsStorage.setY(
      storedValues.indexOf(selectedSensor.name + selectedFace)
    );
    chart.data.datasets[0].data = pointsStorage.points;
    chart.options.scales.yAxes[0].scaleLabel.labelString =
      selectedSensor.caption;
    chart.update();
  }

  function startDrawing() {
    isDrawing = true;
    timeStart = 0;
    startLogging();
    unsubscribeData = data.subscribe(addRow);
  }

  function startLogging() {
    logId = `Thermo-Electricity-Sensors`,
    ipcRenderer.send(
      'createFile',
      logId,
      [`${$__('time')}, ${$__('s')}`].concat(
        storedValues.map(
          key =>
            `${PELTIER_PARAMS[key].symbol}(${$__(key.endsWith('Cool') ? 'cold' : 'hot')}), ${
              $__(PELTIER_PARAMS[key].units)
            }`
        )
      )
    );
  }

  function addRow(data) {
    const row = storedValues.map(key => data[key].value);
    pointsStorage.addRow(row);
    writeExcel([timeStart++].concat(row));
    updateChartData();
  }

  function writeExcel(row) {
    ipcRenderer.send('excelRow', logId, row);
  }

  function updateChartData() {
    chart.data.datasets[0].data = pointsStorage.points;
    chart.update();
  }
</script>

<div class="layout">
  <header>{$__('charts')}</header>
  <main>
    <div class="selects">
      <RadioGroup
        group={faceOptions}
        on:change={selectFace}
        type="horizontal" />
      <RadioGroup
        group={sensorsOptions}
        on:change={selectSensor}
        type="horizontal" />
    </div>
    <div class="chart">
      <canvas id="measures-chart" height="120" />
    </div>
  </main>
  <footer>
    <Button on:click={goBack}>{$__('back')}</Button>
    <SaveButton  {logId} />
    <Button style="width:8rem" on:click={toggleDrawing}>
      {isDrawing ? $__('stop') : $__('start')}
    </Button>
  </footer>
</div>

<style>
  main {
    padding: 0 6.4rem;
  }
  .selects {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: var(--gutter-width);
  }
  canvas {
    width: 100%;
    height: 100%;
  }
  footer {
    padding: var(--gutter-width) 6.4rem;
  }
</style>
