<script>
import { onMount } from 'svelte'
import _ from 'lodash'
import Chart from 'svelte-frappe-charts'

const defaultFreqThreshold = 150
const defaultTimeThreshold = 500
const defaultAmpThreshold = -133


const autoAmpThresholdOffset = 15
const ampHistoryBufferLength = 20
const ampHistoryUpdateDelay= 200
const ampAvgUpdateDelay = 100
const freqValidMax = 250
const alarmLength = 600

let trainingMode = "lower"
let frequency = 0
let rawFrequency = 0
let ampThreshold = defaultAmpThreshold
let freqThreshold = defaultFreqThreshold
let timeThreshold = defaultTimeThreshold
let lastDetectedCross = null
let lastDetectedCorrect = null
let avgAmp = 0
let movingAmpAvg = 0
let initialMovingAmpAvg = 0
let loadedFromSavedState = false
let ampHistory = []
let freqHistory = []
let frequencyData = null
let isTriggered = false
let chartData = {labels: [], datasets: [
    {values: []}
]}

let tuner = null
let frequencyBars = null

onMount(() => {
  loadState()
  initTrainer()
  updateChartData()
  window.addEventListener("unload", saveState);
  return saveState
})

function loadState() {
  const savedState = window.localStorage.getItem('trainingConfig')
  try {
    if (savedState) {
      let state = JSON.parse(savedState)
      ampThreshold = state.ampThreshold
      timeThreshold = state.timeThreshold
      freqThreshold = state.freqThreshold
      trainingMode = state.trainingMode
      loadedFromSavedState = true
    }
  } catch (e) {
    console.error('Could not load state', e)
  }
}
function saveState() {
  window.localStorage.setItem('trainingConfig', JSON.stringify({
    ampThreshold,
    timeThreshold,
    freqThreshold,
    trainingMode
  }))
}

function initTrainer() {
  tuner = new Tuner()
  frequencyBars = new FrequencyBars('.frequency-bars')

  tuner.onFrequencyRaw = onFrequencyRaw

  swal.fire('Welcome to Transpeak', 'Transpeak helps you train your voice to be lower than higher than a specified pitch using your microphone. Click OK so we can request permission to access your microphone. Sound from your microphone is not transmitted to any remote servers').then(function() {
    tuner.init()
    frequencyData = new Uint8Array(tuner.analyser.frequencyBinCount)
  })

  updateFrequencyBars()
}
function onFrequencyRaw(detectedFrequency) {
    rawFrequency = detectedFrequency
  if (avgAmp == 0 || detectedFrequency == 0 || detectedFrequency > freqValidMax) {
    return
  }

  frequency = detectedFrequency

  movingAmpAvg = ampHistory.reduce( (o,i) => o + i, 0) / ampHistory.length

  if ((ampHistory.length == ampHistoryBufferLength) && (initialMovingAmpAvg == 0)) {
    initialMovingAmpAvg = parseInt(movingAmpAvg)
    if (!loadedFromSavedState) {
      ampThreshold = initialMovingAmpAvg + autoAmpThresholdOffset
    }
  }

  if (avgAmp == 0 || frequency == 0 || frequency > freqValidMax || initialMovingAmpAvg === 0) {
    return
  }




  const isAmpOverThreshold = avgAmp > Math.max(movingAmpAvg, ampThreshold)

  let isOutOfRange = (trainingMode === 'lower') ? frequency < freqThreshold : frequency > freqThreshold

  if (isOutOfRange) {
        if (!lastDetectedCross) {
          if (isAmpOverThreshold) {
            lastDetectedCross = Date.now()
          }
        } else {
          const timeSinceCross = Date.now() - lastDetectedCross
          if (timeSinceCross > timeThreshold && isAmpOverThreshold) {
            lastDetectedCross = null
            isTriggered = true
            tuner.play(freqThreshold)
            setTimeout(() => {
              tuner.stop()
            }, alarmLength)
          }
        }
      } else {
      // we ignore the amp threshold 
        if (!lastDetectedCorrect) {
            lastDetectedCorrect= Date.now()
        } else {
          const timeSinceCorrect = Date.now() - lastDetectedCorrect
          if (timeSinceCorrect > (timeThreshold / 2)) {
            lastDetectedCross = null
            lastDetectedCorrect = null
            isTriggered = false
            tuner.stop()
          }
        }
      }
}


const updateAmpHistory = _.throttle(function() {
  if (!isFinite(avgAmp)) {
    return
  }
  if (ampHistory.length == ampHistoryBufferLength) {
    ampHistory = [...ampHistory.slice(1), avgAmp]
  } else {
    ampHistory.push(avgAmp)
  }
    if (freqHistory.length == ampHistoryBufferLength) {
        freqHistory = [...freqHistory.slice(1), frequency]
    } else {
      freqHistory.push(frequency)
    }
  updateChartData()
}, ampHistoryUpdateDelay, { leading: true})

const updateFrequencyBars = _.throttle(function() {
  if (tuner.analyser) {
    tuner.analyser.getByteFrequencyData(frequencyData)
    var floatFrequencies = new Float32Array(tuner.analyser.frequencyBinCount);
    tuner.analyser.getFloatFrequencyData(floatFrequencies)
    const avg = floatFrequencies.reduce((o,i) => o + i, 0) / floatFrequencies.length.toFixed(2)
    avgAmp = avg
    updateAmpHistory()
    let color
    if (isTriggered) {
      color = '#ff0000'
    } else {
      if (lastDetectedCross) {
        color = '#ffc600'
      } else {
        color = '#ecf0f1'
      }
    }
    frequencyBars.update(frequencyData, color)
  }
  requestAnimationFrame(updateFrequencyBars)
}, ampAvgUpdateDelay, { leading: true})

const guessThreshold = (e) => {
  e.preventDefault()
  ampThreshold = parseInt(movingAmpAvg) + autoAmpThresholdOffset
}
const reset = () => {
    freqThreshold = defaultFreqThreshold
    timeThreshold = defaultTimeThreshold
    ampThreshold = defaultAmpThreshold
}

const updateChartData = () => {
    let nextData = {labels: [], 
        colors: ['green', 'blue'],
        datasets: [
        {name: "frequency", values: []}
    ]}
    ampHistory.forEach( (item, index) => {
        nextData.labels.push(index)
        nextData.datasets[0].values.push(freqHistory[index])
    })
    chartData = nextData
}


$: isAmpOverThreshold = avgAmp > Math.max(movingAmpAvg, ampThreshold)
</script>
<style>
.frequency-bars {
  position: fixed;
  left: 0px;
  right: 0px;
  bottom: 0;
  z-index: -1;
}
.controlGroupLabel {
    display: flex;
    flex-direction: row;
    align-items: center;

}
.frappe-chart .legend {
    display: none;
}
.controlGroupLabel input[inputmode="numeric"] {
    margin-left: 2em;
    width: 50px;
}
.controlGroupLabel .inputSuffix {
    margin-left: 2em;
    height: 5ex;
}

.btclink {
  font-size: 10pt;
}
</style>
<em>Transpeak</em><br/>
<p>Train your voice to stay above or below a target frequency in real-time</p>
<canvas class="frequency-bars"></canvas>
<b>Current detected readings</b>
<table>
  <tr>
    <th>Frequency</th><th>Volume</th>
  </tr>
  <tr>
    <td>
      {frequency.toFixed(0)} Hz
    </td>
    <td>
      Current: {avgAmp.toFixed(0)} dB<br/>
      Avg: ({(ampAvgUpdateDelay * ampHistoryBufferLength)/1000}s) {movingAmpAvg.toFixed(0)} dB
    </td>
  </tr>
</table>
<Chart data={chartData} type="line" axisOptions={{xAxisMode: 'tick', xIsSeries: 1}} lineOptions={{hideDots: 1}} />
<form>
  <fieldset>
    <label for="mode">Training Mode</label>
    <select bind:value={trainingMode} name="mode">
      <option value="lower">Lower</option>
      <option value="higher">Higher</option>
    </select>
    <label for="thresholdFrequency">Frequency</label>
    <div class="controlGroupLabel">
        <input bind:value={freqThreshold} type="range" step="10" name="thresholdFrequency" min="70" max="200"/>
        <input inputmode="numeric" bind:value={freqThreshold} pattern="\d*" />
        <div class="inputSuffix">
            hz
        </div>
    </div>
    <p>
    This is the frequency where we will sound the alarm and play the reference tone if the detected pitch is {trainingMode}
    </p>
    <label for="ampThreshold">Amp speaking volume threshold ({ampThreshold} db)</label>
    <div class="controlGroupLabel">
        <input bind:value={ampThreshold} name="ampThreshold" type="range" step="5" min="-150" max="-50"/>
        <input inputmode="numeric" bind:value={ampThreshold} pattern="\d*" />
        <div class="inputSuffix">
        db
        </div>
    </div>
    <p>
      Do not count frequencies whose volume is lower than this number. Make this number higher to decrease sensitivity.
      <button class="button button-outline" on:click={guessThreshold}>Guess Threshold</button>
    </p>
        <label name="timeThreshold">Time delay threshold: ({timeThreshold} ms)</label>
    <div class="controlGroupLabel">
        <input bind:value={timeThreshold} name="timeThreshold" type="range" step="100" min="100" max="2000"/>
        <input inputmode="numeric" bind:value={timeThreshold} pattern="\d*" />
        <div class="inputSuffix">
            ms
            </div>
    </div>
    <p>
    When an out-of-range pitch is detected, this is how long we should continuously detect this pitch before triggering the alarm. Raise this to decrease sensitivity.
    </p>
  </fieldset>
</form>
<button type="button button-outline" on:click={reset}>Reset to defaults</button>
<pre id="output">
lastDetectedCross: {lastDetectedCross}
ampHistory: {ampHistory.join(',')}
</pre>
Made with 💖 by <a href="http://twitter.com/tastycode">@tastycode</a>.
<p>
To support this project, please send BTC to <br/>
<a class="btclink" href="bitcoin:bc1qnm44qzsnwyrl3uynywv4n58jy2tmtkd5wpjwp0">bc1qnm44qzsnwyrl3uynywv4n58jy2tmtkd5wpjwp0</a>
<br/>@tastycode on venmo/cashapp<br/>
</p>


Shortlink: <a href="//bit.ly/transpeak">bit.ly/transpeak</a>



