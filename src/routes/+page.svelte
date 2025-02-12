<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { tweened } from 'svelte/motion';
  import { cubicOut } from 'svelte/easing';
  import { fade, scale } from 'svelte/transition';

  let beganStudy = false;

  let currentFrequencyIndex = 0;
  type AudioConfig = 
    | { type: 'binaural'; name: string; left: number; right: number }
    | { type: 'pure'; name: string; frequency: number };
  const audioConfigurations: AudioConfig[] = [
    { type: 'binaural', name: '40Hz Binaural', left: 200, right: 240 },
    { type: 'binaural', name: '15Hz Binaural', left: 200, right: 215 },
    { type: 'pure', name: '240Hz Control', frequency: 240 }
  ];
  let currentFocusLevel = tweened(0, {
    duration: 800,
    easing: cubicOut
  });
  let historyData: { frequency: number; focusLevel: number }[] = [];

  let ws: WebSocket;

  let audioContext: AudioContext | null = null;
  let leftOscillator: OscillatorNode | null = null;
  let rightOscillator: OscillatorNode | null = null;
  let leftPanner: StereoPannerNode | null = null;
  let rightPanner: StereoPannerNode | null = null;
  let masterGain: GainNode | null = null;
  let isPlaying = false;
  let timer: number | null = null;

  let isConnected = false;
  let statusMessage = "Connecting to EEG stream...";

  let remainingSeconds = 120;
  let countdownInterval: number | null = null;

  let participantName = '';
  let showModal = true;

  let isCooldown = false;
  let cooldownSeconds = 60;
  let cooldownInterval: number | null = null;

  let gainNode: GainNode | null = null;
  const FADE_TIME = 0.01; // 10ms fade in/out to prevent clicks

  let isCalibrating = false;
  let calibrationSeconds = 120; // 2 minutes
  let calibrationInterval: number | null = null;

  function updateFromEEGData(data: any) {
    console.log('Processing EEG data:', data);
    
    if (data.connected === false) {
      console.log('EEG not connected:', data.message);
      isConnected = false;
      statusMessage = data.message || "EEG stream not connected";
      return;
    }

    console.log('EEG connected, focus level:', data.focus_level);
    isConnected = true;
    statusMessage = "Connected to Muse";  // Changed from "Connected to Muse 2"
    currentFocusLevel.set(data.focus_level);

    if (isPlaying) {
      const currentConfig = audioConfigurations[currentFrequencyIndex];
      historyData = [...historyData, {
        frequency: currentConfig.type === 'binaural' ? 
          currentConfig.right - currentConfig.left : 
          currentConfig.frequency,
        focusLevel: data.focus_level
      }];
    }
  }

  function connectWebSocket() {
    console.log('Attempting to connect to WebSocket...');
    
    ws = new WebSocket('ws://localhost:8000/ws');
    
    // Add heartbeat interval
    let heartbeatInterval: number;
    
    ws.onopen = () => {
      console.log('WebSocket connected successfully');
      // Start heartbeat
      heartbeatInterval = window.setInterval(() => {
        if (ws.readyState === WebSocket.OPEN) {
          ws.send(JSON.stringify({ type: 'ping' }));
        }
      }, 1000); // Send ping every second
    };
    
    ws.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        console.log('Raw WebSocket data received:', event.data);
        
        // Skip processing pong messages
        if (data.type === 'pong') {
          return;
        }
        
        // For all other messages, update the UI
        if (data.focus_level !== undefined) {
          console.log('Focus level update:', data.focus_level);
          currentFocusLevel.set(data.focus_level);
        }
        
        updateFromEEGData(data);
      } catch (error) {
        console.error('Error parsing WebSocket data:', error);
      }
    };

    ws.onclose = () => {
      console.log('WebSocket connection closed');
      clearInterval(heartbeatInterval);
      isConnected = false;
      statusMessage = "Connection lost. Reconnecting...";
      setTimeout(connectWebSocket, 1000);
    };

    ws.onerror = (error) => {
      console.error('WebSocket error:', error);
      isConnected = false;
      statusMessage = "Connection error. Check if backend is running on correct port";
    };
  }

  function startTone() {
    if (isPlaying) return;
    
    audioContext = new AudioContext({ sampleRate: 96000 });
    masterGain = audioContext.createGain();
    masterGain.gain.setValueAtTime(0, audioContext.currentTime);
    masterGain.gain.linearRampToValueAtTime(0.1, audioContext.currentTime + FADE_TIME);
    masterGain.connect(audioContext.destination);

    const currentConfig = audioConfigurations[currentFrequencyIndex];

    if (currentConfig.type === 'binaural') {
      // Create and configure left channel
      leftOscillator = audioContext.createOscillator();
      leftPanner = audioContext.createStereoPanner();
      leftOscillator.type = 'sine';
      leftOscillator.frequency.setValueAtTime(currentConfig.left, audioContext.currentTime);
      leftPanner.pan.setValueAtTime(-1, audioContext.currentTime); // Full left
      leftOscillator.connect(leftPanner);
      leftPanner.connect(masterGain);
      leftOscillator.start();

      // Create and configure right channel
      rightOscillator = audioContext.createOscillator();
      rightPanner = audioContext.createStereoPanner();
      rightOscillator.type = 'sine';
      rightOscillator.frequency.setValueAtTime(currentConfig.right, audioContext.currentTime);
      rightPanner.pan.setValueAtTime(1, audioContext.currentTime); // Full right
      rightOscillator.connect(rightPanner);
      rightPanner.connect(masterGain);
      rightOscillator.start();
    } else {
      // Pure tone (control)
      leftOscillator = audioContext.createOscillator();
      leftOscillator.type = 'sine';
      leftOscillator.frequency.setValueAtTime(currentConfig.frequency, audioContext.currentTime);
      leftOscillator.connect(masterGain);
      leftOscillator.start();
    }

    isPlaying = true;
    remainingSeconds = 120;

    // Start countdown
    countdownInterval = setInterval(() => {
      remainingSeconds--;
      if (remainingSeconds <= 0) {
        stopTone();
      }
    }, 1000);

    // Update timer duration to 2 minutes (120000ms)
    timer = setTimeout(() => {
      stopTone();
    }, 120000);
  }

  function stopTone() {
    if (!isPlaying) return;
    
    const stopTime = audioContext?.currentTime || 0;
    
    // Fade out master gain
    if (masterGain && audioContext) {
      masterGain.gain.linearRampToValueAtTime(0, stopTime + FADE_TIME);
    }
    
    // Stop everything after fade-out
    setTimeout(() => {
      leftOscillator?.stop();
      rightOscillator?.stop();
      leftOscillator?.disconnect();
      rightOscillator?.disconnect();
      leftPanner?.disconnect();
      rightPanner?.disconnect();
      masterGain?.disconnect();
      audioContext?.close();
      
      leftOscillator = null;
      rightOscillator = null;
      leftPanner = null;
      rightPanner = null;
      masterGain = null;
      audioContext = null;
    }, FADE_TIME * 1000);

    isPlaying = false;
    remainingSeconds = 120;

    if (timer) {
      clearTimeout(timer);
      timer = null;
    }
    if (countdownInterval) {
      clearInterval(countdownInterval);
      countdownInterval = null;
    }

    // Start cooldown with proper timing
    isCooldown = true;
    cooldownSeconds = 60;
    cooldownInterval = window.setInterval(() => {  // Use window.setInterval for more precise timing
      cooldownSeconds--;
      if (cooldownSeconds <= 0) {
        isCooldown = false;
        if (cooldownInterval) {
          clearInterval(cooldownInterval);
          cooldownInterval = null;
        }
      }
    }, 1000);  // Ensure it's exactly 1000ms
  }

  function changeFrequency(direction: 'next' | 'prev') {
    if (direction === 'next') {
      currentFrequencyIndex = (currentFrequencyIndex + 1) % audioConfigurations.length;
    } else {
      currentFrequencyIndex = (currentFrequencyIndex - 1 + audioConfigurations.length) % audioConfigurations.length;
    }

    // If currently playing, stop the current tone and start the new one
    if (isPlaying) {
      stopTone();
      startTone();
    }
  }

  function handleKeydown(event: KeyboardEvent) {
    if (event.key === 'ArrowLeft' && !isFirstFrequency()) {
      changeFrequency('prev');
    } else if (event.key === 'ArrowRight' && !isLastFrequency()) {
      changeFrequency('next');
    } else if (event.key.toLowerCase() === 'r') {
      currentFrequencyIndex = 0;
      if (isPlaying) {
        stopTone();
        startTone();
      }
    }
  }

  // Add helper functions
  function isFirstFrequency() {
    return currentFrequencyIndex === 0;
  }

  function isLastFrequency() {
    return currentFrequencyIndex === audioConfigurations.length - 1;
  }

  function downloadHistoryAsCSV() {
    const headers = ['Participant Name', 'Frequency (Hz)', 'Focus Level (%)'];
    const csvContent = [
      headers.join(','),
      ...historyData.map(row => `${participantName},${row.frequency},${row.focusLevel.toFixed(2)}`)
    ].join('\n');

    const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
    const link = document.createElement('a');
    const url = URL.createObjectURL(blob);
    link.setAttribute('href', url);
    const fileName = `[Focus Study] ${participantName}.csv`;
    link.setAttribute('download', fileName);
    link.style.display = 'none';
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    URL.revokeObjectURL(url);
  }

  function handleSubmitName() {
    if (participantName.trim()) {
      beganStudy = true;
      showModal = false;
    }
  }

  function startCalibration() {
    if (!isConnected) return;
    
    isCalibrating = true;
    calibrationSeconds = 120;
    
    // Send calibration start message to WebSocket
    ws.send(JSON.stringify({ type: 'start_calibration' }));
    
    calibrationInterval = window.setInterval(() => {
      calibrationSeconds--;
      if (calibrationSeconds <= 0) {
        stopCalibration();
      }
    }, 1000);
  }

  function stopCalibration() {
    isCalibrating = false;
    if (calibrationInterval) {
      clearInterval(calibrationInterval);
      calibrationInterval = null;
    }
    ws.send(JSON.stringify({ type: 'stop_calibration' }));
  }

  onMount(() => {
    connectWebSocket();

    return () => {
      if (ws) ws.close();
      stopTone();
      if (cooldownInterval) {
        clearInterval(cooldownInterval);
      }
      if (calibrationInterval) {
        clearInterval(calibrationInterval);
      }
    };
  });

  onDestroy(() => {
    if (ws) ws.close();
    stopTone();
    if (cooldownInterval) {
      clearInterval(cooldownInterval);
    }
    if (calibrationInterval) {
      clearInterval(calibrationInterval);
    }
  });
</script>

<svelte:window on:keydown={handleKeydown}/>

<svelte:head>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap" rel="stylesheet">
</svelte:head>

<div class="min-h-screen bg-white py-12 px-4 sm:px-6 lg:px-8 font-sans text-gray-900">
  <div class="max-w-7xl mx-auto space-y-12">
    <header class="text-center mb-16">
      <div class="space-y-2 mb-6">
        <h1 class="text-4xl font-medium text-gray-900">Audio Frequency Focus Study</h1>
        <p class="text-md text-gray-400">Made by Jude Kim in BNSS</p>
      </div>
      
      {#if beganStudy}
        <div class="flex items-center justify-center gap-4 text-sm">
          <div class="flex items-center gap-2 px-3 py-1.5 bg-gray-50 rounded-lg">
            <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 text-gray-400" viewBox="0 0 20 20" fill="currentColor">
              <path fill-rule="evenodd" d="M10 9a3 3 0 100-6 3 3 0 000 6zm-7 9a7 7 0 1114 0H3z" clip-rule="evenodd" />
            </svg>
            <span class="text-gray-600">{participantName}</span>
          </div>

          <div class={`flex items-center gap-2 px-3 py-1.5 rounded-lg
            ${isConnected ? 'bg-emerald-50 text-emerald-700' : 'bg-red-50 text-red-700'}`}>
            <div class={`w-2 h-2 rounded-full ${isConnected ? 'bg-emerald-500' : 'bg-red-500'}`}></div>
            <span class="text-xs">{statusMessage}</span>
          </div>
        </div>
      {/if}
    </header>

    <div class="grid grid-cols-1 md:grid-cols-2 gap-8">
      <div class="bg-white rounded-3xl p-8 shadow-sm border border-gray-100 flex flex-col">
        <h2 class="text-sm font-medium text-gray-500 mb-6">Audio Frequency</h2>
        <div class="flex-1">
          <div class="flex items-center space-x-6">
            <button 
              class="transition-colors duration-200 {isFirstFrequency() ? 'text-gray-300' : 'text-gray-900 hover:text-gray-700'}" 
              on:click={() => changeFrequency('prev')}
              disabled={isFirstFrequency()}
            >
              <svg xmlns="http://www.w3.org/2000/svg" class="h-8 w-8" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 19l-7-7 7-7" />
              </svg>
            </button>
            
            <div class="text-center flex-1">
              <p class="text-5xl font-medium text-gray-900">
                {(() => {
                  const config = audioConfigurations[currentFrequencyIndex];
                  return config.type === 'binaural' ? 
                    config.right - config.left : 
                    config.frequency;
                })()}
                <span class="text-lg text-gray-400 ml-1">Hz</span>
              </p>
              <p class="text-sm text-gray-500 mt-2">
                {audioConfigurations[currentFrequencyIndex].name}
              </p>
              {#if audioConfigurations[currentFrequencyIndex].type === 'binaural'}
                <p class="text-xs text-gray-500 mt-1">
                  L: {(audioConfigurations[currentFrequencyIndex] as { left: number, right: number }).left}Hz, 
                  R: {(audioConfigurations[currentFrequencyIndex] as { left: number, right: number }).right}Hz
                </p>
              {/if}
            </div>

            <button 
              class="transition-colors duration-200 {isLastFrequency() ? 'text-gray-300' : 'text-gray-900 hover:text-gray-700'}" 
              on:click={() => changeFrequency('next')}
              disabled={isLastFrequency()}
            >
              <svg xmlns="http://www.w3.org/2000/svg" class="h-8 w-8" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
              </svg>
            </button>
          </div>
        </div>
        <div class="mt-auto space-y-2">
          <button 
            class="w-full py-2 bg-gray-900 text-white text-sm rounded-lg hover:bg-gray-800 transition-colors duration-200 disabled:opacity-50"
            on:click={isPlaying ? stopTone : startTone}
            disabled={isCooldown}
          >
            {isPlaying ? 'Stop' : 'Play'} Tone
          </button>
          {#if isPlaying || isCooldown}
            <p class="text-sm text-center text-gray-500">
              {isPlaying ? `${remainingSeconds}s` : `Cooldown: ${cooldownSeconds}s`}
            </p>
          {/if}
        </div>
      </div>

      <div class="bg-white rounded-3xl p-8 shadow-sm border border-gray-100 flex flex-col">
        <h2 class="text-sm font-medium text-gray-500 mb-6">Focus Level</h2>
        <div class="flex-1 flex flex-col items-center justify-center mb-8">
          <div class="relative w-32 h-32">
            <svg class="w-full h-full" viewBox="0 0 100 100">
              <circle class="text-gray-50 stroke-current" stroke-width="6" cx="50" cy="50" r="44" fill="transparent"/>
              <circle class="text-gray-900 progress-ring stroke-current" stroke-width="6" cx="50" cy="50" r="44" fill="transparent"
                      stroke-dasharray="276.46"
                      stroke-dashoffset={276.46 - (276.46 * $currentFocusLevel / 100)}
              />
            </svg>
            <div class="absolute inset-0 flex items-center justify-center">
              <span class="text-3xl font-medium text-gray-900">{$currentFocusLevel.toFixed(0)}%</span>
            </div>
          </div>
        </div>
        <div class="mt-auto">
          <button 
            class="w-full py-2 bg-gray-100 text-gray-900 text-sm rounded-lg hover:bg-gray-200 transition-colors duration-200 disabled:opacity-50"
            on:click={startCalibration}
            disabled={!isConnected || isCalibrating || isPlaying}
          >
            Calibrate
          </button>
        </div>
      </div>
    </div>

    <div class="bg-white rounded-xl border border-gray-100 overflow-hidden">
      <div class="p-4 border-b border-gray-100 flex justify-between items-center">
        <h2 class="text-sm font-medium text-gray-500">History</h2>
        <button 
          class="px-3 py-1.5 bg-gray-100 text-gray-900 text-sm rounded-lg hover:bg-gray-200 transition-colors duration-200 disabled:opacity-50"
          on:click={downloadHistoryAsCSV}
          disabled={historyData.length === 0}
        >
          Download
        </button>
      </div>
      <div class="overflow-x-auto">
        <table class="w-full">
          <thead>
            <tr class="bg-gray-100">
              <th class="py-4 px-6 text-left text-sm font-medium text-gray-600 uppercase tracking-wider">Frequency (Hz)</th>
              <th class="py-4 px-6 text-left text-sm font-medium text-gray-600 uppercase tracking-wider">Focus Level</th>
            </tr>
          </thead>
          <tbody class="divide-y divide-gray-200">
            {#each historyData as { frequency, focusLevel }, i}
              <tr class="bg-white">
                <td class="py-4 px-6 text-sm text-gray-900">{frequency}</td>
                <td class="py-4 px-6 text-sm text-gray-900">{focusLevel.toFixed(2)}%</td>
              </tr>
            {/each}
          </tbody>
        </table>
      </div>
    </div>
  </div>
</div>

{#if showModal}
  <div class="fixed inset-0 bg-gray-600 bg-opacity-50 overflow-y-auto h-full w-full z-50 flex items-center justify-center"
       transition:fade={{ duration: 200 }}>
    <div class="relative mx-auto p-8 w-full max-w-md bg-white rounded-2xl shadow-2xl transform transition-all"
         in:scale={{ duration: 200, start: 0.95 }}
         out:scale={{ duration: 200, start: 1 }}>
      <div class="text-center space-y-6">
        <div class="space-y-2">
          <h3 class="text-2xl font-bold text-gray-900">Welcome to the Focus Study</h3>
          <p class="text-gray-500">Please enter the participant's name to begin</p>
        </div>
        
        <form on:submit|preventDefault={handleSubmitName} class="space-y-4">
          <div>
            <input
              type="text"
              bind:value={participantName}
              placeholder="Enter participant name"
              class="w-full px-4 py-3 rounded-lg border border-gray-300 focus:ring-2 focus:ring-blue-500 focus:border-blue-500 transition-colors duration-200"
              autofocus
            />
          </div>
          
          <button
            type="submit"
            class="w-full py-3 px-4 bg-blue-500 hover:bg-blue-600 text-white font-medium rounded-lg transition-colors duration-200 disabled:opacity-50 disabled:cursor-not-allowed"
            disabled={!participantName.trim()}
          >
            Begin Study
          </button>
        </form>
      </div>
    </div>
  </div>
{/if}

{#if isCalibrating}
  <div class="fixed inset-0 bg-black bg-opacity-30 backdrop-blur-sm overflow-y-auto h-full w-full z-50 flex items-center justify-center"
       transition:fade={{ duration: 300 }}>
    <div class="relative mx-auto p-8 w-full max-w-md bg-white rounded-2xl shadow-xl"
         in:scale={{ duration: 200, start: 0.95 }}
         out:scale={{ duration: 200, start: 1 }}>
      <div class="text-center space-y-6">
        <h3 class="text-2xl font-semibold text-gray-900">Calibrating</h3>
        <p class="text-gray-600 text-sm">
          Close your eyes and relax for the next 2 minutes.
        </p>
        <div class="text-4xl font-bold text-gray-900">
          {calibrationSeconds}s
        </div>
        <div class="w-full bg-gray-100 rounded-full h-1">
          <div class="bg-gray-900 h-1 rounded-full transition-all duration-1000"
               style="width: {((120 - calibrationSeconds) / 120) * 100}%">
          </div>
        </div>
      </div>
    </div>
  </div>
{/if}

<style>
  :global(body) {
    font-family: 'SF Pro Display', 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen-Sans, Ubuntu, Cantarell, 'Helvetica Neue', sans-serif;
  }
  .progress-ring {
    transition: stroke-dashoffset 1s cubic-bezier(0.4, 0, 0.2, 1);
    transform: rotate(-90deg);
    transform-origin: 50% 50%;
  }
  :global(body.modal-open) {
    overflow: hidden;
  }
  
  input:focus {
    outline: none;
  }
</style>