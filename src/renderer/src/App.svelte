<script lang="ts">
  import { onMount, onDestroy } from 'svelte'

  // eslint-disable-next-line no-undef
  let interval: NodeJS.Timeout
  let time = new Date()

  const alarms = [
    new Date().setHours(7, 30, 0, 0),
    new Date().setHours(9, 0, 0, 0),
    new Date().setHours(10, 30, 0, 0),
    new Date().setHours(12, 0, 0, 0),
    new Date().setHours(13, 0, 0, 0),
    new Date().setHours(16, 36, 0, 0),
    new Date().setHours(16, 0, 0, 0),
    new Date().setHours(17, 30, 0, 0),
    new Date().setHours(19, 0, 0, 0)
  ].map((timestamp) => new Date(timestamp))

  let alarmSoundSet = localStorage.getItem('alarmSound') ? true : false
  let initialSoundSet = localStorage.getItem('initialSound') ? true : false

  onMount(() => {
    const savedInitialSound = localStorage.getItem('initialSound')
    if (savedInitialSound) {
      const audio = document.createElement('audio')
      audio.src = savedInitialSound
      document.body.appendChild(audio)
    }

    time = new Date()

    const delay = 1000 - time.getMilliseconds()
    setTimeout(() => {
      time = new Date()
      interval = setInterval(() => {
        time = new Date()

        if (!alarmSoundSet && !initialSoundSet) return

        const isWarningTime = alarms.some((alarm) => {
          const alarmMinutes = alarm.getHours() * 60 + alarm.getMinutes()
          const currentMinutes = time.getHours() * 60 + time.getMinutes()

          return alarmMinutes - currentMinutes === 45 && time.getSeconds() === 0
        })

        if (isWarningTime) {
          window.electron.ipcRenderer.send('ping', localStorage.getItem('initialSound'))

          return
        }

        const isAlarmTime = alarms.some(
          (alarm) =>
            alarm.getHours() === time.getHours() &&
            alarm.getMinutes() === time.getMinutes() &&
            alarm.getSeconds() === time.getSeconds()
        )

        if (isAlarmTime) {
          window.electron.ipcRenderer.send('ping', localStorage.getItem('alarmSound'))

          return
        }
      }, 1000)
    }, delay)
  })

  onDestroy(() => {
    clearInterval(interval)
  })

  $: hours = Number(time.getHours().toString().padStart(2, '0'))
  $: minutes = time.getMinutes().toString().padStart(2, '0')
  $: seconds = time.getSeconds().toString().padStart(2, '0')

  let audio: HTMLAudioElement | null

  function handleInitialSound(event: Event): void {
    const file = (event.target as HTMLInputElement).files?.[0]
    if (file) {
      const reader = new FileReader()

      reader.onload = (e): void => {
        const base64Audio = e.target?.result as string
        localStorage.setItem('initialSound', base64Audio)

        initialSoundSet = true
      }

      reader.readAsDataURL(file)
    }
  }

  function handleAlarmSound(event: Event): void {
    const file = (event.target as HTMLInputElement).files?.[0]
    if (file) {
      const reader = new FileReader()

      reader.onload = (e): void => {
        const base64Audio = e.target?.result as string
        localStorage.setItem('alarmSound', base64Audio)

        alarmSoundSet = true
      }

      reader.readAsDataURL(file)
    }
  }
</script>

<div class="clock">
  {#if audio}
    <div class="modal-overlay">
      <div class="modal">
        <h2>Playing sound</h2>
        <button
          class="close-button"
          on:click={() => {
            audio.pause()
            audio.currentTime = 0
            audio = null
          }}>Close</button
        >
      </div>
    </div>
  {/if}

  <section class="next-alarm">
    <div class="alarm-info">
      <div class="info-row">
        <div class="sound-status-column">
          <div class="sound-status">
            <h2 class="small-text">Initial Sound: {initialSoundSet ? '' : 'None'}</h2>
            {#if initialSoundSet}
              <button
                on:click={() => {
                  audio = new Audio(localStorage.getItem('initialSound'))

                  audio.play()
                }}
                class="play-button small-button">Play Sound</button
              >
            {/if}
          </div>
          <div class="sound-status">
            <h2 class="small-text">Alarm Sound: {alarmSoundSet ? '' : 'None'}</h2>
            {#if alarmSoundSet}
              <button
                on:click={() => {
                  audio = new Audio(localStorage.getItem('alarmSound'))

                  audio.play()
                }}
                class="play-button small-button">Play Sound</button
              >
            {/if}
          </div>
        </div>
        <h2>
          Next Alarm: {alarms.find((alarm) => alarm > time)?.toLocaleTimeString() ||
            'No upcoming alarms'}
        </h2>
      </div>
    </div>
  </section>

  <section class="time-display">
    <h1>
      {(hours % 12 || 12).toString().padStart(2, '0')}:{minutes}:{seconds}
      {hours >= 12 ? 'PM' : 'AM'}
    </h1>
  </section>

  <section class="audio-controls">
    <div class="input-container">
      <div class="input-wrapper">
        <div class="input-group">
          <label for="initial">Initial Sound</label>
          <input
            on:change={handleInitialSound}
            type="file"
            id="initial"
            accept="audio/mp3"
            class="file-input"
          />
        </div>
        <div class="input-group">
          <label for="alarm">Alarm Sound</label>
          <input
            on:change={handleAlarmSound}
            type="file"
            id="alarm"
            accept="audio/mp3"
            class="file-input"
          />
        </div>
      </div>
    </div>
  </section>
</div>

<style>
  .modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0, 0, 0, 0.5);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1000;
  }

  .modal {
    background: #2d2d2d;
    padding: 2rem;
    border-radius: 12px;
    box-shadow: 0 4px 16px rgba(0, 0, 0, 0.3);
  }

  .close-button {
    margin-top: 1rem;
    padding: 0.5rem 1rem;
    background: #4a9eff;
    border: none;
    border-radius: 6px;
    color: white;
    cursor: pointer;
  }

  .close-button:hover {
    background: #3b8be6;
  }

  /* Add these new styles */
  .small-text {
    font-size: 1rem !important;
    opacity: 0.7;
  }

  .small-button {
    font-size: 0.8rem;
    padding: 0.3rem 0.8rem;
  }

  .alarm-info {
    background: rgba(45, 45, 45, 0.8);
    padding: 1.5rem;
    border-radius: 12px;
    box-shadow: 0 4px 16px rgba(0, 0, 0, 0.15);
    backdrop-filter: blur(10px);
  }

  .info-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 2rem;
  }

  .sound-status {
    display: flex;
    align-items: center;
    gap: 1rem;
    margin-top: 1rem;
  }

  .play-button {
    background: #4a9eff;
    color: white;
    border: none;
    padding: 0.5rem 1rem;
    border-radius: 6px;
    cursor: pointer;
    transition: background-color 0.2s;
  }

  .play-button:hover {
    background: #3b8be6;
  }

  .next-alarm {
    margin-bottom: 2rem;
  }

  .next-alarm h2 {
    color: #ffffff;
    font-size: 1.5rem;
    font-weight: 400;
    opacity: 0.8;
  }

  .clock {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    height: 100vh;
    padding: 2rem;
  }

  .time-display {
    text-align: center;
    margin-bottom: 4rem;
  }

  h1 {
    font-size: 7rem;
    font-family: 'Inter', 'Arial', sans-serif;
    font-weight: 300;
    color: #ffffff;
    margin: 0;
    text-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
  }

  .audio-controls {
    width: 100%;
    max-width: 800px;
  }

  .input-container {
    background: rgba(45, 45, 45, 0.8);
    padding: 2.5rem;
    border-radius: 12px;
    box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
    backdrop-filter: blur(10px);
  }

  .input-wrapper {
    display: flex;
    gap: 2rem;
    justify-content: space-between;
  }

  .input-group {
    flex: 1;
  }

  .input-group label {
    display: block;
    margin-bottom: 0.75rem;
    font-weight: 500;
    color: #ffffff;
    font-size: 1rem;
    letter-spacing: 0.5px;
  }

  .file-input {
    width: 100%;
    padding: 1rem;
    border: 2px solid rgba(255, 255, 255, 0.1);
    border-radius: 8px;
    background: rgba(51, 51, 51, 0.8);
    color: #ffffff;
    transition: all 0.2s ease;
    cursor: pointer;
  }

  .file-input:hover {
    border-color: rgba(255, 255, 255, 0.2);
    background: rgba(51, 51, 51, 0.9);
  }

  .file-input:focus {
    outline: none;
    border-color: #4a9eff;
    box-shadow: 0 0 0 3px rgba(74, 158, 255, 0.2);
  }

  @media (max-width: 768px) {
    .input-wrapper {
      flex-direction: column;
      gap: 1.5rem;
    }

    h1 {
      font-size: 5rem;
    }
  }
</style>
