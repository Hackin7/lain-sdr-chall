<script>
  import { onDestroy, onMount } from 'svelte'
  import { describeDevice, requestHfsdrDevice } from './lib/webusb.js'
  import {
    parseFrequencyInput,
    parseTlv320GainInput,
    readClockFrequency as readDeviceClockFrequency,
    readPllLock as readDevicePllLock,
    setClockFrequency as writeClockFrequency,
    setTlv320Gain as writeTlv320Gain,
    tlv320GainDbToRaw,
    TLV320_GAIN_MAX_DB,
    TLV320_GAIN_MIN_DB,
    TLV320_GAIN_STEP_DB,
  } from './lib/control.js'
  import { startVendorIqStream } from './lib/data.js'
  import { createFmAudioPlayer } from './lib/fm-audio.js'
  import {
    createSpectrogramRenderer,
    FFT_ROW_INTERVAL,
    FFT_SIZE,
    IQ_SAMPLE_RATE_HZ,
  } from './lib/spectrogram.js'

  const READY_STATUS = 1
  const ROW_RATE_HZ = IQ_SAMPLE_RATE_HZ / FFT_ROW_INTERVAL
  const BIN_WIDTH_HZ = IQ_SAMPLE_RATE_HZ / FFT_SIZE
  const HALF_BANDWIDTH_KHZ = IQ_SAMPLE_RATE_HZ / 2000
  const QUARTER_BANDWIDTH_KHZ = IQ_SAMPLE_RATE_HZ / 4000
  const TLV320_GAIN_DEBOUNCE_MS = 100
  const PLL_POLL_INTERVAL_MS = 200
  const FREQUENCY_LABELS = [
    `-${HALF_BANDWIDTH_KHZ.toFixed(0)} kHz`,
    `-${QUARTER_BANDWIDTH_KHZ.toFixed(0)} kHz`,
    'DC',
    `+${QUARTER_BANDWIDTH_KHZ.toFixed(0)} kHz`,
    `+${HALF_BANDWIDTH_KHZ.toFixed(0)} kHz`,
  ]

  let device = null
  let frequency = ''
  let tlv320Gain = 0
  let pllLocked = null
  let deviceLabel = 'No device paired'
  let statusMessage = 'Pair a WebUSB device to read or set the clock frequency.'
  let streamMessage = 'Pair a device to start the live I/Q spectrogram.'
  let isConnecting = false
  let isReading = false
  let isSetting = false
  let isSettingGain = false
  let streamStats = null
  let streamHandle = null
  let spectrogramCanvas = null
  let spectrogramRenderer = null
  let historySeconds = 0
  let pendingTlv320Gain = null
  let tlv320GainDebounceTimer = null
  let tlv320GainFlushRequested = false
  let lastTlv320GainSentAt = 0
  let fmAudioPlayer = null
  let isAudioStarting = false
  let isAudioPlaying = false
  let audioMessage = 'Pair a device, then start FM audio playback.'
  let hoverFrequencyInfo = null
  let pllPollTimer = null
  let pllPollGeneration = 0
  let pllPollInFlight = false

  function stopIqStream(message = 'Stream stopped.') {
    if (streamHandle) {
      streamHandle.stop()
      streamHandle = null
    }

    streamStats = null
    streamMessage = message
    fmAudioPlayer?.reset()

    if (isAudioPlaying) {
      audioMessage = 'FM audio is running and waiting for live I/Q samples.'
    }
  }

  function startIqStream(selectedDevice) {
    stopIqStream('Waiting for incoming I/Q samples...')
    spectrogramRenderer?.clear()

    streamHandle = startVendorIqStream(selectedDevice, {
      onIqSamples(iqSamples) {
        spectrogramRenderer?.pushIqSamples(iqSamples)
        fmAudioPlayer?.pushIqSamples(iqSamples)
      },
      onStatus(nextStats) {
        streamStats = nextStats
        streamMessage = `Streaming ${Math.round(nextStats.framesPerSecond).toLocaleString()} complex samples/s at ${nextStats.mbPerSecond.toFixed(3)} MB/s.`
      },
    })

    streamHandle.done.catch((error) => {
      if (error?.name === 'AbortError') {
        return
      }

      console.error('Vendor I/Q stream stopped:', error)
      streamMessage = error?.message || 'I/Q stream stopped unexpectedly.'
    })
  }

  function stopPllLockPolling() {
    pllPollGeneration += 1
    pllPollInFlight = false

    if (pllPollTimer !== null) {
      clearInterval(pllPollTimer)
      pllPollTimer = null
    }
  }

  async function pollPllLock(selectedDevice = device, generation = pllPollGeneration) {
    if (!selectedDevice || pllPollInFlight || generation !== pllPollGeneration) {
      return
    }

    pllPollInFlight = true

    try {
      const { status, locked } = await readDevicePllLock(selectedDevice)

      if (generation !== pllPollGeneration || selectedDevice !== device) {
        return
      }

      pllLocked = status === READY_STATUS ? locked : null
    } catch {
      if (generation !== pllPollGeneration || selectedDevice !== device) {
        return
      }

      pllLocked = null
    } finally {
      if (generation === pllPollGeneration) {
        pllPollInFlight = false
      }
    }
  }

  function startPllLockPolling(selectedDevice = device) {
    stopPllLockPolling()

    if (!selectedDevice) {
      pllLocked = null
      return
    }

    const generation = pllPollGeneration
    pllPollTimer = setInterval(() => {
      void pollPllLock(selectedDevice, generation)
    }, PLL_POLL_INTERVAL_MS)
    void pollPllLock(selectedDevice, generation)
  }

  async function startFmAudio() {
    if (!device) {
      audioMessage = 'Pair a device before starting FM audio.'
      return
    }

    isAudioStarting = true

    try {
      if (!fmAudioPlayer) {
        fmAudioPlayer = await createFmAudioPlayer({
          iqSampleRateHz: IQ_SAMPLE_RATE_HZ,
        })
      }

      fmAudioPlayer.reset()
      await fmAudioPlayer.start()
      isAudioPlaying = true
      audioMessage = `Playing FM-demodulated audio at ${(fmAudioPlayer.audioSampleRateHz / 1000).toFixed(0)} kHz.`
    } catch (error) {
      isAudioPlaying = false
      audioMessage = error?.message || 'Failed to start FM audio playback.'
    } finally {
      isAudioStarting = false
    }
  }

  async function stopFmAudio() {
    if (!fmAudioPlayer) {
      isAudioPlaying = false
      audioMessage = 'FM audio stopped.'
      return
    }

    try {
      await fmAudioPlayer.stop()
    } catch (error) {
      audioMessage = error?.message || 'Failed to stop FM audio playback cleanly.'
    } finally {
      isAudioPlaying = false
    }

    if (audioMessage.startsWith('Playing FM-demodulated audio')) {
      audioMessage = 'FM audio stopped.'
    }
  }

  async function toggleFmAudio() {
    if (isAudioPlaying) {
      await stopFmAudio()
      return
    }

    await startFmAudio()
  }

  async function closeFmAudioPlayer() {
    if (!fmAudioPlayer) {
      isAudioPlaying = false
      isAudioStarting = false
      return
    }

    const player = fmAudioPlayer
    fmAudioPlayer = null
    isAudioPlaying = false
    isAudioStarting = false

    try {
      await player.close()
    } catch (error) {
      console.error('Failed to close FM audio player:', error)
    }
  }
  async function readClockFrequency(selectedDevice = device) {
    if (!selectedDevice) {
      throw new Error('No device paired.')
    }

    isReading = true

    try {
      const [{ status, frequencyHz }, { status: pllStatus, locked }] = await Promise.all([
        readDeviceClockFrequency(selectedDevice),
        readDevicePllLock(selectedDevice),
      ])

      frequency = frequencyHz.toString()
      pllLocked = pllStatus === READY_STATUS ? locked : null
      statusMessage =
        status === READY_STATUS
          ? `Current device frequency: ${frequency} Hz`
          : `Device reported status ${status} while returning ${frequency} Hz`
    } finally {
      isReading = false
    }
  }

  async function pairDevice() {
    isConnecting = true
    pllLocked = null
    stopPllLockPolling()
    stopIqStream('Pairing device...')

    try {
      const selectedDevice = await requestHfsdrDevice()

      device = selectedDevice
      deviceLabel = describeDevice(selectedDevice)
      statusMessage = 'Device paired. Reading current frequency...'

      await readClockFrequency(selectedDevice)
      //startPllLockPolling(selectedDevice)
      startIqStream(selectedDevice)
    } catch (error) {
      pllLocked = null
      statusMessage = error?.message || 'Failed to pair with the WebUSB device.'
      streamMessage = 'Pair a device to start the live I/Q spectrogram.'
    } finally {
      isConnecting = false
    }
  }

  async function setClockFrequency() {
    if (!device) {
      statusMessage = 'Pair a device before setting the frequency.'
      return
    }

    isSetting = true

    try {
      const parsedFrequency = parseFrequencyInput(frequency)
      await writeClockFrequency(device, parsedFrequency)

      statusMessage = `Frequency update sent: ${parsedFrequency.toString()} Hz. Refreshing...`
      await readClockFrequency(device)
    } catch (error) {
      statusMessage = error?.message || 'Failed to set the frequency.'
    } finally {
      isSetting = false
    }
  }

  function clearTlv320GainDebounceTimer() {
    if (tlv320GainDebounceTimer !== null) {
      clearTimeout(tlv320GainDebounceTimer)
      tlv320GainDebounceTimer = null
    }
  }

  function scheduleTlv320GainSend(delayMs) {
    clearTlv320GainDebounceTimer()
    tlv320GainDebounceTimer = setTimeout(() => {
      tlv320GainDebounceTimer = null
      void flushTlv320Gain()
    }, delayMs)
  }

  async function flushTlv320Gain(forceImmediate = false) {
    if (!device || pendingTlv320Gain === null) {
      return
    }

    if (isSettingGain) {
      if (forceImmediate) {
        tlv320GainFlushRequested = true
      }
      return
    }

    const elapsedMs = performance.now() - lastTlv320GainSentAt
    if (!forceImmediate && lastTlv320GainSentAt !== 0 && elapsedMs < TLV320_GAIN_DEBOUNCE_MS) {
      scheduleTlv320GainSend(TLV320_GAIN_DEBOUNCE_MS - elapsedMs)
      return
    }

    const gainToSend = pendingTlv320Gain
    pendingTlv320Gain = null
    isSettingGain = true

    try {
      await writeTlv320Gain(device, gainToSend)
      const gainRaw = tlv320GainDbToRaw(gainToSend)
      statusMessage = `TLV320 gain update sent: ${gainToSend.toFixed(1)} dB (CHx_CFG1 0x${gainRaw.toString(16).padStart(2, '0').toUpperCase()})`
      lastTlv320GainSentAt = performance.now()
    } catch (error) {
      statusMessage = error?.message || 'Failed to set the TLV320 gain.'
      pendingTlv320Gain = null
      tlv320GainFlushRequested = false
    } finally {
      isSettingGain = false
    }

    if (pendingTlv320Gain !== null) {
      if (tlv320GainFlushRequested) {
        tlv320GainFlushRequested = false
        void flushTlv320Gain(true)
      } else {
        scheduleTlv320GainSend(TLV320_GAIN_DEBOUNCE_MS)
      }
    }
  }

  function queueTlv320Gain(nextGain = tlv320Gain, { forceImmediate = false } = {}) {
    const parsedGain = parseTlv320GainInput(String(nextGain))
    tlv320Gain = parsedGain
    pendingTlv320Gain = parsedGain

    if (!device) {
      statusMessage = 'Pair a device before setting the ADC gain.'
      return
    }

    if (forceImmediate) {
      tlv320GainFlushRequested = true
      clearTlv320GainDebounceTimer()
      void flushTlv320Gain(true)
      return
    }

    if (isSettingGain || tlv320GainDebounceTimer !== null) {
      return
    }

    const elapsedMs = performance.now() - lastTlv320GainSentAt
    const delayMs =
      lastTlv320GainSentAt === 0
        ? TLV320_GAIN_DEBOUNCE_MS
        : Math.max(0, TLV320_GAIN_DEBOUNCE_MS - elapsedMs)

    scheduleTlv320GainSend(delayMs)
  }

  function handleTlv320SliderInput() {
    queueTlv320Gain(tlv320Gain)
  }

  function handleTlv320GainCommit() {
    queueTlv320Gain(tlv320Gain, { forceImmediate: true })
  }

  function formatHz(value) {
    return `${value.toLocaleString()} Hz`
  }

  function getHoverTooltipTransform(anchor) {
    return anchor === 'left'
      ? 'translate(0, 0)'
      : anchor === 'right'
        ? 'translate(-100%, 0)'
        : 'translate(-50%, 0)'
  }

  function clearSpectrogramHover() {
    hoverFrequencyInfo = null
  }

  function updateSpectrogramHover(event) {
    if (!spectrogramCanvas) {
      hoverFrequencyInfo = null
      return
    }

    const normalizedFrequency = frequency.trim()
    if (!/^\d+$/.test(normalizedFrequency)) {
      hoverFrequencyInfo = null
      return
    }

    const rect = spectrogramCanvas.getBoundingClientRect()
    const relativeX = Math.min(Math.max(event.clientX - rect.left, 0), rect.width)
    const normalizedX = rect.width > 0 ? relativeX / rect.width : 0
    const offsetHz = Math.round((normalizedX - 0.5) * IQ_SAMPLE_RATE_HZ)
    const absoluteHz = BigInt(normalizedFrequency) + BigInt(offsetHz)
    const dbfs = spectrogramRenderer?.getDbfsAtNormalizedX(normalizedX) ?? null
    const anchor =
      normalizedX <= 0.18 ? 'left' : normalizedX >= 0.82 ? 'right' : 'center'

    hoverFrequencyInfo = {
      x: relativeX,
      y: Math.min(Math.max(event.clientY - rect.top, 0), rect.height),
      offsetHz,
      absoluteHz,
      dbfs,
      anchor,
    }
  }

  onMount(() => {
    if (spectrogramCanvas) {
      spectrogramRenderer = createSpectrogramRenderer(spectrogramCanvas)
    }

    const handleResize = () => spectrogramRenderer?.resize()

    window.addEventListener('resize', handleResize)

    return () => {
      window.removeEventListener('resize', handleResize)
      clearTlv320GainDebounceTimer()
      stopPllLockPolling()
      stopIqStream()
      void closeFmAudioPlayer()
    }
  })

  onDestroy(() => {
    clearTlv320GainDebounceTimer()
    stopPllLockPolling()
    stopIqStream()
    void closeFmAudioPlayer()
  })

  $: historySeconds = spectrogramCanvas ? spectrogramCanvas.height / ROW_RATE_HZ : 0
</script>

<main class="app">
  <section>
    <p>powered by hfsdr</p>
    <h1>Wireless World</h1>
    <p>
      The wired is so 2000s, let's <b>waterfall</b> the wireless environment into your brain!
      
    </p>
    <ol>
      <li>Connect & Pair a device</li>
      <li>Set the frequency & gain</li>
      <li>Pair the device again</li>
      <li>Waterfall of the wireless world should start</li>
    </ol>
  </section>

  <section class="content">
    <section>
      <h2>Complex baseband spectrum</h2>
      <p>
        {#if historySeconds > 0}
          {historySeconds.toFixed(1)} s visible history
        {:else}
          Waiting for canvas
        {/if}
      </p>
      <div class="canvas-wrap">
        <canvas
          bind:this={spectrogramCanvas}
          class="spectrogram-canvas"
          on:mousemove={updateSpectrogramHover}
          on:mouseleave={clearSpectrogramHover}
        ></canvas>

        <div class="overlay-top">
          {#each FREQUENCY_LABELS as label}
            <span>{label}</span>
          {/each}
        </div>

        <div class="overlay-bottom">{streamMessage}</div>

        {#if hoverFrequencyInfo}
          <div
            class="hover-tooltip"
            style:left={`${hoverFrequencyInfo.x}px`}
            style:top={`${Math.min(hoverFrequencyInfo.y + 18, (spectrogramCanvas?.clientHeight ?? hoverFrequencyInfo.y) - 8)}px`}
            style:transform={getHoverTooltipTransform(hoverFrequencyInfo.anchor)}
          >
            <p>{formatHz(hoverFrequencyInfo.absoluteHz)}</p>
            {#if hoverFrequencyInfo.dbfs !== null}
              <p>{hoverFrequencyInfo.dbfs.toFixed(1)} dBFS</p>
            {/if}
          </div>
        {/if}
      </div>

      <section>
        <h2>Stream telemetry</h2>
        <div class="metrics-columns">
          <ul class="metrics-list">
            <li>Throughput: {streamStats ? `${streamStats.mbPerSecond.toFixed(3)} MB/s` : 'Idle'}</li>
            <li>Complex Samples: {streamStats ? Math.round(streamStats.framesPerSecond).toLocaleString() : '0'} /s</li>
            <li>Captured: {streamStats ? `${streamStats.totalMb.toFixed(2)} MB` : '0.00 MB'}</li>
            <li>FFT Cadence: 1 row / {FFT_ROW_INTERVAL} samples</li>
          </ul>
          <ul class="metrics-list">
            <li>Sample Rate: {(IQ_SAMPLE_RATE_HZ / 1000).toFixed(0)} kS/s</li>
            <li>FFT Size: {FFT_SIZE} bins</li>
            <li>Bin Width: {BIN_WIDTH_HZ.toFixed(1)} Hz</li>
            <li>Waterfall Rate: {ROW_RATE_HZ.toFixed(1)} rows/s</li>
          </ul>
        </div>
      </section>
    </section>

    <div>
      <section>
        <h2>Device control</h2>
        <p>Paired device: {deviceLabel}</p>
        <p>
          <button
            on:click={pairDevice}
            disabled={isConnecting || isReading || isSetting || isSettingGain}
          >
            {isConnecting ? 'Pairing...' : 'Pair device'}
          </button>
        </p>

        <fieldset>
          <legend>Clock frequency</legend>
          <p>
            PLL:
            {pllLocked === true
              ? 'locked'
              : pllLocked === false
                ? 'not locked'
                : 'status unknown'}
          </p>
          <p>
            <input
              id="frequency"
              type="text"
              inputmode="numeric"
              bind:value={frequency}
              placeholder="7067333"
              disabled={!device || isConnecting || isReading || isSetting || isSettingGain}
            />
          </p>
          <p>
            <button
              on:click={setClockFrequency}
              disabled={!device || isConnecting || isReading || isSetting || isSettingGain}
            >
              {isSetting ? 'Setting...' : 'Set frequency'}
            </button>
          </p>
        </fieldset>

        <fieldset>
          <legend>ADC gain</legend>
          <p>Sends the same gain to TLV320 `CH1_CFG1` and `CH2_CFG1` in 0.5 dB steps.</p>
          <p>Current gain: {tlv320Gain.toFixed(1)} dB</p>
          <p>
            <input
              type="range"
              min={TLV320_GAIN_MIN_DB}
              max={TLV320_GAIN_MAX_DB}
              step={TLV320_GAIN_STEP_DB}
              bind:value={tlv320Gain}
              on:input={handleTlv320SliderInput}
              on:change={handleTlv320GainCommit}
              disabled={!device || isConnecting || isReading || isSetting}
            />
          </p>
          <p>
            <input
              type="number"
              min={TLV320_GAIN_MIN_DB}
              max={TLV320_GAIN_MAX_DB}
              step={TLV320_GAIN_STEP_DB}
              bind:value={tlv320Gain}
              on:change={handleTlv320GainCommit}
              disabled={!device || isConnecting || isReading || isSetting}
            />
          </p>
          <p>CHx_CFG1 = 0x{tlv320GainDbToRaw(tlv320Gain).toString(16).padStart(2, '0').toUpperCase()}</p>
        </fieldset>

        <!--<fieldset>
          <legend>FM audio</legend>
          <p>
            Demodulates the live DC-centered I/Q stream into mono audio with a phase discriminator,
            4:1 decimation, and de-emphasis.
          </p>
          <p>
            <button
              on:click={toggleFmAudio}
              disabled={!device || isConnecting || isAudioStarting}
            >
              {#if isAudioStarting}
                Starting...
              {:else if isAudioPlaying}
                Stop audio
              {:else}
                Start audio
              {/if}
            </button>
          </p>
          <p>{audioMessage}</p>
        </fieldset>-->

        <fieldset>
          <legend>Status</legend>
          <p>{statusMessage}</p>
        </fieldset>
      </section>

      
    </div>
  </section>
</main>
