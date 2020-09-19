<script>
import { onMount } from 'svelte'
import _ from 'lodash'

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
let frequencyData = null
let isTriggered = false

let tuner = null
let frequencyBars = null

onMount(() => {
  loadState()
  initTrainer()
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

  swal.fire('Click OK to continue').then(function() {
    tuner.init()
    frequencyData = new Uint8Array(tuner.analyser.frequencyBinCount)
  })

  updateFrequencyBars()
}
function onFrequencyRaw(detectedFrequency) {
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
</style>
<em>Transpeak</em><br/>
<p>Train your voice to stay above or below a target frequency in real-time</p>
<canvas class="frequency-bars"></canvas>
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
<form>
  <fieldset>
    <label for="mode">Training Mode</label>
    <select bind:value={trainingMode} name="mode">
      <option value="lower">Lower</option>
      <option value="higher">Higher</option>
    </select>
    <label for="thresholdFrequency">Frequency ({freqThreshold} hz)</label>
    <input bind:value={freqThreshold} type="range" step="10" name="thresholdFrequency" min="70" max="200"/>
    <p>
    This is the frequency where we will sound the alarm and play the reference tone if the detected pitch is {trainingMode}
    </p>
    <label for="ampThreshold">Amp speaking volume threshold ({ampThreshold} db)</label>
    <input bind:value={ampThreshold} name="ampThreshold" type="range" step="5" min="-150" max="-50"/>
    <p>
      Do not count frequencies whose volume is lower than this number. Make this number higher to decrease sensitivity.
      <a href="#" on:click={guessThreshold}>Guess threshold</a>
    </p>
    <label name="timeThreshold">Time delay threshold: ({timeThreshold} ms)</label>
    <input bind:value={timeThreshold} name="timeThreshold" type="range" step="100" min="100" max="2000"/>
    <p>
    When an out-of-range pitch is detected, this is how long we should continuously detect this pitch before triggering the alarm. Raise this to decrease sensitivity.
    </p>
  </fieldset>
</form>
<pre id="output">
lastDetectedCross: {lastDetectedCross}
ampHistory: {ampHistory.join(',')}
</pre>
Made with 💖 by <a href="http://twitter.com/tastycode">@tastycode</a>. To support this project, please send BTC to <a href="bitcoin:bc1qnm44qzsnwyrl3uynywv4n58jy2tmtkd5wpjwp0">bc1qnm44qzsnwyrl3uynywv4n58jy2tmtkd5wpjwp0</a>. Or venmo/cashapp @tastycode<br/>
Shortlink: <a href="//bit.ly/transpeak">bit.ly/transpeak</a>



